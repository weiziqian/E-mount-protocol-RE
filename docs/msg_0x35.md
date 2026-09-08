# Message 0x35 — a second carrier for the focus-indexed correction rows

**Summary.** Delivers the same focus-indexed row data that message 0x05 streams, on a
request/response channel instead of in the status loop, together with the focal length.

**Direction:** L→B.

**Class:** **init** (`0x02`) — despite carrying live status data.

**Frame length:** **48 bytes**, fixed.

**Never observed on the wire.** No body measured offers the ID.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

Byte offsets in the frame map below are **frame-relative** (`[0x00]` is the `0xF0` start byte), not
payload-relative, because the interesting fields straddle the header.

## It is a response, not a status packet — but it is rebuilt live every time

Three facts, and they pull in different directions:

1. The class byte is written as a literal `2`, so it is **init class** — like 0x01 / 0x07 / 0x08 and
   unlike the 0x05 / 0x06 status pair.
2. It is built **inside the inbound dispatcher**, so the lens emits it only when the body sends an
   0x35 request. It is not in the ~60 Hz loop.
3. But unlike every other init-class message, it is **not a fixed blob**. Messages 0x01, 0x07, 0x08,
   0x09, 0x0D, 0x10, 0x16, 0x3D and 0x4C are copied out of a fixed table; 0x35 is **computed from
   scratch on each request**, reading the [live focus position](live_focus_position.md) and the live
   velocity byte.

Point 3 is why "asked once at init" is unlikely: nobody writes a live builder for a constant. The
honest statement is that the implementation is **prepared to be asked at any time and to answer with
current data**, and that how often a real body actually asks is **UNKNOWN**.

## Frame map — 48 bytes

| Bytes | Len | Content | Source |
| --- | --- | --- | --- |
| `[0x00]` | 1 | `F0` | Framing |
| `[0x01..0x02]` | 2 | `30 00` = 48 | Framing |
| `[0x03]` | 1 | `02` — init class | Literal |
| `[0x04]` | 1 | Sequence | Framing |
| `[0x05]` | 1 | `35` — message ID | Literal |
| `[0x06..0x09]` | 4 | Derived from `steps − (x + y)` | Live |
| `[0x08]` | 1 | Forced to `0` after the above write | — |
| **`[0x0A..0x0D]`** | **4** | **Focal length pair, two u16 LE, mm × 10** | Same writer as msg 0x05 |
| `[0x0E]` | 1 | `1`, or `3` when a GPIO bit is set | Live |
| `[0x0F..0x10]` | 2 | A position delta, on the [live focus position](live_focus_position.md) scale — message 0x28 carries the position itself at the same offset | Live, conditional |
| **`[0x11..0x16]`** | **6** | **[Optical row A](optical_data.md#4-slot-a--the-field-sampling-grid)** | Focus-indexed table |
| `[0x17..0x19]` | 3 | Secondary block bytes 13–15 | Constant block |
| **`[0x1A..0x1F]`** | **6** | **[Optical row B](optical_data.md#1-what-the-rows-carry)** — pupil size or pupil magnification, by [type flag](optical_data.md#34-byte-0-bit-7--the-row-type-flag--probable) | Same record |
| `[0x20..0x22]` | 3 | Secondary block bytes 16–18 | Constant block |
| `[0x23..0x24]` | 2 | Zero | — |
| `[0x25]` | 1 | The same velocity/delta byte msg 0x05 carries at its `[0x3C]` | Live |
| `[0x26]` | 1 | Zero | — |
| `[0x27..0x28]` | 2 | Copied from a scratch frame; **not traced** | — |
| `[0x29..0x2C]` | 4 | Zero | — |
| `[0x2D..0x2E]` | 2 | Checksum | Framing |
| `[0x2F]` | 1 | `55` | Framing |

Two things to note about the 18-byte optical region:

- It is **interleaved**, not contiguous: `[rowA 6][secondary 3][rowB 6][secondary 3]`.
- The six "secondary" bytes are **not** focus-indexed. They come from a selector that returns one of
  two fixed 25-byte blocks — and on the lens checked both blocks are byte-identical, so the
  selection is a no-op there. They are zero on the YN35 and **non-zero on the TECHART LM-EA9**, so
  the field is real and that particular lens simply does not use it.

## The row data is the same table message 0x05 uses

Both builders call the same selector:

```c
record(pos, parity) {
    k = index(pos);                          /* focus bucket */
    return TABLE + (parity ? (2*k + 1)*14 : k*28);   /* record 2k or 2k+1 */
}

index(pos) {
    v = (pos * 3)/256;
    if (v < 53) v = 53;
    return v - 53;
}
```

`(pos × 3)/256` is exactly the grid index `n` of message 0x05's `n × 256/3` focus scale. **So the
correction rows are indexed by the focus position, on the protocol's own distance grid**, one bucket
per grid point. That is the mechanism behind the wire observation that "slot A changes only when
focus moves".

The table is 42 records × 14 bytes (`[rowA 6][rowB 6][pad 2]`), paired: record `2k` and `2k+1` share
a row A and differ in row B. **Message 0x05 alternates between the pair by frame parity; message
0x35 always passes parity 0**, so 0x35 can only ever emit the even record of a bucket — 21 distinct
values, against 42 for 0x05.

Message 0x05 puts the same 12 bytes at `pl[32..43]` plus the 13th at `pl[44]`; message 0x35 splits
the same 12 into two halves with three constant bytes between them. **The two messages are two views
of one table.**

The rows themselves — encoding, meaning and sample data — are documented in
**[the 6-byte optical rows](optical_data.md)**. Because 0x35 always passes parity 0, a few buckets deliver the
type-0 row and no aperture profile at all: see
[which parity holds which type](optical_data.md#which-parity-holds-which-type-is-not-fixed).

## The focal length appears here too — and it can disagree with message 0x05

One writer, called by **both** the 0x05 and 0x35 builders, emits the focal pair as two immediate
u16 values (358 and 350 on the YN35) plus a third constant (310). In message 0x35 that pair lands at
`[0x0A..0x0D]`.

So **message 0x35 carries a focal length in the same mm × 10 encoding as message 0x05's
`pl[24..25]`/`pl[26..27]`**, and a conforming device must keep the two consistent. The TECHART
LM-EA9 does not.

## Who supports it

| Device | Asserts bit 52 (ID 0x35) | Builds a packet | Handles an inbound request |
| --- | --- | --- | --- |
| Yongnuo YN35mm F1.8S DA | **yes** | Builds it live | Yes |
| Sony SELP1650, SEL55210 (A6000) | no | — | — |
| Viltrox EF adapters | no | — | — |
| **TECHART LM-EA9** | **no** | 48 static bytes | **No** |

Yongnuo is the only device measured that asserts 0x35, consistent with its mask being the richest
measured (42 IDs). Body offers do cover the bit: bit 52 falls in byte 6 of the mask, which is `0xff`
in both the NEX-7 and A6000 offers.

## Manufacturer notes

- **Yongnuo** — the only device measured that asserts and implements 0x35, and it rebuilds the
  message live on every request.
- **TECHART** — the LM-EA9 does not assert the bit and has no handler, but carries a static 48-byte
  response with the secondary block populated.
- **Sony**, **Viltrox** — do not assert the ID.

## Open questions

- How often, if ever, a real body requests it. Nothing has been observed.
- What `[0x06..0x09]` and `[0x27..0x28]` carry.
- Why 0x35 is init class while carrying live data.
- Why it only ever emits the even record of a bucket, where message 0x05 sends both.
