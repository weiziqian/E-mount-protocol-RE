# Message 0x4F — exposure start and end

**Summary.** Sent by a Sony a7CR in pairs inside the pause of the normal loop at a shutter release, one pair per frame (four in a burst
of four), 43 to 49 ms apart: `n 01 a a 00 00 00` then `n+1 00 a a 00 00 00`. `pl[0]` counts, `pl[1]` goes `01` then `00`, `pl[2..3]` is the
shooting aperture. No reply. Likely the start and end of each exposure. **PROBABLE** (Sony SEL40F25G and SEL24F28G captures).

**Direction:** B→L. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

Sent to a Sony SEL40F25G, neither frame drew a reply or changed anything the lens reports. **POSSIBLE.**

## Open questions

- Whether the gap is the exposure time.
