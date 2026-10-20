# Message 0x06 — focus range, subject distance and event channel

**Summary.** Focus travel limits, the absolute focus / subject-distance report, and the lens's
focus-drive status and event channel. Frame 2 of the four-frame 60 Hz loop.

**Direction:** L→B only.

**Class:** normal (`0x01`).

**Payload:** 39 bytes normally, but **the message is variable-length** in two independent ways:

- **The base length, 48 or 93 bytes, is set once per session by the body**, through `pl[8]` of its
  [message 0x0A](msg_0x0A.md) request: `7f` selects 93, anything else 48. See
  [The 93-byte form](#the-93-byte-form).
- **A per-frame tail** — an event appendix of 2 or 4 bytes, and query answers of 3 bytes each —
  is added to that base, giving 50 or 52 bytes and longer.

Only the 48-byte base has ever been observed on the wire: every body request recorded selects it.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Reference payloads

```
Sony SELP1650   81 00 00 40 07 10 10 f4 3e c4 42 00 00 16 00 14 47 47 eb 08 4e 43 00 00 27 00 fe 01 00 ...
TECHART LM-EA9  00 00 00 10 00 30 10 30 10 00 16 00 00 20 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00 ...
```

## Field map

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | **Partly decoded.** Yongnuo writes `0x02`/`0x82`, then sets **bit 4 (`0x10`) when the position is within `0x20` counts of the lower travel limit** and **bit 3 (`0x08`) when within `0x20` of the upper**. Observed `0x81`/`0x82` on SELP1650, `0x82` on SEL55210, `0x00` on the Viltrox and the LM-EA9. | Near-limit bits **CERTAIN** on Yongnuo; bit 7 and the rest UNKNOWN. Still a native/adapter discriminator |
| `pl[2..3]` | **Focus position, one frame AHEAD** — current position plus a motion forecast. SELP1650 reads 16384, inside its own reported range. **Scale: the lens's own encoder/step counts plus a fixed base** (Yongnuo: `16080 + steps`) — *not* message 0x05's normalised scale; it is **not** the [aperture value](aperture_value.md)'s scale, and the two are contrasted there. | **CERTAIN** |
| `pl[5..6]` | u16 — 4144 on the LM-EA9, the same as its lower clamp | POSSIBLE (a third copy of the lower limit) |
| **`pl[7..8]`** | **u16 LE lower focus travel limit** | **CERTAIN** on the LM-EA9; PROBABLE generally |
| **`pl[9..10]`** | **u16 LE upper focus travel limit** | as above |
| `pl[20..21]` | **Focus position NOW** — the current, un-forecast position. Cheaper EF-mount adapters report nothing here; native E-mount lenses do. | **CERTAIN** |
| **`pl[25]`** | **Focus-drive status.** On the LM-EA9, stepped by the command state machines: `0x10` command accepted → `0x20` first stage done → `0x30` both stages done → `0x00` cleared when the machine goes idle. Zero in every idle observation, which is consistent — none contains a focus drive. | **CERTAIN** on the LM-EA9; meaning on other devices UNKNOWN |
| **`pl[26..27]`** | **u16 LE absolute focus position / subject distance.** Optional — see below. | **CERTAIN** |
| `pl[32..38]` | **A 7-sample velocity trace.** Yongnuo keeps a 9-deep history of the position variable, pushed one per frame, and writes its **first differences** into these 7 bytes, newest last. Varies frame to frame on the SELP1650 (`66 7f 7f 7f 7f 7f 7b` → zeros); zero on all adapters. | **CERTAIN** on Yongnuo; PROBABLE that Sony's identical-shaped field is the same thing |
| **`pl[39..40]`** | **Event appendix — present only in the 50-byte form.** `pl[39]` = event code, `pl[40]` = parameter. See below. In the 93-byte form this is `pl[84..85]`, and `pl[39..83]` is the [reserved block](#the-93-byte-form). | Transport **CERTAIN**; code semantics **UNKNOWN** — the vendors disagree on `0x1f` |

---

## The message is variable-length

Message 0x06 is not a fixed 48-byte packet. It carries a focus-drive status byte and an event
appendix, and the frame grows to accommodate them. Every device builds it as a fixed 43-byte core
followed by an optional appendix; the framing layer then adds the 2 length bytes and the 3-byte
trailer:

```
F0 | len_lo len_hi | cls seq id | payload[0..38] | <appendix> | ck_lo ck_hi | 55
   \___________________ 43-byte core ___________/  0/2/4 B
```

| Appendix | Total frame | Seen on |
| --- | --- | --- |
| none | **48** | Both vendors; every frame observed on the wire |
| 2 bytes | **50** | TECHART LM-EA9 during a focus drive; Yongnuo for most event codes |
| 4 bytes | 52 | Yongnuo only, event code `0x3C` |
| none, 93-byte base | 93 (95 / 97 with an appendix) | Yongnuo, when the body's message 0x0A `pl[8]` is `7f`; never observed on the wire |

All offsets in this table and the next paragraph are for the 48-byte base. In the 93-byte base the
tail starts 45 bytes later — see [The 93-byte form](#the-93-byte-form).

**The appendix always lands at frame offsets `0x2D`/`0x2E` = `pl[39]`/`pl[40]`, in both
implementations, and the two arrive there by different routes** — which is what makes this a real
protocol feature rather than one vendor's quirk. Yongnuo computes the offset (core length 43, plus
2, transmitted as core + appendix, framing adding 5 → 48 / 50 / 52); the LM-EA9 hardcodes it.

The appendix is not the only thing in the tail. **Query answers** — 3 bytes each, `0x22 lo hi` or
`0x2E lo hi`, answering the queries the body sends in [message 0x04](msg_0x04.md) — ride there too
and lengthen the frame by 3 per answer. See
[autofocus §4.2](autofocus.md#42-the-tail-of-0x06-events-and-query-answers).

### The 93-byte form

The base length is chosen by the body, once per session, in its [message 0x0A](msg_0x0A.md)
request:

| Body's message 0x0A `pl[8]` | Base frame | Payload before the tail | Tail starts at |
| --- | --- | --- | --- |
| anything but `7f` (every body observed: `3f`) | **48** | `pl[0..38]` | `pl[39]`, frame `0x2D` |
| `7f` | **93** | `pl[0..83]` | `pl[84]`, frame `0x5A` |

The two forms, as Yongnuo builds them:

```
48:  F0 len cls seq 06 | pl[0..38] |                      | tail | ck ck 55
93:  F0 len cls seq 06 | pl[0..38] | pl[39..83] = 45 × 00 | tail | ck ck 55
```

- **`pl[0..38]` is identical in both.** The same builder fills it either way — travel limits, both
  focus positions, the status byte, the velocity trace. Every field in the [field map](#field-map)
  keeps its offset.
- **`pl[39..83]` is 45 bytes of zeros.** Yongnuo writes nothing into it; the buffer is cleared
  before every frame and nothing is stored past `pl[38]`.
- **The tail moves** from `pl[39]` to `pl[84]`. Its content — the event appendix and any query
  answers — is unchanged, so a 93-byte frame with an event is 95 or 97 bytes.

So in Yongnuo's implementation the 93-byte form is the 48-byte form with a 45-byte reserved block
inserted before the tail. It carries no extra information.

That says what Yongnuo sends, not what the block is for. A lens that implements the extension fully
may put data there; the block is 45 bytes, and the body asks for it explicitly. Its contents on any
other device are UNKNOWN.

| Aspect | Confidence |
| --- | --- |
| Body's 0x0A `pl[8] = 7f` selects the 93-byte base on Yongnuo (three models, two separate code bases) | **CERTAIN** |
| On Yongnuo: `pl[0..38]` identical in both forms, `pl[39..83]` all zero, tail moved to `pl[84]` | **CERTAIN** |
| A Sony lens does the same | **POSSIBLE** — no body has been seen sending `7f`, so no 93-byte frame has ever been captured |
| What `pl[39..83]` is meant to carry | **UNKNOWN** |

### Appendix format: `[code][parameter]`

Yongnuo builds it from an event queue:

```c
u16 ev; int n = 0;
if (queue_receive(event_queue, &ev, 0) == 1 && (ev & 0xff) != 0x1f) {
    appendix[0] = (u8)ev;                 /* the code */
    appendix[1] = (ev >> 10);             /* parameter, top 6 bits of the queue entry */
    if ((u8)ev == 0x1c) appendix[1] = 1;  /* code 0x1c forces parameter 1 */
    n = ((u8)ev == 0x3c) ? 4 : 2;         /* code 0x3c gets a 4-byte appendix */
}
return n;                                 /* 0, 2 or 4 */
```

`0x1f` is Yongnuo's **sentinel for "no event"** — it emits no appendix at all, leaving the frame at
48 bytes.

The LM-EA9 has no queue: it stages one code byte and always writes parameter `0`. Its codes
`0x1C`/`0x1D`/`0x1F` identify which focus command just finished. **Caution:** `0x1F` is a real code
there but is Yongnuo's *sentinel*, so the two vendors do not agree on it — **do not assume a shared
code table.**

### The status byte `pl[25]`

Independently of the appendix, both implementations write a status byte at frame offset `0x1F` =
`pl[25]`. The LM-EA9 steps it `0x10` → `0x20` → `0x30` → `0x00` through its command state machines;
Yongnuo pulls it from a **second** event queue and special-cases `0x1f` there too, converting it to
`0`.

Two independent implementations writing the same offset from a status/event source is good evidence
`pl[25]` is a real protocol field, not a vendor scratch byte.

**Why nothing on the wire shows this:** every 0x06 frame observed is 48 bytes with `pl[25] = 0`,
because the observations are init plus idle — no focus drive ever happens in them. **A 50-byte
message 0x06 with a non-zero `pl[25]` is a specific, cheap thing to look for in any future AF
observation**, and would confirm this whole section on the wire.

---

## `pl[2..3]` vs `pl[20..21]` is NOW vs. ONE-FRAME-AHEAD

The two focus-position fields are a feed-forward pair:

```c
pl[20..21] = pos;                 // the current position
pl[2..3]   = pos + forecast();    // current position + predicted one-frame displacement
```

The forecast is a **motion simulation**:

1. Returns **0 if the motor is idle** — a stationary lens reports both fields equal, so the fields
   differ *only while focus is actually moving*.
2. Remaining distance = |current − target| from the motion controller.
3. Runs the step-period ramp — accelerate, cruise (capped), decelerate — accumulating elapsed time
   and counting one position unit per iteration, stopping when the distance runs out or the time
   budget is spent.
4. The time budget is **a measured quantity, not a configured one**: at init the lens reads a
   hardware timer, takes the delta since the previous call, and stores it only if it falls in a band
   that reads as **10–30 ms, i.e. 33–100 Hz — the body's frame rate**. The lens times the body's
   polling and forecasts exactly one frame of it.
5. Sign = direction of travel.

So a native lens hands the body a **feed-forward term**: where focus is, and where it will be by the
time the body next acts.

### The forecast never predicts past the target

`pl[2..3]` always lies **between the current position and the target** and cannot overshoot. There
is no explicit clamp; the bound is structural. The routine initialises a counter to the remaining
distance and every phase decrements it in lockstep with the step count it returns:

| Phase | Guard | Decrement |
| --- | --- | --- |
| Accelerate | Break if `remaining < 1` or `remaining <= guard_band` | −1 per counted step |
| Cruise (4× unrolled) | Exits on `remaining < 2 / < 3 / < 4`, and `remaining <= guard_band` | −1/−2/−3, matching the steps added |
| Decelerate | Loop condition includes `0 < remaining` | −1 per step |

So the returned displacement satisfies `|forecast| ≤ |target − pos|`, signed by travel direction.

The bound is in fact slightly **tighter than the target**: every phase also stops once the remaining
distance falls to a guard band, so the forecast stops short rather than landing exactly on the
target. That is the property that makes the field safe for a body to act on — it converges toward
the target monotonically and never predicts an overshoot.

**Expected behaviour, then:** a lens recomputes both fields on **every** message 0x06, reports the
measured position in `pl[20..21]`, and reports position plus predicted one-frame displacement in
`pl[2..3]`, bounded by the target as above. The two are equal while focus is stationary and diverge,
in the direction of travel, while it moves.

Confidence: the *sending* side is CERTAIN. What the **body** does with the pair is inference — this
shows a native lens computes and sends a forecast, not that Sony's servo consumes it as feed-forward.

Two open points on the bound, flagged rather than assumed:

- **The remaining distance is latched, not recomputed per call.** Whether the latch is refreshed
  *during* a move — i.e. whether the bound tracks live remaining distance or the move's initial
  total — is **not established**. If it is the latter, the forecast could in principle run past the
  target late in a long move.
- The target the bound uses is certainly the motion controller's target, but it has **not** been
  traced back to a body command message. "Target" is confirmed; "body-commanded target" is inference.

### One vendor suppresses its own forecast

Yongnuo writes `pl[2..3] = pos + forecast` and `pl[20..21] = pos`, then in one mode collapses the
feed-forward term with `pl[2..3] = pl[20..21]`. So even a native lens sends the two fields equal in
some states — a native does not *always* provide a forecast.

---

## The travel clamps are real limits, not just a report

`pl[7..8]` and `pl[9..10]` are not advisory: at least one implementation clamps every commanded
focus target against them before driving the motor. The fields are byte-packed at odd offsets, which
is why implementations tend to read them a byte at a time.

Observed ranges:

| Device | Lower | Upper | Span |
| --- | --- | --- | --- |
| TECHART LM-EA9 1.6.0 | 4144 | 5728 | 1584 |
| TECHART LM-EA9 1.7.0 / 1.8.0 | 4144 | **5632** | 1488 |
| Sony SEL5518Z | 16178 | 21987 | **5809** |
| Sony SELP1650 | 16116 | 17092 | 976 |

The native SEL5518Z exposes roughly **4× finer focus resolution over its travel** than the LM-EA9.
Recorded as a fact; not linked to any symptom.

Note the constraint this implies: **a lens's actual travel limits should agree with the range it
advertises in message 0x08.**

---

## `pl[26..27]` subject distance — encoding, and it is OPTIONAL

At infinity the value is ≈ `110 × sqrt(focal_mm)`, and it *increases* as the lens focuses closer —
so it is a reciprocal-ish encoding, not millimetres.

| Lens | Value |
| --- | --- |
| Sony SELP1650 | 510 |
| Sony SEL55210 @55 mm | 793 |
| Sony SEL55210 @210 mm | 1685 |

Sony's own lenses populate it; Yongnuo's do not — all three of their models write a hardcoded **0**,
and those lenses get full, correct autofocus.

| Device | `pl[26..27]` |
| --- | --- |
| Sony natives | Real; varies with focus and focal length (510; 793 @55 → 1685 @210) |
| Yongnuo (3 models, native AF, works correctly) | **constant 0** |

So the field is **not required for autofocus**. What may still depend on it is MF assist and the
distance slider — untested.

## Manufacturer notes

- **Sony** — natives populate the near-limit flags in `pl[0]`, the velocity trace, and the subject
  distance. Travel spans are wide (the SEL5518Z spans 5809 counts).
- **Yongnuo** — computes the forecast and the near-limit flags, drives the appendix from an event
  queue with `0x1F` as its "no event" sentinel, and writes a hardcoded 0 for subject distance.
- **TECHART** — the LM-EA9 sends `pl[0] = 0`, a frozen subject distance, no forecast, and a narrow
  1488–1584 count travel span. It stages one appendix code with parameter always 0, and treats
  `0x1F` as a real code rather than a sentinel.
- **Viltrox** — the EF adapters report nothing in `pl[20..21]` and send `pl[0] = 0`.

## Open questions

- `pl[0]` bit 7 and the remaining bits.
- The event code table. The two implementations disagree on `0x1F`, so no shared table is
  established.
- `pl[25]`'s meaning on devices other than the LM-EA9.
- `pl[5..6]`: whether it really is a third copy of the lower travel limit.
- Whether the body actually consumes the `pl[2..3]` forecast as feed-forward.
- Whether the forecast's remaining-distance bound tracks live distance or the move's initial total.
- What `pl[26..27]` is used for, given autofocus works without it.
- Which bodies send message 0x0A `pl[8] = 7f`, and what `pl[39..83]` of the 93-byte form carries on
  a lens that fills it.
