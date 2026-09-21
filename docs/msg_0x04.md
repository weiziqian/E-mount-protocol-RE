# Message 0x04 — body mode block and focus target, per frame

**Summary.** A per-frame body→lens block. It has **several payload lengths**: a short form that is
static through idle, and longer forms selected by a tag byte, one of which carries the **focus
target** the body wants the lens to move to. Frame 4 of the four-frame 60 Hz loop.

**Direction:** B→L only.

**Class:** normal (`0x01`).

**Payload:** 13 bytes on a Sony A6000. A Sony a9 II sends 13, 14 or 18 bytes depending on state.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## The forms replace one another — they are not extra frames

Over one 10-second session with a Sony a9 II, with the shutter half-pressed a few times:

| Message | Payload | Frames |
| --- | --- | --- |
| 0x03 | 20 B | 605 |
| **0x04 short** | **13 B** | **595** |
| **0x04 tagged** | **14 B** | **5** |
| **0x04 target** | **18 B** | **5** |

`595 + 5 + 5 = 605`, exactly the 0x03 count. The body sends **one 0x03 and one 0x04 per frame**,
and the longer 0x04 forms **replace** the short one rather than being sent alongside it. CERTAIN —
the arithmetic closes to the frame.

So a receiver must accept every length: a device that only recognises the 13-byte form will silently
discard every focus command the body issues.

## Observed payloads

| Body | Device | Payload |
| --- | --- | --- |
| A6000 | Sony SELP1650, SEL55210 | `00 00 19 83 00 00 3B 1F 00 00 31 00 00` |
| A6000 | Viltrox + Canon EF-S 24 | `00 00 19 83 00 00 3B 00 00 00 01 00 00` |
| a9 II | TECHART LM-EA9, idle | `00 00 19 83 00 00 28 00 00 00 09 00 00` |
| a9 II | TECHART LM-EA9, tagged | `00 00 19 83 00 00 28 00 00 00 41 00 00 1C` |
| a9 II | TECHART LM-EA9, target | `00 00 19 83 00 00 28 00 00 00 40 00 00 1D F1 11 00 00` |
| a9 II | TECHART LM-EA9, target | `00 00 19 83 00 00 28 00 00 00 41 00 00 1D FF 7F 00 00` |

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..5]` = `00 00 19 83 00 00` | **Fixed prefix.** Identical on every frame of every body and device observed, at every length. | **CERTAIN** (as a constant); meaning UNKNOWN |
| `pl[6]` | `0x3B` from the A6000, `0x28` from the a9 II. Constant within a session. | UNKNOWN |
| `pl[7]` | `0x1F` to natives, `0x00` to adapters | UNKNOWN — but it discriminates device class |
| `pl[8..9]` | zero | UNKNOWN |
| `pl[10]` | **Mode byte.** `0x31` A6000→native, `0x01` A6000→Viltrox. On the a9 II it is **not static**: `0x09` while idle, `0x01`/`0x40`/`0x41` on the frames that carry a tag. | PROBABLE (a mode or state code) |
| `pl[11..12]` | zero | UNKNOWN |
| **`pl[13]`** | **Tag byte — present only in the longer forms.** Selects what follows. See below. | **PROBABLE** |
| **`pl[14..15]`** | **Focus target**, u16 LE, when `pl[13] = 0x1D` | **PROBABLE** |
| `pl[16..17]` | Second 16-bit field alongside the target. Zero in every frame observed. | UNKNOWN |

## The tag byte at `pl[13]`

The 13-byte form has no `pl[13]`. Every longer form begins with a tag there, and the tag selects
the operands that follow:

| `pl[13]` | Payload length | What follows | Seen from |
| --- | --- | --- | --- |
| — | 13 | nothing; this is the short form | A6000, a9 II |
| `0x1C` | 14 | nothing | a9 II; Yongnuo |
| **`0x1D`** | **18** | **two u16 LE fields: the focus target and a second, always-zero field** | a9 II |
| `0x2F` | 17, 30 | a row-index pair and further operands — the same tag and operand shape message 0x03 uses at `pl[20..22]` | Yongnuo |

PROBABLE that this is one tagged-operand namespace rather than three unrelated coincidences: the
`0x2F` case is handled as "row-index select, two operand bytes follow" by an implementation that
also drives message 0x05's index field from it, and it appears at the same relative position.

`0x1C` also appears as an operation code in [message 0x06](msg_0x06.md)'s event appendix. Whether
that is one instruction namespace shared across messages or reuse of a byte value is UNRESOLVED.

## The focus target — `pl[14..15]`

Present when `pl[13] = 0x1D`. A u16 little-endian value in the same numeric space the lens uses to
report its position.

Observed from a Sony a9 II driving a TECHART LM-EA9, which advertises a travel of 4144…5632 and
reports its own position as 4864:

| Value | Reading |
| --- | --- |
| `0x12EC` = 4844 | inside the advertised travel |
| `0x11F1` = 4593 | inside the advertised travel |
| **`0x7FFF` = 32767** | **no-target sentinel** — far outside any travel a lens advertises |

Two distinct in-range values rule out a constant. PROBABLE that this is the commanded absolute
focus position; the sentinel reading of `0x7FFF` is PROBABLE on the same evidence.

`pl[16..17]` sits immediately after it and has been zero in every frame seen, so its role — a second
target, a limit, a velocity — is UNKNOWN.

### It is not quantised like the reported position

The [live focus position](live_focus_position.md) advances in steps of `256/3` = 85.333…, and the
values in message 0x03's position fields land exactly on that grid. **The `pl[14..15]` targets do
not**:

| Value | `× 3 / 256` | On the grid? |
| --- | --- | --- |
| 4096 (message 0x03) | 48.0 | yes |
| 4608 (message 0x03) | 54.0 | yes |
| 4864 (lens's reported position) | 57.0 | yes |
| **4844** (`pl[14..15]`) | 56.77 | **no** |
| **4593** (`pl[14..15]`) | 53.82 | **no** |

So the target is expressed in the same numeric range but at finer resolution than the report.
Whether that is a genuinely finer scale or a different quantity that merely overlaps the range is
an open question. POSSIBLE.

## Longer forms

Yongnuo's implementation carries 0x04 forms with payloads of 17 and 30 bytes, fully framed, tagged
`0x2F`. Shown whole, header and trailer included, with the **payload** between the bars:

```
F0 27 00 01 CB 04 | 00 00 19 83 00 00 21 00 00 00 48 00 00 2F 09 0A 1F 02 02 83 88 3D 49 ED 47 3F 46 03 00 00 | AE 05 55
F0 27 00 01 DF 04 | 00 00 19 83 00 00 18 00 00 00 48 00 00 2F 07 08 1F 06 02 83 98 7D 45 F5 43 10 52 03 00 00 | E6 05 55
F0 1A 00 01 E8 04 | 00 00 19 83 00 00 18 00 00 00 09 00 00 2F 00 00 1C                                        | 0F 02 55
F0 17 00 01 00 04 | 00 00 19 83 00 00 21 00 00 00 08 00 00 1C                                                 | FD 00 55
```

The fixed prefix and the tag position survive into all of them. The fourth line is the same 14-byte
`0x1C`-tagged form the a9 II sends.

### The `0x2F` forms sample every branch of the row-index dispatch

Three of them carry `idx_lo` values `9`, `7` and `0` — the "retransmission" (`7`, `9`) and "nothing
new" (`0`) branches of the row-index dispatch described in
[message 0x03](msg_0x03.md#0x2f-is-an-instruction-tag). With the `0x15`/`0x16` pair on the
corresponding 0x03 form, they cover every branch of that dispatch between them.

## Manufacturer notes

- **Sony A6000** — sends only the 13-byte form, byte-identical on every frame of a session. Uses
  `pl[7] = 0x1F`, `pl[10] = 0x31` to natives and `pl[7] = 0x00`, `pl[10] = 0x01` to the Viltrox EF
  adapter; both bytes discriminate device class. **It never sends a focus target in this message.**
- **Sony a9 II** — sends all three lengths, and treats the TECHART LM-EA9 as adapter class
  (`pl[7] = 0x00`). `pl[10]` varies with state rather than being session-static.
- **Yongnuo** — carries the `0x2F`-tagged 17- and 30-byte forms.

## Open questions

- The meaning of the fixed prefix `00 00 19 83 00 00`. UNKNOWN.
- `pl[6]`, `pl[7]`, `pl[10]`: values differ by body and device class, but the quantities are
  UNKNOWN.
- `pl[16..17]` — always zero in every frame observed, so its role is unestablished.
- Why the target at `pl[14..15]` is not on the `256/3` grid the position report uses.
- Whether further tag values and longer target-carrying forms exist. Only `0x1C`, `0x1D` and
  `0x2F` have been seen.
- Whether `0x1C` is shared with message 0x06's event appendix or a coincidence.
