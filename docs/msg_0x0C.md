# Message 0x0C — baud-rate negotiation

**Summary.** The lens's reply sets the bus speed for the rest of the session: `01` = switch to
1.5 Mbaud, `00` = stay at 750 kbaud. The last exchange at 750 kbaud. **CERTAIN.**

**Direction:** both. Body request 1 byte; lens reply 1 byte.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## Observed payloads

| Direction | Device | Payload | Speed afterwards |
| --- | --- | --- | --- |
| B→L | Sony A6000, every lens | `02` | — |
| L→B | Sony SELP1650, SEL55210, Viltrox EF adapter | `01` | 1.5 Mbaud |
| L→B | Yongnuo (firmware) | `01` | 1.5 Mbaud |
| L→B | TECHART LM-EA9 | `00` | 750 kbaud |

## Reply `pl[0]`

| Value | Meaning |
| --- | --- |
| `01` | Switch to 1.5 Mbaud from the next exchange (0x0B) on |
| `00` | Stay at 750 kbaud |

The lens sends the reply at 750 kbaud, waits for it to finish transmitting, then reconfigures its
UART. The body switches on `01` regardless of whether the lens does, so a lens that answers `01`
must actually switch — otherwise the body stops talking after this message (seen on an a9 II).

## Request `pl[0]` — the speed the body asks for

The request is one bit, mirroring the speed bits the lens declared in
[message 0x07](msg_0x07.md) `pl[1]`: `02` asks for speed 1 (`pl[1]` bit 1), `04` for speed 2
(`pl[1]` bit 2). The A7 IV picks the fastest speed both sides support, and **does not send 0x0C at
all** if the lens declared neither. CERTAIN on the A7 IV firmware; every device captured declares
bit 1 and receives `02`, consistent with it.

The A7 IV tests only bit 0 of the reply: clear means "stay at the base speed" (log `UART_HZ NG`)
and the handshake carries on. CERTAIN on the A7 IV firmware.

Yongnuo ignores the request byte.

## Open questions

- The baud rate of speed 2. No device has been seen to declare it.
