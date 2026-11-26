# Message 0x41 — select the 0x42 sample rate

**Summary.** `pl[0]` = `0x60` + k sets how the lens packs its [0x42](msg_0x42.md) focus-position samples: k = 0 gives one frame of 16
samples, 1 frames of 8, 2 frames of 4, 3 frames of 2, the frames coming correspondingly more often. The body sends 3 or 7 bytes; the lens
also sends a 3-byte form. **PROBABLE** (Sony a7CR captures with a SEL40F25G and SEL24F28G; all four values sent to a SEL40F25G).

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Item | Behaviour | Confidence |
| --- | --- | --- |
| `pl[0]` | `60`, `61`, `62`, `63`: the 0x42 form, switched within about 25 ms | **PROBABLE** |
| 7-byte form | `6k 00 00 44 x t 00`, `t` the lens's 0x42 counter, `x` `81`, `02` or `01`; the lens answers [0x44](msg_0x44.md) `t 00 00` (seen for `61 … 81`, `60 … 02` and `60 … 01`) | **PROBABLE** |
| 3-byte form | `6k 00 00`, no reply | **POSSIBLE** |
| Use by the a7CR | `61` before an autofocus run and on entering movie recording, `60` after the run and on leaving it, `62` around the exposure of a shutter release, `60` at shutdown | **PROBABLE** |
| From the lens | `1a 00 00` after the init-time [0x51](msg_0x51.md); `6c 00 00` at a shutter release; `pl[0]` of the 3-byte form is then the lens's loop counter | **POSSIBLE** |

## Open questions

- What `x` (`pl[4]`) selects, and what the lens's own 0x41 reports.
