# Message 0x03 — body status / command, per frame

**Summary.** The body's per-frame command and state block, and the richest body→lens message. It
is frame 3 of the four-frame 60 Hz loop, sent after the lens has already reported with 0x05 and
0x06.

**Direction:** B→L only.

**Class:** normal (`0x01`).

**Payload:** 23 bytes to natives, **20 bytes to the Viltrox adapter** — from the same Sony A6000.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Example — Sony SELP1650

```
bb 2e 00 bd 13 bd 13 1c 00 00 06 04 00 02 00 03 01 00 00 00 2f 15 17
```

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..1]` | u16 LE. Takes body-side values (`0x399c` = 14748, `0x2ebb` = 11963, `0x1a2c` = 6700). **The same values appear across two different lenses**, so it is body state, not lens data. | UNKNOWN |
| `pl[3..4]`, `pl[5..6]` | **Duplicated u16 LE pair = commanded focus position.** SELP1650: `bd 13` = 5053, which equals the lens's own reported position in its first 0x05. Constant through idle, when no AF is commanded. | **PROBABLE** |
| `pl[7]` | `0x94` at start of session, `0x1c` thereafter | UNKNOWN |
| `pl[10]`, `pl[11]` | Small counters — `pl[11]` cycles 0…5 | POSSIBLE (frame/phase counter) |
| `pl[12]` | 0 or 1 | UNKNOWN |
| `pl[15]` | `02` or `03` | UNKNOWN |
| `pl[20]` | `0x2f` constant on natives — an instruction tag rather than an arbitrary constant, see below | PROBABLE |
| `pl[21..22]` | **Table row index pair** — same value space as message 0x05's `pl[77..78]` (`0x15 0x16 0x17`) | PROBABLE that it is the same index; see below |

## The tail is bound to the table-transfer mechanism

The **20-byte variant sent to the Viltrox adapter has no `2f` + index tail at all**. The body
simply stops asking a device that never answers. That is direct evidence the tail belongs to the
optical-table transfer, and it is one of the clearer native/adapter behavioural differences on
record.

It also means the body sizes this message according to what the lens declared during init — do not
assume a fixed 23-byte layout.

## `0x2f` is an instruction tag

`0x2f` at `pl[20]`, followed by two operand bytes, is very likely **"row-index select, two operand
bytes follow"** rather than a constant. PROBABLE.

The evidence is in Yongnuo's implementation, which handles `0x2f` exactly that way: it reads the
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

Knowing that `0x2f` is an instruction tag upgrades what the byte *means* without answering this
question: it says nothing about whether the body issues the instruction or merely carries a value
the lens already uses internally.

## Manufacturer notes

- **Sony** — natives receive the full 23-byte form with the `2f` + index tail.
- **Viltrox** — the EF adapter receives the 20-byte form, tail omitted, from the same body.
- **Yongnuo** — treats `0x2f` as a row-index-select instruction with two operand bytes, and
  derives message 0x05's `pl[80]` locally instead of echoing the body.

## Open questions

- `pl[0..1]`: body-side value, purpose UNKNOWN.
- `pl[7]`: why `0x94` only on the first frame of a session. UNKNOWN.
- `pl[12]`, `pl[15]`: two-valued fields, meaning UNKNOWN.
- `pl[2]`, `pl[8..9]`, `pl[13..14]`, `pl[16..19]`: not yet assigned a meaning.
- Whether the row index at `pl[21..22]` is a request from the body or a value shared by both
  sides. Unresolved — see above.
