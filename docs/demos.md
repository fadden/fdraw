fdraw Demo README
=================

The fdraw distribution comes with a handful of demonstration programs.
Most of them are written in Applesoft BASIC, and use the amperfdraw
interface.  This is a somewhat poor way to demonstrate animation
performance, as Applesoft adds a tremendous amount of overhead, but it
is the only way to show what you *can* do with Applesoft.

The easiest way to run them is with the "DEMO" program, which scans the
DEMOS directory for BASIC programs and presents a list.  You can also
just run them directly.

* INTRO : Sort of a "hello, world" for fdraw.  Mix of single- and
  double-buffered animation.

* CIRCULAR : Draws lots of circles.

* RECTSPLAT : Draws lots of rectangles.

* CUBIC : Draws a spinning wireframe 3D cube.  (The 3D coordinates are
  pre-computed -- fdraw doesn't do matrix transforms.)

* TUNNEL : Animates circles to simulate driving through a tunnel.

* LINEAR : Draws lots of lines.  The wipes show speed differences for
  horizontal and vertical special cases, while the circular spinner
  shows HPLOT is not as fast as &HPLOT which is not as fast as &PLOT for
  a set of lines at a variety of angles.

* LINE.DIFF : Draws several lines with the ROM routines and fdraw
  side-by-side to illustrate the difference in line style.

* CLEARLY : Clears the screen 32 times, 4 sets in each of the 8 colors.
  The first round is done with the Applesoft ROM routine ("CALL 62454"),
  the second round uses the fdraw &CLEAR function.

* HRFAN : A simple line-art demo, using "xdraw" DrawLine with lines in
  different colors.  Not a great demo, as the Applesoft code driving it
  is rather slow, but it looks pretty good if you bump up the emulation
  speed or switch to IIgs "fast" mode.  (This deserves a conversion to
  assembly language.)

* BRIAN.THEME.ORI : The Brian's Theme demo from the DOS 3.3 System
  Master.  Unmodified except for integration with the demo menu
  system, and with the bug on line 31112 fixed.

* BRIAN.THEME.NEW : The Brian's Theme demo with '&' placed in front of
  the various draw calls.  There isn't a huge difference in speed, as
  there's a lot of overhead from Applesoft, but its interesting to note
  the change in the appearance of the lines.

* WIGGLE : Sample program that shows direct use of rasterization tables.

When the demos are launched from the menu, they will assume that fdraw
is already loaded and won't try to load it again.  If you run the demo
program directly, it will try to load FDRAW.FAST and AMPERFDRAW from the
parent directory before doing any drawing.


## Extras ##

The EXTRAS directory has some additional software that isn't "officially"
part of fdraw, but may be of use.

NOTE: some of these assume fdraw and amperfdraw are already loaded, and
will hang if not.  Run DEMO and hit &lt;esc&gt; before running these.

* ARRAY.EXAMPLE : The &PLOT example from the documentation.

* XDRAW.ANIM : A demonstration of line animation using "xdraw" mode and
  a simple shape that is drawn twice by a single &PLOT call.  One copy
  is offset by 2 pixels, so each &PLOT call erases the previous copy and
  draws a new copy 2 pixels to the right.  The animation is shown twice,
  once with "erase all, draw all", and once with the erase and draw calls
  interleaved for every line.

* LINEFONT : Program for creating draw-array tables for text phrases.  Used
  to create data files for the "intro" demo.  See the "LINEFONT Details"
  section for more information.

* DAVIEWER: Views the contents of .DA files created by LINEFONT.

* BENCHCLEAR : Calls the "clear" function 256 times from a small
  assembly-language program.  Handy for benchmarks, but slightly silly
  since it's relatively easy to calculate the exact cycle cost.


## LINEFONT Details ##

NOTE: this program is an unfinished rough cut ("pre alpha"), used for
preparing data for demos.

The program includes a font definition, routines for displaying
characters, and code for generating and exporting pre-rendered strings.

Character vertices are expressed as floating-point values.  The baseline
is at zero, the peak ascent is at 1.0, the lowest descent is -1.0. The
leftmost pixel is at zero, the maximum value for the rightmost pixel is 1.0.
Characters don't have to fill out the entire cell -- proportionally-spaced
fonts are supported -- but they are expected to start at the left edge.

So a capital 'M' might look like this:

  0.0,0.0 -> 0.0,1.0 -> 0.5,0.7 -> 1.0,1.0 -> 1.0,0.0

There is currently no "user interface", unless the "user" can program in
Applesoft BASIC.  To generate strings, add a series of statements that set
variables and call 20000 to add rendered strings to the set.  The relevant
variables are:

 * S$ - string to add
 * DW - desired width, in pixels, of a cell 1.0 units wide
 * DH - desired height, in pixels of a cell 2.0 units high (ascent + descent)
 * IS% - inter-character spacing, in pixels
 * SW% - width of the space character (usually same as DW)
 * MO% - monospace flag; if nonzero, all chars are treated as 1.0 units wide

Remove the REM from the start of line 1010 to enable the character viewer.
At present only a couple of lower-case letters are defined.


#### LINEFONT Output ####

The LINEFONT program outputs a binary blob that can be passed to
the &PLOT array-draw function.  The file structure is:

    +0  byte - number of array sets in the list.
    +1  2 bytes * N - table of offsets to individual array sets.  One of
        these per array set.  The value is the offset from the start of the
        file.

    (2N+1) array set #1:
    +0  byte - number of vertices (0-127)
    +1  byte - number of index pairs (0-127)
    +2  2 bytes * V - vertices (values are signed X/Y)
    +X  2 bytes * I - index pairs (values are 0-127)

To display phrase #3, you would get the 16-bit value from the offset
table with `PEEK(start + 1 + 3 * 2) + PEEK(start + 2 + 3 * 2) * 256`.
You get the number of vertices from `PEEK(start + offset)`, and the number
of index pairs from `PEEK(start + offset + 1)`.  Finally, call the array-draw
function with:

    VA = start + offset + 2
    IA = VA + num_vertices * 2
    &PLOT va, ia, num_index_pairs

The 0,0 point in the blob is in the center of the phrase horizontally
(which allows a maximum width of 255 pixels), and at the font baseline
vertically (so most of the font will appear above the zero point, but
descenders will extend below).


#### Future Enhancements ####

Right now the font definition is embedded in the program.  This takes up
a lot of space -- before too long the BASIC program is going to intrude
on the hi-res page -- and is unnecessarily restrictive.  The font should be
defined by a separate program, and BSAVEd into a line-font file that
LINEFONT can load.

Generating strings should be menu-driven and interactive, rather than
requiring manual changes to the code to fiddle with sizes and spacing.
DAVIEWER should be folded into the generation program (though it's kind
of handy as a simple example of how to unpack and access content).

