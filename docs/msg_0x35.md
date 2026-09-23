# Message 0x35 — focal length and a split pair of optical rows

**Summary.** Carries the focal length and two [6-byte optical rows](optical_data.md), the rows split
by three bytes apiece. The same content [message 0x28](msg_0x28.md) carries contiguously.

**Direction:** L→B.

**Class:** init (`0x02`).

**Frame length:** 48 bytes — 39 payload bytes, `pl[0..38]`.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

## Response payload

| `pl` | Size | Name | Description | Confidence |
| --- | --- | --- | --- | --- |
| `0..3` | 4 | — | Purpose unknown | **UNKNOWN** |
| **`4..5`** | 2 | **Focal length** | u16 LE, mm × 10. Same encoding as [message 0x05](msg_0x05.md) `pl[24..25]` | **CERTAIN** |
| **`6..7`** | 2 | **Focal length** | u16 LE, mm × 10. The second of the pair; equal to `pl[4..5]` on a prime | **CERTAIN** |
| `8` | 1 | — | Purpose unknown. Observed values: `0x01`, `0x03`, `0xFF` | **UNKNOWN** |
| `9..10` | 2 | — | A value on the [aperture value](aperture_value.md) scale. [Message 0x28](msg_0x28.md) carries the aperture itself at the same offset | **POSSIBLE** |
| **`11..16`** | 6 | **Optical row A** | [Slot A](optical_data.md#4-slot-a--the-field-sampling-grid) | **CERTAIN** |
| `17..19` | 3 | Secondary block | Not part of the optical row pair. Zero on some devices, non-zero on others | Existence **CERTAIN**; purpose **UNKNOWN** |
| **`20..25`** | 6 | **Optical row B** | [Slot B](optical_data.md#1-what-the-rows-carry) | **CERTAIN** |
| `26..28` | 3 | Secondary block | The continuation of `pl[17..19]` | Existence **CERTAIN**; purpose **UNKNOWN** |
| `29..38` | 10 | — | Zero in every payload seen | **UNKNOWN** |

The optical region is **interleaved**: `[row A, 6][secondary, 3][row B, 6][secondary, 3]`. The six
secondary bytes are not optical-row data and do not track the row index.

A device that carries the rows in this message and not in [0x05](msg_0x05.md) or
[0x28](msg_0x28.md) is inconsistent; so is one whose focal length here disagrees with message
0x05's.

## Example frame

```
F0 30 00 02 00 35 | 42 01 00 7F 21 02 26 02 FF 00 00 B0 C8 46 43 2E 0D 20 42 46 26 E4 F0 F0 24 08 F4 E8 EC 00 00 00 00 00 00 00 00 00 00 | 35 0B 55
```

| Field | Bytes | Value |
| --- | --- | --- |
| Focal length | `21 02` / `26 02` | 545 / 550 = 54.5 mm / 55.0 mm |
| Optical row A | `B0 C8 46 43 2E 0D` | |
| Secondary | `20 42 46` | |
| Optical row B | `26 E4 F0 F0 24 08` | |
| Secondary | `F4 E8 EC` | |

## Who requests it

Bit 52 of the [capability bitmap](msg_0x01.md) covers this ID. Every body measured offers it; **no
body has been observed requesting it**, and nothing this message carries has been traced to any
effect on a body. When, or whether, a body asks for it is unestablished.

[Message 0x28](msg_0x28.md) carries the same focal length and the same optical row pair, and that
one *is* requested — at the shutter press — so the content is reachable by another route.

| Device | Asserts bit 52 |
| --- | --- |
| Yongnuo YN35mm f/1.8S DA | yes |
| Sony SELP1650, SEL55210 | no |
| Viltrox EF adapters | no |
| TECHART LM-EA9 | no |

## Open questions

- Whether any body requests the message.
- `pl[0..3]`, `pl[8]`, `pl[29..38]`, and the six secondary bytes at `pl[17..19]` / `pl[26..28]`.
- Whether both records of an optical-row pair are reachable through this message.
- Why an init-class message carries data that changes with the lens's state.
