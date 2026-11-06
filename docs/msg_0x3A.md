# Message 0x3A — size of the 0x3B optical file

**Summary.** The lens announces how large the file it serves through [message 0x3B](msg_0x3B.md) is,
and in how many pages. Seen only in lens firmware; never captured on the wire.

**Direction:** both; the body request has not been observed on the wire. **Class:** `0x03`
(see [the frame format](frame_format.md)). **Length:** 32 bytes (23-byte payload).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Body request — from Yongnuo firmware

`pl[0]` selects the file that the following 0x3B requests read: `00` selects the `LF` file, and `08`
selects a second, one-page object. Any other value clears the selection. Confidence: CERTAIN in
Yongnuo firmware. TTArtisan ignores the byte.

## Field map — lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | status: must be `00`, or the body abandons the transfer | CERTAIN on the A7 IV firmware |
| `pl[1..4]` | u32 LE — **size of the transfer**, the number of bytes the body keeps. It must fit the body's buffer. Yongnuo sends file size + 16; TTArtisan sends a constant | CERTAIN on the A7 IV firmware |
| `pl[5..6]` | u16 LE — **number of 0x3B pages**. Must agree with the size: the last page has to carry 1–488 of its bytes. Yongnuo computes it as `⌊size / 488⌋ + 1` | CERTAIN on the A7 IV firmware |
| rest | zero | |

## Other files — CERTAIN on the A7 IV firmware

The A7 IV reads six files through 0x3A/0x3B, with the same checks for each. Its class names:
`EclipseCom` (the optical file of [message 0x3B](msg_0x3B.md)), `BreathingCorrectionData`,
`CamcurveData`, `FnoCorrectAbility`, `IrisDrivabilityData`, `SteadyShotInfo`. The request value that
selects each one has not been traced.

TTArtisan 50/1.8 sends `00 a0 0a 00 00 06 00 …` for both of its files: 2720 bytes, which is the size
of the larger file, and 6 pages. The stored template has `07` at `pl[5]`, and the code overwrites it
with `06` before sending.

## Manufacturer notes

- **TTArtisan** — fixed reply, apart from the page count written at send time.
- **Yongnuo** — builds the reply at runtime, in a handler of its own for 0x3A/0x3B/0x4A outside the
  init dispatcher. It asserts 0x3A in [message 0x01](msg_0x01.md).

## Open questions

- The body's request, and whether a body ever sends it to a lens that does not assert 0x3A.
- What the `08` object is; why Yongnuo adds 16 to the size; which `pl[0]` selects which of the six
  files.
