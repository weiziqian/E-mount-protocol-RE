# Message 0x01 — capability bitmap

**Summary.** Each side declares which message IDs it supports. This is the first exchange of the
init phase, and it is what the rest of the session is negotiated against.

**Direction:** both. The body sends its offer first; the lens replies with its own set.

**Class:** init (`0x02`). **Payload:** 32 bytes in both directions.

Payload offsets below are written `pl[n]`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Payload — identical layout both ways

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..31]` | **256-bit LE bitmap. Bit `n` (byte `n/8`, bit `n%8`) set ⇒ message ID `n+1` is supported.** | **CERTAIN** |

The bitmap is the whole 32-byte payload, not the first 8 bytes: the Sony A7 IV reads bit `id−1` for
any ID up to `0x60`, and offers IDs up to `0x60` itself (below). So IDs above `0x40` — `0x4C`,
`0x4E`, `0x5A` — are declared in `pl[8..11]` like any other. CERTAIN on the A7 IV firmware.

## The off-by-one is not optional

Bit `n` means ID `n+1`, not ID `n`. Decoding the TECHART LM-EA9's mask
`ff 9f 60 7d 80 07 08 50` under `bit n ⇒ ID n+1` yields

```
01 05 06 07 08 09 0a 0b 0c 0d 10 16 19 1b 28 34 3d 3f
```

— exactly the 18 IDs that device implements, 18 for 18. Under `bit n ⇒ ID n` the IDs `0x0d` and
`0x10` would land on clear bits, and both are exchanged on the wire during init. CERTAIN.

## The reply is not masked by the offer

A lens may assert an ID the body never offered. The TECHART LM-EA9 asserts bit 62 — ID `0x3f`, the
lens-name message — which the A6000 does not offer. CERTAIN.

So the two masks are independent declarations: a lens should declare what it implements, not
echo the offer.

**The body does intersect them when deciding what to ask for.** The A7 IV treats an ID as usable
only if its bit is set in the lens's reply **and** in its own mask; ID `0x01` is always usable.
Every optional step of its handshake ([§3.1](../README.md#31-init-phase--class-0x02-strict-requestresponse))
is gated this way. CERTAIN on the A7 IV firmware.

## Observed masks

Body offers grow with generation:

| Body | Offer | Highest ID offered |
| --- | --- | --- |
| Sony NEX-7 | `ff ff ff ff ff ff ff 01` | `0x39` |
| Sony A6000 | `ff ff ff ff ff ff ff 1f` | `0x3d` |
| Sony A7 IV (firmware) | `ff ff ff ff ff ff ff ff ff ef ff ff` then zeros | `0x60`, all except `0x4D` |

The A7 IV has a second, reduced mask offering only `01 07 0C 14 15`; when it is used is UNKNOWN.

Lens replies (first 8 bytes; the rest is zero on every lens measured):

| Device | Mask | # IDs |
| --- | --- | --- |
| **Yongnuo YN35 / 50F1.8S** (native AF) | `ff 9f ff 5d ee 60 18 5e` | **42** |
| Sony SELP1650 (native) | `ff 9f ff 7d ee 60 08 00` | 37 |
| Sony SEL55210 (native) | `ff 9f ff 5d ee 60 08 08` | 37 |
| Viltrox EF adapter | `ff 9f 7d 5f 80 07 00 00` | 30 |
| **TECHART LM-EA9** | `ff 9f 60 7d 80 07 08 50` | **29** |

Byte 2 (`ff` native vs `60`/`7d` adapter) and byte 4 (`ee` native vs `80` adapter) separate the two
groups cleanly, which makes this message a usable native/adapter discriminator.

## Manufacturer notes

- **TECHART** — the LM-EA9 is the least capable device measured, at 29 IDs. Its reply is a fixed
  constant: it sends the same 8 bytes regardless of what the body offered.
- **Yongnuo** — the YN35 and 50F1.8S assert the largest set seen, 42 IDs, above both Sony natives
  measured.
- **Sony** — the two natives measured differ from each other only in the last byte.
- **Viltrox** — the EF adapter asserts 30 IDs and sits with the adapters on both discriminator
  bytes.

## Open questions

- The mask is a *claim*, and nothing verifies it. A device that asserts an ID it has no handler for
  can stall init when the body actually uses that ID, so an implementation should assert only what
  it can answer.
