# Message 0x10 — small init exchange, with a long delay

**Summary.** A 1-byte init exchange in which **the lens reports whether it is ready**. The body's
request differs by device class, and the reply can take about 1.3 seconds — by far the longest gap
in the handshake.

**Direction:** both. Body request 1 byte; lens reply 1 byte.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000 → natives | `1f` |
| B→L | Sony A6000 → Viltrox EF adapter | **`3f`** |
| L→B | Sony SELP1650, SEL55210, Viltrox EF adapter | `00` |
| L→B | TECHART LM-EA9 | implements a reply |

## Reply `pl[0]` bit 0 — not ready

The A7 IV treats bit 0 set as "not ready" and **keeps waiting** in this step; bit 0 clear lets the
handshake proceed to [message 0x0A](msg_0x0A.md). Every lens captured replies `00`. CERTAIN on the
A7 IV firmware.

## The request carries body-side state

The same Sony A6000 sends `1f` to natives and `3f` to the Viltrox adapter. So by this point in the
handshake the body has already classified the device — from the capability bitmap, the identity
message, or the large descriptor — and message 0x10 carries something derived from that
classification.

The A7 IV builds the request from [message 0x07](msg_0x07.md) `pl[0]`: `1f` for a native, `3f` for
an adapter (or `17` in one mode), and `80` on one body variant when 0x07 `pl[25]` bit 2 is set —
matching the A6000's `1f` / `3f`. The individual bits are UNKNOWN. CERTAIN on the A7 IV firmware.

This is the same pattern as message 0x03, where the body varies the frame *length* by device class.
The body adapts its own frames to what the lens declared during init.

## The long delay

On the Sony SELP1650 the request is at ~169 ms and the reply at ~1504 ms. The lens is presumably
initialising hardware in that window; PROBABLE. Nothing else in the handshake comes close to this
gap.

An implementation must not treat a slow 0x10 reply as a failure.

## Manufacturer notes

- **Sony**, **Viltrox** — all reply `00`.
- **TECHART**, **Yongnuo** — both implement the message; Yongnuo's reply is identical across its
  lineup.

## Open questions

- What the individual request bits mean.
- What the lens does during the ~1.3 s before replying.
- Whether there is a timeout after which a body gives up.
