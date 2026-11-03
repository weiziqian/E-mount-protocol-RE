# Message 0x3B — the paged optical file

**Summary.** The lens serves a file of about 2.7 KB, 488 bytes per request, announced beforehand by
[message 0x3A](msg_0x3A.md). It holds a table indexed by aperture and a second axis, with values that
behave like **pupil and vignetting data across the image field**. On an a9 II it decides whether
**AF-C and video** can use phase-detect AF — [below](#effect-on-the-body). Never captured on the wire.

**Direction:** L→B reply to a paged body request, which has not been observed. **Class:** `0x03`.
**Length:** 500 bytes (491-byte payload).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Paging — CERTAIN in Yongnuo and TTArtisan firmware

| Field | Meaning |
| --- | --- |
| request `pl[0]` | page number, 1-based; Yongnuo clamps it to the page count from [0x3A](msg_0x3A.md) |
| request `pl[1]` | echoed into reply `pl[2]` by Yongnuo; `00` in TTArtisan's stored replies |
| reply `pl[0..2]` | `00 <page> <request pl[1]>` |
| reply `pl[3..490]` | the next 488 bytes of the file |

The pages joined in order give the file. Its end is zero-filled. The file is chosen by the preceding
0x3A request.

## The file

A 16-byte header followed by three chunks. Every chunk has an 8-byte header: a 2-character tag, a u16
header size (`08 00`), and a u32 LE data length.

| Part | Contents | Confidence |
| --- | --- | --- |
| `LF` header | `4c 46 10 00`, u32 LE length of what follows, u32 version (`0b`, `0d`, `01` seen), u32 UNKNOWN | layout CERTAIN |
| `ED` | **the main table**, [below](#the-table-ed) | layout CERTAIN |
| `CS` | **the table's axes** — header `01 00 <Nf> 00 <Na> 00 00 00`, then `Nf` u16, then `Na` u16 | CERTAIN |
| `LC` | 12 bytes: u16 `01` on full-frame lenses / `00` on APS-C, then four u16 UNKNOWN | format flag PROBABLE |
| last 4 bytes | likely a checksum; not CRC-32 of any obvious range | UNKNOWN |

### The two axes (`CS`)

- **`Na` aperture steps, in ⅓ stop**: `0x000, 0x055, 0x0ab, 0x100 …`. 256 counts are one stop, the
  scale of the [aperture value](aperture_value.md) without its offset. The last entry (`0xa55`,
  `0xaaa`) breaks the progression and looks like an end marker.
- **`Nf` values of a second axis**, increasing, always ending in `0x700`. A 50 mm lens has 8 or 9 of
  them and a 17 mm lens has 4, so the axis is **probably focus position**. That is POSSIBLE.

### The table (`ED`)

**Layout — CERTAIN, it parses all five files exactly.** An 8-byte header of four u16, then `Na × 3`
blocks:

| Part | Contents |
| --- | --- |
| header | `Nh` = 6 (values per row), then `1`, then `Nf`, `Na` — the same `Nf` and `Na` as `CS` |
| block | u16 `1`, u16 `R` (number of focus segments), u16 `0`, u16 `0`, `R − 1` u16 focus breakpoints, then `R` rows of `Nh` u16 |
| block order | aperture-major: three blocks (types 0, 1, 2) for aperture 0, then three for aperture 1, … |

A row holds for the focus indices from its breakpoint up to the next one. The last segment always
starts at `Nf − 1`, so the end of the focus axis (`0x700`) has a row of its own. Most files use two
segments, one for every other focus position and one for that end. The Yongnuo 35 uses up to four.

**Meaning — POSSIBLE.** The readings below fit every file, but the columns have not been tied to
anything:

- **Types 1 and 2 are the two edges of the pupil, and the six columns run from the image centre to
  the edge.** In column 0 the two types agree to within 1 in three of the five files, and nearly
  everywhere in the other two (to within 3 in 95 % and 88 % of rows). They separate after column 0.
  That is what the pupil does: it is symmetric on axis and loses symmetry off axis.
- **Type 1 shows mechanical vignetting.** Its outer columns hold a fixed value at wide apertures and
  only start to rise once the aperture is smaller than that value. The lens barrel limits the off-axis
  beam until the iris becomes the smaller stop. Type 2 rises slowly toward the edge, or not at all.
- **The values are f-numbers on a 256-per-stop scale.** Column 0 rises by about `0x55` (⅓ stop) per
  aperture row. At the `0x700` end it reads about 461 on f/1.8 lenses, which is f/1.85 on that scale.
  The other focus segment reads higher (573, about f/2.2), the way the working f-number grows at close
  focus.
- **Type 0 barely changes with aperture.** It may be the exit-pupil position.

TTArtisan 50/1.8, type 1, `0x700` end, the first rows (decimal):

```
461  532  638  779  970 1217     wide open
546  550  638  779  970 1217
631  635  648  779  970 1217
717  721  733  779  970 1217     1 stop down
```

If this reading holds, the file is the per-lens pupil data that on-sensor PDAF needs away from the
frame centre.

## Effect on the body

Tested on a **Sony a9 II**, with a device that serves 0x3A/0x3B and asserts both in
[message 0x01](msg_0x01.md):

- **with** 0x3A/0x3B, **AF-C and video use phase-detect AF**;
- **without** them, those modes get no phase-detect AF;
- **AF-S is not affected** either way.

**CERTAIN** on the a9 II; other bodies not tested.

## Observed files

| Lens | `LF` version | `Nf` × `Na` | focus segments | `LC` format |
| --- | --- | --- | --- | --- |
| Yongnuo 50 DF | `0b` | 8 × 24 | 2 | `01` |
| Yongnuo 35 DA | `0b` | 15 × 24 | 2–4 | `00` |
| TTArtisan 50/1.8 — set A | `0b` | 8 × 24 | 2 | `01` |
| TTArtisan 50/1.8 — set B | `01` | 9 × 25 | 2–3 | `01` |
| TTArtisan 17/1.8 | `0d` | 4 × 24 | 2 | `00` |

The TTArtisan 50/1.8 picks between its two sets according to the body: one set for some bodies, the
other for the rest, which include the a9 II.

These are all third-party lenses. No Sony lens's file has been seen, so a field can be a vendor
convention rather than the protocol.

## Manufacturer notes

- **Yongnuo** — the 50 DF stores the file bare and cuts it into pages at send time. It also stores
  one ready-built page, which is the `08` object of [0x3A](msg_0x3A.md). The 35 DA stores its file as
  ready-built 500-byte pages. Both lenses assert 0x3B in [message 0x01](msg_0x01.md).
- **TTArtisan** — stores ready-built pages. The 50/1.8 carries two sets selected by body, the 17/1.8
  carries one, and both assert 0x3B.

## Open questions

- Which other bodies depend on it, and what AF-S uses instead. No captured lens asserts 0x3A/0x3B,
  so no capture could show the request.
- Whether the columns are image heights, and which ones; what type 0 is; the zero point of the
  scale.
- The `1` in the `ED` header (one zoom position, on a prime?) and the two zero words in each block.
- The `LF` header's last word, the `LC` values, and the trailing checksum.
