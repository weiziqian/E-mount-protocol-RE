# Message 0x49 — focus step during autofocus

**Summary.** Sent by a Sony a7CR about once per loop while it autofocuses on a half-press (AF-S and AF-C), between the
[0x45](msg_0x45.md)/[0x46](msg_0x46.md) sweeps: a counter and a signed step. The lens acknowledges each one. **PROBABLE** (Sony SEL40F25G and SEL24F28G, five captures).

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Request | 7 bytes. `pl[0]` low two bits `01`, upper bits a counter (`11`, `15`, `19`, `1d`, …); `08` or `0c` once at the end of a run | **POSSIBLE** |
| `pl[2]` | signed; its sign is the direction of the sweep (below): `04`, `02`, `fc`, `ff` around a point in AF-S; a shrinking run `1a`, `17`, `14` … `04`, `02`, then `fe`, `fc` in AF-C | **POSSIBLE** |
| Reply | 3 bytes, `pl[0]` a counter `00` to `03` | **POSSIBLE** |

Sent to a Sony SEL40F25G on its own, after a [0x45](msg_0x45.md)/0x46 run, the lens acknowledged every step; the third drove focus to the
near limit and the others moved nothing visible. **POSSIBLE.**

In three a7CR captures (AF-C, AF-E lock, tracking) focus moved about 865 position counts per two loops while 0x49 frames were sent, toward
near when `pl[2]` was positive and toward infinity when negative, whatever its size. So its sign is the direction of the sweep under way;
the size is not a distance. **PROBABLE** for the sign.

## Open questions

- What the size of `pl[2]` counts.
