# Message 0x28 — a third carrier for the same correction rows

**Summary.** Focal length plus a 27-byte block of the focus-indexed correction data. Structurally
the twin of [message 0x35](msg_0x35.md), but it carries rows A and B **back to back**, making it the
cleanest carrier of the pair.

**Direction:** L→B.

**Class:** **init** (`0x02`).

**Frame length:** **44 bytes**.

**Never observed on the wire** — see below, which is itself informative.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

Byte offsets in the frame map below are **frame-relative** (`[0x00]` is the `0xF0` start byte).

## Construction

```c
frame[0..1] = F0, length 44
frame[3] = 2;  frame[5] = 0x28;             /* init class, ID */
pos = live_msg05_ladder_position();
write_focal_length(scratch);                /* same writer as msg 0x05 and msg 0x35 */
rec = record(pos, /* parity */ 0);          /* SAME focus-indexed record as msg 0x05 and 0x35 */
frame[0x0A..0x0D] = focal_pair;
memcpy(frame + 0x11, rec_scratch, 27);      /* 27 contiguous bytes */
if (flag) frame[0x0F..0x10] = pos;          /* live focus position, n x 256/3 */
```

## Frame map

| Bytes | Content |
| --- | --- |
| `[0x06..0x09]` | Derived block, same source as message 0x35's `[0x06..0x09]` |
| **`[0x0A..0x0D]`** | **Focal length pair**, two u16 LE, mm × 10 — identical encoding to 0x05 and 0x35 |
| `[0x0F..0x10]` | **[Live focus position](live_focus_position.md)** — the same value message 0x05 reports at `pl[0..1]` |
| **`[0x11..0x16]`** | **[Optical row A](optical_data.md#4-slot-a--the-field-sampling-grid)** |
| **`[0x17..0x1C]`** | **[Optical row B](optical_data.md#1-what-the-rows-carry)** — pupil size or pupil magnification, by [type flag](optical_data.md#34-byte-0-bit-7--the-row-type-flag--probable) |
| `[0x1D..0x2B]` | 15 further bytes of the same 27-byte block, not traced |

Unlike 0x05 (interleaved into slots) and 0x35 (split by three constant bytes), **message 0x28
carries rows A and B contiguously.**

## Three carriers, one table

The same per-focus-bucket record reaches the body through **three** messages:

| Message | Layout | Bytes of the record |
| --- | --- | --- |
| [0x05](msg_0x05.md) | Interleaved into slots, alternating parity | 12 + 1 |
| [0x35](msg_0x35.md) | Two halves split by three constant bytes | 18 |
| **0x28** | **Contiguous** | **27 — the largest copy** |

A device that populates one and not the others is inconsistent.

The rows themselves — encoding, meaning and sample data — are documented in
**[the 6-byte optical rows](optical_data.md)**. Note that message 0x28 always requests **parity 0**, so only
the even record of each focus bucket is reachable through it, and in a few buckets that record holds
the type-0 row rather than the type-1 one: see
[which parity holds which type](optical_data.md#which-parity-holds-which-type-is-not-fixed).

## Why this matters more than 0x35 does

| | [msg 0x35](msg_0x35.md) | **msg 0x28** |
| --- | --- | --- |
| TECHART LM-EA9 advertises it | **no** (bit 52 clear) | **yes** (bit 39 set) |
| LM-EA9 has a handler | **no** | **yes** |
| LM-EA9 payload | One frozen record, never sent | Near-zero, **and it is sent** |
| To populate it | Capability bit + dispatch entry + new handler + code | **A data edit** |

The LM-EA9's 0x28 response is

```
f0 00 00 02 00 28 | 00 07 00 ff f4 01 f4 01 | 01 00 00 | 00 00 09 00 00 00 22 00 …
```

The focal length at `[0x0A]`/`[0x0C]` is live, written every frame; `[0x11..0x2B]` is the 27-byte
correction block, which the adapter fills with mostly zeros.

Note `[0x01] = 0x00`: the length byte is a runtime placeholder, and the handler passes 44
explicitly. This is one of the messages whose header is filled at send time rather than a fixed
blob.

## Never observed — and that is itself informative

Checked exhaustively: **zero** message 0x28 frames across 3 393 A6000 frames and 6 572 NEX-7 / A7
frames. The IDs actually seen are `01 03 04 05 06 07 08 09 0a 0b 0c 0d 10` and
`03 04 05 06 07 08 09 0a 0b 0d 10 16 1d` respectively.

**Both bodies *offer* bit 39 and the Sony SELP1650 *asserts* it, yet neither body ever asks.**

**Whether a modern body ever asks is UNKNOWN.** Like message 0x35, this is a message with a
servable answer and no confirmed evidence that any real body — old or new — ever requests it.
Settling that needs an observation on a body that offers the newer PDAF tier, taken during actual
autofocus.

## Manufacturer notes

- **Sony** — the SELP1650 asserts bit 39, but no Sony body measured ever requests the message.
- **TECHART** — the LM-EA9 asserts it, has a handler, and answers with a near-zero correction block
  and a live focal length.
- **Yongnuo** — builds it live from the same focus-indexed record as messages 0x05 and 0x35.

## Open questions

- Whether any body ever sends the request.
- What `[0x06..0x09]` carries.
- What the remaining 15 bytes of the 27-byte block at `[0x1D..0x2B]` are.
