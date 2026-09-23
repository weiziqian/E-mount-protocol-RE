# Message 0x28 — lens state, requested at the exposure

**Summary.** Carries the focal length, the current [aperture value](aperture_value.md), and two
[6-byte optical rows](optical_data.md) back to back. The body requests it at the shutter press, and
**the focal length it returns is the one written to the photograph's EXIF**.

**Direction:** L→B.

**Class:** init (`0x02`).

**Frame length:** 44 bytes — 35 payload bytes, `pl[0..34]`.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

## Response payload

| `pl` | Size | Name | Description | Confidence |
| --- | --- | --- | --- | --- |
| `0..3` | 4 | — | Purpose unknown | **UNKNOWN** |
| **`4..5`** | 2 | **Focal length** | u16 LE, mm × 10. Same encoding as [message 0x05](msg_0x05.md) `pl[24..25]` | **CERTAIN** |
| **`6..7`** | 2 | **Focal length** | u16 LE, mm × 10. The second of the pair; equal to `pl[4..5]` on a prime | **CERTAIN** |
| `8` | 1 | — | Purpose unknown. Observed value: `0x01` | **UNKNOWN** |
| `9..10` | 2 | Aperture | u16 LE [aperture value](aperture_value.md), the same quantity message 0x05 reports at `pl[0..1]` | **PROBABLE** |
| **`11..16`** | 6 | **Optical row A** | [Slot A](optical_data.md#4-slot-a--the-field-sampling-grid) | **CERTAIN** |
| **`17..22`** | 6 | **Optical row B** | [Slot B](optical_data.md#1-what-the-rows-carry) | **CERTAIN** |
| `23..34` | 12 | — | Purpose unknown | **UNKNOWN** |

The two optical rows are **contiguous** here. [Message 0x05](msg_0x05.md) interleaves them into
separate slots and [message 0x35](msg_0x35.md) splits them with three bytes between, so this is the
layout in which the pair is easiest to read.

A device that carries the rows in one of these three messages and not the others is inconsistent;
so is one whose focal length here disagrees with message 0x05's.

## Example frame

A TECHART LM-EA9, which carries a focal length and leaves the optical rows nearly empty. The length
field is `00 00` and the checksum `00 00`; both are filled in at send time.

```
F0 00 00 02 00 28 | 00 07 00 FF F4 01 F4 01 01 00 00 00 00 09 00 00 00 22 00 00 00 00 00 16 00 0E 00 56 00 00 16 50 00 00 00 | 00 00 55
```

`pl[4..5]` and `pl[6..7]` are both `F4 01` = 500 = 50.0 mm.

## Who requests it

Bit 39 of the [capability bitmap](msg_0x01.md) covers this ID.

| Body | Offers bit 39 | Requests the message |
| --- | --- | --- |
| Sony a9 II | yes | **yes — twice in one session** |
| Sony A6000 | yes | no, in 3393 frames |
| Sony NEX-7, A7 | yes | no, in 6572 frames |

On the a9 II both requests arrive **at the shutter press**, inside a burst of init-class requests —
`0x0A 0x0B 0x1B 0x28 0x19 0x19 0x28 0x0B 0x0A` — and not during the init handshake or the idle loop.
The older bodies offer the ID and never ask for it.

## The focal length in this message is the one written to EXIF

A device reporting **40.0 mm** in [message 0x05](msg_0x05.md) `pl[24..25]` and **50.0 mm** here
produced photographs tagged **50 mm** on a Sony a9 II. The same device changed to report 52.0 mm in
both messages produced photographs tagged **52 mm**.

The two messages disagreeing is what makes the first observation decisive: the body took the focal
length from this message. Confidence **CERTAIN** for that body; whether every body does is
**UNKNOWN**.

A device whose focal length differs between messages 0x05, 0x28 and 0x35 will therefore be described
one way in the camera's display and another way in the file it writes.

## Reading: the exposure-time state request

The message is requested only at the shutter press, and it carries exactly three things: the
aperture, the focal length, and the optical correction rows. That is the set a body needs to record
and correct an exposure, fetched at the moment the exposure is taken rather than relied on from the
60 Hz status loop.

On that reading the request is the body re-reading the lens's current state for the frame it is
about to capture, which is consistent with the focal length reaching EXIF and with the request
arriving inside the exposure sequence rather than during init or idle. **PROBABLE** — it accounts
for the timing, the field set and the EXIF result together, but only the focal length has been
traced to an output.

## Open questions

- `pl[0..3]`, `pl[8]` and `pl[23..34]`.
- Whether a body requests the message more than once per exposure, and whether it ever requests it
  outside an exposure sequence.
- Whether the aperture at `pl[9..10]` and the optical rows are also consumed at exposure time, as
  the focal length is, or are read from the status loop instead.
- Which message supplies the focal length a body *displays*, as opposed to the one it records.
