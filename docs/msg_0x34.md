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
| `pl[0..1]` | u16 LE **target**, in the units of the [0x06](msg_0x06.md) focus position |
| `pl[2]` | `00` |
| `pl[3]` | bits 0–1: **mode** (`0`, `2` and `3` seen in the code); bit 2 and bits 6–7: further flags, UNKNOWN |
| `pl[4]` | **time limit**: milliseconds left until the body's deadline, capped at 87 for a native lens |
| `pl[5..22]` | zero |

## Lens reply

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | status; `00` = accepted | The A7 IV reads it: **CERTAIN**. Meaning **PROBABLE**: both implementations send `00` |
| `pl[1..2]` | u16 LE **the position the lens is moving to** | The A7 IV reads it: **CERTAIN**. Meaning **CERTAIN** in Yongnuo firmware |
| `pl[3..22]` | zero in both implementations | — |

The A7 IV reads nothing beyond `pl[2]`.

## When the body sends it — CERTAIN on the A7 IV firmware

- **From the release sequence.** The A7 IV's lens sequencer (`BLSequencer`, with states such as
  Focus, FocusWait, Iris, IrisWait and WaitExpStartingRes) sends 0x34 in its Focus step. It moves to
  FocusWait once the request is out. What the move is for is UNKNOWN. Nearby code computes focus-drive
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

## Yongnuo's implementation — CERTAIN in Yongnuo 50 DF firmware

Yongnuo handles 0x34 in its init dispatcher (`FUN_0801620c`):

- **Mode 0, or a target inside the travel.** The step limit is `L = 18 × n` position units, where `n`
  is `pl[4]`, less 13 when it is above 13. If the target is more than `L` from the current position,
  it moves `L` toward it; otherwise to the target. It moves at its maximum speed and replies
  `pl[0] = 00`, `pl[1..2]` = the position it is moving to.
- **Modes 3, 6 and 7 with a target outside the travel.** The target is converted first: mode 3
  through a distance-to-position lookup, modes 6 and 7 through another table. The same step limit
  then applies, with a faster speed setting.

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
- The meaning of modes 2 and 3 on the body side, of the flag bits in `pl[3]`, and of reply `pl[0]`
  values other than `00`.
- Message `0x4A`, its newer form.
