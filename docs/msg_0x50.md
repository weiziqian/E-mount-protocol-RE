# Message 0x50 — lens state for the picture

**Summary.** The class-3 counterpart of [0x28](msg_0x28.md): sent by a Sony a7CR at a shutter release, inside the pause of the normal loop,
before and after each frame's exposure (eight times in a burst of four); to a lens without the class-3 group it sends 0x28 instead. The body sends 23 bytes,
`n df 9e 1f` then zeros except `pl[19]` = `01`; the lens answers with its focus distance, two distances either side of it,
the focal length and the current aperture: likely the picture's metadata. **PROBABLE** (SEL40F25G and SEL24F28G captures; the reply checked on a Sony SEL40F25G
at two focus positions).

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Lens reply

37 bytes, sent to a SEL40F25G at 1.4 m and at 0.49 m:

| Field | At 1.4 m | At 0.49 m | Meaning | Confidence |
| --- | --- | --- | --- | --- |
| u16 | `9f 01` (415) | `3f 01` (319) | the subject distance code, equal to [0x05](msg_0x05.md) `pl[20..21]` | **PROBABLE** |
| u16, u16 | `8e 01`, `b2 01` (398, 434) | `3a 01`, `44 01` (314, 324) | distance codes either side of it, the gap growing with distance: likely the near and far limits of the depth of field | **POSSIBLE** |
| u16 | `9c 01` (412) | `9c 01` | constant, about 3 % above the focal length (`f7 00` = 247 on the SEL24F28G) | **UNKNOWN** |
| u16 | `90 01` (400) | `90 01` | the focal length in 0.1 mm: 40.0 mm; `f0 00` = 24.0 mm from a SEL24F28G | **PROBABLE** |
| u16 | `55 13` | `55 13` | the current aperture ([0x05](msg_0x05.md) `pl[0..1]`) | **PROBABLE** |

In the capture the same layout carried `85 01`, `79 01`, `93 01`. The other bytes **UNKNOWN**.

## Open questions

- The constant `0x019C`, the remaining bytes, and what `df 9e 1f` in the request selects.
