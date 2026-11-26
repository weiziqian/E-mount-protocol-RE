# Message 0x52 — iris move complete

**Summary.** The lens's report that the iris has reached the target of a [0x51](msg_0x51.md) whose `pl[1]` is `80`; sent as the iris
arrives, or at once when it is already there. A 0x51 with `pl[1]` = `00` draws none. **PROBABLE** (0x51 frames sent to a Sony SEL24F28G; the same
report follows the a7CR's init-time 0x51 in a capture of the SEL40F25G).

**Direction:** L→B. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Layout

11 payload bytes (frame length 20).

| `pl` | Meaning | Confidence |
| --- | --- | --- |
| `0` | the counter of the 0x51 it answers | **PROBABLE** |
| `1` | varies (`1d`, `58`, `96`) | **UNKNOWN** |
| `2` | `85` | **UNKNOWN** |
| `3..4` | the aperture reached, u16 LE | **PROBABLE** |
| `5..10` | zero | |

## Open questions

- What `pl[1]` and `pl[2]` carry, and whether this is the `L2B_MOVE_IRIS_COMP_A` reply named on the 0x51 page.
