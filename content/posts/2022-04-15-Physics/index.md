---
title: "Xoboing physics"
date: "2022-04-15"
cover: "cover.jpg"
---

I spent a lot of thoughts on how to implement the physics for the Xoboing demo.
In the original it is done with lots of floating point arithmetic but that is
obviously not the right thing for a 6502, whose only math operation is integer
addition. So I looked if things could be simplified to integer arithmetic,
and found that it should be doable without too much effort. First I gave implementing
everything in assembly a try, but doing 16 bit additions that way is ugly.
Since I am using ca65 as a assembler anyway, why not use cc65 and write the
whole thing in C? Since Copper does all of the heavy lifting, the 6502 is idling
most of the time anyway.

So what is necessary to integrate C code? First of all you need to choose an
architecture. Since we are not using one of the classic home computers, the best
choice is probably the 'none'-architecture. cc65 is using its own stack for C code,
so we have to supply that, which is mostly a bit of linkerscript magic:

```
SYMBOLS {
    __STACKSIZE__:  type = weak, value = $0800; # 2k C stack
    __STACKSTART__: type = weak, value = $800;
}
```

And then initialize the global sp variable in zeropage:

```
.import __STACKSTART__ ; Linker generated

.include "zeropage.inc"

.segment "INIT"
    ; initialize C runtime stack
    lda #<__STACKSTART__
    ldx #>__STACKSTART__
    sta sp
    stx sp+1
```

All C symbols are also available in assembly code, they are just prefixed with an
underscore.

So calling the physics engine is pretty easy. Values are exchanged through global
variables. They could also be put into function parameters, but that means pushing and
popping them on the stack every cycle which is really some unnecessary overhead
for this usecase. So the assembly side looks like that:
```
    jsr _init_physics
    xosera_wr_extended (XR_COPPER_ADDR + $0002 << 1 | $1), (1<<4)

loop:
    jsr _do_physics

    xosera_wr16 XM_XR_ADDR, (XR_COPPER_ADDR | $01)
    xosera_wrp16 XM_XR_DATA, _dst
    xosera_wr16 XM_XR_ADDR, (XR_COPPER_ADDR | $11)
    xosera_wrp16 XM_XR_DATA, _scroll
    xosera_wr16 XM_XR_ADDR, (XR_COPPER_ADDR + ($0002 << 1 | $1))
    xosera_wrp16 XM_XR_DATA, _gfxctrl
    xosera_wr_extended XR_COPP_CTRL, (1<<15)
```

The physics part itself is really tiny:
```
/* vertical speed is scaled by 10 to improve accuracy */
v_y += a_y;
top_left_y += (v_y / 10);
if ((top_left_y > WALL_BOTTOM) || (top_left_y < WALL_TOP)) {
    v_y = -v_y;
    top_left_y += (v_y / 10); /* conservation of momentum */
}

/* horizontal speed is constant */
top_left_x += v_x;
if ((top_left_x > WALL_RIGHT) || (top_left_x < WALL_LEFT))
    v_x = -v_x;
```

But the result looks quite nice.
{{< youtube 2yqg9M2u6ts >}}

What puzzled me was the fading track behind the ball (you can see it clearly on
title image), but this really seems to be only an artifact from the slow TFT
display from the early 2000s.

So what's next? On my TODO list there is the [Romulator](https://bitfixer.com/product/romulator/)
which is waiting to be assembled and tested. And Xark has some preliminary audio
support for the Xosera up and running, so I could add some boing sounds to this
demo. Stay tuned.
