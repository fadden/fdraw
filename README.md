fdraw
=====

Fast graphics routines for the Apple II  
By Andy McFadden  
Version 0.3, August 2015

## Overview ##

The fdraw library provides fast rendering of points, lines, rectangles,
and circles, as well as high-speed screen clears, for Apple II hi-res
graphics.  It can be used from Applesoft or 6502 assembly language.

Two disk images are available in the [fdraw-disks](fdraw-disks.zip) zip
archive.  `fdrawdemo.do` is a 140K disk image with the demos that will
run on an Apple ][+ or later.  `fdrawdev.po` is an 800K disk image with
the source code, demos, and a few extras.

A video of the demos running in the AppleWin emulator
[is available](https://www.youtube.com/watch?v=z2RFGVoaROE).

Learn more about how fdraw works in the
[library documentation](docs/manual.md).  In addition to documenting the
API, it provides an overview of Apple II graphics, an assessment of the
performance of each of the routines, and some ideas on how the speed
could be increased (usually by spending more memory).

Learn about the demos in the [demo documentation](docs/demos.md).

Learn more about what possessed me to write a graphics library for the
Apple II more than 20 years after the platform was discontinued in the
[fadden's brain documentation](docs/personal-notes.md).

The main bits of source code are accessible from git for easy viewing,
but the "official" home is the `fdrawdev.po` image.

All code is copyright 2015 by Andy McFadden.  All rights reserved.  The
source code is available under the Apache 2 license (a very friendly
open-source license).


### Version History ###

##### v0.1 March 13, 2006

No source code, just a demo with fast filled circles and screen clears.

##### v0.2 March 20, 2006

Polished up the sources and published.  This version implemented Clear,
FillRect, FillCircle, and FillRaster.

##### v0.3 August 21, 2015

Added DrawPoint, DrawLine, DrawRect, DrawCircle, and SetLineMode.  Various
size and performance improvements.

Added Amperfdraw to make Applesoft BASIC programming easier.

Added several more demos and tests.

Added documentation.
