---
title: "PetroLED"
date: "2024-04-05"
cover: "cover.jpg"
---

My wife has restored an old kerosene lantern that the neighbours wanted to throw
away. She really put a lot of effort to bring it back to its original glory.
It is a Feuerhand Atom and probably was used somewhere in WW2. The red glass screen
suggests that it was used as a signal light of some kind.

{{< imgproc PXL_20240405_191956390 Resize "1000x" "Beautiful old lantern" />}}

We have already tried it with petroleum and it works fine. For indoor use it would 
be nice to also be able to operate it with some kind of electric lighting.
Since modifying the mechanical construction is out of question my idea was to put
a board with a battery and some kind of LED inside the screen. But any kind of point
light source would look totally wrong, so I thought about this nice filament LED
bulbs. Such a filament is just a series of very small LEDs connected in series.
Fortunately my favorite supermarket had a special offer with a pack of three
filament bulbs. I immediately got one. To my surprise the screen was made of glass.
I read that it is filled with some kind of gas for cooling purposes, so it is
sealed. I used a metal saw to carefully cut the socket open, it worked quite well.

{{< imgproc PXL_20240329_111657864 Resize "1000x" "Filament LED bulb cut open" />}}

I removed the filaments and had to learn that they are quite fragile, I broke
two in the process. Next I wanted to know, what voltage I needed to light them up.
So I connected one to my Lab supply, and at 60V it finally started glowing.
Since I want to supply the board from a 1.5V battery (and it should still work
when the voltage is falling), I considered the circuit options. A standard boost
converter would probably not work very well here. The input voltage is too low for
most ICs I saw. But I found another circuit I had not heard of before. It is called
a "Joule thief" because it can suck most of the energy from a battery, even when
the voltage gets low. It is a simple transistor circuit, that uses an inductor for
boosting the voltage. It is often used in torches to get a reasonable battery
life and boost the voltage to LED levels. I even found a project documentation
where it was used to [power a filament](https://www.instructables.com/Joule-Thief-Filament/).

{{< imgproc circuit Resize "1000x" "Joule thief circuit" />}}

To fit this into the Feuerhand glass screen I decided to do a two sided board.
The battery holder goes to the front, the joule thief circuit is SMT on the back
side. Since JLCPCB has some really nice discounts for assembly I decided to use
their stock parts for the board.

{{< imgproc back Resize "1000x" "SMT parts on the back side" />}}

 The critical parts in this design are the transistors (which must properly operate
at low voltage) and the inductor. To make sure that I find parts that work properly
I decided to do a simulation in LTSpice, which is also something I have not done
before. To do the simulation it was crucial to have a model for the filament.
Luckily there is a [research paper](https://converter-magazine.info/index.php/converter/article/view/299/292) on that topic. The model presented in this paper was
a little overkill and took endlessly too simulate. But at least there is a V-I
curve and with the help of [another article](https://luminusdevices.zendesk.com/hc/en-us/articles/10072290905101-Electrical-How-do-I-extract-Spice-IV-parameters-from-an-LED-datasheet)
I was able to find matching diode parameters. The blue curve is from the research
paper, the green curve is from my diode model.

{{< imgproc curve-fit Resize "1000x" "Find matching diode model parameters" />}}

With this model I could rig a simulation and after some fails I finally found a
combination that works reasonably well:

{{< imgproc sim Resize "1000x" "Simulation at 1.5V supply" />}}

The boards are in production now. I am really looking forward to do a reality check
once they arrive. I will report.

This is all open hardware and can be found on github:
https://github.com/ZeroMips/PetroLED