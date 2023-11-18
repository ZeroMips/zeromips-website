---
title: "Gprobe"
date: "2023-11-18"
cover: "cover.jpg"
---

My very first encounter with Lukas F. Hartmann was over at the bird site.
I hadn't heard about MNT Research or the Reform. I only saw a post where he
was talking about doing a design with the STDP2600 HDMI to DisplayPort converter
and I told him that this probably meant trouble.

My previous job was at [Guntermann and Drunck](https://www.gdsys.com/en/start),
a german manufacturer of high end KVM (meaning Keyboard Video Mouse) equipment.
I worked there from 2002 to 2019 and we did some awesome products using bleeding
edge technology. There I also did [my first design](https://www.gdsys.com/en/product/CCD-Control-Card-card-f-CCD) with the [Freescale QorIQ P1022](https://www.nxp.com/products/processors-and-microcontrollers/power-architecture/qoriq-communication-processors/p-series/qoriq-p1022-and-p1013-dual-and-single-core-communications-processors:P1022)
Communication Processor, the PowerPC ancestor of the NXP LS1028A that is available 
for the Reform today.

The Video part in KVM meant, that we had to deal with lots of special purpose
ICs implementing different video standards. For DisplayPort 1.2a we had the
STDP4320, at that time the IP was owned by MegaChips. That chip really caused
lots of headache for us. We had a customized firmware from MegaChips that still had
some bugs. It was really hard to communicate with them and get things fixed.

At one point we were desperate enough to start investigating if we could do something
about this ourselves. We had heard that the STDP line of products had an embedded
x86 core of some kind. So we started loading the firmware file into [Radare2](https://rada.re/n/),
a reverse engineering framework. We could disassemble parts, but as soon as there
were calls into other segments, things did not make sense anymore.
Doing even more research we found the [MonitorDarkly exploit](https://github.com/RedBalloonShenanigans/MonitorDarkly)
that was presented at DEFCON 24 by Red Balloon Security.
From their presentation we learned, that the x86 was in fact a Turbo186 core with very special
segment addressing. In x86 real mode, a segment address is the 16 most significant bits
of a 20 bit address, so it is shifted 4 bits left. The Turbo186 uses 24 bit addressing,
so it is a 16 bit address that gets shifted 8 bits to the left.
We did changes to Radare2 to support that (there is now a asm.seggrn parameter that
is 4 by standard and can be set to 8).
The firmware itself gets written to SPI-NOR-Flash, where the Turbo186 can access it.
There is also some vendor tooling to communicate with the core for programming
and debugging purposes via UART. Luckily the protocol - Gprobe - is nicely documented,
though available under NDA only. So we implemented the protocol as a Radare2 plugin
and could directly read and write the RAM of the Turbo186. As we had all the tooling
in place, MegaChips finally addressed the firmware bugs and delivered a version
working properly. So things settled dust.

Until Lukas started playing around with the STDP2600. And sure enough, [it meant
trouble](https://mastodon.social/@mntmn/109552847731111745). Luckily Kinetic Technologies
as the new owner of the IP seems to be much more approachable and helped Lukas with 
firmware and documentation. I also could contribute one tip or the other. So
the adapter finally started working.

Again, things settled dust, until [one evening](https://mastodon.social/@mntmn/111183953268952693):
{{< imgproc vendortool Resize "1000x" "Cursed tools" />}}

At Guntermann and Drunck I had been missing the documentation and the Turbo186 
flash loader code to implement this in the Radare2 plugin. But Kinetic
Technologies was friendly enough to provide all this to Lukas.
So Lukas sent me a MNT RHDP module and I could start hacking.

{{< imgproc PXL_20231027_181813952 Resize "1000x" "Hacking equipment" />}}

All it needed was a power supply with 5 and 3.3 Volts, a USB to serial converter
and a logic analyzer. Regarding this I can highly recommend the AZ-Delivery one
shown in the photo that by strange coincidence is compatible with the Saleae
software.

{{< imgproc PXL_20231030_220843815.MP Resize "1000x" "Serial protocol analyzer" />}}

While implementing the gprobe commands for flashing I also found some bugs in my
original implementation that I could fix on the way. The Radare2 maintainer, pancake,
also kindly contributed some improvements for the ihex plugin, so it can now be
used together with the intel hex file of the flasher. The implementation is
already merged, documentation can be found here: https://github.com/radareorg/radare2/blob/master/doc/gprobe.md.

This was really a fun trip to the past for me and I could even make the
Reform2 ecosystem a little bit more open. For the next RHDP production
run there will be no more virtual machines and cursed vendor tools required.
