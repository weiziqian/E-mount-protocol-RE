# Message 0x45 — set up a focus sweep

**Summary.** The Sony a7CR sends it before each [0x46](msg_0x46.md): at power-on, and one to three times in each autofocus run on a
half-press. 19 bytes; the positions change from run to run. **PROBABLE** (five captures). Power-on example:
`02 02 80 10 4b 40 eb 3e 7f 4d 00 00 3a 00 55 13 00 00 00`.
Sent to a Sony SEL24F28G with that lens's limits, it moved focus toward near; the 0x46 that follows starts the sequence.

**Direction:** B→L. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| `pl` | a7CR value | Meaning | Confidence |
| --- | --- | --- | --- |
| `0..3` | `02 02 80 10` | `pl[0]` steps by 4 between frames (`02`, `06`, `0a`); `pl[2..3]` `80 10` or `80 20` in a sweep, `00 01` in a hold form whose positions are all equal. Needed: with zeros focus did not move and 0x46 started no sweep, though 0x47 and 0x48 still came | **POSSIBLE** |
| `4..5` | `4b 40` (16459) | one end of the sweep: `f6 4d` (near the upper limit) in every AF sweep seen | **PROBABLE** |
| `6..7` | `eb 3e` | the other end: the lower focus travel limit ([0x06](msg_0x06.md) `pl[7..8]`) at power-on and in some AF runs, a nearer position (`d6 4c`, `ce 40`) in others. A value well away from the lens's limit has also made the following 0x46 start nothing | **PROBABLE**; the check **POSSIBLE** |
| `8..9` | `7f 4d` | near the upper travel limit (0x06 `pl[9..10]` was `f8 4d`). With 17000 instead of the SEL24F28G's 18158 the following 0x46 started nothing | **POSSIBLE** |
| `12..13` | `3a 00` | zeros changed nothing visible | **UNKNOWN** |
| `14..15` | `55 13` | an aperture value; `0x1600` instead of `0x130E` made the move on receipt longer, without moving the iris | **POSSIBLE** |

## What the lens does

On receipt the lens runs focus to about 250 counts beyond whichever end is nearer its current position; the following
[0x46](msg_0x46.md) then sweeps it to the other end, like the run-up and sweep of the Scan record ([autofocus.md](autofocus.md) §3.3).
A Sony SEL40F25G at 17000 given the AF-S frame (ends 19958 and 16590) went to 16330, then swept to 19740 on 0x46; a SEL24F28G at
16384 given the power-on frame (ends 16459 and 16099) went to about 16750, then swept down past the lower limit and settled on it.
In the hold form the lens moves to the one position and stays: the SEL40F25G went to 17152 for `00 43` and did not move on 0x46.
**POSSIBLE** (four runs on two lenses).

## Open questions

- What `pl[8..9]` and `pl[2..3]` set (speed or range), and how exactly the lens checks `pl[6..9]`.
