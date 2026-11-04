# How the FPGA SVP actually works: srg320's core, instruction by instruction

This is a reference for the SVP implementation used by the **Mega EverDrive Pro** (and CORE), **MegaSD** and **MiSTer**. Everything here was read directly from the HDL. Where the core differs from real hardware (svpdoc, the project wiki, PicoDrive), that's called out, because code written against hardware documentation can behave differently on these carts.

Sources (line numbers refer to these exact revisions):

| File | Revision | Role |
|---|---|---|
| `krikzz/MEGA-PRO: fpga/mapper/map_svp/SVP/SSP160x.vhd` | `7064f45` | The DSP CPU (decoder, ALU, registers, RAM banks, stack) |
| `krikzz/MEGA-PRO: fpga/mapper/map_svp/SVP/SVP.vhd` | `7064f45` | Memory controller (PMx/PMC), XST/68k registers, IRAM, fetch sequencer |
| `krikzz/MEGA-PRO: fpga/mapper/map_svp/SVP/SSP160x_PKG.vhd` | `7064f45` | Register numbers, `ModAdj` (pointer modulo) |
| `krikzz/MEGA-PRO: fpga/mapper/map_svp/map_svp_v2.sv` | `7064f45` | Pro wrapper: 68k address decode, PSRAM arbitration, cell arrange |
| `MiSTer-devel/Genesis_MiSTer: rtl/SVP/*.vhd`, `rtl/system.sv` | `adc0c42` | Same core with a different wrapper; differences noted |

`map_svp_v1.sv` is the Mega EverDrive **CORE** wrapper. `SVP_LS.vhd` is an alternate controller in the tree, not used by the Pro project file. MegaSD's wrapper is closed source, but Terraonion credits the same SSP160x core.

Notation: `0x1234` is hex. "Word" means a 16-bit SSP address unit. 68k byte address = SSP word address × 2.

---

## 1. Block diagram

```
 68000 bus ──► map_svp_v2.sv (svp_core)
                 │  0x000000-0x1FFFFF  ROM  ──► mem_dp_svp(rom) ──► PSRAM0  (rom0)
                 │  0x300000-0x3FFFFF  DRAM ──► mem_dp_svp(ram) ──► PSRAM1  (rom1, 128KB used)
                 │     0x39xxxx / 0x3Axxxx = cell-arrange address scramble
                 │  0xA15000-0xA1500F  ──► SVP.vhd I/O (XST, status, "halt")
                 ▼
              SVP.vhd ── fetch sequencer (MAS state machine)
                 │        PMAR[0..4] (write channels), PMAR[5..9] (read channels), PMC
                 │        IRAM 1K×16 (dual-port: SSP fetch + PM access)
                 ▼
              SSP160x.vhd ── CPU: X, Y, A(32), P(32)=X*Y*2, ST, 6-entry stack, PC(16)
                              r0-r7 (8-bit), RAM-A / RAM-B (256×16 each)
```

The 68k ROM port and the SSP ROM port are both **20 address bits** (`map_svp_v2.sv:172`, `SVP.vhd:469`). Both share one PSRAM through `mem_dp_svp`, where the 68k has priority (§8.3).

---

## 2. The execution model (`SVP.vhd`, MAS state machine)

The CPU only advances when `SS = 1`. `SS` is high for exactly one clock per instruction cycle, in state `MAS_EXEC` (`SVP.vhd:166-169`). Every instruction **word** goes through this loop:

```
MAS_IDLE ──(a PM access is pending?)──yes──► MAS_ROM_RD / MAS_DRAM_RD / MAS_DRAM_WR /
   │                                          MAS_IRAM_RD / MAS_IRAM_WR ──► MAS_IDLE
   no
   ├─ PC < 0x0400 ──► MAS_PRAM_RD (1 clk, IRAM) ──► MAS_EXEC
   └─ PC ≥ 0x0400 ──► MAS_PROM_RD (wait for PSRAM ack) ──► MAS_EXEC
MAS_EXEC: CPU executes this word (1 clk), then back to MAS_IDLE
```

Consequences:

* **A PM access blocks the CPU.** The access triggered by an instruction runs in the IDLE slot *after* that instruction, before the next fetch. There's no overlap.
* **Instructions are 1 word = 1 round, 2 words = 2 rounds.** Immediate forms (`ldi`, `OPi A, imm`, `bra`, `call`, `ldi (ri), imm`) take two rounds. Indirect program reads (`((ri))`, `(a)`) also take two rounds: the second round's "fetch" reads the data word from program space at the indirect address (`PA <= IND_ADDR`, `SSP160x.vhd:719`).
* **Fetching from IRAM costs 1 clock; fetching from ROM costs a full PSRAM access**, which `mem_dp_svp` can abort and restart if the 68k wants the bus (§8.3). Hot loops belong in IRAM even more on the FPGA than on hardware.
* **There is no internal ROM.** `IRAM_SEL` is only `PA < 0x0400` (`SVP.vhd:164`). Fetches at 0xFC00–0xFFFF go to cart ROM words 0xFC00–0xFFFF (68k bytes 0x1F800–0x1FFFF). See §9.
* **Reset PC is 0x0400** (`SSP160x.vhd:504`). On hardware the SSP boots through the internal ROM at 0xFC08, which jumps to 0x0400. On the FPGA, nothing in IRAM or IROM runs first.
* **There are no interrupts.** IE is a stored bit only, and no vector logic exists.

---

## 3. CPU registers as implemented

General register numbers (`SSP160x_PKG.vhd`) are the `dddd`/`ssss` fields:

| # | Name | Read gives | Write does |
|---|---|---|---|
| 0 | `-` | 0xFFFF | nothing (destination for blind reads) |
| 1 | `X` | X | X (P updates combinationally) |
| 2 | `Y` | Y | Y (P updates combinationally) |
| 3 | `A` | A[31:16] | A[31:16] (A[15:0] kept), except `ld a, p` (full 32 bits) |
| 4 | `ST` | see below | control bits 0–9 only. Flags are **not** writable |
| 5 | `STACK` | pop | push |
| 6 | `PC` | PC | jump |
| 7 | `P` | P[31:16] | nothing |
| 8–14 | `EXT0`–`EXT6` | external bus (§5) | external bus |
| 15 | `AL` | A[15:0] | A[15:0] |

### ST layout (`SSP160x.vhd:538`)

```
bit: 15 14 13 12 11   10   9    8  7  6   5   4:3 2:0
     N  OV Z  L  GPI0 GPI1 MACS OP IE ST6 ST5 RB  RPL
```

* **N and Z** are the only real flags. They're set from the full 32-bit ALU result on ALU, `mod` and multiply ops.
* **L and OV are cleared on every flag update and never set** (`SSP160x.vhd:407,420`).
* **GPI0/GPI1 are wired to `00`** (`SVP.vhd:170`). Hardware's toggling USR0 doesn't exist here.
* **MACS and OP are stored but have no effect.** There's no saturation, and the product is always ×2.
* **IE reads back as written** (hardware reads 0).
* **RB is stored but ignored**: `r3`/`r7` short addressing is always 0–3 (§4.3).
* **ST5 and ST6** gate the external registers (§5).

### P, the multiplier

`P = signed(X) * signed(Y) * 2` (`SSP160x.vhd:567`). This is combinational, so P is valid the instant X or Y is written.

### Stack

Six entries, **circular, with no overflow detection** (`SSP160x.vhd:488-529`). A seventh push silently overwrites the oldest entry. `call` pushes PC+1 of the address word. `ret` (`ld pc, stack`) pops.

---

## 4. Instruction set as decoded (`SSP160x.vhd:99-265`)

### 4.1 Opcode map

Bits 15:13 select the operation, bits 12:8 the addressing form. ALU group `ooo`: `001` sub, `011` cmp, `100` add, `101` and, `110` or, `111` eor. `000` is ld.

| Encoding (hex high byte) | Form | Assembler (`ssp16asm`) | Core notes |
|---|---|---|---|
| `00` | `ld d, s` | `ld x, y` | EXT↔EXT moves are broken (§5.4) |
| `20 60 80 A0 C0 E0` | `OP A, s` | `add a, x` | `s = P` or `A` → full 32-bit operand |
| `02/03` | `ld d, (ri)` | `ld a, (r0+)` | |
| `22..E3` | `OP A, (ri)` | `add a, (r4)` | |
| `04/05` | `ld (ri), s` | `ld (r0), a` | |
| `06/07,26..E7` | `OP A, adr` / `ld A, adr` | `ld a, A[0x10]` | direct RAM-A/RAM-B address |
| `08` | `ldi d, imm16` | `ld a, 0x1234` | 2 rounds |
| `28..E8` | `OPi A, imm16` | `addi a, 0x1234` | 2 rounds, operand = imm << 16 |
| `0A/0B` | `ld d, ((ri))` | `ld a, ((r0))` | 2 rounds; the RAM word is post-incremented, **ri itself is not modified** even if a modifier is written |
| `2A..EB` | `OP A, ((ri))` | `add a, ((r0))` | same |
| `0C/0D` | `ldi (ri), imm16` | `ld (r0), 0x1234` | 2 rounds |
| `0E/0F` | `ld adr, A` | `ld A[0x10], a` | |
| `12/13` | `ld d, ri` | `ld a, r3` | **r3/r7 readable** |
| `32..F3` | `OP A, ri` | `add a, r0` | |
| `14/15` | `ld ri, s` | `ld r3, a` | **r3/r7 writable** (`1433` decodes) |
| `18 19 1A 1C 1D 1E` | `ldi ri, imm8` | `ld r0, 0x05` | **`1B`/`1F` (`ld r3/r7, imm`) do nothing** |
| `38..F8` | `OPi simm8` | `andi 0x02` | operand = imm8 << 16 |
| `37 / 97 / B7` | `mpys / mpya / mld` | `mpya (r4), (r0)` | X ← RAM-A, Y ← RAM-B, A ±= P (old P) |
| `48/49` | `call cond, addr` | | 2 rounds |
| `4A` | `ld d, (a)` | `ld x, (a)` | 2 rounds, program read at A[31:16] |
| `4C/4D` | `bra cond, addr` | | 2 rounds |
| `90` | `mod cond, op` | `mod always, shr` | **only with f = 0** (§4.2) |
| `9405` | `setie` | `mod f, setie` | the only working flag op |
| everything else | | | **NOP** (`INST_NON`) |

### 4.2 Conditions and `mod`: the biggest behaviour gaps

`COND` looks only at `IR[6:4]` (`SSP160x.vhd:427-434`):

```
000 -> true (always)        101 -> Z == f        111 -> N == f        anything else -> FALSE
```

Encodings from `ssp16asm`, checked by assembling each one, and what the core does with them:

| Source | Word(s) | Core behaviour |
|---|---|---|
| `bra z=1, x` / `bra n=0, x` / `call …` | `4D50` / `4C70` | correct |
| `bra l=0, x` | `4C40 xxxx` | **never taken** (condition 100 → false) |
| `bra ov=1, x` | `4D60 xxxx` | **never taken** |
| `bra gpi0=1, x` | `4D20 xxxx` | **never taken** |
| `bra diof=1, x` | `4D80 xxxx` | **always taken**: bit 7 is ignored, so the condition decodes as `000` |
| `mod always, shr/shl/neg/abs` | `9002/9003/9006/9007` | correct (shr is arithmetic) |
| `mod always, ror/rol/inc/dec` | `9000/9001/9004/9005` | **NOP** |
| `mod z=0, shr` / `mod n=0, neg` | `9052` / `9076` | correct |
| `mod z=1, shr` / `mod n=1, neg` | `9152` / `9176` | **wrong**: `0x91` decodes as `ADD A, 0`, so A is unchanged and N/Z are recomputed |
| `mod f, setie` | `9405` | sets IE (nothing uses it) |
| `mod f, resie/setop/resop/setl/resl/set/res` | `9404/9409/9408/9403/9402/940F/940E` | **NOP** |

Example from this repo: `samples/SVP_speed_test/tests.svp` uses `bra l=0, …` for its carry. On the FPGA that branch is never taken, so `addi 0x01` always runs.

**Portable rule:** branch only on `always`, `z=`, `n=`. Use `mod` only with `always`, `z=0`, `n=0`. Build carry from compares, as VR does.

### 4.3 Pointer registers and modifiers

The 4-bit field `mmpp` (`IR[3:0]` for RAM-A, `IR[7:4]` for RAM-B in `mpya`-type ops) is decoded in `SSP160x.vhd:569-660`:

| `mm` | r0–r2 / r4–r6 | r3 / r7 |
|---|---|---|
| `00` | `(rX)`, no change | `(r3\|00)` → address 0 |
| `01` | `(rX+!)`: +1, full 8-bit wrap, RPL ignored | `\|01` → 1 |
| `10` | `(rX-)`: −1 modulo 2^RPL (RPL = 0 → plain −1) | `\|10` → 2 |
| `11` | `(rX+)`: +1 modulo 2^RPL | `\|11` → 3 |

The modulo is computed by `ModAdj` (`SSP160x_PKG.vhd`): the bits above the mask are kept and the low bits wrap. RPL = 4 gives a 16-entry ring.

**r3/r7 as scratch storage:** `ld r3, a` / `ld a, r3` work (8 bits), but they never affect `(r3|nn)` addressing. That gives two free byte registers. `ld r3, 0x05` (immediate) is a NOP.

### 4.4 ALU details

* All ALU ops work on the full 32-bit A. Immediate and 16-bit operands are placed in the high word (`operand << 16`). So `andi 0x0002` also clears A[15:0], and `cmp`/`sub` compare all 32 bits.
* `ld a, s` (16-bit source) writes A[31:16] only, and **does not touch flags**. The only way to set the low word is `ld al, s`.
* `ld a, p` copies all 32 bits of P.
* `mld` clears A, so Z=1 and N=0 afterwards.

---

## 5. External registers (EXT0–EXT7) exactly as decoded

`EA` = the ext number. `R_NW = 0` when the ext register is the **destination** (`SSP160x.vhd:721-729`). A **blind read** is `ld -, extN`; a **blind write** is `ld extN, -`. srg320's comment admits the blind-access definition is a guess.

### 5.1 Read and write behaviour (`SVP.vhd:204-322`)

| Reg | VR name | Read | Write | Usable as PM when |
|---|---|---|---|---|
| EXT0 | PM0 | ST5=0: `{14'b0, CA, SA}` and **clears CA**. ST5=1: PM0 read data | ST5=1: PM0 write | **ST5 = 1** |
| EXT1 | PM1 | ST5=0: 0. ST5=1: PM1 data | ST5=1: PM1 write | **ST5 = 1** |
| EXT2 | PM2 | ST6=0: 0. ST6=1: PM2 data | ST6=1: PM2 write | **ST6 = 1** |
| EXT3 | XST | ST6=0: XST. ST6=1: PM3 data | ST6=0: XST ← value, **sets SA**. ST6=1: PM3 write | **ST6 = 1** |
| EXT4 | PM4 | PM4 data | PM4 write | always |
| EXT5 | — | 0 | ignored | **never** (hardware: write-only PM, per the wiki) |
| EXT6 | PMC | PMC_SEL=0: PMC[15:0]. PMC_SEL=1: PMC[15:0] with the nibbles of each byte swapped. **Every access toggles PMC_SEL** | PMC_SEL=0 → PMC[15:0] (address), 1 → PMC[31:16] (mode). Toggles PMC_SEL and clears PMAR_SET | — |
| EXT7 (= `AL`, reg 15) | — | CPU side: returns **A[15:0]**. Bus side, blind read only: `CD[last PMAR] ← PMC[15:0]`, PMC_SEL ← 0 | CPU side: **A[15:0] ← value**. Bus side: nothing | — |

Hardware differences:

* On hardware, ST5 alone appears to enable EXT1 and EXT2 (wiki). **Here EXT2 and EXT3 need ST6.** Always use `ld st, 0x0060` before using PM0–PM3 as memory ports.
* EXT2 reads with the ST bits clear return 0, not hardware's "random" value.

> **EXT7 is AL.** Register number 15 is both "EXT7" (bus address `EA = 7`) and the accumulator low word. `ssp16asm` has no `al` name and calls it `ext7`, so `ld ext7, 0x0000` (used throughout `samples/`) **clears A[15:0]**, and `ld a, ext7` reads A[15:0]. `ld -, ext7` is the "dummy-programmed" blind access that svpdoc describes for AL (it resets PMC to "waiting for address"). On this core it also loads the custom displacement.

### 5.2 The PMAR record

There are ten PMARs: index 0–4 are the write channels of EXT0–EXT4, and 5–9 their read channels (`SVP.vhd:54-61`). Each one holds:

* `MA[31:0]` = `{mode word, address word}`
* `DATA[15:0]` = the last word read (prefetched) or written
* `CD[15:0]` = custom displacement

The mode word is `MA[31:16]`, laid out `d s nnn v ?? ??? aaaaa`:

| Bits (of MA) | Field | Implemented as (`GetNextMA`, `SVP.vhd:107-138`) |
|---|---|---|
| 31 | `d` | subtract the displacement (ignored when `s=1`) |
| 30 | `s` | special: +1 if the address is even, **+31** if odd |
| 29:27 | `nnn` | 0 → no change, 1..6 → +1,2,4,8,16,32, **7 → `CD`** (hardware: 128) |
| 26 | `v` | overwrite (DRAM writes only, §5.5) |
| 25:21 | `?` | **ignored entirely** (hardware: programming fails, per the wiki) |
| 20:16 | `aaaaa` | address bits 16–20 |

Only `MA[19:0]` is incremented. **The carry never reaches bit 20.** A ROM stream at 0x0FFFFF wraps to 0x000000. A DRAM stream at 0x18FFFF steps to 0x190000, which isn't decoded (§5.3), so further accesses silently do nothing.

### 5.3 What a PM address can reach (`SVP.vhd:391-418`)

| Address (21-bit word) | Target | Read | Write |
|---|---|---|---|
| `0x000000–0x0FFFFF` (bit 20 = 0) | ROM | yes | **silently ignored** |
| `0x180000–0x18FFFF` (`[20:16] = 11000`) | DRAM, 64K words | yes | yes (overwrite supported) |
| `0x1C8000–0x1CFFFF` (`[20:15] = 111001`) | IRAM, 1K words, mirrored ×32 | yes | yes |
| anything else | — | **no access**; DATA keeps its old value | ignored |

### 5.4 Programming sequence, cycle by cycle

```
        ld ext6, 0x0000     ; PMC[15:0] = 0x0000       PMC_SEL 0->1, PMAR_SET=0
        ld ext6, 0x0818     ; PMC[31:16] = 0x0818      PMC_SEL 1->0, PMAR_SET=0
        ld ext4, -          ; blind WRITE, PMAR_SET=0  -> PMAR[4].MA = 0x08180000
                            ;   PMAR_SET=1. No memory access. No increment.
        ld ext4, a          ; write: DRAM[0x0000] <- A.hi, then MA += 1
        ld ext4, x          ; write: DRAM[0x0001] <- X,    then MA += 1
```

For a read channel:

```
        ld ext6, 0x0400
        ld ext6, 0x0800     ; ROM 0x000400, inc 1
        ; (ld st, 0x0060 beforehand so EXT0-EXT3 act as PMs)
        ld -, ext1          ; blind READ (needs ST5=1): PMAR[6].MA = 0x08000400,
                            ;   AND the memory controller immediately PREFETCHES
                            ;   ROM[0x0400] into PMAR[6].DATA
        ld a, ext1          ; returns DATA (= ROM[0x0400]), MA becomes 0x0401,
                            ;   prefetches ROM[0x0401]
```

Things that follow from this code and nowhere else:

1. **Reads are prefetched.** The value you get was read at the *previous* access (or at programming time). If something else writes that memory in between, you get the stale word.
2. **Blind writes do not increment the address.** The comment in `samples/SVP_basic_gfx/svp_gfx.svp:59-71` says they do ("program 0x180FFF to land at 0x181000"). In the core, the first write lands at 0x180FFF. That sample's −1 actually lines up with the **DMA one-word latency** (§8.2), not with the PM.
3. **A second blind access without rewriting PMC is a normal access.** If PMAR_SET is already 1, `ld ext4, -` writes **0xFFFF** (the value of `-`) to memory and increments. `ld -, ext1` reads and discards one word (a "skip").
4. **Reading EXT6 toggles PMC_SEL.** Any stray `ld a, ext6` misaligns the next two-write PMC setup. To resynchronise, use `ld -, ext7`, which resets PMC_SEL to 0, but it also overwrites `CD` of the last-used PMAR.
5. **After every PM access, `PMC ← that PMAR's next MA`.** `ld a, ext6` therefore returns the next address of the last stream you touched, which is handy for "how far did I get". Read it while PMC_SEL = 0.
6. **EXT↔EXT moves are broken.** `ld ext4, ext1` (`00C9`): EA comes from the *source* field, but R/W comes from the *destination*. The core performs a **write to EXT1's write channel** with EXT1's read data, and PM4 is never touched. Always go through a register: `ld a, ext1` then `ld ext4, a`.
7. **ALU ops can read a PM directly.** `add a, ext1` reads and advances the PM1 stream in one instruction. That's useful for checksums and accumulators.

### 5.5 Overwrite mode (`v`, MA bit 26)

`OverWrite()` (`SVP.vhd:140-158`) writes back the old DRAM word with each nibble replaced only where the new nibble is non-zero. The memory controller supplies the old word because `DRAM_DO` is computed against `DRAM_DI`. This is transparent 4bpp pixel compositing. It applies to DRAM only, not IRAM.

### 5.6 Custom displacement: arbitrary strides (FPGA-specific)

```
        ; Program PM1 for reads from ROM 0x010000 with nnn=7 (custom stride)
        ld ext6, 0x0000
        ld ext6, 0x3801     ; nnn=111 (bits 13:11 of the mode word), aaaaa=0x01
        ld -, ext1          ; program + prefetch; PMAR_NUM now = 6 (PM1 read)
        ld ext6, 0x0140     ; PMC[15:0] = 320 (stride); PMC_SEL -> 1
        ld -, ext7          ; CD[6] = 0x0140; PMC_SEL -> 0
        ld a, ext1          ; ROM[0x10000]; next access ROM[0x10140], ...
```

Caveat: `ld ext6, …` clears PMAR_SET. The next blind access to any PM would re-program it with the half-written PMC. Do the EXT7 step straight after programming, before any other blind access. The displacement can be up to 0xFFFF, or used as a negative via `d`. Real hardware's `nnn=7` is a fixed 128, so code using CD is FPGA-only.

---

## 6. IRAM

1K×16 dual-port (`SVP.vhd:476-488`):

* port A is the PM side, address `MEM_ADDR[9:0]`;
* port B is CPU fetch, `PA[9:0]` when `PA < 0x400`.

Writes go through any write PM aimed at 0x1C8000. IRAM reads through a PM work too (two clocks).

The standard loader, VR's `0xC8DA` pattern:

```
        ld ext6, 0x8000
        ld ext6, 0x081C     ; IRAM word 0, inc 1
        ld ext4, -
copy:   ld ext4, ((r0))     ; ROM (program space) -> IRAM, RAM-A pointer word auto-increments
        subi 0x01
        bra n=0, @copy
```

`((r0))` reads **program space**, which is limited to PC-reachable addresses (IRAM or ROM words 0–0xFFFF). To load overlays from anywhere in the 2MB, use a PM1/PM2 read stream instead (`ld a, ext1` then `ld ext4, a`).

---

## 7. The 68k-side registers (`SVP.vhd:197-276`)

The wrapper only forwards 0xA15000–0xA1500F (`map_svp_v2.sv:113`). Registers decode `A[3:1]`:

| 68k addr | Write | Read |
|---|---|---|
| 0xA15000 / 0xA15002 | XST ← data. **CA ← 1 only if data ≠ 0** | XST (last value written by either side) |
| 0xA15004 | — | `{14'b0, CA, SA}`, then **SA ← 0** |
| 0xA15006 | HALT ← (data[3:0] == 0xA) | 0 |
| 0xA15008–0xA1500E | ignored | **0** (hardware: 0xFFFF) |

Gotchas:

* **Writing 0x0000 to XST is not signalled.** The SSP's `andi 0x2` poll never sees it. Always send non-zero commands.
* **Byte writes:** the core latches on `BUS_LWR_N` (the low-byte strobe, `we_lo`). `move.b` to the even address 0xA15000 only strobes the high byte and is **ignored**. Use `move.w`.
* **HALT does nothing on the Pro**: it's latched but never used (`SSP_SS <= EN and not SSP_WAIT`, `SVP.vhd:169`). On MiSTer it stops the SSP (`SSP_SS <= … and not HALT`). VR writes 0x0A around DRAM DMA, which is harmless on the Pro only because the wrapper gives the 68k priority anyway. Don't rely on 0xA15006 to freeze the DSP. Park it in an IRAM loop instead.
* **Detecting the FPGA:** reading 0xA15008 returns 0 here and 0xFFFF on real hardware (svpdoc §3).

Handshake truth table:

| Event | CA | SA |
|---|---|---|
| 68k writes non-zero to XST | → 1 | |
| SSP reads EXT0 with ST5=0 | → 0 | |
| SSP writes EXT3 with ST6=0 | | → 1 |
| 68k reads 0xA15004 | | → 0 |

So the SSP polls bit 1 of EXT0 for "68k spoke" and the 68k polls bit 0 of 0xA15004 for "SSP answered", which is exactly what the samples do.

---

## 8. The Pro wrapper (`map_svp_v2.sv`)

### 8.1 68k address decode

| 68k range | Pro behaviour | Real hardware (svpdoc) |
|---|---|---|
| 0x000000–0x1FFFFF | ROM, `abus[20:1]` | ROM |
| 0x200000–0x2FFFFF | drives the bus with the **stale DRAM read register** | SSP internal bus echo |
| 0x300000–0x3FFFFF | DRAM `abus[16:1]`, so 128KB mirrored ×8. Reads **and writes** | DRAM + 3 mirrors to 0x37FFFF, 0x380000 echo, 0x3B0000+ = 0xFFFF |
| 0x390000–0x39FFFF | cell arrange 1: `{0, a[15:13], a[6:2], a[12:7], a[1]}`. **Writes are remapped too** | cell arrange 1 (read side documented) |
| 0x3A0000–0x3AFFFF | cell arrange 2: `{0, a[15:12], a[5:2], a[11:6], a[1]}`. Writes remapped too | cell arrange 2 |
| 0xA15000–0xA1500F | SVP registers, DTACK from the wrapper | same |

Writing *through* the cell-arrange windows is an FPGA capability worth using: the 68k can deposit tile-ordered data and have the SSP see it linear, or the reverse. **The only DRAM the cell windows reach is the first 64KB.**

### 8.2 DMA latency emulation

`mem_do_a = !mem_as_a ? mem_do_a_cpu : mem_do_a_dma`. At the start of every access, `mem_do_a_dma <= mem_do_a_cpu` (`map_svp_v2.sv:280,335`). During bus cycles without the 68k's /AS (VDP DMA), each word returned is **the data of the previous access**. That reproduces the hardware rule "set the DMA source 2 bytes higher" for both ROM and DRAM.

The very first word of a DMA is whatever the previous access returned. **Start each DMA one word early, or treat its first VRAM word as junk.**

MiSTer does it differently: it subtracts one word from the VDP's source address (`system.sv: MBUS_A - 1'd1`), which is exact.

### 8.3 Arbitration between the 68k and the SSP

`mem_dp_svp` gives port A (the 68k) priority. If a 68k access arrives while an SSP access is more than one clock from completing, **the SSP access is aborted and restarted** (`map_svp_v2.sv:316`). A 68k main loop that runs from cart ROM therefore slows any SSP code that runs from ROM or streams ROM through PMs.

Remedies:

* run the 68k from work RAM, or `stop`, while the SSP works;
* put hot SSP code in IRAM;
* stream DRAM rather than ROM where possible (DRAM is a separate PSRAM, `rom1`).

---

## 9. FPGA vs real hardware: a portability checklist

| # | Behaviour | FPGA (this core) | Real SVP | What to do |
|---|---|---|---|---|
| 1 | Internal ROM 0xFC00–0xFFFF | **absent**: fetches cart ROM 0x1F800–0x1FFFF | 1K-word IROM (boot, math, sine table) | Put a byte-exact copy of the IROM dump at ROM 0x1F800 so both behave identically. Don't depend on IROM routines being "free" on the FPGA. |
| 2 | Reset | PC = 0x0400 | IROM at 0xFC08, then 0x0400 | none |
| 3 | Interrupts | none | IE0–IE2 | poll instead |
| 4 | L / OV flags, `l=`/`ov=`/`gpi=` conditions | always false | real | branch on Z/N only |
| 5 | `diof` condition | always true | real | don't use |
| 6 | `mod` with f=1 (`z=1`, `n=1`) | ADD A,0 (NOP + flag refresh) | works | invert the logic, use f=0 |
| 7 | `ror/rol/inc/dec`, `mod f` ops except `setie` | NOP | work (`setop` saturates) | `addi`/`subi`, shifts via `shl`/`shr` |
| 8 | MACS / OP bits | no effect | affect P shift / saturation | assume ×2 product, no saturation |
| 9 | RB bits (r3/r7 range) | ignored (0–3 only) | extends to 16 slots | stay within `\|00`–`\|11` |
| 10 | `ld r3/r7, imm` | NOP | — | load via a register |
| 11 | EXT2/EXT3 as PMs | need ST6 | ST5 reported sufficient | use ST = 0x60 |
| 12 | EXT5 | dead | write-only PM | don't use |
| 13 | EXT↔EXT move | corrupt | ? | go through A/X/Y |
| 14 | PM mode `?` bits 25:21 | ignored | programming fails | keep 0 |
| 15 | `nnn = 7` | custom displacement CD | +128 | FPGA-only feature |
| 16 | Increment carry into bit 20 | no | yes (svpdoc) | don't cross 1M-word boundaries |
| 17 | Blind write increments address | no | disputed | program the exact first address |
| 18 | Second blind write | writes 0xFFFF and increments | ? | always rewrite PMC first |
| 19 | 0xA15006 halt | no effect on the Pro, works on MiSTer | probably halts | park the SSP in IRAM |
| 20 | 0xA15008–E reads | 0 | 0xFFFF | can be used to detect the FPGA |
| 21 | 0x200000–0x2FFFFF reads | stale DRAM latch | SSP bus echo | don't use |
| 22 | DMA first word | stale (Pro) / exact (MiSTer) | source−2 | start DMA one word early |
| 23 | Writes via 0x390000/0x3A0000 | remapped | unknown | FPGA-only convenience |
| 24 | Byte write to 0xA15000 | ignored | ? | word writes only |
| 25 | XST write of 0 | no CA flag | ? | non-zero commands |

---

## 10. Where the 2MB limit lives (exact lines)

1. `SVP.vhd:24` `ROM_A : out std_logic_vector(20 downto 1)`, and `:393`: only `MEM_ADDR(20) = '0'` reaches ROM.
2. `SVP.vhd:469` `ROM_A <= MEM_ADDR(19 downto 0) … else "0000" & SSP_PA`.
3. `map_svp_v2.sv:172` `rom_oe_smd = abus[23:21] == 0`, and `mem_addr_a(abus[20:1])` / `mem_addr_b(svp_rom_a[20:1])`.

The PSRAM ports are already `[21:1]` and the cart has 2×8MB. Widening those three places, plus adding a bank register, would remove the limit. **A custom mapper is out of scope for this project** (see `30-flashcarts-rom-size.md` §3B). These lines are documented so the limit is understood, not changed.

Also on the SSP side, `GetNextMA` only increments 20 bits. That's harmless for banking but means a stream never crosses into a bank by itself.
