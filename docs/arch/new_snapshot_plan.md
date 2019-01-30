## Metadata
```
  Title: A Proposal For A New Snapshot Plan
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2018.12.28
  Last Modified: ~2018.12.28
```
## Open Issues / Bugs / TO DO for this document

XXXX there are both image files and patch files?


## Background

A fresh-from-the-factory urbit starts empty and clean and new.

When it is run, it is instantiated as two processes: the king, which
reads the keyboard, and interacts with humans and other urbits over
networks, and the serf, which holds the state and updates it as new
events are received from the king.

Each event an urbit receives changes its state.

The updated state lives in the urbit's memory, but we also store the
events into a database.  This allows for replay: if the urbit crashes
(or merely if we choose to), we can take a new year-zero urbit and
replay the complete history of events into it, and generate an
identical clone.

This is clean and pure and good...but not tremendously efficient.

A better approach is to
* store every events
* periodically store the internal state of the urbit
* optionally purge events that are already captured in the snapshot.

Currently we do this.  In directory ./< shipname >/.urb/chk there are
two files north.bin and south.bin which are copies of the urbit's loom (memory).

What is the format of these files, when are they updated, how are they
updated, and how are they loaded?

An memory of the urbit VM is "the loom".  It's a section of underlying
machine memory managed by the VM, using a custom memory allocatory.
The loom is divided into pages, and used pages grow from each end
towards the center.

There are two checkpoint files, representing the two ends of the loom.
By "representing", we mean "literal byte-for-byte images of the
memory".

At various times (upon boot start in _pier_play(), upon boot
completion in _pier_boot_complete(), and upon shutdown in
u3_pier_exit() ), the king issues a writ to the serf which contains a
command to save the current state.  The serf receives the writ and
calls into file noun/events.c .  Function u3e_save() calls
_ce_patch_compose() which looks at each page of the loom and for each
page with a dirty bit calls _ce_patch_save_page() which calls
_ce_patch_write_page(), which lseek()s in the file and writes the
page.

When the serf is launched its main() calls u3m_boot_new() which calls
u3e_live(), which calls _ce_image_blit(), which copies bytes from
files into memory, populating the loom (and thus restoring exactly the
same state as when the urbit was checkpointed).

## Requirements

This document proposes to replace the current checkpointing system
with a new one, which differs in a few ways:

* pluggable checkpoint storage engines (similar in concept to the FoundationDB /
  LMDB / RocksDB / SQLite3 pluggable persistence engines used to store
  events).  The two initially supported checkpoint engines would be
  disk files (similar to current) and Amazon AWS S3.

* move the functionality of interfacing with the checkpoint storage
  engine from the serf to the king.  (This makes a ton of sense for
  boot, because the king is the sovereign, and instructs his serf /
  compute engine / worker "this is the state that I choose to begin
  with".  The model is less beautiful / the implementation a bit more
  work when we snapshot checkpoints, though, because it requires that
  the king query the serf for changes to state, the serf sends those
  answers "across the wire" (from one process to another), and the
  king to act as a mostly passive conduit between the serf and the
  checkpoint storage engine.

* Ensure that we do not create any impediments or bottlenecks that
  would stop "giant multi-GB, even multi-TB ships".

* checkpoints should no longer be pages of memory image; they should be jammed nouns.

* speed matters, so conversion from memory to blob is highly
  backgroundable — in theory this is kind of what copy-on-write fork
  is for.

## Overview

1. A new command line argument 'C' is supported.  'C' stands for
'checkpoint' and takes an argument specifying the checkpointing
storage system used.  This flag may be supplied on the first or any
subsequent invocation of an urbit.

2. New functionality is added to the serf to maintain knowledge of the
previous checkpoint cor and to generate a noun diff between the
current arco state and the checkpointed state.

3. Two new command are added to the writ/plea protocol between king and
serf: 'init' and 'diff'.  Init supplies a snapshot to a serf and tells
the serf to initialize his arvo state to that snapshot.  Diff asks the
serf for a list of deltas between a previous snapshot and the current
state.

4. Two new functions u3r_diff() and u3r_patch() are written.  The former
diffs two nouns, generating a diff; the applies a patch to a noun and
generates a new noun.

5. A new code module vere/chek.c is created which is linked into the king
which provides an API to store a checkpoint or retrieve one.

6. Two new code modules vere/cham.c and vere/chid.c are created which
implement the checkpoint API, the first via Amazon AWS S3, the second
via local disk files.

## Details

### 1. Command Line

A new command line argument 'C' is supported, and takes an argument.
If it is not supplied, "disk" is assumed.  If it is supplied, it must
either be "disk" or "aws:<accesskey>:<secretkey>".

Amazon AWS S3 requires three fields to connect: an access key, a
secret access key, and a a bucket name.  We do not require the user to
supply the bucketname, as we compute that ourselves using the ship name.

If the user supplies the 'C' flag twice, urbit runs in a special mode
that reads the checkpoint from checkpoint storage specified by the
first invocation of the 'C' flag, writes it to the second invocation
of the 'C' flag, and then exits.  This provides the capability for
users to migrate from one checkpointing method to another.

### 2. Serf last-checkpoint storage

The serf currently has a datastructure of type u3v_arvo, a tuple
that contains ent_d (last event seen) and roc (current core).

This will be modified so that the serf have two such instances.

The first event id / core pair is the current state.

The second event id / core pair is the most recent checkpoint.

### 3. Writ / Plea Protocol

When urbit is launched the king retrieves the previous checkpoint and
pushes it to the serf with writ command 'init', which takes two
arguments: an event id, and a jammed noun.  If the serf does not have
already have arvo state, the response is success.  If the serf does
already have arvo state, the response is an error code (overwriting a
serf at event 1,024 with state at event 99 is a very weird thing to
do, and it should not happen).

From time to time (specified in this document as every 1,000 events,
for purpose of argument), the king sends a writ holding a command
'diff' to the serf. Diff takes 1 argument, the event id that the diff
is against.  If the serf can compute a diff between the serf's current
arvo state and the specified event id, it responds with a jammed noun
that we shall call a "diff list" or a "patch list". More on this below.

### 4. u3r_diff() and u3r_patch()

u3r_diff() takes two nouns as arguments.  The naive implimentation of
the diff algorithm is described first.  A more sophisticated model is
described later.

To diff two trees X and Y we recursively diff the head of X vs the
head of Y, and also the tail of X vs the tail of Y.  The terminating
condition for a recursive diff is where one one input is a cell and
one is an atom, or where both are atoms and have different values.

The diff of two atoms X and Y is the pair [1, Y].

The diff of an atom X and and a cell Y is the pair [1, Y].

The diff of an cell X and and an atom Y is the pair [1, Y].

The union of diff A and no diff is the list [A].  The union of no diff
and diff A is the list [A].  The union of diff A and diff B is the list [A, B].

When the recursive diff is complete we get a list of patches [[n1, Y1] [n2, Y2] ... ].

Diffing tree structures is not a new topic, and it has been researched.

Some references:

	 https://stackoverflow.com/questions/5894879/detect-differences-between-tree-structures

	 https://projecteuclid.org/euclid.pjm/1103043674

Another thought: perhaps we can instrument the arvo state with dirty
bits (and/or keep a list of dirty indices), so that we don't have to
diff the full trees.

u3r_patch() takes a noun and a patch list as arguments.  A new noun is
created by cloning the first noun and replacing each subtree at axis n
with the specified replacement subtree.  If the diff list includes a
modification at axis n, and axis n does not exist in the input noun, a
bail error is triggered.

### 5. Checkpoint API interface

In vere/chek.c four static function pointers are created:

_check_init()
_check_read()
_check_writ()
_check_shut()

Five new exposed functions are created:

u3_check_init() is passed the '-C' command line argument in and takes
care of sanity checking, and assigns implimentation functions into the
three above function pointers.

u3_check_read() returns the most recent event ID and checkpoint noun
pair.  It merely calls the corresponding function pointer.

u3_check_writ() takes an event ID and a checkpoint and writes the pair
to storage.  It merely calls the corresponding function pointer.

u3_check_shut() is a shutdown function.  It merely calls the
corresponding function pointer.

u3_check_transfer() is a quirky function.  It is called in those cases
where two instances of the -C flag have been passed in, it is called
with the second argument value.  It reads the last checkpoint from the
first-specified checkpoint engine, and then writes it via the
second-specified checkpoint engine.

### 6. Checkpoint API implementation

Two new code modules vere/cham.c and vere/chid.c are created.

In both cases checkpoints are stored as a jammed noun consisting of a diff list.

The first checkpoint stored is the degenerate case of diffing a noun
vs an empty tree, which consists of a list of one diff, and that one
diff is a tree located at axis 1.

#### 6.1 Amazon AWS S3

Cham.c provides

  c3_o u3_cham_init(*c3_y);  // pass in value following -C command line flag
  c3_o u3_cham_read(c3_d * event_id, u3_noun * checkpoint) // 2 return arguments
  c3_o u3_cham_writ(c3_d event_id, u3_noun checkpoint) // 2 inputs
  c3_o u3_cham_shut()

which implement checkpoint storage on Amazon AWS S3.

We use the C++ API here

   https://github.com/aws/aws-sdk-cpp

   https://github.com/aws/aws-sdk-cpp/tree/master/aws-cpp-sdk-s3

Amazon AWS S3 requires three fields to connect: an access key, a
secret access key, and a a bucket name.  We compute the bucketname by using the ship name.

( Bucket name specifiers may include lower case letters and dashes, may
not include tilda, and must be between 3 and 63 characters long
inclusive, so everything from 'zod' to
'satnet-rinsyr-silsec-navhut--bacnec-todmeb-harwen-fabtev' fits.

Because every ship has a unique name and thus a unique AWS S3 bucket
name, Tlon Inc. can provide urbit-as-a-service / multi-tenant
operations (re checkpoint storage) without any extra work. )

If the account does not exist, the credentials do not exist, or any
other error occurs, u3_cham_init() fails with a bail error.

Within each bucket snapshots are stored under a name equal to the
highest event ID.  Thus u3_cham_read() opens the bucket, scans all
checkpoint names, and picks the highest.

A maximum of 10 checkpoint files are retained.

#### 6.2 Disk 

Chid.c provides

  c3_o u3_chid_init(*c3_y);  // pass in value following -C command line flag
  c3_o u3_chid_read(c3_d * event_id, u3_noun * checkpoint) // 2 return arguments
  c3_o u3_chid_writ(c3_d event_id, u3_noun checkpoint) // 2 inputs
  c3_o u3_chid_shut()

which implement checkpoint storage on local disk.

Checkpoint files are stored in ./<ship>/.urb/chk and are named chek.1, chek.2, etc.

The new checkpoint file is written first as check.0 and then all
checkpoint files are renamed with rename(2), one at a time.

A maximum of 10 checkpoint files are retained.

### Threading

* diffs and patches can take place slowly on another thread. Think about how we want to leverage this, if at all.


## Open Issues

* How many checkpoints do we want to store?

* 