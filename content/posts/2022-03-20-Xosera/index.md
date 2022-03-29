---
title: "The Xosera project"
date: "2022-03-20"
cover: "success.jpg"
---

The Ben Eater 6502 kit comes with a two line LCD character display. This is very
nice for doing some experiments. But I always thought how cool it would be to
do output to a real screen.

I finally found the Xosera project, which is an FPGA based graphics adapter.
It was originally designed for the rosco_m68k hardware, which uses the
Motorola 68000 and derivatives. Fortunately the Xosera board is designed for
an 8 bit databus, though internal registers are 16 bit. The project has a very
welcoming Discord channel (xosera-users on https://discord.gg/jNcvmRPn).
There is a nicely designed [PCB](https://www.tindie.com/products/rosco/xosera-fpga-video-r1/)
available from Tindie, the FPGA comes as a [module](https://tinyvision.ai/products/upduino-v3-0).

I have added a BOM for the german distributor Reichelt to the ZeroMips [hardware repo](https://github.com/ZeroMips/zeromips-hardware).
It includes everything needed but the GAL22V10. It also includes some spare parts
and also parts of the circuit that are not needed for connecting to the 6502.

Connecting Xosera to the CPU breaboard is actually pretty simple:
![Zeromips schematic](/img/zeromips.svg)

U2 is a PAL used as a simple chip select decoder. Sources can be found in the
[hardware repo](https://github.com/ZeroMips/zeromips-hardware).

IC4 on the Xosera needs a different configuration which is also included.

When you power up the Xosera board, you should get a grey screen in 640x480.
{{< imgproc grey-screen Resize "1000x" "grey screen" />}}

In the beginning I had the problem, that IC5 got quite warm. This was due to
a RGB LED connected to the FPGA pins that sunk their current into the poor thing.
Fortunately there is a jumper trace on the FPGA board that can be cut to
disable the LED. That worked nicely.
{{< imgproc verify-cut Resize "1000x" "verify cutting trace was succesful" />}}

To match the bus timing I had some logic analyzer fun.
{{< imgproc analyzer Resize "1000x" "logic analyzer fun" />}}

Fortunately the board has debug connectors to attach the logic probes.
{{< imgproc probing Resize "1000x" "logic probes" />}}

After all the timing looks pretty neat.
{{< imgproc scrprint Resize "2000x" "timing looks neat" />}}

After writing a small assembly program, I got this output:
{{< imgproc partial-success Resize "1000x" "not quite there" />}}

I was hoping to see the characters 'ABC' in the upper right corner instead,
so something was still wrong. It needed some more sleep on my side to find that
simply forgot to include the character codes that I wanted to output.

The sourcecode is included in the ZeroMips [playground repo](https://github.com/ZeroMips/zeromips-playground).

So some very nice progress here. Now I have an extremly powerful video output
for my system. Thanks to Xark for all the help while debugging this.
