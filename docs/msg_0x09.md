# Message 0x09 — small init exchange

**Summary.** A short init exchange, called **START UP INFO** in Sony's body firmware. The body
checks two bytes of the reply and, for adapters, reads a sub-type from one more.

**Direction:** both. Body request 4 bytes; lens reply 11 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Field map — lens reply

What the Sony A7 IV firmware does with the reply:

| Field | Meaning | Confidence |
| --- | --- | --- |
| **`pl[0] & 3`** | **Adapter sub-type**, read only when [message 0x07](msg_0x07.md) `pl[0]` declared an adapter: `1` and `2` select two different adapter classes; natives ignore it | Read **CERTAIN** on the A7 IV firmware; what the classes are UNKNOWN |
| **`pl[7]`, `pl[8]`** | **Must be `00`.** Anything else fails the handshake (log `SYNC START UP INFO ERR`), except on one body variant | **CERTAIN** on the A7 IV firmware |
| other bytes | Stored, not read by the handshake | — |

The body's 4-byte request is not always zero on the A7 IV: `pl[0]` can carry a saved value, or `01`
for one Sony lens ID. Meaning UNKNOWN.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `00 00 00 00` |
| B→L | Sony a9 II | `00 00 00 00` |
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

- What the two adapter sub-types are.
- Why the two Sony natives reply differently in bytes the A7 IV does not read.
