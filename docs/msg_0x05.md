# Message 0x05 — lens status and optical table transfer

**Summary.** The lens's per-frame status: current aperture, focal length, aperture limits, and the
optical-correction rows streamed a few per frame. Frame 1 of the four-frame 60 Hz loop.

**Direction:** L→B only.

**Class:** normal (`0x01`).

**Payload:** 96 bytes (105-byte frame) or 108 bytes (117-byte frame). A lens uses one variant and
never the other.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Two payload sizes

| Variant | Devices |
| --- | --- |
| 96 B payload / 105 B frame | Sony SELP1650, SEL55210, SEL2870, SEL5518Z; Voigtländer 15 mm; Zeiss Loxia 21 mm; Viltrox EF adapters; TECHART LM-EA9; Yongnuo YN35 f/1.8S DA |
| 108 B payload / 117 B frame | Techart EOS-NEX III in Fn mode; Yongnuo 50F1.8S DF |

The two share a layout: **`pl[0..82]` is identical**, and the long form appends 12 bytes. Slots C
and D are the head of one 25-byte record — the long form carries all 25 at `pl[83..107]`, the short
form the first 13 at `pl[83..95]`. Every field below `pl[83]` sits at the same offset in both, so
offsets carry across variants. **CERTAIN.**

## Reference payloads

```
TECHART     00 13 00 13 00 00 10 00 07 2A 00 2A 00 54 01 54 01 00 00 00 00 07 80 FF 90 01 90 01 0F 01 00 00
LM-EA9      | 00 00 00 00 00 00 | 26 00 00 00 00 00 | 00 00 ... (all zero to the end)

Sony        DB 13 DB 13 00 1E 00 00 07 2A 00 2A 00 54 01 54 01 00 00 00 00 07 98 FF A5 00 A0 00 00 01 00 00
SELP1650    | A0 EA 70 49 1E 0F | E1 B9 12 A6 E1 F6 | ... 15 16 ... DE F2 B6 B9 C2 D3 | D0 F3 5A 54 33 13
```

---

## Field map

| `pl` | Name | Description | Confidence |
| --- | --- | --- | --- |
| **`0..1`** | **Aperture** | u16 LE [aperture value](aperture_value.md), the lens's current aperture | **CERTAIN** |
| `2..3` | Aperture, repeated | The same value again, in every frame of every device observed | **CERTAIN** |
| `4` | **Aperture settle countdown** | The number of status frames still needed for the aperture to reach the value the body commanded; counts down by one per frame to a resting value. [Section below](#the-aperture-settle-countdown-pl4). Frozen `00` on the LM-EA9 | **CERTAIN** |
| `5` | — | A per-device value that jumps for a frame or two around motion. Observed: `0x1E`/`0x5E` SELP1650, `0x2C` SEL5518Z, `0x28`/`0xCA` SEL2870, `0x15`/`0x1A` SEL55210, `0x00` on the manual lenses, Yongnuo and the LM-EA9 | **UNKNOWN** |
| `6..7` | Start-up value | u16, non-zero **only in the first status frame after init**, then zero: 19 SELP1650, 20 SEL55210, 17 SEL5518Z and Viltrox + EF 50. Zero every frame on Yongnuo. The LM-EA9 sends 16 forever, a value no other device sends at all | Behaviour **CERTAIN**; quantity **UNKNOWN** |
| `8` | State | Not a constant. **Bit 0 = aperture at rest**, cleared while the aperture is moving: `0x02` moving / `0x03` at rest on Yongnuo; Sony lenses alternate `0x06`/`0x07`, `0x06` appearing in frames around motion; frozen `0x07` on the LM-EA9. Other bits **UNKNOWN** | Existence **CERTAIN**; bit 0 **PROBABLE** |
| `9..10`, `11..12` | — | Duplicated u16 pair. **42/42** on every Sony lens, both Viltrox adapters and the LM-EA9; **203/203** on both Yongnuo lenses; **0/0** on the Voigtländer and the Loxia across 1267 frames. Splits by autofocus capability, not by vendor | Values **CERTAIN**; the correlation **PROBABLE** |
| `13..14`, `15..16` | — | Same split: **340/340** on Sony and the adapters, **407/407** on Yongnuo, **0/0** on both manual lenses | as above |
| `17..18` | — | u16 LE on the [aperture value](aperture_value.md) grid, floored at `0x11C0` (4544 = f/1.83) with a default of `0x1400` (5120 = f/4). Zero on every Sony lens, both Viltrox adapters and the LM-EA9 | Encoding **CERTAIN**; quantity **UNKNOWN** |
| `19` | — | A boolean. Zero on every device observed | **UNKNOWN** |
| `20..21` | **Subject distance** | u16 LE [distance code](autofocus.md#25-the-distance-code): `384 + 64 × log2(D)`, `D` in metres, so 1 m = 384 and each doubling adds 64; **`0x0700` = infinity, or no distance data**. The Voigtländer 15 mm walks 272, 299, 320, 351, 384, 448, `0x0700` as its ring turns 0.3 m → ∞ — exactly 0.3, 0.4, 0.5, 0.7, 1 and 2 m, the marks on its distance scale. A lens with electronic focus computes it from its focus position every frame. Every Sony lens measured, both Viltrox adapters and the LM-EA9 send a constant `0x0700` | Encoding **CERTAIN**; Sony's constant `0x0700` read as "not reported" **PROBABLE** |
| `22` | **Flags** | **Bit 7** set on every device that reports motion. **Bit 6 = lens in motion**: set while focus is being driven, while the focus ring is being turned (see `pl[60]`), and, on at least one lens, while the aperture is moving. Bits 0–5 are per-lens and static within a session: `0x38` SELP1650, `0x3C` SEL55210, `0x04` SEL2870, `0x08` SEL5518Z, `0x00` Loxia and Voigtländer. Frozen `0x80` on the LM-EA9 | Bit 7 and bit 6 for focus motion **CERTAIN**; bit 6 for aperture motion **POSSIBLE**; bits 0–5 **UNKNOWN** |
| `23` | Subject distance, coarse | Focus-dependent. Over the Voigtländer's 0.3 m → ∞ sweep it takes 55 distinct values climbing monotonically `0x93` → `0xFF`. As a signed byte it is ≈ **−32 × dioptres**: `0x93` = −109, 109/32 = 3.41 dpt = 0.293 m, that lens's minimum focus distance; `0xFF` = −1 ≈ infinity. Sony lenses send a per-session constant (`0xFF` SELP1650 / SEL55210 / SEL5518Z, `0xD9` = 1.22 dpt ≈ 0.8 m SEL2870). The LM-EA9 sends `0xFF` forever | Focus dependence **CERTAIN** (536 frames, one lens); the −1/32 dpt scale **PROBABLE** |
| **`24..25`**, **`26..27`** | **Focal length** | Duplicated u16 LE pair, mm × 10. Wide/tele on a zoom, near-equal on a prime. [Table below](#focal-length) | **CERTAIN** |
| `28..29` | — | u16. Observed: 271 LM-EA9, **310 on both Yongnuo lenses**, 320 SEL5518Z, 312 SEL2870, 272 Voigtländer, 256 SELP1650, 384 SEL55210, **0 on the Loxia and both Viltrox adapters**. Two lenses of different focal length and format sharing one value rules out a per-lens optical quantity; three devices sending 0 while working show the field is optional | **UNKNOWN.** Ruled out: exit-pupil distance, maximum aperture, and "required" |
| `30..31` | — | An affine function of `pl[0..1]`, not independent. On the Voigtländer, `pl[0..1] − pl[30..31] = 5205` exactly across all 41 sampled values of a 461-frame observation. Zero on the LM-EA9, both Viltrox adapters and every Sony lens except the SEL2870 (345) and SEL55210 @210 (246) | **CERTAIN** |
| **`32..37`** | **Optical row, slot A** | [Slot A](optical_data.md#4-slot-a--the-field-sampling-grid). Aperture-independent; changes only when the row index changes | Encoding **CERTAIN**; quantity **POSSIBLE** |
| **`38..43`** | **Optical row, slot B** | [Slot B](optical_data.md#1-what-the-rows-carry). Alternates every frame between a type-1 and a type-0 row. Type 1 scales as `1/F` | Encoding **CERTAIN**; `∝ 1/F` **CERTAIN** |
| **`44..59`** | **Aperture descriptor** | Maximum and minimum aperture in a second encoding. [Section below](#the-aperture-descriptor-pl4459) | Encoding **CERTAIN** |
| `54..55` | Coarse focus position | `pl[54]` = the focus position counted from the infinity end, divided by 256 (0 near infinity); `pl[55]` zero. Non-zero (`0x01`) on the SEL5518Z during its power-on sweep, zero once settled at infinity. Always zero on the LM-EA9 | Behaviour **CERTAIN**; the position reading **POSSIBLE** — "distance still to run" fits the same observations |
| `60` | **Focus ring direction** | Signed byte: `0x01` and `0xFF` for the two directions the manual focus ring is being turned, `0x00` when it is still. Held for a few frames (about ten) after the ring stops. Moves the body commands do not set it. Observed on the Voigtländer, a manual lens: `0xFF` throughout a ring sweep, `0x00` once settled, `0x01` on a brief reverse. Always zero on the LM-EA9 | Ring rotation **PROBABLE**; hold time **POSSIBLE** |
| `61` | — | `0x01` on most devices | **UNKNOWN** |
| `62` | Lens switch | Two values, `0x01` and `0x03`, following a two-position control on the lens barrel. Updated only while no focus move is running or pending. Frozen `0x01` on the LM-EA9. Which value means which switch position is not established | Existence **CERTAIN**; switch reading **POSSIBLE** (supersedes the earlier "drive status" reading) |
| **`77..78`** | **Row index** | A duplicated byte pair naming the row carried in slots C and D. Cycles `0x15, 0x16, 0x17` in the main loop and `0x09, 0x0B, 0x0C` during init. Some values mark **"nothing new this frame"** rather than a position — `0x00`, and on Yongnuo also `0x07`/`0x09`, deliver null rows with distinctive tag bytes. Whether the marker values are shared across vendors is **UNKNOWN** | Index **CERTAIN**; null-marker convention **PROBABLE**; the values **not portable** |
| `79` | Second index pair, byte 0 | `0x00` on every device measured | **UNKNOWN** |
| **`80`** | Second index pair, byte 1 | A 14-entry lookup on the row tag `pl[78]`. [Section below](#pl80-and-modern-body-compatibility) | **CERTAIN** |
| **`81..82`** | **Effective focal length** | mm × 10, the focal length corrected for focus breathing, in the same units as `pl[24..25]`. On the Voigtländer, nominal 150, this field walks 154 → 163 as focus moves far → near. On Yongnuo the value at infinity equals that lens's own `pl[24..25]` — 358 on the YN35, 514 against 512 on the YN50 DF. Zero on every Sony lens, both Viltrox adapters and the LM-EA9 | **PROBABLE**; whether Sony uses the field at all **UNKNOWN** |
| **`83..88`** | **Optical row, slot C** | [Slots C and D](optical_data.md#8-slots-c-and-d). A strict function of `pl[77]` on the SEL2870; carries null rows on some devices | Lookup **CERTAIN**; whether it carries per-index data is device-dependent |
| **`89..94`** | **Optical row, slot D** | Round-robin retransmission of rows also seen in slots A, B and C | **PROBABLE** |
| `83..107` | Slot C/D record | Slots C and D are the head of one 25-byte record. The 108-byte payload carries all 25; the 96-byte payload carries the first 13, `pl[83..95]` | **CERTAIN** |
| `95` | 13th byte of that record | Does not track the row index: the SEL2870 holds `0x11` across all 109 frames while slots C and D cycle. Observed: `0x11` SELP1650 / SEL55210 / SEL2870, `0x02` SEL5518Z and both Yongnuo lenses, `0x00` Voigtländer, Loxia, both Viltrox adapters and the LM-EA9. On the SELP1650 and SEL5518Z it is `0x00` in the first status frame — the one whose row index is still `00 00` — and takes the lens's value from the second frame on | Position **CERTAIN**; why it is constant per Sony lens **UNKNOWN** |

---

## The optical rows are an index→row table transfer

On the SEL2870, across all 109 frames without exception:

```
pl[77] = 0x15  ->  slot C = E0 70 2B 1F 15 0D
pl[77] = 0x16  ->  slot C = F1 27 DC E0 D2 17
pl[77] = 0x17  ->  slot C = FF 59 2E 21 43 45
```

The SELP1650 gives one row for both its indices; the SEL55210 at 55 mm gives a distinct row for
each of `0x1314`, `0x1515`, `0x1616`, `0x1717`. So the four slots are a table streamed a few rows
per frame: **24 bytes of row payload plus 4 index bytes every frame**. Distinct non-zero rows seen:
7 on the SEL2870, 19 on the SEL5518Z.

### Indexing

Rows are indexed by the [aperture value](aperture_value.md), one record per grid point of that
scale, and each record is a **pair** — slot A holds the value common to the pair and slot B
alternates between the two halves frame by frame. That is what produces the observed "slot A changes
only when the index changes, slot B alternates every frame".

The table is **per-lens**: the YN35's records appear in neither 50 mm lens. Row A saturates — on the
YN35 it takes six distinct values over the first six grid points and is constant for the remaining
fourteen — while row B differs in every bucket.

The same record is carried by [message 0x28](msg_0x28.md) and [message 0x35](msg_0x35.md). A device
that populates one and not the others is inconsistent.

### What the rows carry

Encoding, per-slot meaning and all sample data: **[the 6-byte optical rows](optical_data.md)**.

Slot B's aperture behaviour is established: its type-1 row scales by exactly `1/√2` per stop over
five stops, and its first point is the on-axis value, `value[0] × F ≈ 0.100` on seven lenses from
15 mm to 210 mm and f/1.8 to f/5.6 across three manufacturers. Its remaining four points spread out
wide open and collapse one stop down.

Two lenses that cannot autofocus at all — the Voigtländer 15 mm and the Zeiss Loxia 21 mm —
populate the rows richly, so this is not autofocus-only data.

**A body consumes them.** Changing only the 19 payload bytes that carry a device's slot A/B record,
its slot C/D null row and `pl[95]` changed a Sony a9 II's autofocus behaviour measurably, where
every other payload field tried on that device produced none.

---

## `pl[80]` and modern-body compatibility

`pl[80]` is a 14-entry lookup on the row tag at `pl[78]`:

```
table = 06 06 0C 0C 12 12 0A 0A 14 14 1E 1E 32 32
pl[80] = (pl[78] - 7) <= 13 ? table[pl[78] - 7] : 0
```

| Row tag `pl[78]` | `0x07` | `0x08` | `0x09` | `0x0A` | `0x0B` | `0x0C` | `0x0D` | `0x0E` | `0x0F` | `0x10` | `0x11` | `0x12` | `0x13` | `0x14` | `0x15`–`0x17` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `pl[80]` | `06` | `06` | `0C` | `0C` | `12` | `12` | `0A` | `0A` | `14` | `14` | `1E` | `1E` | `32` | `32` | `00` |

It reproduces the wire: the SEL5518Z emits `00`, `0C` and `12` in this field and nothing else — `0C`
for the init tag `0x09`, `12` for the init tags `0x0B`/`0x0C`, and `00` for the main-loop tags
`0x15`–`0x17`, which fall outside the table.

A Yongnuo 50 mm DF sending `pl[80] = 0` fails on a Sony A7M5; the same lens sending the computed
value works. Older bodies — A6000, NEX-7 — complete init with devices that send `0` here.

The tags come in pairs sharing a value, and the values fall into two series, 6·(1,2,3) and
10·(1,2,3), then 50. The **mapping** is established; what the values mean is **UNKNOWN**.

---

## The aperture settle countdown, `pl[4]`

`pl[4]` tells the body how many more frames an aperture change will take. It lets the body know,
without waiting and watching `pl[0..1]`, when the aperture will be at its commanded value.

### Mechanism

1. **The body commands a new aperture.** If the lens is already there, nothing happens and `pl[4]`
   stays at its resting value.
2. **The lens works out how long the move will take.** It plans the iris movement from the
   current position to the target — accelerate, run, decelerate — and computes the total time in
   milliseconds.
3. **It converts that time to frames** and writes the result to `pl[4]`. One implementation uses
   `frames = (t_ms + 1) / 19 + 1` with integer division — the move time in units of about 19 ms,
   rounded up.
4. **It counts down.** Each 0x05 carries the current count, and the count drops by one after
   every frame, until it reaches the resting value.
5. **A new command during the move restarts the count** from the time the new move will take.

While the count runs, the aperture is moving: `pl[0..1]` reports the aperture as it changes, and
`pl[8]` bit 0 is clear. When the count reaches its resting value, `pl[0..1]` holds the settled
aperture and `pl[8]` bit 0 is set again.

### Worked example

An iris move planned at 50 ms gives `(50 + 1) / 19 + 1 = 3`:

| Frame | `pl[4]` | Aperture |
| --- | --- | --- |
| first 0x05 after the command | 3 | moving |
| next | 2 | moving |
| next | 1 | moving |
| next | resting value | settled |

### Observed

| Device | Sequence | Resting value |
| --- | --- | --- |
| Sony SEL5518Z | `03 02 01 00` over four consecutive frames while `pl[0..1]` climbs 4543 → 4758 → 4832 → 4908 | `00` |
| Yongnuo lenses | counts down to `01` | `01` |
| TECHART LM-EA9 | frozen | `00` |

| Aspect | Confidence |
| --- | --- |
| `pl[4]` counts the frames until the commanded aperture is reached | **CERTAIN** |
| The count is derived from the planned iris move time | **PROBABLE** |
| About 19 ms per counted frame | **POSSIBLE** |
| A new command restarts the count | **POSSIBLE** |
| The resting value is `00` or `01` depending on the lens, and either is acceptable to a body | **PROBABLE** |

## The aperture descriptor, `pl[44..59]`

A second aperture encoding, in units of ⅛ stop with `0x08` = f/1.0:

```
N = 2^((v − 8) / 16)
```

| `pl` | Meaning |
| --- | --- |
| `44` | **Maximum** aperture |
| `46` | A second aperture, at or wider than `pl[44]`. Purpose **UNKNOWN** — see below |
| `48` | `0xA0` constant. **UNKNOWN** |
| `51` | A second copy of `pl[44]` |
| `52` | **Minimum** aperture |
| `59` | `0x01` constant. **UNKNOWN** |

| Device | `pl[44]` | Maximum | `pl[46]` | `pl[52]` | Minimum |
| --- | --- | --- | --- | --- | --- |
| Viltrox EF adapter + Canon EF 50 mm f/1.8 | `0x16` | f/1.83 | `0x0E` | `0x50` | f/22.6 |
| Viltrox EF adapter + Canon EF-S 24 mm f/2.8 | `0x20` | f/2.83 | `0x18` | `0x50` | f/22.6 |
| TECHART LM-EA9 | `0x18` | f/2.0 | **`0x18`** | `0x70` | f/90 |

**`pl[46]` is not always `pl[44] − 8.`** It is one stop wider than the maximum on the two Viltrox
adapters and **equal to it** on the LM-EA9. So a device that copies the "one stop wider" relation
declares an aperture wider than its own stated maximum, in a field whose purpose is unknown, which
is worth avoiding until the field is understood. Setting `pl[46] = pl[44]` is the safe choice.

**All eight native lenses held send zero here.** They carry E-mount lens IDs (`0x8xxx`) and report
their aperture at `pl[0..1]`; the three devices that populate the descriptor are exactly those with
legacy lens IDs. A zero here marks a device class, not a missing field.

The body reads it: an LM-EA9 with these six bytes zeroed made a Sony a9 II display **F1.0**
(`v = 0` → f/0.71) and refuse to autofocus at all.

The conversion between this encoding and `pl[0..1]`'s is `value = 32 × (v − 8) + 4096`; see
[the aperture value](aperture_value.md#3-the-other-aperture-encoding-and-the-proof).

---

## Fields that split by autofocus capability

| Field | Devices that can drive focus | Manual-focus lenses |
| --- | --- | --- |
| `pl[9..16]` | 42/340 Sony and the adapters, 203/407 Yongnuo | 0/0 |
| `pl[95]` | `0x11` Sony zooms, `0x02` SEL5518Z and Yongnuo | `0x00` |

Both split by **autofocus capability** rather than by native versus adapter — two manual-focus
native lenses sit on the same side as the adapters in each.

`pl[44..59]` looks like the sharpest such discriminator (eight natives all-zero against three
adapters carrying the same structure) and is not one: it is the aperture descriptor above, and the
split is by lens-ID class.

### Optical slots, by device

| Device | Slot A | Slot B | Index `pl[77..78]` | `pl[80]` | Slots C/D |
| --- | --- | --- | --- | --- | --- |
| Sony native lenses | populated, changing | populated, changing | `15`/`16`/`17`, cycling | `00`/`0C`/`12` | populated |
| Yongnuo, current | populated | populated | cycling | computed from the tag | populated |
| Yongnuo 50 DF, older | populated | populated | cycling | **`00` always** | populated |
| TECHART LM-EA9, Viltrox EF adapters | `00 00 00 00 00 00` | `26 00 00 00 00 00` | `00 00`, **static** | `00` | zeros |

The adapters' slot B is a **valid row with null data**, tag `0x26` and five zero bytes, decoding to
a flat 1.500. See [the null rows adapters send](optical_data.md#7-the-null-rows-adapters-send).

---

## Focal length

`pl[24..25]` and `pl[26..27]`, both u16 LE, mm × 10.

| Lens | `pl[24..25]` | `pl[26..27]` | Actual |
| --- | --- | --- | --- |
| Sony SELP1650 @16 mm | 165 | 160 | 16 mm |
| Sony SEL55210 @55 mm | 552 | 550 | 55 mm |
| Sony SEL55210 @210 mm | 2032 | 2100 | 210 mm |
| Sony SEL2870 @70 mm | 680 | 700 | 70 mm |
| Sony SEL5518Z | 545 | 550 | 55 mm |
| Voigtländer 15 mm | 150 | 150 | 15 mm |
| Zeiss Loxia 21 mm | 210 | 210 | 21 mm |
| Yongnuo YN35 | 358 | 350 | 35 mm |
| Yongnuo 50F1.8S | 512 | 500 | 50 mm |
| Canon EF 50 mm f/1.8 + Viltrox | 500 | 500 | 50 mm |
| Canon EF-S 24 mm f/2.8 + Viltrox | 240 | 240 | 24 mm |
| TECHART LM-EA9 | 400 | 400 | fixed 40 mm |

Zooms report slightly different wide and tele values at a given position, so the pair is not a
simple duplicate — **POSSIBLE** that one is nominal and one actual.

The same focal length is carried by [message 0x28](msg_0x28.md) and [message 0x35](msg_0x35.md) in
the same encoding, and a device must keep all three consistent.

### The two focal lengths are consumed separately

A device reporting **40.0 mm** here and **50.0 mm** in [message 0x28](msg_0x28.md) produced
photographs tagged **50 mm** on a Sony a9 II. **EXIF takes the focal length from message 0x28**, not
from this one.

That leaves this field with a consumer of its own, and the shape of the two channels suggests which:
message 0x28 is requested once, at the shutter press, which is all a file needs; this field is sent
**60 times a second**, which is the rate a body needs to steer **in-body stabilisation**. A
stabiliser's correction scales with focal length, and a body offers a manual focal-length setting
for lenses that report none.

**POSSIBLE.** The division of labour fits — per-frame focal length to stabilise with, exposure-time
focal length to record — but nothing here has been traced to stabiliser behaviour. It is testable:
report two very different focal lengths in the two messages and see which one the stabiliser follows
while EXIF follows message 0x28.

---

## Open questions

- `pl[5]`, `pl[19]`, `pl[28..29]`, `pl[61]`: position known, meaning unestablished.
- `pl[6..7]`: a start-up value of unknown quantity.
- `pl[22]` bits 0–5: per-lens and static.
- `pl[48]` and `pl[59]` inside the aperture descriptor.
- What slot A measures. Aperture-independent, with an end-to-end ratio of ≈ 2 on every lens.
- What the `pl[80]` lookup values mean, as opposed to the mapping.
- Whether Sony populates `pl[81..82]`; every Sony lens measured sends 0.
- Why `pl[95]` is constant per Sony lens while the rows it sits with cycle.
