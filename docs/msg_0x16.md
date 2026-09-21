# Message 0x16 — shutdown request

**Summary.** A body→lens request that the lens shut down. A Sony a9 II sends it when the camera is
switched off, and then **waits for the lens to acknowledge before removing power**. A lens that does
not answer is left powered, and the camera's own power-off is delayed accordingly.

**Direction:** B→L request, L→B response.

**Class:** init (`0x02`). This is unusual: the message arrives at the end of a normal-mode session,
after hundreds of normal-class status frames, but carries the init class rather than `0x01`. A
receiver that dispatches init-class frames only during the opening handshake will not see it.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## The acknowledgement is an echo

The response is **the request frame sent straight back**, with the checksum recomputed over the
returned bytes. It is not a stored reply with contents of its own.

Two consequences for an implementer:

- The response carries the **request's** class and sequence number, not values of the lens's
  choosing.
- A lens can acknowledge correctly **without interpreting the payload at all**, which is what makes
  the message cheap to support.

A repeated 0x16 should be echoed again rather than ignored — the body may simply not have received
the first response.

After the exchange the body stops the 60 Hz frame sync on `BODY_VD_LENS` and then removes power.

## An incomplete shutdown blocks the power-off

The echo is necessary but **not sufficient**. The body requires two things before it removes power:
the echoed response, and a **logic line driven high** by the lens. Which of the ten mount contacts
carries that line is UNKNOWN; it is not `RXD`, `LENS_CS_BODY` or `BODY_VD_LENS`, all of which are
accounted for by the serial interface.

Measured on a Sony a9 II:

| Lens behaviour | Body's response |
| --- | --- |
| does not reply at all | holds power **at least 4100 ms**, sends nothing further, no limit reached inside the measurement window |
| echoes the request, never drives the line | the same — **at least 4100 ms**, no limit reached |
| echoes the request **and** drives the line | removes power within about **700 ms** |

The two failing cases were measured separately, so the echo and the line are **independently
necessary** rather than one substituting for the other. In both, the camera's power-off was delayed
by the same interval, and the following power-on with it.

So message 0x16 is not optional for a device that wants a clean power-off, even though nothing in
its payload needs to be understood.

## Request payload

Contents **UNKNOWN**. No field within the payload has been shown to carry meaning, and because the
acknowledgement is an echo, a working implementation need not decode any of it.

## Response payload

| Implementation | Response |
| --- | --- |
| TECHART LM-EA9 | the request frame, echoed back |
| Yongnuo (whole lineup) | 1-byte payload `00` |

## Manufacturer notes

- **Sony a9 II** — sends message 0x16 at power-off, with class `0x02`. It has also been seen
  mid-session, on a run where the body abandoned the session and cut power; a body that has decided
  something is wrong appears to ask the lens to shut down by the same route.
- **Sony A6000** — not observed. The available traffic for that body covers initialisation and idle
  only, so this is absence of observation, not evidence of absence.
- **Yongnuo** — answers with a 1-byte payload `00`, identical across its lineup.

### Releasing the serial line matters to the next power-on

A lens that leaves `RXD` **actively driven** through power removal, instead of releasing it once the
shutdown has been acknowledged, delays the **following** power-on: the body takes noticeably longer
to begin talking in the next session. Releasing the line removes the delay.

PROBABLE — established by comparing two otherwise identical lens implementations differing only in
this, where the non-releasing one reproduced the delay across five consecutive sessions. The
mechanism is UNKNOWN; the observable difference is that `RXD` goes from driven high to released for
the interval between the acknowledgement and power removal.

## Open questions

- The request payload contents.
- Whether the body's wait for the acknowledgement is bounded at all — the failing case did not reach
  a limit within the measurement window.
- Why the message uses the init class when it occurs at the end of a normal-mode session.
- **Which mount contact carries the power-down acknowledgement line**, and whether native lenses
  signal on the same one. Its existence and its effect on the power-off are established; its
  identity is not.
- A class value of `0x00` has been associated with this message ID in at least one implementation
  but has never been observed on the wire; because the response is an echo, the transmitted class is
  the request's.
- Whether A6000-generation bodies use this message at all.

There is a report of a body sending 0x16 after 0x1D/0x20. Given that [0x1D](msg_0x1D.md) and
[0x20](msg_0x20.md) are almost certainly frame *lengths* of message 0x03 rather than message IDs,
that report reads as **a 22-byte (`0x16`) message 0x04 frame** in the same length notation — which
is exactly the observed message 0x04 length. So it is probably not a sighting of message 0x16.
