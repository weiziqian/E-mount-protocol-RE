# Message 0x4C — a version check in the handshake

**Summary.** An optional init exchange. The body compares the first two reply bytes with its own
limits and **aborts the handshake** if the lens's are higher.

**Direction:** both. **Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## What is known

| Manufacturer | Response |
| --- | --- |
| TECHART | LM-EA9 — 16 bytes, all zero |
| Yongnuo | 7 bytes — `28 28 00 …` |

## Where it sits in the handshake

The A7 IV requests 0x4C after [message 0x0C](msg_0x0C.md) and before 0x0B, but only if the lens
declares it in [message 0x01](msg_0x01.md) — at bit 75, `pl[9]` bit 3. It is expressible like any
other ID; the 32-byte bitmap reaches `0x100`. CERTAIN on the A7 IV firmware.

## Reply `pl[0]`, `pl[1]` — compared against the body

Each must be less than or equal to a corresponding limit held by the body; if either is higher, the
A7 IV aborts the handshake. The pair is also forwarded to the main processor. This reads as a
minimum-body-version requirement declared by the lens. Check **CERTAIN** on the A7 IV firmware;
the version reading **PROBABLE**. The body's limit values are set at runtime and not known.

`pl[3..4]` are read elsewhere; meaning UNKNOWN.

## Open questions

- The body's limit values.
- What Yongnuo's `28 28` opening encodes as a version.
