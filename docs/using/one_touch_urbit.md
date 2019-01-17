## Metadata
```
  Title: A Proposal for One Touch Urbit
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2019.01.17
  Last Modified: ~2019.01.17
```

## Overview / Requirements

Tlon wishes to provide customers "one touch" / turn-key urbit service.

What does "one touch" / turn-key urbit mean?

Urbit is a utility.  Think about other utilities in your life - and
cast the net widely.

When you bought your house you had water service turned on.  You've
never thought about it again since then.  When you turn the tap, water
comes out.  Once a month, money is deducated from your checking
account.  The same with your electricity, your heat, etc.  Your
computer works this way too: you purchased it by going to a website
and clicking a button, and now whenever you want to use it, you flip
open the lid and it turns on.

Once every year or two you contact your service provider to either
upgrade the service, ask a question, change your credit card, or turn
the service off.

Urbit should require one mouse click (and a credit card) to purchase,
and then should work seamlessly every time you flip up the lid.  

### Customer Experience

The future Urbit customer buys a new urbit the same way she buys
anything - she goes to a website ( http://sales.urbit.org ) and
compares offerings.  She ponders the "self hosted" option (free), but
sees that the features on the "Tlon hosted - nano" ($10/month) are
better.  She sees the checkbox offering concierge support for
$99/month and leaves it unchecked. She clicks "checkout", provides a
credit card and billing address, and clicks "done".

A few moments later two icons appear on her desktop.  One is a
shortcut to http://help.urbit.org, the other is the urbit remote app.
She clicks the urbit remote app.

A text window opens up on her device ( PC, tablet, or phone) - she's
connected to her new urbit.

Inside the text window a "welcome to urbit" screen is displayed.  She
reads through it.  The first bullet point thanks her for joining, the
second tells her that the she could click the help icon any time she
has questions, and a third tells her that the urbit GUI desktop is
arriving soon.  She taps the space bar to clear the message.

A second message tells her that she needs to either print out her
recovery keys or save them to a USB stick.  She clicks "ignore for
now".

She plays with her new urbit for an hour, then turns her device off
before bed.

In the morning she turns her device back on, clicks the icon to open
her urbit.  The warning message pops up again reminding her to save
her keys.  This time she prints them out and put the papers in her
file cabinet.  She clears the message by typing "yes I have saved my
keys", as instructed, and hits RETURN.

She issues the dojo command +apps, and gets a list the urbit apps
available for download from the urbit store.  The list is short, but
he sees a two she wants to try: a spreadsheet, and a personal contacts
manager.  She hits 3, then return, to select the spreadsheet, then 7,
then return, to select them.  Two apps are downloaded into her urbit.
She exits the 'apps' application with control-X and returns to dojo.

She types +contacts to launch the contacts manager, adds several
contacts, then hits control-X to return to dojo.

She launches +sheets to launch the spreadsheet too, creates a new
sheet, and adds data to it to track her mortgage.  She notes an option
to import her contacts data into the sheet, but ignores it.  Once done
she hits control-X to return to dojo.

A month later he hears about some virus or data breach and reaches out
to Tlon to see if her data is safe.  She clicks the "help" icon on her
device and reaches Tlon's technical support website
(http://help.urbit.org).  A banner across the top of the page answers
her questions, but she wants more handholding, so he fills out the
"contact us" form.  As she is not paying for concierge tier service,
it is two days later when she gets a response from a Tlon customer
support agent, assuring her that her data is safe.

Three months later she loses her laptop in an Uber and worries that
her urbit is gone.  She again goes to Tlon's technical support
website.  A tutorial there reassuures her that she is fine, and walks
her through the process of reinstalling her urbit client on her new
laptop, using the keys she printed out on day 2.

Six months later she launches her urbit and gets an alert - her credit
card has expired.  She ignores it.

A day later she gets an email from tlon.io with the same information.
She clicks the link in the email, gets an SSL secured webpage (
http://billing.urbit.org ).  She provides the new card, and clicks a
button.

Four months after that she gets both an email and an alert in her
urbit telling her that space is running low.  She decides that she
wants more space - and also wouldn't mind her urbit running faster.
She visits http://help.urbit.org, and is redirected to
http://account.urbit.org, where she adjusts a slider control for
storage and for CPU.  She's told that her new configuration will cost
$15/month, and she approves the change.

Around the same time she buys a new phone and decides to issue a moon
from her planet.  This can be done via dojo, but she prefers to do it
via the web interface at http://account.urbit.org.  The computer
interface displays a both urbit phonemes and a QR code.

Using her phone, she downloads the urbit client and launches it.  The
urbit client (urbit client installer ?) knows that it is expected to
be a moon (how?) and asks her to either type in the urbit phonemes or take a picture of the QR code.

Because the urbit mnemonics look too complicated, she uses her phone
to take a picture of the QR code displayed on her PC.  The phone urbit
client parses this and configures itself as a moon beneath the planet.

Immediately on her PC her urbit client announces that a new moon has
been created, and asks if she wants to share +contact information (she
selects "yes") or +sheets information (she selects "no").

She picks up her phone, and starts the urbit client.  +contacts
launches the contact app, and it's already populated with her data.
+sheet gives an error - she has not installed that app.

Three years after that she marries, and her partner already owns a
galaxy, so she decides to move her star into their galaxy.  Again she
goes to http://account.urbit.org and performs the action.

She and her partner move to Australia and she notices that her urbit
is laggy.  A day later she gets an email from Tlon.  Tlon has noticed
that she's been connecting to her urbit from a new location and asks
if she wants to move her urbit from the US West / Oregon datacenter to
the Sydney datacenter.  She clicks "yes".  A few minutes later, it's
done.  Her urbit continues to work.

## Knobs

Urbits may be run on either client-administered or Tlon-administered
hardware.  Running an urbit on one's own hardware is cheaper; running
an urbit on Tlon's hardware is easier.  Tlon's "One-touch Urbit"
supports both.

Tlon-administered hardware is available in a variety of speed and
storage settings, at different price points.

Tlon-administered hardware is available in a variety of georgraphical regions.

## Capabilities Required

### Capabilities Required: Admin Facing

We need a website with admin screens that:

* show all customers
* show all urbits (grouped by customer)
* allow the creation, deletion, and editing of storage tiers
* allow the creation, deletion, and editing of computation tiers
* ban a customer and terminate their urbits
* update a credit card for a customer
* view submitted applications for the app store
* approve or deny submitted applications for the app store
* create customer facing banners for important time critical things
  ("the network is down" ; "the Google data breach does not affect
  you")
* list all admins (via the super-admin account)
* create and delete admins (via the super-admin account)

### Capabilities Required: Customer Facing

We need a website with customer-facing screens

* sales.urbit.org:
		* explains hosting choices
		* allows for secure checkout / credit card processing
		* downloads software installer
		
* help.urbit.org:
		* has a "knowledge base" (searchable database of problems and solutions)
		* has a contact-us page

* account.urbit.org: allows users to
		 * log in
		 * upgrade or downgrade their CPU and storage tiers
		 * create moons

All of these websites used one common Postgres database, which has tables for:

* customer
* mailing address (keys: live) (foreign key: customer)
* urbit instance (foreign key: customer)
* urbit hosting (foreign keys: urbit instance, storage tier, compute tier, AWS S3 machine, creation date)
* storage tier choices (fields: available - we don't delete deprecated choices )
* compute tier choices (fields: available - we don't delete deprecated choices )
* payments (foreign key: customer)


### Capabilities Required: Systems / IT

We require multiple tools:

* Provisioner
* Biller
* Alerter
* Querier

The Provisioner is described in "A Proposed Urbit Hosting Architecture and Ops Plan using AWS".

    https://github.com/BernardoDeLaPlaz/urbit_docs/blob/master/docs/using/aws_ops_architecture.md

	...an app written in either C++ or Ruby and attached to a Postgres
	database, provides a web page that customers can use to subscribe
	to urbit-as-a-service...

The Provisioner is driven by the customer-facing webapp and the admin-facing webapp.

The Provisioner has the following capabilities:

* create a new customer (update the database)
* create a new sun (issue keys for)
* create a new hosted instance of urbit (update the database, spin up
  AWS S3 resources)
* adjust CPU level and storage level of an urbit (update the database,
  spin down an AWS S3 instance, spin up a new one)
* move a hosted instance from one datacenter to another (freeze,
  checkpoint, shut down, spin down AWS S3 instance, spin up new one in
  other data facility, copy checkpoint file... and also update
  database)
* shut down a hosted instance of urbit (spin down an AWS S3 instance,
  update the database)
* consistency check (verify that the hosted instances in the database
  match the S3 resources ; we don't want to leak CPUs that we pay
  for!)

The Biller is a headless app that has the following capabilities:

* running once per day, identify any customers who are do to be billed
* charge their credit cards
* for customers with failing credit cards:
  	  * send them an email
	  * inject an in-urbit alert to their ship(s)
* for customers who have failed credit card charges 10 days in a row:
  	  * send them an email
	  * force their urbit to create a checkpoint
	  * store that checkpoint elsewhere
	  * spin down their AWS S3 hosting

The Alerter injects alert messages into a customer's urbit.  It is
driven both autonomously and via the admin-facing web app.  Alerts are
messages displayed in the urbit client software, much like login
messages in Unix.  They are simple text screens, which may contain
URLs and buttons to dismiss them.  Alerts are might be used for things
like:
	* "This is your third login. We hope you're loving urbit. Click <here> to give us feedback or ask us questions."
	* "The Google data breach does not affect you - thanks for using Urbit!"
	* "Due to the west coast Earthquake, ops has relocated your urbit to US/East"
	* "Your credit card has expired. Click <here> to update it."
	* "Your urbit is almost out of space. Click <here> to select a plan with more storage."

The Querier is a tool that lets admins inspect the system.  It is
accessed via the admin-facing web app.  It can deliver statistics such
as:

	* the number of customers 
	* the number of customers who are behind in their payments
	* growth of customers over time
	* billing for current cycle
	* growth of billing over time

### Capabilities Required: Apps store and first two apps

Customers will not want to use Urbit unless their are apps.

Indeed, many of the core benefits touted about Urbit are "you own your
own data" and "your data is all in your OWN computer, not scattered
across unicorns' databases".

We need to demonstrate this with two or more sample apps, and an app store.

The app store is a simple hoon accessed from dojo: +apps.

When run, the app store lists all the Tlon-approved apps, with a status next to each ( installed or not).

A user can use the keyboard arrow keys to navigate his cursor (via ncurses ?) and select a button to install an app.

When he's done he can exit +apps by hitting control-X.  (N.B. There is instracture work here to allow ncurses, and applications of any type).

The +apps app is important because:
	* it demonstrates that Urbit is a platform, not just a grad student demo project that 170 IQ spergs can play with

The first app we support is +contacts.  This is a trivial app that allows one to create n-tuples of {name, company, url, cell, landline, email, snailmail }.

The +contacts app is important because:
	* it demonstrates proof-of-concept of "your data is inside your urbit"
	* it demonstrates (in phase 2) proof-of-concept of "share your data with your moons"
	* it serves as a forcing function for more infrastructure functionality (both of the two above points)
	* the data and the code are both relatively simple, and well suited to what we can already do

The second app we support is +sheets.  This is a non trivial project ...and that's one reason to do it.

### Capabilities Required: Urbit client 

The urbit client is basically a text terminal that is hardwired to do
one thing: ssh into the urbit hosting network, and connect to the
urbit application there and display it to the customer.

The dojo application is augmented with a few capabilities:

* querying someone upstream ( ~zod ? ) for alerts

* inclusion of ncurses or similar to allow for full page refresh, buttons, checkboxes, etc.

* the ability to switch between multiple apps with Control-X (right
  now we cycle between dojo and talk; we should cycle between
  +contacts, +sheets, +dojo, +talk, etc.)

Note that several times above we have mentioned URLs being displayed
inside alerts inside the urbit client.  This does NOT mean that the
urbit client surfs web pages.  If a customer mouses over one of those
urls and clicks it, the registered web browser on his platform (if
any) is invoked.

## Open issues

XXX discuss lost laptop scenario

XXX discuss creating a moon, QR codes, etc.  Impacts web app, planet,
and moon installer.

* associate customer support requests w a star.  There's a 90/10 rule,
  where some people are so difficult that they're not worth having as
  paying customers.  Do we do this in custom code, or do our back end
  admins just use an existing CRM tool?  The latter likely makes sense.

* auto-updating of the urbit client ?

* backups ?

* what is the namespace of the stars we sell?  One particular galaxy, I presume.

* do we need that galaxy's keys online / in AWS S3 in order for the
  galaxy to generate stars?  Yes, I think.  Is this acceptable?

* what credentials do users use to log in to the various websites?  I
  think non-urbit credentials: use email addresses and passwords, like
  the rest of the web.

* where are credentials for an urbit stored?  In the hosted environment, or on their devices?

## Features included in v1.0

Everything above, except +sheets.

## Features not included in v1.0

+sheets is not included in v1.0

## See also

* companion document "A Proposed Urbit Hosting Architecture and Ops Plan using AWS"

  https://github.com/BernardoDeLaPlaz/urbit_docs/blob/master/docs/using/aws_ops_architecture.md

* companion document "A Tutorial to Spin Up Urbit in AWS (with FoundationDB)".

  https://github.com/BernardoDeLaPlaz/urbit_docs/blob/master/docs/using/aws_foundationdb.md