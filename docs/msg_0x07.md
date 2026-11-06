# Message 0x07 — identity and lens ID

**Summary.** Lens identification. The second exchange of the init phase, immediately after the
capability bitmap.

**Direction:** both. The body request is 1 byte — `00` on the A6000 and the a9 II, `08` in the
A7 IV firmware; meaning UNKNOWN. The lens reply is 34 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Field map — lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | **`01` on every native, `02` on every adapter.** Selects how the body drives focus: [below](#pl0-native-mode-vs-adapter-mode). The A7 IV reads `pl[0] & 3` and **aborts the handshake on `0` or `3`** (log `LENS UNKNOWN`). What the body does differently per class: [lens types](lens_types.md) | Discriminator **CERTAIN**; behaviour **CERTAIN** on one a9 II; abort **CERTAIN** on the A7 IV firmware; mode reading **PROBABLE** |
| **`pl[1]`** | **UART speeds the lens supports.** bit 1 → speed 1, bit 2 → speed 2. The body requests the fastest one it also supports in [message 0x0C](msg_0x0C.md), and skips 0x0C if neither bit is set. Every device measured sends `03`, and the A6000 then sends `0C 02` | **CERTAIN** on the A7 IV firmware; consistent with every capture |
| **`pl[2..3]`** | **Protocol version, u16 LE.** The A7 IV grades it at `0x0108`, `0x0110` and `0x0120`; below `0x0108` it ignores the [message 0x08](msg_0x08.md) `pl[85]` feature bits | **CERTAIN** on the A7 IV firmware |
| **`pl[4]`** | **Must be `00`.** Any other value aborts the handshake on the A7 IV (log `LENS FIRMERR` — a lens-firmware error flag) | **CERTAIN** on the A7 IV firmware |
| `pl[6]` | **BCD firmware version on the TECHART LM-EA9** (`0x18` = 1.8) | CERTAIN for the LM-EA9 |
| `pl[8]` | `0xA0` on **every** device measured | CERTAIN constant, meaning UNKNOWN |
| **`pl[9..10]`** | **u16 LE Sony lens ID.** Its range decides how far the body trusts the rest of the init data: [lens ID](lens_id.md) | **CERTAIN** |
| `pl[25]`, `pl[26..27]`, `pl[28]` | Bit fields the A7 IV reads (`pl[28]` bit 0 feeds its [message 0x0B](msg_0x0B.md) request) | UNKNOWN |
| `pl[15..18]` | `60 92 86 5e` on the LM-EA9, on a third-party native, and on both Yongnuo lenses | **Protocol constant** — CERTAIN |

## `pl[0]`: native mode vs adapter mode

Observed on a Sony a9 II, the same lens switching only this byte:

| | `01` | `02` |
| --- | --- | --- |
| Suspected meaning | native-lens mode | adapter-only mode |
| Focus-area modes | all | only some |
| AF during video | yes | no |
| AF in magnified view | yes | no |
| How the body drives focus | a search: scan, relative moves and absolute moves combined | straight to the target with absolute moves |

With `02`, [message 0x08](msg_0x08.md#pl85) `pl[85] = 0x80` stops AF altogether.

| Aspect | Confidence |
| --- | --- |
| The differences in the table | **CERTAIN** on one a9 II |
| `01` = native mode, `02` = adapter mode | **POSSIBLE** |

## `pl[2..3]`: protocol version

| Device | `pl[2..3]` |
| --- | --- |
| Sony SEL55210 | `0x0135` |
| Sony SELP1650 | `0x0120` |
| Viltrox EF adapter | `0x0108` |

`0x0108` is the lowest grade the A7 IV distinguishes. Which features the higher grades unlock is
UNKNOWN.

## Lens IDs observed

| Device | `pl[0]` | Lens ID | ID space |
| --- | --- | --- | --- |
| Sony SEL55210 | 01 | 32786 = `0x8012` | E-mount (`0x8xxx`) |
| Sony SELP1650 | 01 | 32793 = `0x8019` | E-mount |
| **Yongnuo YN35 *and* 50F1.8S** | 01 | **50496 = `0xC540`** | Third-party block |
| Viltrox + Canon EF 50 / EF-S 24 | 02 | 78 = `0x004E` | Legacy / A-mount |
| **TECHART LM-EA9** (all versions) | 02 | **234 = `0x00EA`** | Legacy / A-mount |

`pl[10]` is simply the **high byte of the lens ID** — native E-mount IDs are ≥ `0x8000` while both
adapters sit low. It is not an independent native/adapter flag.

## The lens ID cannot be what selects optical corrections

The third-party ID space is vendor-blocked (`0xC031` Zeiss, `0xC130` Tamron, `0xC230` Tokina,
`0xC530` Sigma, per exiftool's `LensType2` table). Yongnuo's `0xC540` sits inside **Sigma's** block
and appears **nowhere** in that table — an unassigned gap.

**And Yongnuo ships the same ID, 50496, for a 35 mm APS-C lens and a 50 mm full-frame lens.** Both
autofocus correctly. Two optically unrelated lenses claiming one unregistered ID cannot both be
receiving ID-keyed corrections, so **the correction data travels with the lens** — in messages 0x05
and 0x08 — not with the ID.

What does matter is the **range**. The A7 IV sorts IDs into Sony E-mount, third-party and neither,
and for a third-party ID overwrites much of 0x07, 0x08, 0x09 and 0x3D with fixed values. It also
special-cases individual Sony IDs. See [lens ID](lens_id.md).

Caveat: the `LensType2` table is community-compiled from observed EXIF, so absence proves no one has
published EXIF from that lens, not that Sony is unaware of the ID. And a body's internal correction
table is not a naming table. Strong inference, not proof.

## Manufacturer notes

- **Sony** — natives send `pl[0] = 01` and an E-mount ID in the `0x8xxx` range.
- **Yongnuo** — sends `pl[0] = 01`, presenting as a native, with one third-party ID reused across
  two optically unrelated lenses.
- **TECHART** — the LM-EA9 sends `pl[0] = 02` and legacy ID 234, and puts its own BCD firmware
  version in `pl[6]`.
- **Viltrox** — the EF adapters send `pl[0] = 02` and legacy ID 78, the same value regardless of
  which Canon lens is mounted.

## Open questions

- `pl[8]` = `0xA0`: a constant on every device, meaning UNKNOWN.
- `pl[15..18]` = `60 92 86 5e`: a protocol constant, meaning UNKNOWN.
- What the protocol-version grades unlock.
- Whether `pl[6]` is a version field on devices other than the LM-EA9.
