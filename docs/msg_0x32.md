# Message 0x32 — implemented but not advertised

**Summary.** A 16-byte all-zero response implemented by one manufacturer, which does **not**
advertise the ID in its capability bitmap. Dead or legacy.

**Direction:** UNKNOWN. **Class:** UNKNOWN. **Payload:** 16 bytes, all zero.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Implementation | Advertised in [message 0x01](msg_0x01.md)? |
| --- | --- | --- |
| TECHART | LM-EA9 — 16 bytes, all zero | **No** |

The combination is what makes it interesting: the device can answer the message but never tells the
body it can. Since the capability bitmap is what the body works from, this response is unreachable
in normal operation.

An all-zero payload also carries no information, so even if it were requested, nothing would be
learned from it.

## Open questions

- Whether the ID is a legacy leftover or was deliberately withdrawn.
- What a non-zero 0x32 response would contain.
