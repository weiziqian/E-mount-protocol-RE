# Message 0x28 — lens state, requested at the exposure

**Summary.** The lens's state for the frame being exposed: subject distance, focal length, an
optional [aperture value](aperture_value.md), and one [optical row pair](optical_data.md) back to
back. It is a copy of the lens's current [message 0x05](msg_0x05.md) status. The body requests it
during a still capture, and **the focal length it returns is the one written to the photograph's
EXIF**. [Message 0x35](msg_0x35.md) is the same content in another layout.

**Direction:** body request, lens reply. **Class:** init (`0x02`).

**Length:** in the A7 IV length table, the request is 2 bytes counting the ID and the reply is 36.
The reply frame is 44 bytes, with 35 payload bytes `pl[0..34]`. The request byte `pl[0]` is the
same as [0x35's](msg_0x35.md#request).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

Sources: the A7 IV body firmware (parser), the Yongnuo 50 DF firmware (builder, `FUN_08014f68`), and
the a9 II on the wire.

---

## Reply payload

| `pl` | Yongnuo | A7 IV | Confidence |
| --- | --- | --- | --- |
| **`0..1`** | **Subject distance**: 0x05 `pl[20..21]`, the [distance code](autofocus.md#25-the-distance-code) | Stored as the exposure's subject distance, as in [0x35](msg_0x35.md) | **CERTAIN** |
| `2` | `00` | Not read | **CERTAIN** |
| `3` | 0x05 `pl[23]`, the coarse distance | Not read | **CERTAIN** |
| **`4..5`**, **`6..7`** | **Focal length** pair, mm × 10, as in 0x05 `pl[24..27]` | Stored as the focal length | **CERTAIN** |
| `8` | `00` | The same three flags as 0x35 `pl[8]`. For a third-party lens ID the body forces bit 0 on | Reads **CERTAIN**; meaning **UNKNOWN** |
| `9..10` | The aperture, only when the body asks ([below](#pl910-must-follow-the-bodys-request-and-must-not-be-volunteered)) | Filed in the slot where it files 0x05 `pl[30..31]` | **CERTAIN** |
| **`11..16`** | 0x05 slot A, `pl[32..37]` | Decoded as slot A | **CERTAIN** |
| **`17..22`** | 0x05 slot B, `pl[38..43]`, from the parity-0 half of the row pair | Decoded as slot B only if it is a type-0 row; see [0x35](msg_0x35.md#slot-b) | **CERTAIN** |
| `23..31` | 0x05 `pl[44..52]` | Filed as 0x05 `pl[44..52]`; zeroed for a third-party lens ID | **CERTAIN** |
| `32..33` | 0x05 `pl[53..54]` | The value 0x35 carries at `pl[29..30]`, under the same condition | Reads **CERTAIN**; meaning **UNKNOWN** |
| `34` | 0x05 `pl[55]` | Zeroed for a third-party lens ID | **CERTAIN** |

So `pl[9..31]` lines up with 0x05 `pl[30..52]`: the A7 IV files them in the same slots as the status
message, and Yongnuo fills `pl[11..34]` from the same snapshot. Unlike 0x35, this form has no room
for the long-form row extensions.

A device whose fields here disagree with its message 0x05 is inconsistent. The focal length is the
case with a visible cost ([below](#the-focal-length-in-this-message-is-the-one-written-to-exif)).

## 0x28, 0x35 or 0x50 — which one the body asks for

CERTAIN on the A7 IV firmware. The capture path builds one request:

1. **0x50**, if the lens declares it in [0x01](msg_0x01.md). It is a newer, field-structured form,
   and is never sent to a third-party lens ID ([lens ID](lens_id.md)).
2. Otherwise **0x35**, if the lens is native or type 5 ([lens types](lens_types.md)) **and** any
   of these holds:
   - [0x08](msg_0x08.md) `pl[13] & 0x60 == 0x60`;
   - [0x3D](msg_0x3D.md) `pl[33]` bit 2;
   - 0x3D `pl[33]` bit 3.
3. Otherwise **0x28**.

The reply is parsed by a per-ID routine and filed in the same place either way. 0x08 `pl[13]` bit 5
is set on every full-frame device, so in practice bit 6 decides between 0x35 and 0x28 for a
full-frame lens.

## `pl[9..10]` must follow the body's request, and must not be volunteered

Whether this field is populated is **the body's decision, not the lens's.** Bit 7 of `pl[1]` in the
body's [message 0x08](msg_0x08.md#the-request-enables-an-optional-field-in-messages-0x28-and-0x35)
request carries it, inverted:

| `pl[1]` bit 7 of the 0x08 request | what this field must contain |
| --- | --- |
| **clear** | the lens's current [aperture value](aperture_value.md) |
| **set** | **zero** |

The request arrives once per session, sixth in the init handshake, so a lens latches the bit there
and applies it to every 0x28 it sends afterwards.

**Sending the aperture unasked is not harmless.** A Sony a9 II sets the bit — it does not want the
field. An adapter that filled it in regardless made that body **re-meter after every exposure**, and
produced metering errors in continuous shooting. Zeroing the field, as the bit asks, fixed both;
nothing else changed. CERTAIN on that body.

A Sony A6000 clears the bit, so the same lens must populate the field there. Following the bit is
therefore not optional in practice either — it is the only behaviour that is correct on both.

## Example frame

A TECHART LM-EA9 template. The length field is `00 00` and the checksum `00 00`; both are filled in
at send time.

```
F0 00 00 02 00 28 | 00 07 00 FF F4 01 F4 01 01 00 00 00 00 09 00 00 00 22 00 00 00 00 00 16 00 0E 00 56 00 00 16 50 00 00 00 | 00 00 55
```

`pl[0..1]` = `0x0700`, infinity, and `pl[3]` = `FF`. `pl[4..5]` and `pl[6..7]` are both `F4 01` =
500 = 50.0 mm. `pl[9..10]` is `00 00`: that device never populates the aperture, and its
photographs still record the aperture correctly.

## Who requests it

Bit 39 of the [capability bitmap](msg_0x01.md) covers this ID.

| Body | Offers bit 39 | Requests the message |
| --- | --- | --- |
| Sony a9 II | yes | **yes — twice in one session** |
| Sony A7 IV (firmware) | — | yes, at a still capture, unless it picks 0x35 or 0x50 ([above](#0x28-0x35-or-0x50--which-one-the-body-asks-for)) |
| Sony A6000 | yes | no, in 3393 frames |
| Sony NEX-7, A7 | yes | no, in 6572 frames |

On the a9 II both requests arrive **at the shutter press**, inside a burst of init-class requests —
`0x0A 0x0B 0x1B 0x28 0x19 0x19 0x28 0x0B 0x0A` — and not during the init handshake or the idle loop.
The older bodies offer the ID and never ask for it.

## The focal length in this message is the one written to EXIF

A device reporting **40.0 mm** in [message 0x05](msg_0x05.md) `pl[24..25]` and **50.0 mm** here
produced photographs tagged **50 mm** on a Sony a9 II. The same device changed to report 52.0 mm in
both messages produced photographs tagged **52 mm**.

The two messages disagreeing is what makes the first observation decisive: the body took the focal
length from this message. Confidence **CERTAIN** for that body; whether every body does is
**UNKNOWN**.

A device whose focal length differs between messages 0x05, 0x28 and 0x35 will therefore be described
one way in the camera's display and another way in the file it writes.

## What it is for

The body re-reads the lens's state for the frame it is exposing instead of relying on the last
status frame. The A7 IV then compares the aperture it asked for with the one the lens reports, and
drives the iris again if they differ ([0x35](msg_0x35.md#when-the-body-sends-it--certain-on-the-a7-iv-firmware)).
Only the focal length has been traced to an output (EXIF).

A Sony a7CR sends it to a Sony SELP1650 at each frame of a shutter release (six requests `02 36 00` in a burst), inside the loop
pause; to a lens with the class-3 group it sends [0x50](msg_0x50.md) instead. POSSIBLE. Sent the same request mid-session, a SELP1650 at 48.5 mm answered with
`e5 01` (485) in `pl[4..5]`.

## Open questions

- The three flags in `pl[8]`, and the value at `pl[32..33]`.
- Whether a body requests the message more than once per exposure, or outside an exposure sequence.
- Which message supplies the focal length a body *displays*, as opposed to the one it records.
