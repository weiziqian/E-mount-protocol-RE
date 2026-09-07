# Message 0x3D — implemented, never observed carrying data

**Summary.** An ID implemented by two manufacturers at **different lengths**, with UNKNOWN contents.

**Direction:** UNKNOWN. **Class:** UNKNOWN.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 72 bytes: `01` followed by zeros |
| Yongnuo | 63 bytes, identical across its whole lineup |

The two lengths disagree, which means either the message is variable-length — like
[message 0x06](msg_0x06.md), which has a 43-byte core plus an optional appendix — or one of the two
implementations is wrong about it.

The ID is inside the 64-bit bitmap's range, so it can be advertised in
[message 0x01](msg_0x01.md). It is also the highest ID the Sony A6000 offers.

## Open questions

- Contents. UNKNOWN on both implementations.
- Why the two lengths differ.
- Whether the A6000 offering exactly up to this ID is significant.
