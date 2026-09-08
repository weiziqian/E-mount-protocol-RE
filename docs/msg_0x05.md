# Message 0x05 — lens status and optical table transfer

**Summary.** The single most important message. It carries live focus position, focal length, and
the per-frame optical-correction rows that every adapter measured sends as zeros. Frame 1 of the
four-frame 60 Hz loop.

**Direction:** L→B only.

**Class:** normal (`0x01`).

**Payload:** 96 bytes (105-byte frame) or 108 bytes (117-byte frame). A lens uses one variant or
the other and never both.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Two payload sizes

A lens uses either the **96-byte** payload (105-byte frame) or the **108-byte** one (117-byte
frame), and *never both*.

| Variant | Devices |
| --- | --- |
| 96 B payload / 105 B frame | SELP1650, SEL55210, SEL2870, SEL5518Z, Voigtländer, Loxia, both Viltrox adapters, TECHART LM-EA9, Yongnuo 35F1.8 DA |
| 108 B payload / 117 B frame | Techart EOS-NEX III in Fn mode, Yongnuo 50F1.8S DF |

Tempting hypothesis: 117 = full-frame, 105 = APS-C. One lens sits on each side of that split, so it
is a hypothesis, not a result.

The two variants share a layout. **`pl[0..82]` is identical between them**; the long form's extra
12 bytes are *appended*. Slots C and D are the head of one **25-byte** record: the long form
carries all 25 (`pl[83..107]`), the short form carries only the first 13 (`pl[83..95]`). This is
the same shape as slot A/B, whose record is 14 bytes of which 13 are sent (`pl[32..44]`).
Everything above that — the row index `pl[77..80]`, and `pl[81..82]` — sits at the same offset in
both, confirmed by the Yongnuo 50F1.8S DF's 108-byte payload matching the 96-byte field map field
for field up to `pl[82]`. Only the **extent** of the slot C/D region differs, so field offsets do
carry across variants. CERTAIN.

## Reference payloads

```
TECHART     00 13 00 13 00 00 10 00 07 2a 00 2a 00 54 01 54 01 00 00 00 00 07 80 ff 90 01 90 01 0f 01 00 00
LM-EA9      | 00 00 00 00 00 00 | 26 00 00 00 00 00 | 00 00 ... (all zero to the end)

Sony        db 13 db 13 00 1e 00 00 07 2a 00 2a 00 54 01 54 01 00 00 00 00 07 98 ff a5 00 a0 00 00 01 00 00
SELP1650    | a0 ea 70 49 1e 0f | e1 b9 12 a6 e1 f6 | ... 15 16 ... de f2 b6 b9 c2 d3 | d0 f3 5a 54 33 13
```

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..1]` | **u16 LE [live focus position](live_focus_position.md), on the NORMALISED distance scale** — *not* the lens's own encoder counts. See the scale section below. | **CERTAIN** |
| `pl[2..3]` | Second copy of the same value. Yongnuo writes the two fields **identically**; unlike message 0x06, there is no forecast term here. | **CERTAIN** |
| `pl[4]` | **A "frames until settled" countdown.** Yongnuo loads a counter here and decrements it while it is > 1. Seen on the wire: the SEL5518Z sends `03 02 01 00` over four consecutive frames while its reported position climbs 4543 → 4758 → 4832 → 4908, then holds 0. TECHART LM-EA9: always 0. | **CERTAIN** |
| `pl[5]` | A per-lens value that jumps for a frame or two around motion events — `0x1e`/`0x5e` SELP1650, `0x2c` SEL5518Z, `0x28`/`0xca` SEL2870, `0x15`/`0x1a` SEL55210. Yongnuo zeroes it every frame; both manual lenses and the LM-EA9 send 0. | UNKNOWN |
| `pl[6..7]` | u16, **non-zero only in the first status frame after init**, then zero for the rest of the session: 19 (SELP1650), 20 (SEL55210), 17 (SEL5518Z and Viltrox + Canon EF 50). Yongnuo zeroes it every frame. **The LM-EA9 sends 16 forever** — and 16 is a value no other device sends at all. So this is a start-up field that adapter has frozen into a constant, the same failure mode as `pl[23]` and message 0x06's subject distance. | Behaviour **CERTAIN**; meaning UNKNOWN |
| `pl[8]` | **A state byte, not a constant.** Yongnuo writes `3` when the motion controller is idle and `2` while it is driving. Sony lenses alternate `06`/`07` at the same offset, `06` appearing in the frames around motion. LM-EA9: frozen `07`. | Existence **CERTAIN**; the Sony 6/7 mapping PROBABLE |
| `pl[9..10]`, `pl[11..12]` | Duplicated u16 pair. **42/42** on every Sony lens measured, both Viltrox adapters and the LM-EA9; **203/203** on both Yongnuo lenses; **0/0 on the Voigtländer 15/4.5 and the Zeiss Loxia 21/2.8**, across all 1267 manual-focus frames held. The split is not by vendor but by **autofocus capability**: every device that can drive focus sends a non-zero pair, both manual-focus lenses send zero. So it is not optical data, but it is not a bare vendor constant either. | Values **CERTAIN**; the AF correlation PROBABLE (2 manual lenses vs 7 AF-capable devices, two bodies) |
| `pl[13..14]`, `pl[15..16]` | Same story: **340/340** on Sony and the adapters, **407/407** on Yongnuo, **0/0** on both manual-focus lenses. | as above |
| `pl[17..18]` | u16 LE on the **`n × 256/3` focus grid**, floored at `0x11C0` (4544, the infinity anchor), default `0x1400` (5120). Yongnuo reads it out of the same 22-entry grid table that this message's position uses, indexed by a counter clamped to ≤ 22. Every Sony lens, both Viltrox adapters and the LM-EA9 send 0. | Encoding **CERTAIN**; what the index selects UNKNOWN |
| `pl[19]` | A boolean, written by the same state machine that fills `pl[17..18]`. Zero on every device observed. | Existence **CERTAIN**; meaning UNKNOWN |
| `pl[20..21]` | **Subject distance.** Only the Voigtländer 15/4.5 populates it: turning its focus ring from 0.3 m to infinity walks the u16 through 272, 299, 320, 351, 384, 448 and then `0x0700`. Every Sony lens, both Viltrox adapters and the LM-EA9 send a constant `0x0700`, so `pl[21] = 0x07` with `pl[20] = 0` reads as a **"no distance data" / infinity state**, not a protocol constant. Yongnuo computes it from three live quantities. | Focus dependence **CERTAIN**; units PROBABLE (mm) |
| `pl[22]` | **A flags byte.** Both Yongnuo implementations write `0x80`, then OR in `0x40` when the motion controller reports it is driving — so **bit 6 = focus moving**. The same branch writes the travel direction into `pl[60]`, which ties the two together. Bits 0–5 are per-lens and static within a session: `0x38` SELP1650, `0x3C` SEL55210, `0x04` SEL2870, `0x08` SEL5518Z, `0x00` Loxia and Voigtländer. LM-EA9: frozen `0x80`. | bits 7 and 6 **CERTAIN** (two independent Yongnuo implementations); bits 0–5 UNKNOWN |
| `pl[23]` | **Focus-dependent, not a constant.** Over the Voigtländer's 0.3 m → ∞ sweep it takes **55 distinct values**, climbing monotonically `0x93 → 0xFF`. Read as a signed byte it is ≈ **−32 × dioptres**: `0x93` = −109, and 109/32 = 3.41 dpt = 0.293 m, that lens's minimum focus distance; `0xFF` = −1 ≈ infinity. Sony lenses send a per-session constant (`0xFF` on SELP1650 / SEL55210 / SEL5518Z, `0xD9` = −39 = 1.22 dpt ≈ 0.8 m on the SEL2870). **The LM-EA9 sends `0xFF` — "subject at infinity" — forever**, the same class of defect as its frozen message 0x06 subject distance. | Focus dependence **CERTAIN** (536 frames, one lens); the −1/32 dpt scale PROBABLE |
| `pl[24..25]`, `pl[26..27]` | **Duplicated u16 LE pair = focal length in mm × 10.** Wide/tele on a zoom, equal-ish on a prime. See the table below. | **CERTAIN** |
| `pl[28..29]` | u16. LM-EA9 271; **Yongnuo 310 on *both* lenses, and neither computes it** — it is copied out of a fixed block, and nothing in either build path writes this offset. SEL5518Z 320, SEL2870 312, Voigtländer 272, SELP1650 256, SEL55210 384, **Loxia 0, both Viltrox 0**. Two Yongnuo lenses of different focal length, format and codebase sharing one value rules out a per-lens optical quantity for that vendor; three devices sending 0 while working normally show the field is **optional**. | UNKNOWN. Ruled out: exit-pupil distance, maximum aperture, and "required" |
| `pl[30..31]` | **An affine function of `pl[0..1]`, not an independent quantity.** Yongnuo writes either `grid_value − 0x11C0` (position measured from the infinity anchor) or `0x1800 − pl[0..1]` (6144 minus the live position), depending on which branch of the focus state machine is active. The Voigtländer obeys the same relationship on the wire with the opposite sign — `pl[0..1] − pl[30..31] = 5205` exactly, across all 41 sampled values of a 461-frame observation. LM-EA9, both Viltrox and every Sony lens except the SEL2870 (345) and SEL55210 @210 (246): 0. | **CERTAIN** |
| **`pl[32..37]`** | **[Optical row, slot A](optical_data.md#4-slot-a--the-field-sampling-grid).** 6 bytes = 3-bit exponent + 12-bit start value + 4 signed deltas, decoding to a 5-point curve ([encoding](optical_data.md#3-how-to-decode)). Aperture-independent, focus-dependent; changes only when focus moves. Preferred reading: the **field sampling grid** — the five positions, as tangent angles from the exit pupil, at which the other rows are sampled. | Encoding **CERTAIN**; grid reading POSSIBLE |
| **`pl[38..43]`** | **[Optical row, slot B](optical_data.md#1-what-the-rows-carry).** Same encoding; alternates every frame between a type-1 and a type-0 row. Its aperture behaviour is established: type 1 scales as **`1/F`**. Its focus behaviour is **not** explained — see below. | Encoding **CERTAIN**; `∝ 1/F` **CERTAIN**; focus axis UNKNOWN |
| **`pl[44..59]`** | **The aperture descriptor.** Canon EF convention: 1/8-stop units with `0x08` = f/1.0, i.e. **`F = 2^((v−8)/16)`**. See the aperture section below. | Encoding **CERTAIN**; `pl[48]`/`pl[59]` UNKNOWN |
| `pl[54..55]` | Yongnuo writes `pl[54]` = the **high byte of (target − current position) when that difference is positive**, else 0, and `pl[55]` = 0 — a coarse "distance still to run". Non-zero on the SEL5518Z (`01`) during its power-on focus sweep, 0 once settled. LM-EA9: always 0. | **CERTAIN** |
| `pl[60]` | **Focus travel direction**, a signed byte: `0x00` stationary, `0x01` and `0xFF` the two directions. Yongnuo writes it from the motion controller's sign test, and the same branch sets `pl[22]` bit 6, so direction and the "moving" flag are written together. Seen on the wire only on the Voigtländer (`FF` throughout its focus sweep, `00` once settled, `01` on a brief reverse) — the other observations barely move focus. LM-EA9: always 0. | **CERTAIN** |
| `pl[61]` | `01` on most devices | UNKNOWN |
| **`pl[62]`** | **A drive/status byte**, `1` or `3` on Yongnuo, gated by a busy check, a mode flag and a boolean. Position and general shape (written every frame, near where a lens would report drive state) fit "actively focusing vs. settled", but that reading is inference. The LM-EA9 sends a frozen `1`. | Existence **CERTAIN**; meaning PROBABLE |
| **`pl[77..78]`** | **Row index, a duplicated byte pair.** Cycles `0x15, 0x16, 0x17` in the main loop; `0x09, 0x0B, 0x0C` during init. Certain values are a **"nothing new this frame" marker** rather than a growing position: on Yongnuo, index `0` (and, via a second-level override, `7`/`9`) forces slots C and D to null content with distinctive tag bytes instead of a real fetch. Whether the marker VALUES are shared across vendors is UNKNOWN — Yongnuo's set (`0`/`7`/`9`/`0x15`) does not match Sony's observed cycle byte for byte, though the shape (a small "real data vs. repeat" state machine) is the same. | Index concept **CERTAIN**; null-marker convention PROBABLE; exact vendor values NOT portable |
| `pl[79]` | Second index pair, first byte — `00` on every device measured | UNKNOWN |
| **`pl[80]`** | **Second index pair, second byte = `f(row tag)` via a 14-entry lookup.** Derived from `pl[78]`, not independent. Observed `00`, `0c`, `12` on SEL5518Z; `00` on SEL2870. **This field gates modern-body compatibility** — see below. | **CERTAIN** |
| **`pl[81..82]`** | **Effective focal length × 10, corrected for focus.** Both Yongnuo implementations compute `base + slope × f(focus)`, and **the base constant is the lens's own focal length**: `358.0` on the YN35 (whose `pl[24..25]` is 358) and `0x202` = 514 on the YN50 DF (`pl[24..25]` = 512); slopes `−0.0869` and `−1.04`, applied to a clamped target-minus-current term. The Voigtländer confirms it on the wire: nominal 15 mm (`pl[24..25]` = 150), and `pl[81..82]` walks 154 → 163 as focus moves far → near — focus breathing, in the same mm × 10 units as `pl[24..25]`. Every Sony lens measured, both Viltrox adapters and the LM-EA9 send 0. | **CERTAIN** for Yongnuo (two independent implementations agree, each base equalling its own focal length); PROBABLE for the Voigtländer; that Sony uses the same field UNKNOWN |
| **`pl[83..88]`** | **[Optical row, slot C](optical_data.md#8-slots-c-and-d).** Same 6-byte encoding, different quantity. "A strict function of `pl[77]`" is a Sony (SEL2870) observation, **not a protocol universal** — on Yongnuo the same selector picks between two fixed 25-byte records that are byte-identical (both zero) on the one lens checked, and it is the same record message 0x35 calls its "secondary bytes". | Lookup mechanism **CERTAIN**; whether it carries real per-index data is vendor-dependent |
| **`pl[89..94]`** | **[Optical row, slot D](optical_data.md#8-slots-c-and-d)** — round-robin retransmission of rows also seen in A/B/C on Sony; on Yongnuo, part of the same empty 25-byte record as slot C. | PROBABLE (Sony); empty on the one Yongnuo lens checked |
| **`pl[83..107]`** | Slots C and D are the head of **one 25-byte record**, fetched in a single call. The 108-byte payload carries all 25 bytes at `pl[83..107]`; the 96-byte payload carries only the first 13, `pl[83..95]`. See [Two payload sizes](#two-payload-sizes). | **CERTAIN** |
| **`pl[95]`** | Structurally the **13th byte of the slot C/D record**, written as part of the record copy. On Sony it does **not** track the row index: the SEL2870 holds `0x11` across all 109 frames while slots C and D cycle through three rows each. Observed: `0x11` on SELP1650 / SEL55210 / SEL2870, `0x02` on SEL5518Z and both Yongnuo lenses, **`0x00` on the Voigtländer, the Loxia, both Viltrox adapters and the LM-EA9**. On the SELP1650 and SEL5518Z it is `0x00` in the very first status frame — the one where the row index is still `00 00` — and takes the lens's value from the second frame onward. So it correlates with autofocus capability *and* sits where record data would sit; those two readings have not been separated. | Position in the record **CERTAIN**; why it is constant per Sony lens UNKNOWN |

---

## The focus-position scale is not encoder counts

Summarised here because this message is where the value is read most often. The scale, the
transform in both directions, the plots, and the two implementation defects around them have
their own document: **[the live focus position](live_focus_position.md)**.

This is why message 0x05's position and message 0x06's position never agree on any device.

**One physical variable, reported in two different spaces.** A lens tracks its focus mechanism with
a live **motor step counter**. Both status messages report that one counter, transformed
differently:

| Message | Field | Value reported |
| --- | --- | --- |
| **0x06** | `pl[2..3]`, `pl[20..21]` | **`16080 + steps`** — raw counts plus a fixed base |
| **0x05** | `pl[0..1]`, `pl[2..3]` | **ladder-transformed** onto a shared grid, see below |

The `16080` base is a Yongnuo vendor constant — identical in their 35 mm DA and 50 mm DA, so not
per-lens calibration. It is also what puts Yongnuo's 0x06 position in the 16080–16880 band, the same
band as the SELP1650's observed 16383–17230 and its own 0x06 travel limits 16116–17092.

### The transform

A lens maps its own motor steps onto the shared grid through a per-lens **ladder** of step counts:

```
if steps == 0:  return 4544                        # 0x11C0 — the infinity anchor
find k with P[k-1] <= steps < P[k]                 # per-lens ladder, 19–22 entries, P[0] = 0
base = (k + 52) * 256/3                            # integer-truncated
frac = (steps - P[k-1]) / (P[k-1+dir] - P[k-1])    # interpolate toward the travel direction
return base + frac * 85.33333333333333
```

Special cases: the first bucket and negative inputs use `frac × 64 + 4544`; negative steps return
`0x1155` = 4437; `dir == 0` (stationary) gives zero interpolation width, so the value snaps to the
bucket base.

The map is continuous piecewise-linear and collapses to one statement:

> **ladder point `k` ↦ `(k + 52) × 256/3`.**

So `pl[0..1]` is `n × 256/3` for an integer distance index `n` — an index transmitted at ⅓-count
resolution. That is what "the protocol quantises in thirds of 256" means.

### Landmarks on the grid

| `n` | Value | Significance |
| --- | --- | --- |
| 53.25 | **4544** | The infinity anchor. Both Yongnuo 0x05 defaults; also the floor of the Viltrox + Canon EF 50 observation |
| 57 | **4864** | Exactly the TECHART LM-EA9's fixed 0x05 value |
| 66 | **5632** | Exactly the LM-EA9's upper *travel clamp* (message 0x06) |
| 51…72 | 4352…6144 | The 22-entry u16 table every Yongnuo lens carries — **the protocol grid itself** |

That last row is not per-lens data: the table is byte-identical across the Yongnuo 35 DA, 50 DA and
50 DF. It is just `n × 256/3` tabulated.

Cross-check against observed ranges: every device measured falls in `n ≈ 53–81` (LM-EA9 4864;
SELP1650 5053–5083; SEL55210 5245–6911; Viltrox + Canon EF 50 4544–6336) — a narrow band across a
16–50 power zoom, a 55–210 tele and an EF adapter. That is what a shared scale looks like, and what
device-specific encoder counts do not.

### The ladder is the only per-lens datum in this path

Same transform, different table:

| Lens | Motor steps at each grid point | Entries |
| --- | --- | --- |
| Yongnuo YN35 DA (35 mm APS-C) | 64 96 128 176 224 272 304 336 368 400 432 464 496 528 560 592 624 672 704 | 19 |
| Yongnuo YN50 DA (50 mm APS-C) | 112 160 224 272 304 336 400 432 480 512 560 592 624 656 688 704 736 768 784 | 19 |
| Yongnuo YN50 DF (50 mm FF) | 64 112 160 192 224 272 304 352 384 432 464 512 560 608 640 656 688 704 736 752 768 808 | 22 |

Feeding 400 steps through the first two ladders gives 5376 and 5120 respectively. Everything else
in the path — the `× 256/3` grid, the 4544 anchor, the 16080 offset for message 0x06 — is shared
across the lineup.

| Lens | Ladder entries | `pl[0..1]` at steps 0 | At full travel | Grid span reached |
| --- | --- | --- | --- | --- |
| Yongnuo YN35 DA | 19 | 4544 | 6143 | `n` 53.25 → 72 |
| Yongnuo YN50 DA | 19 | 4544 | 6143 | `n` 53.25 → 72 |
| Yongnuo YN50 DF | 22 | 4544 | 6399 | `n` 53.25 → 75 |

Two protocol-level facts fall out of that:

1. **The scale is absolute, not per-lens normalised.** All three lenses report the identical 4544 at
   their infinity end, but they do *not* stop at the same value — the full-frame 50 DF runs three
   grid points further. A lens occupies whatever portion of the shared grid its travel needs.
2. **The grid extends past `n = 72`.** The 50 DF carries an upward continuation
   `5973 6058 6144 6229 6314 6400` (= `n` 70…75). So `n = 51…72` is the *observed* span, not the
   scale's limit; `4352…6400` is the range attested so far. (`6400 = 75 × 256/3` exactly, a useful
   check that the scale reading is right.)

Note the top-of-travel value is 6143, not 6144: `base` truncates to 6058 from 6058.67 before the
85.33 interpolation term is added. The exact 6144 exists in the tabulated grid and is what the
*inverse* accepts; the forward path is one count short of it. Cosmetic, but do not treat a
6143/6144 mismatch as a decode error.

### Direction dependence

`pl[0..1]` is a function of `(steps, direction)`, not of `steps` alone — the `P[k-1+dir]` index
above selects a different interpolation denominator per direction. Ascending is monotonic and
smooth; the descending branch is not. **A body cannot assume this field is a pure function of
mechanical position.**

### Two report modes

Yongnuo branches on the direction code of the most recent motion record:

| State | `pl[0..1]` reports | `pl[8]` | `pl[4]` |
| --- | --- | --- | --- |
| Moving | The live ladder value | `2` | A countdown of body-frames to go |
| Settled | The *commanded* target as queued | `3` | `1` |

So a moving lens reports measured position plus frames-remaining; a settled one reports the exact
commanded value.

**Caveat, and it is a real one.** This conflicts with the wire observation that `pl[8]` is `07` on
Sony devices, and the SELP1650 does read `07`. So either these two bytes mean different things to
Sony and Yongnuo, or Yongnuo is driving a field that Sony lenses hold constant for another reason.
**No Yongnuo bus observation exists**, so the two readings cannot be reconciled against the wire.
Treat the mode table as PROBABLE and specific to Yongnuo; do not overwrite the `pl[8] = 07`
observation with it.

### The consequence for any adapter

Reporting a correct live value in this field requires the device's own steps→`n` ladder. An adapter
with no electrical contact with the mounted lens cannot have one without shipping a hardcoded
per-lens profile — and copying a raw encoder position across would put counts into a field that is
not in counts.

---

## The rows are an index→row table transfer

On the SEL2870, across all 109 frames without exception:

```
pl[77] = 0x15  ->  slot C = e0 70 2b 1f 15 0d
pl[77] = 0x16  ->  slot C = f1 27 dc e0 d2 17
pl[77] = 0x17  ->  slot C = ff 59 2e 21 43 45
```

Reproduced on the A6000: the SELP1650 gives one row for both its indices; the SEL55210 @55 gives a
distinct row for each of `0x1314`, `0x1515`, `0x1616`, `0x1717`. That settles the "is this table
data?" question — **it is an index→row table transfer**, streamed a few rows per frame. A native
lens sends **24 bytes of table payload plus 4 index bytes every frame**. Distinct non-zero rows
seen: 7 on the SEL2870, 19 on the SEL5518Z.

### The rows are indexed by focus position

Yongnuo fills `pl[32..43]` from a record selected by

```
k = (position * 3)/256 - 53          # exactly the grid index n of the n × 256/3 scale
record 2k on even frames, 2k+1 on odd
```

**So there is one correction record per focus grid point**, and the pair-plus-parity is what
produces the observed "slot A changes only when focus moves, slot B alternates every frame".

Three consequences:

1. The table is **per-lens**. The YN35's records appear in neither 50 mm lens.
2. Row A saturates: on the YN35 it takes six distinct values over grid points `n` = 53…58 (the
   third of the travel nearest infinity) and is then constant for the remaining fourteen. Row B
   differs in every bucket.
3. **The same 12-byte record is also sent in message 0x35**, via the same selector. Any device that
   populates one and not the other is inconsistent.

### What the rows carry

Full encoding, per-slot meaning and all sample data: **[the 6-byte optical rows](optical_data.md)**. In
summary — [slot A](optical_data.md#4-slot-a--the-field-sampling-grid) is the field sampling grid,
[slot B type 0](optical_data.md#5-slot-b-type-0--the-pupil-magnification-p) the pupil magnification, and
[slot B type 1](optical_data.md#6-slot-b-type-1--the-pupil-size--f-number-row) the pupil size.

Slot B's aperture behaviour is established:

- Its type-1 row scales by exactly **`1/√2` per stop of aperture** over five stops, i.e. it is
  proportional to `1/F`.
- Its first point is the **on-axis** value: `value[0] × F ≈ 0.100` on seven lenses from 15 mm to
  210 mm, f/1.8 to f/5.6, three manufacturers. Only an on-axis sample can agree across such lenses.
- Its remaining four points spread out wide open and collapse one stop down — mechanical
  vignetting disappearing.

Its **focus** behaviour is not explained. Slot B falls 13–20× faster with focus than a working
f-number can, on two lenses and two implementations, while its across-the-field profile collapses to
perfectly flat — and a quantity sampled across field height does not become uniform when you focus
closer. So "slot B is the effective pupil description" holds on the aperture axis and fails on the
focus axis. The older "is this PDAF data or image-correction data?" question is **open**.

Slot A is aperture-**in**dependent, focus-dependent, and has an end-to-end ratio pinned at ≈ 2 on
every lens; its quantity is UNKNOWN.

Two lenses that **cannot autofocus at all** (Voigtländer 15/4.5, Zeiss Loxia 21/2.8) populate the
block richly, so whatever it is, it is not AF-only data.

---

## `pl[80]` gates modern-body compatibility

`pl[80]` is a 14-entry lookup on the row tag `pl[78]`:

```c
/* tbl = 06 06 0c 0c 12 12 0a 0a 14 14 1e 1e 32 32 */
k = pl[78];                                        /* the row tag; pl[77..78] is duplicated */
pl[80] = (uint8_t)(k - 7) <= 13 ? tbl[k - 7] : 0;
```

| Row tag `pl[78]` | 0x07 | 0x08 | 0x09 | 0x0A | 0x0B | 0x0C | 0x0D | 0x0E | 0x0F | 0x10 | 0x11 | 0x12 | 0x13 | 0x14 | 0x15–0x17 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `pl[80]` | 06 | 06 | 0c | 0c | 12 | 12 | 0a | 0a | 14 | 14 | 1e | 1e | 32 | 32 | **00** |

**It reproduces the wire.** The Sony SEL5518Z was observed emitting `00`, `0c`, `12` in this field
and nothing else. The formula yields `0c` for tag `0x09` and `12` for tags `0x0B`/`0x0C` — the init
tags — and `00` for the main-loop tags `0x15`–`0x17`, which fall outside the valid range. An
independent vendor computes exactly the values Sony's lens sends.

**It matters on modern bodies.** Yongnuo 50 mm DF versions that send `pl[80] = 0` fail on the Sony
A7M5; the version that started sending the computed value works. That release changed other things
besides this field, so the causal link is a **strong lead, not a demonstrated cause** — but the
correlation is clean: every other protocol field of that lens was already correct, and `pl[80]` was
the only one that differed.

Older bodies (A6000, NEX-7) evidently tolerate `pl[80] = 0` — the adapters get through init on them.

Note the tags come in pairs sharing a value, and the values fall into two series — 6·(1,2,3) and
10·(1,2,3), then 50. That looks like a per-row count or step size rather than correction data, but
the *meaning* of the values is UNKNOWN. Only the mapping is established.

---

## The aperture descriptor, `pl[44..59]`

Canon EF convention: 1/8-stop units with `0x08` = f/1.0, i.e. **`F = 2^((v−8)/16)`**.

| Offset | Meaning |
| --- | --- |
| `pl[44]` | **Maximum** aperture |
| `pl[46]` | `pl[44] − 8` — one stop wider — on all three devices that use the field |
| `pl[48]` | `0xA0` constant, UNKNOWN |
| `pl[51]` | A second copy of `pl[44]` |
| `pl[52]` | **Minimum** aperture |
| `pl[59]` | `01` constant, UNKNOWN |

It reads the marked values exactly:

| Device | `pl[44]` | Decoded max | `pl[52]` | Decoded min |
| --- | --- | --- | --- | --- |
| Viltrox + Canon EF 50/1.8 | `0x16` | f/1.83 | `0x50` | f/22.6 |
| Viltrox + Canon EF-S 24/2.8 | `0x20` | f/2.83 | `0x50` | f/22.6 |
| TECHART LM-EA9 | `0x20` → `0x18` | f/2.83 → f/2.0 | `0x50` → `0x70` | f/22.6 → f/90 |

The LM-EA9's `0x20`/`0x50` pair matches the Canon EF 40 mm f/2.8 whose identity it presents, and its
boot routine then widens the pair to `0x18`/`0x70` = f/2.0 … f/90 — and f/2.0 is what a Sony a9 II
displays with an LM-EA9 mounted.

**All eight native lenses held send zero here.** They carry E-mount IDs (`0x8xxx`) and report
aperture elsewhere; the three devices that populate the field are exactly those with legacy or
A-mount IDs (LM-EA9 234, Viltrox 78). So a zero here is not an adapter tell — the field belongs to a
different device class.

**Proof the body reads it:** an LM-EA9 with these six bytes zeroed made an a9 II display **F1.0**
(`v = 0` → f/0.71) and refuse to autofocus at all.

---

## Native/adapter discriminators inside this message

| Field | Natives | Adapters |
| --- | --- | --- |
| `pl[95]` | Non-zero on every AF-capable device (`0x11` Sony zooms, `0x02` SEL5518Z and Yongnuo); `0x00` on both manual lenses | `0x00` |
| `pl[9..16]` | Non-zero on every AF-capable device (42/340 Sony, 203/407 Yongnuo); zero on both manual lenses | 42/340 — the adapters match Sony here |

Both rows split by **autofocus capability** rather than by native/adapter — two manual-focus native
lenses sit on the adapter side of each. For a device trying to present as an AF lens, that is the
more relevant grouping.

`pl[45..60]` is **not** a discriminator despite looking like the sharpest of them (eight natives
all-zero against three adapters carrying the same six-position structure). It is the aperture
descriptor above. The lesson generalises: *"natives send zero, adapters do not" can mean the field
belongs to a different device class, not that the adapter is leaking a tell* — and a field nobody
could name is not therefore a field nobody reads.

### Adapter behaviour

| Device | Slot A | Slot B | Index `pl[77..78]` | `pl[80]` | Slots C/D |
| --- | --- | --- | --- | --- | --- |
| Native lenses | Populated, changing | Populated, changing | `15/16/17`, cycling | `00`/`0c`/`12` | Populated |
| Yongnuo, current | Populated | Populated | Cycling | Computed from tag | Populated |
| Yongnuo 50 DF, older | Populated | Populated | Cycling | **`00` always** | Populated |
| TECHART LM-EA9, both Viltrox | `00 00 00 00 00 00` | `26 00 00 00 00 00` | `00 00`, **static** | `00` | Zeros |

The adapters send slot B tag `0x26` with five zero payload bytes — a **valid row with null data**,
decoding to a flat 1.500. See [the null rows adapters send](optical_data.md#7-the-null-rows-adapters-send).

**The body demonstrably consumes these rows.** An LM-EA9 whose only change was 19 payload bytes (a
native lens's slot A/B record, the donor's null row in slots C/D, `pl[95]`) made a Sony a9 II's
autofocus **measurably worse**. The sign is unhelpful but the fact is not: every other payload
experiment on that device returned no observable change, so this is the first demonstration that
message 0x05's optical slots reach a modern body's AF behaviour. It does **not** show which slot is
responsible; a same-session test varying the lens's physical aperture ring across f/1.5–f/2 produced
no matching change, which argues against slot B (the aperture term) and points at slot A.

Caveat on generalising it: the LM-EA9 sends **one** record for every focus distance and reports a
frozen focus position, so "populated rows made AF worse" and "*wrong* rows made AF worse" are not
separated by this result.

---

## Focal length — confirmed on nine lenses

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
| Canon EF 50/1.8 + Viltrox | 500 | 500 | 50 mm |
| Canon EF-S 24/2.8 + Viltrox | 240 | 240 | 24 mm |
| **TECHART LM-EA9** | **400** | **400** | **fixed 40 mm** |

Zooms report *slightly different* wide and tele values at a given position, so the pair is not a
simple duplicate — POSSIBLE that one is nominal and one is actual.

## Manufacturer notes

- **Sony** — natives populate all four optical slots and cycle the row index. They send zero in the
  aperture descriptor, carrying E-mount IDs and reporting aperture elsewhere.
- **Yongnuo** — computes almost every derived field live: `pl[80]` from the row tag, `pl[81..82]`
  from its own focal length plus a focus term, `pl[30..31]` from the live position. Its 22-entry
  grid table is shared across its lineup; only the step ladder is per-lens.
- **TECHART** — the LM-EA9 freezes `pl[0..1]`, `pl[6..7]` (at 16, a value no other device sends),
  `pl[8]`, `pl[22]`, `pl[23]`, `pl[62]` and the row index, and sends all-zero optical slots. It
  populates the aperture descriptor, and widens it at boot beyond the lens identity it presents.
- **Viltrox** — the EF adapters send zero optical slots and a static index, but populate the
  aperture descriptor from the mounted Canon lens.

## Open questions

- `pl[5]`, `pl[19]`, `pl[28..29]`, `pl[61]`: position known, meaning UNKNOWN.
- `pl[6..7]`: a start-up value whose quantity is UNKNOWN.
- `pl[22]` bits 0–5: per-lens and static, meaning UNKNOWN.
- `pl[48]` (`0xA0`) and `pl[59]` (`01`) inside the aperture descriptor.
- What slot A measures. Aperture-independent, focus-dependent, end-to-end ratio ≈ 2.
- What slot B's focus axis measures — the pupil reading fails there.
- The meaning of the `pl[80]` lookup values, as opposed to the mapping.
- Whether Sony populates `pl[81..82]` at all; every Sony lens measured sends 0.
- Why `pl[95]` is constant per Sony lens while the rows it sits with cycle.
