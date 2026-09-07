# Message 0x0D — small init exchange

**Summary.** A 1-byte init-class request/response with UNKNOWN semantics.

**Direction:** both. Body request 1 byte; lens reply 1 byte.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `00` |
| L→B | Sony SELP1650, SEL55210, Viltrox EF adapter, TECHART LM-EA9 | `01` |

Every device measured replies `01` to a request of `00`. Yongnuo's reply is identical across its
whole lineup, as are its 0x08, 0x10 and 0x16 replies.

## Manufacturer notes

- **Sony**, **Viltrox**, **TECHART** — all reply `01`.
- **Yongnuo** — one identical reply across three lens models.

## Open questions

- The meaning of the exchange. Nothing observed varies, so nothing constrains a reading.
