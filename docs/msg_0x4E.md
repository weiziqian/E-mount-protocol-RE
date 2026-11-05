# Message 0x4E — optional init flag

**Summary.** A short optional init exchange, requested immediately after
[message 0x07](msg_0x07.md). One reply bit diverts the body out of the normal handshake. Known only
from body firmware.

**Direction:** both. **Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## What the Sony A7 IV firmware does

| Item | Behaviour | Confidence |
| --- | --- | --- |
| When | After 0x07, only if the lens declares it in [message 0x01](msg_0x01.md) (bit 77, `pl[9]` bit 5) | **CERTAIN** on the A7 IV firmware |
| Request | `pl[0] = 01` | **CERTAIN** on the A7 IV firmware |
| Reply `pl[0]` bit 0 | Set → the body leaves the normal handshake for another state | **CERTAIN** on the A7 IV firmware; meaning UNKNOWN |

No captured device declares or sends it.

## Open questions

- What the diverted state is.
