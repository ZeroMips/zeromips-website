---
title: "ROMulator"
date: "2022-04-27"
cover: "cover.jpg"
---

The next step in my plan was to integrate the [ROMulator](https://github.com/bitfixer/bf-ROMulator)
into my setup.

The ROMulator is a FPGA based device that can emulate RAM and ROM for the 6502 -
something that with the current setup [6502ctl](https://github.com/ZeroMips/arduino-6502ctl)
is doing for me. So what are the advantages of the ROMulator approach?
- can emulate full 64k RAM (Mega 2560  has only 8k of SRAM)
- can run at full clockspeed
- can save dump of RAM to disk
- can test binaries quickly, just upload it to ROMulator
- debugging with 6502ctl still possible

Here is a block diagram of my new setup:

```
+---------------------+   +---------------------+
|                     |   |                     |
| DIL-40 IC test clip <-+-+ 6502ctl / Mega 2560 |
|                     | | |                     |
+---------------------+ | +---------------------+
                        |
+---------------------+ |
|                     | |
| WDC 65C02S          <-+ Adress, Data, Reset, Power, Clock
|                     | |
+---------------------+ |
                        |
+---------------------+ | +---------------------+ +---------------------+
|                     | | |                     | |                     |
| ROMulator           <-+ | GAL16V8 CS decoder  | | Xosera Graphics     |
|                     | | |                     | |                     |
+---------------------+ | +---------^-----------+ +---------^-----------+
                        |           |                       |
+-----------------------v-----------v-----------------------v-----------+
|                                                                       |
| Breadboard                                                            |
|                                                                       |
+-----------------------------------------------------------------------+
```

Clock is still controlled by 6502ctl, this will be changed to a 1 MHz oscillator
in the next step. RAM and ROM are coming from the ROMulator. You can see the
stack of modules and connectors nicely in the title photo.

The ROMulator is controlled by a Raspberry Pi via SPI. Bitfixer provides a script
that sets everything up nicely (at least if you have a user named pi on the machine,
gnah), including FPGA toolchain and 6502 tools. The FPGA is programmed via this
SPI interface, but it also provides runtime controls for reading and writing to
SRAM.
Alternatively, you can control the ROMulator from a Wemos D1 Mini module, which
I tried first. But after two nights trying to program the module and another replacement
module I finally gave up.

So how does ROMulator know where to emulate memory? This has to be configured
before building the FPGA bitfile. In the `config` directory there are two textfiles
used to configure this:
- memory_set_default.csv
- enable_table_default.csv

In `memory_set_default.csv` you can define binary data that is loaded to memory.

Example:
```
4,basic-4-b000.901465-23.bin,0xb000
```

This means that on DIP-Switch setting 4 the contents of file `basic-4-b000.901465-23.bin`
will be loaded to address 0xb000 in memory.

In `enable_table_default.csv` you define how memory areas are handled by ROMulator.

Example:
```
0-14,0x0000,0x7FFF,"readwrite"
0-14,0x8000,0x8FFF,"writethrough"
0-14,0x9000,0xAFFF,"passthrough"
0-14,0xB000,0xE7FF,"readonly"
0-14,0xE800,0xEFFF,"passthrough"
0-14,0xF000,0xFFFF,"readonly"
```

This means that DIP-Switch setting 0 to 14 are all treated the same.
The next two numbers are start and end address of the defined memory area.

The ROMulator documentation explains the meaning of the different keywords.
- `readwrite` : fully replace any read or write to this range. This is used for RAM that you wish the ROMulator to replace.
- `readonly` : replace any read to this range with the ROMulator's memory, and disable writes. This is used for ROMs.
- `passthrough` : reads and writes to this region should go through to the mainboard and not be intercepted by the ROMulator. This is used for I/O.
- `writethrough` : writes to this region are done both to the mainboard and to the ROMulator. Reads are read from the mainboard. This is useful for sections of memory that are read by something other than the CPU, which is often the case with video RAM.

So for zeromips it looks like this:
```
# RAM
0-14,0x0000,0x7FFF,"readwrite"
# IO
0-14,0x8000,0x9FFF,"passthrough"
# ROM
0-14,0xA000,0xFFFF,"readonly"
```

After building a bitfile and writing it to ROMulator, the xoboing demo worked on
first try. So compliments to Bitfixer who did an awesome job making this amazing
tool ready for production.

Next step for zeromips will be rewiring the breadboard to provide 5V power, reset
 and quartz oscillator clock.
