# Lens types — how a body classes the device, and what changes

**Summary.** During init a body sorts every device into a **lens type**: **native** (type 4), or
one of three **adapter** classes (types 1, 5 and 2, from the adapter sub-type). The type decides:
- whether the body keeps re-checking the device's identity during the session;
- which optional fields it asks for in [0x05](msg_0x05.md) and [0x06](msg_0x06.md);
- whether it trusts the lower travel limit;
- which focus positions its autofocus evaluation works from.

The a9 II shows the visible side of the same split: in native mode the focus-area modes, AF during
video and AF in magnified view are available; in adapter mode they are not. See
[0x07 `pl[0]`](msg_0x07.md#pl0-native-mode-vs-adapter-mode).

**Source.** Read from body firmware; nothing here has been observed on the wire. Device values come
from captures and from device firmware. The type numbers below are the body's internal ones and are
used only to refer to the classes.

`pl` is the payload of a message: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

---

## 1. How the type is decided

### Step 1 — device class: [0x07](msg_0x07.md) `pl[0]`

| `pl[0] & 3` | Type | Class |
| --- | --- | --- |
| `1` | **4** | native |
| `2` | 1, refined in step 2 | adapter |
| `0`, `3` | — | `LENS UNKNOWN`: the handshake ends |

Every Sony lens and every Yongnuo lens send `01`. The stock LM-EA9 and the Viltrox EF adapters send
`02`.

### Step 2 — adapter class: the sub-type

For an adapter, the class comes from the 2-bit **adapter sub-type**: read from
[0x09](msg_0x09.md) `pl[0]` during the handshake, re-checked every frame, and confirmed with
[0x2B](msg_0x2B.md). Native devices never go through this. The classes are described in
[0x2B](msg_0x2B.md); their internal type numbers are:

| Sub-type | Type |
| --- | --- |
| `0` | 1 |
| `1` | **5** |
| `2` | 2 |

The stock LM-EA9 and the Viltrox both reply `01 01 …` to 0x09, so both are type 5.

**The per-frame carrier is 0x05 `pl[59] & 3`** (0x05 field `06`, second byte), read while 0x05 is
parsed. Both the A7 IV and the a7S III firmware read it there; an earlier A7 IV reading that pointed
at 0x06 `pl[0]` was a misread of the field lookup. On the wire the two Viltrox adapters send `01`,
matching their 0x09 sub-type, and the Sony natives send `00`.

### Step 3 — native sub-switch: [0x08](msg_0x08.md) `pl[9]` low nibble

A native device (type 4) has the **in-frame position trace**
([0x06 `pl[32..38]`](msg_0x06.md#pl3238-the-in-frame-position-trace)) used as the AF evaluation's
focus positions, 8 per frame, when either of these holds:
- **0x08 `pl[9] & 0x0F` is 4**; or
- a flag in the [0x3D](msg_0x3D.md) reply is set (bit 2 of 0x3D `pl[34]`).

With low nibble 2, the AF evaluation gets one point per frame, the trace's final point
`pl[20..21]`. Any other native gets one point per frame from `pl[2..3]`, as an adapter does. See
[0x08 `pl[9]`](msg_0x08.md#pl9-low-nibble-selects-the-focus-position-source).

### Examples

| Device | 0x07 `pl[0]` | 0x09 `pl[0]` | 0x08 `pl[9]` | Type | AF focus positions |
| --- | --- | --- | --- | --- | --- |
| Sony native, NEX-7 capture | `01` | — | `01` | 4 | `pl[2..3]`, one per frame |
| Yongnuo 50 DF | `01` | — | `14` | 4 | in-frame trace, 8 per frame |
| Stock LM-EA9 | `02` | `01` | `02` | 5 | `pl[2..3]`, one per frame, gated by `pl[6]` bit 4 |
| Viltrox EF adapter | `02` | `01` | — | 5 | as above |

| Statement | Confidence |
| --- | --- |
| Steps 1 and 2 | **CERTAIN on the A7 IV and a7S III firmware** |
| The 0x05 `pl[59]` carrier | **CERTAIN on the A7 IV firmware** (field table and lookup read from code); consistent with the a7S III firmware and with the Viltrox captures |
| Step 3 | **CERTAIN on the a7S III firmware** |

---

## 2. What the body does differently

### The same for both classes

- The init sequence and its end conditions ([README](../README.md)), the per-message length table,
  and capability gating through the [0x01](msg_0x01.md) bitmap.
- The 0x04 focus records (Move, Scan, Stop, Drive, queries), and how the body reads
  [0x06 `pl[25]`](autofocus.md#how-the-body-reads-pl25).
- 0x06 `pl[0]` bits 4 / 3 (at the lower / upper end).
- 0x06 `pl[4..5]` zoom-movement detection.
- [0x08 `pl[85]`](msg_0x08.md#pl85) bit 7, the position-report latency.

### Different

| Aspect | Native (type 4) | Adapter (types 1 / 2 / 5) |
| --- | --- | --- |
| Identity during the session | Fixed after init | Re-checked every frame; a change triggers 0x2B re-identification |
| Init | Lens-dependent setup runs straight away | Runs as type 1 first, and repeats part of the setup once 0x09 has set the sub-type. The live-view gain setup also takes a different branch (meaning UNKNOWN) |
| Lower travel limit, 0x06 `pl[7..8]` | Used | **Treated as 0.** Only the upper limit is honoured in the body's own limit checks |
| Focus positions given to the AF evaluation | In-frame trace, 8 per frame, when the step 3 condition holds; otherwise one per frame | One per frame, `pl[2..3]`. A frame whose 0x06 `pl[6]` bit 4 is clear gets its neighbours' average |
| 0x06 `pl[6]` bit 4 | Not consulted on the trace path | Required (see above) |
| Position hold | — | With 0x08 `pl[9] & 0x0F` = 2: while 0x06 `pl[0]` bit 6 is set and bit 4 clear, the body freezes the reported position |
| 117-byte 0x05 (long form) | Allowed, with 0x08 `pl[13] & 0x60` = `0x60` | Type 5 only, same condition |
| Extra 0x06 field `0x47` | With 0x08 `pl[12]` bit 5 | — |
| Extra 0x06 field `0x48` | — | Type 5 that declares 0x5A |
| Feature table | Fixed native values; 0x08 `pl[85]` bit 2 is never read | Type 5 only: chosen from 0x08 `pl[85]`. **Bit 2 (`0x04`) is needed** for native-like values, otherwise the values are those of an unsupported device and adapter mode does not run. Types 1 and 2 always get the unsupported values. See [0x08 `pl[85]`](msg_0x08.md#pl85) |

Details of the position handling are in
[0x06 — what the body reads](msg_0x06.md#what-the-body-reads-from-the-position-fields).

| Statement | Confidence |
| --- | --- |
| Each row of the table | **CERTAIN on the a7S III firmware** as code; effects on the user **UNKNOWN** except where noted |
| The purpose of the init and live-view differences | **UNKNOWN** |

---

## Open questions

- What the live-view gain setup does differently for adapters.
- The meaning of 0x08 `pl[9]` beyond its low nibble.
