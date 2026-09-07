# Message 0x08 — the large descriptor

**Summary.** A bulk feature/capability descriptor, sent once during init. The largest message in
the protocol and the most asymmetric: 8 bytes out, 201 bytes back.

**Direction:** both. The body request is 8 bytes — `c2 61 00 00 00 0c 62 02` on the A6000,
identical for every lens; the lens reply is **201 bytes**.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Field map — lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..1]` | u16 — equals the lens's focus position (`00 13` LM-EA9, `bd 13` SELP1650) | PROBABLE |
| `pl[6..7]` | u16 = 5632 on the LM-EA9 — **the focus limit it advertises**, which its own clamp disagreed with in earlier versions | PROBABLE |
| The 16 structured runs below | Native-lens boilerplate | **CERTAIN that they are lens-independent** |
| Everything else | UNKNOWN | |

## Fill rate is a clean native/adapter discriminator

| Source | Non-zero payload bytes | % |
| --- | --- | --- |
| Sony SELP1650 (native) | 93 / 201 | **46.3** |
| Voigtländer 15/4.5 (native, manual) | 77 / 201 | 38.3 |
| **Yongnuo YN35 / 50F1.8S** | 82 / 201 | **40.8** |
| Viltrox + Canon EF 50 | 23 / 201 | 11.4 |
| Viltrox + Canon EF-S 24 | 22 / 201 | 10.9 |
| **TECHART LM-EA9** | **21 / 201** | **10.4** |

Natives cluster at 38–46 %, adapters at 10–11 %.

## The "native-only" runs are lens-INDEPENDENT

57 bytes in 16 structured runs, 2–5 bytes long, are zero on the LM-EA9 and populated on natives:

```
pl[0x28..0x29] e0 43           pl[0x2B..0x2E] e0 ff 84 a4     pl[0x31..0x35] 08 32 25 40 2f
pl[0x3D..0x40] 43 25 04 32     pl[0x48..0x4C] f3 f3 07 fb fb  pl[0x50..0x53] 04 04 20 46
pl[0x5F..0x63] 01 01 01 09 09  pl[0x69..0x6C] e6 80 dd 52     pl[0x6E..0x71] d0 dd df f0
pl[0x73..0x76] 08 a0 dd e0     pl[0x85..0x87] 84 64 27        pl[0x89..0x8C] 06 10 ff 33
pl[0x9F..0xA0] 0c 0c           pl[0xB0..0xB2] 07 07 ff
```

These are **not** per-lens optical tables. All 14 runs with recorded native values are byte for byte
identical between the Sony SELP1650 (a 16–50 mm f/3.5–5.6 power zoom) and the Yongnuo YN35 (a 35 mm
f/1.8 prime). And the Yongnuo 50 mm full-frame lens's message 0x08 is **byte-identical to the 35 mm
APS-C lens's across all 201 payload bytes — zero differences.**

Two lenses of different focal length, aperture range and image circle sharing every byte means
message 0x08 carries **no per-lens optical data at all**. It is fixed native-lens boilerplate.

## One vendor, one descriptor across the lineup

Yongnuo sends a byte-identical 201-byte message 0x08 across three distinct lens models:

| Model | Format |
| --- | --- |
| YN 35 mm f/1.8 DA | APS-C |
| YN 50 mm f/1.8 DA | APS-C |
| YN 50 mm f/1.8 DF | Full-frame |

Messages 0x0D, 0x10 and 0x16 are likewise identical across that lineup; 0x01, 0x07, 0x09 and 0x3D
are identical everywhere they could be compared.

## Three real lenses: what is boilerplate, what is per-lens

Three independent real lenses, deliberately spanning manual/AF and prime/zoom:

| | Voigtländer 15/4.5 | Yongnuo YN35 | Sony SELP1650 |
| --- | --- | --- | --- |
| Type | Manual prime | AF prime | AF power zoom + OSS |
| Non-zero | 77/201 | 82/201 | 93/201 |

A lens that **cannot autofocus at all** still fills in 77 of 201 bytes — further evidence 0x08 is
not AF-specific.

| Category | Bytes | Reading |
| --- | --- | --- |
| All three agree | **154** | Boilerplate — the LM-EA9 differs on **53** of them |
| The two primes agree, the zoom differs | **23** | Zoom/OSS-related, *not* vendor-specific |
| Yongnuo + Sony agree, Voigtländer differs | 13 | |
| Voigtländer + Sony agree, Yongnuo differs | 2 | |
| All three differ | **9** | Genuinely per-lens — the focus head |

The "prime vs zoom" split rules out reading these bytes as vendor-varying: at `pl[0x29]`, `0x2B`,
`0x2E`, `0x33`, `0x35`, `0x3D`, `0x3E`, `0x52`, `0x53`, `0xB2` and others, two *unrelated vendors'*
primes agree and only the zoom differs. Confidence: this is **one zoom against two primes** — a
hypothesis that fits, not a proven axis. A second zoom would test it cheaply.

## Tested on hardware — negative result

**A 66-byte native fill of message 0x08 was built, loaded onto a real TECHART LM-EA9, and made no
difference to autofocus performance on a Sony a9 II.** It took the non-zero payload from 21/201 to
75/201 — 10.4 % → 37.3 %, inside the native 38–46 % band — so the adapter genuinely stopped looking
like an adapter by this measure, and AF behaviour still did not change.

**Protocol conclusion, and it is a hard one: whatever the body uses for off-axis PDAF correction is
not carried in message 0x08.**

Not covered by that test, and therefore **not** ruled out: the focus head and the 9 all-differ bytes
— the lens's own focus position and travel range, where the LM-EA9's values are already right for
its own helicoid.

## Manufacturer notes

- **Sony** — the SELP1650 has the highest fill rate measured, 93/201.
- **Yongnuo** — sends one identical descriptor across APS-C and full-frame, primes of different
  focal length included.
- **TECHART** — the LM-EA9 sends the lowest fill rate measured, 21/201, and differs from the
  three-lens consensus on 53 of the 154 boilerplate bytes.
- **Viltrox** — the EF adapters sit with the LM-EA9 at 10–11 %.
- **TTArtisan / Meike** — corroborating 0x08 blocks exist for these vendors.

## Open questions

- What the descriptor is *for*. The hardware test rules out off-axis PDAF correction, and the
  lens-independence rules out per-lens optics, leaving the purpose UNKNOWN.
- The meaning of the 154 boilerplate bytes.
- Whether the 23-byte "prime vs zoom" group really is zoom/OSS-related. One zoom against two primes.
- The 9 all-differ bytes beyond the focus position and travel range.
