# How auto focus works

**Summary.** The body decides where focus should be; the lens only moves. Every frame of the normal
loop the lens reports where its focus group is and whether it is moving
([message 0x06](msg_0x06.md)), and the body answers with focus instructions packed as **records**
inside [message 0x04](msg_0x04.md). This document describes those records, the motion each one asks
of the lens, and how the lens reports back when the motion is finished.

`pl` is the payload of a message: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.
`rec` is a record inside a payload: `rec[0]` is its tag byte. Ranges `pl[a..b]` and `rec[a..b]` are
inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`. All multi-byte values are
little-endian. Bit 0 is the least significant bit.

---

## 1. The control loop

The normal loop runs at the body's frame rate, about 60 Hz, four frames per cycle, lens first:

| Order | Dir | ID | Role in focusing |
| --- | --- | --- | --- |
| 1 | L→B | **[0x05](msg_0x05.md)** | **focus in physical terms**: subject distance, "in motion" flag, focus ring direction, lens switch (§4.4) |
| 2 | L→B | **[0x06](msg_0x06.md)** | **focus servo status**: position now, position one frame ahead, travel limits, moving/stopped, tail blocks |
| 3 | B→L | [0x03](msg_0x03.md) | body state; can carry records, but focus records have been observed only in 0x04 |
| 4 | B→L | **[0x04](msg_0x04.md)** | **focus instructions**, as records |

So each cycle the lens reports first and the body responds. The body reads the status in 0x05 and
0x06, decides, and puts at most a few records into the 0x04 that closes the cycle. The lens acts on them
and shows the result in the next 0x06. CERTAIN for the ordering; the reading of 0x06 as the status
the body acts on is PROBABLE.

### Focus position units

All positions in this exchange — the targets in 0x04, the positions and limits in 0x06 — are on
**one scale: the lens's own focus position counts plus a fixed base**. Each lens chooses its own
range and resolution and declares it through the travel limits in 0x06 `pl[7..8]` (lower) and
`pl[9..10]` (upper). The body never assumes a scale; it commands inside the range the lens
advertises. CERTAIN that targets and reported positions share a scale. Which end of the range is
infinity is not fixed by the protocol. On a lens whose distance answers (§2.5) have been matched
to positions, position grows toward close focus, so the lower limit is the infinity end. POSSIBLE.

This is **not** the [aperture value](aperture_value.md) scale, although the numeric ranges overlap.

Two further units appear in focus instructions. Both are converted to positions by the lens, because
only the lens knows its own optics:

| Unit | Used by | Conversion to position counts |
| --- | --- | --- |
| **Distance code** (§2.5) | Move mode 3; queries `0x22`, `0x2E` | through the lens's own distance-to-position model |
| **Defocus unit** (§2.6) | Move mode 6 | × the scale the lens reports every frame in 0x06 `pl[13..14]` |

---

## 2. Message 0x04 — lengths and records

### 2.1 Structure

```
pl[0..12]   13-byte header, always present
pl[13..]    zero or more records: 1 tag byte + a fixed number of operand bytes per tag
```

A 0x04 carrying records **replaces** the bare 13-byte form in that cycle; it is not an extra frame.

### 2.2 Header fields

None of the header fields carries a focus target. They are listed here because a receiver must
skip exactly 13 bytes to reach the records.

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..5]` | Prefix `0x00 0x00 0x19 0x83 0x00 0x00` in every frame observed | Constant **CERTAIN**; meaning **UNKNOWN** except `pl[3]` |
| `pl[3]` | **Focus control mode.** `0x83` = focus is driven by the body (AF); `0x81` = manual focus by wire, the lens follows its own focus ring (§3.5). A lens switches only while focus is at rest and nothing is pending. Only `0x83` has been observed | **POSSIBLE** |
| `pl[6]` | Constant within a session; `0x3B`, `0x28`, `0x21`, `0x18` observed | **UNKNOWN** |
| `pl[7]` | `0x1F` or `0x00` depending on the device class | **UNKNOWN** |
| `pl[10]` | Body state code. `0x09` while idle; `0x40`, `0x41` alongside focus-position records; `0x48` alongside scan records; `0x08`, `0x00`, `0x01` otherwise | A state code **PROBABLE**; individual states **UNKNOWN** |
| `pl[8..9]`, `pl[11..12]` | Zero in every frame observed | **UNKNOWN** |

### 2.3 Record table

| Tag | Total size | Name | What it asks the lens to do | Confidence |
| --- | --- | --- | --- | --- |
| **`0x1C`** | 1 B | **Stop** | Stop focus motion | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x1D`** | 5 B | **Move** | Move focus to a position | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x1F`** | 14 B | **Scan** | Sweep focus across a window at a controlled speed | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x3C`** | 8 B | **Drive** | Run focus toward one end at a given speed | **PROBABLE** |
| **`0x22`** | 3 B | **Position → distance query** | Answer with the distance code of the position in `rec[1..2]` | **PROBABLE** |
| **`0x2E`** | 3 B | **Distance → position query** | Answer with the position for the distance code in `rec[1..2]` | **PROBABLE** |
| `0x2F` | 3 B | Row index | Optical-table row selection, not focus | Size **CERTAIN** |
| `0x34` | 24 B | — | Unknown | Size **POSSIBLE** |
| `0x4A` | 12 B | — | Unknown | Size **POSSIBLE** |

A receiver has to know the size of every tag it may meet, because records carry no length field. It
should skip a tag it does not recognise rather than stop parsing.

**Bound the walk by the frame, not by the transfer window.** The window a frame arrives in is
usually larger than the frame and occasionally smaller — see
[the transfer window](frame_format.md#the-transfer-window-may-exceed-the-frame) for the sizes
measured. Records read past the frame's own `len` come out of the checksum, the terminator and the
padding, and `0x55` is a valid operand byte, so the result is not a parse failure but invented
instructions. The safe bound is `min(len, bytes received)`.

### 2.4 Observed lengths

| Payload | Frame | Records | Meaning of the cycle |
| --- | --- | --- | --- |
| 13 | 22 | — | nothing to do |
| 14 | 23 | `0x1C` | stop |
| 17 | 26 | `0x2F` + `0x1C` | row index, then stop |
| 18 | 27 | `0x1D` | move to a position |
| 30 | 39 | `0x2F` + `0x1F` | row index, then scan |
| 21 | 30 | `0x3C` | drive (length implied by the record size; not yet observed) |

The first five are CERTAIN (observed on the wire or in stored frames). **A lens that accepts only
the 22-byte frame never sees a focus instruction.**

### 2.5 The distance code

A u16 on a logarithmic distance scale:

```
code = 384 + 64 × log2(D)          D = subject distance, in metres
```

So `384` (`0x0180`) is 1 m, each doubling of the distance adds 64, and each halving subtracts 64 —
0.5 m is 320, 2 m is 448. Codes run up to `0x06FF`; **`0x0700` means infinity**, or a distance beyond
what the lens can resolve, and a lens answers `0x0700` for any position at or beyond its infinity
end.

| Aspect | Confidence |
| --- | --- |
| Base-2 logarithmic scale, 64 codes per doubling, offset 384, `D` in metres | **CERTAIN** — a manual lens reports 272, 299, 320, 351, 384 and 448 in [message 0x05](msg_0x05.md) `pl[20..21]` at exactly the 0.3, 0.4, 0.5, 0.7, 1 and 2 m marks of its distance scale |
| `0x0700` = infinity / out of range; codes `>= 0x0700` are not accepted as targets | **PROBABLE** |

The lens owns the conversion to position, because it depends on the optics. The body can use it
without knowing the optics, through the two queries.

**`0x22` — position → distance.** `rec[1..2]` is a position (u16 LE). The lens answers with
`0x22 lo hi`: the distance code of that position.

**`0x2E` — distance → position.** `rec[1..2]` is a distance code (u16 LE). The lens answers with
`0x2E lo hi`: the position that focuses at that distance, clamped into the travel limits. An
operand of `0` returns a lens-held value whose meaning is **UNKNOWN**.

Both answers ride in the tail of the next [message 0x06](msg_0x06.md) (§4.2). PROBABLE.

The lens also reports the distance it is focused at, unasked, in every
[message 0x05](msg_0x05.md) `pl[20..21]` (§4.4).

### 2.6 The defocus unit

Every frame, the lens reports in 0x06 `pl[13..14]` (u16 LE; only `pl[13]` has been non-zero) **how
many position counts one defocus unit is worth at the current aperture**. Move mode 6 (§3.1) is a
relative move whose operand is in these units; the lens multiplies it by this same scale.

The scale grows roughly in proportion to the f-number — stopping down by one stop roughly doubles
it — which is how depth of focus behaves. So the defocus unit is best read as a fraction of the
depth of focus. With a unit like that, the body can state a phase-detect result ("this far out of
focus") without knowing the lens's position scale.

| Device, aperture | `pl[13..14]` |
| --- | --- |
| Sony SELP1650 at f/3.5 | 22 |
| Two native f/1.8 primes, wide open | 18 and 19 |
| The same two lenses, seven stops down | 144 and 152 |

| Aspect | Confidence |
| --- | --- |
| `pl[13..14]` is the multiplier Move mode 6 applies | **PROBABLE** |
| It depends on aperture and grows roughly in proportion to the f-number | **PROBABLE** |
| The unit is a fixed fraction of the depth of focus | **POSSIBLE** |
| A lens may report a smaller scale in the gentler body state (§3.1) | **POSSIBLE** |

---

## 3. The instructions, and the motion each requires

### 3.1 Move — record `0x1D`

```
rec[0]      0x1D
rec[1..2]   u16 LE  operand (a target position, or a distance, depending on the mode)
rec[3]      unknown, 0x00 in every record observed
rec[4]      mode byte: bits 0-2 select the mode, bit 3 is ignored, bits 4-7 unknown
```

| `rec[4] & 0x07` | Mode | Target | Confidence |
| --- | --- | --- | --- |
| **0** | **Absolute** | the operand itself, in the position units above | **PROBABLE** — every body frame observed uses this mode |
| 3 | Absolute, by distance | the position for the [distance code](#25-the-distance-code) in the operand, clamped; codes `>= 0x0700` are not accepted | **PROBABLE** |
| 4 | Relative | current position + operand, operand a signed count in position units | **PROBABLE** |
| 6 | Relative, in defocus units | current position + operand × the [defocus scale](#26-the-defocus-unit) the lens reports in 0x06 `pl[13..14]` | **PROBABLE** |
| other | — | ignored | **PROBABLE** |

Observed from a Sony a9 II, mode 0: targets spread across the advertised travel, e.g.

```
F0 1B 00 01 82 04 | 00 00 19 83 00 00 28 00 00 00 40 00 00 1D EC 12 00 00 | C1 02 55   target 0x12EC = 4844
F0 1B 00 01 A1 04 | 00 00 19 83 00 00 28 00 00 00 41 00 00 1D FF 7F 00 00 | 61 03 55   target 0x7FFF
```

**Required motion.**

1. **Clamp** the target into the travel limits the lens advertises in 0x06. A target outside them,
   including `0x7FFF`, becomes the nearer limit. CERTAIN that targets are clamped. Whether the body
   means `0x7FFF` as "drive to the limit" or as "no target" is **UNKNOWN**: it lies far above any
   lens's range, so a lens that clamps drives to its upper limit either way.
2. **Move there within about one body frame.** The speed is chosen per command as the distance
   left to travel divided by a time budget, then limited to the lens's own minimum and maximum
   speeds. The budget is about one frame period: 17–22 ms, the longer values when the body's frame
   period exceeds 18 ms (below about 55 Hz). A short correction therefore moves slowly and a long
   one fast. Moves longer than the lens can manage in one frame simply take more frames. PROBABLE.
   The lens knows the frame period by timing the body's own frames.
3. **Accelerate and decelerate smoothly** — ramp up, cruise, ramp down and stop at the target —
   and keep 0x06 `pl[2..3]` showing where it will be one frame later (§4.1). POSSIBLE.
4. **A new Move replaces the running one.** If the new target is ahead in the direction of travel,
   the lens keeps going to the new target. If it is behind, the lens decelerates to rest and then
   reverses. The replaced command produces **no completion event**; only the latest one does. PROBABLE.
5. **A target equal to the current position still counts as completed**, and is reported as such
   (§4.2). PROBABLE.
6. **Relative moves (modes 4 and 6) arriving while a Move is still running are queued** and
   executed after it, instead of replacing it. An absolute Move, a Scan or a Drive clears the queue.
   POSSIBLE.

Some body states call for gentler motion: a time budget of 22–40 ms (up to about two frames) and a
lower maximum speed. The state is carried in [message 0x03](msg_0x03.md) `pl[10..13]`. Which body
state this is, video recording being the obvious candidate, is **UNKNOWN**. PROBABLE that such a
state exists.

### 3.2 Stop — record `0x1C`

```
rec[0]      0x1C          (no operands)
```

**Required motion:** abandon any pending instruction, including the queued second leg of a Scan,
bring focus to rest with a normal deceleration rather than an abrupt halt, and report the stop
(§4.2). PROBABLE for "stop and report"; the rest is POSSIBLE.

### 3.3 Scan — record `0x1F`

A Scan asks the lens to sweep focus through a window at a steady, body-chosen speed, so the body can
sample the image continuously while focus moves. It is the only instruction that produces two
movements.

```
rec[0]      0x1F
rec[1]      flags
rec[2..3]   unknown (observed 0x02 0x83 both times)
rec[4]      scan speed code: low bits a magnitude (6 bits on one lens, 7 on another), bit 7 a scale flag
rec[5..6]   u16 LE  endpoint A
rec[7..8]   u16 LE  endpoint B
rec[9..11]  unknown (observed 0x3F 0x46 0x03 and 0x10 0x52 0x03)
rec[12..13] u16 LE  centre position C, used only by the centred form (bit 0 of rec[1] set); 0 in both records observed
```

| `rec[1]` bits | Meaning | Confidence |
| --- | --- | --- |
| bit 0 | 0 = the **endpoint form**: A and B are positions; 1 = the **centred form**: the window is `C − B … C + A` (see below) | **PROBABLE** |
| bits 0–2 = `6` | reverse the sweep direction: sweep from B's side to A's side | **POSSIBLE** |
| bit 3 | start from whichever end of the window is nearer the current position | **POSSIBLE** |

Observed records (payload bytes from `pl[13]`, `0x2F` row-index record first):

```
2F 09 0A | 1F 02 02 83 88 3D 49 ED 47 3F 46 03 00 00    A = 0x493D = 18749, B = 0x47ED = 18413, speed code 0x88
2F 07 08 | 1F 06 02 83 98 7D 45 F5 43 10 52 03 00 00    A = 0x457D = 17789, B = 0x43F5 = 17397, speed code 0x98, reversed
```

The window is narrow — a few hundred counts — and sits well inside the travel. In both records
A > B; the geometry below is described for that ordering.

**Required motion — two legs:**

```
position ->   run-up        window [B .. A]         run-out
            |<------>|<========================>|<------>|
leg 1:  fast move to the start of the run-up
leg 2:                  constant scan speed across the whole window  ------------>
```

1. **Leg 1 — approach.** Move at a fixed fast speed to a point **beyond the first endpoint by a
   run-up distance**. The lens chooses the run-up, and it must be at least the distance needed to
   accelerate from rest to the scan speed. Lenses make it a few hundred counts, and may lengthen it
   at smaller apertures, where the depth of focus is larger.
2. **Wait until leg 1 is at rest.**
3. **Leg 2 — sweep.** Move at the **scan speed** through both endpoints to a point beyond the
   second endpoint by a run-out slightly shorter than the run-up, so the lens is still at constant
   speed when it crosses it.

The run-up and run-out mean the focus group moves at constant speed across the entire window A–B.
The approach and deceleration happen outside it, where the body is not measuring.

**The centred form** (`rec[1]` bit 0 set) describes the window around a centre `C = rec[12..13]`:
it reaches `A` counts above `C` and `B` counts below it. An extent close to `0x7FFF` means "open to
the travel limit on that side". The approach leg is slower than in the endpoint form. PROBABLE for
centre and extents; POSSIBLE for the open-ended reading.

**Scan speed.** The speed rises with the magnitude in `rec[4]`, and is capped at the lens's maximum.
Lenses convert it into their own speed units, either through the measured frame period or by scaling
with the current aperture (faster when stopped down). The exact unit is **UNKNOWN**. POSSIBLE.

Both legs are clamped into the travel limits, like every other target.

PROBABLE for "two movements, the second one only after the first has finished", for the run-up
and run-out, and for the endpoint ordering bits. POSSIBLE for the speed handling.

### 3.4 Drive — record `0x3C`

```
rec[0]      0x3C
rec[1]      unknown (its top six bits are reported back in the event parameter, §4.2)
rec[2..3]   s16 LE  signed velocity
rec[4..7]   unknown
```

**Required motion:** take the **lower** travel limit as the target if the velocity is zero or
negative, the **upper** limit if it is positive, and run toward it at a speed proportional to
`|velocity|`. The speed has a floor, so any non-zero request moves at a useful rate, and a ceiling at
the lens's maximum. A lens may scale it with the current aperture. The motion ends at the limit, on a
Stop, or when another instruction replaces it. A continuous drive of this kind suits focus-by-wire
and searching. PROBABLE for the direction and the end conditions; POSSIBLE for the speed law. The
record has not been observed on the wire.

### 3.5 Manual focus by wire

With 0x04 `pl[3] = 0x81`, the body hands focus to the lens's own focus ring. The lens then moves
focus itself as the ring is turned, in proportion to the rotation. How it treats focus records in
this mode is **UNKNOWN**. The reporting changes as follows:

| Where | While the ring is turned |
| --- | --- |
| 0x05 `pl[60]` | ring direction, `0x01` or `0xFF`, held for a few frames after the ring stops |
| 0x05 `pl[22]` | bit 6 set |
| 0x05 `pl[20..21]` | the new subject distance |
| 0x06 `pl[20..21]` | the new position |
| 0x06 `pl[2..3]` | equal to `pl[20..21]`: no one-frame-ahead forecast in this mode |

With `pl[3] = 0x83` the ring is ignored. POSSIBLE throughout, except the ring-direction and
subject-distance behaviour of 0x05, which has been observed on a manual lens (PROBABLE).

---

## 4. How the lens answers

The lens answers in [message 0x06](msg_0x06.md) and [message 0x05](msg_0x05.md), both sent every
cycle:

| Channel | Where | When it changes |
| --- | --- | --- |
| Focus in physical terms | 0x05 `pl[20..23]`, `pl[54]`, `pl[60]`, `pl[62]` | every frame (§4.4) |
| Motion status | 0x06 `pl[0..1]`, `pl[2..3]`, `pl[20..21]` | every frame |
| Scan status | 0x06 `pl[25]` | during a Scan |
| Tail blocks | 0x06 `pl[39..]`, which lengthens the frame | once: when an instruction finishes, or to answer a query |

### 4.1 Every frame: motion status in 0x06

| Field | Content | Confidence |
| --- | --- | --- |
| `pl[0]` | `0x82` when the focus group is **at rest**. `0x02` while it is moving, or has only just stopped. Then bit 4 (`0x10`) is ORed in when the position is within `0x1F` counts of the lower limit, and bit 3 (`0x08`) when it is within `0x1F` counts of the upper limit. | Near-limit bits **PROBABLE**; bit 7 = at rest **PROBABLE** (`0x82` is what native lenses send while idle) |
| `pl[1]` | Direction of travel: `0x00` stopped; `0x02` one direction; `0x04` the other | **POSSIBLE**; which value is which direction **UNKNOWN** |
| `pl[20..21]` | u16 LE focus position **now**, sampled once per frame | **CERTAIN** |
| `pl[2..3]` | u16 LE focus position **one frame ahead**: `pl[20..21]` plus the distance the lens expects to travel before the next frame, never past the target. It equals `pl[20..21]` while at rest. | **CERTAIN** |
| `pl[7..8]`, `pl[9..10]` | u16 LE lower and upper travel limits, the range every target is clamped into | **CERTAIN** |
| `pl[13..14]` | u16 LE [defocus scale](#26-the-defocus-unit): position counts per defocus unit at the current aperture | **PROBABLE** |
| `pl[23]` | `0x40` or `0x00`. Non-zero only while the current instruction is a Move; while moving it follows a four-frame pattern. | **POSSIBLE**; meaning **UNKNOWN** |
| `pl[32..38]` | Seven signed bytes: the frame-to-frame position differences over the last eight frames, newest last. A short velocity history. | **PROBABLE** |

Taken together, these fields let the body follow a move in progress: where focus is, where it will
be next frame, how fast it has been moving, and whether it has come to rest.

### 4.2 The tail of 0x06: events and query answers

The 0x06 payload is a 39-byte core followed by zero or more **tail blocks**, starting at `pl[39]`.
Each block begins with a tag byte, in the same tag space as the 0x04 records:

| Block | Size | Content |
| --- | --- | --- |
| Event | 2 B, or 4 B for `0x3C` | `code param [0x00 0x00]`: an instruction has finished (below) |
| Query answer | 3 B | `0x22 lo hi` or `0x2E lo hi`: the answer to a query record (§2.5) |

```
F0 | len | cls seq 0x06 | pl[0..38] | tail blocks, 0 or more | ck_lo ck_hi | 55
```

At most one event is sent per frame. Query answers from one 0x04 are sent together in the next
0x06. The order of an event and query answers within the tail differs between lenses, so a receiver
must walk the tail by tag. PROBABLE.

#### Events

| Instruction | Appendix | Sent when | Confidence |
| --- | --- | --- | --- |
| Move `0x1D` | `0x1D 0x00` | the lens has arrived at the (clamped) target, or was already there | **PROBABLE** |
| Stop `0x1C` | `0x1C 0x01` | the lens has come to rest | **PROBABLE** |
| Scan `0x1F` | `0x1F 0x00` | the sweep leg has finished; see §4.3 for the delay | **PROBABLE** |
| Drive `0x3C` | `0x3C p 0x00 0x00`, where `p` = `rec[1] >> 2` of the Drive record | the drive has reached its limit, or the next instruction has arrived | **PROBABLE** |

Rules:

- **Only the latest instruction is answered.** An instruction that a newer one replaced before it
  finished produces no appendix. The exception is a Drive: it has no end of its own short of the
  limit, so it is answered when the next instruction arrives. PROBABLE.
- **A Scan's approach leg is not answered on its own.** Only the end of the sweep produces
  `0x1F 0x00`. PROBABLE.
- **No appendix at all** means nothing finished since the previous frame. This is the 48-byte
  frame, and every idle frame looks like this. CERTAIN.
- If several events are waiting, they are sent **one per frame**, in order. PROBABLE.

The appendix is the lens's explicit "done". The motion status in §4.1 carries the same information
implicitly, because `pl[0]` returns to `0x82` and `pl[2..3]` equals `pl[20..21]` once the lens is at
rest.

### 4.3 During a Scan: the status byte `pl[25]`

A Scan also steps the status byte `pl[25]` through its phases, one value per frame, each held for
one or two frames:

| `pl[25]` | Phase |
| --- | --- |
| `0x10` | Scan accepted; the sweep leg is about to be launched |
| `0x20` | Approach finished; the sweep is under way |
| `0x30` | Sweep finished |
| `0x00` | Idle again |

About five frames after `pl[25]` returns to `0x00`, the appendix `0x1F 0x00` follows (§4.2).
`pl[25]` stays `0x00` for Move, Stop and Drive.

PROBABLE for the `0x10 → 0x20 → 0x30 → 0x00` sequence, which two implementations agree on. POSSIBLE
for the one-or-two-frame holding and the five-frame delay before the appendix.

### 4.4 Every frame: focus in physical terms, in 0x05

[Message 0x05](msg_0x05.md) is mostly aperture and optical data, but it also carries focus
information. 0x06 gives the servo view in the lens's own position counts; 0x05 gives the same state
in units the body can use without knowing the lens:

| Field | Content | Confidence |
| --- | --- | --- |
| `pl[20..21]` | u16 LE **subject distance** as a [distance code](#25-the-distance-code), computed from the current focus position every frame; `0x0700` = infinity, or not reported | Encoding **CERTAIN** |
| `pl[22]` | Bit 7 always set by a lens that reports motion. **Bit 6 = in motion**: focus being driven, the focus ring being turned, and on some lenses the aperture moving | Bit 6 for focus **CERTAIN**; for aperture **POSSIBLE** |
| `pl[23]` | Coarse subject distance, a signed byte ≈ −32 × dioptres (`0xFF` ≈ infinity) | **PROBABLE** |
| `pl[54]` | Coarse focus position: counts from the infinity end ÷ 256, 0 near infinity | **POSSIBLE** |
| `pl[60]` | **Focus ring direction**, signed: `0x01` / `0xFF` while the ring is turned, `0x00` otherwise. Not set by body-commanded moves | **PROBABLE** |
| `pl[62]` | A two-position switch on the lens, `0x01` or `0x03`; updated only while no focus move is running or pending. Which value is which position is **UNKNOWN** | **POSSIBLE** |
| `pl[32..43]` | The optical rows in slots A and B are chosen by the current focus position, as well as by aperture — see [the optical rows](optical_data.md) | **PROBABLE** |
| `pl[81..82]` | Effective focal length, mm × 10, which changes with focus (focus breathing) | **PROBABLE** |

A device that sends `0x0700`, a frozen `pl[22] = 0x80`, `pl[60] = 0` and a constant `pl[23]` is
telling the body that focus never moves and its distance is unknown.

---

## 5. Worked sequences

**A Move** (60 Hz, `n` = cycle number):

| Cycle | Body 0x04 | Lens 0x06, next cycle |
| --- | --- | --- |
| n | `1D` target T, mode 0 | 0x06: `pl[0] = 0x02`, `pl[1]` = direction, `pl[2..3]` ahead of `pl[20..21]` toward T; 0x05: `pl[22]` bit 6 set |
| n+1 | (bare, or a revised `1D`) | still moving, or… |
| n+k | (bare) | arrived: 0x06 `pl[0] = 0x82`, `pl[1] = 0x00`, `pl[2..3] = pl[20..21]` = T, **appendix `1D 00`**; 0x05 `pl[22]` bit 6 clear, `pl[20..21]` = the distance of T |

**A Scan:**

| Phase | Lens 0x06 |
| --- | --- |
| Approach to the run-up point | `pl[0] = 0x02`, moving |
| Approach done, sweep launched | `pl[25] = 0x10`, then `0x20` |
| Sweep across A–B at scan speed | `pl[2..3]` / `pl[20..21]` advance steadily; `pl[32..38]` show a constant velocity |
| Sweep done | `pl[25] = 0x30`, then `0x00`, `pl[0] = 0x82` |
| About five frames later | appendix `1F 00` |

---

## Open questions

- Whether the lower travel limit is the infinity end on every lens, and which `pl[1]` value
  corresponds to which direction.
- Whether `0x7FFF` in a Move means "to the limit" or "no target".
- The units of the Scan speed code and of the Drive velocity.
- `rec[2..3]` and `rec[9..11]` of the Scan record.
- The exact definition of the defocus unit, and what the `0x2E` query returns for operand 0.
- Which body state calls for the gentler motion profile.
- The meaning of the 0x04 header fields `pl[6]`, `pl[7]` and `pl[10]`, and whether any body sends
  `pl[3] = 0x81`.
- Which 0x05 `pl[62]` value corresponds to which switch position.
