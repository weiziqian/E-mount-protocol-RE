# Message 0x5D — start the 0x5E stream

**Summary.** Three bytes, `00 01 00`, sent by the Sony a7CR after the last [0x3B](msg_0x3B.md) page of the post-init burst.
It starts the lens's [0x5E](msg_0x5E.md) stream: sent alone after an init without the burst, it made a Sony SEL24F28G and SEL40F25G answer with a
3-byte 0x5D and start 0x5E, which no other burst message did. **PROBABLE**.

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Open questions

- What the payload and the lens's reply mean.
