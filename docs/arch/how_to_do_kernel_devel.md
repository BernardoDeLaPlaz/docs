## Metadata
```
  Title: How to do arvo kernel and urbit development 
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2019.03.21
  Last Modified: ~2019.03.25
```

## Overview

Arvo development consists of editing hoon files in the arvo codebase.

      https://github.com/urbit/arvo

Urbit development consists of editing C files in the urbit codebase.

     https://github.com/urbit/urbit

The stickiest wicket is making changes in both codebases simultaneously.

This document has hints and techniques for all three cases.

## Urbit development

### Coding Style

The preferred coding style is not very well documented.

You can find information on the psuedo-"Hungarian" style of variable names here:

    https://github.com/urbit/urbit/blob/master/Spec/u3.md

In short:
    * never use standard C types such as 'int32_t' ; always use Urbit C types such as 'c3_w'.
    * always name your variables so that they are suffixed with the same underscore-and-character as their type, e.g. "c3_w opt_w"
    * [ deprecated ] the variable name before the prefix should be three characters long and have one vowel

Other coding conventions:
    * no tabs, ever

### Buidling urbit with debugging symbols

In the urbit directory edit
   /scripts/build
and replace
   meson . ./build --buildtype=release "$@"
with
   meson . ./build --buildtype=debug "$@"

Then "rm -rf ./build ; scripts/build".

### Using gdb

Create a .gdbinit file in your home directory, and put this

    handle SIGSEGV nostop noprint

in it.  Why?  Because urbit impliments its own memory manager via use
of libsegv, and intentionally creates and captures SIGSEGV events.
Without this line in your .gdbinit you will be dropped to the debugger
every single time a memory reference is made.


Also put

    file build/urbit

in the file, so that gdb knows what executable to use for reference
when you attach to a running process.

Using gdb can be challenging with the urbit executable, because urbit
grabs control of the terminal, and thus interacts poorly with either
gdb alone, or gdb being run inside emacs in 'gud' mode.  One thing
that has worked very well for me is instrumenting vere/main with this
bit of code:

  #if 1
      int ii = 0;
      fprintf(stderr, "about to sleep for gdb in main.c: - PID = %i\n\r", getpid());
      while (ii != 1){
        fprintf(stderr, "...\n\r");
        sleep(1);
      }
      fprintf(stderr, "post sleep\n");
  #else 
      fprintf(stderr, "no attach\n");    
  #endif

N.B. : DO NOT check this code change in!

Run gdb either in a terminal, or in emacs (if in emacs, do this by
    M-x gdb
    gdb -i=mi
).

Run urbit in a terminal.

Urbit will print out

    about to sleep for gdb in main.c: - PID = 573

or whatever.

In gdb type
   attach <pid-from-above>

If the above line gives you an error like

    Could not attach to process.  If your uid matches the uid of the
	target process, check the setting of
	/proc/sys/kernel/yama/ptrace_scope, or try again as the root user.
	For more details, see /etc/sysctl.d/10-ptrace.conf ptrace:
	Operation not permitted.

then you need to go to the shell and type

    echo "0" | sudo tee  /proc/sys/kernel/yama/ptrace_scope 


Why?  To quote from the man page of ptrace(2):

       On systems with the Yama Linux Security Module installed, the
       /proc/sys/kernel/yama/ptrace_scope (available since Linux 3.4)
       can be used to restrict the ability to trace a process with
       ptrace(2) (and thus also the ability to use tools such as
       strace(1) and gdb(1)).  The goal of such restric‐ tions is to
       prevent attack escalation whereby a compromised process can
       ptrace-attach to other sen‐ sitive processes (e.g., a GPG agent
       or an SSH ses‐ sion) owned by the user in order to gain
       additional credentials and thus expand the scope of the attack.
       A process with the CAP_SYS_PTRACE capability can update this
       file with one of the following values:
	   		0 ("classic ptrace permissions")
			1 ("restricted ptrace")
			2 ("admin-only attach")


Note that many systems reset /proc/sys/kernel/yama/ptrace_scope after
each reboot, so you may end up doing this again after your next reboot.


Try the "attach" command from inside gdb again.

Your debugger will immediately attach to urbit.

Type 'bt' to get a backtrace.

Type 'f 2' to move up two stack frames to the main() function call.

Type 'set ii=1' to set the variable to allow you to exit the infinite loop.

Type 'p ii' to verify that the setting stuck.

Type 'c' to continue.

You are now running urbit under debugger control.


### Overview of noun memory leaks

You are familiar with nouns from reading about Hoon.

Urbit has its own memory manager for nouns.

A Hoon program creates nouns, and they are automatically cleared up at
an appropriate time.  Code written in Hoon never has to worry about
memory leaks, because the interpretted is assumed to be perfect at
doings its job of managing memory.

The C programmer's job is to make this invariant true.

A small portion of the C code is deeply involved in noun creation and
processing, but most of the rest of the C code just interacts with
nouns in two ways:

1) recieves them as inputs (likely as effects).  See term.c,
_term_ef_blit() as an example, where a noun that has been generated by
the hoon computation engine is passed to the terminal code, so that
the terminal can do something grubby-and-base-and-meatspace with this
pure mathematical thing...like displaying it on the screen.

2) creates them and injects them into Hoon processing as events, using
u3v_plan() or u3v_plow().  See term.c, u3_term_ef_winc() as an
example.  Whenever the Unix window size is modified a signal handler
is fired, u3_term_ef_winc() is called, and the code constructs a noun
and injects it into Hoon processing using u3v_plan).

When we create a noun using tools like u3nc(), u3nt(), u3nq(), etc. we
create one reference count to them in urbit's garbage collector.

Before we can let that noun go out of scope, we need to make sure that
either (a) we have handed ownership of that noun to someone else, or
(b) we have manually decremented the reference count via u3z().

For more detail on this, including 'transfer' and 'retain' semantics,
please read

    https://github.com/urbit/urbit/blob/master/Spec/u3.md

It is good style to comment each C function that accepts a noun as an argument with either

   // transfer

or

   // retain

so that the caller knows if your code will:

* take ownership of the noun (i.e. take responsibility for eventually
  calling u3z() on it), and therefore the caller can pass the noun in
  and wipe his hands of it, or

* NOT take ownership of the noun (i.e. NOT take responsibility for
  eventually calling u3z() on it), and therefore the caller himself
  must call u3z() on the noun.

### Testing for memory leaks and other problems

The urbit executable has an optional built-in memory checking tool.

To turn on this memory checker, go the command line and type

    meson configure -Dgc=true build

Then verify it by typing

    meson configure build

expect to see

    Option             Current Value Possible Values Description                                                  
    ------             ------------- --------------- -----------                                                  
    ...
    gc                 true          [true, false]   Add debugging information to heap. Run with -g. Breaks image.


Then rebuild

    scripts/build

Finally: run urbit, interact with it, and then exit urbit with control-d (this is important- use control-d and not control-z).

Upon success you will just see the shell prompt.

Upon failure the run will terminate in a crash that looks something like this

    weak: 0x3908bbbc mug=0 (marked=0 swept=1)
        code: %term
        size: B/32
        data: [80 54]
    leaked: B/32
    Assertion '0' failed in ../noun/allocate.c:1916

    bail: oops
    bailing out
    urbit: ../noun/manage.c:650: u3m_bail: Assertion `0' failed.
    Aborted (core dumped)

In this error report we can see a few things:

The first line tells us what kind of memory error we have.  In this case it is a "weak" error.  There are three types:
  - "leak" means actual uses are 0, but reference count is > 0

  - "dank" means actual uses are > 0, but reference count is 0 (ie
    "reverse-leak": we're still using freed memory; corruption is
    likely to occur).

  - "weak" means actual uses are > 0, but actual uses != reference
    count (i.e. this will be a leak or a dank at some point; check
    mark/sweep numbers to see which).

The second (code: ) line tells us what file of the urbit source code is responsible.

The third line (size: ) tells us the size of the memory leaked.

The third like (data:) tells us the contents of the noun that was leaked.

### Debugging memory leaks and other problems

To debug a memory leak, first look at the "code:" line.  View the relevant code file.

Next look at the "size:" and "data:" lines, and try to figure out where in the code such a noun is being created.

Set breakpoints.

Run the executable up to the break point.  Single step until the application crashes with a memory leak message.

The above memory leak was created by (intentionally) editing some code in term.c so as to create something like this

    void func()
    {
         u3_noun u3nc(80, 54);
    }


This variant of the code does not leak memory:


    void func()
    {
         u3_noun win = u3nc(80, 54);
         u3z(win);
    }

because the u3z() tells the memory manager that the reference count can be decremented.

Another technique that can be useful in hunting memory leaks is to instrument file
    vortex.c
function    
    _cv_nock_poke()
with something like 
   u3m_p("DEBUGGING", ovo);

and then recompile and run.  This will result in the noun pretty
printer function u3m_p() firing every time a noun is inserted into the
system.

### Running tests

When your code is checked in and pushed to github, it will
automatically run tests via an integration with Travis-CI (see
http://travis-ci.com/user/tutorial ).

Run the tests yourself, first, to make sure that they pass.

In file travis.yml there is a block of code like this:

    script:
      - meson . ./build --buildtype=debugoptimized -Dgc=true -Dprof=true
      - cd ./build
      - ninja
      - ninja test
      - sudo ninja install
      - cd ../.travis
      - npm install
      - ulimit -c unlimited -S
      - npm run -s test; bash print-core-backtrace.sh $?

You can run these steps by hand, in a shell (and, note, in a shell,
not in an emacs shell window ... because the final steps will run
urbit, and urbit does not play nicely in a shell window).

To dig into the topic a bit deeper, the travis.yml file specifies that
the tests are written in node_js. The key lines in the file are

	cd ./travis

and

	npm -s test

The second line tells the npm node_js tool to run (in silent mode) the file

	./test.js

which uses the tools located in

	./travis/node_modules/urbit-runner

most specifically

    ./travis/node_modules/urbit-runner/lib/actions.js

and

    ./travis/node_modules/urbit-runner/lib/runner.js

to execute the tests in

   ./test.js

### Writing new tests

XXX material needed here

## Arvo development

### Building and using a pill

Arvo development consists of modifying the hoon files that make up the arvo kernel.

When an urbit boots up, it will load an arvo from one of three places:

1) if -A is specified on the command line, it will load hoon files from the specified directory

2) if -B is specified on the command line, it will load a compiled pill from a specified file

3) if neither of those is specified, it will load a compiled pill from https://bootstrap.urbit.org


Note that pills can come in a few different varieties: solid, brass, etc.

Brass is, apparently, the way of the future, but these instructions
describe a solid pill, because that's how I'm doing things today /
what has been proven to work for me.

A typical development process looks like this

  # first create a pill
  #
  
  git clone https://github.com/urbit/urbit.git
  cd urbit
  scripts/bootstrap
  scripts/build
  ./build/urbit zod
  .pill +solid
  CTRL-d :: exit
  cp zod/.urb/put.pill ../my_new_solid.pill

  # now boot off of it
  #

  rm -rf zod
  ./build/urbit -g -F zod -w zod -B ../my_new_solid.pill


  # now, transition to your own modified hoon code
  #

  |mount %

The mount operation creates a bunch of directories and files

You may now edit the code in ./zod/home/sys (more on this below)


Q: why do we create a new pill at this step, instead of just using the one that urbit will fetch itself?
A: I don't know.




### How to edit hoon files


Once you do |mount % in dojo, you get a clone of the arvo directories
(specifically, the current urbit standard, NOT a clone of what you've
got one directory over) in your pier (e.g. ./urbit/zod/home/sys).

Any time you make a chance to one of these files and save it, clay
(the urbit filesystem) will sync and the hoon will be recompiled.

If you touch a top level file like hoon.hoon or zuse.hoon, in dojo you
will see all of the modules being recompiled, e.g.

    [%tang /~zod/home/~2019.3.21..20.33.04..a73b/sys/zuse ~wichex-fopnes]
    [%vane %a /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/ames ~dibtuc-hatlur]
    [%vane %b /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/behn ~dabted-dozfed]
    [%vane %c /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/clay ~mognex-loslyr]
    [%vane %d /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/dill ~padper-happur]
    [%vane %e /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/eyre ~dibwes-dondys]
    [%vane %f /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/ford ~minlyn-dibpex]
    [%vane %g /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/gall ~libled-finryl]
    [%vane %j /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/jael ~bilnub-halweb]

However if you edit a "leaf" level file, e.g. sys/vanes/dill.hoon, you'll see a smaller recompile, e.g.

    [%vane %d /~zod/home/~2019.3.21..20.33.04..a73b/sys/vane/dill ~padper-happur]

Note that the single letter signifier on each line (e.g. %a, %b,
%c...) corresponds to the vane name (ames.hoon, behn.hoon, clay.hoon
...).

If you have an error in your code you'll get a stack trace instead of
the clean compiler message.  If I introduce an error into dill.hoon by changing

          :*  [hen %pass /heft/ames %a %wegh ~]

to

          :*  [hen %pass /heft/ames %a %wegh ~ ~]

(i.e. the addition of an extra ~), I get a compiling message, and then an error in dojo:

    [%vane %d /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill ~tonfyl-risryd]
    nest-fail
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[255 9].[266 11]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[143 7].[455 9]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[142 7].[455 9]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[140 5].[463 7]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[140 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[139 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[138 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[32 5].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[9 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[6 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[5 1].[554 3]>
    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[4 1].[554 3]>

The compiling message we've seen above.

The next line is the error:
    nest-fail

And after that a stack trace follows.

The first line specifies the innermost error

    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[255 9].[266 11]>

i.e. "in file sys/vane/dill, between line 255 (character 9) and line
266 (character 11), there is a type error."

Following this is another line

    /~zod/home/~2019.3.21..21.16.52..383d/sys/vane/dill::[143 7].[455 9]>

which says that the above expression is nested within "line 143, character 7, and line 455, character 9".

And so forth.


### Where to edit hoon files

We assume that you have two git checkouts, side by side, in the same directory:
* arvo
* urbit

You have two choices of where to edit hoon files:

1) in the ship in your urbit ( e.g. ./urbit/zod/home/sys/zuse.hoon) - this is the one we walked through in the previous section.
2) in your checkout of arvo (e.g. ./arvo/sys/zuse.hoon )

The second choice has some advantages:

* you can save the file as often as you want, without causing urbit to (a) do a clay sync, (b) auto-load the hoon and recompile it
* you are working in your git directory, so it's easy to commit your files, cherry pick changes, etc.

It also has a disadvantage:

* you need to do an extra manual step to get your hoon files compiled
  and loaded by urbit (specifically, cp-ing them into the ship's mounted directory ( e.g. ./urbit/zod/home/sys/zuse.hoon)

You might think that there is a third option:

3) you can cd into your pier and set up a symbolic link with `ln -s`

which would give you the best of both worlds.  Unfortunately this
doesn't work, at least on many computers.  Apparently when you save
the file in arvo, clay does not get reliably updated that the file has
changed, and so no syncing happens.


All in all, I recommend that you choose option #2:

* make your edits in your arvo checkout
* cp them over to your ship home when you want to fire for effect

Now matter which you choose, please read the following section on autoload.

There is a refinement of #2, let's call it "2a", which at least on developer uses:

* edit in the arvo git directory
* have "watch cp -r ~/arvo ~/urbit/pier/home" running in a terminal somewhere
* have autoload off usually.

### Some notes on recompiling hoon ( autoload, reset, and reload )

Dojo has a toggleable loobean setting called "autoload".

By default, autoload is turned on ( %.y ).

When autoload is on, any change to a system file (a file in ./zod/home/sys ) immediately triggers a recompilation.

This can be good, if you want to copy over a single file and then immediately see if it compiles.

...but this can be bad, if you want to copy over six different files,
and don't want to recompile five times (and having compile errors each
time) and then once more after the last file is copied over.

When autoload is off, you have to explicitly trigger recompilation of the new Hoon files.

Turn autload off or on with

  |autoload %.n

and 

  |autoload %.y

As a shortcut 

  |autoload

toggles autoload to the other setting, no matter what it starts as, and prints out the new state.

If autload is turned off, you can trigger recompilation of all of the Hoon system files with

   |reset

There are some reports that this does not always work...so pay attention to the results you get.

Another technique is to reload / recompile just one system file, using

   |reload %a

or
   |reload %b

etc

The argument %<single-letter> corresponds to the specific vane that
you want to recompile ('a' for ames, 'b' for behn, etc.).

### Strategies for editting hoon files / thoughts about developing vanes

You will be editing one of these files:

    ./sys/hoon.hoon        :: language definition
    ./sys/zuse.hoon        :: types for the arvo OS
    ./sys/arvo.hoon        :: the core of the arvo OS

    ./sys/vane/ames.hoon   :: networking
    ./sys/vane/behn.hoon   :: timers
    ./sys/vane/clay.hoon   :: filesystem
    ./sys/vane/dill.hoon   :: terminal handling
    ./sys/vane/eyre.hoon   :: http server
    ./sys/vane/ford.hoon   :: build system
    ./sys/vane/gall.hoon   :: agent execution
    ./sys/vane/jael.hoon   :: keys and secrets
    ./sys/vane/xmas.hoon   :: ??

Changes to any of the first three files will result in recompilation
of everything (if autoload is turned on) or require manual
recompilation of everything by the developer (see above section) if
autoload is turned off.

There is a standard API for vanes.  Every vane has a few arms all with
certain specific names, which get called in certain situations.  This
is documented XXX.

One of the first things you'll want to do is verify that your new file
is getting loaded.

Look for the ++  load arm, which is called every time the vane is reloaded.  E.g.

    ++  load
      |=  old-state=*
      +>.$(state *axle)


Modify this to something like

    ++  load
	  &~  "RELOADING - version 25 Dec 2020"  :: NOTFORCHECKIN
      |=  old-state=*
      +>.$(state *axle)


and reload the file (either through autoload or manually).  Expect to
see this load arm sigpam generating output in dojo.

The load arm is interesting in that it feeds the current state of the urbit into the vane.

As part of your changes you might change some of the data that urbit /
the vane stores.  If so, you might want to add code here to perform a
test to see if the state is already in an acceptable format, and - if
not - convert it to the new format.

XXX this is something I don't understand super-well ; revisit in 3 months when I do!


## Combined Urbit / Arvo development.

Combined Urbit / Arvo development is necessary when you want to, for
example, create a new effect type in the hoon world, and then create
new handling of that effect in the C world.


### On Breaches

One thing to ponder early in the process - because it will affect your
development strategy - is how your feature will be deployed to the
masses.

There are two choices:

* breaching
* non-breaching

A breaching change is a set of changes in C and Hoon code that must be deployed simultaneously.

An example is changing dill.hoon to replace the old "blit" effects
that are sent to the C vere/term.c code with distinct and new ones.
If one deployed either the C code alone, or the Hoon code alone, one
would have a broken system.

A non-breaching change is one where the Hoon and C can be updated
independently.

Breaching changes are a hassle.  Tlon prefers to breach Urbit as
infrequently as possible, because it creates work for users.

At some point, breaches will be forbidden.

So try for non-breaching changes.

On strategy to achieve this is to C code that operates with both the
old Hoon and with the new Hoon that you're about to create.

In the vere/term.c example, this might mean supporting the old "API"
of blits ( "mor", "bel", "lin" ) and then creating a new set of blits
("mor2", "bel2", "lin2"), and changing the hoon to generate the new
blits.

XXX more on migration strategy here.