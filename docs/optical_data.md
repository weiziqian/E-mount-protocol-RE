# The 6-byte optical rows

The protocol moves the lens's optical description as **6-byte rows**. The same row format appears in
four slots of [message 0x05](msg_0x05.md) and is carried again by [message 0x28](msg_0x28.md) and
[message 0x35](msg_0x35.md). This document is the reference for that format: what each row carries,
where it appears, how it decodes, and the sample data behind every claim.

Confidence labels are the repository's: **CERTAIN / PROBABLE / POSSIBLE / UNKNOWN**.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

# 1. What the rows carry

Three row channels decode into exactly the three numbers a body needs to reconstruct the lens's
pupil geometry across the field.

| Row | Quantity | In plain terms | Status |
| --- | --- | --- | --- |
| **Slot A** | **The field grid** — the five sampling positions, as tangent angles seen from the exit pupil | *Where* across the field the other curves are sampled | **POSSIBLE** — §4 |
| **Slot B, type 0** | **Pupil position** — the pupil magnification `p`; exit-pupil distance `= p × f` at infinity | How far from the sensor the exit pupil sits | **PROBABLE** — §5 |
| **Slot B, type 1** | **Pupil size** — `value[0] = C/F`, `C ≈ 0.0998`, scaling `1/√2` per stop | How large the aperture looks | `∝ 1/F` **CERTAIN**; the cross-lens constant **CERTAIN at infinity** — §6 |

Two further slots, **C and D**, use the same encoding for different quantities that remain UNKNOWN.
See §8.

## Why these three belong together

- Pupil size, pupil distance and a field-position grid are precisely the inputs of a pupil-shift
  correction — the mechanism behind off-centre PDAF error on adapters.
- The readings **cross-check numerically.** `corner_height / A[4]` (slot A) and `p × f` (slot B
  type 0) are two fully independent estimates of the exit-pupil distance, and they agree within 3 %
  on the three lenses whose grid samples out to the corner (§4).
- Every implied exit-pupil distance lands on the lens's known physical character — 26 mm for the
  Voigtländer 15 (famous for its short exit pupil and the corner trouble it causes), 94 mm for the
  Sony SEL5518Z (a digital-era design sitting at the sensor's designed pupil distance), 32 mm for a
  collapsing pancake zoom (§5).

## What this framework does NOT explain — the focus axis

Slot A's magnitude and slot B type 1 both fall steeply as focus moves closer — far faster than any
pupil geometry can change, inconsistently between vendors, and with table structure that looks
authored rather than measured (§4.3, §6.4).

So: **the infinity-end rows carry the pupil description; what a vendor does to them as focus moves
closer is UNKNOWN.** Only infinity-end rows are trustworthy pupil data.

---

# 2. Where the rows appear

| Message | Slot | Field |
| --- | --- | --- |
| [0x05](msg_0x05.md) | A | `pl[32..37]` |
| | B | `pl[38..43]` |
| | C | `pl[83..88]` |
| | D | `pl[89..94]` |
| [0x28](msg_0x28.md) | A | `[0x11..0x16]` — frame-relative |
| | B | `[0x17..0x1C]` — contiguous with A |
| [0x35](msg_0x35.md) | A | `[0x11..0x16]` — frame-relative |
| | B | `[0x1A..0x1F]` — separated from A by three constant bytes |

All three messages read **the same per-focus table**. One record holds slot A at bytes `[0..5]` and
slot B at `[6..11]`.

**The table is indexed by the focus position**, on the protocol's own distance grid — one bucket per
grid point:

```
bucket = clamp(pos * 3 / 256, min 53) - 53
```

`pos` is the [live focus position](live_focus_position.md), the `n × 256/3` value all three messages
also carry, and `pos × 3 / 256` is exactly its grid index `n`. That is the mechanism behind the wire
observation that slot A changes only when focus moves — and it means the resolution of the
correction data is the resolution of that scale, nothing finer.

**The parity rule.** A table holds `2N` records for `N` focus buckets. Bucket `k` owns records `2k`
and `2k+1`; the pair **shares one slot A** and carries **two different slot Bs** — one type 1 and
one type 0.

| Message | Parity | Consequence |
| --- | --- | --- |
| 0x05 | Alternates by frame | Both types reach the body, one per frame |
| 0x28, 0x35 | Always parity 0 | Only the even record of each bucket is reachable |

So a body that samples only one parity of message 0x05 gets only one of the two quantities.

---

# 3. How to decode

A row is **not** "a tag byte followed by five values". It is a **block-float 5-point curve**.
CERTAIN.

```
 byte 0   bit 7    row TYPE flag
          bit 6..4 3-bit exponent E
          bit 3..0 high 4 bits of the 12-bit start value
 byte 1            low  8 bits of the 12-bit start value
 byte 2..5         four SIGNED 8-bit DELTAS

 v0 = ((byte0 & 0x0f) << 8) | byte1
 v1 = v0 + s8(byte2)
 v2 = v1 + s8(byte3)
 v3 = v2 + s8(byte4)
 v4 = v3 + s8(byte5)

 value[i] = v[i] * 2^-E
```

Worked example — the Zeiss Loxia 21/2.8's slot B row `f4 a8 be ba c9 da`:

```
byte0 = 0xF4 -> type 1, E field = 7, value high nibble = 4
v0 = 0x4A8 = 1192
deltas = -66, -70, -55, -38
v  = 1192, 1126, 1056, 1001, 963
value = 0.03638, 0.03436, 0.03223, 0.03055, 0.02939      (absolute E = 15, see §3.3)
```

## 3.1 Why the last four bytes are deltas, not values — CERTAIN

Read as five independent values, every row has a first number in the hundreds followed by four
numbers in the tens or negative — a discontinuity in the middle of what is otherwise a smooth
family. Read as **start plus four deltas**, every row from every device resolves into a smooth,
monotone 5-point curve, and the curves of successive table entries nest without crossing.

The same test on the null row adapters send, `26 00 00 00 00 00`, gives a **perfectly flat curve at
`0x600`**, not "a tag with no data" — see §7.

## 3.2 Why byte 0's high nibble is a shared exponent — CERTAIN

Three consecutive slot-A entries from the Yongnuo 50 mm F1.8S DF's per-focus table, buckets 6, 7
and 9:

```
b0 3e 16 10 0d 08   ->  62, 22, 16, 13,  8
c0 7c 2b 22 19 11   -> 124, 43, 34, 25, 17     ~ x2
d0 f8 54 46 33 21   -> 248, 84, 70, 51, 33     ~ x4
```

`0x3E → 0x7C → 0xF8` is exact doubling, and **all five components double together** while the
decoded curve stays put (0.03027, 0.0410, 0.0490, 0.0552, 0.0591 in all three). That is a shared
exponent, not five coincidences.

## 3.3 The exponent field is 3 bits and WRAPS — the one sharp edge

`E` is only bits 6..4, so it counts 0…7 and rolls over. Observed nibble progressions run
`b → c → d → e → f → 8 → 9 → a`, each step **halving** the represented value. **The absolute octave
cannot be read from a single row.**

**Proof that it wraps rather than jumping**, from the Yongnuo 50 mm F1.8S DF's slot-B table,
buckets 6…11: the mantissa falls 1167 → 1065 → 963 within nibble `f` (−102 per step, ≈ −9.6 %/step)
and then continues 1720 → 1511 → 1311 within nibble `8` (−209 per step, ≈ −12 %/step). Only
`E(8) = E(f) + 1` makes 963 → 1720 a −10.7 % step consistent with its neighbours; reading the nibble
as a plain 4-bit exponent makes it a ×229 jump inside a per-bucket table.

**Second, independent proof — in slot A and on a Sony lens**, so it is not one vendor's artefact.
The SEL55210 emits slot-A rows with nibble `b` and with nibble `a` a few frames apart in the same
init:

```
b1 53 75 66 35 29   mantissa 339   ->  0.1655   (nibble b, E = 11)
a0 97 42 2d 15 0d   mantissa 151   ->  0.1475   (nibble a, E = 10)
```

Under `a = 10` the two rows are neighbours. Under the *continuation* reading `a = 18` that slot B
demands, 151 would decode to 0.00058 — three orders of magnitude from anything the same lens sends.
**So the octave a nibble denotes is per slot**, and slot A's `a` is one octave *coarser* than its
`b`, not seven finer.

### How to resolve it in practice

Each slot/type pair uses a narrow window. Census over 2 310 message-0x05 frames, non-zero rows only:

| Slot | Type 1 nibbles (bit 7 = 1) | Type 0 nibbles (bit 7 = 0) |
| --- | --- | --- |
| A | `a` ×288, `b` ×1660, `c` ×3, `e` ×3 | — |
| B | `8` ×532, `9` ×31, `a` ×83, `b` ×2, `c` ×1, `d` ×2, `e` ×156, `f` ×207 | `1` ×108, `2` ×1091, `3` ×97 |
| C | `d` ×242, `e` ×192, `f` ×415, `9` ×1 | `0` ×830, `1` ×244 |
| D | `a` ×25, `b` ×172, `c` ×2, `d` ×242, `e` ×58, `f` ×342 | `0` ×847, `1` ×37, `2` ×120, `3` ×79 |

Slot A effectively uses two exponents, so it is unambiguous. **Slot B type 1 is the only channel
that spans the full field**, because it tracks the f-number over many stops (§6), and it is the only
place where the wrap actually bites.

Anchoring `b = 11` for slot A and slot B puts every lens measured into one consistent band. That
anchor is a **PROBABLE** convention, not a proven one. **All *ratios* in this document are
independent of the anchor**; only absolute magnitudes depend on it.

How a real body disambiguates is **UNKNOWN**. The mantissa is not normalised to a fixed range
(values from 457 to 3929 are observed), so the body is presumed to know the expected magnitude per
slot, or to track continuity across frames.

## 3.4 Byte 0 bit 7 — the row TYPE flag — PROBABLE

Two different quantities share slot B on alternating frames. Bit 7 says which:

- **type 1** (nibbles `8`…`f`) — the pupil-size curve (§6).
- **type 0** (nibbles `0`…`7`) — the much flatter pupil-magnification row (§5).

Evidence it is a type flag and not an exponent bit:

1. Implementations store the two as a **pair per table entry** and select between them with a frame
   parity bit — one entry yields one type-1 row and one type-0 row. That reproduces the long-standing
   wire observation "slot B alternates between 2–3 rows every frame".
2. The two types occupy disjoint, far-apart value ranges in the same slot (type 1 ≈ 0.005–0.06,
   type 0 ≈ 1.0–2.0 under the §3.3 anchor). A single quantity would not.
3. Slot A carries **only** type 1; slots B, C and D carry both.

## 3.5 Reference decoder

```python
def decode_row(r, E):
    """r = 6 bytes; E = absolute exponent (see 3.3). Returns 5 floats."""
    v = ((r[0] & 0x0f) << 8) | r[1]
    out = [v]
    for b in r[2:6]:
        v += b - 256 if b > 127 else b
        out.append(v)
    return [x / 2.0 ** E for x in out]

def row_type(r):
    return 1 if r[0] & 0x80 else 0        # see 3.4

def exp_field(r):
    return (r[0] >> 4) & 7                # 3 bits, wraps -- see 3.3
```

## 3.6 Alternative encodings tested and ELIMINATED

Two rival readings of the same bytes were tested against the whole corpus, and both fail:

- **Logarithmic mantissa** — `value = 2^(m/K − E)`, which would have dissolved the exponent wrap
  into one continuous log scale. The Voigtländer aperture ladder (§6.1) fixes K = 966–976 across
  four consecutive stops, but §3.2's doubling rows require K = 62 and K = 124 — the same encoding
  cannot satisfy both, and the two doubling steps do not even agree with each other. **The mantissa
  is linear and the exponent binary; the wrap is a real property of the format** and cannot be
  decoded away.
- **Chained curve** — the 5 points as samples along the *focus* axis, continuing into later table
  entries. On the YN50 DF, bucket 0's row falls *faster* (0.0526 → 0.0374) than the chain of the
  following buckets' `value[0]` (0.0526 → 0.0416), while bucket 3's row falls *slower* than its
  chain — inconsistent in both directions, so the 5 points are not a resampling of the focus axis.

---

# 4. Slot A — the field sampling grid

## 4.1 What it means

**Slot A is not a quantity sampled at five field positions — it IS the five field positions**,
expressed as tangent angles seen from the exit pupil. POSSIBLE, and the preferred reading.

What is established:

| Property | Evidence | Confidence |
| --- | --- | --- |
| Rises monotonically across the 5 points | Every non-zero slot-A row from every device | **CERTAIN** |
| `value[4] / value[0] ≈ 1.87 – 1.99` on **every** lens | 9 lenses, 14 rows (§4.2), holding across every aperture (§4.3) and every focus position (§4.4) — 30+ rows in all | **CERTAIN** |
| Does not change with aperture | Voigtländer 15/4.5 swept f/4.5 → f/22: 0.1445 → 0.1260, and all of the drop happens at the first click | **CERTAIN** |
| Changes strongly with focus | SEL5518Z 0.1191 → 0.0439 during one init sweep; per-focus tables move it 1.3–4× | **CERTAIN** |
| Its 5 values are the sampling **positions** for the other rows | Explains the ratio-2 lock, the aperture independence and the non-zero start; cross-checks against §5's pupil magnification on three lenses | **POSSIBLE** |

The end-to-end ratio pinning at ≈ 2 while the interior shape varies (second point 1.23 – 1.48) is
the strongest structural clue: the curve looks **normalised at both ends**, with the information
carried in the interior shape and in the overall magnitude.

Three facts fall out of the grid reading at once:

1. **Aperture independence** — sample positions are geometry; the diaphragm cannot move them.
2. **The non-zero first point** — the grid simply starts off-axis. The on-axis values of a pupil
   description are trivial (no vignetting, no shift), so only the outer field is worth sampling.
3. **The end-to-end ratio locked at 2** — a convention: the outermost sampled position is twice the
   innermost, and the lens is free to place the three interior points, which is exactly the only
   thing that varies per lens.

### The cross-check against slot B type 0

If slot A is a tangent-angle grid, then `corner_height / value[4]` estimates the exit-pupil
distance — exact when the lens samples right out to the corner, an overestimate when its grid stops
short. Slot B type 0 gives a fully independent second estimate, `p × f` (§5).

Infinity / first-frame rows; corner = 21.6 mm full-frame, 14.2 mm APS-C:

| Lens | A[4] | corner ÷ A[4] | `p` (§5) | `p × f` | Outermost sample `= p·f·A[4]`, as % of corner |
| --- | --- | --- | --- | --- | --- |
| Sony SEL5518Z (FF) | 0.2363 | 91 mm | 1.703 | 94 mm | **102 % — samples to the corner, estimates agree** |
| Sony SELP1650 @16 (APS-C) | 0.4531 | 31 mm | 2.020 | 32 mm | **103 % — agree** |
| Yongnuo 35 DA (APS-C) | 0.3438 | 41 mm | 1.211 | 42 mm | **103 % — agree** |
| Yongnuo 50 DF (FF) | 0.2329 | 93 mm | 1.164 | 58 mm | 63 % — grid stops inside the corner, consistent |
| Zeiss Loxia 21 (FF) | 0.2393 | 90 mm | 1.787 | 38 mm | 42 % — consistent |
| Voigtländer 15 (FF) | 0.2827 | 76 mm | 1.758 | 26 mm | 35 % — consistent |
| Sony SEL2870 @70 (FF) | 0.1191 | 181 mm | 0.816 | 57 mm | 31 % — consistent |
| Sony SEL55210 @55 (APS-C) | 0.3184 | 45 mm | 1.004 | 55 mm | **124 % — VIOLATION** |
| Sony SEL55210 @210 (APS-C) | 0.2529 | 56 mm | 0.660 | 139 mm | **247 % — VIOLATION** |

Three lenses land the two independent estimates within 3 % of each other — that agreement is the
strongest evidence for the reading. Four more are consistent, with the grid stopping short of the
corner. **The SEL55210 breaks the reading in both zoom states**: its implied outermost sample lands
beyond the corner, which a position grid cannot do. Both its rows come from settled mid-session
states rather than a proven infinity state, and `EPD = p × f` holds only at infinity — but until a
clean infinity-focus observation resolves it, this zoom stands as the open counter-example.

## 4.2 Sample data — different lenses

Every row below is a real row a named lens actually sent or carries. **State** says which sample of
that lens it is.

| Lens | State | Row | E | Mantissas | Decoded value[0..4] | Normalised to value[0] |
| --- | --- | --- | --- | --- | --- | --- |
| Voigtländer 15/4.5 | f/4.5, ∞ | `b1 28 47 51 4b 38` | 11 | 296 367 448 523 579 | 0.1445 0.1792 0.2188 0.2554 0.2827 | 1.000 1.240 1.514 1.767 **1.956** |
| Zeiss Loxia 21/2.8 | f/2.8 | `b0 fe 45 47 39 27` | 11 | 254 323 394 451 490 | 0.1240 0.1577 0.1924 0.2202 0.2393 | 1.000 1.272 1.551 1.776 **1.929** |
| Sony SELP1650 @16 mm | Settled | `a0 ea 70 49 1e 0f` | 10 | 234 346 419 449 464 | 0.2285 0.3379 0.4092 0.4385 0.4531 | 1.000 1.479 1.791 1.919 **1.983** |
| Sony SELP1650 @16 mm | Init | `a0 ed 6d 4a 22 10` | 10 | 237 346 420 454 470 | 0.2314 0.3379 0.4102 0.4434 0.4590 | 1.000 1.460 1.772 1.916 **1.983** |
| Sony SEL5518Z 55/1.8 | First frame | `b0 f4 46 46 3f 25` | 11 | 244 314 384 447 484 | 0.1191 0.1533 0.1875 0.2183 0.2363 | 1.000 1.287 1.574 1.832 **1.984** |
| Sony SEL5518Z 55/1.8 | Settled | `b0 5a 27 18 0f 09` | 11 | 90 129 153 168 177 | 0.0439 0.0630 0.0747 0.0820 0.0864 | 1.000 1.433 1.700 1.867 **1.967** |
| Sony SEL2870 @70 mm | Settled | `b0 7f 2f 1d 17 12` | 11 | 127 174 203 226 244 | 0.0620 0.0850 0.0991 0.1104 0.1191 | 1.000 1.370 1.598 1.780 **1.921** |
| Sony SEL55210 @55 mm | First frame | `b1 53 75 66 35 29` | 11 | 339 456 558 611 652 | 0.1655 0.2227 0.2725 0.2983 0.3184 | 1.000 1.345 1.646 1.802 **1.923** |
| Sony SEL55210 @55 mm | Settled | `a0 97 42 2d 15 0d` | 10 | 151 217 262 283 296 | 0.1475 0.2119 0.2559 0.2764 0.2891 | 1.000 1.437 1.735 1.874 **1.960** |
| Sony SEL55210 @210 mm | First frame | `b1 11 3f 40 3e 38` | 11 | 273 336 400 462 518 | 0.1333 0.1641 0.1953 0.2256 0.2529 | 1.000 1.231 1.465 1.692 **1.897** |
| Sony SEL55210 @210 mm | Settled | `b1 00 41 41 38 2f` | 11 | 256 321 386 442 489 | 0.1250 0.1567 0.1885 0.2158 0.2388 | 1.000 1.254 1.508 1.727 **1.910** |
| Yongnuo 35/1.8 DA | Bucket 0 (∞) | `b1 78 55 5b 50 48` | 11 | 376 461 552 632 704 | 0.1836 0.2251 0.2695 0.3086 0.3438 | 1.000 1.226 1.468 1.681 **1.872** |
| Yongnuo 50/1.8 DF | Bucket 0 (∞) | `b0 f5 48 3f 3c 25` | 11 | 245 317 380 440 477 | 0.1196 0.1548 0.1855 0.2148 0.2329 | 1.000 1.294 1.551 1.796 **1.947** |
| Yongnuo 50/1.8 DA | Bucket 0 (∞) | `b1 30 7e 4b 32 29` | 11 | 304 430 505 555 596 | 0.1484 0.2100 0.2466 0.2710 0.2910 | 1.000 1.414 1.661 1.826 **1.961** |
| **TECHART LM-EA9** | All frames | `00 00 00 00 00 00` | — | 0 0 0 0 0 | 0 0 0 0 0 | — |
| **Viltrox EF-NEX II + Canon EF 50/1.8** | All frames | `00 00 00 00 00 00` | — | 0 0 0 0 0 | 0 0 0 0 0 | — |
| **Viltrox EF-NEX II + Canon EF-S 24/2.8** | All frames | `00 00 00 00 00 00` | — | 0 0 0 0 0 | 0 0 0 0 0 | — |

Fourteen rows from nine lenses, four manufacturers, 15 mm to 210 mm — the last column never leaves
`1.87 … 1.99`, and it does not care what the magnitude in `value[0]` is doing (0.044 to 0.459, a
factor of ten).

The deltas also cluster tightly across unrelated lenses — `+69 +71 +57 +39` (Loxia),
`+70 +70 +63 +37` (SEL5518Z), `+72 +63 +60 +37` (Yongnuo 50 DF), `+71 +81 +75 +56` (Voigtländer).
The row is close to "a magnitude plus a near-universal ramp".

## 4.3 Slot A does not move with aperture

Same lens, same session, focus locked at infinity, the aperture ring closed one click at a time.
Compare §6.1, where slot B moves by a factor of 4.6 over these same six frames:

| Marked f-stop | Row | E | Mantissas | Decoded value[0..4] | Normalised |
| --- | --- | --- | --- | --- | --- |
| f/4.5 | `b1 28 47 51 4b 38` | 11 | 296 367 448 523 579 | 0.1445 0.1792 0.2188 0.2554 0.2827 | 1.000 1.240 1.514 1.767 1.956 |
| f/5.6 | `b1 01 39 46 3e 37` | 11 | 257 314 384 446 501 | 0.1255 0.1533 0.1875 0.2178 0.2446 | 1.000 1.222 1.494 1.735 1.949 |
| f/8 | `b1 04 3e 42 40 36` | 11 | 260 322 388 452 506 | 0.1270 0.1572 0.1895 0.2207 0.2471 | 1.000 1.238 1.492 1.738 1.946 |
| f/11 | `b1 02 3e 42 3f 38` | 11 | 258 320 386 449 505 | 0.1260 0.1562 0.1885 0.2192 0.2466 | 1.000 1.240 1.496 1.740 1.957 |
| f/16 | `b1 03 3e 42 40 36` | 11 | 259 321 387 451 505 | 0.1265 0.1567 0.1890 0.2202 0.2466 | 1.000 1.239 1.494 1.741 1.950 |
| f/22 | `b1 02 3f 42 3f 37` | 11 | 258 321 387 450 505 | 0.1260 0.1567 0.1890 0.2197 0.2466 | 1.000 1.244 1.500 1.744 1.957 |

The whole five-stop sweep moves `value[0]` by 12.8 %, and **all of it happens at the first click**;
f/5.6 → f/22 moves it by 0.4 %. Whatever slot A describes, it is fixed by the optics, not by the
diaphragm — the one exception being the same wide-open irregularity slot B shows (§6.1).

## 4.4 Slot A moves strongly with focus

| Lens | Sample | Row | E | value[0] | value[4]/value[0] |
| --- | --- | --- | --- | --- | --- |
| Sony SEL5518Z | Init, frame 1 | `b0 f4 46 46 3f 25` | 11 | 0.1191 | 1.984 |
| Sony SEL5518Z | Init, frame 5 | `b0 6c 34 1d 0f 09` | 11 | 0.0527 | 1.972 |
| Sony SEL5518Z | Init, frame 9 → end | `b0 5a 27 18 0f 09` | 11 | 0.0439 | 1.967 |
| Voigtländer 15/4.5 | Ring at ∞ (reported distance 1792) | `b1 28 47 51 4b 38` | 11 | 0.1445 | 1.956 |
| Voigtländer 15/4.5 | Ring at 30 cm (reported 272) | `b1 23 47 4e 47 3a` | 11 | 0.1421 | 1.955 |
| Yongnuo 50/1.8 DF | Bucket 0 (∞) | `b0 f5 48 3f 3c 25` | 11 | 0.1196 | 1.947 |
| Yongnuo 50/1.8 DF | Bucket 3 | `b0 6c 40 15 0d 07` | 11 | 0.0527 | 1.972 |
| Yongnuo 35/1.8 DA | Bucket 0 (∞) | `b1 78 55 5b 50 48` | 11 | 0.1836 | 1.872 |
| Yongnuo 35/1.8 DA | Bucket 4 | `b1 17 7a 5f 28 0f` | 11 | 0.1362 | 1.975 |

Three things to read off this:

- The **magnitude** is the focus-carried information — 2.7× on the SEL5518Z inside a single power-on
  sequence, 2.3× across the Yongnuo 50 DF's first four buckets. Compare the 12.8 % the aperture ring
  buys in §4.3.
- The **end-to-end ratio survives all of it**, staying inside 1.87–1.99 in every row above.
- The 15 mm ultra-wide barely moves at all (1.7 % over its whole 30 cm → ∞ range) and uses only
  **two** buckets for the entire ring; the 55 mm moves 2.7×. That is the right scaling if the
  quantity depends on focus *extension*, which goes as focal length.

The Voigtländer's two buckets are worth stating precisely, because it is the one observation where
the switch point is visible. The reported-distance field `pl[20..21]` runs 272 → 299 → 320 → 351 →
384 → 448 → 1792 as the ring turns, and the row changes **exactly at 448**, with 448 itself observed
under both rows.

**A geometric grid has no reason to shrink 2.7× during one focus sweep.** Under the preferred
reading this axis is recorded as UNKNOWN, the same as slot B type 1's (§6.4).

## 4.5 Sample data — one lens across its whole focus range

Both readable Yongnuo lenses' complete per-focus slot-A tables. `steps` is the motor-step range that
maps to that bucket. Bucket 20 on the 35 DA is a spare the travel never reaches.

### Yongnuo YN35mm F1.8S DA (APS-C, 35 mm) — 21 focus buckets

| k | Steps | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0–45 | `b1 78 55 5b 50 48` | 11 | 0.18359 | 0.22510 | 0.26953 | 0.30859 | 0.34375 |
| 1 | 46–89 | `b1 52 61 63 42 2e` | 11 | 0.16504 | 0.21240 | 0.26074 | 0.29297 | 0.31543 |
| 2 | 90–121 | `b1 3a 69 68 39 1f` | 11 | 0.15332 | 0.20459 | 0.25537 | 0.28320 | 0.29834 |
| 3 | 122–165 | `b1 22 71 6d 30 0e` | 11 | 0.14160 | 0.19678 | 0.25000 | 0.27344 | 0.28027 |
| 4 | 166–213 | `b1 17 7a 5f 28 0f` | 11 | 0.13623 | 0.19580 | 0.24219 | 0.26172 | 0.26904 |
| 5 | 214–261 | `a0 86 42 29 10 07` | 10 | 0.13086 | 0.19531 | 0.23535 | 0.25098 | 0.25781 |
| 6–19 | 262–704 | `a0 80 47 22 0c 07` | 10 | 0.12500 | 0.19434 | 0.22754 | 0.23926 | 0.24609 |

### Yongnuo YN50mm F1.8S DF (full frame, 50 mm) — 23 focus buckets

| k | Steps | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0–45 | `b0 f5 48 3f 3c 25` | 11 | 0.11963 | 0.15479 | 0.18555 | 0.21484 | 0.23291 |
| 1 | 46–101 | `b0 d0 46 33 30 1d` | 11 | 0.10156 | 0.13574 | 0.16064 | 0.18408 | 0.19824 |
| 2 | 102–149 | `b0 9e 43 25 1e 12` | 11 | 0.07715 | 0.10986 | 0.12793 | 0.14258 | 0.15137 |
| 3 | 150–185 | `b0 6c 40 15 0d 07` | 11 | 0.05273 | 0.08398 | 0.09424 | 0.10059 | 0.10400 |
| 4 | 186–216 | `b0 5b 30 14 0d 07` | 11 | 0.04443 | 0.06787 | 0.07764 | 0.08398 | 0.08740 |
| 5 | 217–261 | `b0 4a 21 12 0c 08` | 11 | 0.03613 | 0.05225 | 0.06104 | 0.06689 | 0.07080 |
| 6 | 262–296 | `b0 3e 16 10 0d 08` | 11 | 0.03027 | 0.04102 | 0.04883 | 0.05518 | 0.05908 |
| 7 | 297–340 | `c0 7c 2b 22 19 11` | 12 | 0.03027 | 0.04077 | 0.04907 | 0.05518 | 0.05933 |
| 8 | 341–376 | `c0 7c 2a 23 19 11` | 12 | 0.03027 | 0.04053 | 0.04907 | 0.05518 | 0.05933 |
| 9 | 377–420 | `d0 f8 54 46 33 21` | 13 | 0.03027 | 0.04053 | 0.04907 | 0.05530 | 0.05933 |
| 10 | 421–455 | `d0 f9 55 46 33 21` | 13 | 0.03040 | 0.04077 | 0.04932 | 0.05554 | 0.05957 |
| 11–22 | 456–808 | `d0 fa 56 46 33 21` | 13 | 0.03052 | 0.04102 | 0.04956 | 0.05579 | 0.05981 |

**Slot A saturates early, and the two lenses saturate differently.** The 35 DA stops changing after
bucket 6 (step 262 of 704, 37 % of travel) and repeats one row for the remaining fourteen buckets.
The 50 DF runs to bucket 11 (step 456 of 808, 56 %) before settling. **More than half of each table
is a constant** — which is why freezing a single row costs less than it looks like it should, and
why the row that matters is the infinity end.

**The exponent wrap is visible in both.** The 35 DA steps `b1 → a0` between buckets 4 and 5; the
50 DF steps `b0 → c0 → d0` at buckets 7 and 9, with the decoded value unchanged across the first of
those (§3.2).

---

# 5. Slot B type 0 — the pupil magnification `p`

## 5.1 What it means

Values sit between 0.66 and 2.02 under the §3.3 anchor and are nearly flat across the five points:
a ratio-like quantity. The preferred reading identifies it as the **pupil magnification** — exit
pupil ÷ entrance pupil diameter, equivalently exit-pupil distance ÷ focal length, since
`EPD = p × f` exactly at infinity focus. **PROBABLE.**

It changes by ~1 % across a five-stop aperture sweep, so it is **not** aperture-driven — consistent
with pupil geometry, which the diaphragm cannot move.

Anchor used in this section: nibble `1` → E 9, `2` → E 10, `3` → E 11.

### The evidence, in order of strength

1. **Design-class ordering, 10 devices, no exception.** Ranked by `value[0]`: SELP1650 @16 mm
   **2.02** (strong retrofocus pancake, wide end) > Loxia 21 1.79 (retrofocus Distagon) >
   Voigtländer 15 1.76 (retrofocus) > SEL5518Z 1.70 > adapters' default 1.50 > YN35 1.21 (mild
   retrofocus) > YN50 1.16 (≈ symmetric) > SEL55210 @55 1.00 (zoom short end) > SEL2870 @70 0.82
   (tele end) > SEL55210 @210 **0.66** (tele end). That is exactly the ordering pupil magnification
   takes across those design types — retrofocus > 1, symmetric ≈ 1, telephoto < 1.
2. **Two of the ten are out-of-sample predictions.** The SELP1650 and SEL55210 @210 rows were
   decoded *after* the prediction was made from the other seven, and landed at the two extreme ends
   as predicted.
3. **The implied exit-pupil distances (`p × f`) match each lens's known physical character.**
   Voigtländer 15 → 26 mm: this lens's very short exit pupil, and the corner behaviour it causes on
   digital full-frame, are its best-known property. SELP1650 → 32 mm: collapsing pancake zoom,
   heavily correction-dependent in the corners. Loxia 21 → 38 mm: compact, moderately
   non-telecentric. SEL5518Z → **94 mm**: a digital-era flagship prime sitting at the sensor's
   designed pupil distance (~100 mm). SEL55210 @210 → 139 mm: tele end, exit pupil far. **No value
   in the set is physically implausible.**
4. **Independent cross-check via slot A**: `corner ÷ A[4]` agrees with `p × f` within 3 % on the
   three lenses whose grid reaches the corner (§4.1) — two unrelated slots telling one story.

## 5.2 Sample data — different lenses

| Device | Row | E | Mantissas | Decoded value[0..4] |
| --- | --- | --- | --- | --- |
| TECHART LM-EA9 / Viltrox | `26 00 00 00 00 00` | 10 | 1536 1536 1536 1536 1536 | 1.500 1.500 1.500 1.500 1.500 |
| Sony SELP1650 @16 | `14 0a e0 fe 18 04` | 9 | 1034 1002 1000 1024 1028 | 2.020 1.957 1.953 2.000 2.008 |
| Voigtländer 15/4.5 @f4.5 | `27 08 35 30 40 39` | 10 | 1800 1853 1901 1965 2022 | 1.758 1.810 1.856 1.919 1.975 |
| Zeiss Loxia 21/2.8 | `27 26 20 1a ee e3` | 10 | 1830 1862 1888 1870 1841 | 1.787 1.818 1.844 1.826 1.798 |
| Sony SEL5518Z | `26 d0 e4 e8 e4 60` | 10 | 1744 1716 1692 1664 1760 | 1.703 1.676 1.652 1.625 1.719 |
| Sony SEL2870 @70 | `36 88 20 10 08 08` | 11 | 1672 1704 1720 1728 1736 | 0.816 0.832 0.840 0.844 0.848 |
| Sony SEL55210 @55 | `24 04 dc ec f0 f8` | 10 | 1028 992 972 956 948 | 1.004 0.969 0.949 0.934 0.926 |
| Sony SEL55210 @210 | `35 48 00 f0 f0 c8` | 11 | 1352 1352 1336 1320 1264 | 0.660 0.660 0.652 0.645 0.617 |
| Yongnuo 35/1.8 DA | `24 d8 08 10 08 00` | 10 | 1240 1248 1264 1272 1272 | 1.211 1.219 1.234 1.242 1.242 |
| Yongnuo 50/1.8 DF, ∞ | `24 a8 f0 f0 f0 f4` | 10 | 1192 1176 1160 1144 1132 | 1.164 1.148 1.133 1.117 1.105 |

## 5.3 Sample data — one lens across its whole focus range

### Yongnuo YN35mm F1.8S DA — constant across every bucket

Every one of the 21 buckets carries the **same** type-0 row:

| Buckets | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0–20 (all) | `24 d8 08 10 08 00` | 10 | 1.21094 | 1.21875 | 1.23438 | 1.24219 | 1.24219 |

Pupil magnification being focus-independent on this lens is exactly what the geometric reading
predicts.

### Yongnuo YN50mm F1.8S DF — drifts slightly with focus

| k | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | `24 a8 f0 f0 f0 f4` | 10 | 1.16406 | 1.14844 | 1.13281 | 1.11719 | 1.10547 |
| 1 | `24 a4 f0 f0 f4 f4` | 10 | 1.16016 | 1.14453 | 1.12891 | 1.11719 | 1.10547 |
| 2 | `24 a4 f4 f0 f4 f8` | 10 | 1.16016 | 1.14844 | 1.13281 | 1.12109 | 1.11328 |
| 3 | `24 a0 f4 f4 f4 f8` | 10 | 1.15625 | 1.14453 | 1.13281 | 1.12109 | 1.11328 |
| 4 | `24 a0 f4 f8 f8 f8` | 10 | 1.15625 | 1.14453 | 1.13672 | 1.12891 | 1.12109 |
| 5 | `24 a0 f8 f8 f8 fc` | 10 | 1.15625 | 1.14844 | 1.14062 | 1.13281 | 1.12891 |
| 6 | `24 9c f8 f8 fc fc` | 10 | 1.15234 | 1.14453 | 1.13672 | 1.13281 | 1.12891 |
| 7 | `24 9c f8 f8 fc fc` | 10 | 1.15234 | 1.14453 | 1.13672 | 1.13281 | 1.12891 |
| 8 | `24 98 f8 f8 fc fc` | 10 | 1.14844 | 1.14062 | 1.13281 | 1.12891 | 1.12500 |
| 9 | `24 94 f8 f8 fc fc` | 10 | 1.14453 | 1.13672 | 1.12891 | 1.12500 | 1.12109 |
| 10 | `24 90 f8 f8 fc fc` | 10 | 1.14062 | 1.13281 | 1.12500 | 1.12109 | 1.11719 |
| 11 | `24 90 f8 f8 fc fc` | 10 | 1.14062 | 1.13281 | 1.12500 | 1.12109 | 1.11719 |
| 12 | `24 8c f8 f8 fc fc` | 10 | 1.13672 | 1.12891 | 1.12109 | 1.11719 | 1.11328 |
| 13 | `24 8c f8 f8 fc fc` | 10 | 1.13672 | 1.12891 | 1.12109 | 1.11719 | 1.11328 |
| 14 | `24 8c f8 f8 fc 00` | 10 | 1.13672 | 1.12891 | 1.12109 | 1.11719 | 1.11719 |
| 15 | `24 8c f8 f8 fc 00` | 10 | 1.13672 | 1.12891 | 1.12109 | 1.11719 | 1.11719 |
| 16 | `24 8c f8 f8 fc 00` | 10 | 1.13672 | 1.12891 | 1.12109 | 1.11719 | 1.11719 |
| 17–22 | `24 8c f8 fc f8 00` | 10 | 1.13672 | 1.12891 | 1.12500 | 1.11719 | 1.11719 |

`value[0]` falls 1.164 → 1.137 over the whole travel — **2.3 %**, against the 9× that slot B type 1
moves over the same range. Whatever the focus axis is doing to type 1, it is not doing it here.

---

# 6. Slot B type 1 — the pupil-size / f-number row

## 6.1 It is proportional to 1/F — CERTAIN

From an observation that moves **only** the aperture ring, focus locked at infinity — Voigtländer
15/4.5, six click stops:

| Marked f-stop | Row | E | Mantissas | Decoded value[0..4] | Step ratio | `value[0] × F` |
| --- | --- | --- | --- | --- | --- | --- |
| f/4.5 | `85 42 97 86 97 b6` | 16 | 1346 1241 1119 1014 940 | 0.02054 0.01894 0.01707 0.01547 0.01434 | — | 0.0924 |
| f/5.6 | `84 96 c1 ba bb cc` | 16 | 1174 1111 1041 972 920 | 0.01791 0.01695 0.01588 0.01483 0.01404 | 0.8723 | 0.1003 |
| f/8 | `96 7b a7 9e 9e b6` | 17 | 1659 1570 1472 1374 1300 | 0.01266 0.01198 0.01123 0.01048 0.00992 | **0.70654** | 0.1013 |
| f/11 | `94 93 c3 b9 bc cb` | 17 | 1171 1110 1039 971 918 | 0.00893 0.00847 0.00793 0.00741 0.00700 | **0.70585** | 0.0983 |
| f/16 | `a6 77 a9 9e 9e b5` | 18 | 1655 1568 1470 1372 1297 | 0.00631 0.00598 0.00561 0.00523 0.00495 | **0.70663** | 0.1010 |
| f/22 | `a4 94 c2 bb ba cb` | 18 | 1172 1110 1041 971 918 | 0.00447 0.00423 0.00397 0.00370 0.00350 | **0.70823** | 0.0984 |

Notice the raw mantissas barely move between f/5.6 and f/22 (1174 → 1172, 1171 → 1172) — **the
entire stop-by-stop scaling is carried by the exponent nibble** `8 → 9 → a`. Decoding without §3.2
makes this ladder look like noise.

Every full stop multiplies the row by **0.7068 ± 0.0010**, against `1/√2 = 0.70711`. A quantity that
halves every two stops is the **aperture diameter**, i.e. `1/F`.

The wide-open row is 8 % below the line. Solving `value[0] × F = 0.0998` for each state gives
F = 4.85, 5.56, 7.87, 11.15, 15.78, 22.28 — the marked f/4.5 behaves as ~f/4.9. Whether that is a
T-stop-style correction or the wide-open row also carrying mechanical vignetting is **UNKNOWN**.

## 6.2 `value[0]` is the on-axis value, and `value[0] × F` is a protocol constant — CERTAIN

Each row below is one named lens's slot-B type-1 row, taken **wide open and at the infinity end of
the focus range** — the state in which the working aperture is known to equal the marked f-number.

| Lens | State | F | Row | E | Mantissas | Decoded value[0..4] | **`value[0]×F`** | `v4/v0` | Falloff |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Voigtländer 15/4.5 | f/5.6, ∞ | 5.6 | `84 96 c1 ba bb cc` | 16 | 1174 1111 1041 972 920 | 0.01791 0.01695 0.01588 0.01483 0.01404 | **0.1003** | 0.784 | 0.70 EV |
| Voigtländer 15/4.5 | f/8, ∞ | 8.0 | `96 7b a7 9e 9e b6` | 17 | 1659 1570 1472 1374 1300 | 0.01266 0.01198 0.01123 0.01048 0.00992 | **0.1013** | 0.784 | 0.70 EV |
| Voigtländer 15/4.5 | f/16, ∞ | 16.0 | `a6 77 a9 9e 9e b5` | 18 | 1655 1568 1470 1372 1297 | 0.00631 0.00598 0.00561 0.00523 0.00495 | **0.1010** | 0.784 | 0.70 EV |
| Zeiss Loxia 21/2.8 | Wide open | 2.8 | `f4 a8 be ba c9 da` | 15 | 1192 1126 1056 1001 963 | 0.03638 0.03436 0.03223 0.03055 0.02939 | **0.1019** | 0.808 | 0.62 EV |
| Sony SEL5518Z 55/1.8 | First frame of init | 1.8 | `d1 d0 d5 e8 e9 ef` | 13 | 464 421 397 374 357 | 0.05664 0.05139 0.04846 0.04565 0.04358 | **0.1020** | 0.769 | 0.76 EV |
| Sony SEL2870 @70 mm | Settled | 5.6 | `84 b1 b9 da e4 ec` | 16 | 1201 1130 1092 1064 1044 | 0.01833 0.01724 0.01666 0.01624 0.01593 | **0.1026** | 0.869 | 0.40 EV |
| Sony SELP1650 @16 mm | Init | 3.5 | `e1 c9 18 9f de f6` | 14 | 457 481 384 350 340 | 0.02789 0.02936 0.02344 0.02136 0.02075 | **0.0976** | 0.744 | 0.85 EV |
| Sony SELP1650 @16 mm | Settled | 3.5 | `e1 b9 12 a6 e1 f6` | 14 | 441 459 369 338 328 | 0.02692 0.02802 0.02252 0.02063 0.02002 | **0.0942** | 0.744 | 0.85 EV |
| Yongnuo 35/1.8 DA | Bucket 0 (∞) | 1.8 | `d1 d8 d7 b6 be d0` | 13 | 472 431 357 291 243 | 0.05762 0.05261 0.04358 0.03552 0.02966 | **0.1037** | 0.515 | 1.92 EV |
| Yongnuo 50/1.8 DF | Bucket 0 (∞) | 1.8 | `e3 5d b3 be c6 d0` | 14 | 861 784 718 660 612 | 0.05255 0.04785 0.04382 0.04028 0.03735 | **0.0946** | 0.711 | 0.98 EV |

The SELP1650's `value[1] > value[0]` is the one non-monotone row in the set — its second mantissa
rises 457 → 481 before falling. It does so in both its init and its settled row, so it is a property
of the lens's data, not an observation glitch. Unexplained.

Seven optically unrelated lenses — 15 mm to 70 mm, f/1.8 to f/16, four manufacturers, APS-C and
full-frame, two camera bodies — agree on `value[0] × F ≈ 0.100` to within ±6 %, and to within ±3 %
if the SELP1650's two states are averaged.

**That is only possible if point 0 is the on-axis value**, where no lens has vignetting. A 15 mm
ultra-wide and a 55 mm normal cannot agree at any off-axis height.

```
value[0] = C / F                        C ≈ 0.0998 under the §3.3 anchor
value[i] / value[0] = the relative pupil transmission at field height i
```

### Rows that do NOT land on the constant, and why

| Lens | State | Marked F | Row | E | value[0] | `value[0]×F` | Implied F |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Sony SEL5518Z | Settled after init sweep | 1.8 | `f5 50 e3 f5 f4 f7` | 15 | 0.04150 | 0.0747 | 2.40 |
| Sony SEL55210 @55 mm | Settled | 4.5 | `e1 1c ba d1 ea f6` | 14 | 0.01733 | 0.0780 | 5.76 |
| Sony SEL55210 @210 mm | Settled | 6.3 | `83 73 98 b1 bb c3` | 16 | 0.01347 | 0.0849 | 7.41 |
| Yongnuo 50/1.8 DA | Bucket 0 | 1.8 | `d1 49 a6 cd e6 e9` | 13 | 0.04016 | 0.0723 | 2.48 |

None of these four contradicts §6.2; all four are states where the working aperture is **not known**
to be the marked maximum:

- The SEL5518Z row is the same lens as the fifth row of the table above, after its power-on focus
  sweep has moved the focus group — see §6.3.
- The two SEL55210 rows come from init observations in which the body's set aperture is not
  recorded. Both are ⅔ stop slow against the marked maximum, i.e. a *constant* offset across two
  zoom positions with different maximum apertures — which is what a body-set aperture looks like,
  not what a per-lens calibration error looks like.
- The Yongnuo 50 DA's table is poorly behaved end to end (§9).

**The rule for anyone harvesting donor rows: take the infinity bucket.**

## 6.3 The four remaining points are increasing field heights — PROBABLE

The falloff column in §6.2 is `−2·log₂(value[4]/value[0])`, i.e. the vignetting in stops if the row
is a *diameter* and illumination goes as diameter². The numbers (0.4 – 1.0 EV, and 1.9 EV for one
fast APS-C prime) are the right size for real lenses.

Two behavioural confirmations from the aperture sweep:

- The **shape** goes from 0.698 at f/4.5 to **0.784 at f/5.6 and then never changes again** (0.784,
  0.784, 0.784, 0.783 for f/8…f/22). Mechanical vignetting disappearing after one stop and leaving
  only the aperture-independent natural falloff is textbook behaviour.
- Slot A, over the same five stops, does not move at all.

What the four heights are in absolute terms is **UNKNOWN**. The measured corner falloff is smaller
than the true corner vignetting of, e.g., the Loxia 21/2.8, so point 4 is probably **not** the
extreme corner.

## 6.4 The focus axis is NOT the effective aperture

Holding the aperture fixed and moving the focus group moves this row too, always in the same
direction: **`value[0]` falls and the profile flattens.**

| Lens | Sample | Row | E | Decoded value[0..4] | `value[0]` | Shape `v4/v0` |
| --- | --- | --- | --- | --- | --- | --- |
| Sony SEL5518Z | Init, frame 1 | `d1 d0 d5 e8 e9 ef` | 13 | 0.05664 0.05139 0.04846 0.04565 0.04358 | 0.05664 | 0.769 |
| Sony SEL5518Z | Init, frame 5 | `e3 0e d4 e8 ea f0` | 14 | 0.04773 0.04504 0.04358 0.04224 0.04126 | 0.04773 | 0.864 |
| Sony SEL5518Z | Init, frame 7 → end | `f5 50 e3 f5 f4 f7` | 15 | 0.04150 0.04062 0.04028 0.03992 0.03964 | 0.04150 | 0.955 |
| Voigtländer 15/4.5 @f4.5 | Ring at ∞ | `85 42 97 86 97 b6` | 16 | 0.02054 0.01894 0.01707 0.01547 0.01434 | 0.02054 | 0.698 |
| Voigtländer 15/4.5 @f4.5 | Ring at 30 cm | `85 4b a2 93 a7 bc` | 16 | 0.02068 0.01924 0.01758 0.01622 0.01518 | 0.02068 | 0.734 |
| Yongnuo 50/1.8 DF | Bucket 0 (∞) | `e3 5d b3 be c6 d0` | 14 | 0.05255 0.04785 0.04382 0.04028 0.03735 | 0.05255 | 0.711 |
| Yongnuo 50/1.8 DF | Bucket 2 | `e3 09 cc d2 d9 e0` | 14 | 0.04742 0.04425 0.04144 0.03906 0.03711 | 0.04742 | 0.782 |
| Yongnuo 50/1.8 DF | Bucket 4 | `e2 a9 e8 eb ee f1` | 14 | 0.04156 0.04010 0.03882 0.03772 0.03680 | 0.04156 | 0.885 |
| Yongnuo 35/1.8 DA | Bucket 0 (∞) | `d1 d8 d7 b6 be d0` | 13 | 0.05762 0.05261 0.04358 0.03552 0.02966 | 0.05762 | 0.515 |
| Yongnuo 35/1.8 DA | Bucket 2 | `e3 82 c1 8d a4 c3` | 14 | 0.05481 0.05096 0.04395 0.03833 0.03461 | 0.05481 | 0.631 |
| Yongnuo 35/1.8 DA | Bucket 4 | `e3 01 d2 a8 b4 f1` | 14 | 0.04694 0.04413 0.03876 0.03412 0.03320 | 0.04694 | 0.707 |

A live Sony lens and two tables from a different manufacturer produce the **same two trends**, so
the effect is in the protocol's data model, not in one implementation.

### The magnitude refutes the working-aperture reading

Decoding the full tables per focus bucket rather than sampling three frames. Grid index → distance
uses 806 grid units per dioptre.

| Bucket | Distance | YN50 DF `value[0]` | Implied `F` | **Actual working `F`** | YN35 DA `value[0]` | Implied `F` | **Actual** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | ∞ | 0.05255 | 1.90 | 1.80 | 0.05762 | 1.73 | 1.80 |
| 2 | 4.73 m | 0.04742 | 2.10 | 1.82 | 0.05481 | 1.82 | 1.81 |
| 4 | 2.36 m | 0.04156 | 2.40 | 1.84 | 0.04694 | 2.13 | 1.83 |
| 6 | 1.58 m | 0.03561 | 2.80 | 1.86 | 0.03833 | 2.60 | 1.84 |
| 8 | 1.18 m | 0.02939 | 3.40 | 1.88 | 0.03137 | 3.18 | 1.85 |
| 11 | 0.86 m | 0.02000 | 4.99 | 1.91 | — | — | — |

Both lenses are f/1.8. Over ∞ → 1.18 m the observed fall is **0.84 EV** (YN50 DF) and **0.88 EV**
(YN35 DA). `F_eff = F·(1+m)` over the same interval is **0.062 EV** and **0.043 EV**. The data is
**13× and 20× too fast**, on two lenses from one vendor built on two separate implementations, and
the same shape appears in a live Sony observation. This is not a calibration offset; it is the wrong
quantity.

**A second, independent refutation — the profile collapses.** `v4/v0` on the YN50 DF runs
`0.711 → 0.739 → 0.782 → 0.830 → 0.885 → 0.949 → 0.999` and then stays at 0.999 for every remaining
bucket. At close focus the five points become **identical**. A quantity sampled across field height
does not become uniform across the field when you focus closer — mechanical vignetting, if anything,
gets worse.

### What survives

| Claim | Evidence | Status |
| --- | --- | --- |
| Encoding (block float, start + 4 deltas) | Every row from every device resolves smoothly | **CERTAIN** |
| `∝ 1/F` on the aperture axis | Voigtländer, focus locked, 5 stops, ×0.7068 ± 0.0010 per stop vs `1/√2` = 0.70711 | **CERTAIN** |
| `value[0] × F ≈ 0.0998` | 7 lenses, 15–70 mm, 3 makers — but **all sampled at the infinity end** | **CERTAIN, at infinity only** |
| The 4 remaining points are field heights | §6.3 | **Doubtful away from infinity** — they become identical at close focus |
| It is the effective aperture at every focus state | — | **REFUTED** |

### The focus axis is vendor-authored, not optical — PROBABLE

The YN50 DF's decoded `value[0]` is piecewise **linear in the bucket index** end to end, with the
slope stepping down twice — −0.00296/bucket (buckets 1–11), −0.00165 (12–17), −0.00089 (18–22) —
each segment ≈ half the previous. The two breaks fall **mid-octave** (inside E16 and inside E17), so
this is in the data, not an artefact of exponent anchoring. A measured optical curve does not halve
its slope twice at round table indices; a hand-authored ramp does.

Second, the focus axis is **inconsistent across vendors in a way no single physical quantity
allows**:

| Vendor / lens | Predicted by `F·(1+m)` | Observed |
| --- | --- | --- |
| Voigtländer 15/4.5, ∞→0.3 m | Value falls ~5 % | **Rises 0.7 %** |
| Sony SEL5518Z, init sweep | Falls ×0.88 | Falls ×0.73 — 2.4× too fast in EV |
| Yongnuo tables | Falls ×0.96 | 13–20× too fast |

Three vendors, three different answers on the focus axis — while all three sit on the same
`1/√2`-per-stop line on the aperture axis, which stays exact everywhere it can be tested. **The
focus axis does not encode a physical optical quantity; it is the vendor's own scheduling.**

One unexplained regularity, recorded without interpretation: holding the exponent fixed, the YN50 DF
mantissa falls by exactly **48 per bucket** (861, 825, 777, 729, 681, 633 …) — linear in the focus
grid index, not exponential. Extrapolating to zero lands at ≈ 0.50 m; the YN35 DA's slope
extrapolates to ≈ 0.54 m. Two lenses of different focal length pointing at the same distance may
mean something or may be two points on a coincidence. Neither table actually reaches zero — both
flatten.

## 6.5 Sample data — one lens across its whole focus range

### Yongnuo YN35mm F1.8S DA — 21 focus buckets

| k | Steps | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0–45 | `d1 d8 d7 b6 be d0` | 13 | 0.05762 | 0.05261 | 0.04358 | 0.03552 | 0.02966 |
| 1 | 46–89 | `e3 94 ba 81 95 b6` | 14 | 0.05591 | 0.05164 | 0.04388 | 0.03735 | 0.03284 |
| 2 | 90–121 | `e3 82 c1 8d a4 c3` | 14 | 0.05481 | 0.05096 | 0.04395 | 0.03833 | 0.03461 |
| 3 | 122–165 | `e3 41 cb 9d aa de` | 14 | 0.05084 | 0.04761 | 0.04156 | 0.03632 | 0.03424 |
| 4 | 166–213 | `e3 01 d2 a8 b4 f1` | 14 | 0.04694 | 0.04413 | 0.03876 | 0.03412 | 0.03320 |
| 5 | 214–261 | `e2 ba d4 a9 c8 f5` | 14 | 0.04260 | 0.03992 | 0.03461 | 0.03119 | 0.03052 |
| 6 | 262–296 | `e2 74 d5 aa dd f7` | 14 | 0.03833 | 0.03571 | 0.03046 | 0.02832 | 0.02777 |
| 7 | 297–328 | `e2 34 d5 ac f2 f8` | 14 | 0.03442 | 0.03180 | 0.02667 | 0.02582 | 0.02533 |
| 8 | 329–360 | `e2 02 d6 bb f4 f9` | 14 | 0.03137 | 0.02881 | 0.02460 | 0.02386 | 0.02344 |
| 9 | 361–392 | `f3 9a ae 98 ec f4` | 15 | 0.02814 | 0.02563 | 0.02246 | 0.02185 | 0.02148 |
| 10 | 393–423 | `f3 44 b0 b5 ee f5` | 15 | 0.02551 | 0.02307 | 0.02078 | 0.02023 | 0.01990 |
| 11 | 424–455 | `f2 fc b6 c8 f0 f6` | 15 | 0.02332 | 0.02106 | 0.01935 | 0.01886 | 0.01855 |
| 12 | 456–487 | `f2 b4 be da f2 f7` | 15 | 0.02112 | 0.01910 | 0.01794 | 0.01752 | 0.01724 |
| 13 | 488–519 | `84 cd 8d d9 e8 f1` | 16 | 0.01875 | 0.01700 | 0.01640 | 0.01604 | 0.01581 |
| 14 | 520–551 | `84 63 9f de ea f3` | 16 | 0.01714 | 0.01566 | 0.01514 | 0.01480 | 0.01460 |
| 15 | 552–583 | `83 f9 b2 e1 ed f4` | 16 | 0.01552 | 0.01433 | 0.01385 | 0.01357 | 0.01338 |
| 16 | 584–615 | `97 2a 86 ca de ea` | 17 | 0.01399 | 0.01306 | 0.01265 | 0.01239 | 0.01222 |
| 17 | 616–658 | `95 a8 cc d6 e6 f0` | 17 | 0.01105 | 0.01065 | 0.01033 | 0.01013 | 0.01001 |
| 18 | 659–695 | `83 56 c4 8f 9a b0` | 16 | 0.01303 | 0.01212 | 0.01039 | 0.00883 | 0.00761 |
| 19 | 696–704 | `e1 7c ca 96 9b b0` | 14 | 0.02319 | 0.01990 | 0.01343 | 0.00726 | 0.00238 |
| 20 | — (spare) | `d1 11 d2 a3 a8 ba` | 13 | 0.03333 | 0.02771 | 0.01636 | 0.00562 | −0.00293 |

**This table breaks at bucket 18.** `value[0]` runs monotone down to bucket 17 and then climbs back
up — 0.0110, 0.0130, 0.0232 — while the lens keeps focusing closer, with the shape ratio falling
0.906 → 0.584 → 0.103.

The break coincides with the nibble sequence stopping. Buckets 0…17 walk
`d e e e e e e e e f f f f 8 8 8 9 9` — a clean §3.3 progression, one octave per step, mantissa
falling inside each nibble and jumping at each step. Buckets 18, 19 and 20 then read `8`, `e`, `d`,
going *backwards*. **No exponent assignment makes that a continuation**, so either the last two
reachable buckets are miscalibrated or they encode something this model does not cover. UNKNOWN —
and one more reason to harvest from the infinity end.

### Yongnuo YN50mm F1.8S DF — 23 focus buckets

| k | Steps | Row | E | value[0] | [1] | [2] | [3] | [4] |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 0 | 0–45 | `e3 5d b3 be c6 d0` | 14 | 0.05255 | 0.04785 | 0.04382 | 0.04028 | 0.03735 |
| 1 | 46–101 | `e3 39 be c6 ce d7` | 14 | 0.05035 | 0.04633 | 0.04279 | 0.03973 | 0.03723 |
| 2 | 102–149 | `e3 09 cc d2 d9 e0` | 14 | 0.04742 | 0.04425 | 0.04144 | 0.03906 | 0.03711 |
| 3 | 150–185 | `e2 d9 da df e3 e8` | 14 | 0.04449 | 0.04218 | 0.04016 | 0.03839 | 0.03693 |
| 4 | 186–216 | `e2 a9 e8 eb ee f1` | 14 | 0.04156 | 0.04010 | 0.03882 | 0.03772 | 0.03680 |
| 5 | 217–261 | `e2 79 f6 f7 f9 fa` | 14 | 0.03864 | 0.03802 | 0.03748 | 0.03705 | 0.03668 |
| 6 | 262–296 | `f4 8f ff ff 00 01` | 15 | 0.03561 | 0.03558 | 0.03555 | 0.03555 | 0.03558 |
| 7 | 297–340 | `f4 29 ff 00 00 00` | 15 | 0.03250 | 0.03247 | 0.03247 | 0.03247 | 0.03247 |
| 8 | 341–376 | `f3 c3 ff 00 00 00` | 15 | 0.02939 | 0.02936 | 0.02936 | 0.02936 | 0.02936 |
| 9 | 377–420 | `86 b8 fe 00 ff 01` | 16 | 0.02625 | 0.02621 | 0.02621 | 0.02620 | 0.02621 |
| 10 | 421–455 | `85 e7 fe 00 ff 01` | 16 | 0.02306 | 0.02303 | 0.02303 | 0.02301 | 0.02303 |
| 11 | 456–499 | `85 1f fe 00 ff 01` | 16 | 0.02000 | 0.01997 | 0.01997 | 0.01996 | 0.01997 |
| 12 | 500–547 | `84 a4 fe 00 ff 01` | 16 | 0.01813 | 0.01810 | 0.01810 | 0.01808 | 0.01810 |
| 13 | 548–595 | `84 35 fe 00 ff 01` | 16 | 0.01643 | 0.01640 | 0.01640 | 0.01639 | 0.01640 |
| 14 | 596–631 | `97 90 fc 00 fe 02` | 17 | 0.01477 | 0.01474 | 0.01474 | 0.01472 | 0.01474 |
| 15 | 632–651 | `96 b8 fe fe 00 00` | 17 | 0.01312 | 0.01311 | 0.01309 | 0.01309 | 0.01309 |
| 16 | 652–679 | `95 de fe fe 00 00` | 17 | 0.01146 | 0.01144 | 0.01143 | 0.01143 | 0.01143 |
| 17 | 680–699 | `95 30 fe fe 00 00` | 17 | 0.01013 | 0.01012 | 0.01010 | 0.01010 | 0.01010 |
| 18 | 700–727 | `94 bc 00 fe 00 00` | 17 | 0.00925 | 0.00925 | 0.00923 | 0.00923 | 0.00923 |
| 19 | 728–747 | `94 46 fe 00 00 00` | 17 | 0.00835 | 0.00833 | 0.00833 | 0.00833 | 0.00833 |
| 20 | 748–763 | `a7 a4 fc 00 00 00` | 18 | 0.00746 | 0.00745 | 0.00745 | 0.00745 | 0.00745 |
| 21 | 764–796 | `a6 bc 00 00 00 00` | 18 | 0.00658 | 0.00658 | 0.00658 | 0.00658 | 0.00658 |
| 22 | 797–808 | `a5 d0 00 00 00 00` | 18 | 0.00568 | 0.00568 | 0.00568 | 0.00568 | 0.00568 |

Monotone across all 23 buckets, 0.0526 → 0.0057 — a factor of 9 — with the shape ratio flattening
0.711 → 1.000 throughout. This is the clean example of the focus-axis behaviour §6.4 leaves
uninterpreted. Its nibble sequence walks `e e e e e e f f f 8 8 8 8 8 9 9 9 9 a a` straight through
the wrap point §3.3 exists to explain.

---

# 7. The null rows adapters send

| Device | Slot A | Slot B |
| --- | --- | --- |
| TECHART LM-EA9 | `00 00 00 00 00 00` | `26 00 00 00 00 00` |
| Viltrox EF-NEX II (both Canon lenses) | `00 00 00 00 00 00` | `26 00 00 00 00 00` |

Decoded, `26 00 00 00 00 00` is a **type-0 row, exponent field 2, flat at `0x600` = 1536** — exactly
**1.500** under the §3.3 anchor. **It is a neutral default value, not a null.**

Slot A's all-zero row decodes to a flat zero curve, which is a *different* kind of statement: for
slot A the adapters send the value zero, for slot B they send a plausible constant.

Under the preferred reading the defaults acquire a concrete meaning:

- **Slot A = 0** says "no field grid — nothing is sampled".
- **Slot B type 0 = flat 1.5** says "**exit pupil at 1.5 × focal length**". For the LM-EA9's claimed
  40 mm that is a fixed 60 mm exit-pupil distance announced to the body regardless of which M-mount
  lens is actually mounted. Since typical M-mount glass has a much shorter exit-pupil distance, this
  is exactly the wrong-pupil-distance input to the body's pupil-shift correction.

---

# 8. Slots C and D

Same 6-byte encoding — **CERTAIN**, the decoder produces smooth curves — but different content.

**Slot C is a strict lookup on the row index `pl[77]`.** On the Sony SEL2870, across all 109 frames
without exception. Anchor used here: nibble `d` → E 13, `e` → E 14, `f` → E 15.

| `pl[77]` | Slot C row | E | Mantissas | Decoded value[0..4] |
| --- | --- | --- | --- | --- |
| 0x15 | `e0 70 2b 1f 15 0d` | 14 | 112 155 186 207 220 | 0.00684 0.00946 0.01135 0.01263 0.01343 |
| 0x16 | `f1 27 dc e0 d2 17` | 15 | 295 259 227 181 204 | 0.00900 0.00790 0.00693 0.00552 0.00623 |
| 0x17 | `ff 59 2e 21 43 45` | 15 | 3929 3975 4008 4075 4144 | 0.11990 0.12131 0.12231 0.12436 0.12646 |

Row `0x17`'s mantissa 3929 sits near the 12-bit ceiling of 4095 — slot C is the only place a row is
observed running that close to overflow, and its exponent anchor is correspondingly the least
certain here.

So slot C is a **multiplexed table transfer**: a larger table streamed a few rows per frame, tagged
by `pl[77]`. Distinct non-zero rows seen: 7 on the SEL2870, 19 on the SEL5518Z. Slot D round-robins
rows also seen in A/B/C.

Semantics of C and D: **UNKNOWN**. Their exponent windows (§3.3) differ from A's and B's, so they
are different quantities again.

**This richness is not universal.** On the Yongnuo YN35mm F1.8S DA the equivalent selector does not
look at focus position at all: it is a bare `idx == 0x17` switch between two fixed 25-byte records,
and both records are entirely zero. So on that lens slot C/D carries no information regardless of
what the index bytes say. The SEL2870 data above stays CERTAIN as a Sony fact, but "slot C is a rich
per-index table" does not generalise to every E-mount lens.

---

# 9. Practical notes for implementers

## Which parity holds which type is NOT fixed

Every one of the 44 buckets across the two complete tables carries **exactly one type-1 row and one
type-0 row**. So the alternation is not redundancy or dithering — the lens is multiplexing two
quantities down one field, one per frame.

But in six buckets the two are swapped:

| Lens | Swapped buckets | Parity 0 there |
| --- | --- | --- |
| Yongnuo 35 DA | 9, 10, 11 | `24 d8 08 10 08 00` — type 0 |
| Yongnuo 50 DF | 16, 17, 22 | `24 8c f8 …` — type 0 |

Messages 0x28 and 0x35 always request parity 0. **In those six buckets they therefore deliver the
type-0 row and no aperture profile at all.** Whether that is deliberate (those buckets have nothing
useful to say about aperture) or a table-authoring slip is UNKNOWN — either way, a decoder that
assumes "parity 0 = type 1" will silently pick up the wrong quantity there. **Always test bit 7.**

Bucket 0 is type 1 on both lenses.

## One vendor's table drifts far out of band

The Yongnuo 35 DA and 50 DF are exemplary at their **first** table entry — `value[0] × F` = 0.1037
and 0.0946, right on the seven-lens constant of §6.2. At the far end of the same two tables slot B
falls to `value[0] × F ≈ 0.010`, a tenfold departure, with the field profile going perfectly flat
(shape 0.998–1.000) and the 50 DF's slot A dropping 4× (0.1196 → 0.0303). The **Yongnuo 50 DA** is
worse still: it never reaches the constant at all, starting at 0.0723, and its table interleaves
several sub-tables rather than running monotonically.

No Sony, Zeiss or Voigtländer row measured moves anything like that far over any axis. That is a
statement about one vendor's calibration, not about the protocol. It matters here only as a warning:
**when harvesting donor rows, take the first (infinity) entry.**

---

# 10. Where the samples come from

Every row quoted in this document comes from one of these. Nothing here is synthesised.

| Lens | Body | What varied | Msg-0x05 frames |
| --- | --- | --- | --- |
| Voigtländer Super Wide-Heliar 15/4.5 (manual focus) | Sony NEX-7 | Aperture ring only | 461 |
| Voigtländer Super Wide-Heliar 15/4.5 | Sony NEX-7 | Focus ring only, f/4.5 | 536 |
| Zeiss Loxia 21/2.8 (manual focus) | Sony NEX-7 | Init | 64 + 75 |
| Sony SEL5518Z (Sonnar T* FE 55/1.8 ZA) | Sony NEX-7 | Power-on, includes the init focus sweep | 281 |
| Sony SEL2870 @70 mm | Sony NEX-7 | Init | 109 |
| Sony SELP1650 @16 mm | Sony A6000 | Init | 216 |
| Sony SEL55210 @55 mm | Sony A6000 | Init | 76 |
| Sony SEL55210 @210 mm | Sony A6000 | Init | 136 |
| Viltrox EF-NEX II + Canon EF 50/1.8 | Sony A6000 | Init | 205 |
| Viltrox EF-NEX II + Canon EF-S 24/2.8 | Sony A6000 | Init | 151 |
| TECHART LM-EA9 | — | Its fixed rows, never rewritten at runtime | — |
| Yongnuo YN35mm F1.8S DA DSM WL | — | Complete 21-bucket table | — |
| Yongnuo 50mm F1.8S DF | — | Complete 23-bucket table | — |
| Yongnuo 50mm F1.8S DA | — | Complete table | — |

---

# Open questions

| Question | Status |
| --- | --- |
| **The focus axis of slot A and slot B type 1** — the framework's one hole | **UNKNOWN.** Both fall far faster with close focus than pupil geometry allows (§4.4, §6.4), the vendors disagree with each other, and the piecewise-linear slope-halving structure looks authored, not measured. Until modelled, only infinity-end rows are trustworthy pupil data. |
| The SEL55210 counter-example | Its implied outermost grid sample lands beyond the corner in both zoom states (§4.1) — either those samples are not infinity-state (where `EPD = p·f` stops holding), or the grid reading fails on this zoom. Needs a clean infinity-focus observation. |
| Slot B point 0: on-axis (§6.2's cross-lens constant) vs sampled at grid position A[0] | Unresolved tension inside the preferred reading — either the grid applies only to the outer points, or the ±3 % constant tolerates the small vignetting at the innermost height. |
| Whether type 0 is `p` itself or a monotone relative of it | Open (§5); does not change the framework. |
| Absolute units — the exponent-anchor octave | **UNKNOWN.** All ratios are anchor-independent; only the absolute scale is affected. |
| How a body resolves the 3-bit exponent wrap | **UNKNOWN** (§3.3). A log-mantissa encoding that would have removed the wrap is tested and eliminated (§3.6). |
| Slots C and D | **UNKNOWN** (§8). |
| Why the YN35 DA's table reverses at bucket 18 | **UNKNOWN** (§6.5). |
| Why the SELP1650's type-1 row is non-monotone at point 1 | **UNKNOWN** (§6.2). |
