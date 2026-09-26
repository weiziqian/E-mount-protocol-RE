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

## The transfer window may exceed the frame

A frame is delimited by its chip-select line, but the **number of bytes clocked inside that window
is not necessarily the length of the frame**.

### The normal loop

A Sony a9 II clocks **32 bytes** in the body→lens windows of the normal loop, whatever the frame
inside declares:

| Frame | `len` | Bytes in the window | Bytes after the terminator |
| --- | --- | --- | --- |
| message 0x03 | 29 | 32 | 3 |
| message 0x04, short form | 22 | 32 | 10 |
| message 0x04, tagged form | 23 | 32 | 9 |
| message 0x04, target form | 27 | 32 | 5 |

CERTAIN on that body: the trailing bytes are genuinely transmitted, not an artefact of the
receiver. A receiver that had pre-filled its buffer with a marker byte found none of the marker
left in any window. Confirmed again on a second receiver, which captured whole windows with their
declared length beside the byte count:

```
id 0x03, len 29, window 32:  F0 1D 00 01 9B 03 ... FE 01 55 | 00 00 00
id 0x04, len 22, window 32:  F0 16 00 01 9B 04 ... 80 01 55 | 00 00 00 00 00 00 00 00 00 00
```

The padding is zero in both, and none of it was the marker, so the body really does clock it.

### 32 is not the only window size

Counting the distinct window sizes across whole sessions on the same body gives more than one, and
32 is not always among the first seen:

| Session | Distinct window sizes, in order of first appearance |
| --- | --- |
| 1 | 48, 16, 32 |
| 2 | 48, 16, 32, 27 |
| 3 | 48, 16, 32 |
| 4 | 48, 16, 32, 23 |

`48` and `16` appear before `32` in every session, and the init handshake precedes the normal loop,
so the two are **probably** init-class windows — but which frame arrives in which window has not
been measured, only inferred from the order. `23` and `27` appear late and only in some sessions.

| Aspect | Confidence |
| --- | --- |
| Windows of 16, 23, 27, 32 and 48 bytes all occur on one body in one session | **CERTAIN** |
| 32 is the size used throughout the normal loop | **CERTAIN** |
| 48 and 16 belong to the init handshake | **PROBABLE** |
| What determines the size of any given window | **UNKNOWN** |

### Windows shorter than the frame also occur

Three of those four sessions recorded exactly one window carrying **fewer** bytes than the frame in
it declared. One per session is too few to characterise and too consistent to dismiss.

### Parse by `len`, and clamp to what arrived

**Parse by `len`, never by the byte count of the window.** The frame is self-contained, its
declared length is honest, and its checksum verifies over exactly that length. A receiver that
treats "bytes received" as "frame length" is wrong on every frame this body sends.

For a message whose payload is a [record stream](autofocus.md#2-message-0x04--lengths-and-records)
this is not a matter of tidiness. Walking records to the end of the *window* takes in the checksum,
the terminator and the padding, and `0x55` — the terminator — is a perfectly good operand byte. A
receiver that did so manufactured focus instructions out of its own trailer: in one session, 1019
of 1019 message 0x04 frames produced a spurious record, including four Drives and eight queries the
body never sent, and one invented Move whose operand was the checksum's high byte and the
terminator. It drove the focus group the full length of its travel.

So the bound is **both**: `min(len, bytes received)`. The declared length alone is wrong when a
window is short of it, and the received count alone is wrong whenever the window is padded.

The content of the trailing bytes is UNKNOWN. After the short 0x04 form they are zero; after the
tagged and target forms they carry a checksum-shaped byte pair and a `0x55` at window offset 28 —
the position a 29-byte frame would put its terminator. A fixed-size transmit buffer retaining the
tail of a previous, longer frame would explain that, but the short 0x04 form should then show the
same residue and it does not. Recorded as an open question; nothing in the protocol depends on it.

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
