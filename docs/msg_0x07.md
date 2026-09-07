# Message 0x07 — identity and lens ID

**Summary.** Lens identification. The second exchange of the init phase, immediately after the
capability bitmap.

**Direction:** both. The body request is 1 byte (`00`); the lens reply is 34 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Field map — lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | **`01` on every native, `02` on every adapter** | Discriminator **CERTAIN**; meaning PROBABLE (device class) |
| `pl[2]` | `35` SEL55210, `20` SELP1650, `60` on another native, `08` LM-EA9 and Viltrox | UNKNOWN |
| `pl[6]` | **BCD firmware version on the TECHART LM-EA9** (`0x18` = 1.8) | CERTAIN for the LM-EA9 |
| `pl[8]` | `0xA0` on **every** device measured | CERTAIN constant, meaning UNKNOWN |
| **`pl[9..10]`** | **u16 LE Sony lens ID** | **CERTAIN** |
| `pl[15..18]` | `60 92 86 5e` on the LM-EA9, on a third-party native, and on both Yongnuo lenses | **Protocol constant** — CERTAIN |

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

What may still matter is the **range** — E-mount space (≥ `0x8000`) versus the adapters' legacy
range — not the value. That is the hypothesis worth testing; the value itself is not.

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

- `pl[2]`: differs per device, quantity UNKNOWN.
- `pl[8]` = `0xA0`: a constant on every device, meaning UNKNOWN.
- `pl[15..18]` = `60 92 86 5e`: a protocol constant, meaning UNKNOWN.
- Whether a body gates behaviour on the ID *range* (E-mount vs legacy) rather than the value.
- Whether `pl[6]` is a version field on devices other than the LM-EA9.
