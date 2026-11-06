# AF range — full frame or centre only

**Summary.** After init, the body decides how much of the frame autofocus may use with this device:
the **full range**, **centre only**, or **none**. The decision is made from fields of
[0x07](msg_0x07.md), [0x08](msg_0x08.md) and [0x3D](msg_0x3D.md) and from the body's built-in lens
library. A native device with no library entry gets the full range **only if it declares a 0x3B
file** in 0x3D `pl[0]`; otherwise it is classed centre-only.

**Source.** Read from the Sony A7 IV 6.02 camera-processor firmware; **CERTAIN on the A7 IV
firmware** unless marked. What the main processor does with each class beyond the logs named below
is not visible. Other bodies are untested.

`pl` is the payload of a message: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`.

## The inputs

| Input | Taken from |
| --- | --- |
| category | the [lens type](lens_types.md): native (4) → `1`, type 5 → `3`, any other → `0` |
| ID | [0x07](msg_0x07.md) `pl[9..10]`; for type 5, [0x08](msg_0x08.md) `pl[34..35]` |
| prohibitions | 0x08 `pl[32]` bit 0; 0x3D `pl[32]` bits 0–7 |
| lens supplies its own data | 0x3D `pl[0] ≠ 0` — it declares a [0x3A/0x3B](msg_0x3A.md) file |
| library entry usable | 0x08 `pl[14]` bit 0 |
| version | 0x07 `pl[2..3]`; category 3 requires more than `0x0107` |

The body reads these after the handshake, when it prepares the 0x3A/0x3B transfer, from the replies
as stored. For a [third-party ID](lens_id.md) that means after its filtering: 0x3D `pl[0]` is
untouched, 0x3D `pl[32]` keeps only bit 0, and 0x08 `pl[32]` is zero.

## The decision, category 1 (native)

| Condition | Class |
| --- | --- |
| any prohibition set | none |
| library entry for the ID, `pl[14]` bit 0 set | **full** |
| library entry for the ID, `pl[14]` bit 0 clear | none |
| no library entry, 0x3D `pl[0] ≠ 0` | **full** — the lens's own file is used |
| no library entry, 0x3D `pl[0] = 0` | **centre only** (ID `0x8010` with `pl[14]` bit 0 clear: none) |

The built-in library for category 1 holds 18 early Sony E-mount IDs, `0x8011`–`0x802D`. Any other
native — every third-party lens, and every newer Sony lens — has no entry and depends on 0x3D
`pl[0]`.

Category 0 (adapters) always gets class none. Categories 2 and 3 use the same prohibitions and
their own library tables (A-mount IDs).

## What the classes do

- **Centre only** disables Scanless AF (log `Scanless NG because of CenterOnlyLens`) and feeds
  several AF-area decisions.
- **Full** leaves both available.
- **None** is used by other AF-area calculations; what it restricts is UNKNOWN.

## Implementing it

- A native device should **declare 0x3D, 0x3A and 0x3B, set 0x3D `pl[0]` bit 0, and serve the file**.
  Clearing 0x3D `pl[0]` turns it centre-only.
- Keep 0x08 `pl[32]` bit 0 and 0x3D `pl[32]` clear.

Every device measured — Yongnuo, TTArtisan, the LM-EA9 — sends 0x3D `pl[0] = 01` and 0x3D
`pl[32] = 00`.

## Focus-drive speed

A separate input, not part of the class. The body takes a drive-speed calibration from the lens
when 0x3D `pl[49] & 0x30` is non-zero; otherwise from a 53-entry table of Sony IDs in its own
firmware; otherwise it has none and schedules focus moves the older way. A
[third-party ID](lens_id.md) has `pl[49]` cleared and is in no table, so it never gets one. The
calibration's format in 0x3D has not been traced.

## Open questions

- What class none restricts, and whether other bodies make the same decision.
- The format of the 0x3D speed calibration.
