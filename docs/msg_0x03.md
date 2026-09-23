# Message 0x03 — body status / command, per frame

**Summary.** The body's per-frame command and state block, and the richest body→lens message. It
is frame 3 of the four-frame 60 Hz loop, sent after the lens has already reported with 0x05 and
0x06.

**Direction:** B→L only.

**Class:** normal (`0x01`).

**Payload:** 23 bytes to natives, **20 bytes to adapters**. The A6000 sends 23 bytes to Sony
natives and 20 to the Viltrox EF adapter; a Sony a9 II sends 20 bytes to a TECHART LM-EA9. Two
bodies, two adapters, the same rule.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Examples

Sony A6000 → SELP1650, 23-byte native form:

```
BB 2E 00 BD 13 BD 13 1C 00 00 06 04 00 02 00 03 01 00 00 00 2F 15 17
```

Sony a9 II → TECHART LM-EA9, 20-byte adapter form, two consecutive frames early in a session:

```
6C 3D 00 00 10 00 12 94 00 00 01 00 01 02 00 03 01 00 00 00
F6 31 00 00 12 00 12 94 00 00 01 00 00 02 00 03 01 00 00 00
```

The tail `02 00 03 01 00 00 00` at `pl[13..19]` is byte-identical between the two bodies. What the
adapter form drops is the `2F` + index tail at `pl[20..22]`, nothing else.

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..1]` | u16 LE. Takes body-side values (`0x399C` = 14748, `0x2EBB` = 11963, `0x1A2C` = 6700 on an A6000; `0x3D6C` = 15724, `0x31F6` = 12790 on an a9 II). **The same values appear across two different lenses**, so it is body state, not lens data. Changes every frame. | UNKNOWN |
| `pl[3..4]`, `pl[5..6]` | **u16 LE pair, both [aperture values](aperture_value.md).** `pl[5..6]` is the aperture the body is asking for. `pl[3..4]` is `0x1000` = 4096 = f/1.0 in the first frame of a session and equal to `pl[5..6]` in every frame after it. | Both are apertures **CERTAIN**; what distinguishes the two fields **UNKNOWN** |
| `pl[7]` | `0x94` at start of session, `0x1C` thereafter. Holds on both bodies. | UNKNOWN |
| `pl[10]`, `pl[11]` | Small counters — `pl[11]` cycles 0…5 | POSSIBLE (frame/phase counter) |
| `pl[12]` | 0 or 1 | UNKNOWN |
| `pl[15]` | `02` or `03` | UNKNOWN |
| `pl[20]` | `0x2F` constant on natives — an instruction tag rather than an arbitrary constant, see below | PROBABLE |
| `pl[21..22]` | **Table row index pair** — same value space as message 0x05's `pl[77..78]` (`0x15 0x16 0x17`) | PROBABLE that it is the same index; see below |

### The aperture pair

`pl[5..6]` is an [aperture value](aperture_value.md), and through an idle session it holds one value
per device — that device's maximum aperture:

| Device | `pl[5..6]` | AV | Aperture | Marked maximum |
| --- | --- | --- | --- | --- |
| Viltrox EF adapter + Canon EF 50 mm f/1.8 | 4544 | 1.750 | f/1.83 | f/1.8 |
| Viltrox EF adapter + Canon EF-S 24 mm f/2.8 | 4864 | 3.000 | f/2.83 | f/2.8 |
| Sony SELP1650 at 16 mm | 5053 | 3.738 | f/3.65 | f/3.5 |
| Sony SEL55210 at 55 mm | 5224 | 4.406 | f/4.60 | f/4.5 |
| Sony SEL55210 at 210 mm | 5470 | 5.367 | f/6.42 | f/6.3 |

The lens acts on it. With the Viltrox adapter and the EF 50 mm the body sends 4544 from the first
frame of the session, and the lens's [message 0x05](msg_0x05.md) `pl[0..1]` walks toward it in
one-stop steps over the next four frames — 6336, 6080, 5824, 5568, then 4544, where it stays.

A lens may clamp the value into its own aperture range before acting on it: below the lens's maximum
aperture the command is taken as "wide open", and above its minimum it is taken as that minimum.

Once settled, the aperture the lens reports back in message 0x05 is not always the one commanded.
The two Viltrox adapters report exactly the commanded value; the three Sony lenses report 21–30
counts wider, about a tenth of a stop.

`pl[3..4]` carries an aperture on the same scale. It is `4096` — `AV 0`, f/1.0, the bottom of the
encoding — in the first message 0x03 frame of a session, in all five sessions observed, and equal to
`pl[5..6]` in every one of the 840 frames after it. In the one session where `pl[5..6]` changed
between the first and second frames, from 5224 to 5470, `pl[3..4]` took 5470 immediately rather than
the earlier value. What the field is for is **UNKNOWN**.

`pl[14..15]` of [message 0x04](msg_0x04.md) is a focus target on a different scale whose numeric
range overlaps this one; the two are not interchangeable.

## The tail is bound to the table-transfer mechanism

The **20-byte variant sent to an adapter has no `2F` + index tail at all** — the A6000 to the
Viltrox, the a9 II to the TECHART LM-EA9. The body simply stops asking a device that never answers. That is direct evidence the tail belongs to the
optical-table transfer, and it is one of the clearer native/adapter behavioural differences on
record.

It also means the body sizes this message according to what the lens declared during init — do not
assume a fixed 23-byte layout.

## `0x2F` is an instruction tag

`0x2F` at `pl[20]`, followed by two operand bytes, is very likely **"row-index select, two operand
bytes follow"** rather than a constant. PROBABLE.

The evidence is in Yongnuo's implementation, which handles `0x2F` exactly that way: it reads the
next two bytes as an `(idx_lo, idx_hi)` pair and feeds them into the same row-index mechanism that
drives message 0x05's index field. That is the exact byte value and the exact operand shape seen
at `pl[20..22]` above.

Candidate semantics for the operand pair, from that same implementation — vendor-specific until
seen on a second lens, not confirmed as protocol-wide:

| `idx_lo` | Meaning |
| --- | --- |
| `0` | Nothing new to report |
| `7` or `9` | Retransmission of previously-sent data |
| `0x15` (main-loop range `0x15`–`0x17`) | Genuine new data |

POSSIBLE.

## Who drives the row index — deliberately left open

The same 2-byte index values appear in the body's 0x03 and in the lens's 0x05, drawn from the same
small set. It is tempting to conclude that the body requests rows and the lens answers. The
alignment does not support that cleanly:

| Device | body idx == same-`seq` lens idx | body idx == *next* lens idx |
| --- | --- | --- |
| Sony SELP1650 | 59/180 | 118/180 |
| Sony SEL55210 @55 | 24/74 | 39/74 |
| Sony SEL55210 @210 | 47/131 | 68/131 |

"Predicts the next" wins consistently but is nowhere near a lock. So: **the index is shared
between the two directions; causality is not established.** POSSIBLE, not PROBABLE.

One further data point does not settle it either. Yongnuo's implementation computes message
0x05's `pl[80]` *locally* from its own `pl[78]`, so at least that derived byte is lens-side
arithmetic rather than an echo of anything the body sent. That constrains the mechanism a little —
the lens is not merely mirroring the body's tail — but it says nothing about who chooses the tag.

Knowing that `0x2F` is an instruction tag upgrades what the byte *means* without answering this
question: it says nothing about whether the body issues the instruction or merely carries a value
the lens already uses internally.

## Manufacturer notes

- **Sony** — natives receive the full 23-byte form with the `2F` + index tail.
- **Viltrox** — the EF adapter receives the 20-byte form, tail omitted, from the same body.
- **Yongnuo** — treats `0x2F` as a row-index-select instruction with two operand bytes, and
  derives message 0x05's `pl[80]` locally instead of echoing the body.

## Open questions

- `pl[0..1]`: body-side value, purpose UNKNOWN.
- `pl[7]`: why `0x94` only on the first frame of a session. UNKNOWN.
- `pl[12]`, `pl[15]`: two-valued fields, meaning UNKNOWN.
- `pl[2]`, `pl[8..9]`, `pl[13..14]`, `pl[16..19]`: not yet assigned a meaning.
- Whether the row index at `pl[21..22]` is a request from the body or a value shared by both
  sides. Unresolved — see above.
