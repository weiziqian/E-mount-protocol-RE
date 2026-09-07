# Message 0x16 — implemented, never observed carrying data

**Summary.** An ID implemented by more than one manufacturer, with UNKNOWN contents. Notable for
being the **only message anywhere that uses `class = 0x00`**.

**Direction:** UNKNOWN.

**Class:** `0x00` on the TECHART LM-EA9 — a value no other frame in the protocol uses. UNKNOWN
whether that is meaningful or a defect in that implementation.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Implementation |
| --- | --- |
| TECHART | LM-EA9 — response uses `class = 0x00`, unique in the protocol |
| Yongnuo | 1-byte response `00`, identical across its whole lineup |

There is a report of a body sending 0x16 after 0x1D/0x20. Given that
[0x1D](msg_0x1D.md) and [0x20](msg_0x20.md) are almost certainly frame *lengths* of message 0x03
rather than message IDs, that report reads as **a 22-byte (`0x16`) message 0x04 frame** in the same
length notation — which is exactly the observed message 0x04 length. So it is probably not evidence
of message 0x16 on the wire at all.

## Open questions

- Everything about the contents. UNKNOWN.
- Whether `class = 0x00` is a real third class or an implementation defect.
- The direction.
