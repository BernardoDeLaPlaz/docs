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

Urbit is a utility.  Think about other utilities in your life - and cast the net widely.

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
interface displays a QR code.  Using her phone, she downloads the
urbit client, launches it, and uses it to take a picture of the QR
code displayed on her PC.  The phone urbit client configures itself as a moon.

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

Tlon-administered hardware is available in a variety of speed and storage settings, at different price points.

Tlon-administered hardware is available in a variety of georgraphical regions.

## Capabilities Required: Admin Facing

We need a website with admin screens that:

* show all customers
* show all urbits (grouped by customer)
* allow the creation, deletion, and editing of storage tiers
* allow the creation, deletion, and editing of computation tiers
* ban a customer and terminate their urbits
* update a credit card for a customer
* view submitted applications for the app store
* approve or deny submitted applications for the app store
* create customer facing banners for important time critical things ("the network is down" ; "the Google data breach does not affect you")
* list all admins (via the super-admin account)
* create and delete admins (via the super-admin account)

## Capabilities Required: Customer Facing

We need a website with customer-facing screens

* sales.urbit.org: explains hosting choices, allows for secure checkout / credit card processing 
* help.urbit.org: has a searchable database of problems and solutions, along with a contact-us page
* account.urbit.org: allows users to log in, upgrade or downgrade their CPU and storage tiers

## Capabilities Required: Systems / IT

Tlon needs to implement everything in "A Proposed Urbit Hosting Architecture and Ops Plan using AWS".






## Open issues

* what credentials do users use to log in to the various websites?  I
  think non-urbit credentials: use email addresses and passwords, like
  the rest of the web.


## Future features not included in v1.0



## See also

* companion document "A Proposed Urbit Hosting Architecture and Ops Plan using AWS"
* companion document "A Tutorial to Spin Up Urbit in AWS (with FoundationDB)".

