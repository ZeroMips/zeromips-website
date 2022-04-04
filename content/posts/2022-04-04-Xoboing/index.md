---
title: "Xoboing demo"
date: "2022-04-04"
cover: "cover.jpg"
---

In the last blog post I described how the background image for the demo gets
rendered to the screen. Next target is to render the ball.
The background image is rendered as a bitmap, which would not be a good idea for
the ball. The ball is made from tiles, 8x8 pixels in size. Moving these tiles on
the screen is much easier than moving individual pixels.

I was really fascinated by the way the rotating ball effect was done.
Since calculating a 3D rotating sphere  in realtime was out of question when the
original demo was presented on the Amiga and probably still is today (contrary
examples welcome) it is done with a trick. The ball image is rendered in sectors.
Each sector gets a different color index. The colors for the sectors are defined
in palettes. So the palette defines which sector gets rendered in white and which
gets rendered in red. By swapping palettes you can create the illusion of rotation.
For illustration, a picture with the sectors colored in different shades.

{{< imgproc sectors Resize "1000x" "I see colors" />}}

Tile data and palettes are pre rendered and RLE compressed and directly written
to VRAM at runtime. The first shot was (as usual) a little lacking.

{{< imgproc firstshot Resize "1000x" "Interesting..." />}}

So we have transparency instead of red, which was not expected. Just for fun I
started rotating palettes.

{{< youtube 2yqg9M2u6ts >}}

Interesting, also the parts of the screen that are not filled with ball tiles get
changed. First of all I got byte order in the palettes wrong, so red got transparent.
Fixing this resulted in:

{{< youtube qByllDhbdpM >}}

So colors are alright, but still the rest of the screen changes. So let's have
a look at the palette layout. The first color in the palette is transparent and
is used for all non-ball tiles. The second color is translucent and is used for
the ball shadow. The other 14 indices colors are red/white for ball rotation.
There was a bug in my code that got the palette base offset wrong. This resulted
in indices 0/1 getting red/white colors assigned. With this fixed, things started
to look better.

{{< youtube nAct785i1BI >}}

This still doesn't look perfect because there is some tearing. This is to be expected
since there is no synchronization to video output. The XR_SCANLINE register
can be used to wait for VSYNC. This was the first time I actually read a register
from Xosera, but it worked nicely, as you can see here.

{{< youtube I1S_KCjjZzs >}}

Xosera has an Amiga-inspired video synchronized Coprocessor which is named "Copper".
Together with the Blitter, moving the ball and switching palettes can be done
without any CPU usage.
The actual copper sourcecode looks like this:
```
[0x0000] =  COP_MOVEC(vram_base_b, 0x0041 << 1 | 0x1),                                      // Fill dst
[0x0001] =  COP_MOVEC(0, 0x0106 << 1 | 0x1),                                                // Fill scroll
[0x0002] =  COP_MOVEC(MAKE_GFX_CTRL(0x00, 0, XR_GFX_BPP_4, 0, 0, 0), 0x0107 << 1 | 0x1),    // Fill gfx_ctrl with colorbase
            COP_JUMP(0x0040 << 1),

[0x0040] =  COP_MOVEC(COP_MOVEC(0, 0x0104 << 1 | 0x1) >> 16, 0x0041 << 1 | 0x0),    // Make following move point to dst
[0x0041] =  COP_MOVEC(0, 0),
            COP_MOVEC(COP_MOVEC(0, 0x0101 << 1 | 0x1) >> 16, 0x0041 << 1 | 0x0),    // Make preceding move point to prev_dst

[0x0043] =  COP_JUMP(0x0080 << 1),  // Jumps either to blitter load or to wait_f
[0x0044] =  COP_MOVEC(COP_JUMP(0x0080 << 1) >> 16, 0x0043 << 1 | 0x0),  // Make branching jump go to 0x0080 << 1
            COP_WAIT_F(),

[0x0080] =  COP_MOVEC(COP_JUMP(0x0044 << 1) >> 16, 0x0043 << 1 | 0x0),  // Make branching jump go to 0x0080 << 1
            COP_JUMP(0x00C0 << 1),

            // Load fixed blitter settings
[0x00C0] =  COP_MOVER(XB_(0x00, 8, 8) | XB_(0, 5, 1) | XB_(0, 4, 1) | XB_(0, 3, 1) | XB_(0, 2, 1) | XB_(1, 1, 1) | XB_(0, 0, 1), BLIT_CTRL),
            COP_MOVER(0x0000, BLIT_MOD_A),
            COP_MOVER(0x0000, BLIT_MOD_B),
            COP_MOVER(0xFFFF, BLIT_SRC_B),
            COP_MOVER(0x0000, BLIT_MOD_C),
            COP_MOVER(0x0000, BLIT_VAL_C),
            COP_MOVER(WIDTH_WORDS_B - BALL_TILES_WIDTH, BLIT_MOD_D),
            COP_MOVER(XB_(0xF, 12, 4) | XB_(0xF, 8, 4) | XB_(0, 0, 2), BLIT_SHIFT),
            COP_MOVER(BALL_TILES_HEIGHT - 1, BLIT_LINES),

            COP_WAIT_V(480),
            COP_JUMP(0x0100 << 1),

            // Blank existing ball
[0x0100] =  COP_MOVER(vram_base_blank, BLIT_SRC_A),
[0x0101] =  COP_MOVER(vram_base_b, BLIT_DST_D),         // Fill in prev_dst
[0x0102] =  COP_MOVER(BALL_TILES_WIDTH - 1, BLIT_WORDS),

            // Draw ball
[0x0103] =  COP_MOVER(vram_base_ball, BLIT_SRC_A),
[0x0104] =  COP_MOVER(0, BLIT_DST_D),                   // Fill in dst
[0x0105] =  COP_MOVER(BALL_TILES_WIDTH - 1, BLIT_WORDS),

[0x0106] =  COP_MOVER(0, PB_HV_SCROLL),                 // Fill scroll

[0x0107] =  COP_MOVER(0, PB_GFX_CTRL),                  // Fill gfx_ctrl with colorbase

            COP_JUMP(0x0003 << 1),
```

This program is assembled by the C preprocessor. There is also a standalone assembler
for copper code in the Xosera repo.
The code is run every frame. Copper only knows four main instruction: `WAIT` (for a
position on the screen), `SKIP`(instructions depending on screen position),
`MOVE` (immediate to register, but not VRAM) and
`JUMP`. As `MOVE` can write to copper memory, self modifying code is possible. The
xoboing code is self modifying, have a look at address 0x0040 to 0x0080.
The copper binary is RLE compressed and directly written to VRAM.
Parameters are written into the copperlist at runtime.

With all this prepared and working, the next project is getting a little physics
engine implemented. Different from the m68k version, I will definitely not be
using floating point arithmetic.
