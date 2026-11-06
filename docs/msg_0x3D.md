# Message 0x3D — the lens's file and feature declarations

**Summary.** An init-class exchange requested by later Sony bodies. The reply declares which
0x3A/0x3B files the lens has and which optional status fields it sends.

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

## Lens reply — a file bitmap and feature bits

63-byte payload (72-byte frame). The A7 IV reads it as declarations: which
[0x3A/0x3B](msg_0x3A.md) files the lens has, and which optional 0x05/0x06 fields it sends.
CERTAIN on the A7 IV firmware, except where marked.

| Bits | Meaning |
| --- | --- |
| `pl[0..1]`, bit `n` | the lens has 0x3A/0x3B file `n` ([table](msg_0x3A.md#other-files--certain-on-the-a7-iv-firmware)) |
| `pl[33]` bit 2 | [0x05](msg_0x05.md) `pl[54..55]` (coarse focus position) is valid |
| `pl[33]` bit 4 | 0x05 field `12` is sent |
| `pl[33]` bit 7 | 0x05 field `13` is sent |
| `pl[34]` bit 2 | the [0x06](msg_0x06.md) in-frame position trace is used — a7S III firmware ([lens types](lens_types.md)) |
| `pl[34]` bit 5 | 0x05 field `14` is sent |
| `pl[35]` bits 4, 5 | 0x06 fields `49`, `4A` are sent |
| `pl[50]` bit 3 | 0x05 field `15` is sent |

The field bits are copied into the body's [0x0A](msg_0x0A.md) request, so a lens that sets one
**must** then send that field.

| Manufacturer | Reply |
| --- | --- |
| TECHART | LM-EA9: `pl[0] = 01`, the rest zero |
| Yongnuo | identical across its whole lineup: `pl[0] = 01`, `pl[34] = 01` (a bit not read above) |
| TTArtisan | 50/1.8: `pl[0] = 01`, `pl[33] = 0C` (bits 2 and 3) |

All three declare file 0 only.

It is the highest ID the Sony A6000 offers in [message 0x01](msg_0x01.md).

## Open questions

- The meaning of the request's `bd`. UNKNOWN.
- `pl[33]` bit 3, `pl[34]` bit 0, and the rest of the payload.
- Whether the A6000 offering exactly up to this ID is significant.
