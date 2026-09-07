# Message 0x1D — probably not a message ID

**Summary.** Listed here because it appears in early reports as a message ID, but the evidence now
says it is **a frame length written in hex, not an ID**.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## The correction

An early report described "the body replying with packet 0x1D during AF and packet 0x20 in MF",
30 bytes each. Observed body frame lengths settle what that was:

| Message | Frame lengths observed |
| --- | --- |
| 0x03 | 29 (`0x1D`) and 32 (`0x20`) bytes — 840 frames each |
| 0x04 | 22 (`0x16`) bytes |

So "packet 0x1D" and "packet 0x20" are the **two length variants of [message 0x03](msg_0x03.md)**,
in a length notation — not two separate message IDs. The body varies message 0x03's length by device
class, which is exactly the AF/MF-looking split that report describes.

## What remains

No frame with message ID `0x1D` has ever been observed, and no manufacturer is known to implement
one. Whether the ID exists at all is UNKNOWN.

## Open questions

- Whether message ID `0x1D` exists as a distinct message.
