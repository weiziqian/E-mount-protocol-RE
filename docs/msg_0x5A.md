# Message 0x5A — beyond the capability bitmap

**Summary.** A 64-byte response implemented by one manufacturer, at an ID that **cannot be expressed
in the 64-bit capability bitmap at all**. Contents UNKNOWN.

**Direction:** UNKNOWN. **Class:** UNKNOWN. **Payload:** 64 bytes.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 64 bytes, mostly zero, with `07` at `pl[32]` |

The single non-zero byte at `pl[32]` is the only structure in it.

## It lies outside the bitmap

[Message 0x01](msg_0x01.md)'s bitmap is 64 bits under `bit n ⇒ ID n+1`, so it can express IDs
`0x01`…`0x40` only. `0x5A` = 90 is well past that ceiling, so **no device can advertise this message
and no body can learn of it from the handshake.**

[Message 0x4C](msg_0x4C.md) is in the same position. Both are candidates for whatever
`pl[8..31]` of message 0x01 is reserved for.

## Open questions

- Everything about the contents beyond the lone `07`.
- How a body would ever ask for a message it cannot be told about.
