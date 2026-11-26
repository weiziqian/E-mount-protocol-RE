# Message 0x44 — short class-3 exchange

**Summary.** Three bytes. The lens's reply to [0x5F](msg_0x5F.md) in the init and to a 7-byte [0x41](msg_0x41.md); the body also sends it once in the init,
twice around an aperture ring move, and once at shutdown.

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known (Sony a7CR, Sony SEL40F25G)

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Reply to 0x5F | `1a 00 00` | **POSSIBLE** |
| Reply to a 7-byte [0x41](msg_0x41.md) `6k 00 00 44 x t 00` | `t 00 00`, `t` the lens's 0x42 counter | **PROBABLE** (captures; a SEL40F25G sent the same frames) |
| On leaving movie mode | the a7CR sends `02 2e 00`; the lens answers `2e 00 00` | **POSSIBLE** |
| Ring move | two B→L frames, no reply; the body's [0x03](msg_0x03.md) `pl[10]` is `04` between them and `02` otherwise | **POSSIBLE** |
| Not sent | when the aperture is changed from the body | **POSSIBLE** |
| Needed | no: the Sony SEL40F25G ran through the same ring moves without it | **PROBABLE** |

## Open questions

- What `pl[0..1]` encode.
