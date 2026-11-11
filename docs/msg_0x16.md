# Message 0x16 — shutdown request

**Summary.** A body→lens request that the lens shut down, sent when the camera is switched off.
The body waits for a reply, cuts the motor supply, and then **waits for `RXD` to go low** before it
finishes powering the lens down. A lens that does not reply, or keeps `RXD` high, delays the
camera's power-off.

**Direction:** B→L request, L→B response.

**Class:** init (`0x02`), although it arrives at the end of a normal-mode session. A receiver that
dispatches init-class frames only during the opening handshake will not see it.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Request payload — CERTAIN on the A7 IV firmware

One byte. `pl[0] = 01`, or `03` when the body's system state is `0xc05` (meaning UNKNOWN).

## Response

The A7 IV accepts **any frame with ID 0x16**; it reads none of its contents. Known replies:

| Implementation | Response |
| --- | --- |
| Yongnuo (whole lineup) | fixed frame, class `02`, seq `00`, 1-byte payload `00` |
| TECHART LM-EA9 | the request frame echoed back, checksum recomputed |

A repeated 0x16 should be answered again; the body may not have received the first reply.

## The power-off sequence — CERTAIN on the A7 IV firmware

The lens interface's power state machine runs, in order:

1. Send [message 0x0A](msg_0x0A.md#00-00-at-the-end-of-the-session) with an all-zero payload, and
   wait for its reply.
2. Send 0x16, and wait for a reply with ID 0x16.
3. **Cut the motor supply** (`LENS_POWER`). Anything the lens wants to do with its motor, such as
   parking, must happen **before** it replies.
4. Drop `BODY_CS_LENS`, switch the body's UART off, and turn `TXD` and `RXD` back into plain GPIO.
5. **Read `RXD` every 5 ms until it is low.** Give up after 6000 reads and log
   `[HS]POWER OFF SEQ TIME OUT`; the power-off then continues anyway.
6. Drive `TXD` low. The lens is now off as far as this layer is concerned.

The body does not wait on `BODY_VD_LENS` or `LENS_CS_BODY` in this sequence.

Step 5 is the mirror of power-on. There the body raises `TXD` and waits for `RXD` to go **high**,
i.e. for the lens's UART to be idling.

**The limit in step 5 is about 30 s.** The 5 ms poll interval is PROBABLE. It assumes microsecond
units, which also make the power-on handshake's two timeout settings equal (5000 × 400 and
1000 × 2000, both 2 s).

### After 0x16: the body keeps the lens — CERTAIN on the a7C II firmware

There is no separate "sleep" message: standby and power-off both use 0x16. What differs is where
the body parks afterwards:

| Condition | Lens logic power | Next power-on |
| --- | --- | --- |
| system state `0xc05` (`pl[0] = 03`) | kept | wake sequence; if the state has meanwhile become `0xc06`, a native lens is woken and sent 0x16 again |
| body controls lens power directly (a per-model setting) | **cut** | wake sequence |
| otherwise | kept | wake sequence |

In all three the body still treats the lens as attached and keeps its cached 0x07/0x08 data, so the
next power-on runs the short
[wake sequence](../README.md#wake-the-short-sequence-after-0x16): `0B` first, at the fast speed,
with no 0x01. If `LENS_XDETECT` (pin 10) goes high in the meantime, the cache is cleared and the next
power-on is a full init.

A lens that loses power at shutdown restarts at 750 kbaud and cannot answer the wake sequence. The
body retries with a full init, so the lens still comes up, one or more power cycles later.
PROBABLE: follows from the firmware, not captured.

### Measured on a Sony a9 II

| Lens behaviour after the request | Body |
| --- | --- |
| no reply | holds power **at least 4100 ms** |
| replies, `RXD` left idling high | holds power **at least 4100 ms** |
| replies, then runs its own power-down step | removes power within about **700 ms** |
| replies, switches its UART off and drives its TX line (`RXD`) low, nothing else | no noticeable delay; two power cycles |

This matches steps 2 and 5. Neither measured hold reached the 30 s limit. The last row is the same
lens as the third with its power-down step replaced by `RXD` low alone, so `RXD` low is sufficient.
The power-off time in that row is the user's observation, not a recorded figure. Switching the UART
off without driving the line has not been tried.

### Releasing the serial line matters to the next power-on

A lens that keeps its UART transmitter driving `RXD` through power removal delays the **following**
power-on: the body takes noticeably longer to begin talking in the next session. Switching the UART
off after the reply removes the delay. PROBABLE: shown on two otherwise identical lens
implementations that differed only in this, with the delay reproduced across five consecutive
sessions. The mechanism is UNKNOWN.

## Manufacturer notes

- **Sony a9 II** sends 0x16 at power-off. It has also been seen mid-session, on a run where the body
  abandoned the session and cut power.
- **Sony A6000**: not observed. The available traffic covers initialisation and idle only.
- **Yongnuo** answers with the fixed 1-byte frame above.
- **TECHART LM-EA9** echoes the request, switches its UART off and drives an internal pin (PA23).
  How that pin brings `RXD` low on the adapter's board is not established.

## Open questions

- What system states `0xc05` (`pl[0] = 03`) and `0xc06` are. From the body's behaviour, `0xc05`
  looks like a standby the camera may resume from and `0xc06` like the later move to fully off.
  PROBABLE at best.
- What the lens is expected to do on `pl[0]` bit 1, if anything.
- Whether A6000-generation bodies use this message at all.

There is a report of a body sending 0x16 after 0x1D/0x20. [0x1D](msg_0x1D.md) and
[0x20](msg_0x20.md) are almost certainly frame *lengths* of message 0x03, not message IDs. In the
same length notation, the report reads as a 22-byte (`0x16`) message 0x04 frame, which is exactly
the observed message 0x04 length. So it is probably not a sighting of message 0x16.
