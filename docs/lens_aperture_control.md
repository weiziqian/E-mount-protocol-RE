# Lens control of the aperture

**Summary.** A lens can tell the body what the aperture is, or move the body's aperture setting. It
does this in two fields, and no message of its own carries a command: [message 0x08](msg_0x08.md)
declares, once at init, whether the lens has an aperture ring and what kind it is. Then
[message 0x05](msg_0x05.md) field 3 (`pl[17..19]`) reports, every frame, either an aperture set
on the lens (absolute) or a ±1 step (relative). [Message 0x1B](msg_0x1B.md) runs the other way:
it is the body commanding the lens.

All findings are from the Sony A7 IV firmware 6.02 (CP processor). The body passes what it decodes
to its main processor, whose firmware is not available. So what the body *does* with these values
at the user level is **UNKNOWN** unless stated.

---

## 1. The declaration — message 0x08 `pl[23..26]`

The body keeps the 0x08 reply and reads `pl[23..26]` as one u32 LE:

| Bits | Debug name | Meaning | Confidence |
| --- | --- | --- | --- |
| `pl[23..24]` bits 0–12 | `b13_IRIS_cont_w_close_limit_MOVIE` | A 13-bit value; by its name, a limit on closing the iris in movie mode | Extraction **CERTAIN on the A7 IV firmware**; meaning from the name only |
| `pl[24]` bit 5 (`0x2000` of the u16) | `b_IsIrisRingTypeMecha` | The aperture ring is mechanical | **CERTAIN on the A7 IV firmware** |
| `pl[24]` bit 6 or bit 7 (`0xC000` of the u16, either one) | `b_SupportManualAperture` | The lens supports a manually set aperture | **CERTAIN on the A7 IV firmware** |
| `pl[25..26]` bits 0–11 | `b12_IrisZmFNoShiftMax` | A 12-bit value; by its name, a limit on the f-number shift while zooming | Extraction **CERTAIN on the A7 IV firmware**; use not traced |

Both flags are copied into the body's lens-status record every frame, beside the field-3 values
below.

**Effect on the asynchronous iris.** A lens that declares both "manual aperture" and "mechanical
ring" is not driven through [message 0x51](msg_0x51.md) and its companions 0x52/0x53, unless a
model configuration bit overrides that. This applies only to a lens that declares 0x51–0x53 in
[message 0x01](msg_0x01.md). **CERTAIN on the A7 IV firmware.**

## 2. The report — message 0x05 field 3, `pl[17..19]`

Present when the body's [message 0x0A](msg_0x0A.md) request includes field 3.

| `pl` | Debug name | Meaning | Confidence |
| --- | --- | --- | --- |
| `17..18` | `us_IrisContManual` | **The manually set aperture**, u16 LE on the [aperture value](aperture_value.md) scale | Name and use **CERTAIN on the A7 IV firmware**; encoding **PROBABLE** (observed values sit on the grid) |
| `19` bit 0 | `uc_IrisIdctManualSet` | **The aperture is set manually**, and `pl[17..18]` is valid | **CERTAIN on the A7 IV firmware** |
| `19` bit 1 | — | Copied to the status record; no reader found | **UNKNOWN** |
| `19` bit 6 | — | **A step this frame** | **CERTAIN on the A7 IV firmware** |
| `19` bit 7 | — | **The step's direction**: set = +1, clear = −1 | Sign **CERTAIN**; which way the aperture moves **UNKNOWN** |

### 2.1 Absolute: bit 0 and `pl[17..18]`

At a still capture the body reads the status record. If bit 0 is set, it marks the exposure as
"lens manual aperture", keeps `pl[17..18]` as that aperture, and notifies its main processor
(event `0x12000007`). If bit 0 is clear, it sends the same event with 0. **CERTAIN on the A7 IV
firmware.** Whether the main processor then exposes at `pl[17..18]` is **UNKNOWN**.

### 2.2 Relative: bits 6 and 7

Each frame with bit 6 set gives one step of ±1. The body adds the steps up and, every second
frame, sends the running total to its main processor as event `0x12000006`, then clears it. So:

- **One step per frame.** No other bit carries a count, so N steps take N frames, and one event
  carries at most ±2.
- **Natives only, and not during a capture.** The step counts only for lens type 4
  ([lens types](lens_types.md)) and only while the lens link is in its normal state. A capture
  clears that state at its start.

**CERTAIN on the A7 IV firmware.** That event `0x12000006` moves the body's aperture setting by
that many steps, and the size of a step, are **PROBABLE** and **UNKNOWN** respectively. It reads
like an electronic aperture ring turning the body's aperture setting.

### 2.3 Third-party lens IDs

For a third-party [lens ID](lens_id.md) (`0xC000` and above, low 12 bits not `0xFFF`) the body
masks `pl[19]` with `0xC1` before using it. Bits 0, 6 and 7 are kept, and so is `pl[17..18]`; bit 1
and bits 2–5 are cleared. A model configuration flag can apply the same mask to every lens.
**CERTAIN on the A7 IV firmware.** So a third-party lens can use both the absolute and the
relative report.

## 3. Open questions

- What the main processor does with events `0x12000006` and `0x12000007`: whether it changes the
  aperture setting, and whether it requires the 0x08 declaration.
- The direction of a +1 step, and its size (1/3 stop or more).
- `pl[19]` bit 1.
- Which lenses set these fields. No lens observed sets `pl[19]` bit 0 or bit 6
  ([message 0x05](msg_0x05.md)); the 0x08 flags have not been checked against captures.
