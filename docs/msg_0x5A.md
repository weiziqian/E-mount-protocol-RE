# Message 0x5A — optional init exchange

**Summary.** A 64-byte response implemented by one manufacturer. The body's request carries a
bitmap of its own. Contents UNKNOWN.

**Direction:** both. **Class:** init (`0x02`). **Payload:** 64 bytes.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 64 bytes, mostly zero, with `07` at `pl[32]` |

The single non-zero byte at `pl[32]` is the only structure in it.

## Where it sits in the handshake

The A7 IV requests 0x5A after 0x3F, 0x32 and 0x3D and before 0x08, only if the lens declares it in
[message 0x01](msg_0x01.md) (bit 89, `pl[11]` bit 1). Its request carries a 32-byte bitmap, in the
[message 0x01](msg_0x01.md) layout, of the IDs the body ended up enabling — the intersection of both
sides' masks — followed by a flags byte. The A7 IV reads reply `pl[4]` and
`pl[32]`. CERTAIN on the A7 IV firmware; meanings UNKNOWN.

## Open questions

- Everything about the contents beyond the lone `07`.
