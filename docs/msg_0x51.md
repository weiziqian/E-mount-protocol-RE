# Message 0x51 — asynchronous iris move

**Summary.** The body's aperture command for lenses that support an asynchronous iris. It is sent
every frame in live view and again at the release. First known from body firmware; the layout
below comes from captures of an a7CR with a SEL40F25G.

**Direction:** B→L, with a lens reply (`L2B_MOVE_IRIS_A`, `L2B_MOVE_IRIS_COMP_A` in the body's
names). **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## What the body does

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Who gets it | Only a lens that passes a lens-capability check for asynchronous iris. Never a third-party lens ID: `0x50`–`0x60` are masked for those ([lens ID](lens_id.md#messages-the-body-will-use)) | Check **CERTAIN on the a7C II firmware**, what it tests UNKNOWN; mask **CERTAIN on the A7 IV firmware** |
| Every frame | The body copies the live aperture of [message 0x03](msg_0x03.md#where-the-body-takes-the-aperture-pair-from) into it: `pl[5..6]` if 0x03 `pl[7]` bit 7 is set, else `pl[3..4]` | **CERTAIN on the a7C II firmware** |
| At the release | The sequencer's Iris step sends the exposure aperture here. A lens without async iris gets [message 0x1B](msg_0x1B.md) instead | **CERTAIN on the a7C II firmware** |
| Payload | Carries an [aperture value](aperture_value.md), u16 LE. Layout: see below | Aperture **CERTAIN on the a7C II firmware**; layout **OBSERVED** on an a7CR |

The body sends class-3 frames through a path of its own, one frame in flight at a time, separate
from the init-class request queue.

## Layout seen on an a7CR (SEL40F25G, class `0x03`, seq `00`)

Body to lens, 11 payload bytes (frame length 20):

```
pl[0]     n         session counter: 01 in the init, then +1 per frame
pl[1]     flags     80 in the init (ask for a 0x52 report), 00 during aperture moves, 02 in event frames
pl[2]     tick + 2  the lens's 0x42 counter (0x42 pl[0]) plus 2
pl[3]     00
pl[4..5]  u16 LE    the aperture target, as in the body's 0x03 pl[3..4] / pl[5..6]
pl[6..7]  bb 02     ff ff in the init frame, 51 06 in event frames
pl[8]     1d        18 or 1c in event frames
pl[9..10] 00 00
```

`pl[2]` was the lens's last 0x42 counter plus 2 in both captures. The lens does not check it: with the counter plus 0 or plus 10 a
Sony SEL24F28G acknowledged every frame and the iris followed. **PROBABLE.**

Lens to body, 7 bytes (frame length 16): `n 01 t 00 00 00 00` when the move is accepted, `t` the lens's own current 0x42 counter;
`n 02 00 00 00 00 00` when it is refused, and then the iris does not move. A Sony SEL24F28G or SEL40F25G refused every 0x51 when the init had left out
[0x53](msg_0x53.md), and accepted single 0x51 frames otherwise, even with the [0x03](msg_0x03.md) target unchanged. With `pl[1]` = `80` the
lens also sends [0x52](msg_0x52.md) when the iris arrives; with `00` it does not. **PROBABLE.**

One pair every other loop while the target is moving and for about 0.3 s after, stepping by `0x0080` per frame toward the value in
[0x05](msg_0x05.md) `pl[17..19]`. At a shutter release the a7CR sends one with `pl[1]` = `01` (target `0x1300`, `pl[3]` = `26`, `pl[6..8]` = `f6 6e 38`) just before the exposure. One 0x51 is also sent in the init, in the burst after the first loop frame ([0x5F](msg_0x5F.md)). The
meaning of `pl[1]` = `02` and of `pl[6..8]` is **UNKNOWN**; the SEL40F25G followed 0x51 targets the same with `pl[1]` = `00`, `02` or `80` and `pl[6..10]` = `bb 02 1d 00 00`, `51 06 18 00 00`, `ff ff 1d 00 00` or zeros. **PROBABLE** (one lens).

## Open questions

- What `pl[1]` = `02` and `pl[6..8]` mean.
- What the capability check reads: a [message 0x01](msg_0x01.md) bit, the lens ID, or a descriptor
  flag.
