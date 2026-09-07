# Message 0x4C — beyond the capability bitmap

**Summary.** An ID implemented by two manufacturers that **cannot be expressed in the 64-bit
capability bitmap at all**, with UNKNOWN contents.

**Direction:** UNKNOWN. **Class:** UNKNOWN.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 16 bytes, all zero |
| Yongnuo | 7 bytes — `28 28 00 …` |

## It lies outside the bitmap

[Message 0x01](msg_0x01.md)'s bitmap is 64 bits under `bit n ⇒ ID n+1`, so it can express IDs
`0x01`…`0x40` only. `0x4C` = 76 is past that ceiling, so **no device can advertise this message and
no body can learn of it from the handshake.**

That gives the `pl[8..31]` region of message 0x01 — 24 bytes that are zero on every device observed
— a plausible purpose: extension space for IDs above 64. That reading is UNKNOWN, but IDs like this
one are what would need it.

[Message 0x5A](msg_0x5A.md) is in the same position.

## Open questions

- How a body would ever ask for a message it cannot be told about.
- What Yongnuo's `28 28` opening encodes.
- Whether `pl[8..31]` of message 0x01 is the extension space for these IDs.
