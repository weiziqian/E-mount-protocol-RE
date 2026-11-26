# Message 0x43 — move focus to a position

**Summary.** The body asks the lens to drive focus to the position in `pl[1..2]`, in the units of [0x06](msg_0x06.md) `pl[2..3]`.
The lens moves there and answers with the request's counter.

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Item | Behaviour | Confidence |
| --- | --- | --- |
| `pl[0]` | counter, +1 per request | **PROBABLE** |
| `pl[1..2]` | target focus position, u16 LE. A single request moved a Sony SEL24F28G and SEL40F25G to 17000, 18000 and 16384 | **PROBABLE** |
| `pl[3..10]` | zero, except `pl[6]`: `01` while the a7CR holds focus for manual override (DMF), `82` in its last request of an AF-S run | `pl[6]` **POSSIBLE**. Sent to a Sony SEL40F25G, a request with `pl[6]` = `01` moved focus and was answered; one with `82` moved focus but drew no reply |
| Reply | 7 bytes, sent when the move ends (54 ms for no move, 118 ms for 1616 counts): `pl[0]` = the counter of the request answered; `pl[1..2]` is `ff ff` when the next request arrives first, otherwise a value between `0x110F` and `0x11CA` that does not follow the distance; the rest zero | timing and counter **PROBABLE**; `pl[1..2]` **UNKNOWN** |
| Speed | the same as a 0x04 Move: 1616 counts in about 117 ms against 111 ms, the difference about one loop | **PROBABLE** |
| Use by the Sony a7CR | autofocus: one request per loop while the half-press is held, the target updated as it goes (the lens answers `ff ff` to each request the next one overtakes), and one to the chosen position at the end of an AF-S run; also 13 to the infinity end about 1.9 s after power-on. Mixed with [0x45](msg_0x45.md)/[0x46](msg_0x46.md) sweeps and [0x49](msg_0x49.md) steps | **PROBABLE** (four captures) |

## Open questions

- What the reply's `pl[1..2]` is.
- How it differs from a Move record in [0x04](msg_0x04.md), which also drives focus to a position.
