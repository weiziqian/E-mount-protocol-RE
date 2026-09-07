# Message 0x3F — lens name string

**Summary.** The human-readable lens name, as ASCII.

**Direction:** L→B.

**Class:** init (`0x02`).

**Payload:** 65 bytes on the TECHART LM-EA9 — ASCII, NUL-padded, with a leading `00`.

**Never observed on the wire.** The Sony A6000 does not offer ID `0x3F`. The LM-EA9 asserts bit 62
and would reply anyway, so it is presumably sent to bodies that ask.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

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

- **TECHART** — asserts bit 62 for this ID even though no body measured offers it.
- **TTArtisan**, **Meike** — both implement the message; the Meike string ships with a typo.
- **Sony**, **Yongnuo**, **Viltrox** — not observed implementing it.

## Open questions

- Whether any body requests the message, and what it does with the string.
- Whether the payload length is fixed at 65 bytes or sized to the string.
- What the leading `00` byte selects, if anything.
