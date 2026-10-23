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
| **`pl[13]`** | **Declares the [message 0x05](msg_0x05.md) length.** `(pl[13] & 0xFE) == 0x80` → 105 bytes, anything else → 117 bytes, before the body's [message 0x0A](msg_0x0A.md) can shorten it. **Also tracks the image format**: bit 5 (`0x20`) is set on every full-frame device and clear on every APS-C one, and clearing it confined a Sony a9 II's AF area to the APS-C region. [Below](#pl13-declares-the-message-0x05-length) | Length rule **CERTAIN** in Yongnuo firmware, **PROBABLE** as protocol; format observation **CERTAIN**, its meaning **POSSIBLE** |
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
f/1.8 prime). And the Yongnuo 50 mm full-frame lens's stored message 0x08 is **byte-identical to the
35 mm APS-C lens's across all 201 payload bytes — zero differences.** At runtime each model then
patches a few bytes, and only two of them differ between models: `pl[13]`, which declares the
message 0x05 length ([below](#pl13-declares-the-message-0x05-length)), and `pl[17]` (`0x35` on the
YN35, `0x50` on both 50 mm lenses, meaning UNKNOWN).

Two lenses of different focal length, aperture range and image circle sharing all but two bytes,
neither of them optical, means message 0x08 carries **no per-lens optical data at all**. It is fixed
native-lens boilerplate.

## One vendor, one descriptor across the lineup

Yongnuo sends one 201-byte message 0x08 across three distinct lens models, identical apart from
`pl[13]` and `pl[17]`:

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

## `pl[13]` declares the message 0x05 length

`pl[13]` of the lens's reply tells the body which form of [message 0x05](msg_0x05.md) the lens
wants to send:

| `pl[13] & 0xFE` | Declared message 0x05 length |
| --- | --- |
| `0x80` (i.e. `pl[13]` is `0x80` or `0x81`) | **105 bytes** |
| anything else | **117 bytes** |

It is only a starting point. The body's [message 0x0A](msg_0x0A.md) request, two exchanges later,
can force 105 bytes whatever this byte says (`pl[0..1] = ff 7f`), and cannot force 117. So
`pl[13]` is a **ceiling**: 117 bytes are sent only if this byte asks for them and the body allows it.

**The body picks that request from this byte.** On a Sony a9 II, `0x81` drew `ff 7f 00` (105
bytes), and `0xe1` drew `ff ff 03` — and the body then **required** the 117-byte form: a lens that
kept sending 105 lost the session and was power-cycled. Details in
[message 0x0A](msg_0x0A.md#the-request-depends-on-the-lenss-message-0x08).

| Device | `pl[13]` | Declares | Actually sent |
| --- | --- | --- | --- |
| Sony SELP1650 | `0x8c` | 117 | 105 — the A6000 sends `ff 7f` |
| Voigtländer 15 mm | `0xe1` | 117 | 105 — the NEX-7 sends `ff 7f` |
| Yongnuo YN35 / 50 DA | `0x81` | 105 | 105 |
| Yongnuo 50 DF | `0xe1` | 117 | 117 unless the body sends `ff 7f` |
| Viltrox EF adapters | `0x20` | 117 | 105 — the A6000 sends `ff 7f` |

This byte is one of the nine "all three differ" bytes in the table above (Voigtländer `e1`, YN35
`81`, SELP1650 `8c`): not optical data, but a per-device choice of status format.

### It also tracks the image format

| Device | Format | `pl[13]` |
| --- | --- | --- |
| Yongnuo 50 mm DF | full frame | `0xe1` |
| Voigtländer 15 mm | full frame | `0xe1` |
| Viltrox EF adapter | full-frame mount | `0x20` |
| TECHART LM-EA9 | full-frame mount (Leica M) | `0x20` |
| Yongnuo 35 mm DA, 50 mm DA | APS-C | `0x81` |
| Sony SELP1650 | APS-C | `0x8c` |

Bit 5 (`0x20`) is set on every full-frame device and clear on every APS-C one.

**Observed on a Sony a9 II (full-frame body):** a device sending `0x20` here could place the AF
point anywhere in the frame. The same device sending `0x81`, with nothing else changed, could
only place it inside the APS-C region. Restoring `0x20` restored the full range.

So the body reads more from this byte than the status length. That bit 5 means "full-frame image
circle" fits both the table and the a9 II result, but is not established. What bits 0, 2, 3, 6
and 7 mean is UNKNOWN.

| Aspect | Confidence |
| --- | --- |
| Yongnuo firmware applies exactly this rule (three models, two separate code bases) | **CERTAIN** |
| It is protocol behaviour that Sony lenses follow too | **PROBABLE** — every capture agrees with it and none contradicts it |
| A Sony a9 II answers `0xe1` with a 0x0A of `ff ff 03` and then requires 117 bytes | **CERTAIN** on one a9 II |
| `0x20` declares 117 by the `& 0xFE` rule, yet a lens sending it and 105 bytes works on an a9 II | **CERTAIN** — so the body reads more of this byte than `& 0xFE` |
| Bit 5 set on full-frame devices, clear on APS-C ones (six devices) | **CERTAIN** as an observation |
| Clearing bit 5 confines a Sony a9 II's AF area to the APS-C region | **CERTAIN** for `0x20` → `0x81` on one a9 II; which bit is responsible is not isolated |
| Bit 5 means "full-frame image circle" | **POSSIBLE** |

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

## Request frames, observed

Whole frames, checksums verified:

```
Sony a9 II   F0 11 00 02 00 08 | C6 E1 00 00 F9 0D 62 02 | 2C 03 55
Sony A6000                     | C2 61 00 00 00 0C 62 02 |
```

17 bytes: the 6-byte header, 8 payload bytes, and the 3-byte trailer.

| `pl` | a9 II | A6000 | |
| --- | --- | --- | --- |
| `0` | `C6` | `C2` | **UNKNOWN** |
| **`1`** | **`E1`** | **`61`** | **bit 7 is the capability bit below**; the rest **UNKNOWN** |
| `2..3` | `00 00` | `00 00` | **UNKNOWN** |
| `4` | `F9` | `00` | **UNKNOWN** |
| `5` | `0D` | `0C` | **UNKNOWN** |
| `6..7` | `62 02` | `62 02` | Constant across both bodies. **UNKNOWN** |

## The request enables an optional field in messages 0x28 and 0x35

**Bit 7 of `pl[1]` of the body's request is a capability bit.** Cleared, it tells the lens to
populate `pl[9..10]` of [message 0x28](msg_0x28.md) and of [message 0x35](msg_0x35.md); set, those
two bytes are left at zero in both.

| `pl[1]` bit 7 | msg 0x28 / 0x35 `pl[9..10]` |
| --- | --- |
| 0 | the lens's current [aperture value](aperture_value.md) |
| 1 | **not written** |

The lens latches the bit when it answers message 0x08 and applies it to every later 0x28 and 0x35
of the session, so it is negotiated once at init rather than per request. The message is sent
**exactly once per session**, sixth in the handshake, in every session observed.

### The two bodies disagree, and the bit is the only payload byte that says so

| Body | `pl[1]` | bit 7 | wants `pl[9..10]` |
| --- | --- | --- | --- |
| Sony A6000 | `0x61` | clear | **yes** |
| Sony a9 II | `0xE1` | set | **no** |

`0xE1 ^ 0x61 = 0x80` — the two differ in exactly that bit.

### Ignoring the bit has a visible cost

**A lens that populates `pl[9..10]` when the body did not ask for it breaks the body's metering.**
On a Sony a9 II — which sets the bit, i.e. does not want the field — an adapter that filled it in
anyway made the camera **re-meter after every exposure**, and produced metering errors during
continuous shooting. Removing the value, so the field is zero as the bit asks, fixed both.

CERTAIN on that body: the behaviour appeared and disappeared with that field alone, everything else
held still.

So the field is not merely optional — it is **harmful to send unasked**, and a lens must follow the
bit rather than always filling the field in "for completeness".

| Aspect | Confidence |
| --- | --- |
| `pl[1]` bit 7 of the request gates `pl[9..10]` in both messages | **PROBABLE** |
| It is latched at init and not re-read per request | **PROBABLE** |
| Populating the field against the bit disturbs metering | **CERTAIN** on a Sony a9 II |
| What the body does with the field when it *is* populated | **UNKNOWN** |

A device that never populates the field is not broken: a TECHART LM-EA9 sends zero in message 0x28
`pl[9..10]` always, and the photographs it produces still record the aperture correctly. So the body
has another source for the same quantity, most obviously [message 0x05](msg_0x05.md) `pl[0..1]`,
which arrives sixty times a second.
