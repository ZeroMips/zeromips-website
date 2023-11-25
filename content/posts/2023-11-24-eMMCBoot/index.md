---
title: "eMMC boot"
date: "2023-11-24"
cover: "cover.jpg"
---

The MNT Reform system image is deployed on a SD card. This works nicely, but is
really more on the slow side, especially when you got used to these new fangled
SSDs. Luckily the motherboard has a m2 slot that in case of the LS1028A
processor module is suitable for the installation of SATA SSDs.
There is even some scripting available to migrate your system from SD.
As the processor can not boot directly from SATA, you are
still dependent on the SD card for holding the boot image, consisting of
RCW/PBI, Trusted Firmware-A (TF-A) and U-Boot.

The MNT Reform is a mobile device, it gets stuffed into bags and pulled out
again, which leads ever so often to accidental ejection of the SD card.
Without it, the system simply will do nothing when powered on. As I found that
really annoying, I was looking for alternatives. Luckily the processor module
comes with an eMMC on it. An eMMC is a solid state drive, very similar to a SD
card, but in an IC package, so it is not removable. Which is just what I want.
Unfortunately you have to tell the processor from where to boot on powerup.
This means the processor samples its configuration from the status of certain pins
that have to be strapped to 1.8V or GND with a resistor. This is called
Power-on reset(POR) configuration.

One part of the POR configuration is the RCW source. RCW means Reset Control
Word, which is a structure of 32 32-bit words that can configure many
aspects of the system. The RCW source is encoded in four bits. The source can either
be one of the hardcoded configurations, or it can be stored in some kind of nonvolatile
memory like an SD card, an eMMC, SPI flash or I2C eeprom.

The LS1028A has an on-chip service processor that is responsible for setting up
everything for the main processor to boot. It reads the RCW and writes it to
the main processors Device Configuration (DCFG) registers. The service processor 
can also execute a custom assembly program called PBI(Pre-Boot Initialization).
It has full access to all main processor registers and memory trough the chip
interconnect. Normally part of the PBI program is a block copy operation, that
reads the bootloader from nonvolatile memory and writes it to on-chip memory,
where it can be executed by the main processor.

{{< imgproc PXL_20231120_183940724 Resize "1000x" "Hard wired to SD card boot" />}}

On my revision of the processor module(Rev02b), the plan was to encode three bits
of the RCW source with DIP switches on the motherboard. Running these signals
through a 3.3V to 1.8V levelshifter did not work as intended. Lukas finally
decided to remove the levelshifter and hardwire the RCW source to SD card.
This was no good news for me since it meant that I would have to modify the
hardware in order to get rid of the SD card.

First of all I had to get rid of the hardwired strapping and cleanup everything
nicely.

{{< imgproc PXL_20231120_184330252 Resize "1000x" "Hard wired strapping removed" />}}

To do this I had my ad hoc setup from work with me. Honestly my eyes are not
good enough anymore to do this without a proper microscope.

{{< imgproc PXL_20231120_184715210 Resize "1000x" "Rework setup" />}}

Next step was to hardwire a strapping to eMMC boot. Before I had already copied
the boot image there.

{{< imgproc PXL_20231120_204005884 Resize "1000x" "Hard wired to eMMC boot" />}}

I anxiously turned on the system, just to get some sobering results: nothing happened.
Now it was pretty hard to tell what went wrong. But it was pretty clear that I
would have to undo my hardware change and be able to boot from SD card again.
As it was obvious that this would take some more iterations, I decided to install
a jumper on the processor module to be able to switch between SD and eMMC boot.

{{< imgproc PXL_20231121_191248164 Resize "1000x" "Jumper for selecting boot source" />}}

To understand what went wrong I decided to have a look at the generated boot image.
It was built using the scripts from the reform-ls1028a-uboot repository.
As it is no fun to go through RCW/PBI code by hand, I decided to implement a
radare2 disassembler plugin for the service processor machine code.
Disassembly of the beginning of the image looked like this:

```
0x00000004      0000108010..   Load RCW with Checksum
0x0000008c      0013ea3120..   CCSR Write sys_addr 0x1ea1300, data 0x80104e20
0x00000094      0004e03100..   CCSR Write sys_addr 0x1e00400, data 0x1800d000
0x0000009c      0900008000..   Block Copy src 0x9, from 0xa000, to 0x1800d000, size 0x7332
0x000000ac      0000ff8000..   Stop
```

Most important here is the src parameter in the Block Copy command. 0x9 is for
eMMC, so this was exactly what I wanted. There were still a lot of things that
could go wrong later on in the boot process. So my idea was to verify, that the
PBI code was executed properly in the first place. Since I have full access to
the main processor registers, it would be not problem to turn on a GPIO. Luckily
there is a LED wired to GPIO1_25 on the board. So I quickly hacked together a
simple PBI program.

```
0x00000004      0000108010..   Load RCW with Checksum
0x0000008c      0000303240..   CCSR Write sys_addr 0x2300000, data 0x40
0x00000094      0800303240..   CCSR Write sys_addr 0x2300008, data 0x40
0x0000009c      0013ea3120..   CCSR Write sys_addr 0x1ea1300, data 0x80104e20
0x000000a4      0000ff8000..   Stop
```

The registers for the first GPIO module are at 0x2300000. At offset 0 there is the
direction register, at offset 8 there is the data register. To set GPIO 25, one
would expect that bit 25 must be set, which would be a value of 0x02000000.
But the GPIO registers are encoded in big endian *bit* order, meaning bit 0 is
for GPIO 31. This is a legacy from its PowerPC ancestors, that I luckily have
worked with in the past. It took me a few iterations to remember that. So the correct
value for ǴPIO 25 is 0x40.

{{< imgproc PXL_20231123_202831109 Resize "1000x" "Finally" />}}

The eMMC shows up as three different drives. There is mmcblk1, mmcblk1boot0 and
mmcblk1boot1. As I was not sure, where the service processor was getting its data,
I decided to copy the image on all three drives.

Writing is as simple as this:
`sudo dd if=flash.bin of=/dev/mmcblk1 conv=notrunc bs=512 seek=8`
So finally my LED lit up. Joy.

Now I knew, that my setup was alright. The processor recognized my strapping,
loaded the RCW and the PBI from eMMC and executed it perfectly fine.
So what was wrong, when I tried to boot the full image? I remembered, that on
my first try I had only written the boot image to mmcblk1boot0. Mainly from fear
that my processor might die from heat without heatsink installed.
So I quickly wrote the full boot image to all three eMMC drive instances.

{{< imgproc PXL_20231123_221751573 Resize "1000x" "What a nice U-Boot message" />}}

To my surprise the system the booted perfectly fine into U-Boot. The boot message
also clearly said, that we were running from eMMC. Great!

My next concern was, whether SATA was working properly in U-Boot, because that would
be the requirement for having the `/boot` partition on SSD. A quick check showed
that it worked perfectly fine.

{{< imgproc PXL_20231124_133051518 Resize "1000x" "SATA working in U-Boot" />}}

Now everything was in place. There even was a `sata_boot` script defined in the
U-Boot environment. Running that worked fine until the point where the kernel
wanted to mount the root filesystem. That made me worry, but a quick inspection
of the kernel commandline showed, that only the root parameter was missing.
So I added `root=/dev/sda1` to the `bootargs` variable. After that, the system
booted to the rootfs. systemd went bonkers because of some issue with mmcblk0p1.
I realized, that in fstab there was still an entry for the /boot partition on
SD card. After changing that, the system booted perfectly fine. I was delighted.

{{< imgproc PXL_20231124_152200065.MP Resize "1000x" "Everything mounted from SATA" />}}

So that was quite a ride, but I am really pleased with the result. The only
issue left is that the SD card is not recognized at all anymore. But I guess, we
will find a solution for that.

If you want to try this yourself, be careful. The hardware mod is doable but you
can easily damage your system. Also running the processor without a heatsink should
only be done for very short amounts of time.

In future releases Lukas will include a working solution to select the bootdevice
externally.

Once again, full access to all schematics, board layouts and sourcecode makes it
possible to do such modifications. The Freescale/NXP documentation is also really
helpful. I really appreciate that and prefer such vendors anytime
over others that don't give a damn before your order is seven figures, no matter
how sexy their products are.
