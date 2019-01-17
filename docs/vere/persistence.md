## Metadata
```
  Title: A Modular Persistence Store for Urbit
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2018.11.19
  Last Modified: ~2018.11.19
```

## Overview

A ship receives events, either over the network or via user input.  A
ship must persist these events to allow replay later (for example,
after shutdown and restart) to recover the current state.

What technology should be used to store these events?  Different use
cases have different requirements.  One organization might want to
store events in a certain database approved by IT, another user might
want removable media, a third might care about minimizing
latency for high speed performance, etc.

In the past, in both mainline and branches, we've alternated between a
hand-rolled disk-based log file, a kafka store, and other techniques.
What we really want is a "meta solution": an API that allows multiple
persistence engines to be integrated, and a set of command line flags
that allow a user to specify which one(s) to use.

We also want the ability to use two different persistence stores for
one ship.  This allows us to boot off of store A and write those
events (and all future events) to store B. This means that urbit
itself can serve as the transition tool to move a ship off of one
solution and onto another.

## Specification

### Command Line Arguments

The urbit command line now supports two new flags: '-i' for specifying
input persistence store and '-o' for specifying output persistence
store.  Each takes an argument.  The argument is composed of a
persistence store specifier, and (optionally) a colon separator and
then other information to be passed to the persistence store.  The list
of supported options is shown in the usage screen, but includes things
like 'disk' (legacy disk file), 'sqlt' (sqlite3), 'found'
(FoundationDB), 'lmdb' (LMDB), and 'rock' (RocksDb).

One valid command line might look like

```-o sql  // input defaults to 'disk' ```

Another might look like:

```-i sql:username:password -o found:jb7EK47A:DTu9A6qm@127.0.0.1\:4500```

Another might include no input specifier (defaulting to the 'disk'
choice) and no output specif-er (defaulting to the 'disk' choice).

At the time of this document creation none of the drivers make use of
these optional fields, but the parsing infrastructure is in place (any
such supplied options are passed to the drivers).

### Event Loop

Urbit promises an invariant: no effects will be emitted by a ship that
alter the global network state without the events that triggered those
effects first being logged to the ship's persistent store.

This makes sense: we never want to, e.g., emit a message saying "your
novel has been uploaded and stored", "your bitcoin has been placed in
your wallet", "your stock trade has been processed", etc. without
truly storing the information - what if you crash a millisecond later?
You would have just written a check you can't cash.

The core of the urbit process is the event loop.

Events are received, either from network or from keyboard.  An event
arriving triggers a callback handler from libuv, and that handler
places a data structure (called a 'writ') that represents the event in
a linked list.

The event loop is a thread that works through the linked list of
writs.  Each writ has two state flags, one representing computation of
the event, one representing persistent storage of the event.  They are
orthogonal to each other, because the underlying processes are
orthogonal to each other.

The effective states for each of these two flags are "not yet
submitted", "submitted", and "completed" (although each is actually
implemented as two loobeans: submitted yes/no, completed yes/no, to
avoid even the remotest chance of race conditions that would require
mutexes.  Yes, this creates on impossible state of
complete-but-not-submitted, that will never occur, and thus need not
be worried about).

The event loops tries to push each writ forward.  If a given writ is
not yet submitted for computation, it is submitted for (asynchronous)
computation (and marked thusly).  If a given writ is not yet submitted
for persistence, it is submitted for (asynchronous) persistence (and
marked thusly).  Persistence and computation are done by other threads
(or processes!), and the state flags in each writ get updated behind
the scenes.  When the event loop notices that a given writ has both
computation and persistence completed, it is free to emit effects from
that event, and to delete the write from the linked list.

Note that one principle in the simplification of the event loop is to
remove implicit invariants and expectations and replace them with
explicit ones.  E.g.: the state of a writ is encoded in the two state
flags in the writ data structure, not encoded implicitly for multiple
writs via a monotonically increasing integer in pier.c.  Less elegant,
perhaps, but easier for new blood to look at an understand, either
reading code or driving a debugger.

### Persistence API

There are three function calls that make up the write half of the persistence API:

	write_init()
	write_write()
	write_shut()

Their meanings are obvious and need not be belabored.

There are four function calls that make up the read half of the persistence API:

	read_init()
	read_read()
	read_done()
	read_shut()    

Note that both write_write() and read_read() take an event ID as an
argument.  This is in line with principle expressed earlier of
preferring explicit things over implicit, and of preferring single
responsibilities (the persistence layer does one thing: store
bytes. It has no need to know that events tend to be be read and
written sequentially, and even less of a need to store the 'current'
ID internally).

Note that the read API has one extra function: read_done(), which is
called after each call to read_read().  Why?  The answer is that
almost all database APIs (and here we're referring to the underlying
database, not our persistence layer) do reads by returning to the
caller a pointer to the database's own internal memory, and the
database wishes to free that memory after the caller is done with it.

This presents a design choice: we could either

(1) in the persistence layer allocate new space, copy the data, and
then let the database free its own internal representation

(2) above the persistence layer allocate space ahead of time and ask
the database to write into that, or

(3) in the persistence layer use the data to create a new noun,

(4) define a persistence store API that has two calls (read_read() and
read_done()).  The former returns a handle (cast to an opaque void *)
up to the calling layer.  The latter takes the opaque void *, casts it
back to the appropriate type, and lets the underlying database free
the read memory.

Option one was not chosen because this creates an extra run-time cost,
on every single atom.

Option two was not chosen because most databases do not support an
option to read into a supplied buffer...and because we do not know,
before a read, how big the buffer needs to be.

Option three was not chosen because it is a conflation of concerns.
It should be possible to, e.g., write tests for the persistence layer
that use only simple strings.

Option four was chosen.  The extra effort of calling the read_done()
method is minimal, and only occurs at one place in the code.

### API Function Pointer Implementation

The use of a persistence store chosen at run time is implemented in
pier.c, where we have seven function pointers.  Two different
cascading 'if' statements (one for input, one for output) look at the
command line flags and assign values to the function pointers.

### Persistence Data Structure Implementation

In vere.h we define u3_disk, which holds data relating to the use of a
disk file for persistence, and u3_sqlt, which holds data relating to
the use of an SQLite3 database for persistence, and u3_fond, which
holds data relating to the use of a FoundationDB database for ... you get the idea.

We also define u3_pers, which holds a n pointers to the above
implementation structures, 1 of them pointing to a structure, n-1 of
them null.

A running urbit ship has two pointers of type u3_pers * in u3_pier -
one for input, one for output.  If the same mechanism is used for
both, the pointers can point to the same underlying data.


### Fragmenting

Some underlying databases support arbitrarily large blobs.

Other databases have max record sizes, and some of our larger nouns
can not fit in a single record. (e.g. FoundationDB, which appears -
based on our experiments - to have a maximum record size of 65,536
bytes).

To deal with this, we resort to fragmenting: storing one urbit record
in multiple database records.  This is currently implemented manually
in a single persistence implementation (vere/fond.c), but the
functionality could and should be removed from there to serve as a
general utility that other drives can use.

Fragmenting splits one record across multiple database records upon
write, and reassembles them upon read.

In the most of the current persistence drivers we use the urbit event
ID as they key (so the bytes of event 3 are stored as record 3, for
those databases that use integer keys, or as record "3" for those
databases that use strings).  In persistence drivers that shard the
data, events are stored with a key of the form "<event ID>:<fragment
ID>", e.g. 3:0 or 12:4.

When writing one large noun into a number of fragments, it's easy to
chunk the data by the max record size, write the first fragment with
the key 12:0 (for example), the next with key 12:1, the next with key
12:2, and so forth.

Reading could be relatively simple: we try to read 12:0 (which always
works if event 12 is stored in the database), and then read 12:1
(which either works or does not), and continue on until we find a read
that fails - at which point we can conclude that we have already read
the last fragment.

There's a problem with this tactic, though: it requires n+1 reads from
the database.  If the average number n was on the order of 100, an n+1
tactic would represent only a 1% increase in overhead. ...but the fact
is that the majority of events we write will fit in a single db
record, so the average number n is actually closer to 1.01, and thus
n+1 represents almost a 100% overhead, which is unacceptable.

Instead what we do is include a brief header at the front
of each fragment (and, recall, in a persistence driver that implements
fragmenting, EVERY record is a fragment, even if only a single
fragment is needed for a given event record).

The header format is: <fragment ID>:<total fragment count>:

Both the fragment ID and the total fragment count are represented in
ASN.1 DER format: If the high bit of the first byte is unset, the
lower 7 bits encode the length of the value. If it is set, they encode
the number of bytes that encode the length of the value.  Right now
the code handles the 7 bit case and the 4-byte case, on both read and
write (on read, values 0 through 2^7 - 1 are stored in 1 byte, and
larger values are stored in 5 bytes : 1 byte for length, 4 for data)).
The code is cognizant that larger sizes are possible (64, 128,
etc. bit numbers) and bails on any such fragments with an
"unsupported" error message.

In the case where a write has been fragmented across multiple rows, on
read, we need to reassemble the bytes into one array and hand that
array to the caller.  Note that this is in direct contradiction to our
earlier statements about not memcpy()ing bytes.  Sadly, it has to be
done, at least in multi-fragment cases. 

How do we know how much memory to allocate for the buffer we use to
reassemble the fragments?  Upon reading the initial fragment ( n:0),
we can look at the three byte header, learn how many fragments are in
the total write, and allocate the worst case of max-fragment-size x
number of fragments.  After mallocing the space and memcpy()ing the
data we can free each of the db read buffers.

Note that this creates a situation where fragmenting persistence
drivers can return two types of data: a single read allocated by the
database, or a multi-read buffer allocated by the persistence layer.
We address the two distinct types of cleanup in the read_done()
function, as needed.

### Ugly details of mallocs and memcpys on write

When we are handed a buffer larger than the max fragment size, we must
memcpy() it into multiple smaller buffers, each with its own fragment
header.

...but the TYPICAL case is that a write fits entirely in one fragment,
so an optimization in this case pays dividends.  It would be nice to
avoid memcpy()ing every single buffer, merely to shift it a few bytes
to create room to prepend the fragment header.

We achieve this optimization by calculating the size of the fragment-0
header FIRST, and unpacking the noun via u3r_bytes() into a buffer
that we allocate that is big enough to hold both the fragment header
and the serialized noun.

If the noun fits in one fragment, we have room at the front of the
buffer for the fragment zero header.

If the noun requires multiple fragments, we have to malloc() and
memcpy() all the fragments 1+, but at least fragment 0 doesn't require
a memcpy().


### Ugly details of fragmenting API

From the point of view of pier.c, the API to the persistence layer is
as described above in 'API Function Pointer Implementation'.

Pier.c does not know anything about fragmenting, and it does not want to know anything about fragmenting.

Any individual persistence driver that wishes to use the fragmenting
layer does so internally, thusly:

1. the implementation of the read() API call calls u3_frag_read(), passing
through all the arguments it was called with, and also passing in a
function pointer to the actual persistence engine function that does a
read.

2. the implementation of the read_done() API call calls u3_frag_read_done(), passing
through all the arguments it was called with, and also passing in a
function pointer to the actual persistence engine function that does a
read.

3. the implementation of the write() API call calls u3_frag_writ(),
passing through all the arguments it was called with, and also passing
in a function pointer to the actual persistence engine function that
does a write.

4. the implementation of the write callback routine (which fires when
the database engine says that the write is complete) calls
u3_frag_write_check() to do sanity checks, does its own business, and
then calls u3_frag_write_done().

The fragmenting layer maintains a data structure for each fragmented
write (and a persistent engine that uses the fragmenting layer
fragments EVERY write, even if a given write requires just a single
fragment).  This data structure is used to coordinate between the
multiple parallel fragment writes - only after every single one is
done can we mark a writ as fully written.  A mutex is used internally
to protect the race condition code.

## Threading

We want our event loop to run as quickly as possible (defined as
maximum throughput).  This means that it hands off compute tasks to
the compute engine, and lets the compute engine labor in a separate
process.  This also means that it hands off storage tasks to the
persistence layer, and lets the persistence layer do its work in a
separate thread.

Many underlying databases have APIs that support asynchronous
work flow: the insert() call allows the caller to supply a callback,
and quickly takes the work given, hands it off internally, and then
returns.  At some later point when the persistence is accomplished,
the callback is called.  This model works very well for us: our
persistence driver just marshalls data, supplies a callback that
updates the state flag in the writ, and calls the underlying database.

Other underlying databases, though, have synchronous APIs: the
insert() call chugs and chugs and chugs and returns only when it is
done.  This is not acceptable for us, so in such cases we use the
pthread_create() function to create the asynchronous behavior we want
to see.  A typical use is u3_sqlt_write_write(), which mallocs /
builds a data structure with all the information the true database
write will need (pointers into the writ, mostly), calls
pthread_create() to make a new thread, and then returns.  The new
thread then unpacks the data and calls the underlying database API.

On the topic of threading: note that our persistence API is
asynchronous for writes (the normal case, and the case where speed
matters), and is synchronous for reads (which only occur at startup).
If faster startup is desired, this could be changed.

## Rationale

A persistent store has always been part of Urbit.

Various users have various requirements for what persistent store they
want to use.  Thus, providing a simple modular way that concerns of
urbit (computing events, managing network traffic, etc.) may be
separated from the concerns of a persistence layer (reading and
writing records) makes it easier for (a) write new persistence
engines, (b) write tests for the code we write.

## Integration Plan

Joe and Bernardo de la Plaz will be coordinating to integrate work on
the cc_merge branch ASAP.