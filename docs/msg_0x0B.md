# Message 0x0B — small init exchange

**Summary.** A 2-byte init-class request/response with UNKNOWN semantics. The exchange immediately
after the baud-rate negotiation.

**Direction:** both. Body request 2 bytes; lens reply 2 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `60 00` |
| L→B | Sony SELP1650 | `60 20` |
| L→B | Sony SEL55210 | **`60 00`** |
| L→B | Viltrox EF adapter | `60 00` |
| L→B | TECHART LM-EA9 | `60 20` |

## It is not a native/adapter discriminator

The obvious reading — that `60 20` marks a native and `60 00` an adapter — does not hold. The Sony
SEL55210, a native, replies `60 00`, the same as the Viltrox adapter; and the LM-EA9, an adapter,
replies `60 20`, the same as the SELP1650. The split runs across both groups.

Whatever `pl[1]` selects, it is a per-lens property, not a device class.

## Position in the handshake

The gap between the 0x0C and 0x0B exchanges (~143 ms → ~164 ms) is where the **baud-rate
negotiation** sits, so 0x0B is the first exchange at the higher rate.

## Manufacturer notes

- **Sony** — the two natives measured disagree: SELP1650 `60 20`, SEL55210 `60 00`.
- **TECHART** — the LM-EA9 replies `60 20`.
- **Viltrox** — replies `60 00`.

## Open questions

- What `pl[1]` = `0x20` versus `0x00` selects.
- Whether the body acts on the reply at all.
