# How to write a jet

A guide for new Urbit developers.

Problem statement: coder A has written some hoon, and you - coder B - need to jet it.

Preconditions:

* You have a pill `jetted.pill`, located in ~/tlon.
* You have an example of hoon code that, executed in dojo, does the right thing.

## Get all the source code you need.

The source code you will need to modify lives in two different repos:
urbit and arvo.  The urbit repo has the source code for the
intrepretter / personal server, written in C.  The arvo repo has the
operating system, written in hoon.  Since you're jetting the hoon, you
need to modify both.

For simplicitly, I'm going to assume that you are doing all of your
development in ~/tlon.  This saves the bother of writing nonsense like
the/path/to/your/sandbox, and also means that one can copy-and-paste
these instructions straight into a terminal with zero editing.  Do
your development elsewhere, if you like, and make a symbolic link.

1. mkdir ~/tlon
1. cd ~/tlon
1. git clone https://github.com/urbit/urbit
1. cd urbit
1. git fetch
1. git checkout <branch> # optional
1. scripts/bootstrap
1. 
1. cd ..
1. git clone https://github.com/urbit/arvo
1. cd arvo
1. git fetch
1. git checkout <branch> # optional
1. cd ..

## Add new subprojects / libraries to urbit, if necessary

If we have a complicated hoon function that, say, computes square
roots, we need to speed it up, and there is an open source library
libsquareroots, adding libsquareroots to our build might solve 90% of
our problems.

If this describes your task, read this section.  If not, skip to the next section - none of this is necessary.

1. have someone with github.com/urbit repo privileges fork libsquareroots into github.com/urbit
1. fork this new repo github.com/urbit/libsquareroots
1. cd ~tlon/urbit/subprojects
1. git submodule add <url>
1. edit ~tlon/urbit/subprojects/meson.build
    1. add a line `<foo>_dep = dependency('<foo>', version: '>=0.1.0', fallback: ['<foo>', '<foo>_dep'])`
    1. add `<foo>_dep` to the list of dependencies at the bottom of the file. Remember to use a comma.
1. generate a meson.build for the new subproject: cd subprojects/<new directory>; meson wrap promote subprojects/secp256k1
1. edit the meson.build until it works ("works" is defined as "in the top level ~/tlon/urbit invoking scripts/build builds the library and links it into the urbit executable")
1. git add meson.build ; git commit -m"now builds with meson for Tlon" .
1. cd .. ; git add .gitmodules ; git commit -m"added submodule" .
1. test the whole thing by
    1. cd /tmp
    1. rm -rf urbit
    1. git clone https://github.com/<yourname>/urbit
    1. cd urbit
    1. scripts/bootstrap
    1. verify that the new submodule is present
    1. scripts/build
    1. verify that the urbit executable is built and linked to the new library
1. submit a pull request from your copy of <libsquareroots> to the urbit copy
1. submit a pull request from your copy of urbit to the urbit copy

### If the submodule is modified

The steps above add a submodule to the urbit repo.  Or, more specifically, they add a specific version of the submodule to the repo.

If the submodule has commits made to it after that point it time, the urbit repo will not automatically track those new commits.

To make urbit use an updated version of the submodule:

1. change to the submodule directory: cd submodule_dir
1. checkout desired branch: git checkout master
1. update: git pull
1. get back to your project root : cd ..
1. now the submodules are in the state you want, so: git commit -am "Pulled down update to submodule_dir"



## Edit the source code

There are several things you're going to do:

1. hint the hoon code, so that the urbit interpreter will know to look for C code
1. for each jet you want you're going to declare two functions in two .h files (one takes the arguments as separate arguments, one takes them as one big tree)
1. then implement the two C functions in a single .c file
1. register the new C jet functions so the urbit interpretter knows the names / function pointers of the C funcs


### Edit the Hoon Source code

Let's edit the hoon source code first.

There are multiple runes to do hinting, but we're going to concentrate
on three.  All hinting runes start with a tilda ("sig").  A mnemonic
is "we're signalling some information to the interpretter".

1. ~/  sigfas: simple jet registration, using defaults for some arguments
1. ~%  sigcen: more complicated jet registration, specifying all arguments
1. ~&  sigpam: debugging printf

Feel free to use ~& liberally! (Just remove them when you're done).

Let's use the three to jet some code.  Starting with this example:
```
  ++  aaa 
    ...
    ++  bbb
      ++  ccc
        |=  dat=@
        ^-  pont
        =+  x=(end 3 w a)
        =+  y=:(add (pow x 3) (mul a x) b)
        =+  s=(rsh 3 32 dat)
        :-  x
        ?:  =(0x2 s)  y
        ?:  =(0x3 s)  y
        ~|  [`@ux`s `@ux`dat]
        !!
```		

our steps are:

1.  add a line under `++  aaa` like this: `~%  %aaa  ..is  ~`
1.  add a line under `++  bbb` like this: `~/  %bbb`
1.  add a line under `++  ccc` like this: `~/  %ccc`

What we're doing here is hinting `ccc`, and then adding a string of
hints up the enclosing tree of arms.  Each of these hints is
implemented w the simple / minimalist `~/` rune, which takes one
argument: the symbol that should be used to label the hint (these
symbols will be used later in the C code to match up C code with hoon code).

The top hint (for `aaa`) uses the more verbose `~%` hint syntax, which
specifies two additional fields: `...is` means "parent jet can not be
determined by context, so use 'is' as the parent" ("is" is the name of a system-supplied jet).

[ N.B. saying " `is` is the parent " is a quick-and-dirty lie.  The
fuller version is: `+is` is an arm of the Arvo core, so `..is` is a
reference to the entire Arvo core. The whole Arvo core is hinted with
the jet label `%hex`, which is used as the parent for all the
top-level jet hints in %zuse. ]

When hinting your own code, make sure to hint each nesting arm.
Skipping any one will result in the jet code not being run.

Note that you do not need to provide C implementations for everything
you hint.  In this above example we hint aaa, bbb, and ccc - even if
our intent is only to jet ccc.

### Edit the C Source code (overview)

Having hinted our hoon, we now need to write the matching C code.  If
we don't, there isn't a problem - hinting code merely tells the
interpretter to look for a jet, but if a jet is not found, the hoon
still runs just fine.

There are two distinct tasks to be done in C:
1. write the jet
1. register the jet

For each jet you will write one u3we() function and one u3qe() function.

### Edit the C Source code to add registration

* edit the header file 
            include/jets/w.h
       to have a declaration for each of your u3we() functions.  Every u3we()  function looks the same, e.g.
	       u3_noun u3we_xxx(u3_noun);

* edit the header file
          ~/tlon/urbit/include/jets/q.h

   to have a declaration for your u3qe() function.  u3qe() functions
   can differ from each other, taking distinct numbers of u3_nouns and/or u3_atoms, e.g

          u3_noun u3qe_yyy(u3_atom, u3_atom);
          u3_noun u3qe_zzz(u3_noun, u3_noun, u3_atom, u3_atom);

* create a new .c file to hold your jets (both the u3we_() and u3qe_() functions go in the same file).  For example, ~/tlon/urbit/jets/e/secp.c
   The new file should have three things:
   1.   \#include "all.h"
   1.   the new u3we() function
   1.   the new u3qe() function

* edit ~/tlon/urbit/jets/tree.c to register jet

In the hoon code we hinted some leaf node functions (`ccc` in our example) and then hinted each parent node up to the root `aaa`).

We need to replicate this structure in C.  Here's example C code to jet our above example Hoon:

```
			// 1: register a C func u3we_ccc()
			static u3j_harm _143_hex_hobo_reco_d[] =
			  {
				{".2", u3we_ccc, c3y},
				{}
			  };  

			// 2: that implements a jet for hoon arm 'ccc'
			static u3j_core _143_hex_hobo_bbb_d[] =
			  {
				{ "ccc",            _143_hex_hobo_ccc_d },                         
				{}
			  };

			// 3: ... that is inside a hoon arm 'bbb'
			static u3j_core _143_hex_hobo_hobo_d[] =
			  {
				{ "bbb", 0,   _143_hex_hobo_bbb_d },                               
				{}
			  };

			// 4: ... that is inside a hoon arm 'aaa'
			static u3j_core _143_hex_d[] =
			{ { "down", 0, _143_hex_down_d }, 
			  { "lore", _143_hex_lore_a },
			  { "loss", _143_hex_loss_a },
			  { "lune", _143_hex_lune_a },
			  { "coed", 0, _143_hex_coed_d },
			  { "aes", 0, _143_hex_aes_d },
			  { "hmac", 0, _143_hex_hmac_d },
			  { "aaa", 0, _143_hex_hobo_d },                                       
			  {}
			};

```

There are 4 things steps here.  Let's look at each in turn.

Section 1 names the C function that we want to invoke: 'u3we_ccc()'.
The precise manner in which it does this is by putting entries in an
array of 'u3j_harm's.  The first one specifies the jet; the second one
is empty and serves as a termination to the array, similar to how a C
string is null terminated with a zero.  The jet registration supplies
two fields {".2", u3we_secp}, but this does not initialize all of the
fields of u3j_harm.  Other fields can be specified.

The first field, with value ".2" in this example, is "arm 2".

[ N.B. ".2" labels the axis of the arm in the core. With a %fast hint
(~/), we're hinting a gate, so the relevant arm formula is always just
the entire battery at +2. I assume the jet dashboard is flexible
enough to hint arms at different axes, but I've never seen it done. ]

The second field, with value u3we_secp in this example, is a function
pointer (to the C implementation of the jet).

The third field (absent here) is a flag to turn on verification of C
jet vs hoon at run time.  It can take value c3n (which means verify at run time) or c3y
(which means don't verify).  If not present, it is set to don't verify.

[N.B. The use of the flags seems inverted. Why is c3y ("yes") used to
turn OFF verification?  A: because the flag is actually "is the jet
already known to be perfect?" ]

There are additional flags; see ~/tlon/urbit/include/noun/jets.h


Section 2 associated the previous jet registration with the name
"ccc".  Note that this is the symbol used in the Hoon hint!  Note also
that we again have a "null terminated" (metaphorically) list, ending with `{}`.

Section 3 references structure built in step 2 and slots it under
`bbb` (again, note that this is exactly the same symbol used in the
hinting in Hoon).

The line in section 2
```	
      { "ccc",            _143_hex_hobo_ccc_a },                         
```

looks very similar to the line in section 3
```
	  { "bbb", 0,   _143_hex_hobo_bbb_d },
```

But note that the line in section 2 fill in the first 2 fields in the
struct, and the line in section 3 fills in the first three fields.
Section 2 is registering an array of u3j_harm, i.e. is registering an actual C
jet.

Section 3 is specifying 0 for the array of u3j_harm and is instead
specifying an array of u3j_core, i.e. it is registering nesting of
another core which is not a leaf node.

Section 4 is much like section 3, but it's the root of this particular
tree.  Section 4 is also an example of how a given node in the
jet-registration tree may have multiple children.

Using this example you should be able to register jets whether your
nesting is 2 layers deep, 3 (like this example), or more.  You should
also be able to register multiple jets at the same nesting level (e.g.
a function u3we_ddd() which is a sibling of  u3we_ccc() inside data structure _143_hex_hobo_reco_d[] ).


### Edit the C source code to add the u3we_() function

There are two C functions per jet, because separation of concerns is a good thing.

The first C function - named u3we_something() - unpacks arguments from
the Hoon code and gets them ready.

The second C function - named u3qe_something() - takes those arguments
and actually performs the operations that parallel the hoon code being
jetted.

Let's write the u3we_() function.

Your function is going to accept one argument, of type u3_noun.  This
is the same type as a hoon noun ('*').

This one argument is the payload.

The payload is a tree, obviously.

The payload consists of (on the right branch) the context (you'd think
of "global variables and available methods", if analogies to other
programming languages were allowed!) and on the left branch the
sample (the arguments to this particular function call).

Your u3we_() function does one thing: unpack the sample from 'cor',
sanity check them, and pass them in to the u3qe_() function.

To unpack the sample

We use the function u3r_mean() to do this, thusly:

      u3_noun arg_a, arg_b, arg_c  ... ;

      u3r_mean(cor,
	           axis_a, & arg_a,
  	           axis_b, & arg_b,
  	           axis_c, & arg_c
			   ...
			   0)


If we want to to assign the data located at axis 3 of cor to arg_a, we'd set axis_a = 3.

u3r_mean() takes varargs, so we can pass in as many axis /
return-argument pairs as we wish, terminated with a 0.

When pulling arguments out of a hoon remember that Nock and Hoon store
lists as skinny right-descending trees (because a linked list is a
degenerate case of a tree).

The quick way to figure out axis locations is to remember that we
start with axis 1, and left children have axis 2n and right children
have axis 2n+1.

If one argument is passed in, it is found at axis 1.

If we have two arguments, axis 1 holds pointers to the first argument
in the left child at axis 2 (when n is 1, 2n = 2 ) and the second
arguments in the right child at axis 3 (when n is 1, 2n+1 is 3).

If we have three arguments, axis 1 holds pointers to the first
argument in the left child at axis 2 (when n is 1, 2n = 2 ) and to the
next cell at 3, which doesn't contain an argument, but points to
argument 2 in the left child at axis 6 (when n is 3, 2n = 6) and to
argument 3 in the right child at axis 7 (when n is 3, 2n + 1 = 7).

Note that there are #defines to help you specify the axises you want.
They are in include/noun/xtract.h

If the sample has one item, that one item it is located at the axis
specified by the #define u3x_sam_1.

If the sample has 2 items, they are located at the axis specified by
the #defines u3x_sam_2 and u3x_sam_3.

If the sample has 3 items, they are located at the axis specified by
the #defines u3x_sam_2, u3x_sam_6, and u3x_sam_7.

etc.

If you have four arguments and want to tease them out and verify them, you could use code like this

N.B. in our fprintf()s we're using both \n and \r to achieve line feed
(move cursor down one line) and carriage return (move it to the left),
because Urbit uses ncurses to control the screen and it changes the
default behavior you may be used to (where \n accomplishes both). 
```
		   c3_o ret;
		   u3_noun sample;
		   ret = u3r_mean(cor, u3x_sam_1,  &sample, 0);            
		   fprintf(stderr, "ret = %i\n\r", ret); // we want ret = 0 = yes 
		   u3m_p("sample", sample);			// pretty print the entire sample

		   ret = u3r_mean(cor, u3x_sam_2,  &sample, 0);            
		   fprintf(stderr, "ret = %i\n\r", ret);
		   u3m_p("2: ", sample);		

		   ret = u3r_mean(cor, u3x_sam_6,  &sample, 0);            
		   fprintf(stderr, "ret = %i\n\r", ret);
		   u3m_p("6: ", sample);		

		   ret = u3r_mean(cor, u3x_sam_14,  &sample, 0);            
		   fprintf(stderr, "ret = %i\n\r", ret);
		   u3m_p("14: ", sample);		

		   ret = u3r_mean(cor, u3x_sam_15,  &sample, 0);            
		   fprintf(stderr, "ret = %i\n\r", ret);
		   u3m_p("15: ", sample);		
```

If the hoon that you're jetting looks like this
```
		  ++  make-k
			~/  %make-k
			=,  mimes:html
			|=  [aaa=@ bbb=@ ccc=@]
```
In the C code you'd fetch them out of the payload with
```
		  u3r_mean(cor,
				   u3x_sam_2, & arg_aaa,
				   u3x_sam_5, & arg_bbb,
				   u3x_sam_6, & arg_ccc
				   ...
				   0)
```
If you're confident, go ahead and write code.  If you want to inspect
your arguments to see what's going on, you can pretty print the
sample.

You could - IN THEORY - inspect / pretty-print the noun by calling
```
		u3m_p("description", cor);  :: DO NOT DO THIS !!!
```
...but you don't want to do this, because, recall, cor contains the
entire context - the whole enchilada.

Do instead, perhaps,
```
    c3_o ret;
    u3_noun sample;

    ret = u3r_mean(sample, u3x_sam_1,  &sample, 0);
    fprintf(stderr, "ret = %i\n\r", ret); // we want ret = 0 = yes 
	u3m_p("sample", sample);			// pretty print the entire sample
```

After our C function pulls out the arguments it needs to type check them.

If arg_a is supposed to be a atom, trust but verify:
```
   		u3ud(arg_a);  // checks for atomicity; alias for u3a_is_atom()
```
If it's supposed to be a cell:
```
       u3du(arg_a);  // checks for cell-ness
```
There are other tests you might need to use
```
		u3a_is_cat()
		u3a_is_dog()

		u3a_is_pug()
		u3a_is_pom()
```
All of these rests return Hoon loobeans (yes/no vs true / false), so
check return values vs c3n / c3y.  If any of these u3_mean(), u3ud()
etc return u3n you have an error and should return
```
      return u3m_bail(c3__exit);
```
otherwise pass the arguments into your "inner" jet function and return the results of that.

### Edit the C source code to add the u3qe_() function

The u3qe_() function is the "real" jet - the C code that replaces the hoon.

First, you may need to massage your inputs a bit to get them into types that you can use.

You have received a bunch of u3_nouns or u3_atoms, but you presumably want to do
things in a native C / non-Hoon manner: computing w raw integers, etc.

A u3_noun will want to be further diassembled into atoms. 

A u3_atom represents a simple number, but the implementation may or
may not be simple.  If the value held in the atom is 31 bits or less,
it's stored directly in the atom.  If the value is 32 bits the atom
holds a pointer into the loom where the actual value is stored. ( see https://urbit.org/docs/vere/nouns/ )

You don't want to get bogged down in the details of this - you just
want to get data out of your atoms.

If you know that the data fits in 32 bits or less, you can use
```
   u3r_word(c3_w    a_w, u3_atom b);
```

If it is longer than 32 bits, use
```
  u3r_words(c3_w    a_w, c3_w    b_w, c3_w*   c_w, u3_atom d);
```

or
```
   u3r_bytes(c3_w    a_w, c3_w    b_w, c3_y*   c_y, u3_atom d)
```

If you need to get the size
```
   u3r_met(3, a);
```   

The actual meat of the function is up to you.  What is it supposed to do?

Now we move on to return semantics.

First, you can transfer raw values into nouns using
```
    u3_noun u3i_words(c3_w a_w, const c3_w* b_w)
```
and you can build cells out of nouns using
```
   u3nc(); // pair
   u3nt(); // triple
   u3nq(); // quad

```
There are two facets here: (i) data format, (ii) memory allocation.

Regarding data format, if the hoon is expected to return a single atom (e.g. if the hoon looks like this
```
      ++  make-k            
        ~/  %make-k
        |=  [has=@uvI prv=@]     ::  <---- input parguments
        ^-  @                    ::  <---- return value is a single value of type '@' (atom)
		...
```
then your C code - at least when you're stubbing it out - can do something like
```
	 return(123);
```
Or, if you want to create an atom more formally, you can build it like this
```
     // this variable is on the stack and will disappear
	 unsigned char nonce32[32];  

	 // this allocates an indirect (> 31 bits) atom in the loom, 
	 // does appropriate reference count, and returns the 32 bit handle to the atom
     u3_noun nonce = u3i_words(8, (const c3_w*) nonce32);

	 // this returns the 32 bit handle to the atom
	 return(nonce);
```
If, on the other hand, your hoon looks like
```
      ++  ecdsa-raw-sign                                ::  generate signature
        ~/  %ecdsa-raw-sign
        |=  [has=@uvI prv=@]
        ^-  [v=@ r=@ s=@]
        ...
```
ending your C code with
```
	 return(123);
```
is wrong and will result in a dojo error because you are returning a single atom, instead of a list of three atoms.

Instead do one of these:
```
     return(u3nc(a, b));		// for two atoms 
     return(u3nt(a, b, c));		// for three atoms 
     return(u3nq(a, b, c, d));	// for four atoms 
```
If you need to return a longer list, you can compose your own.  Look
at the definitions of these three functions and you will see that they
are just recursive calls to the cell constructor u3i_cell() e.g.
```
		 u3i_cell(a, u3i_cell(b, u3i_cell(c, d));
```

Understanding the memory model, allocation, freeing, and ownership ('transfer' vs 'retain' semantics) is important.  
Some info is available at https://www.urbit.org/docs/vere/nouns/


## Compile the C code

You've edited C code; now you need to compile it to build a new urbit executable!


1. cd ~/tlon/urbit
1. rm -rf build
1. configure: choose one of 
    1. meson configure -Dbuildtype=release build
    1. meson configure -Dbuildtype=debugoptimized build
    1. meson configure -Dbuildtype=debug build
	Note that debugoptimized builds run slower than release builds, and debug builds run slower yet.  Booting a debug builkd of urbit may take several minutes. [ N.B. compiler flags -O2 vs -O3 ]
1. scripts/build

## Compile the Hoon code into a pill

You've compiled the Hoon code.  Now you need to compile it.

Wait, compile?  Isn't Hoon that one types at the dojo prompt interpretted, or compiled on the fly?

Yes.  But.

When the urbit executable runs the first thing it does is load the
complete arvo operating system.  That step is much faster if it can
load a "jam"-ed "pill", where all of the hoon has already been parsed
from text file into Hoon abstract syntax tree, and then compiled from
the hoon into the Nock equivalent.

Note that this means that if you edit hoon.hoon, zuse.hoon, arvo.hoon,
vane/ames.hoon, etc. and then restart the urbit executable **YOU ARE
NOT RUNNING YOUR NEW CODE**.

The only way to run the new code is to follow the following process:

1. start up urbit, tell it to be a fake zod (act as a galaxy and disconnect from the rest of the network) and have it know where your edited arvo files are (although not execute them, as discussed above):
    1. rm -rf ~/zod
	1. rm -rf ~/tlon/mypill.pill
    1. ~/tlon/urbit/build/urbit  -FI zod -A ~/bus/tlon/arvo/ -c zod
	This should take < 90 seconds.
	1. If you accidentally exit urbit you can restart it 
1. inside your urbit, from the dojo command line, load the hoon files and compile them into a pill file:
    1. .pill +solid
	This should take < 90 seconds.
	1. examine the output.  Expect to see something like this
	```
	%solid-start
	%solid-loaded
	%solid-parsed
	%solid-compiled
	%solid-arvo
	[%solid-kernel 0x6aa7.627e]
	%arvo-assembly
	[%solid-veer p=%$ q=/zuse]
	[%tang /~zod/home/~2018.7.25..20.47.51..0027/sys/zuse ~mondyr-rovmes]
	```
	1. if, on the contrary, you see anything like this:
	```
	syntax error
	{3,356 7}
	/~zod/home/~2018.7.25..20.47.51..0027/sys/zuse
	```
	you have an error.  In this case the error is on line 3,356, character 7 of file ~/tlon/arvo/sys/zuse.hoon.


	1. the urbit executable will not reload the .hoon files as you edit them.  Neither `|reset` nor `.pill +solid` will make edits visible.  Instead do this:
	    1. in dojo: `|mount /=base=`
	    1. in dojo: `|autoload %.y`
		1. at the command line : `ln -s ~/tlon/arvo/sys/zuse.hoon ~/zod/base/sys/zuse.hoon `   (make changes to both arguments of the ln command if you're editing, say, base/sys/vane/ford.hoon)
		This does three things: it makes urbit's file system mount your own, it makes urbit scan its own filesystem for changes and load them automatically, and it keeps your zuse.hoon
		Make a change to the hoon file.  In dojo you should see `<<sync>>`.  If you do not, one of the above steps was done incorrectly. If you do see this, it will be followed by any compile errors.
	1. Edit the hoon and try again. Keep iterating until the hoon compiles.
1. exit urbit with control-D
1. save the pill file
    1. cp ~/zod/.urb/put.pill  ~/tlon/mypill.pill

[ N.B. another take on explaining the above:

Here's some things to note about the pill (and -A):

An urbit has to boot into the Arvo kernel -- a nock core with a
particular interface. It'd be possible to make some ad-hoc procedure
to initialize Arvo, but it would be a pretty drastic layering
violation and couple urbit to all sorts of internal implementation
details of Arvo and Hoon. In contrast, the pill is basically a
serialized, declarative set of steps to initialize Arvo.

Right now, the boot process is something of a hack, and the pill is
basically a snapshot of a pristine Arvo kernel. In the future (see
+metal or +brass), the pill will contain something like:

1. a boot formula
1. the hoon compiler as nock
1. the hoon compiler as source
1. arvo as source
1. the vanes as source
1. userspace as source (ie, an initial %clay sync)

For now, when you boot a ship that's not a galaxy, the initial %clay
filesystem is synced in from the parent ship. When you boot a galaxy,
you need to pass in a path to a copy of the Arvo repository so that
the %clay filesystem can be initialized without recourse to another
ship.

Hopefully this helps to demystify these startup procedures.

]


## Run the compiled C / compiled Hoon pill

1. type `cd ~`
1. type `rm -rf zod`
1. type `~/tlon/urbit/build/urbit -FI zod -A ~/tlon/arvo/ -B ~/tlon/mypill.pill -c zod`.

   N.B. booting should not take more than about 90 seconds.  If it
   goes longer than that you might have created a 'poison pill', which
   hangs things.  Try booting without the -B flag, and/or reverting
   your hoon changes, generating a new pill based on that, and
   launching urbit with the known-clean pill.  If these steps and boot
   in < 90 seconds, but a boot with a pill created from your own hoon
   does not, you have a hoon bug of some sort.

   Hoon bugs that disable booting can be as simple as the wrong number
   of spaces.  Many, but not all of them, will result in compile
   errors during the `.pill +solid` step.  If your booting takes > 90
   seconds, abort it, and look at your hoon.  Do a git diff, and
   remove half the edits.  Run through the entire process again.  If
   booting from the pill still hangs, remove half of what remains.  Do
   a binary search to find the precise error.

1. inside dojo: `|reset`
1. You now have created a galaxy fakezod, on its own detached network, running your own strange variant of the OS.
1. inside dojo: `(make-k:secp256k1:secp:crypto `@uvI`12 33)`  # or whatever command runs the hoon you're jetting
1. do your development work
1. If you happen to exit, you can restart with
     `urbit -FI zod zod`

To expand:
* F means "fake"
* I means "run as galaxy zod"
* the final argument is the unix directory holding the files for the galaxy

## Run the jetted code and compare to the non jetted code

You are now running your jetted code in a fake zod.

It would be convenient to fire up a second urbit in a terminal next to your fake zod, this one using non-jetted hoon.

Then you can run identical commands in each, in parallel, and confirm that they do the same thing.

1. edit ~/tlon/urbit/jets/e/secp.c (or whatever) to include `fprintf(stderr, "*** u3we_sign 2\n");`
(or similar) in your jetted functions.  This will let us verify that the jets exist and are running.
1. `cd ~/tlon/ubit ; scripts/build `

1. `cp ~/tlon/arvo ~/tlon/arvo_nojet`

1. edit ~/tlon/arvo_nojet/sys/zuse_nojetting.hoon to remove jetting

1. Because we can not run two instances of the same fake galaxy on the
same machine (their ports would conflict), we need to pick a new
galaxy name.  Galaxy names are not arbitrary. You may not name your
galaxy "mygalaxy", "withjet", "withoutjet", or anything like that.
Galaxy names are restricted to the range 0-255, converted into
pronounceable syllables.  'Zod' is just the pronunciation of '0' ( to
see this, in a dojo, cast ~zod to an untyped atom thusly: `@`~zod).
Let's pick 1 as our new galaxy id.  Use dojo to convert this into a
urbit name: `@p`1 yields ~nec.

1. Open a new terminal.  In that terminal:   `cd ~ ; rm -rf nec ; ~/tlon/urbit/build/urbit -FI nec -A ~/tlon/arvo_nojetting/ -c nec`
1. This should take < 90 seconds.	
1. inside your urbit, from the dojo command line, load the hoon files and compile them into a pill file:
    ` .pill +solid`
1. This should take < 90 seconds.
1. exit urbit with control-D
1. save the pill file
1. type `cp ~/nec/.urb/put.pill  ~/tlon/mypill_nojetting.pill `
1. type `cd ~`
1. type `rm -rf nec`  
1. type `~/tlon/urbit/build/urbit -FI nec -A ~/tlon/arvo_nojetting/ -B ~/tlon/mypill_nojetting.pill -c nec`


