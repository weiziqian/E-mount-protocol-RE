# Message 0x5E — aperture and focus status pair

**Summary.** The Sony SEL40F25G and SEL24F28G send two of these every loop, from the body's [0x5D](msg_0x5D.md) on; the SELP1650 sends none, and the SEL40F25G and SEL24F28G send none in movie mode on an a7CR.

**Direction:** L→B. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Layout

43 payload bytes (frame length 52).

| `pl` | Meaning | Confidence |
| --- | --- | --- |
| `0` | counter, +1 per frame | **PROBABLE** |
| `3` | the lens's loop counter ([0x42](msg_0x42.md) `pl[0]`) plus 1 | **POSSIBLE** |
| `5..6` | the current aperture ([0x05](msg_0x05.md) `pl[0..1]`). The second frame of the pair carries only this field | **PROBABLE** |
| `7..8` | the aperture target (the body's [0x03](msg_0x03.md) `pl[5..6]`) | **PROBABLE** |
| `9..10` | rises with the aperture: 3252 at `0x12F9` to 6413 at `0x16EB` on a Sony SEL24F28G, the same slope on the SEL40F25G; likely the iris drive position | **POSSIBLE** |
| `14..15` | focus position, equal to [0x06](msg_0x06.md) `pl[20..21]` | **PROBABLE** |
| `20..21` | focus position, within about 10 of 0x06 `pl[2..3]` | **POSSIBLE** |
| `22..23`, `24..25` | the focus target: the position of the last [0x04](msg_0x04.md) Move | **PROBABLE** |
| `26..27` | lower focus travel limit, equal to 0x06 `pl[7..8]` | **PROBABLE** |
| `28..29` | upper focus travel limit in its manual-focus form: equal to 0x06 `pl[9..10]` in manual focus, above it in AF by 574 (SEL40F25G) or 631 (SEL24F28G) | **PROBABLE** |
| the rest | **UNKNOWN** | |

## Open questions

- What stops it in movie mode on an a7CR. Sent to a SEL24F28G or SEL40F25G, the movie-mode [0x53](msg_0x53.md) and [0x41](msg_0x41.md), 0x03
  `pl[7]` = `08`, `pl[12]` = `28`, and 0x04 `pl[6..7]` = `7b 40`, `pl[10]` = `09` (as in movie recording), alone or together, did not.
- The meaning of `pl[11..13]` (`d0`, `d4` and others), and of `pl[30..32]` (`1c 00 02` throughout).
