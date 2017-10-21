fdraw Library Documentation
===========================

Fast graphics primitives for the Apple II  
By Andy McFadden  
Version 0.3, August 2015

## Overview ##

The fdraw library provides fast rendering of points, lines, rectangles,
and circles, as well as high-speed screen clears, for Apple II hi-res
graphics.  It can be used from Applesoft or assembly language.

The Applesoft ROM routines were designed to be as compact as possible,
and were unable to use self-modifying code techniques, so their speed is
less than what the Apple II is capable of.  The fdraw routines pick a
different point in the speed/space trade-off continuum, providing fast
speeds at a reasonable size.  Not everyone agrees on what "reasonable"
means, so the fdraw code can be built in two modes, one that favors
speed, one that reduces size.

**Contents:**

- [Applesoft BASIC Ampersand API](#amperapi)
- [Raw API](#rawapi)
- [Building the Code](#building)
- [Apple II Hi-res in a Nutshell](#nutshell)
- [Notes on the Drawing Functions](#notes)
- [Enhancement Ideas](#ideas)
- [Additional Notes](#additional-notes)


<div id='amperapi'/>

## Applesoft BASIC Ampersand API (Amperfdraw) ##

The ampersand API acts as a bridge between Applesoft BASIC and fdraw.
It's more convenient and has less overhead than POKE and CALL, though
you are not prevented from using that approach if you prefer.  It's
best to use one or the other though, not mix and match.

All arguments are checked for validity.  An appropriate Applesoft
error is thrown if invalid syntax or arguments are discovered.

This is not intended to be compatible with, nor a replacement for, the
ampersand utilities in Beagle Graphics.

* &NEW - calls the fdraw Init function (which sets the color to 0 and
  selects hi-res page 1).  You must do this once, at the start of
  your program, after fdraw has been loaded.  This also resets internal
  amperfdraw state, setting the "HPLOT TO" origin to (0,0) and the "AT"
  point to (139,95).
* &HGR - does what HGR does, only faster. Equivalent to executing
  `&HCOLOR=0:&SCRN(1):&CLEAR:&HCOLOR=[prevcolor]`, and then setting the
  display softswitches to display hi-res page 1 in mixed mode.  Also sets
  $e6 (HPAG) for convenience in case you want to mix & match with ROM
  routines.
* &HGR2 - like &HGR, but for page 2.  Like HGR2, this turns off
  mixed-text mode.
* &SCRN({1,2}) - sets the hi-res page that will be used for drawing.  Does
  not change which page is displayed.  (Use the softswitches, or call
  &INVERSE.)
* &INVERSE - flips the render page to the other page, and hits the
  display softswitches to show the page that was just rendered.  Intended
  for double-buffered animation.
* &HCOLOR={0-7} - sets color, using the same numbering scheme as Applesoft.
  Does not affect the color used by the ROM routines.
* &CLEAR - clears screen to current color.
* &HPLOT [TO] x,y [TO x,y ...] - draws a point or a line.  Works the same as
  Applesoft, e.g. "&HPLOT TO" starts from the end of the previously
  drawn line, and you can chain multiple "TO x,y" in a single statement.
* &EXP {0,1} - set line mode.  0 is normal, 1 is "xdraw".
* &XDRAW left,top,right,bottom - draws outline rectangle.
* &DRAW left,top,right,bottom - draws filled rectangle.
* &COS cx,cy,r - draws outline circle.
* &SIN cx,cy,r - draws filled circle.

* &AT cx,cy - sets center offset for array-based rendering.  Position must
  be on the hi-res screen (0-279, 0-191).
* &PLOT vertexAddr, indexAddr, indexCount [AT cx,cy] - draws from the
  specified byte-arrays.  See the "Drawing Lines with Indexed Byte-Arrays"
  section for the full explanation.


<div id='rawapi'/>

## Raw API ##

The code is assembled at $6000 by default.  The program's length includes
all data tables and work areas, and no memory outside of the program,
zero page, and the current hi-res page is modified.

Input parameters and the function jump table are located near the start
of the program.  The API description below describes the addresses in
relative terms.

Input parameters are not checked for validity.  They must be in the range
specified by the API, or undefined (but probably bad) behavior will result.
The values will not be modified by fdraw functions.

All drawing operations use the current color.

* +0   Init - call this when the library is first loaded.  It must be
       called before any other functions are used.  It initializes the
       color to zero and the page to $20.
* +3   (major version number, currently 0)
* +4   (minor version number, currently 3)
* +5   Input parameter area:
  *  +5   arg - used for misc functions, e.g. SetColor and SetPage
  *  +6   x0l - low part of the X0 coordinate (0-279)
  *  +7   x0h -   high part of X0
  *  +8   y0  - Y0 coordinate (0-191)
  *  +9   x1l - low part of X1 (0-279)
  *  +10  x1h -   high part of X1
  *  +11  y1  - Y1 coordinate (0-191)
  *  +12  rad - circle radius (0-255)
* +13  (reserved)
* +16  SetColor - set the color used for drawing (0-7) to the value in "arg".
       The numbering is the same as the Applesoft hi-res colors.
* +19  SetPage - set the hi-res page used for drawing to the value in "arg",
       which must be $20 or $40.  Does not change the page that is displayed.
       (Because a bad value can cause memory corruption, this value *is*
       checked, and bad values rejected.)
* +22  Clear - erase the current hi-res page to the current color.
* +25  DrawPoint - plot a single point at x0,y0.
* +28  DrawLine - draw a line from x0,y0 to x1,y1 (inclusive).
* +31  DrawRect - draw a rectangle with corners at x0,y0 and x1,y1 (inclusive).
       x0,y0 is the top-left, x1,y1 is the bottom-right.  The left and
       right edges will be drawn two bits wide to ensure that the edges
       are visible (drawn at x0+1, x1-1).
* +34  FillRect - draw a filled rectangle with corners at x0,y0 and x1,y1
       (inclusive).
* +37  DrawCircle - draw a circle with center at x0,y0 and radius=rad.
* +40  FillCircle - draw a filled circle with center at x0,y0 and radius=rad.
* +43  SetLineMode - set the DrawLine mode to the value in "arg", which can
       be 0 (normal) or 1 (xdraw).
* +46  (reserved)

* +49  FillRaster - draw an arbitrary shape from the rasterization tables.
       For each line from top to bottom, the left and right edges will
       be read from rastx1/rastx2 and a raster drawn in the current color.
* +52  (byte) topmost line to rasterize (0-191)
* +53  (byte) bottom-most line to rasterize (0-191), inclusive
* +54  (2 bytes) address of rastx1l table
* +56  (2 bytes) address of rastx1h table
* +58  (2 bytes) address of rastx2l table
* +60  (2 bytes) address of rastx2h table

The rasterization table addresses are read-only; changing them will have
no effect.

fdraw uses a fair number of zero page locations.  The exact set can be
determined by looking at FDRAW.S.  The locations were chosen to not
interfere with DOS, ProDOS, Applesoft, or the Monitor.  They may
interfere with Integer BASIC, SWEET16, or your own application code.
Remapping them to different locations is straightforward: just change
the assignment of zptr/zloc values near the top of FDRAW.S to use
different addresses.  fdraw does not expect any zero page value to be
preserved across calls, so you're welcome to use those locations in your
own code, but understand that fdraw functions will overwrite them.


<div id='nutshell'/>

## Apple II Hi-res in a Nutshell ##

This is a quick overview of the Apple II hi-res graphics architecture
for anyone not recently acquainted.

The Apple II hi-res graphics screen is a quirky beast.  The typical
API treats it as 280x192 with 6 colors (black, white, green, purple,
orange, blue), though the reality is more complicated than that.

There are two hi-res screens, occupying 8K each, at $2000 and $4000.
You turn them on and flip between them by accessing softswitches in
memory-mapped I/O space.

Each byte determines the color of seven adjacent pixels, so it takes
(280 / 7) = 40 bytes to store each line.  The lines are organized into
groups of three (120 bytes), which are interleaved across thirds of
the screen.  To speed the computation used to find the start of a
line in memory, the group is padded out to 128 bytes; this means
((192 / 3) * 8) = 512 of the 8192 bytes are part of invisible
"screen holes".  The interleaving is responsible for the characteristic
"venetian blind" effect when clearing the screen.

Now imagine 280 bits in a row.  If two consecutive bits are on, you
get white.  If they're both off, you get black.  If they alternate
on and off, you get color.  The color depends on the position of the bit;
for example, if even-numbered bits are on, you get purple, while
odd-numbered bits yield green.  The high bit in each byte adjusts the
position of bits within that byte by half a pixel, changing purple and
green to blue and orange.

This arrangement has some curious consequences.  If you have green and
purple next to each other, there will be a color glitch where they meet.
The reason is obvious if you look at the bit patterns when odd/even meet:
`...010101101010...` or `...101010010101...`.  The first pattern has two
adjacent 1 bits (white), the latter two adjacent 0 bits (black).  Things
get even weirder if split occurs at a byte boundary and the high bit is
different, as the half-pixel shift can make the "glitch" pixel wider or
narrower by half a pixel.

The Applesoft ROM routines draw lines that are 1 bit wide.  If you execute
a command like `HGR : HCOLOR=1 : HPLOT 0,0 to 0,10`, you won't see
anything happen.  That's because HCOLOR=1 sets the color to green,
which means it only draws on odd pixels, but the HPLOT command we gave
drew a vertical line on even pixels.  It set 11 bits to zero, but since
the screen was already zeroed out there was no apparent effect.

If you execute `HGR : HCOLOR=3 : HPLOT 1,0 to 1,10`, you would expect a
white line to appear.  However, drawing in "white" just means that no
bit positions are excluded.  So it drew a vertical column of pixels at
X=1, which appears as a green line.

If (without clearing the screen after the previous command) you execute
"HCOLOR=4 : HPLOT 5,0 to 5,10`, something curious happens: the green line
turns orange.  HCOLOR=4 is black with the high-bit set.  So we drew a
line of black in column 5 (which we won't see, because that part of the
screen is already black), and set the high bit in that byte.  The same
byte holds columns 0 through 6, so drawing in column 5 also affected
column 1.  We can put it back to green with "HCOLOR=0 : HPLOT 5,0 to 5,10".

It's important to keep the structure in mind while drawing to avoid
surprises.

Note that the Applesoft ROM routines treat 0,0 as the top-left corner,
with positive coordinates moving right and down, and lines are drawn
with inclusive end coordinates.  This is different from many modern
systems.  fdraw follows the Applesoft conventions to avoid confusion.

Handy table of graphics softswitches:

name   | addr  | decimal | purpose
------ | ----- | ------- | ------------------
TXTCLR | $c050 | -16304  | enable graphics
TXTSET | $c051 | -16303  | text-only
MIXCLR | $c052 | -16302  | disable mixed mode
MIXSET | $c053 | -16301  | enable mixed mode (4 lines of text)
LOWSCR | $c054 | -16300  | display page 1
HISCR  | $c055 | -16299  | display page 2
LORES  | $c056 | -16298  | show lo-res screen
HIRES  | $c057 | -16297  | show hi-res screen


<div id='building'/>

## Building the Code ##

The main fdraw code is written for the Merlin assembler (specifically
Merlin-16 3.40, though other versions should work).  It uses plain 6502
code, and is expected to run on an Apple ][+.

For convenience when editing the files on an Apple II, and to allow the
code to be compiled by Merlin-16 running under ProDOS 8, the code is
broken into four files.  The main file, FDRAW.S, includes the other
three with PUT directives.  FDRAW.S holds the API entry points and some
of the drawing code.  FDRAW.LINE.S has the code for drawing points and
lines, while FDRAW.CIRCLE.S has the code for drawing circles.
FDRAW.TABLE.S holds the data tables, as well as empty space for work
areas.  The empty space is included in the binary so you can determine
the full memory footprint by looking at the length of the file.

Near the top of FDRAW.S is a constant, `USE_FAST`, which may be set
to 0 or 1.  If set to 0, some code optimizations are disabled,
reducing the size of the code and data areas.  Further, the page
alignment on data tables is disabled, reducing the internal fragmentation
of the data area.

The USE_FAST setting also determines which file recevies the assembler
output: FDRAW.FAST or FDRAW.SMALL.  To generate both, it is necessary to
assemble the file, change the constant, and then assemble the file again.

Tests and demos are written in Applesoft BASIC, with a couple of
exceptions.


### Why So Big? ###

The fdraw code weighs in at a hefty 5KB (or 4KB for the "small" build).
That doesn't sound like much in the age of multi-gigabyte mobile phones,
but it's a sizeable fraction of the space available on an Apple ][+.

If you want to modify individual pixels quickly, you need two things:
a line base-address table, and a divide-by-7 table.  Computing base
addresses and dividing by 7 aren't hugely expensive, but we're going
to be doing them often, so they need to be as fast as possible.

The line address table has 192 entries, one for each line, 2 bytes per
entry.  The divide-by-7 table has 280 entries, one for each horizontal
pixel position, with one byte for the dividend and one for the quotient.
(The quotient can be expressed as a numeric value from 0 to 6, or as
a byte with a specific bit set.)

That's 944 bytes.  For optimum performance, each table must fit on a
single page of memory.  We can split the division table into two pieces,
one for 0-255 and one for 256-279, and put the smaller half on the same
page as the Y table, along with 16 bytes of padding.  The final size is
256 + 256 + (192+24+24+pad) + 192 = 960.  So you can write off 1K of
memory before you've written any code.

(There's a clever way to reduce the size of the y-lookup table to 24
entries, but it's slightly faster and much easier to use full tables.)

For the FillRaster function, fdraw needs to record the left and right
X coordinates on each line (2 bytes each), so that's 192 * 4 = 768 bytes.
Again, for optimum performance, each table needs to be on its own page,
so for USE_FAST=1 that expands to 1024 bytes.

Add to that another full page of unrolled rasterization code, and you've
got 2304 bytes of tables.

The rest is code, most of which was written with a flagrant disregard
for size.  Many common code fragments are repeated inline, rather than
called as a subroutine, because a subroutine call (JSR+RTS) costs 12
cycles.  Calling a common "plot a point" function from the line-drawing
code would increase the per-pixel cost by 15-20%.


<div id='notes'/>

## Notes on the Drawing Functions ##

### Screen Clear ###

The Clear function erases the current hi-res page to the current color.
It's several times faster than the version built into the ROM.

#### Performance ####

The fastest possible way to clear the screen to a specific color on a
6502 is to write to every visible location with an absolute store
instruction.  Subtracting the screen holes, that's 7680 address *
4 cycles = 30720 cycles.  The code to do that would be 23,040 bytes long,
making it impractical.

A slower but more memory-efficient approach has one store statement for
each line, and iterates through 40 times (280 / 7 = 40).  Factoring in the
loop overhead, that comes out to 40 * (192 * 5 + 9) = 38760 cycles.
192 sets of store instructions fills 576 bytes, which is much better
than 23K, but still quite a lot.

We can reduce the size further by taking the lines 3 at a time, erasing
the first 120 bytes in each 128-byte group (the last 8 bytes are the
screen hole).  We'd need to use 7680/120 = 64 store instructions, for a
total of 120 * (64 * 5 + 9) = 39480 cycles, with 192 bytes for the main
part of the erase loop.  We're not quite 2% slower, but 384 bytes
smaller, which seems a fair trade-off.  Because we're accessing memory
linearly we now have a "venetian blind" clear, which is something of an
Apple II trademark, but we can fix that by spending an additional 522
cycles to erase the screen in thirds (top/middle/bottom).

Any further changes that make the code smaller also increase the execution
time.  When built with USE_FAST=0, the code will use a different loop
with 32 stores that write 248 bytes each, and takes 41416 cycles.  It's
half the size, but nearly 2000 cycles slower, and overwrites half of the
screen holes.

At the extreme end of space over speed is the Applesoft ROM routine -- HGR
or "CALL 62454" -- which only needs about 30 bytes for its main loop, but
takes (8192 * 33)+(12 * 64)+17 = 271121 cycles for black or white, or
(8192 * 40)+(12 * 64)+17 = 328465 cycles for green/purple/blue/orange --
7-8x slower than our preferred implementation.

The screen clear is wired to a specific hi-res page, so the SetPage
function must rewrite the store instructions when the page changes (or
we need to keep two full copies of the function around).  For an
application that is constantly doing flip-erase, the overhead must be
factored into the efficiency of the approach -- for example, rewriting
stores with indexed LDA/EOR/STA in a loop will take 20 cycles per iteration,
1280 cycles for the full set of 64.  The "slow" clear has half the
number of store instructions, so takes half the time to fix up after
a page flip.


### Raster Fill ###

Drawing an outline of a rectangle or circle can be done efficiently by
drawing lines or plotting points.  Drawing a filled shape is more
expensive if one point is plotted at a time, especially on the Apple II
where every byte affects 7 pixels.

For filled shapes, fdraw populates a rasterization table.  The table has
192 entries, each of which holds the left and right edges of the shape
on that line.  The code fills in the pixels one line at a time, using
a simple byte store for the middle parts, and bit masks at the edges.

External applications can use the raster renderer directly by filling
out the rasterization table and calling FillRaster.

While the FillRaster function itself will not modify the contents of the 
raster tables, other fdraw calls will, sometimes unexpectedly.  For
example, drawing a horizontal line is performed with a single-line
fill call.  Filled rectangles might populate the table in the way you'd
expect, or might use some internal shortcut that only fills out one line
and sets a "repeat" flag.  Don't make assumptions about what will be in
the table after a call to one of the drawing functions.  You *can* count
on whatever you wrote there yourself to be unmodified after calls to
FillRaster, SetColor, or SetPage, so you can do page-flipping and
color-cycling without having to repopulate the tables.

#### Performance ####

The fill code needs about 100 cycles to set up each line when drawing
a rectangle, more if the line doesn't start and end on byte boundaries.
The inner loop costs 10 cycles per byte.  To clear the screen with the
raster fill code, it would take (192 * (100 + 40 * 10)) = 96000 cycles,
or nearly 2.5x the time required for the dedicated clear code.  Which is
about what you'd expect, as the screen erase needs 4 cycles per byte, and
has lower per-line overhead.  (This can be improved significantly; see
the notes in the "enhancements" section.)

Non-rectangular shapes take slightly longer to set up, as the edges must
be recomputed for each line.


### Lines ###

The goal is to provide a replacement for Applesoft's HPLOT function
that is faster and more consistent in appearance.  Lines are drawn using
Bresenham's run-length algorithm.

Internally, there are five separate functions.  Horizontal and vertical
lines each get a special-case handler.  There's another for mostly-vertical
lines, one for mostly-horizontal lines, and one for wide mostly-horizontal
lines (255 pixels or wider).  The latter requires 16-bit math, and is
slightly slower.

The Applesoft routine isn't quite the same as the standard Bresenham
algorithm, because it doesn't move diagonally.  Consider a line from
(0,0) to (50,10) -- gently sloping down and to the right.  The standard
algorithm would plot exactly 51 pixels, one in each horizontal position.
The "pen" always moves one pixel right, but sometimes also moves down.

In Applesoft, the "pen" can move either right or down, but can't do
both at once.  This results in lines that feel thin when near horizontal
or vertical, but become thicker as they approach 45 degrees.  This
reduces performance, because Applesoft draws twice as many pixels for a
diagonal line as the basic algorithm.  It can also be visually jarring
when animated, because lines get very thick when near diagonal.

Different applications have used different styles; for example:

- Stellar 7 and Elite for the Apple II use Bresenham-style lines.  If
  you look at near-diagonal lines on a color monitor you can see the
  pixels alternating green and purple.
- A2-FS1 Flight Simulator appears to be using Bresenham lines but with
  doubled bits, effectively treating the screen as having 140 pixels.  This
  gives solid white lines with a fairly consistent feel.
- GraFORTH doubles the bits, but treats the screen as 256 pixels wide
  (not 280... it gives up 24 pixels to improve performance).  White
  lines are thick like Flight Simulator, but feel less jagged because
  each step can move left or right by one bit rather than two.

The SetLineMode function lets you choose between "draw" and "xdraw".  The
former draws color pixels, setting and clearing bits as needed, while
the latter inverts whatever is currently on the screen.  This can have
some unusual effects.  Drawing the same line twice erases the line.
Drawing a green line over a purple line gives you a white line.  Drawing
with colors 5 and 6 can produce odd results, because the high bit inverts
every time you touch a byte -- which means the ends of a horizontal line
will be a different color if the byte holds an even number of affected
pixels.  It's best to draw with colors 0-3 when in xdraw mode.  Clearing
the background to color 4, rather than 0, will cause drawing in colors
0-3 to actually be 4-7.

#### Performance ####

Mostly-horizontal lines step horizontally each iteration, and sometimes
step vertically.  Mostly-vertical lines step vertically each iteration,
and sometimes step horizontally.  Each part of the operation has a cost,
so the fastest lines are the ones drawn primarily in a single direction.
Diagonal lines are the worst case for performance.

The current code requires just under 80 cycles per pixel for diagonal
movement, and about 56 for single-direction movement.  There's another
150 cycles or so per line for the initial setup.

Vertical lines cost about 43 cycles per pixel.  Horizontal lines are
handled as a trivial FillRaster call, which at peak performance can write
7 pixels in 10 cycles.

This is about as fast as you can get with the Bresenham run-length
algorithm and Applesoft-style color handling.

It's possible to go faster by switching to a different pixel style, or
using the run-slice approach.  Drawing the line toward the center from
both ends is likely a losing proposition due to the small number of
registers on the 6502, and is tricky when rendering in color because of
the need to keep the high bit set.  (fdraw keeps the line number and
byte offset in the X/Y registers, and always draws left to right to
make the bit-shifting simpler; neither of these would be possible if we
plotted from both ends).  Employing Wu and Rockne's double-step
algorithm (see e.g. Wyvill's article in Graphics Gems, page 101) should
offer some improvement over the basic algorithm, as it computes two
successive pixels at each step, creating an opportunity to set two
bits with a single store.

Unrolling the loops is impractical because of the number of
instructions in the core loop, with one exception: pure-vertical lines
would only need about 20 bytes per loop.  By creating 8 instances, we
can replace the line base address lookup with a simple `ADC #$04` for
7 out of 8 iterations, saving 8 cycles per pixel.  For maximum effect
we'd need a "finishing" loop at the end, and a branch table to jump into
the middle of the unrolled code at the start, for a total cost of around
200 bytes (less if we're not drawing Applesoft-style lines).  The cost
would be justified for a game like Stellar 7 where the view does not roll,
which means vertical lines (such as those on obstacle cubes) will
always be vertical.


### Rectangles ###

Filled rectangles are currently implemented by putting the left and
right edges into the rasterization table, and calling FillRaster.

Outline rectangles could be drawn as four lines, but that doesn't look
very good in color unless you get the lines on the right columns.  To
ensure that the edges are in the correct color, outline rectangles are
drawn as four separate items: a two-pixel-wide left edge, a two-pixel-wide
right edge, and horizontal lines at the top and bottom.  FillRaster does
the actual work.

#### Performance ####

FillRaster is suboptimal for rectangles, because it works by rows rather
than by columns (see "Vertically-Challenged Rasterization" later in this
document).  Rectangles could be drawn 2.5x faster with dedicated code,
but at a cost of hundreds of bytes of memory.

The advantage of using FillRaster is that we need it for filled circles,
so adding support for rectangles was nearly free.  And it's still pretty
fast.


### Circles ###

Circles are computed with Bresenham's algorithm.  The idea is to compute
one octant of the circle with this bit of magic:

    void drawOutline(int cx, int cy, int rad) {
        int x, y, d;

        d = 1 - rad;
        x = 0;
        y = rad;

        while (x <= y) {
            plot(cx, cy, x, y);

            if (d < 0) {
                d = d + (x * 4) + 3;
            } else {
                d = d + ((x - y) * 4) + 5;
                y--;
            }
            x++;
        }
    }

Then each X/Y coordinate is plotted eight times:

    (cx+x, cy+y) (cx-x, cy+y) (cx+x, cy-y) (cx-x, cy-y)
    (cx+y, cy+x) (cx-y, cy+x) (cx+y, cy-x) (cx-y, cy-x)

For an outline circle, we plot every point.  For a filled circle, we add
each point to a rasterization table.  Near the top and bottom of the
circle there will be multiple updates to the same line, with each update
replacing the previous one (which works, as we are moving "outward").

The center point of the circle must be on screen, but it's not necessary
for the entire circle to fit.  Coordinates outside screen space are clipped.

#### Performance ####

The implementation of Bresenham's algorithm is straightforward, and is
about as fast as it's going to get.  There are actually two versions of
the core computation.  If the radius is less than 41, we can keep all of
the variables in 8 bits.  For circles with radius 41 and larger, we need
to use 16 bits, slowing each step slightly.

There are also two versions of the octant plot.  If the circle fits entirely
on-screen, we use a simple version.  If it doesn't, we use a version that
clips values.  For rasterization that means clamping X to the left or
right edge, and skipping updates that are off the screen in the Y dimension.
For an outline circle we simply don't plot any clipped points.

The rendering of filled circles is very fast, though there is a possibility
of optimizing the center-fill of large circles.  Outline circles were
added by inserting JSR PLOT at key points, and could perhaps be faster.

The best way to accelerate circle drawing is to not draw a full circle.
Instead, draw an approximation of a circle using a series of lines.  This
is a common technique when using 3D graphics hardware, as they are
generally geared toward polygon rendering.  Outline circles would see a
significant improvement, with some reduction in quality.


### Drawing Lines with Indexed Byte-Arrays ###

The &PLOT command allows a BASIC program to execute a series of line-draw
commands with a single statement.  Think of it like shape-table animation
with lines instead of plotted points.

Suppose you want to draw a rectangle with an X through the middle.  We'll
make it 11 units wide and 21 units high.  To draw that in the middle of
the screen, we'd set CX=139 and CY=95, then draw lines offset from that
by +/- 5 in X and +/- 10 in Y:

    HPLOT CX-5,CY-10 TO CX-5,CY+10 : REM LEFT  
    HPLOT CX-5,CY-10 TO CX+5,CY-10 : REM TOP  
    HPLOT CX+5,CY-10 TO CX+5,CY+10 : REM RIGHT  
    HPLOT CX-5,CY+10 TO CX+5,CY+10 : REM BOTTOM  
    HPLOT CX-5,CY-10 to CX+5,CY+10 : SLASH  
    HPLOT CX+5,CY-10 to CX-5,CY+10 : BACKSLASH  

Six lines, each of which needs four coordinates.  We'd need 24 bytes
to store that in an integer array.

Suppose instead we identified the four vertices, and numbered them:

    #0 CX-5,CY-10  
    #1 CX+5,CY-10  
    #2 CX-5,CY+10  
    #3 CX+5,CY+10

and then created a list of line segments using the vertex indices:

    HPLOT #0 TO #2  
    HPLOT #0 to #1  
    HPLOT #1 TO #3  
    HPLOT #2 TO #3  
    HPLOT #0 TO #3  
    HPLOT #1 TO #2  

This requires (4 * 2) + (6 * 2) = 20 bytes, for a small savings.  The real
value in the approach is that it separates the description of the shape
from the placement of the points.  For example, if you want to change
vertex #0 to (CX-7,CY-12), you don't have to make changes two three
separate HPLOT calls.  (This is particularly useful when you have code
that scales and rotates the vertices.)

For the current release of fdraw, the only built-in transform is
translation.  Using "&AT cx,cy", you can place the center point anywhere
on the screen.  This allows you to animate movement of the shape by
simply calling &AT to change the position, and &PLOT to draw.

The &PLOT command takes three arguments: the address of a vertex array,
the address of an index array, and the number of line segments to draw.
These are referred to as "byte arrays" because they are arbitrary
locations in memory where you have BLOADed or POKEd your shape data, not
Applesoft arrays.  The count can be from 0 to 127.  You can optionally
add an AT to the end; if not present, the coordinates of the previous AT
are used.  The initial value is the center of the screen (x=139 y=95).

The vertex array uses two signed bytes per vertex (-128 to 127), one for
the X coordinate and one for the Y coordinate.

The index array uses two bytes per line segment.  Each byte is an index
into the vertex array, from 0 to 127.

Here's an Applesoft program that implements the above example.  (The DATA
statements use negative numbers for clarity; if you replace the negative
values with 256+value, e.g. -5 becomes 251, then you can avoid the IF
statement and just poke the value directly.)

    100  TEXT : NORMAL : HOME 
    200  &  NEW : &  HGR : VTAB 21
    210  &  HCOLOR= 3
    500  REM ARRAY TEST
    510 AD = 768: REM $300
    520  READ D: IF D = 1000 THEN 560
    530  IF D < 0 THEN D = 256 + D
    540  POKE AD,D:AD = AD + 1: GOTO 520
    560  &  PLOT 768,776,6: &  AT 50,50: &  PLOT 768,776,6
    570  POKE 768,256 - 10: POKE 769,256 - 20: &  PLOT 768,776,6 AT 100,50
    600  DATA -5,-10, 5,-10, -5,10, 5,10
    610  DATA 0,2, 0,1, 1,3, 2,3, 0,3, 1,2, 1000  

This draws the shape twice, once at the middle of the screen, once centered
at 50,50.  It then adjusts the top-left coordinate, and draws the shape
centered at 100,50.  Looking at the output, you can see that the top-left
corner of the third instance has moved, and all three lines from that
point have moved with it.

If a vertex ends up off-screen, lines that use that vertex are omitted
(not clipped).  If you tried to draw the example shape at (0,0), nothing
would happen, because every line has at least one point that would be
off-screen -- only point #3 is still visible, and all of the lines that
use that point extend off screen.

You can specify a maximum of 128 vertices and 128 index pairs for a
single call.  If none of the line segments share vertices, you'll need
two vertices per line, which means a cap of 64 lines.

#### Performance ####

There isn't a whole lot to it -- it just feeds the lines to DrawLine.
The key speed advantage is the removal of the Applesoft overhead.


<div id='ideas'/>

## Enhancement Ideas ##

Some ideas for future versions of fdraw.

### fdraw ###

Line clipping would make the array-draw function more useful for
animation projects.  If we accepted signed 16-bit values as input to
the clip function, we could specify an AT point outside the screen bounds.
That could be extended to circles, which could have off-screen centers.

A "game line" function or line mode that restricts coordinates to 0-255
and ignores color might be worth an experiment.

Triangle rasterization is possible, but perhaps a bit silly.

We could handle ellipses, but they're more complicated than circles, and
are slower to compute -- you need a couple of multiplications during
setup, and the asymmetry means you have to compute a quadrant rather
than an octant.  If the goal is fast animation rather than general-purpose
picture painting then there's little value in supporting ellipses.

Some of the inner loops are almost certainly paying an extra cycle to
cross a page boundary.  That's not easy to fix without adding absurd
amounts of padding.

The core FillRaster unrolled loop works for color or black & white,
but the latter doesn't require bit-flipping for odd/even bytes.  Having
a second loop would allow 8 cycles per byte rather than 10 when the
color is black or white, and would avoid the overhead incurred by
color changes (which must rewrite the EORs), for a cost of ~120 bytes.

"USE_FAST" could be applied more aggressively to reduce the size.

Having "fast" vs. "small" builds was mostly an experiment to see how
much of a difference in size and speed we'd get by dropping some of
the more expensive operations.  Another way to reduce size would be to
make the build modular, so you could (say) omit circle drawing or only
include line drawing.  Some trade-offs would have to be made, e.g. if
you only wanted line drawing then it makese sense to disable (or replace)
the horizontal-line optimization that calls FillRaster, as that requires
some sizeable tables that would otherwise be unused.

### Amperfdraw ###

The Amperfdraw API is somewhat minimal and could be improved.  Taking a
cue from Beagle Graphics, the rect and circle calls should probably look
more like:

    &DRAW width,height [AT left,top]
    &COS radius [AT left,top]

The "&AT" coordinate, currently only used by &PLOT, should be more
widely used.  Not only is it more convenient, it's also slightly faster,
since we don't have to parse the left/top coordinates each time.

The existing code is (somewhat lazily) using the Applesoft routines to
parse coordinates, which includes the range check.  We wouldn't be able
to use them for width/height, because we would need to take values in the
range (0-280, 0-192), where width/height of zero means "draw nothing".

I deliberately used Applesoft tokens, rather than arbitrary words, to
make commands simpler to parse.  Some of them don't fit that well.  COS
and SIN are circle-related, but it's not obvious which is outline and
which is filled.  DRAW and XDRAW don't really sound like rectangle-draw
calls, and would be much more appropriate if used to set the line draw
mode.  Spending a few bytes & cycles to get better names might be
worthwhile.

It's possible to store &PLOT arrays in actual BASIC integer arrays,
which might make them easier to code for.  The fact that arrays are
DIM()ed once, cannot be resized, and cannot be discarded makes them
difficult to use for dynamic data.

Currently &PLOT takes a list of vertices and a list of line segments.
We could also support "continuous line" mode, where it just plays
connect-the-dots (saves space, doesn't really affect speed).  Being
able to embed color changes could be handy.

&PLOT handles lines and vertices the way Applesoft does, with inclusive
coordinates.  This results in overdraw when vertices are shared.  This
is a (small) performance hit, and causes graphical glitches when connected
lines are drawn in "xdraw" mode.


<div id='additional-notes'/>

# Additional Notes #

Getting into the gory details here.

## Setting a pixel ##

Hi-res pixels are curious creatures.

Pixel color values are determined by adjacent bits.  The various drawing
routines only set one bit at a time, so "drawing" in green (hcolor=1) will
cause bits to be set in odd columns, cleared in even columns.  We don't
touch adjacent bits, so drawing purple (hcolor=2) in column 0 and green
in column 1 will produce a white line, while drawing them with the columns
reversed will produce a black line.

Making life more complicated is the use of the high bit in each byte, which
affects the color.  If you draw a purple line in column 0, and a black1
line with hcolor=4 in column 6, the purple line turns blue, because the
black1 line sets the high bit.

To set a bit at an arbitrary X offset, we need to do the following:

(1) Determine which byte to change (xc / 7) and which bit (xc mod 7).
(2) Determine the color mask for that byte.  For green, it's 0x2a
    (00101010) in even columns, 0x55 (01010101) in odd columns.
(3) Set or clear the target bit and the high bit, leaving the others
    intact.

One way to do this is illustrated below.  Assume we're drawing a green
line at X=17.  There's already a green dot at X=15, which gives us a
bit pattern of 00000010.  (Bits are "backwards", i.e. the bit on the
right is the pixel on the left.)

    LDY byteoffset                    X=2
    LDX bitoffset                     X=3
    LDA bitmask,x                     A=0x88 (10001000)
    STA <andmask              
    LDA oddevencolor,y   4 cyc        A=0x2a (00101010)
    EOR (hbasl),y        5 cyc        A=0x28 (00101010 ^ 00000010 = 00101000)
    AND <andmask         3 cyc        A=0x08 (00101000 & 10001000 = 00001000)
    EOR (hbasl),y        5 cyc        A=0x0a (00001000 ^ 00000010 = 00001010)
    STA (hbasl),y        6 cyc

As a second example, here's how we plot a black1 (hcolor=4) point at X=6
when there's a purple point (hcolor=2) at X=0 (00000001).

    LDA bitmask,x                     A=0xc0 (11000000)
    STA <andmask              
    LDA oddevencolor,y   4 cyc        A=0x80 (10000000)
    EOR (hbasl),y        5 cyc        A=0x81 (10000000 ^ 10000001 = 00000001)
    AND <andmask         3 cyc        A=0x81 (00000001 & 11000000 = 00000000)
    EOR (hbasl),y        5 cyc        A=0x81 (00000000 ^ 10000001 = 10000001)
    STA (hbasl),y        6 cyc

Note the purple pixel is still set, but now the high bit is as well,
changing it to blue.

The trick is to start with the color pattern, which specifies how we want
the bits to be set or cleared.  We EOR in the screen, which causes the
bits in A to be inverted wherever they were set on the screen.  Next we
use the AND mask to zero out the bits we don't want to update on-screen.
When we do the second EOR from the screen, the bits we just zeroed will
take on the values from the screen, while the bits we didn't zero will
return to their original values from the color pattern (because EORing
twice with the same value restores the original).

It might look a little nicer if we always set two adjacent bits.  That
would avoid the phenomenon where drawing from 0,0 to 0,10 in green doesn't
appear to do anything.  For 6 out of 7 pixels this is easy, a simple
adjustment to the bitmask, but for the 7th pixel we'll need to update an
adjacent byte... unless it's the rightmost byte, which would cause us to
overflow and wrap around (or write into a screen hole).  GraFORTH
renders lines this way, avoiding the overflow issue by limiting the X
coordinate range to (0,255).

To implement "xdraw" mode, where instead of setting pixels we invert
the current value, we can just omit (or NOP out) the first EOR.

We could draw faster if we simply set the new bits, rather than setting
some and clearing others according to the color mask.  This could result
in some odd behavior, e.g. drawing a horizontal green line over a
horizontal purple line would result in a white line.  Given how strange
things are in general this might not be an issue.

For 3D games like Stellar 7 or Elite, which essentially draw thin
monochromatic lines, we can drop the color mask and just set the bit on
the screen.  Plotting a pixel is then simply:

    LDA (hbasl),y        5 cyc
    ORA <bitmask         3 cyc
    STA (hbasl),y        6 cyc

This cuts the cycle count from 23 to 14.  It's also not necessary to
worry about the high bit, which can save a few cycles when shifting
the bitmask.  Most games are also able to limit the "active" part of
the screen to fewer than 255 pixels, which eliminates some 16-bit math
during setup.

For "xdraw" mode, the "ORA <bitmask" becomes "EOR <bitmask".


## Single- or Double-Buffered Animation ##

Because the Apple II has two hi-res graphics pages, it's possible to
double-buffer the animation to reduce or eliminate flicker.  The
application displays one page while erasing and redrawing the other.

In most cases it's faster to erase the entire screen with the Clear
function than it is to draw over with black.  For example, consider four
diagonal lines in a diamond shape, 100 pixels on a side.  Diagonal
lines are the most expensive, as each step requires advancing in
both vertical and horizontal directions.  The current implementation
needs about 80 cycles per diagonal pixel, or 100 * 4 * 80 = 32,000 cycles
to draw four medium-length lines (ignoring the setup cost for each line).
If you assume that the average cost to draw a pixel is about 70 cycles,
you can draw 570 pixels in the time it takes to erase the full screen.

We can clear the entire screen in about 40,000 cycles.  If the drawing
area is smaller, a custom clear routine could do it in even less.
(Imagine your drawing routines keep track of the highest and lowest
line that anything touches, and then just erase the "dirty" lines.) So
unless you're doing relatively light rendering, you'll get the best
performance by wiping all or part of screen rather than drawing over the
previous contents.

The &INVERSE command is intended to make double-buffered animation
easier from BASIC.  Use &HGR2 to switch to full-screen mode, then call
`&SCRN(1):&HCOLOR=0:&CLEAR` to select page 1 and clear it.  Draw your
first frame, then call &INVERSE to display page 1 and select page 2
for drawing.


An alternative approach is exemplified by Elite.  The game only uses
one hi-res page, but doesn't noticeably flicker (though distant objects
sort of "sparkle").  Suppose you're writing a similarly line-oriented
game, and your rendering cycle looks like this:

 - Step 1: draw over previous content with black
 - Step 2: draw new content with white

Your game will flicker badly without double-buffering, because there will
be a few display refresh periods where most of the lines have been erased.
Suppose instead you did this:

 - For each line in the shape, erase the old line, then draw the line in
   its new position

Now you might get some flickering on certain lines if the beam crosses
them while they're black, but the shape as a whole will be visible most
of the time.  The trouble with this approach is that, if your shape is
moving across the screen, you'll be drawing black over some recent white
lines, causing some distracting artifacts.

The way to make this work is to use "xdraw" mode, where bits are toggled
rather than set or cleared.  If you draw a new line across an old line that
will soon be erased, the crossing point is cleared.  When the old line
is erased, the crossing point is set white again, so your new line
appears unbroken.

It should be noted that this works well for Elite because they use backface
elimination, so lines within a single shape don't cross.  It's also
important to avoid re-drawing points at shared vertices, or your corners
will disappear unless there are an odd number of lines.

If there's very little on screen, this could be faster than a full clear.
Mostly it's of value if you need the 8KB occupied by the second hi-res
page for something other than output.


## Vertically-Challenged Rasterization ##

As noted earlier, we can clear the screen in about 40,000 cycles with
the Clear function, but drawing a screen-sized filled rectangle takes
about 96,000.  Why the difference?

The FillRaster function handles one horizontal line at a time.  For
each line it sets any pixels sticking out on the left and right edges,
and then it jumps into an unrolled byte-stomp function that blasts
its way through the middle at 10 cycles per byte.  Compare this to the
Clear function, which only needs 5 cycles per byte.

The trick to improving the speed at which we draw filled rectangles
is to make it more like the Clear function, which operates on columns
rather than rows.

Suppose, for example, we figured out which bits we need to set on the
left edge, and then applied them to every row.  Then we did the same
for the right edge.  The set-up cost for each edge went from
(N cycles * Y rows) to (N cycles).  Can we apply this to the middle
byte as well?

It turns out we can.  The fundamental problem with setting bytes
horizontally is that we have to index off of a direct page register,
e.g. "STA ([hbasl),y".  The only ways around this either add too much
loop overhead, too much setup overhead, or require too much memory.
For any given line, we need to find the base address, and issue a
6-cycle indirect store, followed immediately by an increment of the Y
register.  If we're drawing in color it's worse than that, because we
also have to exclusive-OR the color because the bit pattern flips for
odd/even columns.

We're much better off unrolling vertically.  Suppose you have 192
"STA abs,y" instructions, one for each row, one after the other.  You
no longer need the base address lookup, because it's baked into the
code, and since we're only touching one column we don't need to worry
about odd/even color values here.  To use this to draw rows 50-100, you
would replace the STA in row 101 with an RTS, and then JSR to the 50th
STA instruction.  After the row is painted, you increment Y, exclusive-OR
the color value, and jump through again.  (You can make this a little
faster by JMPing in and out instead, but you pay a bit more for setup
and cleanup, especially when you have to restore the base address that
got overwritten by the JMP.)

With this change we're working at 5 cycles per byte, plus the loop
overhead.  A full-screen FillRect will be about as fast as a Clear.

There are a couple of down sides.  First, you need 192 * 3=576 bytes to
hold this pile of store instructions.  If you're drawing a lot of filled
rectangles, though, the 2x speed improvement would make the size penalty
worthwhile.  The other problem arises if you use double-buffered animation,
as the table is hard-wired to page 1.  You can either spend a couple
thousand cycles when the page flips to rewrite the addresses, or you can
have a second full copy of the stores for page 2.

The current horizontally-focused implementation uses 256 bytes for its
unrolled code area, but you wouldn't be able to get rid of that by
switching to the vertical approach.  The reason the code works the way
it does is that it's designed to render circles, and those are hard to do
vertically.  With horizontal rasters, when you look at the left and right
edges you only need to examine the current row, and set pixels in a
single byte.  With vertical strips, each byte spans seven columns of
pixels, so the top and bottom "edges" might be several bytes deep.  The
code would have to iterate in "edge space" until it reached the meaty
center, and the cost of doing so would likely erase the benefit of vertical
fills until your circles got reasonably large.

It's possible that a hybrid approach, in which selected rectangles in the
center of a large circle are drawn with a fast vertical fill, could be
used, with slower code rendering the outer edges.  The trick would be to
come up with an approach that doesn't leave gaps, minimizes overdraw, and
is sufficiently faster to make the effort worthwhile.

