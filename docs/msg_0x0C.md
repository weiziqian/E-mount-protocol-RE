# Message 0x0C — small init exchange

**Summary.** A 1-byte init-class request/response with UNKNOWN semantics. The third exchange of the
handshake on every body observed, immediately before the baud-rate negotiation.

The reply value matters: a TECHART LM-EA9 answering `01` instead of `00` causes a Sony a9 II to
**abandon the handshake at this message** and issue no further request. The body treats the wrong
value as a fatal answer rather than ignoring it, and the failure is indistinguishable from a lens
that never replied at all. CERTAIN on that body.

**Direction:** both. Body request 1 byte; lens reply 1 byte.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `02` |
| L→B | Sony SELP1650, SEL55210, Viltrox EF adapter | `01` |
| L→B | TECHART LM-EA9 | `00` |

Every device measured replies with the same single byte, so this exchange carries no per-device
information on the wire.

## Position in the handshake

The gap **after** this exchange (~143 ms → ~164 ms, before 0x0B) is where the baud-rate negotiation
sits. So 0x0C is the last exchange at the initial 750 kbaud.

## Manufacturer notes

- **Sony**, **Viltrox** — all reply `01`.
- **TECHART**, **Yongnuo** — both implement the message.

## Open questions

- The meaning of the request byte `02` and the reply byte `01`.
- Whether the reply is checked at all, given every device sends the same value.
