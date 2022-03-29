---
title: "Xosera progress"
date: "2022-03-29"
cover: "cover.jpg"
---

Since Xosera is basically working, I wanted to see what I could achieve with it.
Thomas Jager did a very nice [job](https://gitlab.com/0xTJ/xoboing/) implementing 
the well known bouncing ball demo for the rosco_m68k project.
Having a look at the code I found lots of floating point math and trigonometric
operations which are not a perfect fit for the 6502. Most of the content is
static, so I decided to pre calculate the background bitmap and the tile data
for the ball. The resulting binary data has to be included in the ROM
and can then be transferred to Xosera VRAM. Together this is more than 50k in
size, so some kind of compression might come handy. RLE is pretty simple to
[implement](https://codebase64.org/doku.php?id=base:rle_pack_unpack) so I gave
it a try. After compression there is only about 16k left which is good enough
for me.

Running code resulted in an interesting image:
{{< imgproc bitorder Resize "1000x" "At least it looks vaguely like a grid" />}}

At least the beginning of the screen looked vaguely like what it should be,
but somehow the contents seemed to be mixed up horizontally. After a very helpful
comment from Xark about the bitorder (bit 7 and not bit 0 is  the first bit that
gets displayed in a line) this was an easy fix. Next the content looked vertically
compressed so I assumed that somehow only every second line gets painted.
(In the screenshot I have run the draw routine twice, so the whole screen gets filled).
The demo runs in 640x480 resolution, but the background is rendered in 320x240
only and then zoomed using the controls in the XR_PA_GFX_CTRL register.
But this actually means that lines are still 640 pixels in VRAM, but only the
first 320 pixels are shown. Which I did not consider in my code. After fixing
that, things looked a little better:
{{< imgproc partial Resize "1000x" "Better, but what is this stuff in the middle?" />}}

I could not quite explain, where the stuff in the middle of the screen was coming
from. I had verfied that my pre generated bitmaps were allright by converting them
to PNGs with ImageMagick. So I suspected that an address line in my setup was not
working properly. The ROM is emulated by 6502ctl, so I wrote a little test to check
them:
```
rS-- a00b a5 LDA $10
r--- a00c 10
r--- 0010 00
rS-- a00d a5 LDA $20
r--- a00e 20
r--- 0020 00
rS-- a00f a5 LDA $40
r--- a010 40
r--- 0040 00
rS-- a011 a5 LDA $80
r--- a012 80
r--- 0080 00
rS-- a013 ad LDA $0100
r--- a014 00
r--- a015 01
r--- 0100 00
rS-- a016 ad LDA $0200
r--- a017 00
r--- a018 02
r--- 0200 00
rS-- a019 ad LDA $0400
r--- a01a 00
r--- a01b 04
r--- 0400 00
rS-- a01c ad LDA $0800
r--- a01d 00
r--- a01e 08
r--- 0800 00
rS-- a01f ad LDA $1000
r--- a020 00
r--- a021 10
r--- 1000 10
rS-- a022 ad LDA $2000
r--- a023 00
r--- a024 20
r--- 2000 20
rS-- a025 ad LDA $4000
r--- a026 00
r--- a027 40
r--- 4000 40
rS-- a028 ad LDA $8000
r--- a029 00
r--- a02a 80
r--- 8000 00
```

In the 6502ctl output I could see, that the 6502 was accessing all the addresses
as expected. Then I implemented a 6502ctl-command that can stop execution when
an address is accessed from the 6502. So I could stop the program exactly at the
location where things went wrong. I was quite puzzled to see that the 6502 accessed
the right address at the right time but read the wrong value. Since this value comes
from the 6502ctl ROM emulation there had to be something wrong there. And indeed
it was:
```
#define ROMMASK                         (ROMSIZE - 1)

...

return pgm_read_byte(&rom[addr & ROMMASK]);
```

I had simply kept this code from the original implementation where ROMSIZE was 16k
without giving it any thought.
This means the index into ROM data is generated from the address on the bus
with a bitmask. For 16k it is `(0x4000 - 1) = 0x3FFF`, which is in binary `11 1111 1111 1111`.
For 24k it is `(0x6000 - 1) = 0x5FFF`, which is in binary `101 1111 1111 1111`.
As you can see there is a zero inside this bitmask, which means that the clever
trick to decode the address is only working for ROMSIZEs that are power of 2.
The solution is simple:
```
#define ROMMASK                         (ROMSIZE - 1)

...

return pgm_read_byte(&rom[addr - ROMBASE]);
```

So now that I have the background display working properly it is time to take
care of the ball but that is for another blogpost.
