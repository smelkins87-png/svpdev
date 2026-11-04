# CLAUDE.md: svpdev

Sega SVP (Virtua Racing DSP, Samsung SSP1601) development repo: boards (KiCad), samples (68k + SSP assembly), tools (`ssp16asm` assembler and `svptool`, both Rust).

**The SVP knowledge base is in `docs/svp/`. Read `docs/svp/00-quick-reference.md` before writing or reviewing any SSP code.** For FPGA behaviour details, see `docs/svp/20-fpga-core-internals.md` (§9 is the portability checklist).

## Project context and decisions

* The goal is homebrew that uses the SVP with **more than 2MB of ROM**, running on the **Mega EverDrive Pro** and **MegaSD**. Both use srg320's FPGA SVP core. MiSTer uses the same core and is the reference. The code should also stay correct on a real VR cart.
* **No custom FPGA mapper.** Only stock mappers. Don't propose building one.
* SSP code and everything the SSP reads must be in the first 2MB on every target. Anything beyond 2MB is 68k-side only and per-cart; see `docs/svp/30-flashcarts-rom-size.md`.

## Rules for SSP code (summary; details are in the quick reference)

* Branch and `mod` conditions: only `always`, `z=`, `n=` for `bra`/`call`. Only `always`, `z=0`, `n=0` for `mod`. The only `mod` ops are `shr`, `shl`, `neg`, `abs`.
* Don't use `ror`/`rol`/`inc`/`dec`, flag ops other than `setie`, the L/OV flags, interrupts, EXT5, or EXT→EXT moves.
* Use `ld st, 0x0060` for PM0–PM3, and `ld st, 0x0000` for XST and status.
* Re-write PMC (`ld ext6, addr`, then `ld ext6, mode`) before **every** blind access. Reads are prefetched.
* In `ssp16asm`, `ext7` means **AL** (accumulator low word): `ld ext7, 0` clears A[15:0].
* 68k → SSP commands must be **non-zero word** writes to 0xA15000.
* Keep the serial `GM MK-1229 -00` at ROM 0x180. Keep a copy of the internal-ROM dump at ROM 0x1F800–0x1FFFF, because the FPGA cores have no internal ROM.
* Don't rely on 0xA15006 halting the SSP; park it in an IRAM loop instead. Set VDP DMA sources +2 bytes for cart ROM and DRAM.

## Tooling

* Build the assembler: `cd tools/ssp16asm && cargo build --release`. Known bugs are in `docs/svp/40-toolchain.md`. The big ones:
  * code beyond the `-b` base-file size is silently dropped;
  * labels wrap at word 0xFFFF;
  * an instruction directly after `DW` fails, so insert an `org`;
  * always use `--2meg` for real-hardware builds.
* After editing SSP code, assemble it to check syntax. The templates in `docs/svp/examples/` all assemble.
* There's no automated test suite for the samples. They need real hardware or an SVP-capable emulator/cart.

## Keeping the knowledge base current

* Log new findings and test results in `docs/svp/50-open-questions-and-tests.md` (date, cart, firmware, outcome). Update the quick reference if a rule changes.
* External sources and pinned revisions are in `docs/svp/60-sources.md`. Re-fetch from there instead of relying on memory. The wiki is reachable via `raw.githubusercontent.com/wiki/jdesiloniz/svpdev/<Page>.md`.
