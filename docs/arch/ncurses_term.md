## Metadata
```
  Title: An Ncurses Based Terminal
  Author: ~rigdyn-sondur
  Status: 
  Created: ~2019.02.13
  Last Modified: ~2019.02.13
```

## Background

As per

	https://urbit.org/docs/learn/arvo/arvo-internals/dill/

	Unix sends keyboard events to %dill from either the console or telnet,
	and %dill produces terminal output. The only app that should directly
	talk to %dill is the terminal app. Command-line apps are run by,
	receive input from, and produce output to, the %shell app, which is
	controlled by %terminal, which talks to %dill, which talks to
	Unix. Clay also uses %dill directly to print out the file system change
	events, but this is questionable behavior.

	%dill has two main components. First, it controls the terminal on a
	very basic, event-by-event level. This includes keeping track of
	things like the dimensions of the terminal, the prompt type (plain
	text or password), which duct to produce effects on, and so
	forth. Second, it handles terminal events, keystroke by
	keystroke. Most characters are simply pushed onto the buffer and
	blitted to the screen, but some characters, including
	control-modified keys, arrow keys, etc. require special
	handling. Most of the read-line functionality is in %dill.

In vere, actual terminal handling is implemented in term.c.  Term.c
seems to have support for multiple terminal connections, but it's
unclear if this functionality is actually used.

Currently term.c is written using termios, libuv, and ANSI escape functionality.

Termios is a Linux API that allows one to declare or overwrite
attributes of the terminal (e.g. "this terminal has size X x Y", "this
terminal can consume bytes at up to rate X/second", "when physical key
X is struck on the keyboard, this terminal will now emit the byte
sequence ABC".

See:   https://linux.die.net/man/3/termios

Libuv is an asynchronous I/O and threading library that allows one to
register all manner of signal or event producing piece of
functionality into a standard framework, and register callbacks for
each.  We use, and have used this, as the core implementation of how
vere detects real world events such as "a packet arrives" or "a key is
struck" and creates Urbit events out of them.  Term.c registers with
libuv so that any keystroke fires a callback, and the callback injects
events (each event bearing one keystroke)  into Hoon via u3_pier_plan().

Hoon injects effects into term.c via u3_term_ef_blit().  Each call of
this function is passed one noun.  The noun contains a pseudo-opcode
and (optionally) arguments.

The supported opcodes are

	* c3__bee - toggle spinner
	* c3__bel - ring the bell
	* c3__clr - clear the current line
	* c3__hop - move cursor to far left
	* c3__lin - clear current line, print 1 line, starting at location of cursor, move cursor to end of line
	* c3__mor - emit linefeed ("\r\n"), move cursor to far left
	* c3__sav - save file by path (??)
	* c3__sag - jam then save file by path (??)
	* c3__url - write url (??)

Blits like c3__bel contain no arguments.

Blits like c3__lin do contain an argument.

In the case of c3__lin and c3__url, the argument is the string to be
placed on the screen - or, more precisely (and somewhat distinctly)
the bytes to be emitted to the terminal.  Note that these are not
identical concepts.  Some byte strings are consumed by the standard
Linux terminal and trigger effects, such as changing the foreground
color.

These escape codes include CSI (Control Sequence Introducer) codes
that do things https://en.wikipedia.org/wiki/ANSI_escape_code

## Current Deficiencies / Near Term Goals

Deficiencies:

1. Under commented.
2. There are ugly code paths.  For example, the blit c3__lin carried a noun from Hoon to C, which is unpacked into bytes, then repacked into a noun, then sent back to Hoon, converted from utf32 to utf8, then sent back to C, unpacked into bytes, and consumed.
3. Separation of concerns is not respected; dill.hoon knows about ANSI escape codes like "<ESC> [ 4 9 m", when it should not.
4. Does not currently use a technology (e.g. ncurses) that makes extension to the blit list easy or possible.
5. The list of blits is inadequate.  Hoon coders want more control over what can be done with the terminal, including setting arbitrary foreground and background colors, setting arbitrary X and Y coordinates, and others.

Goals are the inverse:

1. Add comments.
2. Remove ugly code paths.
3. Separate concerns: remove all ANSI escape knowledge from dill.hoon
4. Use 'ncurses' library to reimplement existing functionality and make adding these new blits easy. (  https://en.wikipedia.org/wiki/Ncurses )
5. Augment the blits by adding:
	* c3__bog - set background color for future printing
	* c3__fog - set foreground color for future printing
	* c3__seg - starting at current location of cursor, place text, move cursor to end of emitted text
	* c3__sxy - set x and y coordinates

## Status

As of ~2019.02.13:

1. comments: 100%
2. ugly code paths: 50%
3. remove ANSI escapes from dill.hoon: 50%  The tricky bit for this beginning Hoon coder: learning enough Hoon to rewrite part of dill so that instead of of converting a 'stub' (a list of pairs of style-and-text) into one string, which is emitted in one c3__lin blit, instead have Dill.hoon emit multiple effects of form c3__bog, c3__fog, c3__seg, c3__bog, c3__fog, c3__seg ...
4. use 'ncurses' library to reimplement existing functionality: 100%
5. add three new blits: 80%


## References

*  https://urbit.org/docs/learn/arvo/arvo-internals/dill/

*  https://linux.die.net/man/3/termios

*  https://en.wikipedia.org/wiki/ANSI_escape_code

*  https://en.wikipedia.org/wiki/Ncurses

*  https://urbit.org/docs/learn/arvo/arvo-internals/dill/



