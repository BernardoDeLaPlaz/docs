## Metadata
```
  Title: A Proposal for One Touch Urbit (Phase 1)
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2019.03.01
  Last Modified: ~2019.03.01
```

## Customer Experience

A customer C visits hosting.urbit.org, sees the description of free
hosting, and clicks to sign up.

The second page explains that the site does not sell  the free hosting, the performance guarantees
(none), and the insurance coverage in case Tlon Inc. leaks his keys
and thus loses his ship.

Lower on the page is a web form, which prompts him for credentials for
either a star or a planet.

He has credentials for a star (which he purchased from a friend who
has a galaxy).

He clicks on his brower to verify the the SSL certificate shows that
he is truly visiting urbit.org.

He pastes his credentials into the web form, and clicks the button.

He reaches the second page, which tells him to

   (a) run command
   	   ssh-keygen -t rsa -f ~/.ssh/keypairs/hosted_<shipname>

   (b) open the file
   	   		~/.ssh/keypairs/hosted_<shipname>.pub
	   and copy the contents into the text entry field

He does so.

He reaches the third page, which tells him that

   (a) to be very careful with his ssh private key file
   (b) back up his ssh private key file
   (c) his ship is now hosted
   (d) he can reach it by entering the following in a terminal:

   	   ssh -p <port> <shipname>@<shipname>.urbit.org -i .ssh/keypairs/hosted_<shipname> 

   (e) he should save the above line of text, and/or print it out

He copies-and-pastes the line of text into a terminal and is connected
to his ship.

The familiar dojo prompt greets him.

He plays with dojo for a while, then hits CONTROL-d to exit.

He is logged out of dojo...and out of the ssh session.


No other capabilities (app store, scaling RAM or CPU use, etc.) are supported.

## Capabilities Required


### Capabilities Required: Customer Facing Website

We need a website hosting.urbit.org which three four pages:

* page 1: explanation, ask customer for ship credentials, go button
* page 2: ask customer for ssh public key, confirm button
* page 3: instructions on how to connect

The capabilities are:

* w-1: contact the provisioner and ask for a hosting slot to be
          allocated and for a DNS entry be created
* w-2: contact the provisioner with the customer's ssh key and ask
  		  for it to be stored on the AWS EC2 instance


### Capabilities Required: Provisioner

The Provisioner is described in more detail in "A Proposed Urbit Hosting Architecture and Ops Plan using AWS".

    https://github.com/BernardoDeLaPlaz/urbit_docs/blob/master/docs/arch/aws_ops_architecture.md

	...an app written in ...  Ruby and attached to a Postgres
	database, provides a web page that customers can use to subscribe
	to urbit-as-a-service...

The original conception of this was as a Ruby app, but we can, if
necessary, write it in Hoon.

The Provisioner is driven by the customer-facing webapp and the admin-facing webapp.

The Provisioner has the following capabilities:

	* p-1: maintain a list of AWS EC2 instances
	* p-2: maintain a list of hosted urbit instances
	* p-3: listen for messages from the customer-facing website
	* p-4: spin up a new AWS EC2 instances
	* p-5: create a new user account on an AWS EC2 instances
	* p-6: create a new hosted instance of urbit 
	* p-7: update DNS (see below) to map <shipname>.urbit.org to an AWS EC2 instance
	* p-8: respond to queries by the querier and give data
	* p-9: hold AWS credentials in its jael
	
Details:

	* p-1: maintain a list of AWS EC2 instances - done within the provisioner's urbit state.

	* p-2: maintain a list of hosted urbit instances - done within the provisioner's urbit state.

	* p-3: listen for messages from the customer-facing website - via
      events over Ames.  

	* p-4: spin up a new AWS EC2 instances

	  using HTTP get requests from vere/cttp.c. (all AWS EC2 API commands
      are available using HTTP / HTTPS GET / POST - see page
	  1489 of this document.

	 * p-5: create a new user account on an AWS EC2 instances - 
	  
	  as per p-4.

 	* p-6: create a new hosted instance of urbit -

	  If this is the first instance of urbit on the EC2 instance we
	  need to push a copy of the urbit executable to the EC 2
	  instance.  This can be done by using EC2 templates.

	  However, we also need to (a) create a new user on the EC2
	  instance, (b) take an ssh public key that the user pasted in,
	  and put that in a file.  So far I can not find a way to do
	  either of these two things in the EC2 HTTP API.

	  If there is no way to do it via the API, we have a few other
	  options:

	     1) install some sort of web-based ssh gateway on the
	     provisioner, so that the provisioner can post HTTP requests
	     to localhost, and the gateway turns these into ssh
	     connections to the target machine

		 	 https://en.wikipedia.org/wiki/Web-based_SSH

		 2) on the template that we use for new EC2 instances, install
		 a webserver that has some CGI script that we use to implement
		 the missing API calls: the provisioner POSTs to machine823
		 port 80, with a payload of <new username>,<new ssh key>, and
		 the script does everything we need.

	* p-7: update DNS (see below) to map <shipname>.urbit.org to an
      AWS EC2 instance

	  Using an HTTP interface.

	* p-8: respond to queries by the querier and give data


	* p-9: hold AWS credentials in its jael


In phase 1, the provisioner does not have the following capabilities:

	* deallocate an urbit instance from an EC2 instance
	* deallocate an EC2 instance
	* monitor EC2 instance load

### Capabilities Required: DNS

The DNS Tool is a standard off-the-shelf DNS daemon.  We need one that
has an HTTP API, so that we can set new mappings from the provisioner.

Capabilities required:

	* d-1: provide DNS service for <shipname>.urbit.org, for every
	          ship hosted, so that customers can find their ships
	          (even if, in the future, we move them to other EC2
	          instances, or outside of Amazon entirely)
	* d-2: accept some form of HTTP GET / POST API to add new records


### Capabilities Required: Urbit client 

The "urbit client" is merely a terminal and an ssh command.

### Capabilities Required: Querier 

The Querier is a tool that lets admins inspect the system.  It is
a Hoon app accessed via dojo.

It has the following capabilities:

	* q-1: return the total number of hosted ships 
	* q-2: return the growth of hosted ships over time
	* q-3: list all data centers
	* q-4: list EC2 instances per data center
	* q-5: list all urbit instances per EC2 instance

All of these capabilities are implemented by connection to the
Provisioner and asking the latter questions via the querier /
provisioner protocol.  Thus:


	* q-6: speak to the Provisioner with a protocol


### Capabilities Required: Urbit hoster (in flux)

Every EC2 instance has the capability of running multiple ships.

The primary goal is to do so in a way that allows us to run many ships
with minimal expenditure on resources.

We take advantage of the fact that most ships are not running most of
the time. Using the law of large numbers, the more ships we fit on a
given EC2 instance, the closer the behavior of those ships approachs
the average.

How exactly do we conserve CPU resources?

To quote Curtis' hosts.txt file:

	There are three general ways to get idle ships out of RAM:

	One: suspend the process, capture its I/O, and wake on input.
	For example, receive its packets and load it to handle them.

	Two: configure plenty of swap and overprovision the RAM.

	Three: run in Aquarium and make sure shared nouns are shared.

I've read the Aquarium source code and talked to Philip Monk.  On the
one hand, Aquarium seems to be ready for prime time.  On the other
hand, Philip says that shared nouns may be years away.  Without shared
nouns, I have not yet convinced myself that the normal swap space
algorithms will effectively swap out the memory used by one "sleeping"
ship.

Thus, we may wish to not choose option 3.

Option 1 is work.

Option 2 is trivial.

I suggest we choose option 2.

So, now, the question: what tool (if any) do we need on each EC2
instant?  Curtis' document presupposes a comet, in the section entitled

     Execution protocol (between control comet and Vere lord)

I am not (yet) convinced of the need for a control comet on each EC2 instance.

## Protocol: customer facing website <--> the Provisioner  

The CWF initiates all messages; P responds.

message: provide hosting

		 [ %provide  <ship-name>  <ssh public key> ]

		 :: responses

		 [ %response %success ]
		 [ %response %already_hosted ]
		 [ %response %other_error ]


## Protocol: customer Querier <--> the Provisioner  

message: number of hosted ships?

		 [ %query %ships-hosted ]

		 :: responses
		 
		 [ %response %success @ud ]		
		 [ %response error ]

message: number of hosted ships over time?

		 [ %query %ships-hosted-timeseries ]

		 :: responses

		 [ %response (list (pair @d @ud)) ]			:: answer
		 [ %response %other_error ]


message: list all data centers

		 [ %query %datacenters-used (list (list @ta)) ]

		 :: responses

		 [ %response (list (pair @d @ud)) ]			:: answer
		 [ %response %other_error ]

message: list EC2 instances per data center

		 [ %query %instances datacenter=(list @ta) ]

		 :: responses

		 [ %response (list (list @ta)) ]			:: answer  
		 [ %response %other_error ]

message: list urbit ships per EC2 instance

		 [ %query %ships instance=(list @ta) ]

		 :: responses

		 [ %response (list (list @p)) ]			:: answer  
		 [ %response %other_error ]


## Not covered in (this version of) this document

* multiple hosting regions
* snapshots (part of a distinct project)
* multi-pier kings (not required / out of scope)

