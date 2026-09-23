# Message 0x1B — the aperture command

**Summary.** The body commands an aperture; the lens replies with an aperture. Both directions carry
an [aperture value](aperture_value.md) at `pl[0..1]`, repeated at `pl[2..3]`.

**Direction:** both.

**Class:** init (`0x02`).

**Frame length:** 19 bytes — 10 payload bytes, `pl[0..9]`.

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

---

## Request payload — body → lens

| `pl` | Size | Name | Description | Confidence |
| --- | --- | --- | --- | --- |
| `0..1` | 2 | Aperture | The commanded aperture, u16 LE, on the [aperture value](aperture_value.md) scale | **CERTAIN** |
| `2..9` | 8 | — | Purpose unknown; values not recorded | **UNKNOWN** |

## Response payload — lens → body

| `pl` | Size | Name | Description | Confidence |
| --- | --- | --- | --- | --- |
| `0..1` | 2 | Aperture | An aperture, u16 LE, on the [aperture value](aperture_value.md) scale | **CERTAIN** |
| `2..3` | 2 | Aperture, repeated | The same value again | **CERTAIN** |
| `4` | 1 | — | Purpose unknown. Observed values: `0x00`, `0x01` | **UNKNOWN** |
| `5..6` | 2 | — | Purpose unknown. Observed value: `0x00 0x00` | **UNKNOWN** |
| `7..8` | 2 | — | Purpose unknown. Observed value: `0x0A 0x0A` | **UNKNOWN** |
| `9` | 1 | — | Purpose unknown. Observed values: `0x01`, `0x07` | **UNKNOWN** |

The value in `pl[0..1]` is either the aperture the request asked for or the lens's own current
aperture; both are observed, and a settled lens reports the same number either way.

## Frame examples

Response to a request carrying f/2.0 (`pl[0..1]` = `0x00 0x12` = 4608), TECHART LM-EA9:

```
F0 13 00 02 00 1B  00 12 00 12 00 00 00 0A 0A 01  69 00 55
```

Response to a request carrying f/8 (`pl[0..1]` = `0x00 0x16` = 5632), same device:

```
F0 13 00 02 00 1B  00 16 00 16 00 00 00 0A 0A 01  71 00 55
```

Request, Sony a9 II to a TECHART LM-EA9: `pl[0..1]` = `0x00 0x12` = 4608 = f/2.0, on two separate
occasions. The remaining payload bytes were not recorded.

## When it is sent

At the shutter press. On a Sony a9 II the frame appears only after the shutter is released, inside a
burst of init-class requests — `0x0A 0x0B 0x1B 0x28 0x19 0x19 0x28 0x0B 0x0A` — and not in the idle
loop. It appears in no idle or init-only observation.

The value the a9 II sent, f/2.0, is the maximum aperture that device declares in
[message 0x05](msg_0x05.md) `pl[44]`.

## Open questions

- **`pl[4..9]` of the response.** Six bytes; two of them take more than one observed value.
- **The full payload of a request.** Only `pl[0..1]` has been recovered.
- **Whether a native E-mount lens ever receives this message.** It has been observed only at devices
  that populate the aperture descriptor at [message 0x05](msg_0x05.md) `pl[44..59]`. Devices that
  leave that block empty may be commanded by another route.
- **Whether the class byte is significant.** Only init-class frames have been observed.
