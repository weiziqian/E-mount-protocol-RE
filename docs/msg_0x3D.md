# Message 0x3D — requested during init, contents unknown

**Summary.** An init-class exchange requested by later Sony bodies, implemented by two
manufacturers at **different lengths**, with UNKNOWN contents.

**Direction:** both. **Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Requested during init by later bodies

A Sony a9 II requests this message during the init handshake, between the `0x0B` and `0x08`
exchanges. A Sony A6000 never requests it — the ID lies beyond the range that body offers in its
[capability bitmap](msg_0x01.md). CERTAIN on both bodies.

A device that answers only the A6000's nine init messages will stall here on a later body.

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
