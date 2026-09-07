# Message 0x09 — small init exchange

**Summary.** One of six short init-class request/response exchanges whose semantics are UNKNOWN.
Documented because the values discriminate between devices.

**Direction:** both. Body request 4 bytes; lens reply 11 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `00 00 00 00` |
| L→B | Sony SELP1650 | `10 00 00 00 00 00 00 00 00 fd ff` |
| L→B | Sony SEL55210 | all zero |
| L→B | Viltrox EF adapter | `01 01 00 …` |
| L→B | TECHART LM-EA9 | `01 01 00 …` |

The two Sony natives disagree with each other, so the reply is not a fixed native constant. The
adapter and the LM-EA9 agree on the `01 01 00` opening.

## Manufacturer notes

- **Sony** — the SELP1650 and SEL55210 send different replies; the SEL55210's is all zero.
- **TECHART**, **Viltrox** — both open `01 01 00`.
- **Yongnuo** — implements the message; its reply is identical across its lineup.

## Open questions

- The meaning of the whole exchange. UNKNOWN.
- Why the two Sony natives reply differently.
