# Message 0x06 — focus range, focus position and drive status

**Summary.** Focus travel limits, the focus position (two fields), and the lens's
focus-drive status. The lens's replies to focus instructions follow it in the same frame. Frame 2 of the four-frame 60 Hz loop.

**Direction:** L→B only.

**Class:** normal (`0x01`).

**Payload:** 39 bytes normally (a 48-byte frame). Two separate things change the frame length:

- **Which fields the message carries is set once per session by the body**, through `pl[8..9]` of
  its [message 0x0A](msg_0x0A.md) request, one bit per field. Every body observed asks for the six
  fields that make up `pl[0..38]`. See [Fields](#fields-and-which-the-body-asks-for).
- **Further messages can follow 0x06 in the same frame**: the lens's replies to finished
  instructions and to queries. They are separate messages, not part of 0x06. See
  [Messages that follow 0x06](#messages-that-follow-0x06-in-the-same-frame).

Only the 39-byte payload has ever been observed on the wire.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Reference payloads

```
Sony SELP1650   81 00 00 40 07 10 10 f4 3e c4 42 00 00 16 00 14 47 47 eb 08 4e 43 00 00 27 00 fe 01 00 ...
TECHART LM-EA9  00 00 00 10 00 30 10 30 10 00 16 00 00 20 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00 ...
```

## Field map

**Third-party** below means a lens whose [lens ID](lens_id.md) is in the third-party range. The A7 IV
masks those bytes in every received 0x06 before anything reads them
([lens ID](lens_id.md#0x06-every-frame)), so for such a lens they have no effect on that body.

| Field | Meaning | Confidence |
| --- | --- | --- |
| `pl[0]` | **Status flags**, one per bit — see [`pl[0]` bit by bit](#pl0-bit-by-bit). Bit 7 = at rest; bit 4 / bit 3 = at the lower / upper end of travel; bit 6 an adapter position hold. | see the bit table |
| **`pl[1]`** | **Direction of travel**: `0x02` toward higher position values, `0x04` toward lower, `0x00` not moving. Same in Yongnuo and TTArtisan firmware. **Bit 0:** the A7 IV waits for it after sending [message 0x29](#bit-7-and-messages-0x29--0x2a), as it waits for `pl[0]` bit 7 after 0x2A. **Third-party:** only bits 1–2 are kept, so bit 0 has no effect. | Direction **CERTAIN** in two vendors' firmware, PROBABLE as protocol; bit 0 use **CERTAIN on the A7 IV firmware**, meaning UNKNOWN |
| `pl[2..3]` | **The position the body works from**, in the units of the travel limits `pl[7..8]`/`pl[9..10]`. **Body:** its current focus position, refreshed every frame; 13 unchanged frames end a body-run Drive; on devices without the [in-frame trace](#pl3238-the-in-frame-position-trace), the AF evaluation's per-frame position. **What to send:** [below](#pl23-what-to-send). | Units **CERTAIN**; body use **CERTAIN on the a7S III firmware** |
| `pl[4]` | `00` on primes (SEL5518Z, Yongnuo) and adapters; `d0` / `a1` SEL55210 @55 / @210 mm, `05` SEL2870 @70 mm, `07` SELP1650. Equal to `pl[11]` on every lens but the SELP1650 (`22`). **Third-party: zeroed — no effect.** | **UNKNOWN** |
| `pl[5]` | `10` on primes; `11` / `4c` SEL55210 @55 / @210 mm, `4f` SEL2870, `10` SELP1650; `00` on the Viltrox. Equal to `pl[12] + 0x10` on every Sony lens — an addition, not a bit set (`3c` → `4c`). Read as u16, `pl[4..5] = pl[11..12] + 0x1000` covers both rules except the SELP1650's `pl[4]`; no carry into `pl[5]` has been observed, so the u16 reading is untested. **Body:** reads `pl[4..5]` as **one u16**. In one body mode a change of more than 256 raises its "zoom moving" flag, with the direction; in the other mode that flag follows 0x05's focal length instead. Separately, it marks `pl[4..5]` stable after 16 unchanged frames. This supports the u16 reading and a zoom position. **Third-party: zeroed — no effect**, so the zoom-moving flag never comes from 0x06. | **UNKNOWN**; u16 read **CERTAIN on the a7S III firmware** |
| `pl[6]` | `0x10` on every Sony native and every adapter observed. The SELP1650 adds a 2-bit counter in bits 5–6 that steps every frame (`10 30 50 70`). In the NEX-7 captures the Voigtländer 15 mm sends `0x10` only while its ring turns and `0x00` at rest, and the Loxia 21 sends `0x00`. **Body:** bit 4 = **this frame's position is valid**. For devices without the in-frame trace, a frame with it clear gets its neighbours' average position, and an adapter's in-frame interpolation needs it set three frames running. Not consulted on the trace path. **Third-party:** only bit 4 is kept; the other bits, including the SELP1650's counter, have no effect. | **UNKNOWN**; body use **CERTAIN on the a7S III firmware** |
| **`pl[7..8]`** | **u16 LE lower focus travel limit** **Body:** treated as 0 for adapters. | **CERTAIN** on the LM-EA9; PROBABLE generally |
| **`pl[9..10]`** | **u16 LE upper focus travel limit** | as above |
| `pl[11..12]` | u16 LE. 0 on primes; on zooms it grows with focal length — SEL55210 464 @55 mm, 15521 @210 mm; SEL2870 16133 @70 mm (tele end); SELP1650 34. **Body:** interpolated onto each AF sample record together with `pl[2..3]`. | **UNKNOWN**; zoom position **POSSIBLE** |
| `pl[13..14]` | u16 LE [defocus scale](autofocus.md#26-the-defocus-unit): Move mode 6 multiplies its operand by it. That it measures position counts per defocus unit at the current aperture is inferred from Yongnuo's per-aperture table. | Use by mode 6 **CERTAIN**; what it measures **PROBABLE** |
| `pl[15..19]` | SELP1650 `14 47 47 eb 08`. **Third-party: `pl[16..19]` zeroed — no effect.** `pl[15]` is kept. | **UNKNOWN** |
| `pl[20..21]` | **Focus position NOW** — the current, un-forecast position. Cheaper EF-mount adapters report nothing here; native E-mount lenses do. **Body:** the final point of the in-frame trace. On the trace path these points are the AF evaluation's positions. | **CERTAIN**; body use **CERTAIN on the a7S III firmware** |
| `pl[22]` | `00` on the SELP1650. **Third-party: zeroed — no effect.** | **UNKNOWN** |
| `pl[23]` | Bit 6 (`0x40`) flips once per body cycle: on the frame where [message 0x03](msg_0x03.md) `pl[11]` returns to 0, or every second frame when the cycle length `pl[10]` is 1. Other bits zero. | Flip rule **PROBABLE**; meaning **UNKNOWN** |
| **`pl[25]`** | **Focus-drive status.** On the LM-EA9, stepped by the command state machines: `0x10` command accepted → `0x20` first stage done → `0x30` both stages done → `0x00` cleared when the machine goes idle. Zero in every idle observation, which is consistent — none contains a focus drive. **Body:** reads bits 4–6 only and takes phase 3 (`0x30`) as "Scan finished" ([autofocus](autofocus.md#how-the-body-reads-pl25)). **Third-party:** `& 0x73`, bits 3 and 7 cleared; the phase bits are untouched. | **CERTAIN** on the LM-EA9; meaning on other devices UNKNOWN; body reading **CERTAIN on the a7S III firmware** |
| `pl[26..27]` | u16 LE. Lens- and focal-length-dependent; no observation shows it changing with focus. See below. **Third-party: zeroed — no effect.** | Meaning **UNKNOWN** |
| `pl[28..31]` | **Third-party: zeroed — no effect.** | **UNKNOWN** |
| `pl[32..38]` | **The position trace within the previous frame.** Yongnuo latches the position at frame sync (`L0`) and samples it again at every eighth of the frame period (`L1`..`L7`); the next sync gives `L8`. `pl[32+k] = L(k+1) − L(k+2)` — **older minus newer**, one eighth of a frame each, newest last. Varies frame to frame on the SELP1650 (`66 7f 7f 7f 7f 7f 7b` → zeros); zero on all adapters. **Body:** rebuilds eight in-frame points from it, ending at `pl[20..21]`, and uses them as the AF evaluation's positions when 0x08 `pl[9]` low nibble is 4. See [the trace](#pl3238-the-in-frame-position-trace). | **CERTAIN** on Yongnuo; PROBABLE that Sony's identical-shaped field is the same thing; body reading **CERTAIN on the a7S III firmware** |


### `pl[0]` bit by bit

Body behaviour is from the A7 IV firmware, where every reader of `pl[0]` sits in the per-frame AF
update. **Third-party** = a [third-party lens ID](lens_id.md#0x06-every-frame).

| Bit | Meaning | What the A7 IV does with it | Third-party | Confidence |
| --- | --- | --- | --- | --- |
| 7 (`0x80`) | **At rest**: the focus group has stopped. Clear while it moves | Ends the wait after the body sends [message 0x2A](#bit-7-and-messages-0x29--0x2a) | **Cleared — no effect** | Meaning **PROBABLE**; body use **CERTAIN on the A7 IV firmware** |
| 6 (`0x40`) | UNKNOWN | Adapters only (types 1 / 2 / 5), with 0x08 `pl[9]` low nibble 2: bit 6 set and bit 4 clear freezes the reported position at the previous frame's value. Also copied into a per-frame record | kept | Body use **CERTAIN on the A7 IV firmware** |
| 5 (`0x20`) | UNKNOWN | Not read | kept | **CERTAIN on the A7 IV firmware** |
| 4 (`0x10`) | **At the lower end of travel**, near `pl[7..8]` | ORed with the body's own "position ≤ lower limit" test; passed to the AF state | kept | **CERTAIN on the A7 IV firmware**; Yongnuo sets it the same way |
| 3 (`0x08`) | **At the upper end of travel**, near `pl[9..10]` | ORed with the body's own "position ≥ upper limit" test; passed to the AF state | kept | as bit 4 |
| 2 (`0x04`) | UNKNOWN | Not read | kept | **CERTAIN on the A7 IV firmware** |
| 1–0 (`0x03`) | UNKNOWN. Not the adapter sub-type; that is 0x05 `pl[59]` ([message 0x2B](msg_0x2B.md)) | Not read | kept | **CERTAIN on the A7 IV firmware** |

An earlier reading, from the a7S III firmware, had the body take bit 6 as the lower end and bit 5
as the upper end. The A7 IV reads bits 4 and 3, the bits Yongnuo writes, and does not read bit 5.
The a7S III reading is withdrawn.

#### Bit 7 and messages 0x29 / 0x2A

- When [message 0x05](msg_0x05.md) `pl[62]` bit 4 goes from 0 to 1, the body sends the lens
  **message 0x2A**, a bare ID with no reply.
- From then on it holds part of its AF processing for at least five frames. The hold ends when 0x06
  `pl[0]` bit 7 is set.
- `pl[62]` bit 5 works the same way, with **message 0x29**, and the hold ends on 0x06 `pl[1]` bit 0.
- A lens that never sets the bit does not hang the body. The wait gives up after about 250 frames.

So bit 7 is the lens saying it has finished and stopped. For a third-party ID none of this runs: the
body masks 0x05 `pl[62]` down to bits 0–1 and clears 0x06 `pl[0]` bit 7. What 0x29 and 0x2A ask the
lens to do is UNKNOWN. CERTAIN on the A7 IV firmware.

---

## Fields, and which the body asks for

The body splits 0x06 into numbered fields with a fixed table, and its
[message 0x0A](msg_0x0A.md) request selects which of them the lens sends: `pl[8]` bit `n` asks for
field `0x40 + n`, `pl[9]` bit `n` for field `0x48 + n`. Fields that are not asked for are left out,
and the fields after them move up. CERTAIN on the A7 IV firmware.

| Field | Size | `pl` when every earlier field is present | A7 IV asks for it |
| --- | --- | --- | --- |
| `0x40` | 2 | `pl[0..1]` | always |
| `0x41` | 11 | `pl[2..12]` | always |
| `0x42` | 9 | `pl[13..21]` | always |
| `0x43` | 4 | `pl[22..25]` | always |
| `0x44` | 6 | `pl[26..31]` | always |
| `0x45` | 7 | `pl[32..38]` | always |
| `0x46` | 45 | `pl[39..83]` | **never** |
| `0x47` | 4 | next | native lens, and [0x08](msg_0x08.md) `pl[12]` bit 5 set |
| `0x48` | 8 | next | adapter that declares [0x5A](msg_0x5A.md) in [0x01](msg_0x01.md) |
| `0x49` | 8 | next | a [0x3D](msg_0x3D.md) feature bit |
| `0x4A` | 24 | next | a 0x3D feature bit |

`0x48`–`0x4A` are never asked of a [third-party ID](lens_id.md#what-third-party-changes). What
`0x47`–`0x4A` carry is UNKNOWN; the body stores `0x47` as two u16 and `0x48`/`0x49` as four u16 each.

So the 48-byte frame is `pl[8] = 3f`, fields `0x40`–`0x45`, which is what every body observed sends.
The A7 IV can also ask for a 43-byte payload, a 52-byte frame (`pl[8] = bf`, adding field `0x47`),
or longer ones through `pl[9]`.

### The 93-byte form

The 93-byte form is field `0x46` present: `pl[8] = 7f`. Yongnuo implements it (three models, two
separate code bases) as 45 bytes of zeros between `pl[38]` and the messages that follow:

```
48:  F0 len cls seq 06 | pl[0..38] |                      | following messages | ck ck 55
93:  F0 len cls seq 06 | pl[0..38] | pl[39..83] = 45 × 00 | following messages | ck ck 55
```

Yongnuo tests `pl[8] == 7f` exactly. It does not look at the other bits, so a body asking for
field `0x47` would not get it from Yongnuo.

| Aspect | Confidence |
| --- | --- |
| Field `0x46` is 45 bytes | **CERTAIN on the A7 IV firmware** |
| The A7 IV never asks for it | **CERTAIN on the A7 IV firmware** |
| On Yongnuo: `pl[0..38]` the same in both forms, `pl[39..83]` all zero | **CERTAIN** |
| What `pl[39..83]` is meant to carry | **UNKNOWN**; no body has been seen sending `7f` |

## Messages that follow 0x06 in the same frame

**A frame can carry several messages back to back.** The body reads the message ID, looks up that
message's length in its own per-ID table, and steps to the next message, until the frame's payload
is used up. 0x06 is one message; whatever follows it in the frame is another message with its own
ID. CERTAIN on the A7 IV firmware.

```
F0 | len | cls seq | 06 pl[0..38] | [ID b ...] [ID b ...] ... | ck_lo ck_hi | 55
                     \__ 0x06 __/  \__ further messages, 0 or more __/
```

The lens uses this to answer the body's instructions and queries from [message 0x04](msg_0x04.md).
Each answer carries **the ID of the instruction it answers**. The body's 0x04 records are messages
from the same ID space, so the "operation code" seen here is simply that message ID:

| Message | Length, ID included | Content |
| --- | --- | --- |
| `0x1C` Stop, `0x1D` Move, `0x1E`, `0x1F` Scan | 2 | ID + one byte |
| `0x22`, `0x2E` | 3 | ID + u16, the answer to a query |
| `0x3C` Drive | 4 | ID + three bytes |

The lengths are fixed by the body's table, not by the lens. A message whose length does not match
makes the body reject the rest of the frame (`BL_MOUNT_ERR_DATASIZE_OVER`, `ANALYSE_NG_ID`). When an
answer's ID matches the instruction the body is waiting on, the body runs that instruction's
completion handler. When each answer is sent, and what its byte means, is in
[autofocus §4.2](autofocus.md#42-replies-in-the-0x06-frame).

The same frame can also carry the asynchronous messages `0x41` and up, which have their own lengths.

| Implementation | What follows 0x06 |
| --- | --- |
| Yongnuo | One reply from an event queue (2 bytes, or 4 for `0x3C`), plus query answers. Its queue uses `0x1F` internally to mean "no event"; that value is never sent. |
| TECHART LM-EA9 | One reply byte pair at a fixed offset, second byte always `0`: `0x1C`, `0x1D` or `0x1F`, naming the finished command |

Both put the first following message at frame offset `0x2D`, right after `pl[38]`, by different
routes: Yongnuo computes it and the LM-EA9 hardcodes it.

## The status byte `pl[25]`

Independently of the replies that follow 0x06, both implementations write a status byte at frame offset `0x1F` =
`pl[25]`. The LM-EA9 steps it `0x10` → `0x20` → `0x30` → `0x00` through its command state machines;
Yongnuo pulls it from a **second** event queue and special-cases `0x1f` there too, converting it to
`0`.

Two independent implementations writing the same offset from a status/event source is good evidence
`pl[25]` is a real protocol field, not a vendor scratch byte.

**Confirmed from the body side.** The body queues every non-zero phase
`(pl[25] >> 4) & 7`, so a value shown for one frame is not lost. Phase 1 marks a Scan accepted, and
**phase 3 fires the AF layer's Scan-completion callback**; the `1F` reply message does not. Details are
in [autofocus — how the body reads `pl[25]`](autofocus.md#how-the-body-reads-pl25). **CERTAIN on the a7S III firmware.**

**Why nothing on the wire shows this:** every 0x06 frame observed is 48 bytes with `pl[25] = 0` and nothing after it,
because the observations are init plus idle — no focus drive ever happens in them. **A 0x06 with a
non-zero `pl[25]`, or a reply message after it, is a specific, cheap thing to look for in any future AF
observation**, and would confirm this whole section on the wire.

---

## `pl[2..3]`: what to send

The body takes `pl[2..3]` as its current position every frame, and treats it **not changing for 13
frames** as the end of a Drive it is running. So the field has to be **steady when focus is at rest**
and must **change while focus moves**. Jitter at rest keeps that fallback from firing. CERTAIN on the
a7S III firmware for the readers; the requirement follows from them.

What devices send:

| Device | At rest | While moving |
| --- | --- | --- |
| Sony natives | steady, while `pl[20..21]` jitters ±1 | the end of the move, from the first moving frame |
| Yongnuo | equal to `pl[20..21]` | `pl[20..21]` + a one-frame forecast that stops short of the target; one mode sends it equal to `pl[20..21]` |

Both run ahead of `pl[20..21]` while moving and never past the target. Whether the body needs that is
not established: its AF planner was not followed to where it reads the current position.

## `pl[32..38]`: the in-frame position trace

The seven bytes describe how focus moved **during the last frame**, at one-eighth-frame resolution:

```
frame sync n-1                                             frame sync n
     |------|------|------|------|------|------|------|------|
          s[0]   s[1]   s[2]   s[3]   s[4]   s[5]   s[6]   s[7] = pl[20..21]

pl[32+k] = s[k] - s[k+1]        k = 0..6, signed, older minus newer
```

- **Body side.**
  - The body inverts exactly that: `s[7] = pl[20..21]`, `s[k] = s[k+1] + pl[32+k]`.
  - The result is stored as eight positions belonging to one frame.
  - When the body has no trace to use, it fills the same eight slots by linear interpolation
    between two frames' positions. That only makes sense if the slots are sub-frame points.
  - When the sensor runs at 2× or 4× the lens frame rate, the body splits the eight points 4 + 4
    or 2 + 2 + 2 + 2 across the sensor frames.
  - The frame the eight points are filed under is two frames back, or one frame back when
    [0x08 `pl[85]`](msg_0x08.md#pl85) bit 7 is set. A trace that is a frame earlier or later
    than declared shifts every AF position by a frame of travel.
- **On the wire.** If the bytes were a history across frames, each frame's array would be the
  previous frame's array shifted by one. On the NEX-7 / SEL2870 capture that holds for 2 of 34
  frame pairs, so it is not a history across frames.

| Statement | Confidence |
| --- | --- |
| The body reconstructs eight in-frame points, older minus newer, ending at `pl[20..21]` | **CERTAIN on the a7S III firmware** |
| Not a history across frames | **PROBABLE** (body code and one capture agree) |

## What the body reads from the position fields

Read from body firmware; not confirmed on the wire.

**How the body takes the message apart.**
- It splits 0x06 into the fields listed [above](#fields-and-which-the-body-asks-for). The position
  fields are in `0x40` = `pl[0..1]`, `0x41` = `pl[2..12]`, `0x42` = `pl[13..21]`, `0x45` = `pl[32..38]`.
- Once per frame it stores a snapshot of them, keeping 32 frames of history.
- How the position fields are used then depends on the [lens type](lens_types.md):

| Path | Condition | Examples |
| --- | --- | --- |
| **Native, trace** | 0x07 `pl[0] = 01`, and 0x08 `pl[9] & 0x0F = 4` (or the 0x3D flag) | Yongnuo 50 DF |
| Native, no trace | 0x07 `pl[0] = 01`, any other 0x08 `pl[9]` | the NEX-7 capture's native (`pl[9] = 01`) |
| Adapter | 0x07 `pl[0] = 02` (types 1 / 2 / 5) | stock LM-EA9, Viltrox |

### Native device with the in-frame trace

Per frame, the body does three things with the position fields:

1. **Eight in-frame points.**
   - `s[7] = pl[20..21]`, `s[k] = s[k+1] + pl[32+k]`.
   - The eight points go into a time-stamped position ring, under the frame set by
     [0x08 `pl[85]`](msg_0x08.md#pl85) bit 7 (see [the trace](#pl3238-the-in-frame-position-trace)).
2. **The AF evaluation reads them.**
   - It asks for the focus positions belonging to a run of frames and gets these points: up to
     128 of them, 8 per frame.
   - A device without the trace gives it at most 16, one per frame.
   - So on this path, the in-frame trace is the AF evaluation's focus-position axis.
3. **Each AF sample record gets two interpolated positions.**
   - The body knows when each AF sample was exposed, as a fraction `w` of a frame.
   - It stores `(1 − w)·previous + w·current`, once from `pl[2..3]` and once from `pl[20..21]`.
   - Apart from a debug dump, no reader of these two copies was found.

| Field | Native with trace: what the body does with it |
| --- | --- |
| `pl[20..21]` + `pl[32..38]` | The eight in-frame points: the AF evaluation's positions |
| `pl[2..3]` | The current position and the 13-frame end-of-Drive fallback ([autofocus](autofocus.md#how-the-body-reads-pl25)); not the AF position |
| `pl[7..8]`, `pl[9..10]` | Travel limits, both used |
| `pl[0]` bit 4 / bit 3 | At the lower / upper end, ORed with the body's own check |
| `pl[0]` bit 7, `pl[1]` bit 0 | End the waits after messages 0x2A / 0x29 |
| `pl[0]` bits 0–2 and 5, `pl[6]` | Not consulted |
| `pl[4..5]` | Zoom-movement detection, in one body mode |

**What such a device has to get right.** `pl[20..21]` and `pl[32..38]` must describe the frame's
real motion: one eighth of the frame period per step, ending at the frame sync. That is the time
base the AF evaluation assumes. `pl[2..3]` should agree with them, because the body's current
position comes from it ([what to send](#pl23-what-to-send)).

The eight points are produced only while the body's lens link is in its normal state and the
async position message 0x41 is not in use. With 0x41, positions arrive through that message
instead, up to 256 samples.

### Differences on the other paths

| Field | Native, no trace | Adapter |
| --- | --- | --- |
| AF evaluation positions | One per frame: that frame's `pl[2..3]`. If the 0x08 `pl[9]` low nibble is 2, `pl[20..21]` is used instead | One per frame: that frame's `pl[2..3]` |
| `pl[6]` bit 4 | A frame with the bit clear gets its neighbours' average | Same. Also, the eight-point ring is interpolated from `pl[2..3]` only if the bit was set in the last three frames |
| Eight-point ring | Linear interpolation of `pl[2..3]` between frames | As before, with the `pl[6]` condition |
| `pl[32..38]` | Unused | Unused |
| `pl[7..8]` | Used | **Treated as 0** |
| `pl[0]` bit 6 | Unused | For 0x08 `pl[9]` low nibble 2: bit 6 with bit 4 clear freezes the reported position |
| `pl[2..3]` in a Scan | — | With 0x08 `pl[9]` low nibble 1, the body may defer a Scan: it first sends a Move, and once the lens has stopped it recomputes the Scan's start from `pl[2..3]` |

| Statement | Confidence |
| --- | --- |
| The field split and the readers listed above | **CERTAIN on the a7S III firmware** |
| The three paths and their conditions | **CERTAIN on the a7S III firmware** (see [lens types](lens_types.md)) |
| The AF evaluation's sample count (16 / 128 / 256) | **PROBABLE** |
| The two per-sample interpolated copies have no functional reader | **POSSIBLE**: the search covered the CPU-side code only |

---

## The travel clamps are real limits, not just a report

`pl[7..8]` and `pl[9..10]` are not advisory: at least one implementation clamps every commanded
focus target against them before driving the motor. The fields are byte-packed at odd offsets, which
is why implementations tend to read them a byte at a time.

Observed ranges:

| Device | Lower | Upper | Span |
| --- | --- | --- | --- |
| TECHART LM-EA9 1.6.0 | 4144 | 5728 | 1584 |
| TECHART LM-EA9 1.7.0 / 1.8.0 | 4144 | **5632** | 1488 |
| Sony SEL5518Z | 16178 | 21987 | **5809** |
| Sony SELP1650 | 16116 | 17092 | 976 |

The native SEL5518Z exposes roughly **4× finer focus resolution over its travel** than the LM-EA9.
Recorded as a fact; not linked to any symptom.

Note the constraint this implies: **a lens's actual travel limits should agree with the range it
advertises in message 0x08.**

---

## `pl[26..27]` — UNKNOWN

| Device | `pl[26..27]` | Within the capture |
| --- | --- | --- |
| Sony SELP1650 (A6000) | 510 | constant |
| Sony SEL55210 @55 mm / @210 mm (A6000) | 793 / 1684 | constant |
| Sony SEL5518Z (NEX-7) | 354 | constant |
| Sony SEL2870 @70 mm (NEX-7) | 851 | constant |
| Voigtländer 15 mm, Zeiss Loxia 21 mm (manual) | 0 | constant, including a 0.3 m → ∞ ring sweep |
| Viltrox + EF 50 mm / EF-S 24 mm | 691 / 968 | constant |
| Yongnuo 35 DA, 50 DA / 50 DF v04 | 0 / 254 | hardcoded, never computed |

It differs between lenses and with focal length (SEL55210). None of the Sony captures contains a
focus move, so whether it varies with focus on an AF lens is unobserved. Yongnuo sends a constant and
autofocuses correctly, so the field is **not required for autofocus**.

## Manufacturer notes

- **Sony** — natives populate the near-limit flags in `pl[0]`, the velocity trace, and `pl[26..27]`. Travel spans are wide (the SEL5518Z spans 5809 counts).
- **Yongnuo** — computes a one-frame forecast for `pl[2..3]` and the near-limit flags, sends its replies from an event
  queue (using `0x1F` internally for "no event"), and writes a constant to `pl[26..27]`.
- **TECHART** — the LM-EA9 sends `pl[0] = 0`, a constant `pl[26..27]`, no forecast, and a narrow
  1488–1584 count travel span. It sends one reply (`1C`, `1D` or `1F`) with its byte always 0.
- **Viltrox** — the EF adapters report nothing in `pl[20..21]` and send `pl[0] = 0`.

## Open questions

- Who reads the per-AF-sample interpolations of `pl[2..3]` and `pl[20..21]`, if
  anything beyond a debug dump.
- Whether the AF planner places commands relative to `pl[2..3]`, which would make running ahead
  of `pl[20..21]` necessary rather than just harmless.

- `pl[0]` bits 0–2, 5 and 6.
- What messages 0x29 and 0x2A ask the lens to do.
- What the byte in each 2-byte reply means, beyond `0x1C`'s `01`.
- `pl[25]`'s meaning on devices other than the LM-EA9.
- `pl[4]`, `pl[5]`, `pl[6]` and `pl[11..12]`, and whether `pl[4..5]` is one u16.
- What `pl[26..27]` is, and whether it varies with focus on an AF lens.
- Which bodies send message 0x0A `pl[8] = 7f`, and what `pl[39..83]` of the 93-byte form carries on
  a lens that fills it.
- What fields `0x47`–`0x4A` carry.
