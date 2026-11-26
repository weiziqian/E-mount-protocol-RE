# Message 0x2E — alternate form of the lens status frame

**Summary.** The Sony SELP1650 sends this in the loops where it does not send [0x06](msg_0x06.md): the 0x06
payload behind a 3-byte prefix. Not the `0x2E` query record of [autofocus.md](autofocus.md).

**Direction:** L→B. **Class:** normal (`0x01`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Timing | every second loop, in place of 0x06 | **PROBABLE** |
| Layout | the 0x06 payload behind 3 bytes: `pl[i + 3]` = 0x06 `pl[i]` | **PROBABLE** (two sessions) |
| Prefix | `pl[0]` varies (`7E`, `22`); `pl[1]` copies 0x06 `pl[10]`; `pl[2]` = `06` | `pl[1]` **PROBABLE**; the rest **POSSIBLE** |
| Differences | 0x06 `pl[6]` (`30`/`70`) is `pl[9]` (`50`/`10`); 0x06 `pl[23]` and `pl[26]` alternate `00`/`40` in opposite phase | **POSSIBLE** |

## Open questions

- What the prefix is for.
