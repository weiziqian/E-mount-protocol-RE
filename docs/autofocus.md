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
| 1 | L→B | [0x05](msg_0x05.md) | not involved |
| 2 | L→B | **[0x06](msg_0x06.md)** | **focus status**: position now, position one frame ahead, travel limits, moving/stopped, event appendix |
| 3 | B→L | [0x03](msg_0x03.md) | body state; can carry records, but focus records have been observed only in 0x04 |
| 4 | B→L | **[0x04](msg_0x04.md)** | **focus instructions**, as records |

So each cycle the lens reports first and the body responds. The body reads the status in 0x06,
decides, and puts at most a few records into the 0x04 that closes the cycle. The lens acts on them
and shows the result in the next 0x06. CERTAIN for the ordering; the reading of 0x06 as the status
the body acts on is PROBABLE.

### Focus position units

All positions in this exchange — the targets in 0x04, the positions and limits in 0x06 — are on
**one scale: the lens's own focus position counts plus a fixed base**. Each lens chooses its own
range and resolution and declares it through the travel limits in 0x06 `pl[7..8]` (lower) and
`pl[9..10]` (upper). The body never assumes a scale; it commands inside the range the lens
advertises. CERTAIN that targets and reported positions share a scale; which end of the range is
infinity is **UNKNOWN**.

This is **not** the [aperture value](aperture_value.md) scale, although the numeric ranges overlap.

---

## 2. Message 0x04 — lengths and records

### Structure

```
pl[0..12]   13-byte header, always present
pl[13..]    zero or more records: 1 tag byte + a fixed number of operand bytes per tag
```

A 0x04 carrying records **replaces** the bare 13-byte form in that cycle; it is not an extra frame.

### Header fields

None of the header fields carries a focus target. They are listed here because a receiver must
skip exactly 13 bytes to reach the records.

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0..5]` | Fixed prefix `0x00 0x00 0x19 0x83 0x00 0x00` in every frame observed | Constant **CERTAIN**; meaning **UNKNOWN** |
| `pl[6]` | Constant within a session; `0x3B`, `0x28`, `0x21`, `0x18` observed | **UNKNOWN** |
| `pl[7]` | `0x1F` or `0x00` depending on the device class | **UNKNOWN** |
| `pl[10]` | Body state code. `0x09` while idle; `0x40`, `0x41` alongside focus-position records; `0x48` alongside scan records; `0x08`, `0x00`, `0x01` otherwise | A state code **PROBABLE**; individual states **UNKNOWN** |
| `pl[8..9]`, `pl[11..12]` | Zero in every frame observed | **UNKNOWN** |

### Record table

| Tag | Total size | Name | What it asks the lens to do | Confidence |
| --- | --- | --- | --- | --- |
| **`0x1C`** | 1 B | **Stop** | Stop focus motion | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x1D`** | 5 B | **Move** | Move focus to a position | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x1F`** | 14 B | **Scan** | Sweep focus across a window at a controlled speed | Size **CERTAIN**; meaning **PROBABLE** |
| **`0x3C`** | 8 B | **Drive** | Run focus toward one end at a given speed | **POSSIBLE** |
| `0x2F` | 3 B | Row index | Optical-table row selection, not focus | Size **CERTAIN** |
| `0x22` | 3 B | Query | Carries a u16 position operand; answered by the lens | **POSSIBLE**; meaning **UNKNOWN** |
| `0x2E` | 3 B | Query | Carries a u16 operand; answered by the lens | **POSSIBLE**; meaning **UNKNOWN** |
| `0x34` | 24 B | — | Unknown | Size **POSSIBLE** |
| `0x4A` | 12 B | — | Unknown | Size **POSSIBLE** |

A receiver has to know the size of every tag it may meet, because records carry no length field. It
should skip a tag it does not recognise rather than stop parsing.

### Observed lengths

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
| 3 | Absolute, from a distance code | a position the lens derives from the operand; codes of `0x0700` and above are out of range | **POSSIBLE** |
| 4 | Relative | current position + operand, operand as a signed step count | **POSSIBLE** |
| 6 | Relative, scaled | current position + operand × a lens-defined scale factor | **POSSIBLE** |
| other | — | ignored | **POSSIBLE** |

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
2. **Move there within about one body frame.** The speed is chosen per command from the distance
   left to travel and a time budget of roughly one frame period (about 18–22 ms at 60 Hz), then
   limited to the lens's own minimum and maximum speeds. A short correction therefore moves slowly
   and a long one fast. Moves longer than the lens can manage in one frame simply take more frames.
   POSSIBLE. The lens knows the frame period by timing the body's own frames.
3. **Accelerate and decelerate smoothly** — ramp up, cruise, ramp down and stop at the target —
   and keep 0x06 `pl[2..3]` showing where it will be one frame later (§4.1). POSSIBLE.
4. **A new Move replaces the running one.** If the new target is ahead in the direction of travel,
   the lens keeps going to the new target. If it is behind, the lens decelerates to rest and then
   reverses. The replaced command produces **no completion event**; only the latest one does. POSSIBLE.
5. **A target equal to the current position still counts as completed**, and is reported as such
   (§4.2). POSSIBLE.

Some body states call for gentler motion — a longer time budget and a lower maximum speed. The
state is carried in [message 0x03](msg_0x03.md) `pl[10..13]`. Which body state this is, video
recording being the obvious candidate, is **UNKNOWN**.

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
rec[4]      scan speed code: bits 0-6 magnitude, bit 7 a scale flag
rec[5..6]   u16 LE  endpoint A
rec[7..8]   u16 LE  endpoint B
rec[9..11]  unknown (observed 0x3F 0x46 0x03 and 0x10 0x52 0x03)
rec[12..13] u16 LE  a third position, used only by the alternative form (bit 0 of rec[1] set); 0 in both records observed
```

| `rec[1]` bits | Meaning | Confidence |
| --- | --- | --- |
| bit 0 | 0 = the standard form below; 1 = an alternative form using `rec[12..13]`, with a slower approach | **POSSIBLE**; the alternative form's geometry is **UNKNOWN** |
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
   run-up distance**. The run-up is the distance the lens needs to accelerate from rest to the scan
   speed, plus a small margin.
2. **Wait until leg 1 is at rest.**
3. **Leg 2 — sweep.** Move at the **scan speed** through both endpoints to a point beyond the
   second endpoint by a run-out slightly shorter than the run-up, so the lens is still at constant
   speed when it crosses it.

The run-up and run-out mean the focus group moves at constant speed across the entire window A–B.
The approach and deceleration happen outside it, where the body is not measuring.

The scan speed is expressed relative to the body's frame period: the lens converts `rec[4]` into its
own speed units using the frame period it has measured. The magnitude is `rec[4] & 0x7F`. When bit 7
is set it is multiplied by a lens-defined factor. The unit is **UNKNOWN**; "counts per body frame"
fits the arithmetic but is unconfirmed. POSSIBLE.

PROBABLE for "two movements, the second one only after the first has finished", which two
implementations agree on. POSSIBLE for the run-up, the run-out and the speed handling.

### 3.4 Drive — record `0x3C`

```
rec[0]      0x3C
rec[1]      unknown (its top six bits are reported back in the event parameter, §4.2)
rec[2..3]   s16 LE  signed velocity
rec[4..7]   unknown
```

**Required motion:** take the **lower** travel limit as the target if the velocity is zero or
negative, the **upper** limit if it is positive, and run toward it at a speed proportional to
`|velocity|`, up to the lens's maximum. The motion ends at the limit, on a Stop, or when another
instruction replaces it. A continuous drive of this kind suits focus-by-wire and searching. POSSIBLE
throughout. The record has not been observed on the wire.

---

## 4. How the lens answers

The lens has three ways to answer, all in [message 0x06](msg_0x06.md), which it sends every cycle:

| Channel | Where | When it changes |
| --- | --- | --- |
| Motion status | `pl[0..1]`, `pl[2..3]`, `pl[20..21]` | every frame |
| Scan status | `pl[25]` | during a Scan |
| Event appendix | `pl[39..]`, which lengthens the frame | once, when an instruction finishes |

### 4.1 Every frame: motion status

| Field | Content | Confidence |
| --- | --- | --- |
| `pl[0]` | `0x82` when the focus group is **at rest**. `0x02` while it is moving, or has only just stopped. Then bit 4 (`0x10`) is ORed in when the position is within `0x1F` counts of the lower limit, and bit 3 (`0x08`) when it is within `0x1F` counts of the upper limit. | Near-limit bits **PROBABLE**; bit 7 = at rest **PROBABLE** (`0x82` is what native lenses send while idle) |
| `pl[1]` | Direction of travel: `0x00` stopped; `0x02` one direction; `0x04` the other | **POSSIBLE**; which value is which direction **UNKNOWN** |
| `pl[20..21]` | u16 LE focus position **now**, sampled once per frame | **CERTAIN** |
| `pl[2..3]` | u16 LE focus position **one frame ahead**: `pl[20..21]` plus the distance the lens expects to travel before the next frame, never past the target. It equals `pl[20..21]` while at rest. | **CERTAIN** |
| `pl[7..8]`, `pl[9..10]` | u16 LE lower and upper travel limits, the range every target is clamped into | **CERTAIN** |
| `pl[32..38]` | Seven signed bytes: the frame-to-frame position differences over the last eight frames, newest last. A short velocity history. | **PROBABLE** |

Taken together, these fields let the body follow a move in progress: where focus is, where it will
be next frame, how fast it has been moving, and whether it has come to rest.

### 4.2 On completion: the event appendix

When an instruction finishes, the lens appends an event to the **next** 0x06, once. The frame grows
from 48 bytes to 50, or to 52 for a Drive. The appendix always starts at `pl[39]`.

```
F0 | len | cls seq 0x06 | pl[0..38] | code param [0x00 0x00] | ck_lo ck_hi | 55
                                      \___ 2 or 4 bytes ___/
```

| Instruction | Appendix | Sent when | Confidence |
| --- | --- | --- | --- |
| Move `0x1D` | `0x1D 0x00` | the lens has arrived at the (clamped) target, or was already there | **PROBABLE** |
| Stop `0x1C` | `0x1C 0x01` | the lens has come to rest | **PROBABLE** |
| Scan `0x1F` | `0x1F 0x00` | the sweep leg has finished; see §4.3 for the delay | **PROBABLE** |
| Drive `0x3C` | `0x3C p 0x00 0x00`, where `p` = `rec[1] >> 2` of the Drive record | the drive has ended | **POSSIBLE** |

Rules:

- **Only the latest instruction is answered.** An instruction that a newer one replaced before it
  finished produces no appendix. POSSIBLE.
- **A Scan's approach leg is not answered on its own.** Only the end of the sweep produces
  `0x1F 0x00`. PROBABLE.
- **No appendix at all** means nothing finished since the previous frame. This is the 48-byte
  frame, and every idle frame looks like this. CERTAIN.
- If several events are waiting, they are sent **one per frame**, in order. POSSIBLE.

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

---

## 5. Worked sequences

**A Move** (60 Hz, `n` = cycle number):

| Cycle | Body 0x04 | Lens 0x06, next cycle |
| --- | --- | --- |
| n | `1D` target T, mode 0 | `pl[0] = 0x02`, `pl[1]` = direction, `pl[2..3]` ahead of `pl[20..21]` toward T |
| n+1 | (bare, or a revised `1D`) | still moving, or… |
| n+k | (bare) | arrived: `pl[0] = 0x82`, `pl[1] = 0x00`, `pl[2..3] = pl[20..21]` = T, **appendix `1D 00`** |

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

- Which end of the focus range is infinity, and which `pl[1]` value corresponds to which direction.
- Whether `0x7FFF` in a Move means "to the limit" or "no target".
- The units of the Scan speed code and of the Drive velocity.
- `rec[2..3]` and `rec[9..11]` of the Scan record, and the geometry of its alternative form.
- Which body state calls for the gentler motion profile.
- What the `0x22` and `0x2E` records ask, and where their answers are carried.
- The meaning of the 0x04 header fields `pl[6]`, `pl[7]` and `pl[10]`.
