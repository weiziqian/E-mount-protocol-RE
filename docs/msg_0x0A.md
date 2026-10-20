# Message 0x0A — init echo and status-length select

**Summary.** A 16-byte init-class exchange in which **the lens echoes the body's request exactly**.
The last exchange of the init handshake. The request is more than an echo pattern: its first two
bytes can **shorten [message 0x05](msg_0x05.md) to its 105-byte form**, and `pl[8]` selects the
length of [message 0x06](msg_0x06.md).

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
| B→L | Sony a9 II | end of init | not recorded — **UNKNOWN** |
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

Both bodies whose request is known send `ff 7f`, so on them every lens should send 105 — and every
lens captured on them does, including two whose message 0x08 declares the long form (SELP1650
`pl[13] = 0x8c` on the A6000, Voigtländer 15 mm `pl[13] = 0xe1` on the NEX-7).

| Aspect | Confidence |
| --- | --- |
| Yongnuo firmware implements exactly this rule (three models, two separate code bases) | **CERTAIN** |
| It is protocol behaviour, i.e. Sony lenses follow it too | **PROBABLE** — every native capture agrees with it and none contradicts it; but no capture has a body that sends `ff ff`, so the rule has never been seen *making* a difference on the bus |
| `ff ff` means "the body accepts the 117-byte form" | **POSSIBLE** — inferred from the rule; no body has been seen sending it |

## `pl[8]` selects the length of message 0x06

In the same handler, Yongnuo sets the base length of [message 0x06](msg_0x06.md) from `pl[8]`, for
the rest of the session:

| Request `pl[8]` | Message 0x06 base frame length | Payload before the tail |
| --- | --- | --- |
| `7f` | **93** | `pl[0..83]`, 84 bytes |
| anything else | **48** | `pl[0..38]`, 39 bytes |

"Base" because message 0x06 also carries a per-frame tail (events, query answers) that adds to
either length; the tail simply starts later in the 93-byte form. What the two forms contain is in
[message 0x06 — the 93-byte form](msg_0x06.md#the-93-byte-form).

Unlike the [message 0x05](msg_0x05.md) length, this one is chosen **by the body alone**: the lens
declares nothing for it, and the body can select the longer form, not only the shorter.

**Never observed.** Every body request recorded carries `3f`: about twenty 0x0A requests, checksums
verified, from the A6000 (five lenses and adapters) and the NEX-7 (seven captures, five devices).
The end-of-session `00 00` request carries `00`. Every message 0x06 ever captured is 48 bytes long
before its tail. No capture of a newer body's 0x0A request exists.

| Aspect | Confidence |
| --- | --- |
| Yongnuo selects 93 on `pl[8] = 7f`, 48 otherwise (three models, two separate code bases) | **CERTAIN** |
| The value it sets is the 0x06 base length | **CERTAIN** — the 0x06 sender extends the payload by exactly that amount |
| It is protocol behaviour that Sony lenses follow too | **POSSIBLE** — no body has been seen sending `7f`, so nothing on the wire bears on it either way |

Note the shape: `ff 7f` vs `ff ff` in `pl[0..1]`, and `3f` vs `7f` in `pl[8]` — in both places the
"extended" value is the observed one with **one more bit set**. That reads like a bitmap of optional
formats the body accepts, in the manner of [message 0x01](msg_0x01.md). POSSIBLE.

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

- What the a9 II, or any body newer than the A6000, sends in `pl[0..1]` and `pl[8]`.
- Whether a Sony lens actually switches to 117 bytes when a body sends `ff ff`.
- What the remaining request bytes encode.
- Whether a device that returned altered bytes would be rejected.
