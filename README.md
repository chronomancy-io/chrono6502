# chrono6502

A headless, cycle-exact MOS 6502/6510 core (Rust, no dependencies) purpose-built
to measure and verify [ChronoForth](https://github.com/chronomancy-io/chronoforth).
It boots the **real** ChronoForth kernel in-process — no VIC-II/SID/disk, just the
CPU plus minimal KERNAL/IEC stubs that serve Forth source from a ChronoForth
checkout's `forth/` and `test/` — so the whole Forth-2012 test suite runs in
under half a second (~0.45 s warm) and any word's exact cycle cost is one
command away.

It was extracted from `chronoforth/tools/chrono6502` into its own repository so it
can stand on its own; it has no build-time dependency on ChronoForth, only a
run-time one (you point it at an assembled image and a source tree).

## Scope

The cycle numbers are trustworthy for self-contained code, within these limits:

- **Documented opcodes only.** Undocumented/illegal opcodes are not emulated;
  executing one is a hard error. There is no IRQ/NMI — nothing ever interrupts
  straight-line code.
- **Decimal-mode ADC deviates from hardware.** ChronoForth math never sets the
  D flag; do not use this core to verify BCD arithmetic.
- **The 6510's `$00`/`$01` I/O port and ROM banking are not modeled.** Those
  addresses are plain RAM and no ROM is mapped; KERNAL calls are trapped and
  serviced in Rust instead.
- **KERNAL I/O is charged a nominal flat cost** (20 cycles per trapped call),
  so I/O-bound words are *not* exactly timed; self-contained stack/math words
  are.

## Build

```bash
cargo build --release
cargo test            # 11 unit tests of the cycle model (no ChronoForth needed)
```

The unit tests cover the CPU core alone. To exercise the emulator's real job —
booting ChronoForth — you need an assembled image (`durexforth.prg`), an ACME
symbol dump (`labels.vice`), and the ChronoForth source tree. Build them from a
ChronoForth checkout:

```bash
git clone https://github.com/chronomancy-io/chronoforth
cd chronoforth
make durexforth.prg                                      # needs the `acme` assembler
acme -I asm --vicelabels labels.vice asm/durexforth.asm  # same assembly, plus symbols
```

(ChronoForth's Makefile has no `labels.vice` target; the second command re-runs
the same ACME assembly with a VICE label dump added.)

## Usage

Point the three path flags at your ChronoForth checkout (`--repo` is where the
`forth/`/`test/` sources live; it defaults to `../..`, i.e. when the binary sits
in `chronoforth/tools/chrono6502`):

```bash
chrono6502 --prg   /path/to/chronoforth/durexforth.prg \
           --labels /path/to/chronoforth/labels.vice \
           --repo   /path/to/chronoforth \
           <command>
```

| Command | What it does |
|---------|--------------|
| `selftest` | Measure 22 primitives; assert stack effects for all 22, exact cycles for 7 (exit 1 on mismatch) |
| `ledger`   | Print cycles for the curated primitive set |
| `word NAME [in…]` | JSR one word by symbol, print cycles + resulting stack |
| `boot "<forth>" [more…]` | Boot the full system, run Forth one-liners, print output |
| `gate` | Run the entire Forth-2012 suite; exit 0 ⇔ 0 errors (the correctness gate) |
| `defcyc "<defs>" NAME [in…]` | Compile a definition, JSR-measure it on the post-boot image |
| `dict` / `dis` | Dictionary dump / disassembly helpers |

`CHRONO_MAXCYC=<n>` raises the per-word cycle ceiling for `defcyc`/`word`
(default 5,000,000) — useful for heavy user code. Other env knobs: `CHRONO_WATCH`,
`CHRONO_CRASH`, `CHRONO_DEBUG`, `CHRONO_TRACE`, `CHRONO_DUMP`.

## How it measures

`call_word` sets up the split LSB/MSB zero-page stack (`X = 256 − n`, TOS at
index X, zero-page-X wrap), pushes a sentinel return address, sets PC to the
word, and counts cycles until the matching `RTS`. `boot`/`gate` run the kernel
from its SYS entry; a write to `$D7FF` halts with an exit code (mirroring VICE's
debug cart). The base.fs build-time `0 $d7ff c!` line is stripped on the fly so
the boot reaches the interactive prompt.

## Validation

1. `selftest`: stack effects asserted on all 22 straight-line primitives; exact
   agreement with hand-derived cycle counts asserted on 7 of them (drop 14,
   dup 24, swap 38, over 24, 1+ 15, + 34, - 34).
2. `cargo test`: page-cross penalties, branch timing, ADC/SBC flags, ZP-X wrap, JSR/RTS.
3. The full Forth-2012 suite passes in-emulator (`gate`).
4. VICE cross-check (external, via the ChronoForth project — the harness is not
   shipped in this repo): during ChronoForth development this core was checked
   against a CIA-timer benchmark run in real VICE and reproduced VICE's counts,
   including a placement-dependent branch page-cross.

CI runs the unit tests on every push, plus a `gate` job that checks out
ChronoForth, assembles the image, and runs `selftest` + the Forth-2012 gate.

## License

MIT — see [LICENSE](LICENSE).
