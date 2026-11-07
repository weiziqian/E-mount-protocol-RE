# Message 0x0A — init echo and status-length select

**Summary.** A 16-byte init-class exchange in which **the lens echoes the body's request exactly**.
The last exchange of the init handshake. The request is more than an echo pattern: its first two
bytes can **shorten [message 0x05](msg_0x05.md) to its 105-byte form**, and `pl[8..9]` select the
fields of [message 0x06](msg_0x06.md). **The body chooses the request from the lens's own
[message 0x08](msg_0x08.md) `pl[13]`**, so it is not a fixed value.

**Direction:** both. Body request 16 bytes; lens reply 16 bytes.

**Class:** init (`0x02`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Observed payloads

| Direction | Device | When | Payload |
| --- | --- | --- | --- |
| B→L | Sony A6000, every lens | end of init | `ff 7f 00 00 00 00 00 00 3f 00 …` |
| B→L | Sony NEX-7, every lens | end of init | `ff 7f 00 00 00 00 00 00 3f 00 …` |
| B→L | Sony NEX-7 | **end of session**, immediately before [message 0x16](msg_0x16.md) | `00 00 00 00 00 00 00 00 00 00 …` |
| B→L | Sony a9 II, lens sends 0x08 `pl[13] = 0x81` | end of init | `ff 7f 00 00 00 00 00 00 3f 00 …` |
| B→L | Sony a9 II, lens sends 0x08 `pl[13] = 0xe1` | end of init | **`ff ff 03 00 00 00 00 00 3f 00 …`** |
| L→B | Sony SELP1650, SEL55210, SEL2870, SEL5518Z; Voigtländer 15 mm; Zeiss Loxia 21 mm; Viltrox EF adapter | both | **echoes the request exactly** |
| L→B | TECHART LM-EA9 | end of init | `ff 7f … 3f …` — the same shape |

Whole frame, checksum verified (A6000 + SELP1650, identical in both directions):

```
F0 19 00 02 00 0A | FF 7F 00 00 00 00 00 00 3F 00 00 00 00 00 00 00 | E2 01 55
```

The all-zero exchange is in three NEX-7 captures (SEL2870, Voigtländer 15 mm twice), always as
`0x0A` immediately followed by `0x16`, at the end of the trace.

## It is an exact echo

Every device measured returns the body's 16 bytes unchanged. That makes it plausibly a **link test
or echo/loopback check** — a way for the body to confirm the newly negotiated 1.5 Mbaud link carries
16 bytes intact in both directions before the status loop starts. PROBABLE.

Its position supports the reading: 0x0A is the final init exchange, after the baud-rate change and
after every descriptor has been transferred.

The echo is not the whole story. A lens can also **read** the request and change its later
behaviour on it — the next three sections.

## `pl[0..1]` can shorten message 0x05 to 105 bytes

[Message 0x05](msg_0x05.md) has a 105-byte and a 117-byte form. The lens picks a starting length from
its own [message 0x08](msg_0x08.md) `pl[13]`; the body's 0x0A request can then **force the short
form**:

| Request `pl[0]` `pl[1]` | Effect on the message 0x05 length |
| --- | --- |
| `ff 7f` | **set to 105** |
| `ff ff` | unchanged — keeps the value picked from message 0x08 |
| anything else | unchanged |

The request can only **shorten**. No value of `pl[0..1]` turns a 105-byte lens into a 117-byte one;
the long form needs both the lens's message 0x08 to declare it **and** the body not to send `ff 7f`.

Every request recorded from the A6000 and the NEX-7 is `ff 7f`, and every lens captured on them
sends 105 — including two whose message 0x08 declares the long form (SELP1650 `pl[13] = 0x8c` on the
A6000, Voigtländer 15 mm `pl[13] = 0xe1` on the NEX-7). The a9 II sends `ff ff` when the lens
declares `0xe1` — see the next section.

| Aspect | Confidence |
| --- | --- |
| Yongnuo firmware implements exactly this rule (three models, two separate code bases) | **CERTAIN** |
| It is protocol behaviour, i.e. Sony lenses follow it too | **PROBABLE** — every native capture agrees with it and none contradicts it |
| `ff ff` means the body expects the 117-byte form the lens declared | **CERTAIN** on the a9 II for `pl[13] = 0xe1` — see below |

## The request depends on the lens's message 0x08

The body builds this request from what the lens declared in [message 0x08](msg_0x08.md) `pl[13]`,
two exchanges earlier. Observed on one Sony a9 II, with nothing else in the lens's replies changed:

| Lens 0x08 `pl[13]` | a9 II 0x0A request `pl[0..2]` | Message 0x05 the body then expects |
| --- | --- | --- |
| `0x81` (declares 105) | `ff 7f 00` | 105 bytes |
| `0xe1` (declares 117) | `ff ff 03` | **117 bytes** |

With `0xe1`, a lens that sent the 105-byte form anyway completed the handshake and then lost the
body: the a9 II never sent [message 0x16](msg_0x16.md), cut the lens's power within seconds and
restarted the session, repeatedly. The same lens sending 117 bytes worked normally. So `ff ff` is
not merely permission — a body that sends it **requires** the long form. CERTAIN on this body.

### The rule, from the A7 IV firmware

The A7 IV starts the request at `ff 7f 00` and, if **0x08 `pl[13]` bits 5 and 6 are both set**
(`pl[13] & 0x60 == 0x60`), adds `0x80` to `pl[1]` and `03` to `pl[2]` — giving `ff ff 03`. Nothing
else in `pl[13]` enters the decision. That reproduces every a9 II observation: `0xe1` has both bits
(`ff ff 03`), `0x81` and `0x20` each have only one (`ff 7f`), and is why `0x20` does not draw the
long form although `& 0xFE` would declare it. CERTAIN on the A7 IV firmware; consistent with the
a9 II.

The A7 IV builds this form for natives and for adapters of sub-type `1`
([message 0x2B](msg_0x2B.md)). For the other adapter classes it sends `pl[0] = 40` and zeros.

`pl[2]` bits 0–1 always accompany the long form; their own meaning is UNKNOWN. Further bits of
`pl[2]` and `pl[9]` are taken from the lens's [message 0x3D](msg_0x3D.md) reply.

## `pl[8..9]` select the fields of message 0x06

`pl[8]` bit `n` asks for [message 0x06](msg_0x06.md) field `0x40 + n`, and `pl[9]` bit `n` for field
`0x48 + n`. The field table is in
[message 0x06 — fields](msg_0x06.md#fields-and-which-the-body-asks-for). `3f` is fields
`0x40`–`0x45`, the 39-byte payload `pl[0..38]`. CERTAIN on the A7 IV firmware.

What the A7 IV sends:

| Byte | Value |
| --- | --- |
| `pl[8]` | `3f`, plus `80` (field `0x47`, 4 bytes) for a native lens whose [0x08](msg_0x08.md) `pl[12]` has bit 5 set. Bit 6 is never set |
| `pl[9]` | bit 0 (field `0x48`) for an adapter that declares [0x5A](msg_0x5A.md); bits 1 and 2 (`0x49`, `0x4A`) from [0x3D](msg_0x3D.md) feature bits. None of them for a [third-party ID](lens_id.md#what-third-party-changes) |

Yongnuo reads only one case, `pl[8] == 7f`, which adds field `0x46` (45 bytes) and gives the
93-byte form ([message 0x06 — the 93-byte form](msg_0x06.md#the-93-byte-form)). Any other value
gets the 48-byte form.

Unlike the [message 0x05](msg_0x05.md) length, this one is chosen **by the body alone**: the lens
declares nothing for it.

**Only `3f` observed.** Every body request recorded carries `3f`: about twenty 0x0A requests, checksums
verified, from the A6000 (five lenses and adapters) and the NEX-7 (seven captures, five devices),
and three from the Sony a9 II, including both forms above. The end-of-session `00 00` request carries `00`. Every message 0x06
ever captured has the 39-byte payload.

| Aspect | Confidence |
| --- | --- |
| `pl[8..9]` is a field bitmap; what the A7 IV puts in it | **CERTAIN on the A7 IV firmware** |
| Yongnuo selects 93 on `pl[8] = 7f`, 48 otherwise (three models, two separate code bases) | **CERTAIN** |
| A Sony lens builds the fields the same way | **POSSIBLE** — no body has been seen sending anything but `3f` |

## `00 00` at the end of the session

The NEX-7 sends a second 0x0A with an all-zero payload just before [message 0x16](msg_0x16.md), the
shutdown request. Yongnuo treats `pl[0..1] = 00 00` as a mode of its own, separate from `ff xx`, and
on it shuts down parts of the lens. So `00 00` is likely a "leaving the active session" signal that
precedes the shutdown request. POSSIBLE.

Yongnuo also has a third mode for `pl[0..1] = 40 00`. No body has been seen sending it.

## Manufacturer notes

- **Sony**, **Viltrox**, **TECHART** — all echo. No device measured deviates.
- **Yongnuo** — echoes, and also reads the request: `pl[0..1]` for the message 0x05 length and the
  session mode, `pl[8]` for the message 0x06 length.

## Open questions

- Whether any body sends `7f` in `pl[8]`; the A7 IV does not.
- What the a9 II requests for `pl[13]` values other than `0x81` and `0xe1`, and which bits of
  `pl[13]` it decides on.
- What `pl[2] = 03` in the a9 II's `ff ff` request means.
- What the remaining request bytes encode.
- Whether a device that returned altered bytes would be rejected.
