# Message 0x34 — implemented, never observed carrying data

**Summary.** A 32-byte response with a duplicated u16 pair whose meaning is UNKNOWN. It holds the
last surviving exit-pupil-distance candidate in the protocol.

**Direction:** UNKNOWN. **Class:** UNKNOWN. **Payload:** 32 bytes.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — `00 00 00 7e 52 03 52 03 00 … 01 02 10 00 25` |

The pair `52 03 52 03` is **850, sent twice** — the same duplicated-u16 shape the protocol uses for
focal length in [message 0x05](msg_0x05.md) and for focus position in [message 0x1B](msg_0x1B.md).

## The last exit-pupil-distance candidate

Two other duplicated pairs were once read as exit pupil distance, and both have been reassigned:

| Pair | Message | Actual meaning |
| --- | --- | --- |
| 500 | [0x28](msg_0x28.md) | **Focal length** in mm × 10 — the same writer fills message 0x05's focal pair and message 0x28's |
| 4448 | [0x1B](msg_0x1B.md) | **A focus position**, sent twice; 4448 sits on message 0x05's `n × 256/3` scale |

That leaves **0x34's 850 as the only surviving candidate**, and it is now a lone data point rather
than a matching pair. UNKNOWN.

## Open questions

- What 850 is. Exit pupil distance is the only hypothesis on the table, and it has no support beyond
  the shape of the field.
- The rest of the 32 bytes.
- Direction and class.
