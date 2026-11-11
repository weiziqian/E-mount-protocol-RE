# The 6-byte optical rows

The lens describes its optics to the body as **6-byte rows**. [Message 0x05](msg_0x05.md) carries
four of them per frame (slots A–D), and [message 0x28](msg_0x28.md) and [message 0x35](msg_0x35.md)
carry slots A and B again. This document gives the encoding, **how an A7 IV decodes it**, what each
slot carries, and the decoded data of every lens observed.

Confidence labels are the repository's: **CERTAIN / PROBABLE / POSSIBLE / UNKNOWN**. `pl[n]` is
payload byte `n`.

"Body" below means the A7 IV application firmware 6.02 (`cpapp-b.bin`). Its decoder is
`0x98d09e`; the native-lens consumer is `0x3ab854`.

---

# 1. Summary

| Slot | Body row number | Carries | Status |
| --- | --- | --- | --- |
| **A** | 2 (fixed) | **Shading (vignetting) curve**: 0 on the axis, rising with image height, shrinking as the aperture closes | Decoding **CERTAIN**; meaning **PROBABLE** (§4) |
| **B, type 0** | 3 (fixed) | A nearly flat curve the body stores as its **reciprocal**; orders lenses exactly as pupil magnification (exit-pupil distance ÷ focal length) does | Decoding **CERTAIN**; meaning **PROBABLE** (§5) |
| **B, type 1** | — | Scales as `1/F`. **The body does not decode it** | §5.2 |
| **C, D** | `pl[77]`, `pl[78]` | Rows of a larger table, streamed by row number. Row 0x15 behaves as **distortion**, rows 0x16/0x17 possibly as **lateral colour** | Decoding **CERTAIN**; meanings **POSSIBLE** (§6) |

Shading, distortion and chromatic aberration are the three in-camera lens corrections Sony bodies
offer. That the slots line up with them is the working reading of this document, not a traced
fact: the consumers of the decoded tables run on the main processor and are not traced.

---

# 2. Where the rows appear and how they are chosen

| Message | Slot A | Slot B | Slot C | Slot D |
| --- | --- | --- | --- | --- |
| [0x05](msg_0x05.md) | `pl[32..37]` | `pl[38..43]` | `pl[83..88]`, row `pl[77]` | `pl[89..94]`, row `pl[78]` |
| [0x28](msg_0x28.md) | `pl[11..16]` | `pl[17..22]` | — | — |
| [0x35](msg_0x35.md) | `pl[11..16]` | `pl[20..25]` | — | — |

- **Slots A and B come from one record per table entry**, and the entry is selected by the
  [aperture value](aperture_value.md) the lens reports. CERTAIN on Yongnuo firmware.
- Each entry is a **pair**: one slot A and two slot Bs, one of each type. Message 0x05 alternates the
  halves frame by frame, so slot A holds still while slot B alternates. Messages 0x28 and 0x35 carry
  the parity-0 half (§8).
- **Slots C and D carry rows by number.** The body asks for them in [message 0x03](msg_0x03.md)
  (`2F rowC rowD`), and the lens echoes the numbers in `pl[77..78]`. In normal running the body
  cycles 0x15/0x16/0x17; during init lenses send other numbers once (§6.3).
- The 117-byte long form of 0x05 adds 3 bytes per row: `pl[96..98]` for slot A, `pl[102..104]` for
  slot C, `pl[105..107]` for slot D (§3.4).

---

# 3. How the body decodes a row — CERTAIN

Bytes `b0 b1 b2 b3 b4 b5`.

```
v0  = ((b0 & 0x0f) << 8) | b1           signed 12-bit: bit 11 set -> v0 - 4096
e   = b0 >> 4                           signed 4-bit: 8..f -> -8..-1

P[10] = v0                              the OUTERMOST point
P[8]  = P[10] - s8(b2)                  each delta is SUBTRACTED, moving inward
P[6]  = P[8]  - s8(b3)
P[4]  = P[6]  - s8(b4)
P[2]  = P[4]  - s8(b5)
P[0]  = by row number, see 3.2           the axis
odd P = 4-point cubic interpolation      e.g. P[k] = (9(P[k-1]+P[k+1]) - P[k-3] - P[k+3] + 8) >> 4
P[11..15] = P[10]                        6-byte form: flat beyond the last point

value[i] = P[i] >> e   if e >= 0
           P[i] << -e  if e <  0
```

The result is a **16-point curve**; the six sent points sit at indices 0, 2, 4, 6, 8, 10.

## 3.1 Slot-specific rules

- **Slot B** is decoded only when **bit 7 of `b0` is clear** (type 0). Bit 6 is then the sign of a
  3-bit exponent (the body sets bit 7 from it before decoding). A type-1 row is skipped and the
  previous tables stand.
- **Slot D** is decoded only when `pl[78] != pl[77]`.
- **Rows 0 and 1 are dropped**, so a slot C/D row labelled 0 is never filed. Rows 2–0x18 are filed
  under their number; rows 0x15–0x18 also set a per-row "received" flag.

## 3.2 The axis point

| Row number | `P[0]` |
| --- | --- |
| 2 (slot A) | 0, and every negative value is clamped to 0 |
| 3 (slot B), 4–6, 0x15–0x18 | quadratic extrapolation: `P[0] = P[6] + 3·(P[2] − P[4])` |
| all others (7–0x14) | the five points move one place inward: `P[0]`=innermost, `P[2]`, `P[4] = (P[2]+P[6])/2`, `P[6]`, `P[8]`, `P[10]`=v0 |

## 3.3 Image height — PROBABLE

With slot B the body builds a fixed sample grid: index `i` × `0x58c` (1420), or × `0x5a2` (1442)
when a body setting is 2. `15 × 1442 = 21630` is the full-frame half-diagonal in µm, so the grid is
read as **image height in µm**:

| Index | 0 | 2 | 4 | 6 | 8 | 10 | 15 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Height | axis | 2.84 mm | 5.68 mm | 8.52 mm | 11.36 mm | **14.2 mm** | 21.3 mm |

So **v0 is the value at 14.2 mm** — about the APS-C corner — and a 6-byte row stops there.

## 3.4 Long form — CERTAIN

With the 9-byte form (`0x98d4fc`) the three extra bytes `x0 x1 x2` extend the curve **outward**,
by addition: `P[12] = v0 + s8(x2)`, `P[14] = P[12] + s8(x1)`, `P[15] = P[14] + s8(x0)`, odd points
interpolated. That reaches the full-frame corner.

## 3.5 What the body stores for slot B

Two reciprocal tables: `0x4000 / value[i]` (`0x14ac106`) and `16384000 · 2^e / P[i]`
(`0x14074d4`). A zero value is replaced by the numerator, so there is no division by zero.

## 3.6 Reference decoder

```python
def s8(b): return b - 256 if b > 127 else b

def decode(r, row):
    """r = 6 bytes, row = body row number (slot A 2, slot B 3, C/D pl[77]/pl[78]).
    Returns the body's 16-point curve. For slot B pass only type-0 rows."""
    if row == 3 and r[0] & 0x40: r = bytes([r[0] | 0x80]) + r[1:]
    mode = 1 if row in (3, 4, 5, 6, 0x15, 0x16, 0x17, 0x18) else 0 if row == 2 else 2
    v0 = ((r[0] << 8) | r[1]) & 0xfff
    if v0 & 0x800: v0 -= 0x1000
    P = [0] * 16; P[10] = v0
    for i, b in zip((8, 6, 4, 2), r[2:]): P[i] = P[i + 2] - s8(b)
    if mode == 0: P[0] = 0
    elif mode == 1: P[0] = P[6] + 3 * (P[2] - P[4])
    else: P[0], P[2], P[4] = P[2], P[4], int((P[4] + P[6]) / 2)
    q = lambda x: (x + 8) >> 4
    P[1] = q(5*P[0] + 15*P[2] - 5*P[4] + P[6])
    P[9] = q(P[4] + 15*P[8] + 5*P[10] - 5*P[6])
    for k in (7, 5, 3): P[k] = q(9*(P[k+1] + P[k-1]) - P[k+3] - P[k-3])
    e = r[0] >> 4
    if r[0] & 0x80: e -= 16
    out = [max(x, 0) if mode == 0 else x for x in P[:11]]
    out = [x >> e if e >= 0 else x << -e for x in out]
    return out + [out[10]] * 5
```

## 3.7 Superseded readings

Earlier versions of this document read a row as `v0` **plus** the deltas, `v0` being the innermost
point, with bit 7 of `b0` a type flag and bits 6..4 a 3-bit exponent that wraps. The body does none
of that for slots A, C and D: the deltas are subtracted from the outermost point, and the whole
high nibble is one signed exponent (`8` is ×256, not half of `f`). Every decoded value in the
earlier tables except `v0` is therefore wrong, and the "slot A is a field-position grid" and
"focus axis" analyses built on them are withdrawn. The earlier regularity "slot A's last point is
≈ 2 × its first" is, in the body's reading, "slot A is ≈ 0 at the innermost sample".

---

# 4. Slot A — shading

## 4.1 What it is — PROBABLE

- **Zero on the axis** (forced by the body) and rising roughly with the square of image height —
  the shape of vignetting.
- **Largest on wide and fast lenses wide open**, smallest on the tele end.
- **Shrinks as the aperture closes, then stops changing** (§4.3) — mechanical vignetting
  disappearing within a stop or two and natural falloff remaining. That is textbook vignetting
  behaviour, and no other optical quantity does it.
- On an a9 II, changing slot A alone changes the in-camera shading compensation strongly.

Unit: **UNKNOWN**. The body integer is given as decoded.

## 4.2 Different lenses

| Lens | Aperture | Row | e | axis | 2.8 | 5.7 | 8.5 | 11.4 | 14.2 mm |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Sony SELP1650 @16 mm (APS-C) | f/3.5 | `a0 ea 70 49 1e 0f` | −6 | 0 | 256 | 1216 | 3136 | 7808 | **14976** |
| Yongnuo 35 DA (APS-C) | ≤ f/1.9 | `b1 78 55 5b 50 48` | −5 | 0 | 1536 | 3840 | 6400 | 9312 | 12032 |
| Yongnuo 50 DA | ≤ f/1.9 | `b1 30 7e 4b 32 29` | −5 | 0 | 384 | 1696 | 3296 | 5696 | 9728 |
| Sony SEL55210 @55 mm (APS-C) | f/4.7 | `a0 97 42 2d 15 0d` | −6 | 0 | 384 | 1216 | 2560 | 5440 | 9664 |
| Voigtländer 15/4.5 | f/4.5 | `b1 28 47 51 4b 38` | −5 | 0 | 416 | 2208 | 4608 | 7200 | 9472 |
| Sony SEL55210 @210 mm (APS-C) | f/6.7 | `b1 00 41 41 38 2f` | −5 | 0 | 736 | 2240 | 4032 | 6112 | 8192 |
| Zeiss Loxia 21/2.8 | f/2.8 | `b0 fe 45 47 39 27` | −5 | 0 | 576 | 1824 | 3648 | 5920 | 8128 |
| Yongnuo 50 DF | ≤ f/1.9 | `b0 f5 48 3f 3c 25` | −5 | 0 | 416 | 1600 | 3520 | 5536 | 7840 |
| Sony SEL5518Z | first frame | `b0 f4 46 46 3f 25` | −5 | 0 | 128 | 1312 | 3328 | 5568 | 7808 |
| Sony SEL2870 @70 mm | f/5.6 | `b0 7f 2f 1d 17 12` | −5 | 0 | 320 | 896 | 1632 | 2560 | 4064 |
| Sony SEL5518Z | settled after init | `b0 5a 27 18 0f 09` | −5 | 0 | 96 | 384 | 864 | 1632 | 2880 |
| Viltrox EF-NEX II (both lenses) | — | `00 00 00 00 00 00` | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

The SEL5518Z's drop after init accompanies its power-on iris sweep; the aperture it settled at is
not recorded.

## 4.3 Against the aperture

Voigtländer 15/4.5, focus at infinity, aperture ring closed click by click:

| f-stop | Row | 14.2 mm |
| --- | --- | --- |
| f/4.5 | `b1 28 47 51 4b 38` | 9472 |
| f/5.6 | `b1 01 39 46 3e 37` | 8224 |
| f/8 … f/22 | `b1 04 3e 42 40 36` … `b1 02 3f 42 3f 37` | 8256–8320 |

The complete Yongnuo tables (the aperture column is the range of
[aperture values](aperture_value.md) that selects the entry):

**Yongnuo YN35mm F1.8S DA (APS-C)**

| Aperture | Row | e | axis | 2.8 | 5.7 | 8.5 | 11.4 | 14.2 mm |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ≤ f/1.9 | `b1 78 55 5b 50 48` | −5 | 0 | 1536 | 3840 | 6400 | 9312 | 12032 |
| f/2.0 – f/2.2 | `b1 52 61 63 42 2e` | −5 | 0 | 960 | 2432 | 4544 | 7712 | 10816 |
| f/2.2 – f/2.5 | `b1 3a 69 68 39 1f` | −5 | 0 | 544 | 1536 | 3360 | 6688 | 10048 |
| f/2.5 – f/2.8 | `b1 22 71 6d 30 0e` | −5 | 0 | 192 | 640 | 2176 | 5664 | 9280 |
| f/2.8 – f/3.1 | `b1 17 7a 5f 28 0f` | −5 | 0 | 224 | 704 | 1984 | 5024 | 8928 |
| f/3.1 – f/3.5 | `a0 86 42 29 10 07` | −6 | 0 | 256 | 704 | 1728 | 4352 | 8576 |
| f/3.5 – f/17.4 | `a0 80 47 22 0c 07` | −6 | 0 | 256 | 704 | 1472 | 3648 | 8192 |

**Yongnuo YN50mm F1.8S DF (full frame)**

| Aperture | Row | e | axis | 2.8 | 5.7 | 8.5 | 11.4 | 14.2 mm |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ≤ f/1.9 | `b0 f5 48 3f 3c 25` | −5 | 0 | 416 | 1600 | 3520 | 5536 | 7840 |
| f/2.0 – f/2.2 | `b0 d0 46 33 30 1d` | −5 | 0 | 320 | 1248 | 2784 | 4416 | 6656 |
| f/2.2 – f/2.5 | `b0 9e 43 25 1e 12` | −5 | 0 | 192 | 768 | 1728 | 2912 | 5056 |
| f/2.5 – f/2.8 | `b0 6c 40 15 0d 07` | −5 | 0 | 96 | 320 | 736 | 1408 | 3456 |
| f/2.8 – f/3.1 | `b0 5b 30 14 0d 07` | −5 | 0 | 96 | 320 | 736 | 1376 | 2912 |
| f/3.1 – f/3.5 | `b0 4a 21 12 0c 08` | −5 | 0 | 96 | 352 | 736 | 1312 | 2368 |
| f/3.5 – f/3.9 | `b0 3e 16 10 0d 08` | −5 | 0 | 96 | 352 | 768 | 1280 | 1984 |
| f/3.9 – f/4.4 | `c0 7c 2b 22 19 11` | −4 | 0 | 80 | 352 | 752 | 1296 | 1984 |
| f/4.4 – f/4.9 | `c0 7c 2a 23 19 11` | −4 | 0 | 80 | 352 | 752 | 1312 | 1984 |
| f/4.9 – f/5.5 | `d0 f8 54 46 33 21` | −3 | 0 | 80 | 344 | 752 | 1312 | 1984 |
| f/5.5 – f/6.2 | `d0 f9 55 46 33 21` | −3 | 0 | 80 | 344 | 752 | 1312 | 1992 |
| f/6.2 – f/24.5 | `d0 fa 56 46 33 21` | −3 | 0 | 80 | 344 | 752 | 1312 | 2000 |

Both fall steeply over the first stops and then hold one value for the rest of the table. The
50 DF's exponent steps `b → c → d` with the curve unchanged — the same curve written at a coarser
mantissa, as the signed exponent predicts.

---

# 5. Slot B

## 5.1 Type 0 — PROBABLE: pupil magnification

Nearly flat across the field. The body integer ÷ 256 orders every lens observed exactly as pupil
magnification `p` (exit-pupil distance ÷ focal length) does — retrofocus wides above 1,
near-symmetric designs about 1, tele ends below 1 — and the body uses its reciprocal (§3.5), which
is what a chief-ray angle `h / EPD` needs. It moves about 1 % over a five-stop aperture sweep, so
it is pupil geometry, not the diaphragm.

| Lens | Row | e | axis | 2.8 | 5.7 | 8.5 | 11.4 | 14.2 mm | ÷ 256 at 14.2 mm | × f |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Sony SELP1650 @16 mm | `14 0a e0 fe 18 04` | 1 | 528 | 520 | 522 | 534 | 533 | 517 | 2.02 | 32 mm |
| Zeiss Loxia 21/2.8 | `27 26 20 1a ee e3` | 2 | 464 | 454 | 447 | 443 | 449 | 457 | 1.79 | 38 mm |
| Voigtländer 15/4.5, f/4.5 | `27 08 35 30 40 39` | 2 | 382 | 394 | 408 | 424 | 436 | 450 | 1.76 | 26 mm |
| Voigtländer 15/4.5, f/22 | `26 f7 40 42 3e 2f` | 2 | 378 | 386 | 397 | 413 | 429 | 445 | 1.74 | 26 mm |
| Sony SEL5518Z, settled | `26 38 f0 ec f0 10` | 2 | 395 | 407 | 411 | 407 | 402 | 398 | 1.55 | 86 mm |
| Viltrox EF-NEX II | `26 00 00 00 00 00` | 2 | 384 | 384 | 384 | 384 | 384 | 384 | 1.50 | — |
| Yongnuo 35 DA | `24 d8 08 10 08 00` | 2 | 304 | 302 | 302 | 304 | 308 | 310 | 1.21 | 42 mm |
| Yongnuo 50 DF, ≤ f/1.9 | `24 a8 f0 f0 f0 f4` | 2 | 315 | 313 | 310 | 306 | 302 | 298 | 1.16 | 58 mm |
| Sony SEL55210 @55 mm | `24 40 d8 d4 dc 00` | 2 | 293 | 302 | 302 | 293 | 282 | 272 | 1.06 | 58 mm |
| Sony SEL2870 @70 mm | `36 88 20 10 08 08` | 3 | 200 | 201 | 202 | 203 | 205 | 209 | 0.82 | 57 mm |
| Sony SEL55210 @210 mm | `35 48 00 f0 f0 c8` | 3 | 192 | 180 | 173 | 171 | 169 | 169 | 0.66 | 139 mm |

The implied exit-pupil distances (`× f`) fit each design: the Voigtländer 15's notoriously short
exit pupil, the SEL5518Z near the sensor's design pupil distance, the 210 mm tele end far out.

The Yongnuo 35 DA sends one type-0 row for its whole table; the 50 DF drifts 2 % across its table.

## 5.2 Type 1 — scales as 1/F; not decoded by the A7 IV

Type-1 rows (bit 7 set) alternate with type 0 in message 0x05. Their `v0` falls by `1/√2` per
stop on the Voigtländer 15/4.5 aperture sweep, **if** bits 6..4 are read as a 3-bit exponent that
wraps (`… e → f → 8 → 9 → a`, each step one octave):

| f-stop | Row | v0 | exponent step | v0 relative to f/4.5 |
| --- | --- | --- | --- | --- |
| f/4.5 | `85 42 97 86 97 b6` | 1346 | 0 | 1.000 |
| f/5.6 | `84 96 c1 ba bb cc` | 1174 | 0 | 0.872 |
| f/8 | `96 7b a7 9e 9e b6` | 1659 | +1 | 0.616 |
| f/11 | `94 93 c3 b9 bc cb` | 1171 | +1 | 0.435 |
| f/16 | `a6 77 a9 9e 9e b5` | 1655 | +2 | 0.307 |
| f/22 | `a4 94 c2 bb ba cb` | 1172 | +2 | 0.218 |

Step ratio f/5.6 → f/22: 0.7066 ± 0.0010 per stop against `1/√2` = 0.7071. CERTAIN as data. The
wide-open `v0` is consistent with `C/F` on seven lenses.

**The A7 IV ignores type 1 on the native path**, so neither this exponent rule nor any meaning is
confirmed by a body. Messages 0x28 and 0x35 usually carry the type-1 half and the A7 IV then
ignores their slot B ([0x35](msg_0x35.md#slot-b)).

---

# 6. Slots C and D

## 6.1 Row 0x15 — POSSIBLE: distortion

Near 0 on the axis and growing roughly with the square of image height (the SEL2870 is close to
`16 · k²` at index `2k`). **The sign follows the lens type**: negative on the wide angles
(barrel), positive on the normal and tele lenses (pincushion), largest on the SELP1650 @16 mm, a
kit zoom at its wide end.

| Lens | Row | e | axis | 2.8 | 5.7 | 8.5 | 11.4 | 14.2 mm |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Sony SELP1650 @16 mm | `de f2 b6 b9 c2 d3` | −3 | 80 | −144 | −504 | −1000 | −1568 | **−2160** |
| Zeiss Loxia 21/2.8 | `0e dc be b0 b6 c8` | 0 | 22 | −16 | −72 | −146 | −226 | **−292** |
| Voigtländer 15/4.5 | `0f 37 d4 c6 c9 de` | 0 | 3 | −10 | −44 | −99 | −157 | **−201** |
| Sony SEL5518Z | `10 6a 04 1e 22 1c` | 1 | −6 | 5 | 19 | 36 | 51 | **+53** |
| Sony SEL55210 @55 mm | `f0 9c 4a 2a 18 0d` | −1 | 2 | 6 | 32 | 80 | 164 | **+312** |
| Sony SEL55210 @210 mm | `01 44 73 5b 41 28` | 0 | −2 | 13 | 53 | 118 | 209 | **+324** |
| Sony SEL2870 @70 mm | `e0 70 2b 1f 15 0d` | −2 | −4 | 16 | 68 | 152 | 276 | **+448** |

## 6.2 Rows 0x16 and 0x17 — POSSIBLE: lateral colour, two channels

Two rows that are not zero on the axis and often have opposite signs, which fits two
colour-difference channels (red and blue against green, as a magnification difference). Loxia and
Voigtländer do not fit the opposite-sign pattern — on the Voigtländer row 0x17 is exactly half of
row 0x16 — so this is the weakest reading here.

| Lens | Row 0x16 (axis → 14.2 mm) | Row 0x17 (axis → 14.2 mm) |
| --- | --- | --- |
| Sony SELP1650 @16 mm | 96 −8 144 552 1224 1944 | 1288 1384 1096 424 −32 −1040 |
| Sony SEL55210 @55 mm | 72 112 224 408 676 1012 | 1324 916 528 160 −208 −420 |
| Sony SEL55210 @210 mm | 656 20 −220 −64 −264 48 | −1804 −1000 −620 −664 −540 −784 |
| Sony SEL5518Z | 396 338 304 294 182 42 | −389 −370 −338 −293 −283 −237 |
| Sony SEL2870 @70 mm | 588 772 818 726 662 590 | −906 −764 −626 −492 −426 −334 |
| Zeiss Loxia 21/2.8 | 315 363 363 315 272 218 | 512 0 −292 −364 −180 0 |
| Voigtländer 15/4.5 | 242 0 0 242 364 582 | 121 0 0 121 182 291 |

Raw rows: SELP1650 `d0 f3 5a 54 33 13` / `df 7e 82 c7 ac dc`; SEL55210 @55 `e0 fd 54 43 2e 1c` /
`ef 97 cb a4 a4 9f`; @210 `e0 0c 4e ce 27 c4` / `ef 3c c3 1f f5 5f`; SEL5518Z `f0 15 ba c8 fb ef` /
`1e 26 5c 14 5a 40`; SEL2870 `f1 27 dc e0 d2 17` / `ff 59 2e 21 43 45`; Loxia `11 b4 93 ab 9f 00` /
`e0 00 2d 2e ee b7`; Voigtländer `f1 23 6d 3d 79 00` / `01 23 6d 3d 79 00`.

## 6.3 Rows seen only at init — UNKNOWN

Sent once or twice during the handshake, before the 0x15–0x17 cycle starts. Rows 7–0x14 use the
shifted layout of §3.2.

| Lens | Row | Bytes | Decoded (axis → 14.2 mm) |
| --- | --- | --- | --- |
| Sony SEL5518Z | `09` | `d0 1b 14 07 f8 ff` | 72 64 32 0 56 216 |
| Sony SEL5518Z | `0A` | `d0 30 16 16 fc ff` | 72 64 48 32 208 384 |
| Sony SEL5518Z | `0B` | `df ff 0a 00 f3 ff` | 24 16 −32 −88 −88 −8 |
| Sony SEL5518Z | `0C` | `d0 26 12 13 ff ff` | 24 16 8 8 160 304 |
| Sony SEL55210 @210 mm | `11` | `e5 b2 4a f8 f8 00` | 5600 5600 5584 5568 5536 5832 |
| Sony SEL55210 @210 mm | `12` | `c0 e5 b3 f4 e4 fc` | 5600 5536 5312 5088 4896 3664 |
| Sony SEL55210 @55 mm | `13` | `e4 66 e8 e7 db fc` | 4864 4848 4772 4700 4600 4504 |
| Sony SEL55210 @55 mm | `14` | `c0 6b b5 d2 c1 f3` | 4864 4656 4144 3648 2912 1712 |
| Sony SELP1650 @16 mm | `13` | `90 08 b6 0d f1 e5` | 14208 10752 9728 8832 10496 1024 |
| Sony SELP1650 @16 mm | `14` | `a0 00 ce df d0 a4` | 14272 8384 6848 5312 3200 0 |

## 6.4 Lenses that send nothing here

The Yongnuo YN35mm F1.8S DA sends two fixed, all-zero 25-byte records whatever the row number. The
Viltrox adapters send row number 0 and zero rows. Both work on bodies, so empty slots C/D are an
accepted state (§7).

---

# 7. What zero and default rows mean to the body

| Sent | Body result |
| --- | --- |
| Slot A `00 00 00 00 00 00` | A flat zero curve: no shading. No division involved |
| Slot B `26 00 00 00 00 00` | Type 0, flat 384 (`p` = 1.50) |
| Slot B `22 00 00 00 00 00` | Type 0, flat 128 (`p` = 0.50) — below every lens observed |
| Slot B all zero | Type 0, value 0; the reciprocal tables take their guard value. In 0x28/0x35 this **overwrites** the 0x05 value at every capture ([0x35](msg_0x35.md#slot-b)) |
| Slot C/D with `pl[77] = pl[78] = 0` | Row 0 is dropped and slot D is not decoded: nothing is filed, and the row 0x15–0x18 received flags stay clear |

What the main processor does with a missing or zero table is **UNKNOWN**.

---

# 8. Practical notes for implementers

## Which parity holds which type is NOT fixed

Every entry of both complete Yongnuo tables carries exactly one type-1 and one type-0 slot B row.
In six entries the parities are swapped:

| Lens | Swapped entries | Parity 0 there |
| --- | --- | --- |
| Yongnuo 35 DA | 9, 10, 11 | `24 d8 08 10 08 00` — type 0 |
| Yongnuo 50 DF | 16, 17, 22 | `24 8c f8 …` — type 0 |

Messages 0x28 and 0x35 carry parity 0, so in those entries they deliver the type-0 row. **Always
test bit 7.**

## Writing rows

- Put the **outermost** value in `v0` and encode each delta as `inner = outer − delta`.
- Keep `v0` below `0x800` unless a negative value is meant.
- For slots A, C and D the exponent is the whole signed nibble; for slot B type 0 it is bits 6..4
  with bit 6 as the sign.
- Donor rows are safest taken **wide open**: slot A and slot B type 1 are aperture-dependent.

---

# 9. Sources

| Lens | Body | What varied | 0x05 frames |
| --- | --- | --- | --- |
| Voigtländer Super Wide-Heliar 15/4.5 (manual focus) | NEX-7 | Aperture ring only | 461 |
| Voigtländer 15/4.5 | NEX-7 | Focus ring only, f/4.5; infinity | 536; 131 |
| Zeiss Loxia 21/2.8 (manual focus) | NEX-7 | Init | 64 |
| Sony SEL5518Z | NEX-7 | Power-on, including the init sweep | 281 |
| Sony SEL2870 @70 mm | NEX-7 | Init | 109 |
| Sony SELP1650 @16 mm | A6000 | Init | 216 |
| Sony SEL55210 @55 mm / @210 mm | A6000 | Init | 76 / 136 |
| Viltrox EF-NEX II + Canon EF 50/1.8 | A6000 | Init | 205 |
| Yongnuo YN35mm F1.8S DA, YN50mm F1.8S DF, YN50mm F1.8S DA | — | Complete tables, from firmware | — |
| Sony A7 IV firmware 6.02 | — | The decoder and consumer (`0x98d09e`, `0x98d4fc`, `0x98cff0`, `0x98d266`, `0x3ab854`, `0x3ac878`) | — |

---

# Open questions

| Question | Status |
| --- | --- |
| Units of every slot | **UNKNOWN** |
| Image-height grid in µm | **PROBABLE** (§3.3) |
| Slot A = shading, slot B type 0 = pupil magnification | **PROBABLE** |
| Row 0x15 = distortion, rows 0x16/0x17 = lateral colour | **POSSIBLE** |
| What rows 0x09–0x14 carry | **UNKNOWN** |
| What a body does with slot B type 1 | **UNKNOWN** — the A7 IV ignores it |
| The adapter path (lens type 5) reads slot B byte 0 as an index into a body table (`0xda96d8`, 0–0x50) | Reads **CERTAIN**; table contents **UNKNOWN** |
