# Message 0x34 — timed focus move

**Summary.** The body asks the lens to move focus to a position **within a time limit**, and the
lens answers with the position it is going to. The A7 IV sends it from its release sequence, the
steps it runs around a shutter release. The request and the reply are each a frame of their own,
not records inside [0x04](msg_0x04.md).

**Direction:** both: body request, lens reply. **Class:** `0x02`. **Length:** request and reply
are each 24 bytes with the ID, so 23-byte payloads in 32-byte frames (A7 IV length table).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Body request — CERTAIN on the A7 IV firmware

| Field | Meaning |
| --- | --- |
| `pl[0..1]` | u16 LE **target**; its unit depends on the mode (below) |
| `pl[2]` | `00` |
| `pl[3]` | bits 0–1: **mode**; bit 2: set only with mode 2; bits 6–7: always 0 (no caller sets them) |
| `pl[4]` | **time limit**: milliseconds left until the body's deadline, capped at 87 for a native lens |
| `pl[5..22]` | zero |

### Modes — CERTAIN on the A7 IV firmware

Only three `pl[3]` values reach the wire. The body picks one from the type of its AF request:

| `pl[3]` | Mode | `pl[0..1]` |
| --- | --- | --- |
| `00` | 0, **absolute** | a position in [0x06](msg_0x06.md) units; the body clamps it to `0..0x7FFF` (`0..0xFFFF` when a lens-descriptor flag is set) |
| `03` | 3, **distance code** | `0..0x700`, the body clamps it; `0x700` is infinity |
| `06` | 2 + bit 2, **relative** | a signed step from the current position, in lens-defined units |

The body also sends mode 0 for a position it predicts at the deadline (**PROBABLE**: target computed
from the time left). When it has no target it sends no 0x34 at all and completes the step at once.
Which shooting situations produce which AF request type is UNKNOWN.

## Lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | status; `00` = accepted | The A7 IV reads it: **CERTAIN**. Meaning **PROBABLE**: both implementations send `00` |
| `pl[1..2]` | u16 LE **the position the lens is moving to** | The A7 IV reads it: **CERTAIN**. Meaning **CERTAIN** in Yongnuo firmware |
| `pl[3..22]` | zero in both implementations | — |

The A7 IV reads nothing beyond `pl[2]`. A non-zero `pl[0]` is logged as an error. `pl[1..2]` is
written to the same trace as the request's target, and nothing on this path uses it further
(**CERTAIN on the a7C II and a9 II firmware**: the reply's only consumer writes `pl[1..2]` to a trace
log and the event log).

## When the body sends it — CERTAIN on the A7 IV firmware

- **From the release sequence.** The A7 IV's lens sequencer (`BLSequencer`, with states such as
  Focus, FocusWait, Iris, IrisWait and WaitExpStartingRes) sends 0x34 in its Focus step. It moves to
  FocusWait once the request is out, and leaves FocusWait when the **reply** arrives. It does not
  wait for the move to finish, nor read [0x06](msg_0x06.md) for it. What the move is for is UNKNOWN. Nearby code computes focus-drive
  compensation times before exposure, so it is probably a last focus correction ahead of the exposure
  (**POSSIBLE**).
- **Only if declared.** The request goes out only when the lens declares 0x34 in
  [message 0x01](msg_0x01.md). Without it, the Focus step reports "not done" at once, fires its
  completion with status 0, and the sequence moves on. It does not wait, so there is no timeout. The
  correction is simply skipped.
- **0x4A instead, on newer lenses.** When the lens declares message `0x4A` and a further body
  condition holds, the same move goes out as `0x4A`, with the same target, mode and time limit.

## Implementing it

- **Declare it only with a real move behind it.** A reply that names a target the lens does not
  move to tells the body the correction was made. Not declaring 0x34 is the safe alternative: the
  body then skips the step.
- **Respect the time limit.** Yongnuo moves at most a distance the mechanism can cover in the
  time given, and replies with where that takes it (below).

## Yongnuo's implementation — CERTAIN in Yongnuo 50 DF and 35 DA firmware

Yongnuo handles 0x34 in its init dispatcher (50 DF `FUN_0801620c`, 35 DA `FUN_0800bea8`). It
replies first and moves afterwards.

- **Mode 0, or a target inside the travel.** The step limit is `L = 18 × n` position units, where `n`
  is `pl[4]`, less 13 when it is above 13. If the target is more than `L` from the current position,
  it moves `L` toward it; otherwise to the target. It moves at its maximum speed and replies
  `pl[0] = 00`, `pl[1..2]` = the position it is moving to.
- **Otherwise the target is converted first**, by `pl[3] & 7`: `3` through a distance-to-position
  lookup; `6` or `7` as a relative step, current position + `k × target` (`k` = 8 on the 35 DA; a
  runtime value, default 10, on the 50 DF). The same step limit then applies, with a faster speed
  setting. Any other mode does not move.

**Events.** The 0x34 move is given no completion queue and the 0x04 command counter is not
advanced. So a running [0x04](msg_0x04.md) Move that it replaces is reported superseded at once,
and its `1D 00` event goes out in the next [0x06](msg_0x06.md). The 0x34 move itself produces no
event. CERTAIN in 50 DF firmware (motor task `FUN_0801305c`); PROBABLE on the 35 DA, which has the
same descriptor and event logic but whose motor task was not read.

Its 0x04 record walker also knows a 24-byte `0x34` record and ignores it. The A7 IV never puts 0x34
in a 0x04 frame.

## Manufacturer notes

- **Sony** — both natives measured, the SELP1650 and the SEL55210, declare 0x34 in 0x01.
- **Yongnuo** — declares it and moves, as above.
- **TTArtisan** — the 50/1.8 E declares 0x34 and answers with a **fixed stored frame**:
  `pl[0] = 00`, `pl[1..2] = 0x0007`, the rest zero. Its init reply builder (`0x08007eb8` in the
  decoded image) copies that frame and recomputes the checksum; it reads nothing from the request
  and does not move.
- **TECHART** — the LM-EA9 declares 0x34 and answers from a stored 32-byte template
  (`00 00 00 7e 52 03 52 03 … 01 02 10 00 25`), copying the request's `pl[0..1]` into `pl[1..2]`.
  It does not move. The other bytes, including the `52 03 52 03` pair once read as an exit-pupil
  distance, are not read by the A7 IV.
- **Viltrox** — the EF adapter does not declare it.

## Open questions

- What the release-sequence move corrects, and which shooting modes trigger it.
- Which AF situations send each mode; the meaning of `pl[3]` bit 2 alone; the unit of a mode-2
  step; reply `pl[0]` values other than `00`.
- Message `0x4A`, its newer form.
