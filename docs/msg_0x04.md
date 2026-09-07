# Message 0x04 — body mode block, per frame

**Summary.** A per-frame body→lens block that is static within a session. Frame 4 of the four-frame
60 Hz loop, and the shortest of them. Its purpose is UNKNOWN.

**Direction:** B→L only.

**Class:** normal (`0x01`).

**Payload:** 13 bytes on the Sony A6000.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Device | Payload |
| --- | --- |
| Sony SELP1650, SEL55210 | `00 00 19 83 00 00 3b 1f 00 00 31 00 00` |
| Viltrox + Canon EF-S 24 | `00 00 19 83 00 00 3b 00 00 00 01 00 00` |

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..5]` = `00 00 19 83 00 00` | **Fixed prefix.** Identical on every A6000 frame, and on the longer Yongnuo forms below. | **CERTAIN** (as a constant); meaning UNKNOWN |
| `pl[6]` | `0x3b` on both natives and the adapter | UNKNOWN |
| `pl[7]` | `0x1f` native / `0x00` adapter | UNKNOWN — but it discriminates |
| `pl[10]` | `0x31` native / `0x01` adapter | UNKNOWN — but it discriminates |
| rest | zero | UNKNOWN |

**Byte-identical on every frame within a device.** On the A6000 it is session-static, and whatever
it carries does not change during idle.

## The layout is extensible — longer forms exist

Yongnuo's implementation carries 0x04 forms with payloads of **23–39 bytes** rather than 13, fully
framed. These have not been observed on the wire, but they show the layout is an extensible one
rather than a fixed 13-byte block. Shown here whole, header and trailer included, with the payload
between the bars:

```
f0 27 00 01 cb 04 | 00 00 19 83 00 00 21 00 00 00 48 00 00 2f 09 0a 1f 02 02 83 88 3d 49 ed 47 3f 46 03 00 00 | ae 05 55
f0 27 00 01 df 04 | 00 00 19 83 00 00 18 00 00 00 48 00 00 2f 07 08 1f 06 02 83 98 7d 45 f5 43 10 52 03 00 00 | e6 05 55
f0 1a 00 01 e8 04 | 00 00 19 83 00 00 18 00 00 00 09 00 00 2f 00 00 1c 0f 02 | 55
f0 17 00 01 00 04 | 00 00 19 83 00 00 21 00 00 00 08 00 00 1c fd 00 | 55
f0 20 00 01 00 03 | 42 4e 00 00 11 00 11 1c 00 00 02 01 01 02 00 02 01 00 00 00 2f 15 16 55 01 | 55
```

Two things follow. The prefix `00 00 19 83 00 00` and the `0x2f` tag both survive into the much
longer form. And the 0x03 form in the last line ends `2f 15 16 55 01` — `0x2f` plus the row-index
pair, exactly as seen on the A6000 wire (see [message 0x03](msg_0x03.md)).

### The forms sample every branch of the row-index dispatch

Three of the 0x04 forms carry `idx_lo` values `9`, `7` and `0` — exactly the "retransmission"
(`7`, `9`) and "nothing new" (`0`) branches of the row-index dispatch described in
[message 0x03](msg_0x03.md#0x2f-is-an-instruction-tag). Combined with the `0x15`/`0x16` pair on the
0x03 form (a "real fetch" tag), these five forms between them cover every branch of that dispatch.
That reads less like arbitrary padding and more like a deliberately chosen set of test or default
cases.

The fourth 0x04 form breaks the pattern: no `0x2f` tag at all, ending `1c fd 00` instead. `0x1c` is
also the code that message 0x06's event appendix uses for "force parameter to 1". Whether that is
the same instruction namespace reused across messages, or coincidental reuse of one byte value, is
UNRESOLVED — a lead, not a finding.

## Manufacturer notes

- **Sony** — the A6000 sends the 13-byte form with `pl[7] = 0x1f`, `pl[10] = 0x31` to natives.
- **Viltrox** — the same body sends the same length to the EF adapter but with `pl[7] = 0x00`,
  `pl[10] = 0x01`. Both bytes discriminate native from adapter.
- **Yongnuo** — carries the longer 23–39 byte forms shown above.

## Open questions

- The meaning of the whole block, including the fixed prefix. UNKNOWN.
- `pl[6]`, `pl[7]`, `pl[10]`: values differ by device class but the quantity is UNKNOWN.
- Why a lens carries body-direction frames at all. Traces kept as a test fixture, and expected
  command templates used for matching, are both plausible.
- Whether `0x1c` is shared with message 0x06's event appendix or a coincidence.
