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
| `pl[3..4]`, `pl[5..6]` | **u16 LE pair, both [aperture values](aperture_value.md).** `pl[5..6]` is the aperture the body is asking for. `pl[3..4]` is `0x1000` = 4096 = f/1.0 in the first frame of a session and equal to `pl[5..6]` in every frame after it. `pl[3..4]` is the body's iris drive target; `pl[5..6]` is a second target that the body updates only in one exposure-control state and otherwise holds. [Below](#where-the-body-takes-the-aperture-pair-from) | Both are apertures **CERTAIN**; sources **CERTAIN on the a7C II firmware**; what the exposure state means **UNKNOWN** |
| `pl[7]` | `0x94` at start of session, `0x1C` thereafter. Holds on both bodies. **Bit 7 selects the live aperture field**: set → `pl[5..6]`, clear → `pl[3..4]`. Bits 2, 3 and 4 are three further exposure-control flags, meaning UNKNOWN | Bit 7 **CERTAIN on the a7C II firmware**; other bits UNKNOWN |
| `pl[10]`, `pl[11]` | **Body cycle**: `pl[10]` is the cycle length N in frames, `pl[11]` the frame within it, 0…N−1. The lens follows it in [message 0x06](msg_0x06.md) `pl[23]`. | **PROBABLE** |
| `pl[12]` | 0 or 1 | UNKNOWN |
| `pl[15]` | `02` or `03` | UNKNOWN |
| `pl[20]` | `0x2F` constant on natives — an instruction tag rather than an arbitrary constant, see below | PROBABLE |
| `pl[21..22]` | **Table row index pair** — same value space as message 0x05's `pl[77..78]` (`0x15 0x16 0x17`) | **CERTAIN**: the lens echoes it two cycles later, see below |

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

### Where the body takes the aperture pair from

CERTAIN on the a7C II firmware, unless marked.

The body assembles message 0x03 each frame from fillers registered by three subsystems. The
exposure-control filler writes `pl[0..7]`, from a per-frame snapshot of the exposure-control state:

| Field | Source |
| --- | --- |
| `pl[3..4]` | **The iris drive target.** Exposure control sets it with a mode and a value; mode 0 means "drive the iris to this value". It starts at `0x1000` (f/1.0) with a non-drive mode, which is the first-frame value above. Auto-exposure mostly sets it to its live-view iris target, never wider than the lens's maximum aperture |
| `pl[5..6]` | A second target, updated only while auto-exposure is in one particular state (meaning UNKNOWN) and held otherwise. It also starts at `0x1000`. A body-side override can replace it, and then forces `pl[7]` bit 7 on |
| `pl[7]` bit 7 | Which of the two is live, see the field map |

In another AF mode the body writes one computed value to both `pl[3..4]` and `pl[5..6]`.

**This is how the body drives the iris of a native lens in live view.** For a lens that supports the
asynchronous iris message, the body copies the live field (`pl[5..6]` if `pl[7]` bit 7 is set, else
`pl[3..4]`) into [message 0x51](msg_0x51.md) every frame. A third-party lens never gets 0x51, so for
it message 0x03 is the only place the value appears between captures.

Whether the live field equals the user's chosen exposure aperture at all times is **UNKNOWN**. It is
the target for the iris *during live view*. Auto-exposure decides when that is the chosen aperture
and when it stays wider, for example for focusing. Every capture on record shows the lens's maximum
aperture, so captures cannot yet tell those apart. The exposure aperture of a capture is sent in
[message 0x1B](msg_0x1B.md) or 0x51 at the release.

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

## The body chooses the row; the lens answers two cycles later

Each message 0x05 index `pl[77..78]` equals the `pl[21..22]` of the **second-to-last** message 0x03
the body sent before it. The lens answers with a delay of two cycles.

| Device | lag 1 | **lag 2** | lag 3 |
| --- | --- | --- | --- |
| Sony SELP1650 | 142/214 | **213/213** | 140/212 |
| Sony SEL55210 @55 | 48/75 | **74/74** | 50/74 |
| Sony SEL55210 @210 | 92/134 | **134/134** | 86/133 |

Frames are paired in time order. Pairing by `seq` alone gives false misses, because `seq` wraps at
256. An earlier version of this section tested only lags 0 and 1 and called causality
unestablished. **CERTAIN** on these three sessions, all with an A6000.

The body plays a pattern of its own, for example `15 16` ×3 then `15 17` ×3 to the SELP1650, and the
lens repeats that pattern two frames later.

The lens picks the index itself only before the first request arrives. The first frame is `00 00`;
the next is a tag of the lens's own, `13 14` on the SELP1650 and the SEL55210 @55 or `11 12` on the
SEL55210 @210, with `pl[80]` already set from the lookup.

`pl[80]` is not echoed. The body never sends it, and the lens computes it from the tag
([message 0x05](msg_0x05.md#pl80-and-modern-body-compatibility)).

## Manufacturer notes

- **Sony** — natives receive the full 23-byte form with the `2F` + index tail.
- **Viltrox** — the EF adapter receives the 20-byte form, tail omitted, from the same body.
- **Yongnuo** — treats `0x2F` as a row-index-select instruction with two operand bytes, and
  derives message 0x05's `pl[80]` locally instead of echoing the body.

## Open questions

- `pl[0..1]`: body-side value, purpose UNKNOWN.
- `pl[7]` bits 2–4, and why bit 7 clears and bit 3 sets after the first frame. UNKNOWN.
- Which exposure state updates `pl[5..6]`, and whether the live field follows the user's aperture setting in stills live view.
- `pl[12]`, `pl[15]`: two-valued fields, meaning UNKNOWN.
- `pl[2]`, `pl[8..9]`, `pl[13..14]`, `pl[16..19]`: not yet assigned a meaning.
- How the body chooses which rows to request.
