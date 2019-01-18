## Metadata
```
  Title: A Proposed Urbit Hosting Architecture and Ops Plan using AWS
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2018.12.17
  Last Modified: ~2018.12.17
```

## Overview / Requirements

Tlon wishes to provide customers turn-key urbit service.

What the customer has to worry about:

* "I tap this icon and a terminal opens up and I am logged into my own personal urbit"

What Tlon has to worry about (and the customer does not):

* Spinning up AWS servers, with a good choice of
    * instance power
	* instance OS
	* data center
	* availability zone
* Installing and configuring FoundationDB on some of the servers
* Installing and configuring urbit on others of the servers
* Configuring king and serf
* Configuring S3 to hold snapshots / backups
* Creating or picking some tool to do periodic backups
* Actually running that tool on some schedule to do backups
* Tracking resources used / costs incurred, and associating them with user accounts for billing purposes
* Writing a tool ("the provisioner") to do all of the above for each new customer, so that we don't have an intern frantically mouse-clicking AWS console pages as we acquire users

## AWS EC2 options, pricing, and implications for architecture

Before diving into declaring what we'll use, let's briefly review AWS
EC2 options.

Amazon lets us pick either dedicated machines or spot instances.
Dedicated machines run when we want them to run. Spot instances are
cheaper, but run when Amazon has spare resources.  Spot instances are
unacceptable for our model.

Among dedicated machines, Amazon lets us pick virtual machines of
different capabilities (number of CPUs, amount of memory, amount of
storage), and with different optimizations (GPU, fast memory, etc.).
As Knuth said, premature optimization is the root of all evil.  For
this v0.1 document (and project), it is taken as given that we will
use general purpose machines.

Instance cost is roughly linear with capabilities.  E.g. an instance
with 16 CPUs and 32 GiB of memory ("a1.4xlarge") costs $0.408/hour,
while a machine with exactly 1/16th as much CPU and 1/16th as much
memory ("a1.medium) costs $0.051/hour ... or exactly 1/16th as much.

Given this, I propose right off the bat that that we segment each Tlon
customer's urbit onto their own instance.

Someone is going to be in charge of keeping customer A's computation
separate from customer B's ...and if Amazon is willing to do this for
free, why not let them?

One possible response is "the usage of customer A's urbit and customer
B's urbit are not correlated 1.0, therefore we can get more
computation per dollar by putting multiple urbits on one instance".

I think that we should avoid that temptation at this stage in the game.

## AWS S3 options, pricing, and implications for architecture

S3 pricing starts at $0.023 / GB / month for the typical easy-access
usage model, and can be as cheap as $0.004 / GB / month for "glacier
storage." 

Regular S3 storage has a cost of $0.03 / GB retrieved at expedited
speed (retrievals have 1-5 minute latency).

Glacier S3 storage has a cost of $0.01 / GB retrieved at normal speed
(retrievals have 3-5 hour latency).


We expect that customers will maintain and average of 5 snapshots of
average size bounded at 2 GB.  We expect that customers will retrieve
a snapshot once year.

These figures are such that the savings from Glacier storage are
neglible, and we'll use regular S3 storage.

## Urbit on AWS Architecture 

### Overview

A since EC2 instance hosts the Tlon-written Provisioner.

Each Tlon customers gets a dedicated EC2 instance for their Urbit.

Up to X Tlon Customers share a FoundationDB cluster made up of three
EC2 instances.  The value of X is yet to be determined.

A single S3 account holds multiple buckets - one bucket per Urbit.
Each bucket holds snapshots associated with that Urbit.

## Lifecycle

The Provisioner provides a web page that customers can use to
provision a new urbit instance.  When supplied with a credit card and
credentials (XXX details needed here), the Provisioner uses the AWS
EC2 API ( https://docs.aws.amazon.com/AWSEC2/latest/APIReference/Welcome.html )
to spin up a new instance and to associate it with a FoundationDB
cluster.  If there is not a cluster with an open client slot, the
Provisioner spins up a new FoundationDB cluster as well.

Once per month the Provisioner runs in "billing" mode, charges credit
cards, and decomissions any urbit instance that has not paid its
hosting bill because of expired credit card or other failure.  Decomission means:
* direct the urbit to emit a snapshot
* move the snapshot to AWS S3
* spin down the EC2 instance the urbit was running on
* make a note in the provisioner database that the FoundationDB cluster the urbit was associated with now has one more free slot
* email the customer
* email a Tlon employee

Once per day the Provisioner runs in "snapshot" mode, and copies each
urbit's most recent snapshot (created independently by the urbit) into
the correct AWS S3 bucket, and deletes the oldest snapshot from the
same bucket.

As needed, a Tlon employee logs into the Provisioner admin screen and
views statistics, looks up details, and - for tasks that the
Provisioner does not automate - logs into AWS EC2 admin console, or
into an EC2 instance directly via SSH to do things like recover
snapshots and restore urbits.

## Details

The Provisioner, a web app written in either C++ or Ruby and attached
to a Postgres database, provides a web page that customers can use to
subscribe to urbit-as-a-service.  When supplied with a credit card and
credentials (XXX details needed here), the Provisioner creates a
record for the customer in the local Postgres database and uses the
AWS EC2 API (
https://docs.aws.amazon.com/AWSEC2/latest/APIReference/Welcome.html )
which has bindings in C++, Ruby, Python, and all the other expected
languages, to spin up a new instance.

In v2.0 we will give customers control over how much horsepower they
want in their urbit, but for v1.0 all customers will receive a
t3.nano, which Amazon sells for $0.0052/hour ($3.75 / month).

The Provisioner will also associate the urbit instance with a
persistance store.  There are two possibilities: either each customer
gets three additional t3.nanos to create a dedicated FoundationDB
cluster (total cost = $14.98/month), or multiple urbits share a single
FoundationDB cluster.

Because of the cost savings, it is proposed that multuiple customers
share a FoundationDB cluster.

Despite one or two ambiguous google hits, FoundationDB does not seem
to offer, out of the box, multi-tenant capabilities.

We can engineer multi-tenant capabilities ourselves by altering the
fragmenting system in vere/frag.c that vere/fond.c uses.  By adding a
third field to the header - tenant ID (presumably the urbit ship ID) -
we can ensure that all reads from common database cluster C by tenant
A are restricted just to records written by tenant A.

When we spin up three new instances to create a FoundationDB cluster,
we will store details of that cluster in the Provisioner database, so
that we can configure new urbits to use the same cluster.

## Cost of services provided.

If each urbit is hosted on a t3.nano, and 10 urbits share a cluster of
3 t3.nanos, and each urbit stores five 2GB snapshots and retrieves one
per year, then the base service level costs approximately $5 per
month, plus incidentals for the Provisioner.


## Open issues

* What credentials do customers use to tell the Provisioner to spin up an urbit for them?  
* Do we store credit cards in the Provisioner database?
* How many urbits share one FoundationDB cluster?

TJIC to do:

* speak to IP addresses ; specify that we're not going to pay extra for fixed; we're OK with standard EC2 behavior
* speak to security re the admin screen of the provisioner
* speak to which AWS datacenters we use
* speak to availability zones
* investigate AWS S3 zones

## Future features not included in v1.0

* customer control over instance size / power

* rebalancing FoundationDB clusters: in 1.0 if the first 10 urbits
  share one cluster, and then three of those urbits do inactive, we
  will allow that FoundationDB cluster to be underutilized with just 7
  urbits.




## See also

* companion document "A Tutorial to Spin Up Urbit in AWS (with FoundationDB)".

