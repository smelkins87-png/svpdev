# Sources and how to re-fetch them

All of the external code was read in November 2026. Pin these revisions when re-checking line numbers.

## In this repository

| Path | What |
|---|---|
| `img/decap/06_pinout.jpg`, `08_blocks.jpg` | die pinout (38 ROM pads = A0–A19, D0–D15, /CE, /OE) and floor plan (2KB IROM, 2KB IRAM, 1KB data RAM) |
| `boards/svp-adapter-board/svp-adapter-board.net` | ROM footprint nets: A0–A19 only, pins 1 and 44 unconnected |
| `samples/*` | 68k + SSP sample code |
| `tools/ssp16asm`, `tools/svptool` | assembler and ROM splitter |

## External

| Source | Revision | Fetch |
|---|---|---|
| Project wiki (jdesiloniz/svpdev): Internal-ROM, Timings, External-registers-behaviour, Condition-flags, Interrupts, Virtua-Racing-test-code | as of Nov 2026 | `curl https://raw.githubusercontent.com/wiki/jdesiloniz/svpdev/<Page>.md` (github.com/…/wiki is blocked by the proxy; raw works) |
| notaz, `svpdoc.txt` | `$Id: svpdoc.txt 964 2014-09-23` | `https://notaz.gp2x.de/docs/svpdoc.txt` (the host must be allowed in the session's network settings) |
| PicoDrive SVP emulation | master, Nov 2026 | `raw.githubusercontent.com/notaz/picodrive/master/pico/carthw/svp/{ssp16.c,svp.c,memory.c,compiler.c}` |
| MiSTer Genesis core | `adc0c42` | `git clone --depth 1 https://github.com/MiSTer-devel/Genesis_MiSTer` → `rtl/SVP/*.vhd`, `rtl/system.sv`, `Genesis.sv` |
| Krikzz MEGA-PRO dev sources (MIT) | `7064f45` | `git clone --depth 1 https://github.com/krikzz/MEGA-PRO` → `fpga/mapper/map_svp/*`, `fpga/mapper/lib_base/*`, `edio/*`, `extended-ssf.md` |
| MegaSD | closed | Terraonion's 1.02R01 notes: SVP via "SSP160x core used with permission from Sergey Dvodnenko" |

## Key line references

| What | Where |
|---|---|
| SSP ROM port 20 bits | `SVP.vhd:24`, `:393`, `:469` (MEGA-PRO copy) |
| PM increment logic | `SVP.vhd:107-138` (`GetNextMA`) |
| Overwrite | `SVP.vhd:140-158` |
| EXT read mux | `SVP.vhd:279-322` |
| 68k registers, HALT not used on the Pro | `SVP.vhd:197-276`, `:169` |
| Condition decode | `SSP160x.vhd:427-434` |
| Reset PC 0x400 | `SSP160x.vhd:504` |
| ST layout | `SSP160x.vhd:538` |
| Pro 68k decode | `map_svp_v2.sv:98-113,172,200-215` |
| DMA latency, arbitration | `map_svp_v2.sv:280,316,335` |
| MiSTer 68k ROM decode / SSF2 banking with SVP | `system.sv:926-940,974-995`, `:620` |
| MiSTer SVP detection by serial | `Genesis.sv:998-999` |
| EverDrive I/O file→memory command | `edio/everdrive.c:52 (CMD_F_FRD_MEM)`, `:204-211`, `:618-667` |
| Pro cart memory map (2×8MB PSRAM) | `edio/everdrive.h:97-105` |
