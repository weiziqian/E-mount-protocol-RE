# Message 0x42 — focus position samples

**Summary.** Sent every loop by the Sony SEL40F25G and SEL24F28G (not the SELP1650), from the body's [0x5F](msg_0x5F.md) on: a counter and the focus position sampled 16 times over the loop.

**Direction:** L→B. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Layout

35 payload bytes (frame length 44) in the `60` form; see below for the others.

| `pl` | Meaning | Confidence |
| --- | --- | --- |
| `0` | counter, +1 per loop; the a7CR's [0x51](msg_0x51.md) `pl[2]` is this plus 2, which the lens does not check | **PROBABLE** |
| `1` | the frame's number within the loop: `00` in the `60` form (see below) | **PROBABLE** |
| `2` | the form: `04` = 16 samples per frame (see below) | **PROBABLE** |
| `3..34` | 16 × u16 LE focus-position samples (in the `60` form; see below), in the units of [0x06](msg_0x06.md) `pl[2..3]`, oldest first, about 1 ms apart: each frame continues where the previous one ended, and 0x06 `pl[2..3]` is a few counts past the last sample. Equal at rest; while the focus ring turns in manual focus by wire ([0x04](msg_0x04.md) `pl[3]` = `0x81`) they ramp, rising toward near | **PROBABLE** (a7CR captures; the ring-by-wire ramp also seen without an a7CR) |

## Sample-rate forms

[0x41](msg_0x41.md) selects the form. The samples stay about 1 ms apart; shorter frames come more often.

| 0x41 `pl[0]` | `pl[2]` | Samples per frame | Payload | Frame length | Use by the a7CR |
| --- | --- | --- | --- | --- | --- |
| `60` | `04` | 16 | 35 B | 44 | idle in photo mode |
| `61` | `03` | 8 | 19 B | 28 | autofocus runs, movie recording |
| `62` | `02` | 4 | 11 B | 20 | around the exposure of a shutter release |
| `63` | `01` | 2 | 7 B | 16 | not seen |

`pl[1]` numbers the frames within a loop when there is more than one. **PROBABLE** (a7CR captures; all four forms on a SEL40F25G).

## Open questions

- Whether it is the encoder reading or the commanded position (both coincide when the body drives focus).
- Some senders get 0x05 and 0x42 twice per loop (the extra pair just after VD), with the samples split over the two 0x42 frames and mixed
  with other values; what causes it is **UNKNOWN**. Ruled out: when in the VD cycle the sender sends 0x03/0x04 (2 to 14 ms after VD), how
  long it holds BODY_CS (0.3 ms or the whole frame window), the VD pulse width (0.1 to 8 ms), sending 0x03 and 0x04 in one write, and
  the init (a7CR's or the sender's own).
