# Leia-Image-Format-Splitter #

## Purpose ##

Splits a 3D Leia Image Format (LIF) file into two separate JPEG files;
a left and right stereoscopic pair.  Optionally it can also extract
the two corresponding depth map images, and other binary and text data
found within the LIF file.

## Usage ##

`lif-splitter` is a command line application.

`````shell
$ lif-splitter [options] my-image.jpg [...]
`````
The default operation extracts only the left/right stereoscopic image pair
as ordinary JPEG files.

![LIF to JPEG L/R Pairs](docs/lif-to-jpeg.png?raw=true)

To view a complete manual page, use the `-manpage` option.

`````shell
$ lif-splitter -manpage
`````

For a list of available options, use the `-help` option.

## Requirements and Installation ##

This script requires a Perl version 5 language interpreter which is
typically pre-installed on most Linux systems and macOS up to and
including Big Sur 11.  To confirm if Perl is already installed on your
system, and is in your
[PATH](https://en.wikipedia.org/wiki/PATH_\(variable\)), use the
following terminal command:

`````text
$ perl --version

This is perl 5...
`````

Perl can also be easily installed on Windows, macOS, and Linux by
following this [step-by-step
guide](https://www.perl.com/article/downloading-and-installing-perl-in-2021/).

To install this utility, just copy the single file `lif-splitter` from
the project's `src` directory to [wherever you keep your
scripts](https://shapeshed.com/using-custom-shell-scripts-on-osx-or-linux/).
Sorry there is no automated installer at this time.

## Background ##

The Leia Image Format (LIF) is a proprietary
[stereoscopic](https://en.wikipedia.org/wiki/Stereoscopy) file format
used for storing so called "4-View" images designed to be displayed on
a device using Leia's "3D Lightfield Technology" [autostereoscopic
display](https://en.wikipedia.org/wiki/Autostereoscopy).  When this
was written, this includes the [Red Hydrogen One
smartphone](https://en.wikipedia.org/wiki/Red_Hydrogen_One) and [Leia
Lume Pad tablet
computer](https://www.cnet.com/tech/computing/lume-pad-brings-glasses-free-3d-back-again-on-an-android-tablet/).
Both of these devices is equipped with a 3D camera.  Photos taken with
this camera are saved as a single JPEG file using the proprietary LIF
file format to store the resulting left/right stereo image pair and
corresponding depth maps.  Conventional JPEG viewing software will
only 'see' the left view which is the first image in the data stream.
But when viewed with [Leia Inc's](https://www.leiainc.com/) own
software, all of the images are used for displaying the 3D image using
their lightfield hardware technology.

The fact that the LIF format contains a pair of conventional JPEG
images for the left and right views makes it is possible to extract
these as two independent JPEG image files.  This would he first step
in converting such pairs into an alternate stereoscopic file format
such as
[MPO](https://en.wikipedia.org/wiki/JPEG#JPEG_Multi-Picture_Format) or
[JPS](https://en.wikipedia.org/wiki/JPEG#JPEG_Stereoscopic), or as a
conventional side-by-side format image with a `.jpg` extension.  For
these formats the LIF format depth map images are not needed so they
are not extracted by default.

## Program Bugs and Limitations ##

- As currently written, this utility only works because at the time
  this was written (Feb 2022) LIF files created by the RH1 and Lume
  Pad do not contain
  [thumbnail images](https://entropymine.wordpress.com/2018/07/01/jpeg-thumbnail-formats/).
  Our simplistic parsing algorithm does not correctly handle a JPEG
  image that itself contains any embedded JPEG images, i.e. a thumbnail
  or preview image.  See the **Programming Notes** below for more details.

- As a safety measure, we try to avoid parsing pathologically large
  files which we currently define as any input that exceeds the byte
  count equivalent of four uncompressed 8K images.  However the `-all`
  option [puts a penny in the software
  fusebox](https://www.schererelectric.com/blog/electrical/the-penny-in-the-fuse-box-and-other-diy-electrical-panel-nightmares/)
  to ignore this limit.

## Programming Notes ##

This program takes a very naive approach in that it doesn't try to
understand the proprietary LIF format, nor even common JPEG formats
for that matter, beyond that fact that complete JPEG images
[are bracked by unique two byte tags](https://www.media.mit.edu/pia/Research/deepview/exif.html).
These tags indicated the start-of-image (SOI) and end-of-image (EOI).
In a LIF file, any data found between, or after these concatenated
JPEGs is discarded or can be optionally saved to an auxillary file for
later examination and study.

To perform the parsing of the binary byte stream, the following
[Finite-State Machine](https://en.wikipedia.org/wiki/Finite-state_machine)
(FSM) is used.  In this model each transition is made when the next
single byte is read from the input stream.

![LIF Parser](docs/parser-FSM.png?raw=true)

LIF Finite State Machine (FSM) Parser

"save" means write current image data byte(s) to the active image output file

"echo" means optionally write any non-image data to a 'discard' file.


Note that Q1 is the unique initial starting state.  Q1 and Q5 are the
expected halting states.  In particular reaching the end of the input
stream while in Q3 or Q4 is an error (truncated image).

The above [directed graph](https://en.wikipedia.org/wiki/Directed_graph)
can also be represented as the following state transition table.


`````text
+----------+------------+----------+----------+----------+----------+----------+
| Q1       | Q2         | Q3       | Q4       | Q5       | Q6       | Q7       |
| not      | start      | in       | leave    | end      | in       | leave    |
| image    | image      | image    | image    | image    | thumb    | thumb    |
================================================================================
| IN: ! FF | IN: ! D8   |          |          | IN: ! FF |          |          |
| - echo   | - echo FF  |          |          | - echo   |          |          | Q1
|          | - echo     |          |          |          |          |          | not
|          |            |          |          |          |          |          | image
+----------+------------+----------+----------+----------+----------+----------+
| IN: FF   |            |          |          | IN: FF   |          |          |
|          |            |          |          |          |          |          | Q2
|          |            |          |          |          |          |          | start
|          |            |          |          |          |          |          | image
+----------+------------+----------+----------+----------+----------+----------+
|          | IN: DB     | IN: ! FF | IN: other|          |          | IN: D9   |
|          | - new file | - save   | - save   |          |          | - save   | Q3
|          | - save FF  |          |          |          |          |          | in
|          | - save     |          |          |          |          |          | image
+----------+------------+----------+----------+----------+----------+----------+
|          |            | IN: FF   |          |          |          |          |
|          |            | - save   |          |          |          |          | Q4
|          |            |          |          |          |          |          | leave
|          |            |          |          |          |          |          | image
+----------+------------+----------+----------+----------+----------+----------+
|          |            |          | IN: D9   |          |          |          |
|          |            |          | - save   |          |          |          | Q5
|          |            |          | - close  |          |          |          | end
|          |            |          |   file   |          |          |          | image
+----------+------------+----------+----------+----------+----------+----------+
|          |            |          | IN: D8   |          | IN: other| IN: other|
|          |            |          | - save   |          | - save   | - save   | Q6
|          |            |          |          |          |          |          | in
|          |            |          |          |          |          |          | thumb
+----------+------------+----------+----------+----------+----------+----------+
|          |            |          |          |          | IN: FF   |          |
|          |            |          |          |          | - save   |          | Q7
|          |            |          |          |          |          |          | leave
|          |            |          |          |          |          |          | thumb
+----------+------------+----------+----------+----------+----------+----------+
`````

Find the current state at the top or the table; read down to find
which condition matches the current input byte.  For example `IN:
! D8` means the current input byte is not the `0xD8` character.
If the current input byte matches the condition, perform the
action described, and then read to the right to determine the next
state.  Return to the top of the table to find that state, continue
with the next input.

The beauty of using a FSM model is that the input loop can contain a
case or [switch
statement](https://en.wikipedia.org/wiki/Switch_statement) with a
branch defined for each state.  When a branch is taken, it can examine
the current input byte to determine the action to be performed, and
determine what needs to be the next state.  At any point in the input
stream, the only information needed is the what is the current state,
and what is the current input byte.  Using this approach, a FSM can be
almost mechanically transcribed into code.

The limitation of a FSM is revealed in the name itself -- finite.  ...

Fun fact, [regular expressions describe patterns which can be recognized by finite state machines](https://www.cs.drexel.edu/~kschmidt/CS360/Lectures/2.html).w

https://neilmadden.blog/2019/02/24/why-you-really-can-parse-html-and-anything-else-with-regular-expressions/

