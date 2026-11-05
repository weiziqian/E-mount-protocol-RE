# Message 0x2B — adapter re-identification

**Summary.** A one-byte init-class exchange an adapter uses to confirm its **sub-type** with the
body during a session. The body sends the sub-type it believes, the adapter answers with its own,
and the body re-classifies the adapter from the answer. Only adapters declare it. Known from body
firmware; never captured on the wire.

**Direction:** both. Request 1 byte, reply 1 byte (the A7 IV's length table gives `2` / `2` with
the ID byte). **Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## The adapter sub-type

One 2-bit code, carried in three places, all decoded the same way by the A7 IV:

| Code | Adapter class the body assigns |
| --- | --- |
| `0` | adapter, basic |
| `1` | adapter that supplies lens data — the body re-reads [message 0x08](msg_0x08.md) |
| `2` | adapter, a second class |
| `3` | no change |

| Carrier | When |
| --- | --- |
| [message 0x09](msg_0x09.md) `pl[0] & 3` | once, during the handshake |
| [message 0x06](msg_0x06.md) `pl[0] & 3` | every frame |
| this message, `pl[0] & 3` | on re-identification |

All three are read **only** when [message 0x07](msg_0x07.md) `pl[0]` declares an adapter (`02`).
For a native lens the bits mean nothing to the body. What separates classes `0` and `2` is UNKNOWN.

Mapping and carriers **CERTAIN** on the A7 IV firmware; the reading of class `1` as "supplies lens
data" is **PROBABLE** (it is the only class that re-reads 0x08 and gets the lens identity fields of
0x08 read).

## When the body sends it

The A7 IV runs a short re-identification sequence — `0x0A`, then `0x2B` — in two cases:

1. **The sub-type in an adapter's message 0x06 changes** to one that maps to a different class
   than the current one. The most likely cause is the lens behind the adapter being swapped.
2. The body's main processor asks for it. What triggers that is not visible from the
   lens-control firmware.

## Exchange

| Step | Body | Lens |
| --- | --- | --- |
| 1 | request, `pl[0]` = the sub-type it last saw | — |
| 2 | — | reply, `pl[0]` = its actual sub-type |
| 3a | reply ≠ request: adopts the lens's value, sends 0x2B again | — |
| 3b | reply = request: assigns the class above; for `1` requests [0x5A](msg_0x5A.md) (if declared) and **0x08**, then completes the handshake tail; otherwise sends [0x0A](msg_0x0A.md) and resumes | — |

No outcome leaves the device classed as native. CERTAIN on the A7 IV firmware.

## Implementing it

- **A native lens should not declare 0x2B.** No Sony native or Yongnuo lens measured does, and the
  body only runs this sequence for adapters.
- **An adapter that declares it must answer it.** The body will not send an undeclared ID, but once
  in this sequence it waits for the reply with no timeout of its own; a declared, unanswered 0x2B
  leaves the session stuck.
- **Keep the three sub-type carriers consistent.** If 0x06 `pl[0] & 3` disagrees with what 0x09
  established, the body enters this sequence; if the reply then disagrees again, it repeats.

## Manufacturer notes

- **TECHART**, **Viltrox** — both declare 0x2B (and 0x29, 0x2A) in [message 0x01](msg_0x01.md).
  Both send 0x06 `pl[0] = 00`, sub-type `0`.
- **Sony**, **Yongnuo** — do not declare it.

## Open questions

- What distinguishes classes `0` and `2`.
- What makes the main processor request a re-identification.
