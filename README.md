# The Sony E-mount lens protocol

A consolidated description of the serial protocol spoken between a Sony E-mount camera body and
the lens or adapter mounted on it.

## Purpose

This project documents **what the protocol is** — the electrical interface, the frame format, the
session structure, and the meaning of each message and its payload fields. It is written to be
read on its own and to be sufficient for writing a correct implementation: a parser, a lens
emulator, or an adapter.

The description is **device-independent**. Individual products appear only where a manufacturer's
implementation is worth recording — which messages it supports, where its payloads differ, and
where it deviates from what other implementations do. Those notes are always labelled with the
vendor and model they apply to.

Coverage is uneven and honestly marked. Every claim carries a confidence label, and fields whose
position is known but whose meaning is not are listed as such rather than guessed at.

### Confidence labels

Every field in this documentation carries one of:

| Label | Meaning |
| --- | --- |
| **CERTAIN** | Verified on multiple independent devices, or arithmetically proven. |
| **PROBABLE** | Consistent with all evidence and with at least two devices, but a plausible alternative reading has not been excluded. |
| **POSSIBLE** | One observation, or a pattern that fits but has not been cross-checked. |
| **UNKNOWN** | Field position is known; meaning is not. |

## Contents

- [1. Physical layer](#1-physical-layer)
- [2. Frame format](#2-frame-format)
- [3. Session structure](#3-session-structure)
- [4. Message Catalogue](#4-message-catalogue)

---

# 1. Physical layer

Ten contacts. Pins numbered left to right looking at a lens from the rear:

| Pin | Name | Function | Confidence |
| --- | --- | --- | --- |
| 1 | `LENS_GND` | Motor ground | CERTAIN |
| 2 | `LENS_POWER` | Motor power — 5.0 V or unregulated Vbat (7.4 V nominal). Which one is negotiated somehow. | CERTAIN (mechanism UNKNOWN) |
| 3 | `LOGIC_GND` | Logic ground | CERTAIN |
| 4 | `BODY_VD_LENS` | Body→lens. Normally high, pulses low at low duty cycle at **60 Hz**. | CERTAIN (purpose PROBABLE: vertical-drive / frame sync — the main loop runs at this rate) |
| 5 | `LOGIC_VCC` | 3.15 V logic supply; all data lines are 3.15 V logic | CERTAIN |
| 6 | `LENS_CS_BODY` | Chip select, lens→body. High while `RXD` transfers. | CERTAIN |
| 7 | `RXD` | Serial data lens→body | CERTAIN |
| 8 | `TXD` | Serial data body→lens | CERTAIN |
| 9 | `BODY_CS_LENS` | Chip select, body→lens. High while `TXD` transfers. | CERTAIN |
| 10 | `LENS_XDETECT` | Pulled high by the body. **The Sony SEL55210 shorts it to ground; the Viltrox EF-NEX II grounds it through 680 Ω.** | CERTAIN (values); PURPOSE POSSIBLE — suspected native-vs-adapter detection, unproven |

## UART parameters

- **8N1, LSB-first**, no parity. CERTAIN.
- Starts at **750 kbaud**, negotiates up to **1.5 Mbaud** during init. CERTAIN.
- Data flows **only while the matching CS line is high**, and exactly **one frame per CS-high
  period**. CERTAIN — framing therefore needs no heuristic, the CS line delimits it.
- During the speed change **both CS lines go high with no data for ~5 ms**. This is the only time
  both are high. CERTAIN, and it is a reliable marker for the switch point.

---

# 2. Frame format

```
F0 | len_lo len_hi | class | seq | id | payload[len-9] | ck_lo ck_hi | 55
```

| Field | Size | Meaning | Confidence |
| --- | --- | --- | --- |
| `F0` | 1 | Start byte | CERTAIN |
| `len` | 2 | **u16 LE total frame length**, including start byte, checksum and terminator | CERTAIN |
| `class` | 1 | [Class byte](docs/frame_format.md#class-byte--certain): `0x02` = init, `0x01` = normal | CERTAIN |
| `seq` | 1 | [Sequence byte](docs/frame_format.md#sequence-byte--certain) | CERTAIN |
| `id` | 1 | Message ID | CERTAIN |
| `payload` | len−9 | Message body | — |
| `ck` | 2 | [Checksum](docs/frame_format.md#checksum--certain) | CERTAIN |
| `55` | 1 | Terminator | CERTAIN |

The header is 6 bytes: start byte, 16-bit length, class, sequence, message ID.

Throughout this documentation **`pl` is the payload**, so **`pl[n]` is payload byte `n`** —
absolute frame offset `n + 6`, since the header is 6 bytes.

Ranges are written **`pl[a..b]`, inclusive of both ends**: from `pl[a]` up to and including
`pl[b]`. The length of the field is therefore `b - a + 1`. So `pl[0..7]` is the eight bytes
`pl[0]` through `pl[7]`, and the next field starts at `pl[8]`.


---

# 3. Session structure

## 3.1 Init phase — class `0x02`, strict request/response

The body sends a short query; the lens replies with the same ID and a longer payload. Order
observed on a Sony A6000, identical every time:

```
B->L 0x01  (32 B)  ->  L->B 0x01  (32 B)     capability bitmap exchange
B->L 0x07  ( 1 B)  ->  L->B 0x07  (34 B)     identity / lens ID
B->L 0x0c  ( 1 B)  ->  L->B 0x0c  ( 1 B)
B->L 0x0b  ( 2 B)  ->  L->B 0x0b  ( 2 B)
B->L 0x08  ( 8 B)  ->  L->B 0x08  (201 B)    the big descriptor
B->L 0x09  ( 4 B)  ->  L->B 0x09  (11 B)
B->L 0x0d  ( 1 B)  ->  L->B 0x0d  ( 1 B)
B->L 0x10  ( 1 B)  ->  L->B 0x10  ( 1 B)
B->L 0x0a  (16 B)  ->  L->B 0x0a  (16 B)
```

The gap between the `0x0c` and `0x0b` exchanges (~143 ms → ~164 ms) is where the baud-rate
negotiation sits. The `0x10` reply is separated from its request by a long delay
(169 ms → 1504 ms on the SELP1650) — the lens is presumably initialising hardware. PROBABLE.

**Asymmetry is the norm.** The body's request payload is usually a stub (1 byte, often `00`); the
lens's reply carries the data. Message 0x08 is the extreme case: 8 bytes out, 201 bytes back.

## 3.2 Normal loop — class `0x01`, ~60 Hz

Four frames per cycle, **lens first**, one shared `seq`, period ≈ 16.7 ms — matching the
`BODY_VD_LENS` 60 Hz pulse:

| Order | Dir | ID | Length (payload) |
| --- | --- | --- | --- |
| 1 | L→B | **0x05** | 96 B, or 108 B in the [117-byte frame variant](docs/msg_0x05.md#two-payload-sizes) |
| 2 | L→B | **0x06** | 39 B |
| 3 | B→L | **0x03** | 23 B native / 20 B to the Viltrox adapter |
| 4 | B→L | **0x04** | 13 B |

CERTAIN — this ordering holds for every cycle ever observed.

Two things follow. First, **the lens reports before the body commands**, so 0x05/0x06 are status
and 0x03/0x04 are the body's response to them, not the other way round. Second, the body varies
the length of message 0x03 by device class (20 B to the Viltrox, 23 B to both Sony natives, from
the same A6000) — so the **body adapts its own frames to what the lens declared during init**.
PROBABLE, and worth remembering before assuming a fixed layout.

---

# 4. Message Catalogue

| ID | Direction | Class | Function description |
| --- | --- | --- | --- |
| **[0x01](docs/msg_0x01.md)** | both | init | capability bitmap |
| **[0x03](docs/msg_0x03.md)** | B→L | normal | body status / command, per frame |
| **[0x04](docs/msg_0x04.md)** | B→L | normal | body mode block, per frame |
| **[0x05](docs/msg_0x05.md)** | L→B | normal | lens status + optical table transfer |
| **[0x06](docs/msg_0x06.md)** | L→B | normal | focus range, subject distance, event channel |
| **[0x07](docs/msg_0x07.md)** | both | init | identity and lens ID |
| **[0x08](docs/msg_0x08.md)** | both | init | the large descriptor |
| **[0x09](docs/msg_0x09.md)** | both | init | small init exchange |
| **[0x0A](docs/msg_0x0A.md)** | both | init | init echo |
| **[0x0B](docs/msg_0x0B.md)** | both | init | small init exchange |
| **[0x0C](docs/msg_0x0C.md)** | both | init | small init exchange |
| **[0x0D](docs/msg_0x0D.md)** | both | init | small init exchange |
| **[0x10](docs/msg_0x10.md)** | both | init | small init exchange, long reply delay |
| **[0x16](docs/msg_0x16.md)** | ? | ? | unknown; the only `class = 0x00` frame |
| **[0x19](docs/msg_0x19.md)** | ? | ? | unknown |
| **[0x1B](docs/msg_0x1B.md)** | both | normal | command channel — focus target or aperture |
| **[0x1D](docs/msg_0x1D.md)** | B→L | ? | probably a frame length, not an ID |
| **[0x20](docs/msg_0x20.md)** | B→L | ? | probably a frame length, not an ID |
| **[0x28](docs/msg_0x28.md)** | L→B | init | third carrier for the correction rows |
| **[0x32](docs/msg_0x32.md)** | ? | ? | implemented but not advertised |
| **[0x34](docs/msg_0x34.md)** | ? | ? | unknown; holds the 850 pair |
| **[0x35](docs/msg_0x35.md)** | L→B | init | second carrier for the correction rows |
| **[0x3D](docs/msg_0x3D.md)** | ? | ? | unknown |
| **[0x3F](docs/msg_0x3F.md)** | L→B | init | lens name string |
| **[0x4C](docs/msg_0x4C.md)** | ? | ? | beyond the capability bitmap |
| **[0x5A](docs/msg_0x5A.md)** | ? | ? | beyond the capability bitmap |

Messages **0x05, 0x28 and 0x35 all carry the same 6-byte optical rows**, in three different
layouts. They have their own reference: [the 6-byte optical rows](docs/optical_data.md).

Those same three messages, plus the **0x1B** command channel, all carry the **live focus position** —
an `n × 256/3` distance index shared by every device on the bus, and emphatically *not* the lens's
own encoder count. It has its own reference too:
[the live focus position](docs/live_focus_position.md).
