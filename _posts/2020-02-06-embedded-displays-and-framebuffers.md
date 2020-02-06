---
layout: post
title: Embedded displays and framebuffers
---

This post tries to explain the process of drawing picture on a simple display,
together with related concepts.
Mostly embedded displays and framebuffers are covered, desktop stuff like HDMI
and GPUs are out of scope.

## Receving picture data
Display is effectively a rectangular matrix of dots, with a quickly
changing pictures (30+ times a second). A bit surprisingly, display is updated
pixel-by-pixel instead of some sort of all-at-once, which happens blazingly fast
(e.g. refreshing ~1 million pixels 30 times a second brings us below a
microsecond per pixel).

Imagine going in the same order as reading a book: left-to-right, then going
to the next line, repeating until entire page is read.
Similarly for drawing a picture:
0. Start with a top left corner.
1. Draw a pixel.
2. Move to the next one (right or next row).
3. Repeat steps 1 and 2 until entire frame is drawn.
4. Go back to the top left corner.

Each pixel is represented as a triple of red, green and blue values, constantly
transmitted to display. So, an embedded display might need the following wires:
- pixel data, split into RGB channels (typically 16 or 24 bits/wires in total).
  Pixel data represents current pixel, for example black pixel would be all
  zeros (all wires low).
- clock signal, to indicate that now it's time to move to the next pixel.
- horizontal sync, to indicate that row is over and it's time to move to the next one.
- vertical sync, to indicate that frame is over and it's time to go back to top-left corner and start over.

## Driving the display
So hopefully by now receiving end, that is display, should be (more) clear, 
let's therefore turn to sending end.

With sub-microseconds speeds, driving all the signals usually requires dedicated 
hardware. But, just for completeness, for really small resolutions driving all
the lines from SW (bit-banging) is possible.

In any case, driving a display requires data, and we arrive at the notion of 
framebuffer, which is effectively a chunk of memory containing pixel data
in the same order (left-to-right, top-to-bottom) as order of drawing pixels on
the screen.

Where HW is there's usually a driver, so let's outline some ot its duties using
an example of i.MX eLCDIF driver (drivers/video/fbdev/mxsfb.c):
- allocate framebuffer memory
- configure HW accordingly to device tree and system settings
- implement fbdev subsystem callbacks
- handle IRQs if necessary (e.g. vsync complete)

## Misc details
The above display with lots of wires is usually called a parallel display, of
which there are several flavors (MPU, 8080, DOTCLK, etc). Main drawback is high
number of wires, so there are also serial (e.g. SPI) displays. However they
usually work for small resolutions only.

Actually display works with a rectangle larger than framebuffer, which can be
seen as framebuffer surrounded by "paddings". Those are called porches
(front/back/etc) and are configurable and hw-specific.

Chip that actually drives the picture can be part of host CPU (like eLCDIF in
the example above), or be bundled together with display itself (along with
framebuffer memory). Sometimes display configuration is performed throught
serial interface, while picture is driven via parallel.

There are also "smart" displays, where only portion of the screen that has
actually changed needs to be transmitted.

For a very skilled (or should it be called genius?), example of converting serial
to parallel, see [this](https://spritesmods.com/?art=spitft&page=1).
