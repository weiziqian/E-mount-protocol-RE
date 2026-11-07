# Message 0x35 — lens state at the exposure, split-row form

**Summary.** The lens's state for the frame being exposed: subject distance, focal length, and one
[optical row pair](optical_data.md) with its long-form extensions. The body requests it during a
still capture, in place of [message 0x28](msg_0x28.md), when the lens qualifies (see
[which one the body asks for](msg_0x28.md#0x28-0x35-or-0x50--which-one-the-body-asks-for)). Its
content is a copy of the lens's current [message 0x05](msg_0x05.md) status.

**Direction:** body request, lens reply. **Class:** init (`0x02`).

**Length:** in the A7 IV length table, the request is 2 bytes counting the ID (`35` plus one
payload byte) and the reply is 40. The reply frame is 48 bytes, with 39 payload bytes `pl[0..38]`.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

Sources: the A7 IV body firmware (request and parser) and the Yongnuo 50 DF firmware (builder,
`FUN_08014e0c`). Both are CERTAIN as code. The meanings are graded per field.

---

## Request

| `pl` | Meaning |
| --- | --- |
| `0` bit 0 | Set by one requester, clear on the still-capture path. Meaning **UNKNOWN** |
| `0` bit 1 | Set when the lens's [0x08](msg_0x08.md) `pl[12]` bit 1 is set |
| `0` bit 2 | Always clear |

## Reply payload

The "Yongnuo" column is what that lens writes. The "A7 IV" column is what the body does with it.

| `pl` | Yongnuo | A7 IV | Confidence |
| --- | --- | --- | --- |
| **`0..1`** | **Subject distance**: 0x05 `pl[20..21]`, the [distance code](autofocus.md#25-the-distance-code) | Stored as the exposure's subject distance. On the adapter (type 5) path the body converts it with the distance-code mapping and uses it in its lens-correction lookup; status frames use 0x05 `pl[20..21]` there instead | **CERTAIN** |
| `2` | `00` | Not read | **CERTAIN** |
| `3` | 0x05 `pl[23]`, the coarse distance | Not read | **CERTAIN** |
| **`4..5`**, **`6..7`** | **Focal length** pair, mm × 10, as in 0x05 `pl[24..27]` | Stored as the focal length | **CERTAIN** |
| `8` | `01`, or `03` when an internal flag is clear | Three flags in bits 0, 1 and 2. Bit 1 is used only when 0x08 `pl[12]` bit 4 is set, and bit 2 only when the 0x08 `pl[8]` high nibble is 2. For a third-party lens ID the body forces bit 0 on and keeps only bits 1–2 | Reads **CERTAIN**; meaning **UNKNOWN** |
| `9..10` | An aperture value, written only when the body's 0x08 request asks for it ([below](#pl910)) | Filed in the slot where it files 0x05 `pl[30..31]` | **CERTAIN** |
| **`11..16`** | 0x05 slot A, `pl[32..37]` | Decoded as slot A | **CERTAIN** |
| `17..19` | 0x05 `pl[96..98]`, slot A's long-form extension | Slot A's extension | **CERTAIN** |
| **`20..25`** | 0x05 slot B, `pl[38..43]` | Decoded as slot B **only if it is a type-0 row** ([below](#slot-b)) | **CERTAIN** |
| `26..28` | 0x05 `pl[99..101]`, slot B's extension | Slot B's extension | **CERTAIN** |
| `29..30` | `00 00` | Replaces a value the body keeps itself, but only when the lens is native and 0x08 `pl[14]` bit 0 is set | Reads **CERTAIN**; meaning **UNKNOWN** |
| `31..32` | `pl[31]` is a computed byte, probably the coarse focus position | Filed as 0x05 `pl[54..55]`; used only if [0x3D](msg_0x3D.md) `pl[33]` bit 2 is set | Reads **CERTAIN**; Yongnuo meaning **PROBABLE** |
| `33..34` | 0x05 `pl[81..82]`, the effective focal length | Not read | **CERTAIN** |
| `35..38` | Zero | Not read; zeroed for a third-party lens ID | **CERTAIN** |

Message 0x28 carries the same content with the two rows contiguous. 0x35 adds the long-form
extensions, so it is the form for a lens that sends the 117-byte 0x05.

### `pl[9..10]`

The lens fills this field only when the body's
[0x08 request](msg_0x08.md#the-request-enables-an-optional-field-in-messages-0x28-and-0x35) asks for
it, exactly as in [message 0x28](msg_0x28.md#pl910-must-follow-the-bodys-request-and-must-not-be-volunteered).
Otherwise the field is zero. The A7 IV's request does not ask for it.

### Slot B

Yongnuo fetches the row pair with the parity argument fixed at 0, which selects the half that is
[normally type 1](optical_data.md#which-parity-holds-which-type-is-not-fixed). The A7 IV decodes
slot B only when byte 0 bit 7 is clear (type 0), and ignores a type-1 row. So the A7 IV usually
ignores slot B from a lens that follows Yongnuo.

**An all-zero row has bit 7 clear, so the body reads it as a type-0 row with value zero, not as an
absent row.** A lens should send either its real row or the same type-0 row it sends in 0x05.

## When the body sends it — CERTAIN on the A7 IV firmware

1. **Still capture.** The capture job "move iris" passes the target aperture to the lens layer. The
   lens layer may drive the iris first. It then sends the 0x35 (or 0x28) request.
2. **The reply is parsed.** The body compares the requested aperture with the aperture the lens
   reports, and drives the iris again if they differ.
3. **A second requester** sets request `pl[0]` bit 0. Its context is UNKNOWN.

## Example frame

This frame reports 54.5 / 55.0 mm, which suggests a Sony SEL5518Z:

```
F0 30 00 02 00 35 | 42 01 00 7F 21 02 26 02 FF 00 00 B0 C8 46 43 2E 0D 20 42 46 26 E4 F0 F0 24 08 F4 E8 EC 00 00 00 00 00 00 00 00 00 00 | 35 0B 55
```

| Field | Bytes | Value |
| --- | --- | --- |
| Subject distance | `42 01` | 322 = 0.51 m |
| Coarse distance | `7F` | |
| Focal length | `21 02` / `26 02` | 54.5 mm / 55.0 mm |
| `pl[8]` | `FF` | |
| Optical row A | `B0 C8 46 43 2E 0D` | |
| Extension | `20 42 46` | |
| Optical row B | `26 E4 F0 F0 24 08` | Type 0 |
| Extension | `F4 E8 EC` | |

## Manufacturer notes

| Device | Declares 0x35 (bit 52 of [0x01](msg_0x01.md)) |
| --- | --- |
| Yongnuo YN35mm f/1.8S DA, 50 DF | Yes |
| Sony SELP1650, SEL55210 | No |
| Viltrox EF adapters | No |
| TECHART LM-EA9 (stock) | No; it stores the frame above unchanged |

## Open questions

- The three flags in `pl[8]`, and the value the body keeps at `pl[29..30]`.
- What the second requester is.
