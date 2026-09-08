# Message 0x1B — the command channel

**Summary.** A short bidirectional command message. **Two incompatible readings of its payload are
established, on two different devices**: a focus target, and an aperture command. Both use the same
field, and they collide numerically — see the warning below before decoding it.

**Direction:** both.

**Class:** normal (`0x01`) for the focus reading; **init** (`0x02`) for the aperture reading.

**Payload:** ~13 bytes (19-byte frame on the TECHART LM-EA9).

**Never observed on the wire.** Every observation held is init plus idle, and no 0x1B frame appears
in any of them.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

## Reading 1 — an absolute focus target

The body commands an absolute focus target; the lens acknowledges with its current position.

```
inbound 0x1B:  pl[0..1] = the body's focus TARGET
               steps    = inverse_ladder(target, direction)   /* protocol scale -> motor steps */
               command the motor with (steps, speed)

reply   0x1B:  pl[0..1] and pl[2..3] = the lens's current position, written twice
```

Three things follow, and they are why this message matters:

1. **The target→steps map is the exact inverse of message 0x05's position transform** — the same
   per-lens ladder, run the other way. It returns 0 for any input ≤ 4544. The command path and the
   report path straddle one calibration table.
2. **So message 0x1B speaks message 0x05's units**, the normalised `n × 256/3` distance scale — *not*
   message 0x06's raw encoder counts. The body issues focus targets in the same space the lens
   reports position in, which is what makes that space worth getting right. That scale, and the
   inverse map with its plots, are documented in
   **[the live focus position](live_focus_position.md)**.
3. **The reply is one position sent twice**, exactly as message 0x05 does with `pl[0..1]`/`pl[2..3]`.
   It is not a wide/tele pair.

The move speed is derived from the distance: `speed = f(|target_steps − current_steps|)` with a
floor of 1000.

### Four properties of the command path

All **CERTAIN** for the implementation they were taken from, and all worth knowing before commanding
a lens or emulating one:

1. **Exact at grid points, approximate between them.** The inverse rounds to the *nearest* grid
   index — `q = (3·pos + 128) >> 8`, not a floor — anchors on `ladder[q − 53]`, and extrapolates
   with the width of the **adjacent** segment on the side the direction argument points. Where the
   two neighbouring segments have equal width that is exact; on the YN35 DA's ladder five junctions
   do not, and the error there is a few steps.
2. **A target ≤ 4544 returns 0 steps.** A hard infinity clamp, and the reason 4544 is the anchor
   rather than the grid point 4522 just below it.
3. **The first segment has a genuine discontinuity.** It is scaled `1/64` (spanning 64 output counts,
   not 85.33) and anchored on 4544, but the branch selecting that scale triggers on `pos < 4608`
   while the rounding has already advanced the index at `pos ≈ 4566`. Targets in **4566…4607**
   therefore resolve about **+53 steps too far**, snapping back at 4608. That is the near-infinity
   end of the travel.
4. **The far end can overshoot.** With `dir = −1`, targets between 6144 and ~6186 return up to 720
   steps on the YN35 DA — past `ladder[19] = 704`, the mechanical end. Above that it clamps.

Points 3 and 4 are single-vendor observations, so they are **not** protocol statements — a body must
not assume every lens behaves this way. They are recorded because they bound how precisely a body
can expect a 0x1B target to be honoured, and because an emulator that reproduces the protocol but
not these quirks will differ from a real Yongnuo lens in exactly those two bands.

---

## Reading 2 — an aperture command

**On a Sony a9 II, an init-class message 0x1B carries an APERTURE.** The TECHART LM-EA9 decodes
inbound 0x1B `pl[0]`/`pl[1]` into a 27-position ladder, and two hardware data points fix the
encoding exactly.

| Encoding | Value |
| --- | --- |
| `pl[1]` | Whole stops, `= AV + 16`, where `AV = 2·log₂(N)` is the APEX aperture value |
| `pl[0]` | Fractional stop: `0x00` / `0x55` / `0xAA` = 0, ⅓, ⅔ of 256 |
| `pl[0..1]` as u16 LE | **`256 × AV + 4096`** |

Checked against the LM-EA9's observed behaviour, where the user sets the camera's aperture and
presses the shutter:

| Aperture | AV | `pl[1]` | In the device's accepted range `0x13…0x1B`? | Observed |
| --- | --- | --- | --- | --- |
| **f/8** | 6 | `0x16` | Yes | **Sets its focal length to 50 mm** ✔ |
| **f/2** | 2 | `0x12` | No, one below the bottom → dropped | **No change** ✔ |

Both predictions land exactly. The accepted window `0x13…0x1B` is **f/2.8 … f/45**, which is also
exactly the aperture range a device would offer for whole-stop resolution over eight stops.

**And a shutter press is involved**, not just an aperture change — the LM-EA9 only reacts when the
shutter is released. So the frame is emitted as part of the exposure sequence, which is consistent
with an aperture *command* (stop down to N) rather than a status report.

---

## The two readings collide numerically — do not decode 0x1B blind

`256 × AV + 4096` and `n × 256/3` **both quantise in thirds of 256**, so an aperture payload lands
squarely on the focus grid and vice versa. f/8 → 5632, which is also focus grid point `n` = 66. A
decoder that assumes one meaning will silently produce a plausible-looking value for the other.

What discriminates them is **UNKNOWN**, and the obvious candidate does not work: Yongnuo's message
dispatcher switches on the ID byte alone and **never looks at the class byte**, so an init-class 0x1B
would drive its focus motor just as a normal-class one does. Either

- the two devices are being sent different messages and one of the two decoders is a
  misinterpretation of traffic meant for the other, or
- message 0x1B carries a sub-command selector in a byte neither implementation inspects, or
- the meaning is set by session context.

Until an actual 0x1B exchange is observed, treat `pl[0..1]` as **"a value on the ⅓-of-256 grid whose
referent depends on context"**, not as a focus position.

| Reading | Confidence |
| --- | --- |
| Focus target | **CERTAIN** for Yongnuo; **UNKNOWN** as a protocol-wide statement |
| Aperture command | **PROBABLE** — the arithmetic fits two independent hardware observations exactly, but it is inferred from one vendor's decoder, not read off the wire |

## Manufacturer notes

- **Yongnuo** — implements 0x1B as a focus command, decoding `pl[0..1]` through the inverse of its
  own message 0x05 ladder and driving the motor. It ignores the class byte.
- **TECHART** — the LM-EA9 does **not** implement 0x1B as a command. It echoes the request's
  `pl[0..1]` back, and takes focus commands through an entirely different path that dispatches
  inbound frames on **frame length** rather than message ID.

### Two consequences of that non-conforming path

1. **Body command frames of 27 and 36 bytes exist**, because the LM-EA9 parses focus targets out of
   them. Neither length appears anywhere in the observed corpus, so which message ID carries them is
   **UNKNOWN**.
2. Observed body frame lengths, for comparison: message 0x03 at 29 (`0x1D`) and 32 (`0x20`) bytes,
   message 0x04 at 22 (`0x16`), 840 frames each. An early report of "the body replying with packet
   0x1d during AF and 0x20 in MF" is **this**, in a length notation — the two variants of message
   0x03, not two separate message IDs. See [message 0x1D](msg_0x1D.md) and
   [message 0x20](msg_0x20.md).

## Open questions

- What discriminates the focus reading from the aperture reading.
- Whether a real body ever sends 0x1B at all — nothing has been observed.
- Which message ID carries the 27- and 36-byte body command frames.
