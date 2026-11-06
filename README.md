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

**"CERTAIN on the A7 IV firmware"** marks behaviour read directly from the code of a Sony body —
the A7 IV (ILCE-7M4) 6.02 camera-processor firmware. It is exactly what that body checks, not an
inference from traffic; whether other bodies check the same is not established.

**"on the a7S III firmware"** marks the same kind of evidence from a second body, the Sony a7S III
(ILCE-7SM3) camera-processor firmware, read by static analysis.

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
- Starts at **750 kbaud**, negotiates up to **1.5 Mbaud** in [message 0x0C](docs/msg_0x0C.md). CERTAIN.
- Data flows **only while the matching CS line is high**, and exactly **one frame per CS-high
  period**. CERTAIN — framing therefore needs no heuristic, the CS line delimits it.
- During the speed change **both CS lines go high with no data for ~5 ms**. This is the only time
  both are high. CERTAIN, and it is a reliable marker for the switch point.
- A transfer window may be **larger than the frame inside it**, and occasionally smaller. A Sony
  a9 II clocks 32 bytes in every body→lens window of the normal loop regardless of frame length, so
  a 22-byte frame arrives followed by 10 further bytes; other sizes occur elsewhere in a session.
  Parse by the frame's own `len`, clamped to the bytes actually received — neither alone is safe.
  See [the transfer window](docs/frame_format.md#the-transfer-window-may-exceed-the-frame).
  CERTAIN on that body.

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
B->L 0x0C  ( 1 B)  ->  L->B 0x0C  ( 1 B)
B->L 0x0B  ( 2 B)  ->  L->B 0x0B  ( 2 B)
B->L 0x08  ( 8 B)  ->  L->B 0x08  (201 B)    the big descriptor
B->L 0x09  ( 4 B)  ->  L->B 0x09  (11 B)
B->L 0x0D  ( 1 B)  ->  L->B 0x0D  ( 1 B)
B->L 0x10  ( 1 B)  ->  L->B 0x10  ( 1 B)
B->L 0x0A  (16 B)  ->  L->B 0x0A  (16 B)
```

The gap between the `0x0C` and `0x0B` exchanges (~143 ms → ~164 ms) is the baud-rate switch. The `0x10` reply is separated from its request by a long delay
(169 ms → 1504 ms on the SELP1650) — the lens is presumably initialising hardware. PROBABLE.

### Later bodies request more, in the same order

A Sony a9 II runs the **same sequence with two messages inserted**, and changes nothing else:

```
A6000    0x01  0x07  0x0C  0x0B              0x08  0x09  0x0D  0x10  0x0A     9 exchanges
a9 II    0x01  0x07  0x0C  0x0B  0x3F  0x3D  0x08  0x09  0x0D  0x10  0x0A    11 exchanges
```

The insertions are **[0x3F](docs/msg_0x3F.md), the lens name string**, and
**[0x3D](docs/msg_0x3D.md)**, both requested between `0x0B` and `0x08`. Neither is requested by the
A6000. CERTAIN on both bodies.

Two consequences for anyone implementing a lens:

- **The A6000 sequence is one body's, not the protocol's.** A device that answers only those nine
  IDs stalls on a later body, and the stall is indistinguishable from a dead lens — the body stops
  requesting and drops the device.
- **A message being absent from the A6000 handshake says nothing about whether it is needed.**
  `0x3F` and `0x3D` sit beyond the range the A6000 offers in its
  [capability bitmap](docs/msg_0x01.md), yet a later body asks for both.

### The A7 IV firmware: most steps are optional

The Sony A7 IV's handshake is a state machine in which most steps are taken **only if the lens
declared the message** in its [capability bitmap](docs/msg_0x01.md) (and the body supports it too):

```
01  07  [4E]  [0C]  [4C]  0B  [3F]  [32]  [3D]  [5A]  08  09  0D  10  0A
```

`[ ]` marks an optional step. `0x0C` additionally needs a UART speed declared in
[0x07](docs/msg_0x07.md) `pl[1]`. A lens declaring `0C 3F 3D` and none of `4E 4C 32 5A` gets exactly
the a9 II sequence above. CERTAIN on the A7 IV firmware.

The same code shows where a body **ends the handshake** on the lens's answer:

| Message | Condition | Result |
| --- | --- | --- |
| [0x07](docs/msg_0x07.md) | `pl[0] & 3` is `0` or `3` | abort (`LENS UNKNOWN`) |
| [0x07](docs/msg_0x07.md) | `pl[4] ≠ 0` | abort (`LENS FIRMERR`) |
| [0x4C](docs/msg_0x4C.md) | `pl[0]` or `pl[1]` above the body's limit | abort |
| [0x09](docs/msg_0x09.md) | `pl[7]` or `pl[8]` non-zero | fail (`SYNC START UP INFO ERR`) |
| [0x10](docs/msg_0x10.md) | `pl[0]` bit 0 set | wait — the lens is not ready |
| [0x4E](docs/msg_0x4E.md) | `pl[0]` bit 0 set | leave the normal handshake |

A separate, shorter sequence is used on wake from sleep; it has not been analysed. An adapter can
also be re-identified mid-session with `0x0A` + [`0x2B`](docs/msg_0x2B.md).

Two more conditions surround the message exchange:

- **A pin handshake before the first frame.**
  - The body steps two of its lines and waits for two of the lens's, in four steps.
  - It polls every 5 ms, up to 400 times (1 ms × 2000 on one sensor variant), so the lens has
    about 2 s to respond.
  - If the lens misses that window, the body logs `SetupMode` and fails the lens.
  - The same handshake runs again for the [0x0C](docs/msg_0x0C.md) speed change.
- **Every init request must be answered.**
  - A missing reply ends the handshake, and so does one that fails the frame checks: start byte
    `F0`, length, checksum, `55` terminator, class 1–3, no class-1 frame during init.
  - The reply timer's raw value is `0xE4E1C`; its unit is not established.

| Check | Confidence |
| --- | --- |
| Both conditions | **CERTAIN on the a7S III firmware** |
| Pin-handshake timing (about 2 s, if the timer unit is the microsecond the 5000 / 1000 values suggest) | **PROBABLE** |

`0x0A` remains the last exchange on both bodies, and answering it is what starts the normal loop.
After it, the a9 II sends **no further init-class request** for the rest of the session.

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

**The body sends exactly two frames per `BODY_VD_LENS` cycle**, one 0x03 and one 0x04. Measured on
an a9 II over a 9.5 s session: 123.5 body→lens frames per second against 61.7 VD cycles per second,
a ratio of 2.00. The ratio is exact and independent of any clock; the absolute rate is not
distinguishable from 60 Hz. CERTAIN (ratio); POSSIBLE (absolute rate).

The frame counts close arithmetically over the same session — 605 frames of 0x03 against 605 of
0x04 — which is what establishes that the longer 0x04 forms **replace** the short one rather than
being sent in addition to it.

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
| **[0x04](docs/msg_0x04.md)** | B→L | normal | body state and focus command, per frame |
| **[0x05](docs/msg_0x05.md)** | L→B | normal | lens status + optical table transfer |
| **[0x06](docs/msg_0x06.md)** | L→B | normal | focus range, focus position, event channel |
| **[0x07](docs/msg_0x07.md)** | both | init | identity and lens ID |
| **[0x08](docs/msg_0x08.md)** | both | init | the large descriptor |
| **[0x09](docs/msg_0x09.md)** | both | init | "start up info"; adapter sub-type |
| **[0x0A](docs/msg_0x0A.md)** | both | init | init echo |
| **[0x0B](docs/msg_0x0B.md)** | both | init | small init exchange |
| **[0x0C](docs/msg_0x0C.md)** | both | init | baud-rate negotiation |
| **[0x0D](docs/msg_0x0D.md)** | both | init | small init exchange |
| **[0x10](docs/msg_0x10.md)** | both | init | lens-ready handshake, long reply delay |
| **[0x16](docs/msg_0x16.md)** | both | init | shutdown request |
| **[0x19](docs/msg_0x19.md)** | ? | ? | unknown |
| **[0x1B](docs/msg_0x1B.md)** | both | init | aperture command |
| **[0x1D](docs/msg_0x1D.md)** | B→L | ? | probably a frame length, not an ID |
| **[0x20](docs/msg_0x20.md)** | B→L | ? | probably a frame length, not an ID |
| **[0x28](docs/msg_0x28.md)** | L→B | init | third carrier for the correction rows |
| **[0x2B](docs/msg_0x2B.md)** | both | init | adapter re-identification (adapters only) |
| **[0x32](docs/msg_0x32.md)** | ? | ? | implemented but not advertised |
| **[0x34](docs/msg_0x34.md)** | ? | ? | unknown; holds the 850 pair |
| **[0x35](docs/msg_0x35.md)** | L→B | init | second carrier for the correction rows |
| **[0x3A](docs/msg_0x3A.md)** | L→B | `0x03` | size of the 0x3B file |
| **[0x3B](docs/msg_0x3B.md)** | L→B | `0x03` | paged optical file (pupil/vignetting table) |
| **[0x3D](docs/msg_0x3D.md)** | both | init | the lens's 0x3A/0x3B file bitmap and optional-field bits; requested by later bodies |
| **[0x3F](docs/msg_0x3F.md)** | both | init | lens name string; requested by later bodies |
| **[0x4C](docs/msg_0x4C.md)** | both | init | optional; version check against the body |
| **[0x4E](docs/msg_0x4E.md)** | both | init | optional; one diverting flag |
| **[0x5A](docs/msg_0x5A.md)** | both | init | optional; request carries the enabled-ID bitmap |

Messages **0x05, 0x28 and 0x35 all carry the same 6-byte optical rows**, in three different
layouts. They have their own reference: [the 6-byte optical rows](docs/optical_data.md).

Those same three messages, plus the **0x1B** command channel, all carry the **aperture value** —
an `n × 256/3` distance index shared by every device on the bus, and emphatically *not* the lens's
own encoder count. It has its own reference too:
[the aperture value](docs/aperture_value.md).

Messages **0x04 and 0x06 together form the focus loop**: the body packs focus instructions — move,
scan, stop, drive — as tagged records into 0x04, and the lens reports its motion and signals
completion through 0x06. The instructions, the motion each one requires, and how the lens answers
are described in [how auto focus works](docs/autofocus.md).

During init the body sorts every device into a **lens type**, native or one of three adapter
classes, from [message 0x07](docs/msg_0x07.md) `pl[0]` and the adapter sub-type. The type changes
how the body re-checks the device, which optional fields it requests, and which focus positions its
autofocus works from: [lens types](docs/lens_types.md).

The **lens ID** in 0x07 `pl[9..10]` never makes a body refuse a lens, but its range decides how far
the body trusts the rest of the init data. A third-party ID gets much of 0x07, 0x08, 0x09 and 0x3D
overwritten with fixed values: [lens ID](docs/lens_id.md).

After init the body decides whether autofocus may use the **full frame, the centre only, or
neither**. A native with no entry in the body's own lens library gets the full range only if it
declares a 0x3B file in [0x3D](docs/msg_0x3D.md) `pl[0]`: [AF range](docs/af_range.md).
