---
title: "Colorlight 5a-75b"
date: "2022-05-29"
cover: "cover.jpg"
---

I always fancied the idea of exploring the world of programmable logic a little
more, so I had the idea to use a FPGA for providing some peripherals to the 6502.
Obviously that is what the Xosera board is already doing for graphics.
But it would be nice, to also have PS/2 keyboard and mouse and an SD card interface.

So I came across the Colorlight 5a-75b board, which is meant for controlling LED
lines (as far as my understanding goes). The cool part is, that it features a
Lattice ECP5 FPGA which is quite beefy and lots of the pins are routed to
connectors on the board. As a bonus there is also SDRAM and gigabit ethernet
onboard. And all this for under 20$ on AliExpress. I saw that there were already
some projects using the opensource FPGA tools supporting this board. And there
was already lots of [documentation](https://github.com/q3k/chubby75/tree/master/5a-75b)
available So I quickly ordered one.

Later I found, that the FPGA IOs are connected using driver chips that are wired
as output only on the board.
Luckily Claude Schwarz had already found a way to replace the driver chips with
[bidirectional ones](https://twitter.com/Claude1079/status/1231194849350647808),
that could also used for 5V level adaption, since the FPGA IOs are 3.3V max.
You just have to connect a 1N4007 diode in series to you 5V power supply, so
you have a diode drop and the supply voltage then is 4.3V. If 4.3 V is placed on
the gate of the SN74CBT3245APW FETs, the maximum passable signal 
is 3.3 V (approximately 1 V less than the gate voltage). So you get 5V tolerance.

For connecting PMOD peripherals (PS/2 and SDCard is readily available in that form)
there is an even simpler [solution](https://github.com/Disasm/hc245t-bypass).
These flex PCBs can be ordered from OSH park for 2.80$ including shipping to
germany, which is really a bargain.
{{< imgproc flex Resize "1000x" "These are gorgeous" />}}

As I had all the parts collected and also some free time available, I started
modding the Colorlight hardware. Half of the IOs should be 5V compatible for
connecting it to my 6502, the other half should provide PMOD connectivity.

First of all I had to get rid of the 74HC245T levelshifters. I found this video
very inspiring:
{{< youtube CVsmwFAkf7I >}}

I gave it a try myself. Unfortunately I did not have the nice big chisel tip,
which made things more complicated.
{{< youtube 725ez4gnGx8 >}}

I hope you are not offended by my SMD soldering skills. For me this is a project
where I can practise. I hope I can encourage some people to give it a try too.
If I can do it, anybody can.

So this method basically worked. But it took ages. Since I do not need the 74HC245Ts
anymore, I decided to try cutting the pins with an exacto knife, which works
surprisingly well.
{{< youtube T58vn-xocHw >}}

After that you only have to remove the remaining pins by collecting them with
the tip of the soldering iron.
{{< youtube O7se4k1cuYE >}}

Soldering the flex PCBs was something I had never done before. The trick is to
apply some tin to the pads before laying them on the board. Then it goes
smoothly.
{{< youtube Zl-IGA6Ifec >}}

Soldering the SN74CBT3245APW was a little more challenging and I had to verify
that each pin was soldered properly.
{{< youtube TfkhNkY-EOI >}}

I am quite pleased with the result:
{{< imgproc finalboard Resize "1000x" "Finally..." />}}

So at that point I needed some help to get the GPIOs tested using Verilog code.
Luckily Xark gave me some help on the Zeromips discord channel. He wrote
a [gpio tester](https://github.com/XarkLabs/upduino-gpiotest) for the UPduino
board which is used for Xosera. I could [adapt](https://github.com/ZeroMips/Colorlight-5A-75B-gpiotest)
it to the Colorlight board.
After I removed some stupid copy and pasted typos, all the ports worked perfectly.

Here is a picture of the six GPIOs on one port tested sequentially:
{{< imgproc analyzer Resize "1000x" "Logic analyzer goodness" />}}

So I call this modification a success and can now start implementing something
useful with this board. There is already some PMODs waiting...
{{< imgproc pmods Resize "1000x" "This is a nice collection" />}}
