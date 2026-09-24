# Message 0x04 — body state and focus command, per frame

**Summary.** The channel the body commands focus on. A fixed 13-byte header of body state, sent
once per 60 Hz frame, optionally followed by one or more **tagged records**; the `0x1D` record
carries the focus target.

What each record asks the lens to do, and how the lens reports back, is described in
[How auto focus works](autofocus.md).

**Direction:** B→L only.

**Class:** normal (`0x01`).

**Payload:** 13 bytes with no records; 14, 17, 18 and 30 bytes observed with records.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

## Payload structure

```
pl[0..12]   header, always present
pl[13..]    zero or more records, each: 1 tag byte + a fixed number of operand bytes
```

Records do not extend the message — they **replace** the bare 13-byte form. Over one 10-second
session with a Sony a9 II: 595 frames of 13 bytes, 5 of 14, 5 of 18, against 605 frames of
[message 0x03](msg_0x03.md). `595 + 5 + 5 = 605`, so exactly one 0x04 accompanies each 0x03.

A receiver must accept every length. One that recognises only the 13-byte form discards every focus
command the body issues.

## Header — `pl[0..12]`

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..5]` | Fixed prefix `00 00 19 83 00 00`, identical on every frame of every body and device observed, at every length | Constant **CERTAIN**; meaning **UNKNOWN** |
| `pl[6]` | Constant within a session. Observed: `0x3B`, `0x28`, `0x21`, `0x18` | **UNKNOWN** |
| `pl[7]` | `0x1F` to devices reporting an E-mount lens ID, `0x00` to devices reporting a legacy ID | Discriminates device class **PROBABLE**; quantity **UNKNOWN** |
| `pl[8..9]` | Zero in every frame observed | **UNKNOWN** |
| `pl[10]` | State code. Static within a session from some bodies (`0x31`, `0x01`); varying from others — `0x09` while idle, `0x00`, `0x01`, `0x08`, `0x40`, `0x41` otherwise | A state code **PROBABLE**; the states **UNKNOWN** |
| `pl[11..12]` | Zero in every frame observed | **UNKNOWN** |

## Records — `pl[13]` onward

| Tag | Total size | Operands |
| --- | --- | --- |
| `0x1C` | 1 B | none |
| `0x1D` | 5 B | focus target, u16 LE; then a second u16 LE, zero in every frame observed |
| `0x1F` | 14 B | 13 operand bytes, meaning **UNKNOWN** |
| `0x2F` | 3 B | a row-index pair, the same operand shape [message 0x03](msg_0x03.md) carries at `pl[20..22]` |

The sizes are fixed per tag and account for every observed payload length exactly:

| Payload | Header | Records |
| --- | --- | --- |
| 13 | 13 | — |
| 14 | 13 | `0x1C` |
| 17 | 13 | `0x2F` + `0x1C` |
| 18 | 13 | `0x1D` |
| 30 | 13 | `0x2F` + `0x1F` |

`0x1C` also appears as an operation code in [message 0x06](msg_0x06.md)'s event appendix. Whether
that is one instruction namespace shared across messages, or reuse of a byte value, is **UNKNOWN**.

## The focus target — the `0x1D` record

A u16 little-endian absolute position, on the scale [message 0x06](msg_0x06.md) reports focus in —
not the [aperture value](aperture_value.md) scale that the neighbouring fields of
[message 0x03](msg_0x03.md) use, though the two ranges overlap.

Observed from a Sony a9 II driving a device that advertises a travel of 4144…5632: targets spread
across that travel — 4314, 4381, 4593, 4646, 4844, 4942, 5185, 5439 and the travel limit 5632 among
them — plus one value far outside it:

| Value | Reading | Confidence |
| --- | --- | --- |
| 4144…5632 | An absolute focus position inside the advertised travel | **PROBABLE** |
| **`0x7FFF` = 32767** | No-target sentinel | **PROBABLE** |

The second u16 of the record, immediately after the target, has been zero in every frame observed,
so its role — a second target, a limit, a rate — is **UNKNOWN**.

## Observed frames

Header and trailer included, payload between the bars.

```
Sony A6000 to Sony SELP1650 / SEL55210, every frame of the session
F0 16 00 01 xx 04 | 00 00 19 83 00 00 3B 1F 00 00 31 00 00                                                    | .. .. 55

Sony A6000 to a Viltrox EF adapter, every frame of the session
F0 16 00 01 xx 04 | 00 00 19 83 00 00 3B 00 00 00 01 00 00                                                    | .. .. 55

Sony a9 II, idle
F0 16 00 01 xx 04 | 00 00 19 83 00 00 28 00 00 00 09 00 00                                                    | .. .. 55

Sony a9 II, 0x1C record
F0 17 00 01 xx 04 | 00 00 19 83 00 00 28 00 00 00 41 00 00 1C                                                 | .. .. 55

Sony a9 II, 0x1D record carrying target 0x12EC = 4844
F0 1B 00 01 82 04 | 00 00 19 83 00 00 28 00 00 00 40 00 00 1D EC 12 00 00                                     | C1 02 55

Sony a9 II, 0x1D record carrying the 0x7FFF sentinel
F0 1B 00 01 A1 04 | 00 00 19 83 00 00 28 00 00 00 41 00 00 1D FF 7F 00 00                                     | 61 03 55

To a Yongnuo YN35mm f/1.8S, 0x2F + 0x1C
F0 1A 00 01 E8 04 | 00 00 19 83 00 00 18 00 00 00 09 00 00 2F 00 00 1C                                        | 0F 02 55

To a Yongnuo YN35mm f/1.8S, 0x2F + 0x1F
F0 27 00 01 CB 04 | 00 00 19 83 00 00 21 00 00 00 48 00 00 2F 09 0A 1F 02 02 83 88 3D 49 ED 47 3F 46 03 00 00 | AE 05 55
F0 27 00 01 DF 04 | 00 00 19 83 00 00 18 00 00 00 48 00 00 2F 07 08 1F 06 02 83 98 7D 45 F5 43 10 52 03 00 00 | E6 05 55
```

## Open questions

- The meaning of the fixed prefix `00 00 19 83 00 00`.
- `pl[6]`, `pl[7]`, `pl[10]` — values differ by body and by device class, but the quantities are
  unestablished.
- The 13 operand bytes of the `0x1F` record.
- The second u16 of the `0x1D` record, zero in every frame observed.
- Whether tag values beyond `0x1C`, `0x1D`, `0x1F` and `0x2F` exist, and whether more than two
  records can appear in one frame.
