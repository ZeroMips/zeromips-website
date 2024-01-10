---
title: "My new superpower"
date: "2024-01-08"
cover: "cover.jpg"
---

For a long time I wanted to improve my very basic FPGA skills. Last year I saw a
recommendation for the book "Getting started with FPGAs" by Russell Merrick,
who is running nandland.com and respective Youtube channel. I immediately did
a preorder. In november the freshly printed book arrived. Only then I found that
Russell also did a FPGA development board, ["The Go Board"](https://nandland.com/the-go-board/),
which goes along with the book. So I did another order to the US and only a few weeks
later I had a shiny new hardware toy.

{{< imgproc PXL_20231214_191504228 Resize "1000x" "Shiny new toy" />}}

The board has four LEDs, four switches, two seven segment displays, a VGA output
and even a PMOD connector. Programing is done using the onboard Micro USB.
The FTDI chip has a second UART channel that is routed to the FPGA so it can
also be used for experiments. The FPGA itself is a very basic Lattice ICE40 flavor(ICE40 HX1K, 1280 logic cells).

Now I had everything I needed to get up and running. The book recommends the Lattice
iCECube2 IDE. I gave it a try and well, it works. As I have already played with
the open source FPGA toolchain I soon switched over. So [Yosys and nextnpr](https://yosyshq.net/)
and GNU make were my tools of choice. For transferring the bitstream to the Go
board I use the Lattice Diamond programmer, since my machine is running Windows.

The book gives an excellent introduction with many nice example projects.
So I controlled the LEDs with the keys, built a Flip-Flop, debounced the switch
and played around with the seven segment display. The tiny board is really versatile
and fun to play with. In the book all sources are in Verilog and VHDL so you can
learn both if you like. I decided to go with Verilog. I also played around with
[Amaranth](https://amaranth-lang.org/docs/amaranth/latest/) which is a python based
HDL toolchain and comes with support for the Go board.

In the back of my head I was already considering my own projects ideas. Then 
I came across [FLOPPY.CAFE](https://floppy.cafe/index.html) on Hacker News.
The idea of this site is to control a 3.5 inch floppy drive from a Teensy 4.0
microcontroller. The main thing you have to do is watch the data line from the
drive. I thought: "Hey, I can do this from my FPGA!". The floppy I/O-pins are 5V
rated, but the outputs are open drain and you have to supply a pull up externally.
So we can pull up the "Read Data"-pin to the 3.3V from our Pmod connector and we
are done. No fancy level converters required. And an old floppy drive is still
collecting dust in our attic, together with the required ATX power supply.
So I started screwing all components down on a wood board, to get a robust setup.

{{< imgproc PXL_20231220_113352096 Resize "1000x" "Wooden board with all components" />}}

Then I started wiring everything to the breadboard. I also added my trusty 10$ Saleae
clone, as a Logic analyzer might be helpful in this project.

{{< imgproc PXL_20231220_123957179 Resize "1000x" "Breadboard wiring with logic analyzer" />}}

I pulled down "Drive Select 1" and "Motor On" inputs on the drive and it started
spinning. I could then verify the "Index"-pin was working with the logic analyzer.
A pulse every 200ms meant the disk was spinning with 5 rounds per second, meaning
300 rpm. Exactly in spec.

{{< imgproc PXL_20231220_124003867 Resize "1000x" "It's alive" />}}

So how do you encode your data to flux changes on a magnetic medium? FLOPPY.CAFE
describes that in great detail, so just a few basics here. The disks from the PC
era use MFM (Modified frequency modulation). You have to encode data and clock into
one signal to be able to recover the clock when reading. The main idea is to use
three discrete symbols:
- short(S), 2us between flux changes for HD disks
- medium(M), 3us
- long(L), 4us.

The time is measured between two falling edges on the data pin.

{{< imgproc PXL_20231222_095447549 Resize "1000x" "The first drive I tested was defect, resulting in too long pulses(>>4us)" />}}

They represent the binary values 0b10, 0b100 and 0b1000.

The encoding rule is simple: create a flux change only when the data bit is `1`
or when the clock bit is between `0` data bits.

```
data  0x3A 0   0   1   1   1   0   1   0
clock 0xFF   1   1   1   1   1   1   1   1
MFM stream 0 1 0 0 1 0 1 0 1 0 0 0 1 0 0 ?
```

To find an entry into the data stream, there is a sync sequence which can be detected.
It is based on the `0xA1` data byte.

Normally `0xA1` is encoded as `LMSSM`:
```
data  0xA1 1   0   1   0   0   0   0   1
clock 0xFF   1   1   1   1   1   1   1   1
MFM stream 1 0 0 0 1 0 0 1 0 1 0 1 0 0 1 ?
                             * <- skip for sync
           |  L  | | M | |S| |S| | M |
```

To get a sync, we skip one flux change, marked with `*`.

```
data  0xA1 1   0   1   0   0   0   0   1
clock 0xFF   1   1   1   1   1   1   1   1
MFM sync   1 0 0 0 1 0 0 1 0 0 0 1 0 0 1 ?
                             * <- skipped for sync
           |  L  | | M | |  L  | | M |
```

This is encoded as `LMLM` which is a unique sequence, that only appears for the sync
and is easy to detect in the data stream.
Sector headers start with a sequence of three modified `0xA1` bytes, so you always
get synced up.

As the Go board is clocked with 25 MHz we do not have to recover a clock to find
optimal sampling points. We oversample the 500 kHz data stream and can
measure the pulse lengths simply by counting the FPGA clocks.

As I had hacked together some Verilog code to do quantization I ran into some trouble.
Somehow I had managed to rip the micro USB connector off my board.

{{< imgproc PXL_20231222_192855975 Resize "1000x" "Micro USB ripped off" />}}

Russell was incredibly helpful in sorting this and immediately shipped a replacement
board. But as this was my christmas holiday hacking project, I was really
disappointed.
But I did not give up. I recorded the data pin with the logic analyzer in my scope at 62.5 MSamples
per second while the disk did a full rotation. So I should have all the data from
one track. I wrote a python script to parse the samples and verify that it
contained valid data. And really, I could parse all 18 sector headers that way
and even found some plausible strings in the sector data.
One thing that I always avoided was writing a simulation/testbench. But as my hardware
was still on the way, that was my only option. I decided to give Verilator a try.
I threw the samples from the logic analyzer at it and wanted to see, what it did
with my Verilog quantization code.

And I was completely blown away. It was so cool. I could see any signal inside
the design at any time. I mean, I knew that before. But experiencing it on your
own design with meaningful input data is really awesome.

{{< imgproc gtkwave Resize "1000x" "Viewing data from Verilator in gtkwave" />}}

It felt like a superpower. I could quickly fix the culprits in my code. And as
you can see in the screenshot it works fine now. Apart from the timescale, but I
haven't found the correct way to fix that yet.

Anyway, I could start implementing a state machine for detecting sync sequences now.
I had never written a state machine in Verilog before, but with the help of the
book it was a piece of cake.

```
always @(posedge i_Clk or posedge i_Reset)
begin
    if (i_Reset)
        r_State <= WAIT_L0;
    else
    begin
        case (r_State)
        WAIT_L0:
            if (i_L)
                r_State <= WAIT_M0;
        WAIT_M0:
            if (i_M)
                r_State <= WAIT_L1;
            else if (i_S || i_L || i_Error)
                r_State <= WAIT_L0;
        WAIT_L1:
            if (i_L)
                r_State <= WAIT_M1;
            else if (i_S || i_M || i_Error)
                r_State <= WAIT_L0;
        WAIT_M1:
            if (i_M)
                r_State <= DONE;
            else if (i_S || i_L || i_Error)
                r_State <= WAIT_L0;
        default:
            r_State <= WAIT_L0;
        endcase
    end
end
```

So I could generate sync pulses. With this information it was possible to parse
the data bits from the interleaved data/clock-stream. For this it is very handy
that in programmable logic you can generate registers of arbitrary bit size.
```
reg [19:0] r_Bit_Fifo;
```
All incoming data gets shifted into this 20 Bit register. With the sync information
the logic can derive when the MSB is at bit 19 of the register. At that moment bit
19 to 4 get latched. Data and clock information can be separated from these 16 bits.

{{< imgproc gtkwave1 Resize "1000x" "Retrieving data from the bit fifo" />}}

After that the data goes into a state machine that parses the sector header fields.
And that's all I wanted to achieve in step 1 of this project. In simulation this all
worked nicely. I even could very the crc code.

{{< imgproc gtkwave2 Resize "1000x" "Parsing sector header" />}}

Then, on my birthday, the board finally arrived from the US and I could give my
code a test in real hardware. I routed the sector number to the seven segment display.
My expectation was, to see the display change very fast. I wanted to record it
with the slowmo function of my smartphone camera. But what I saw as a little
disappointing. The number only changed unregularly, sometimes even stuck for a second.
So what was the problem? I checked with my scope that the data signal was alright.
I routed some debug signals to the Pmod connector and checked them. It looked like
the pulse length measurement was skipping some pulses. Staring at the code I got
an idea.

```
    always @(posedge i_Clk)
    begin
        if (r_Last && !i_Data) // detect data pin falling edge
        begin
            ... // compare counter values
        end
        r_Last <= i_Data;
    end
```

What if i_Data was changing while the clock handler logic was active? I remembered,
that I had heard from my former colleagues doing FPGA designs, that input signals
should be registered. That means, that you transfer the value from the data pin to
a register, and only handle it one clockcycle later. That way you are sure,
that every part of the logic sees the same value. The implementation is trivial:

```
    always @(posedge i_Clk)
    begin
        r_Data <= i_Data;
        if (r_Last && !r_Data)
        begin
            ...
        end
        r_Last <= r_Data;
    end
```
And that fixed it. Now the sector display works flawlessly.

![Sector header display](slomo.gif)

So far, my learning experience was epic. I really feel like I gained a new superpower.
Big thanks to Russell Merrick for making this awesome book and the Go board.

All code (and the sample data) can be found here: https://github.com/ZeroMips/floppy-fun
