# Lens ID — what a body does with it

**Summary.** The lens ID is [message 0x07](msg_0x07.md) `pl[9..10]`, u16 LE. No value makes a body
refuse a lens: the 0x07 aborts look only at `pl[0]` and `pl[4]`. But the ID decides **how far the
body trusts the rest of the init data**. An ID in the third-party range makes the A7 IV overwrite
much of what the lens sent in 0x07, 0x08, 0x09 and 0x3D with fixed values, narrow the set of
messages it will use, and skip most of the 0x3A/0x3B files.

**Source.** Everything below is read from the Sony A7 IV 6.02 camera-processor firmware and is
**CERTAIN on the A7 IV firmware** unless marked otherwise. What the main processor then does with
the data is not visible. Other bodies are untested.

`pl` is the payload of a message: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.
Ranges `pl[a..b]` are inclusive of both ends.

## The three ID classes

| ID | A7 IV class | Examples |
| --- | --- | --- |
| `0x8000`–`0xBFFF`, low 12 bits ≠ `0xFFF` | **Sony E-mount** | SEL55210 `0x8012`, SELP1650 `0x8019` |
| `0xC000`–`0xFFFF`, low 12 bits ≠ `0xFFF` | **third-party** | Yongnuo `0xC540`, Voigtländer `0xC730` |
| anything else (`0x0000`–`0x7FFF`, or low 12 bits = `0xFFF`) | neither | Viltrox EF adapter `0x004E`, TECHART LM-EA9 `0x00EA` |

- **E-mount:** the body passes "E-mount ID" to the main processor as one entry of the lens feature
  table. Bits 13–15 of [message 0x08](msg_0x08.md) `pl[23..24]` can veto it: if bit 14 or 15 is
  set, bit 13 must be clear. What the main processor does with the entry is UNKNOWN.
- **Third-party:** the ID's top two bits are both set. The test runs when 0x07 arrives, before
  anything else in the reply is stored. A body setting can force every lens into this class; it is
  off in the image.
- **Neither:** the data is taken as sent, as for a Sony ID, but without the E-mount entry.

## What "third-party" changes

Each reply is edited in place, before the body stores it. Fields not listed keep the lens's value.

### [0x07](msg_0x07.md)

| Field | Becomes |
| --- | --- |
| `pl[0]` | `& 0xF3` — the native/adapter bits survive |
| `pl[1]` | `& 0x03` — the speed bits survive |
| `pl[7]` | `00` |
| `pl[8]` | `A0`, the constant every measured device sends anyway |
| `pl[11..14]` | `00` |
| `pl[25]` | `& 0x64` |
| `pl[26..33]` | `00` |

### [0x08](msg_0x08.md)

Most of the reply is replaced by a fixed image. Its non-zero bytes are the populated runs of a
Sony SELP1650's 0x08: `pl[40..41] = e0 43`, `pl[43..46] = e0 ff 84 a4`,
`pl[49..53] = 08 32 25 40 2f`, … So a third-party lens is described to the body by a template close
to a real Sony lens, whatever it sends.

| Treatment | Fields |
| --- | --- |
| Kept as sent | `pl[0..3]` (apertures), `pl[7..11]`, `pl[16..21]`, `pl[23..30]` (includes the serial number), `pl[78..79]`, `pl[171..173]`, `pl[194]` |
| Partly kept | `pl[12] & 0x56`; `pl[13] & 0x7D` with bit 7 set — bits 5 and 6, which select the 117-byte 0x05 and the sync timing, survive; `pl[14] & 0x07`; `pl[15] & 0x03`; `pl[85] & 0x8C` (bits 7, 3, 2); `pl[195] & 0x1D` with bit 1 set |
| Forced non-zero | `31`=16 `40`=e0 `41`=43 `43`=e0 `44`=ff `45`=84 `46`=a4 `49`=08 `50`=32 `51`=25 `52`=40 `53`=2f `61`=43 `62`=25 `63`=04 `64`=32 `72`=f3 `73`=f3 `74`=07 `75`=fb `76`=fb `80`=04 `81`=04 `82`=20 `83`=46 `86`=01 `95..97`=01 `98..99`=09 `105`=e6 `106`=80 `107`=dd `108`=52 `110`=d0 `111`=dd `112`=df `113`=f0 `115`=08 `116`=a0 `117`=dd `118`=e0 `120`=52 `122..123`=10 `133`=84 `134`=64 `135`=27 `137`=06 `138`=10 `139`=ff `140`=33 `146`=03 `159..160`=0c `168`=05 `170`=01 `176..177`=07 `178`=ff `198`=01 `200`=01 |
| Zeroed | everything else, including the second model ID `pl[34..35]` |

The indices are payload offsets; values are hex.

### [0x09](msg_0x09.md)

`pl[0] & 0x70` — the adapter sub-type bits are cleared, so an adapter's sub-type reads as `0`.
`pl[1] & 0x02`, `pl[2] & 0x80`, `pl[3..6]` and `pl[9..10]` zeroed.

### [0x3D](msg_0x3D.md)

- Of the feature bitmap at the start of the reply, **only bit 0 is honoured**. Bits 8–14 are what
  select the extra [0x3A](msg_0x3A.md#other-files--certain-on-the-a7-iv-firmware) files, so a
  third-party lens is never asked for CamcurveData, BreathingCorrectionData, SteadyShotInfo,
  FnoCorrectAbility or IrisDrivabilityData. The Eclipse file of [0x3B](msg_0x3B.md) does not depend
  on the bitmap and is still read.
- `pl[32..33] & 0x0C01`; `pl[34..62]` zeroed.
- Further feature flags the body takes from `pl[32..35]` for a Sony or legacy ID are not taken.
- `pl[49]` is cleared, so the lens's own focus-drive speed calibration is never used
  ([AF range](af_range.md#focus-drive-speed)). The [AF range](af_range.md) decision still sees `pl[0]`
  as sent, and bit 0 of `pl[32]`.

### [0x05](msg_0x05.md) and [0x06](msg_0x06.md), every frame

Parts of the tagged sub-records of each status frame are masked or zeroed: records 3, 4, 5, 7, 8
and 9 of 0x05, records 0x40–0x44 of 0x06. What those records are is UNKNOWN; a frame in the classic
fixed layout is not known to carry them.

### Messages the body will use

The body's per-ID mask becomes `0x01`–`0x40` plus `0x4B` and `0x4C`, in place of `0x01`–`0x60`
without `0x4D`. So `0x41`–`0x4A`, `0x4E` and `0x50`–`0x60` are never sent to a third-party lens,
including the optional handshake steps [0x4E](msg_0x4E.md) and [0x5A](msg_0x5A.md), whatever the
lens declares in [0x01](msg_0x01.md).

## Individual Sony IDs

The A7 IV also branches on single Sony IDs: a four-entry list (`0x8019`, `0x8020`, `0x8031`,
`0x8034`), a body setting each for `0x8036`–`0x8038`, the lens-switch report for `0x8022`,
`0x8031`, `0x8034` and `0x8057`, switch tables over IDs such as `0x8052`, `0x8054`,
`0x803D` and `0x803E`, and a 53-entry table of Sony IDs holding each lens's focus-drive speed calibration
([AF range](af_range.md#focus-drive-speed)). These
are per-lens quirks, not protocol. A device that borrows a Sony ID inherits them.

## Re-attaching the same lens

For a native (type 4, see [lens types](lens_types.md)), the body remembers the ID and reuses a
stored value when the same lens comes back. Both bytes of the ID must be non-zero and match.

## Open questions

- What the main processor does with the E-mount entry.
- What the 0x05/0x06 sub-records hold.
- Whether other bodies classify IDs the same way. The a9 II autofocuses with third-party
  `0xC540`, so at least nothing there depends on it fatally.
