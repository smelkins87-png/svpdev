# Open questions and the test plan

Status legend: **OPEN** = untested. **ANSWERED** = settled by a source (cited). **DECIDED** = a project decision.

## A. Questions for the flash carts (the project's real targets)

| ID | Question | How to test | Status |
|---|---|---|---|
| F1 | Does the EverDrive Pro's MCU accept EDIO commands (`CMD_F_FOPN`, `CMD_F_FRD_MEM`) while the **SVP mapper** is running? | ROM with the VR serial. `ed_init()`, open a file, `CMD_F_FRD_MEM` into ROM offset 0x100000, then read it back from both the 68k and an SSP PM stream. Reference code: `krikzz/MEGA-PRO/edio/everdrive.c` | OPEN |
| F2 | Is the SVP DRAM at MCU address 0x800000 (PSRAM1) on the Pro? | `CMD_MEM_WR` a pattern to 0x800000, read it at 68k 0x300000 | OPEN (inferred from `map_svp_v2.sv` wiring) |
| F3 | Do MCU DMA and SSP ROM fetches collide safely? | run F1 while the SSP loops in ROM vs in IRAM; compare | OPEN (recommendation: park the SSP in IRAM) |
| F4 | MegaSD: does "SEGA SSF" at 0x100 + "GM MK-1229 -00" at 0x180 give SVP **and** banking? | 4MB image with a marker per 512KB bank; write 0xA130F2–FE; check that the SVP still answers the XST handshake | OPEN |
| F5 | Pro: is the first word of each VDP DMA from cart stale? | DMA a known pattern; inspect VRAM | OPEN (predicted from `map_svp_v2.sv:280,335`) |
| F6 | Do all three cores give identical results for the portability checklist (`20-…` §9)? | build one test ROM that exercises items 1–25 and prints results | OPEN |
| F7 | MiSTer: 3MB linear + SSF2 banks with SVP active | 4MB image, read 0x200000+, bank-switch | OPEN (predicted from `system.sv`) |
| F8 | Custom FPGA mapper | — | **DECIDED: not pursued** |

## B. Questions about the real SVP (for hardware owners; still relevant for portability)

| ID | Question | Status |
|---|---|---|
| H1 | What does an SSP PM read at word 0x100000–0x17FFFF return? (`examples/probe_unused_window.svp`) | OPEN. 68k side ANSWERED by svpdoc (internal-bus echo) |
| H2 | Hidden DRAM above 128KB? | ANSWERED: no. svpdoc says 0x320000–0x37FFFF are mirrors |
| H3 | Unlabelled die pads 72/74/75/98/100 next to the ROM bus (`img/decap/06_pinout.jpg`): an unused A20 or chip-select? | OPEN (needs a scope) |
| H4 | PM mode `?` bits 25:21 in combination | OPEN (single bits make programming fail, per the wiki) |
| H5 | 0xA1500A/C/E function | OPEN |
| H6 | Strided/decrement/special PM reads **on ROM** | OPEN on hardware. On the FPGA they work as `GetNextMA` |
| H7 | Special increment +31 vs +32 | wiki and PicoDrive say +31; svpdoc says +32. Use +31 |
| H8 | Does the blind write increment the address? | FPGA: no. HW: disputed |
| H9 | Does VR's ROM contain an internal-ROM copy at 0x1F800 (which would explain VR running on FPGA cores without an IROM)? | OPEN. Check a VR dump at 0x1F800 against the internal-ROM listing in the wiki |
| H10 | EXT2 "random" reads with ST5/ST6 clear; EXT1 writes with ST clear; UP1 alone | OPEN |

## C. Suggested order

1. F1/F2 (they decide whether the Pro can exceed 2MB with the stock mapper).
2. F4 (MegaSD).
3. F6 (one conformance ROM for all targets).
4. F5 and F7.
5. Hardware-only items, if a real cart and scope are available.

Record results in this file (date, cart, firmware version, outcome).
