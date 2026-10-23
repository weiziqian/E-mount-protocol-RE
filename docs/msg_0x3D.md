# Message 0x3D — requested during init, contents unknown

**Summary.** An init-class exchange requested by later Sony bodies, with UNKNOWN contents.

**Direction:** both. Body request 23 bytes. **Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Requested during init by later bodies

A Sony a9 II requests this message during the init handshake, between the `0x0B` and `0x08`
exchanges. A Sony A6000 never requests it — the ID lies beyond the range that body offers in its
[capability bitmap](msg_0x01.md). CERTAIN on both bodies.

A device that answers only the A6000's nine init messages will stall here on a later body.

## Body request

Observed once, from a Sony a9 II: payload `bd` followed by 22 zero bytes. Checksum verified.

```
F0 20 00 02 00 3D | BD 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | 1C 01 55
```

What `bd` means is UNKNOWN.

## Lens reply

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 63-byte payload (72-byte frame): `01` followed by zeros |
| Yongnuo | 63-byte payload, identical across its whole lineup |

The ID is inside the 64-bit bitmap's range, so it can be advertised in
[message 0x01](msg_0x01.md). It is also the highest ID the Sony A6000 offers.

## Open questions

- Contents of the reply, and the meaning of the request's `bd`. UNKNOWN.
- Whether the A6000 offering exactly up to this ID is significant.
