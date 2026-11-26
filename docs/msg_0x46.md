# Message 0x46 — run the focus sweep; replies 0x47 and 0x48

**Summary.** Three bytes from the body (`97 00 00` from a Sony a7CR). After a [0x45](msg_0x45.md), the lens sweeps focus from the run-up point to the far
end set there; at power-on that is past the lower travel limit (the infinity end; [0x06](msg_0x06.md) `pl[0]` bit 4 set), then back onto it. It sends 0x47 as the
sweep starts and 0x48 near its end (a SEL40F25G: 0x47 at 16333, 0x48 at 19740 of a sweep to 19958). It did not always start: see the checks on [0x45](msg_0x45.md). **PROBABLE** (sent to a Sony SEL24F28G and SEL40F25G; the same sweep follows 0x46 in an a7CR capture of the
SEL40F25G).

**Direction:** B→L (0x46); L→B (0x47, 0x48). **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Message | Payload | Confidence |
| --- | --- | --- |
| 0x46 | `97 00 00` | **UNKNOWN** fields |
| 0x47 | 3 bytes, `pl[0]` counter-like (`98` in the capture, `97` from a SEL24F28G) | **POSSIBLE** |
| 0x48 | 3 bytes (`9a 01 00`, `9e 00 00`) | **UNKNOWN** |

The a7CR also sends 0x46 after each [0x45](msg_0x45.md) during autofocus on a half-press (one to three per run); the lens answers 0x47
(`pl[1]` `00` or `01`) and sometimes 0x48. **PROBABLE** (four captures).

## Open questions

- What 0x47 and 0x48 report, and why 0x48 does not follow every sweep.
