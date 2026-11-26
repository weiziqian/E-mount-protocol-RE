# Message 0x5F — start the 0x42 stream

**Summary.** The first class-3 message the Sony a7CR sends after the first loop frame; it starts the lens's [0x42](msg_0x42.md) stream. Sent alone after an init without the burst, it made a Sony SEL24F28G and SEL40F25G answer
with [0x44](msg_0x44.md) (the SEL24F28G also with a 3-byte 0x5F) and start 0x42, which no other burst message did. **PROBABLE**.

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

15 payload bytes: `00 11 05 00 00 00 00 41 60 00 00 44 01 1a 00` (Sony a7CR, SEL40F25G); `pl[13]` is the lens's loop counter
([0x42](msg_0x42.md) `pl[0]`). Fields **UNKNOWN**. Left out, with the rest of the post-init burst (0x3A, 0x3B, 0x51, 0x44, 0x5D), the Sony
SEL24F28G, SEL40F25G and SELP1650 still initialised and focus, aperture and shutdown worked, without 0x42 and 0x5E. **PROBABLE.**
[0x53](msg_0x53.md), sent before 0x0A, is not part of the burst and is needed for iris moves.

## Open questions

- What the fields mean.
