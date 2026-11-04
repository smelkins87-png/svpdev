# SVP knowledge base

Consolidated research on the Sega Virtua Processor (SVP, the Samsung SSP1601 in Virtua Racing) for homebrew that targets **real hardware and the FPGA re-implementations**: Mega EverDrive Pro/CORE, MegaSD and MiSTer. Compiled Nov 2026 from this repo, the project wiki, notaz's svpdoc, PicoDrive, and the HDL of srg320's SVP core (used by all three carts).

## Read in this order

| File | Use it for |
|---|---|
| [`00-quick-reference.md`](00-quick-reference.md) | **Start here.** Memory maps, registers, safe instruction subset, PM programming, handshake, DMA, ROM layout, ROM-size rules |
| [`20-fpga-core-internals.md`](20-fpga-core-internals.md) | How the FPGA core executes every instruction and PM access, line by line. The 25-item FPGA-vs-hardware portability checklist is in §9 |
| [`30-flashcarts-rom-size.md`](30-flashcarts-rom-size.md) | What each cart allows above 2MB (MiSTer linear + SSF2 banks; Pro SD overlays via EDIO; MegaSD untested) |
| [`10-real-hardware-and-2mb-limit.md`](10-real-hardware-and-2mb-limit.md) | Real-cart research: where the 2MB limit comes from, unexplored address windows, effective-capacity techniques (§C) |
| [`40-toolchain.md`](40-toolchain.md) | `ssp16asm` syntax, verified encodings, assembler bugs, corrections to the samples |
| [`50-open-questions-and-tests.md`](50-open-questions-and-tests.md) | What's still unknown, how to test it, where to log results |
| [`60-sources.md`](60-sources.md) | External sources, pinned revisions, re-fetch commands, key line numbers |
| [`examples/`](examples/) | SSP assembly templates. Each one assembles with the repo's `ssp16asm` |

## Examples

| File | Shows |
|---|---|
| `xst_handshake.svp` | 68k ↔ SSP command/reply through XST |
| `pm_dram_write.svp` | write stream into DRAM through PM4 |
| `pm_rom_read.svp` | read stream from anywhere in the 2MB ROM through PM1, copied to DRAM |
| `iram_overlay_loader.svp` | load code from anywhere in ROM into IRAM and jump to it |
| `custom_stride_fpga.svp` | FPGA-only arbitrary PM stride (`nnn=7` + `ld -, ext7`) |
| `probe_unused_window.svp` | research probe for the SSP word window 0x100000–0x17FFFF (real hardware) |

Build any of them with `tools/ssp16asm/target/release/ssp16asm <file>.svp out.bin`. Add `-b <68k.bin>` to merge with 68k code.

## Project decisions

* **Targets:** Mega EverDrive Pro and MegaSD (with MiSTer as a reference and dev platform). The code should also stay correct on a real VR cart.
* **No custom FPGA mapper.** Work only within the stock SVP mappers.
* **Beyond 2MB:** 68k-side only (SSP code and SSP-read data stay in the first 2MB). Use per-cart mechanisms (`30-…`) plus effective-capacity techniques (`10-…` §C).

## Confidence conventions

Claims are tagged **[FACT]** (read from a cited source or code), **[INFERENCE]** or **[SPECULATION]**, and **[HW]** / **[FPGA]** / **[ALL]** for which implementation they apply to. **Nothing in here has been run on real hardware or on a flash cart by the author of these notes.** The FPGA statements come from reading HDL. Verify them with the tests in `50-…`.
