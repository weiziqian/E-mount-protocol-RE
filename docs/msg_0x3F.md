# Message 0x3F — lens name string

**Summary.** The human-readable lens name, as ASCII.

**Direction:** L→B.

**Class:** init (`0x02`).

**Payload:** 65 bytes on the TECHART LM-EA9 — ASCII, NUL-padded, with a leading `00`.

**Requested by later bodies.** The Sony A6000 does not offer ID `0x3F` and never asks for it; a
Sony a9 II requests it during every init handshake.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Requested during init by later bodies

A Sony a9 II requests this message during the init handshake, between the `0x0B` and `0x08`
exchanges. A Sony A6000 never requests it — the ID lies beyond the range that body offers in its
[capability bitmap](msg_0x01.md). CERTAIN on both bodies.

A device that answers only the A6000's nine init messages will stall here on a later body.

## Strings observed

| Device | String |
| --- | --- |
| TECHART LM-EA9 v1.0.0 – v1.4.0 | `EF40mm f/2.8 STM` |
| TECHART LM-EA9 v1.5.0 – v1.8.0 | `TECHART LM-EA9` |
| TTArtisan | `TTARTISAN 75mm F2.0`, `TTARTISAN 40mm F2.0` |
| Meike | `MEKE 35mmF2.0 ` (vendor typo, shipped) |

## The name and the specification can disagree

The LM-EA9 was renamed from `EF40mm f/2.8 STM` to `TECHART LM-EA9` at v1.5.0. **It still reports the
Canon EF 40 mm's *specification* while calling itself "TECHART LM-EA9"** — its message 0x05 is
byte-identical across the rename, including the 40.0 mm focal length, and its aperture descriptor
still starts from the EF 40 mm's f/2.8.

So this string is independent of every other identity field. A body that trusted it would disagree
with what message 0x05 and [message 0x07](msg_0x07.md) report.

The rename is also what produces one of the stale stored checksums described in
[frame format](frame_format.md#implementations-recompute-the-checksum-on-send): the byte-sum
difference between the two strings is exactly 122.

## Manufacturer notes

- **TECHART** — asserts bit 62 for this ID. The A6000 does not offer it, but the a9 II requests it,
  so the bit is not the dead weight it appears to be against an older body.
- **TTArtisan**, **Meike** — both implement the message; the Meike string ships with a typo.
- **Sony**, **Yongnuo**, **Viltrox** — not observed implementing it.

## Open questions

- What a body does with the string. The a9 II requests it during init; whether it is displayed,
  logged, or used to select behaviour is UNKNOWN.
- Whether the payload length is fixed at 65 bytes or sized to the string.
- What the leading `00` byte selects, if anything.
