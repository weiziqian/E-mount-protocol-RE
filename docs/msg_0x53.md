# Message 0x53 — enable iris moves by 0x51; photo and movie mode

**Summary.** Three bytes. The Sony a7CR sends `80 00 00` once in the init, just before [0x0A](msg_0x0A.md), to a lens that declares
the async-iris group (0x51 to 0x53); the lens replies `00 00 00`. It is what lets [0x51](msg_0x51.md) move the iris: a Sony SEL24F28G or SEL40F25G initialised
without it answered every 0x51 with `n 02 …` and the iris did not follow, while one initialised with it (with or without the rest of the post-init
burst) answered `n 01 …` and followed. It is `pl[0]` bit 7 that counts: `80` or `ff` enabled the moves, `00` or `01` left every 0x51 refused. Everything else worked
without it. **PROBABLE** (SEL24F28G and SEL40F25G).

Bit 7 chooses what drives the iris: set, the lens follows [0x51](msg_0x51.md) and ignores changes of the [0x03](msg_0x03.md) target; clear, it
follows the 0x03 target, which an a7CR in movie mode steps every loop; a single jump of the target is followed too (a SEL24F28G went
to f/8 and f/4 at once, and ignored the same jumps with bit 7 set). The a7CR sends it again to switch modes: `12 87 00` on entering movie mode and `93 21 00` on returning to photo mode. The lens answers with `pl[0]` bit 7 cleared (`12 87 00`,
`13 21 00`). Sent mid-session to a SEL40F25G, `12 87 00` made it refuse 0x51 while the iris still followed 0x03, and `93 21 00` made it
accept 0x51 again. **PROBABLE.**

**Direction:** both. **Class:** async (`0x03`).

`pl` is the payload: `pl[n]` is payload byte `n`, i.e. absolute frame offset `n + 6`. Ranges
`pl[a..b]` are inclusive of both ends: `pl[a]` through `pl[b]`, length `b - a + 1`.

## Open questions

- What the low bits of `pl[0]` (`00`, `12`, `13`) and `pl[1]` (`87`, `21`) select.
