# Message 0x19 — implemented, never observed carrying data

**Summary.** An ID implemented by one manufacturer, with UNKNOWN contents. Nothing has been
observed on the wire.

**Direction:** UNKNOWN. **Class:** UNKNOWN.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Implementation |
| --- | --- |
| TECHART | LM-EA9 — a runtime-built response exists |

The ID falls inside the 64-bit capability bitmap's range, so a device can advertise it in
[message 0x01](msg_0x01.md).

## Open questions

- Contents, direction, class, and length. All UNKNOWN.
- Whether any body ever requests it.
