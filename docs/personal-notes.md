My Quest for Lines
==================

As far back as I can remember, I always wanted to draw lines on the
hi-res screen.

This probably started when I saw Battlezone in the arcades in the early
1980s.  I still think the game is beautiful -- a first-person shooter
reduced to the essential elements.  I wanted to write something similar
for the Apple II, but I didn't know where to start.  (I should probably
mention that I was 11 years old in 1980.)

Battlezone had a dedicated matrix processor (the "math box"), and a
vector display that handled the line drawing.  The Apple II had neither
of those things, which meant that achieving the same level of performance
and graphical detail wasn't possible.  Despite those shortcomings, Damon
Slye create a pretty solid Battlezone-ish game in 1983, called Stellar 7.
A couple of years later, Braben and Bell made another compelling wireframe
combat game, the space sim Elite.  (The A2-FS1 flight simulator
came out much earlier, but the graphics were blinky, enemies were just
dots, and the action was much slower-paced.  Of course, it loaded from
cassette tape and ran in 16KB, so they didn't have much to work with.)

Seeing these games showed me that the problems could be solved.  I decided
that the place to start was line drawing, because (a) line drawing is
pretty fundamental to wireframe 3D, and (b) I wasn't getting the performance
I needed out of HPLOT TO.

Somewhere in the mid-1980s -- I was in high school now -- I began by trying
to figure out how line drawing worked.  Suppose, for example, you want to
HPLOT 0,0 TO 19,5.  How do you decide which pixels to set?

I wrote a program (which I recently found) called "HPLOT SIMULATOR".  It
computed the ratio of vertical to horizontal pixels (e.g. 20 / 6 = 0.3),
and marched horizontally across the screen, adding the fractional value to
the Y coordinate at each step.  The result was a pretty good-looking line.

The trouble was that it used floating-point math and required division,
things that the 6502 is not very good at.  It occurred to me that division
can be performed as a series of integer subtractions.  (It probably occurred
to me because I didn't know any other way to divide on the 6502, not having
encountered the shift-and-subtract approach yet.)  So if you initialize a
counter to zero, and add 6 to it each time you move horizontally, then when
it reaches 20 you know it's time to move vertically.  Subtracting 20 from
the counter resets it, but retains the division remainder as the starting
point, so you don't lose the fractional part.

When I went to college I took a graphics class, and was introduced to
Bresenham's classic line algorithm.  This was essentially the same as what
I'd figured out for myself, but with two refinements: (1) it used signed
values, allowing a slightly cheaper "< 0" comparison, and (2) it started
with the counter half full, correcting the slight lopsidedness of my lines.

The graphics class inspired me to write a 3D game library called Arc3D
in 1990.  I used it to create a
[pair of demos](https://www.youtube.com/watch?v=Oe0x425Yiq4): "Not Modulae",
which animated several 3D shapes on the screen, including a pair of ships from
Elite; and "Not Stellar 7", a graphics demo that let you drive around
(and, sadly, through) some tanks from Stellar 7.  The Arc3D library was
written for the IIgs, in 65816 assembly, and used the super-hi-res screen.
Having a better CPU, lots more memory, and a less-quirky graphics
architecture made things easier than doing the same on a classic Apple II.

I wrote my own super-hi-res line drawing code, of course, but a year later
when I disassembled somebody else's demo I found better code.  Which, it
turned out, they had also lifted from another source, an FTA demo.  I
dropped mine and used theirs.

After I graduated from college, my side projects tended more toward data
compression and Netrek, so Arc3D was never improved upon.

Fifteen years later, in 2006, there was a discussion on a Usenet group
about circle rendering.  Once upon a time I'd drawn circles from BASIC
with trig functions, but it was painfully slow, which made me wonder
about a part of the game Horizon V where you steer through a series of
circles.  I wanted to try it for myself and see what it would take.
(Looking at a youtube video of Horizon V, the animation is more radial
than circular... I suspect it's not really drawing circles at all.)

I first announced my results in a
[comp.sys.apple2.programmer](https://groups.google.com/forum/#!msg/comp.sys.apple2.programmer/Vj_xVjMHaR0/cLU3t2TlPrMJ)
posting.  I had focused on filled circles, rather than outline circles,
since that seemed like a more interesting challenge.  The "fdraw" demo
supported fast rendering of filled circles, filled rectangles, and had
a very fast screen clear.  A week later, after a bit of cleanup, I
[released the fdraw v0.2 sources](https://groups.google.com/d/msg/comp.sys.apple2.programmer/Un4pV5p8Elw/6qZVAPc_da0J).

It occurred to me at the time that this would be a handy place to stick
the hi-res line drawing code I'd always wanted to write.  Somewhere around
this time I also sort of poked at the idea of writing a dedicated hi-res
graphics compression program.

Fast forward another nine years, to 2015.  After learning about the LZ4
format, I went back to my data compression roots and wrote
[fhpack](https://github.com/fadden/fhpack) and some demos.  I had so much
fun doing it that I decided it was finally time to write some hi-res
line drawing code.

Being older, wiser, and having easy access to relevant information, I
began with the appropriate chapters in Michael Abrash's _Graphics
Programming Black Book Special Edition_.  This covered the standard
algorithm, but also had a chapter on a faster "run-slice" approach.
This intrigued me, because instead of the usual "step right, check if
it's time to move down, step right, check if it's time ..." logic, it
says, "figure out how long each line segment is; then, move right 3
times, step down, move right 4 times, step down, ...", saving a lot of
redundant computation.  The trouble is that it requires fixed-point
division, and drawing N adjacent pixels is tricky when your graphics
architecture has 7 horizontal pixels per byte.  You'd have to be a bit
crazy to try to get that to work.

So I went with a standard approach, and used the Applesoft ROM method of
coloring pixels (discussed in the fdraw docs).  I carefully optimized
the code, and squeezed out as much performance as I could.

When I was done, I began looking around at what other people did to see if
there were any tricks I missed.

I looked at the Applesoft ROM code.  Very clever, but very much optimized
for space over speed.  Also, because it's in ROM, self-modifying code is
not possible, so they lose a cycle here and there.

Next I looked at GraFORTH.  I figured out how functions were arranged,
identified the plot function, and disassembled it with CiderPress.  It uses
a pretty standard algorithm, but supports multiple drawing modes and sets
two adjacent bits for better-looking colored lines.  Good use of
self-modifying code, but some choices were made to reduce code size at the
expense of speed.  My code was faster.

Next I looked at Elite.  Digging through memory after the program had
loaded, I found a collection of purpose-built line functions.  Some drew
color, most used EOR to "xdraw" monochrome lines.  Standard Bresenham
approach, with a bit of variation on the Y-lookup table -- their table is
only 24 bytes (1/8th of the screen), and they use a quick "add 4 to the
high byte" 7 out of every eight lines.  I tried applying this to my code,
but it turned out that just using a full lookup table was a tiny bit faster.
Their code was about the same as mine, but somewhat faster because they
treated the screen as monochrome rather than color.

Next I looked at Stellar 7, one of my earliest inspirations.  I scanned
through some files with CiderPress, looking for anything line-draw-esque.
(If you spend enough time drawing lines you start to see patterns in the
code.)  After about five minutes I found the code, in the same file as this
gigantic unrolled division routine.  But as I started to dig into the code
I noticed that it was using a count oddly, and this one function was...
HOLY CATS he did run-slicing.

And he did it big.  There are several line-draw functions, all of them padded
out to live on a single page (so that none of the branches cross page
boundaries, which costs an extra cycle).  It has the usual special cases --
simple horizontal and vertical lines -- and the usual split between
vertically-dominant and horizontally-dominant lines.  But there are *three*
different functions for drawing mostly-horizontal lines, selected based on
slope, all of which try to set multiple horizontal pixels at once.  The
slope of the line affects how the code is structured; for example, for
very shallow lines it expects that it will often be able to set an entire
byte at once.  Color is not supported, so pixels are set with a simple
OR operation.

It's very impressive, and a wee bit terrifying.  But when you're making
a game that will be spending much of its time drawing lines, you really
want to optimize those draw functions.

The tricky part is that divide.  The division routine is unrolled to a
healthy 187 bytes long, and might take 240 cycles to run.  For short
lines and mostly-vertical lines it might have been more efficicent to skip
the division and just use a run-length implementation, but the ability to
set multiple bits at once for mostly-horizontal lines is a huge win.  It's
a fair bet that the code in Stellar 7 has the fastest line drawing
implementation for the Apple II.  (Of course, I haven't looked at Arcticfox,
the sequel...)

The general structure of the code was actually very similar to mine: always
draw left to right, use self-modifying code to handle up vs. down, and so on.
I didn't come away with any new ideas for optimizations to my run-length
implementation from this or the other programs I looked at... but there
are a lot of other games that I haven't disassembled.


So, 30+ years after HPLOT SIMULATOR, here I am with a bunch of code for
drawing lines on the Apple II hi-res screen.

I don't plan on writing Battlezone for the Apple II.  Stellar 7 did that,
and more.  My goal in developing fdraw was to scratch a very old itch.

I had forgotten how much fun this stuff is.  Working in ARM assembly
language on Android offered similar challenges, but you're never entirely
sure exactly how your code will perform on the wide range of CPU
architectures (affecting instruction interleave, cache size and
replacement policy, branch prediction, etc.), you have to guess at cache
misses and the success rate of data prefetching, and it's difficult to
measure results when there's multiple threads running and interrupts firing.
On the Apple II you can count every cycle, and know exactly what will
happen when.

I don't expect that anyone will find the code useful, but that wasn't
really the point.

Andy McFadden  
August 2015

