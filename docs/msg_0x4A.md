# Message 0x4A — timed focus move (class 3)

**Summary.** The body asks the lens to move focus to a position within a time limit; the lens goes as far as it can in that time and
answers with the position it will reach. Sent by a Sony a7CR around a shutter release. It is the
class-3 form of the timed focus move [0x34](msg_0x34.md): the A7 IV firmware sends the same move as 0x4A under a condition (0x34 page), and
the a7CR sends 0x34 instead to a lens without the class-3 group.
**PROBABLE** (Sony SEL40F25G and SEL24F28G captures; sent to a SEL40F25G).

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Request | 11 bytes: `pl[0]` counter (`01`, `02`, `03`), `pl[1..2]` the target focus position (in the units of [0x06](msg_0x06.md) `pl[2..3]`), `pl[7]` the time limit in ms as in 0x34 `pl[4]` (`29` = 41, `06`, `08`), the rest zero | **PROBABLE** |
| Reply | 7 bytes: `pl[0]` the counter, `pl[2..3]` the position the lens will reach, `pl[4..5]` the subject distance code of that position ([0x05](msg_0x05.md) `pl[20..21]`) | **PROBABLE** |
| Motion | sent to a Sony SEL40F25G: 17000 from 16384 with 41 ms reached 16991; 18000 with 41 ms stopped at 17598; 16384 with 6 ms only got to 17533, and with 87 ms reached 16384. Each reply named the position reached | **PROBABLE** |
| Timing (a7CR) | two just before the exposure, one after | **POSSIBLE** |

## Open questions

- What `pl[3..6]` and `pl[8..10]` carry (zero in every request seen), and the mode field 0x34 has.
