# Frame format

```
F0 | len_lo len_hi | class | seq | id | payload[len-9] | ck_lo ck_hi | 55
```

| Field | Size | Meaning | Confidence |
| --- | --- | --- | --- |
| `F0` | 1 | Start byte | CERTAIN |
| `len` | 2 | **u16 LE total frame length**, including start byte, checksum and terminator | CERTAIN |
| `class` | 1 | `0x02` = init, `0x01` = normal | CERTAIN |
| `seq` | 1 | Sequence number | CERTAIN |
| `id` | 1 | Message ID | CERTAIN |
| `payload` | len−9 | Message body | — |
| `ck` | 2 | u16 LE checksum | CERTAIN |
| `55` | 1 | Terminator | CERTAIN |

The header is 6 bytes: start byte, 16-bit length, class, sequence, message ID.

---

## Checksum — CERTAIN

```python
checksum = sum(frame[1 : len(frame) - 3]) & 0xffff    # len bytes, class, seq, id, payload
frame[-3], frame[-2] = checksum & 0xff, checksum >> 8 # little-endian
frame[-1] = 0x55
```

**Verified on 3 393 of 3 393 observed frames, both directions, zero mismatches.**

Note the checksum covers the **length bytes** but not the start byte or the terminator.

### Implementations recompute the checksum on send

Devices that hold pre-built frames do **not** rely on a stored checksum: the checksum is
recomputed at transmit time from the payload as it is actually sent. This is demonstrable on the
TECHART LM-EA9, several of whose fixed-length frames carry stored checksum values that do not
match their payloads, yet are accepted by the body on the wire.

The practical consequence is that a payload can be altered without any checksum fixup — the
sending side will produce the correct value regardless.

## Class byte — CERTAIN

| Value | Name | Used for |
| --- | --- | --- |
| `0x02` | `MESSAGE_CLASS_INIT` | The startup handshake. Strict request/response, `seq` = 0. |
| `0x01` | `MESSAGE_CLASS_NORMAL` | The 60 Hz status loop. |

## Sequence byte — CERTAIN

- Init frames all use `seq = 0`.
- In the normal loop, **all four frames of one cycle share one sequence number**, and it increments
  by 1 per cycle, wrapping at 256.

Observed directly on a Sony SELP1650:

```
1605.72 ms  L->B cls=01 seq= 99 id=05 len=96
1606.94 ms  L->B cls=01 seq= 99 id=06 len=39
1612.47 ms  B->L cls=01 seq= 99 id=03 len=23
1613.99 ms  B->L cls=01 seq= 99 id=04 len=13
1622.39 ms  L->B cls=01 seq=100 id=05 len=96
```

A session joined mid-stream shows a non-zero starting `seq`, which is consistent.
