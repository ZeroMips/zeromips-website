---
title: "Galaxis"
date: "2023-04-08"
cover: "cover.jpg"
---

Recently I picked something from our attic. It is a christmas present from my
childhood, one of my favorite ones. It is the game Galaxis, a kind of electronic
board game from the 80s by german company Ravensburger.
I picked it to show it to my kids and yup, they still love it. Just as I did,
when I was a kid. The clunky switches, the flashy LEDs, the squeaky speaker,
the SciFi background story. It all feels exactly as it should do.

{{< imgproc IMG_20230408_155303343 Resize "1000x" "Flashy LEDs and clunky switches" />}}

I had to do some repairs because a 9V block was left inside and it ate away most
of the battery clip, in addition one of the traces from the barrel jack was cracked.
So not much work, but oddly satisfying to do. And my heart stopped for a moment
when I realised I connected the power supply in reverse. But luckily there are
some diodes that obviously did their job.
Concerning the electronics, there is not much to write home about. A very simple PCB
with an omnious Texas Instruments MP3416 IC, made in Italy in 1980. Probably some micro
with mask programmed ROM.

{{< imgproc IMG_20230407_110016067 Resize "1000x" "Not much to see here" />}}

Galaxis is similar to the "battleship" game. You have spaceprobes that are arranged
in a grid and you can talk to each one and ask how many spaceships they are seeing.
So you can triangulate the positions of four missing spaceships. There are single-
and multiplayer-modes. There are plastic pins to mark your findings on the playfield.
And there is a really nice [manual](https://github.com/ZeroMips/galaxis/blob/master/doc/galaxis.pdf)
with the background story.

{{< imgproc IMG_20230408_155839016 Resize "1000x" "Fresh from the 80s" />}}

I wrote a little [C program](https://github.com/ZeroMips/galaxis) that implements
the single player game, so you can give it a try yourself. I find it is quite enjoyable.
