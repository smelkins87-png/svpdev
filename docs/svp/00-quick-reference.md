# SVP quick reference

One-page cheat sheet for writing SVP (Sega Virtua Processor / Samsung SSP1601) homebrew that has to run on **real Virtua Racing hardware and the FPGA cores** (Mega EverDrive Pro/CORE, MegaSD, MiSTer).

Where to find the detail:

* `20-fpga-core-internals.md`: FPGA behaviour
* `10-real-hardware-and-2mb-limit.md`: real hardware
* `30-flashcarts-rom-size.md`: ROM size on flash carts
* `40-toolchain.md`: `ssp16asm` and `svptool`

Tags: **[HW]** = real SVP, **[FPGA]** = srg320 core (all three carts), **[ALL]** = both.

---

## 1. Memory maps

### 68k view (byte addresses)

| Range | [HW] | [FPGA: EverDrive Pro] | [FPGA: MiSTer] |
|---|---|---|---|
| 0x000000–0x1FFFFF | ROM (2MB) | ROM | ROM |
| 0x200000–0x2FFFFF | echo of the SSP's last bus word | stale DRAM latch | **more ROM if the image is >2MB** (+ SSF2 banks) |
| 0x300000–0x31FFFF | DRAM 128KB | DRAM | DRAM |
| 0x320000–0x37FFFF | 3 DRAM mirrors | DRAM mirrors | DRAM mirrors |
| 0x390000–0x39FFFF | cell arrange 1 (reads) | cell arrange 1 (**reads and writes**) | cell arrange 1 |
| 0x3A0000–0x3AFFFF | cell arrange 2 | cell arrange 2 (reads and writes) | cell arrange 2 |
| 0x3B0000–0x3FFFFF | reads 0xFFFF | DRAM mirror | not decoded by the SVP block (avoid) |
| 0xA15000/2 | XST (r/w) | XST | XST |
| 0xA15004 | status: bit0 SA (SSP wrote XST, cleared on read), bit1 CA | same | same |
| 0xA15006 | "halt" (VR writes 0x0A around DMA) | **latched, no effect** | halts the SSP |
| 0xA15008–E | ? (reads 0xFFFF) | reads 0 | reads 0 |

* Cell arrange covers only the **first 64KB** of DRAM.
* Cell arrange 1 address bits: `{a[15:13], a[6:2], a[12:7], a[1]}`. Cell arrange 2: `{a[15:12], a[5:2], a[11:6], a[1]}` (word-address bits of the 68k offset).

### SSP program space (16-bit PC, word addresses)

| Range | [HW] | [FPGA] |
|---|---|---|
| 0x0000–0x03FF | IRAM (1K words, written through PMs) | IRAM |
| 0x0400–0xFBFF | ROM words 0x400–0xFBFF (68k bytes 0x800–0x1F7FF) | ROM |
| 0xFC00–0xFFFF | **internal ROM** (boot, fill/math routines, sine table at 0xFEE3) | **cart ROM** words 0xFC00–0xFFFF (no internal ROM exists) |

Reset: [HW] internal-ROM boot at 0xFC08, then 0x0400. [FPGA] PC = 0x0400 directly.

**Portable rule:** place a byte-exact copy of the internal-ROM dump at ROM bytes 0x1F800–0x1FFFF. Never put anything else there.

### SSP external space (PM addresses, 21-bit words; ×2 = 68k address)

| Word range | Target |
|---|---|
| 0x000000–0x0FFFFF | ROM, read-only (writes ignored) |
| 0x100000–0x17FFFF | [HW] unknown / untested. [FPGA] no access (PM data stays stale) |
| 0x180000–0x18FFFF | DRAM (64K words) |
| 0x1C8000–0x1C83FF | IRAM ([FPGA] mirrored up to 0x1CFFFF) |

---

## 2. SSP registers

| # | Name | Notes |
|---|---|---|
| 0 | `-` | reads 0xFFFF. As a destination: blind read. As a source: blind write |
| 1, 2 | `X`, `Y` | `P = X*Y*2` updates the moment either is written |
| 3 | `A` | 32-bit. `ld a, s` writes A[31:16] only, **no flags**. `ld a, p` copies 32 bits |
| 4 | `ST` | `N OV Z L GPI0 GPI1 MACS OP IE ST6 ST5 RB[1:0] RPL[2:0]` (bit 15→0). Writing ST doesn't change flags |
| 5 | `STACK` | 6 levels, circular (overflow silently overwrites) |
| 6 | `PC` | write = jump |
| 7 | `P` | read gives P[31:16] |
| 8–14 | `EXT0`–`EXT6` | PM0, PM1, PM2, XST, PM4, EXT5, PMC |
| 15 | `AL` | A[15:0]. **`ssp16asm` calls this `ext7`** |

Pointer registers r0–r2 → RAM-A, r4–r6 → RAM-B (256 words each).

* Modifiers: `(rX)` none, `(rX+!)` +1, `(rX-)` −1 mod 2^RPL, `(rX+)` +1 mod 2^RPL.
* `r3`/`r7` addressing is only `|00 |01 |10 |11` (= slots 0–3).
* [FPGA] RB is ignored. `ld r3, a` / `ld a, r3` store and read a spare byte, and `ld r3, imm` is a NOP.

---

## 3. Instructions: what is safe everywhere [ALL]

* **Conditions:** only `always`, `z=0`, `z=1`, `n=0`, `n=1`.
  * [FPGA] `l=`, `ov=`, `gpi*=` are never true. `diof=` is always true.
* **`mod`:** only `always`, `z=0`, `n=0` with `shr` (arithmetic), `shl`, `neg`, `abs`.
  * [FPGA] `mod z=1|n=1, …` does nothing.
  * [FPGA] `ror`, `rol`, `inc` and `dec` are NOPs.
* **Flag ops:** `mod f, setie` is the only one that does anything, and it has no useful effect. [FPGA] There are no interrupts.
* **Flags:** only N and Z exist in practice. [FPGA] L and OV are always 0. Build carry/overflow from compares.
* **Never move one EXT register straight to another** (`ld ext4, ext1`). Go through A/X/Y.
* **ALU operands:**
  * Immediates and 16-bit operands go to the high word: `andi 0x0002` → `A &= 0x00020000`, so the low word is cleared too.
  * `cmp`/`sub` use all 32 bits.
  * Clear A with `eor a, a` (it covers all 32 bits).
* **Cycles:** 1-word instructions take 1 cycle. Immediate, `bra`, `call`, `((rX))` and `(a)` forms take 2 on the FPGA. The wiki measured `((rX))` at 3 on hardware.
* **Speed:** code runs ~5× faster from IRAM than from ROM on hardware (≈44ns vs ≈216ns per instruction). On the FPGA, ROM fetches also lose to any 68k ROM access.

---

## 4. External registers and PM programming [ALL unless marked]

| Reg | ST5=0 / ST6=0 | ST5=1 / ST6=1 |
|---|---|---|
| EXT0 | read: `{CA,SA}` status, read clears CA | PM0 |
| EXT1 | [FPGA] 0 | PM1 (needs **ST5**) |
| EXT2 | [FPGA] 0 | PM2 (needs **ST6** [FPGA]) |
| EXT3 | XST: write sets SA | PM3 (needs **ST6**) |
| EXT4 | PM4, always | PM4 |
| EXT5 | [FPGA] dead | [FPGA] dead ([HW] write-only PM) |
| EXT6 | PMC (two writes: address word, then mode word) | same |

Use **`ld st, 0x0060`** before PM0–PM3 work, and **`ld st, 0x0000`** before touching XST or status.

**Mode word** `d s nnn v ?? ??? aaaaa`:

| Field | Meaning |
|---|---|
| `aaaaa` | word-address bits 16–20 (ROM 0x00–0x0F, DRAM 0x18, IRAM 0x1C with low word 0x8000+) |
| `nnn` | increment: 0, +1, 2, 4, 8, 16, 32. `nnn=7`: [HW] +128, [FPGA] **custom displacement** |
| `d` | decrement |
| `s` | special: +1 on even addresses, +31 on odd (PicoDrive and FPGA; svpdoc says +32) |
| `v` | overwrite: DRAM writes only change non-zero nibbles (4bpp transparency) |

Keep the `?` bits at 0.

Common modes:

| Mode | Meaning |
|---|---|
| `0x0800` | ROM, inc 1 (add `aaaaa` for the 64K-word page) |
| `0x0818` | DRAM, inc 1 |
| `0x0C18` | DRAM, inc 1, overwrite |
| `0x4018` | DRAM, cell (special) increment |
| `0x081C` with low word `0x8000` | IRAM word 0 |

Programming sequence:

```
ld ext6, <addr low>
ld ext6, <mode>
ld ext4, -        ; write channel: programs, NO memory access, NO increment [FPGA]
ld -, ext1        ; read channel: programs AND prefetches the first word
```

Rules:

* **Reads are prefetched.** Each read returns the word fetched at the previous access, then fetches the next.
* **Re-write PMC before every blind access.** [FPGA] A second `ld ext4, -` without a new PMC writes 0xFFFF and increments.
* **Every EXT6 access, read or write, flips the PMC half-selector.** Don't read EXT6 casually. `ld -, ext7` resets the selector, but [FPGA] it also overwrites the last PM's custom displacement.
* **Increments only affect address bits 0–19.** [FPGA] There's no carry into bit 20, so streams don't cross 1M-word boundaries.
* **PMC after an access** holds that PM's *next* address. Reading EXT6 then tells you where the last stream got to.

Templates that assemble and are byte-checked: `examples/*.svp`.

---

## 5. 68k ↔ SSP handshake [ALL]

68k side:

```
move.w  #cmd, $A15000      ; WORD write. cmd must be NON-ZERO, or the SSP never sees it [FPGA]
.wait:  btst #0, $A15005   ; SA: SSP has answered (reading clears it)
        beq.s .wait
move.w  $A15000, d0        ; reply
```

SSP side: see `examples/xst_handshake.svp`.

* **Byte writes to 0xA15000 are ignored [FPGA].**
* **Don't rely on 0xA15006 to stop the SSP** (no effect on the Pro). Park the SSP in an IRAM loop before the 68k does anything that conflicts.

---

## 6. VDP DMA from the cart [ALL]

* Set the DMA source **+2 bytes**. This applies to both ROM and DRAM, because the cart latches one word.
* [FPGA Pro] The first word of every DMA is stale. Start one word early, or discard the first VRAM word.
* Run the 68k from work RAM, or `stop`, during heavy SSP work. On the FPGA the 68k always wins the PSRAM, and SSP ROM accesses get restarted.

---

## 7. ROM-size rules (what the project decided)

* **SSP code, and everything the SSP reads, must sit in the first 2MB on every target.** In the FPGA cores, `SVP.vhd` `ROM_A` is 20 bits wide.
* **68k-only data above 2MB, per target:**
  * **MiSTer:** linear 0x200000–0x2FFFFF plus SSF2 bank registers (0xA130F3–FF).
  * **EverDrive Pro, stock mapper:** SD-card overlays through the EverDrive I/O command `CMD_F_FRD_MEM` (needs a test, see `50-open-questions-and-tests.md`).
  * **MegaSD:** untested. Try a "SEGA SSF" header plus the VR serial.
* **No custom FPGA mapper** (project decision).
* **Always keep the serial `GM MK-1229 -00` at ROM byte 0x180.** MiSTer, MegaSD and the Pro detect the SVP by it.
* **Effective-capacity techniques** (`10-…` §C) work everywhere: SSP decompression into DRAM, cell-arrange tiling, overwrite compositing, strided reads, procedural generation.

---

## 8. ROM layout template [ALL]

| 68k bytes | SSP words | Contents |
|---|---|---|
| 0x000000–0x0007FF | 0x000–0x3FF | 68k vectors and header. Not executable by the SSP (IRAM shadows it) |
| 0x000800–0x01F7FF | 0x400–0xFBFF | SSP resident code (entry at word 0x400), plus 68k code/data in any free space |
| 0x01F800–0x01FFFF | 0xFC00–0xFFFF | copy of the internal-ROM dump |
| 0x020000–0x1FFFFF | | 68k code; SSP data and overlays (read via PMs); compressed assets |
| 0x200000+ | | 68k-only data (target-dependent, §7) |
