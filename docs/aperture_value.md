# The aperture value

The **aperture value** is an f-number expressed on a single logarithmic scale shared by every device
on the bus:

> **`value = 256 × AV + 4096`**, where `AV = 2·log₂(N)` is the APEX aperture value of the f-number `N`.

It is transmitted as a **u16 little-endian** integer. One whole stop is 256 counts, so the field
advances in thirds of 256 when a body steps the aperture in thirds — which is what "the protocol
quantises in thirds of 256" means.

---

# 1. Where it appears

| Message | Dir | Field | Role | Confidence |
| --- | --- | --- | --- | --- |
| **[0x05](msg_0x05.md)** | L→B | `pl[0..1]`, and an identical second copy at `pl[2..3]` | **The lens's current aperture**, once per 60 Hz frame | **CERTAIN** |
| **[0x05](msg_0x05.md)** | L→B | `pl[44..59]` — the **aperture descriptor** | **Maximum and minimum aperture**, in a *different* encoding. [Section 3](#3-the-other-aperture-encoding-and-the-proof) | **CERTAIN** |
| **[0x1B](msg_0x1B.md)** | B→L | `pl[0..1]` | **The body's aperture command.** The reply echoes the requested value back | **PROBABLE** |
| **[0x03](msg_0x03.md)** | B→L | `pl[5..6]` | A commanded aperture, every frame. At least one lens clamps it to its own range before use | **PROBABLE** |
| **[0x03](msg_0x03.md)** | B→L | `pl[3..4]` | A second value on the same grid | **UNKNOWN** |
| **[0x28](msg_0x28.md)** | L→B | `[0x0F..0x10]` | The same live value, beside the focal length and the optical rows | **PROBABLE** |
| **[0x35](msg_0x35.md)** | L→B | `[0x0F..0x10]` | A delta on this scale | **POSSIBLE** |

Message 0x05 duplicates the value into two adjacent fields. **They are not a "now / one frame ahead"
pair** — that split exists in [message 0x06](msg_0x06.md), for focus. One value, sent twice.

## There is no aperture *step* field

No message carries a step size. The resolution is implied twice over, and the two implications agree:

- the **descriptor** ([section 3](#3-the-other-aperture-encoding-and-the-proof)) is in units of ⅛ stop;
- the **command** grid is thirds of a stop — `pl[0]` takes the values `0x00`, `0x55`, `0xAA`
  (0, ⅓, ⅔ of 256), which is the ⅓-stop dial resolution Sony bodies default to.

A body that wants finer than ⅓ stop can express it: the low byte is a full 8 bits.

---

# 2. Reading the two bytes

The field is a **u16 little-endian**, so on the wire the low byte comes first. Taken separately:

## `pl[1]` — the high byte: whole stops

```
AV = pl[1] − 0x10            and conversely      pl[1] = AV + 0x10
```

`AV` is the APEX aperture value, and the f-number follows from it:

```
N = 2^(AV / 2)               and conversely      AV = 2 · log₂(N)
```

The `/2` is the whole of it: one stop multiplies the f-number by `√2`, and `2·log₂(√2) = 1`, so one
stop is `AV + 1` exactly. The `0x10` bias makes the field unsigned down to `AV = −16`.

| `pl[1]` | AV | N |
| --- | --- | --- |
| `0x10` | 0 | f/1.0 |
| `0x11` | 1 | f/1.4 |
| `0x12` | 2 | f/2.0 |
| `0x13` | 3 | f/2.8 |
| `0x14` | 4 | f/4 |
| `0x15` | 5 | f/5.6 |
| `0x16` | 6 | f/8 |
| `0x17` | 7 | f/11 |
| `0x18` | 8 | f/16 |
| `0x19` | 9 | f/22 |
| `0x1A` | 10 | f/32 |
| `0x1B` | 11 | f/45 |

## `pl[0]` — the low byte: the fraction of a stop

```
fraction of a stop = pl[0] / 256
```

so the complete value is

```
AV = (pl[1] − 0x10) + pl[0] / 256
```

| `pl[0]` | fraction | usual meaning |
| --- | --- | --- |
| `0x00` | 0 | on the whole stop |
| `0x55` | 0.332 | ⅓ stop |
| `0xAA` | 0.664 | ⅔ stop |

## Worked examples

| `pl[0]` | `pl[1]` | u16 | AV | N |
| --- | --- | --- | --- | --- |
| `00` | `10` | 4096 | 0.000 | f/1.00 |
| `00` | `11` | 4352 | 1.000 | f/1.41 |
| `C0` | `11` | 4544 | 1.750 | **f/1.83** |
| `00` | `12` | 4608 | 2.000 | **f/2.00** |
| `55` | `12` | 4693 | 2.332 | f/2.24 |
| `AB` | `12` | 4779 | 2.668 | f/2.52 |
| `00` | `13` | 4864 | 3.000 | **f/2.83** |
| `00` | `14` | 5120 | 4.000 | f/4.00 |
| `00` | `16` | 5632 | 6.000 | f/8.00 |
| `00` | `18` | 6144 | 8.000 | f/16.00 |
| `00` | `1B` | 6912 | 11.000 | f/45.25 |

Note the three bold rows. **f/1.83 is what a lens marked "f/1.8" reports** — 4544, the number this
repository spent months calling the infinity anchor. f/2.83 is what a lens marked "f/2.8" reports.
The nominal markings are rounded; the protocol carries the real value.

## Values measured on the wire

| Device | u16 | `pl[0]` | `pl[1]` | AV | N | Lens marking |
| --- | --- | --- | --- | --- | --- | --- |
| Viltrox + Canon EF 50/1.8 | 4544 | `C0` | `11` | 1.750 | f/1.83 | f/1.8 |
| Viltrox + Canon EF-S 24/2.8 | 4864 | `00` | `13` | 3.000 | f/2.83 | f/2.8 |
| TECHART LM-EA9 (stock, frozen) | 4864 | `00` | `13` | 3.000 | f/2.83 | — |
| Sony SELP1650 @16 mm | 5083 | `DB` | `13` | 3.855 | f/3.80 | f/3.5 |
| Sony SEL55210 @55 mm | 5245 | `7D` | `14` | 4.488 | f/4.74 | f/4.5 |
| Sony SEL55210 @210 mm | 5497 | `79` | `15` | 5.473 | f/6.66 | f/6.3 |
| a9 II → LM-EA9, message 0x1B | 4608 | `00` | `12` | 2.000 | f/2.00 | — |

All were taken with the iris wide open at power-on, so each reads that device's **maximum**
aperture. The three Sony zooms land a consistent 0.15–0.2 stop above their marked maxima, in the
direction and of the magnitude a real measured maximum aperture does, and the two zoom entries move
correctly with focal length.

---

# 3. The other aperture encoding, and the proof

[Message 0x05](msg_0x05.md) `pl[44..59]` carries a second aperture encoding — the **Canon EF
convention**, in units of **⅛ stop** with `0x08` = f/1.0:

```
N = 2^((v − 8) / 16)         so        AV = (v − 8) / 8
```

| Offset | Meaning |
| --- | --- |
| `pl[44]` | **Maximum** aperture |
| `pl[46]` | `pl[44] − 8` — one stop wider — on all three devices that populate the field. Purpose **UNKNOWN** |
| `pl[48]` | `0xA0` constant, **UNKNOWN** |
| `pl[51]` | A second copy of `pl[44]` |
| `pl[52]` | **Minimum** aperture |
| `pl[59]` | `01` constant, **UNKNOWN** |

## Converting between the two

Both encodings are already logarithmic in `N`, so the conversion is a scale and a bias — no
logarithm is involved:

```
value = 32 × (v − 8) + 4096
```

## And that conversion is the proof

Decoding `pl[0..1]` and `pl[44]` from the same capture, on the same device:

| Device | `pl[0..1]` | as f | `pl[44]` | as f |
| --- | --- | --- | --- | --- |
| **Viltrox + Canon EF 50/1.8** | **4544** | **f/1.834** | **`0x16`** | **f/1.834** |
| **Viltrox + Canon EF-S 24/2.8** | **4864** | **f/2.828** | **`0x20`** | **f/2.828** |
| TECHART LM-EA9 (stored) | 4864 | f/2.828 | `0x20` | f/2.828 |

**Two independent encodings of the same number, agreeing exactly, on three devices, with no
fitting.** Under the old focus reading there is no reason for the EF 50/1.8 to sit at exactly 4544
and the EF-S 24/2.8 at exactly 4864 while each decodes to its own maximum aperture.

## The second proof: the power-on aperture self-test

The same captures contain a handful of one-off values that never recur. They are the body
exercising the iris at power-on:

```
Viltrox + EF 50/1.8    5568   5824   6080   6336
                       spaced exactly 256 — exactly one stop
                       = wide open + 4, +5, +6, +7 stops
```

Four values exactly one stop apart is an aperture sequence and nothing else. On the SEL55210 at
210 mm the same moment produces a finer ramp — 5524, 5551, 5578, 5605, **5632**, 5659, … spaced 27
counts (≈ 0.105 stop), passing exactly through `AV 6.000` = f/8. Why that ramp is spaced 27 rather
than a power-of-two fraction is **UNKNOWN**.

## Who populates which

**All eight native lenses held send zero in `pl[44..59]`.** The three devices that populate it are
exactly those with legacy or A-mount lens IDs — the LM-EA9 (ID 234) and both Viltrox adapters
(ID 78).

The corrected reading explains why, where the old one could not: a native reports its aperture in
`pl[0..1]` on the native scale and has no use for the Canon-convention block, which is the **legacy
compatibility path**. `msg_0x05.md` previously recorded only that "the field belongs to a different
device class".

**Proof the body reads it:** an LM-EA9 with those six bytes zeroed made an a9 II display **F1.0**
(`v = 0` → f/0.71) and refuse to autofocus at all.

---

# 4. Aperture value → blade steps

A lens with a motorised iris runs the commanded value through a **per-lens calibration ladder** to
get a blade position. Recovered in full from one vendor's firmware; **CERTAIN for that
implementation, and not a protocol statement** — the ladder is the one genuinely per-lens datum on
this path and every lens will have its own.

```
index  = ((3·value + 128) >> 8) − 53          # NEAREST grid index, not a floor
steps  = interpolate(ladder[index], ladder[index ± 1])
```

Four properties worth knowing before commanding a lens or emulating one:

1. **A value ≤ 4544 returns 0 steps.** That is a hard **wide-open** clamp, and it is why 4544 is the
   anchor rather than the grid point just below it: 4544 is `AV 1.75` = **f/1.834**, the maximum
   aperture of the lens the ladder was read from. A lens with a different maximum will clamp
   somewhere else.
2. **The top of the ladder is the mechanical end.** `ladder[19] = 704` steps, reached at 6144 =
   `AV 8` = **f/16** — the minimum aperture of that same lens.
3. **Exact at grid points, approximate between them.** The interpolation extrapolates with the width
   of the *adjacent* segment on the side the direction argument points; where neighbouring segments
   have equal width that is exact, and on this lens five junctions do not.
4. **The first segment has a discontinuity.** It is scaled `1/64` and anchored on 4544, but the
   branch selecting that scale triggers on `value < 4608` while the rounding has already advanced the
   index at `value ≈ 4566`. Values in **4566…4607** resolve about **+53 steps too far**, snapping
   back at 4608. That is the wide-open end of the range.

Properties 3 and 4 are single-vendor observations. They are recorded because they bound how
precisely a body can expect a command to be honoured, and because an emulator that reproduces the
protocol but not these quirks will differ from that lens in exactly those two bands.

**The descending direction is a sawtooth.** Running the ladder with the opposite direction argument
produces a non-monotonic curve: the reported value can fall while the blades open. On a mechanism
with backlash that is a hysteresis artefact rather than a bug, but a body must not assume the value
is a pure function of blade position.

---

# 5. It also indexes the optical correction rows

The 6-byte optical rows carried by messages [0x05](msg_0x05.md), [0x28](msg_0x28.md) and
[0x35](msg_0x35.md) are fetched with this value as the **index**: `record = table[f(value), parity]`.
See **[the optical rows](optical_data.md)**.

That index being the aperture, rather than focus, resolves a regularity `optical_data.md` records as
unexplained. Slot B type 1 is `value[0] = C/F`, and its decoded `value[0]` is **linear in the table
index**, the mantissa falling by exactly 48 per bucket. If the index is blade position then
`1/F ∝ blade position`, i.e. **pupil diameter linear in blade travel** — which is what an iris
mechanically is.

> **Warning.** `optical_data.md` has not yet been re-derived against this correction. Every table in
> it labelled "focus bucket" is an aperture bucket, and its section 6.4 refutation — *"13–20× too
> fast for `F_eff = F·(1+m)`"* — was measuring the wrong axis.

---

# 6. Manufacturer notes

- **Yongnuo** (YN35mm f/1.8S DA, YN50mm f/1.8S DF) — implements the full path. Message 0x1B drives
  the iris through the ladder above; message 0x05 `pl[0..1]` reports the blade counter through the
  forward map. The iris and the focus group are **two independent motion controllers** with separate
  command queues, tasks and position accessors; focus is commanded through
  [message 0x04](msg_0x04.md) and reported in [message 0x06](msg_0x06.md).
- **TECHART LM-EA9** — a Leica M adapter with **no iris at all**. It writes `pl[0..1]` once at boot
  and never again, reporting a constant f/2.83 — "wide open, permanently", which is the only honest
  thing it can say. It repurposes the message 0x1B aperture command as a **user interface**: values
  in `pl[1]` = `0x13`…`0x1B` (f/2.8 … f/45, 27 positions at ⅓ stop) select the focal length the
  adapter reports, and its declared maximum of f/2.0 sits one step *below* that window so that wide
  open means "no command". Its declared minimum of f/90 — absurd for any real lens — exists to give
  the camera's aperture dial enough travel to address the whole ladder.
- **Viltrox EF adapters** — populate the Canon-convention descriptor from the mounted EF lens and
  report the matching value in `pl[0..1]`.

---

# 7. Confidence

| Claim | Evidence | Status |
| --- | --- | --- |
| `pl[0..1]` is `256 × AV + 4096` | Exact agreement with `pl[44]` on three devices; one-stop sweeps in the same captures | **CERTAIN** |
| The two-byte split (whole stops / fraction) | Follows arithmetically from the u16 encoding; `0x00`/`0x55`/`0xAA` observed | **CERTAIN** |
| `pl[44]` max / `pl[52]` min | Decoded values match the mounted lens on two Viltrox captures; zeroing them changes what the body displays | **CERTAIN** |
| Message 0x1B `pl[0..1]` is an aperture command | Arithmetic fits two LM-EA9 hardware observations exactly; observed on an a9 II carrying f/2.0, the adapter's own declared maximum, after a shutter press | **PROBABLE** |
| Message 0x03 `pl[5..6]` is a commanded aperture | One vendor clamps it to that vendor's own aperture range before use | **PROBABLE** |
| The ladder is per-lens aperture calibration | Recovered from one vendor's firmware, both directions | **CERTAIN for that vendor**, UNKNOWN as a protocol statement |
| Message 0x03 `pl[3..4]` | Position known, meaning not | **UNKNOWN** |

---

# 8. Open questions

- **What `pl[3..4]` of message 0x03 is.** It sits on this grid and is latched by at least one lens.
- **Why the SEL55210's power-on ramp is spaced 27 counts** rather than a power-of-two fraction of a
  stop.
- **What `pl[46]` of the descriptor is for.** It is `pl[44] − 8`, one stop wider than the maximum, on
  every device that populates the field — and the LM-EA9's own boot patch breaks the relation without
  apparent consequence.
- **Whether a lens ever reports a value it was not commanded**, i.e. whether `pl[0..1]` is a
  measured blade position or an echo of the last command. Yongnuo reports the counter, which is a
  measurement; no second implementation has been read.
- **How a native E-mount lens is commanded.** The LM-EA9 and the Viltrox adapters receive message
  0x1B; no native has been observed receiving anything. Since natives leave the Canon-convention
  descriptor empty, the command may travel by a different route entirely — a
  [message 0x03](msg_0x03.md)/[0x04](msg_0x04.md) record is the obvious candidate.
