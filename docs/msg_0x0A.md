# Message 0x0A — init echo

**Summary.** A 16-byte init-class exchange in which **the lens echoes the body's request exactly**.
The last exchange of the init handshake.

**Direction:** both. Body request 16 bytes; lens reply 16 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | Payload |
| --- | --- | --- |
| B→L | Sony A6000, every lens | `ff 7f 00 00 00 00 00 00 3f 00 …` |
| L→B | Sony SELP1650, SEL55210, Viltrox EF adapter | **echoes the request exactly** |
| L→B | TECHART LM-EA9 | `ff 7f … 3f …` — the same shape |

## It is an exact echo

Every device measured returns the body's 16 bytes unchanged. That makes it plausibly a **link test
or echo/loopback check** — a way for the body to confirm the newly negotiated 1.5 Mbaud link carries
16 bytes intact in both directions before the status loop starts. PROBABLE.

Its position supports the reading: 0x0A is the final init exchange, after the baud-rate change and
after every descriptor has been transferred.

## Manufacturer notes

- **Sony**, **Viltrox**, **TECHART** — all echo. No device measured deviates.

## Open questions

- Whether it really is a link test, or whether the body reads meaning into the returned bytes.
- What the request constant `ff 7f … 3f` encodes, if anything.
- Whether a device that returned altered bytes would be rejected.
