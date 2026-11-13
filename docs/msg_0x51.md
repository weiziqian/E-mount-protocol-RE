# Message 0x51 — asynchronous iris move

**Summary.** The body's aperture command for lenses that support an asynchronous iris. It is sent
every frame in live view and again at the release. Known only from body firmware. No capture has
it.

**Direction:** B→L, with a lens reply (`L2B_MOVE_IRIS_A`, `L2B_MOVE_IRIS_COMP_A` in the body's
names). **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## What the body does

| Item | Behaviour | Confidence |
| --- | --- | --- |
| Who gets it | Only a lens that passes a lens-capability check for asynchronous iris. Never a third-party lens ID: `0x50`–`0x60` are masked for those ([lens ID](lens_id.md#messages-the-body-will-use)) | Check **CERTAIN on the a7C II firmware**, what it tests UNKNOWN; mask **CERTAIN on the A7 IV firmware** |
| Every frame | The body copies the live aperture of [message 0x03](msg_0x03.md#where-the-body-takes-the-aperture-pair-from) into it: `pl[5..6]` if 0x03 `pl[7]` bit 7 is set, else `pl[3..4]` | **CERTAIN on the a7C II firmware** |
| At the release | The sequencer's Iris step sends the exposure aperture here. A lens without async iris gets [message 0x1B](msg_0x1B.md) instead | **CERTAIN on the a7C II firmware** |
| Payload | Carries an [aperture value](aperture_value.md), u16 LE. Its offset and the other bytes were not recovered | Aperture **CERTAIN on the a7C II firmware**; layout UNKNOWN |

The body sends class-3 frames through a path of its own, one frame in flight at a time, separate
from the init-class request queue.

## Open questions

- The full payload layout, and the lens reply.
- What the capability check reads: a [message 0x01](msg_0x01.md) bit, the lens ID, or a descriptor
  flag.
