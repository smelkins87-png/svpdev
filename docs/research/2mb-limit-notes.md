# Getting past the 2MB limit on the SVP: research notes

These notes cover a full pass over the repository: the boards, the netlists, the die photos, the samples, `ssp16asm`/`svptool`, the project wiki, notaz's PicoDrive SVP emulation, and notaz's `svpdoc.txt` (2014-09-23 revision, cited below as **svpdoc**). The goal was to list every quirk, bug and oversight that bears on the 2MB ROM ceiling, and every route around it that needs **no additional hardware**.

Confidence tags used below:

* **[FACT]**: backed by evidence in this repo, the wiki, or the emulator source. The source is cited.
* **[INFERENCE]**: follows from the facts but hasn't been checked on hardware.
* **[SPECULATION]**: a lead worth testing with nothing yet to back it.

---

## 0. Bottom line

1. **On a stock cart the physical ROM bus is 2MB wide.** The ROM footprint routes only A0–A19 (word address). The SVP die has exactly 38 pads labelled "ROM", which is A0–A19 + D0–D15 + /CE + /OE. Software alone cannot make one 2MB chip return more than 2MB of distinct bits.
2. **The address space is wider than the ROM.** Both the 68k and the SSP side decode a 4MB cart space (the SSP's PM registers take 5 high-address bits, 21-bit word addresses). The window **0x200000–0x2FFFFF (68k bytes) = 0x100000–0x17FFFF (SSP words)** has been looked at only from the 68k side. svpdoc reports that 68k reads there return "data from internal bus (last word read by SSP160x)", so there's no ROM behind it as seen from the 68k. **What the SSP gets when it aims a PM at that window is still unknown** (svpdoc marks the SSP column "?"), and so is whether any die pad toggles when it does. That's the best remaining "indirect" lead. It can be tested with a pure software ROM on the existing EPROM board, and a probe is included below (§A1).
3. **The practical ways past 2MB are about effective capacity.** The SVP is a 23 MHz DSP sitting next to 128KB of DRAM, with hardware address-remapping views and a nibble-masked write mode. Used as a streaming decompressor, asset transformer and procedural generator, it can make 2MB of ROM do the work of several times that (§C).
4. **Space is being wasted inside the 2MB right now.** The sample ROM layout zero-pads ~124KB that doesn't need padding. Two toolchain bugs also make it easy to silently lose code or data (§B, §D).

---

## 1. Where the 2MB limit actually comes from (evidence)

| Layer | Evidence | Limit |
|---|---|---|
| ROM chip footprint | `boards/svp-adapter-board/svp-adapter-board.net:131-190`: nets A0–A19 only. `:251-254`: ROM pins **1 and 44 are unconnected** (`Net-(U2-Pad1)`, `Net-(U2-Pad44)`). | 2^20 words = 2MB |
| Die pinout | `img/decap/06_pinout.jpg`: ROM-labelled pads are 48–56, 59–67, 69–71, 73, 76–79 (right edge) and 82–91, 96–97 (top edge). That's **38 pads** = 20 addr + 16 data + /CE + /OE. | 2MB |
| EPROM board | `boards/svp-eprom-board/README.md`: 27C160 (16Mbit). | 2MB |
| SSP memory controller (emulated) | PicoDrive `ssp16.c` `pm_io()`: ROM read uses `addr \| ((mode & 0xf) << 16)`, so **only 4 of the 5 high-address bits** reach the ROM. | 2MB |
| SSP program space | 16-bit PC. Only ROM words 0x0400–0xFBFF are executable from ROM (wiki "Timings"). | **128KB of executable ROM code** (separate, smaller limit) |
| 68k map | svpdoc §3: `000000-1fffff` ROM; `200000-2fffff` unused, reads return the SSP's internal bus; `300000-31ffff` DRAM; `320000-37ffff` **3 mirrors of DRAM**; `380000-38ffff` unused (internal bus); `390000`/`3a0000` cell arrange; `3b0000-3fffff` reads 0xFFFF. | 2MB ROM and 128KB DRAM in a 4MB cart window |
| Assembler | `tools/ssp16asm/src/assembler/assembly.rs:330,391`: `ORG` takes a 16-bit word value, so code and data can only be ORG'd into the first 128KB. The output buffer is 4MB (`:323`). | Toolchain-only |

The same table shows that **the address-generating logic (PM mode word, 68k decoder) covers 4MB, but the chip pins and the ROM cover 2MB.** That gap is where the unknowns are.

### 1.1 The SSP external space is the 68k cart space, in words [FACT]

PM mode word layout (wiki "External registers behaviour", notaz naming): `d s nnn v ?? ??? aaaaa`

* `aaaaa` (bits 4–0) = word-address bits 16–20, so the space is 21 bits = 2M words = **4MB**.
* ROM: `0x00–0x0F`. DRAM: `0x18` (word 0x180000 ×2 = 68k **0x300000**). IRAM: `0x1C` + low `0x8000` (word 0x1C8000 ×2 = 68k **0x390000**).

The SSP map is the 68k cart map divided by two, so **SSP word 0x100000–0x17FFFF ≡ 68k 0x200000–0x2FFFFF**. svpdoc confirms "the external memory space seems to match the one visible by 68k" and that "when auto-increment is performed, it affects all 21 address bits", so a ROM-read PM that increments past word 0xFFFFF carries straight into this window. A gate array that carries 5 high-address bits and decodes a 4MB space while leaving a 1MB hole is a strong hint that the hole was *reserved for something*: a second ROM, an SRAM, or the A20 line of a 32Mbit ROM. On standard Sega carts, 0x200000 is where SRAM or the upper half of a 4MB ROM lives. [INFERENCE]

---

## A. Unexplored address space and undocumented behaviour (testable in software on the existing hardware)

### A1. SSP PM reads with address bit 20 set (`mode` low byte 0x10–0x17) [SPECULATION, top priority]

* **What:** Program a PM for ROM-style reads with `aaaaa = 0x10..0x17` (e.g. mode `0x0810`) and read.
* **Why it matters:** The emulator masks with `& 0xf` (`ssp16.c pm_io`, `(mode & 0xfff0) == 0x0800` branch) and logs any other mode as "unhandled" returning 0. svpdoc only covers the 68k view of this window (internal-bus echo, §A3). **Nobody has published what the SSP gets back.** Possible outcomes:
  * a mirror of ROM 0–2MB (A20 ignored): rules this out cleanly;
  * internal-bus echo / last value, the same as the 68k sees: the most likely result given svpdoc, and rules it out;
  * **different data**, or a pad toggling: this would mean the gate array drives a line that the PCB doesn't route (see A4), i.e. a hidden A20 or a second /CE.
* **How to test:** Use the probe below with the 68k side sending `0x0810`, `0x0811`, … `0x0817` through XST and dumping DRAM 0x300000 onwards. Compare against the same read with `0x0800`. The ROM image should place distinct marker words at byte 0x000000 and at every 128KB boundary, so a mirror shows up immediately. This snippet assembles cleanly with the current `ssp16asm`. The bytes were checked: `ld -, ext1` → `0009`, matching VR's own test code.

```
# Probe the "unused" SSP external windows: word 0x100000-0x17FFFF and 0x190000-0x1FFFFF
org 400
probe_init:
                        ld st, 0x0000
wait_md:                ld a, ext0
                        andi 0x2
                        bra z=1, @wait_md
                        ld x, ext3                      # MD sends the PM mode word to probe (e.g. 0x0810)
                        ld st, 0x0060                   # UP0|UP1: ext1/ext2 act as PMs
                        ld ext6, 0x0000                 # PM4 -> DRAM word 0x180000 (68k 0x300000), inc 1
                        ld ext6, 0x0818
                        ld ext4, -
                        ld ext6, 0x0000                 # PM1 read: address low = 0x0000
                        ld ext6, x                      # mode/high = value sent by 68k
                        ld -, ext1                      # blind read programs PM1
                        ld y, 0x0010
probe_loop:             ld a, ext1
                        ld ext4, a
                        ld a, y
                        subi 0x01
                        ld y, a
                        bra z=0, @probe_loop
                        ld st, 0x0000
                        ld ext3, 0x4F4B                 # "OK"
                        bra always, @wait_md
```

(Remember the blind-write increment quirk noted in `samples/SVP_basic_gfx/svp_gfx.svp:59-71`: the first DRAM word written may land one word later than programmed.)

### A2. DRAM capacity and the 0x19–0x1F windows [mostly answered: FACT for the 68k side, SPECULATION for the SSP side]

* **What:** The emulator accepts DRAM only when `(mode & 0x43ff) == 0x0018`. Modes `0x19, 0x1A, 0x1B, 0x1D, 0x1E, 0x1F` (word 0x190000–0x1FFFFF = 68k 0x320000–0x3FFFFF) are untested.
* **Update from svpdoc:** 68k 0x320000–0x37FFFF is **"3 mirrors of DRAM"**. The DRAM decode ignores the address bits above 128KB, which is strong evidence that **there is no hidden DRAM**, at least as seen from the 68k. This lead drops to low priority. The SSP-side modes `0x19`–`0x1B` are almost certainly the same mirrors. `0x1D`–`0x1F` (68k 0x3A0000+) are still worth one quick read.
* **Why it mattered:** The 68k sees only 128KB of DRAM (0x300000–0x31FFFF). If the physical DRAM part on the VR board is larger than 64K×16, or if the SVP has a further DRAM address pad (see the "?" pads 124 and 146–149 next to the DRAM/"SRAM" group on the die photo), there could be **hidden working RAM**. More RAM doesn't add ROM, but it enlarges the decompression cache and makes the §C techniques far stronger.
* **How to test:**
  1. Read the DRAM part number off the VR PCB and look up its organisation. This is a zero-cost check that nobody has written down in this repo.
  2. On the SSP: write a distinct pattern via mode `0x0818` at word 0x180000, then write a different pattern via `0x0819` (and up), then read both back. If the patterns alias, the window is a mirror. If both survive, there's extra RAM.
  3. On the 68k: write and read back 0x320000–0x38FFFF.
  4. On the 68k: write 0x300000 and check whether 0x320000 changes (mirror test).
* **Note:** the left-edge "SRAM" group on the die has about 20 labelled pads. That's too few for 16 data + multiplexed address + RAS/CAS/WE/OE unless data or address lines are shared with another bus. **How the DRAM is wired to the SVP is itself undocumented** and worth tracing on the VR PCB with a continuity meter (a measurement, not added hardware).

### A3. 68k reads of 0x200000–0x2FFFFF and 0x320000–0x38FFFF [answered by svpdoc: FACT, not hardware-verified here]

* Both are inside the 4MB /CE0 cartridge region the SVP sits on. The MD generates DTACK itself for this range, so reads won't hang. Contrast `samples/README.md`, which notes that hitting 0xA15000 without an SVP freezes a real console.
* svpdoc §3 already reports the answer:
  * 0x200000–0x2FFFFF and 0x380000–0x38FFFF: "reads seem to return data from internal bus (last word read by SSP160x). Writes probably have no effect."
  * 0x320000–0x37FFFF: three mirrors of DRAM.
  * 0x3B0000–0x3FFFFF: reads 0xFFFF, writes have no effect.
* So **there's no extra ROM visible from the 68k there.** Still worth a 10-minute confirmation dump with `SVP_mem_reader`'s display code, because svpdoc says "seem to".
* **New quirk this exposes (a side channel):** the 68k can see *the last word the SSP read* through those windows. That's a zero-setup way for the 68k to peek at what the SSP is fetching, useful for debugging and synchronising a streaming decompressor (§C1) without spending XST handshakes. It doesn't add capacity.

### A4. Unlabelled die pads next to the ROM bus [SPECULATION]

* `img/decap/06_pinout.jpg` marks pads **72, 74, 75** with "?". They sit *between* ROM-labelled pads 71/73/76. Pads **98 and 100** sit beside ROM pads 96–97, and **109–117** sit between the CLK pads and the SRAM group.
* A gate-array output that sits in the middle of the ROM address group and isn't bonded to the ROM is the classic signature of an **unused high address line or a second chip-select**.
* **How to test without new hardware:** while running the A1 probe in a tight loop, put the scope used for `img/svp_oscope.jpg` on the cart-PCB traces or pads for those pins, and look for activity that tracks bit 20 of the requested address.
* **Caveat:** if such a line exists, using it to get *more ROM* still requires a bigger ROM chip and one wire (adapter board pin 1, see F1). Finding it is software. Using it isn't strictly "no additional hardware", but it would be a cheap mod on the existing EPROM board (27C322 / 4MB parts).

### A5. The "?" bits of the PM mode word [SPECULATION]

* Bits 9–5 (`?? ???`) are undocumented. The wiki ("External registers behaviour", EXT6 section) reports that setting them *one at a time* makes PM programming fail, with the register keeping its previous programming. **Combinations are untested**, and so is using them on *read* PMs pointed at ROM.
* Mode `0xB800` is explicitly flagged by the emulator (`(mode & 0xb800) == 0xb800` → "FIXME"), which means VR or test code touched it.
* A failed-programming response to a single bit could mean "this bit selects a different target that needs other bits too", e.g. a bank or chip select. Worth a systematic sweep: all 32 values of bits 9–5, × ROM/DRAM targets, × read/write, recording what comes back.

### A6. Mapper or bank registers on the 68k side [SPECULATION]

* The SVP register block is 0xA15000–0xA1500F (`samples/*/svp.asm`: XST 0xA15000/2, status 0xA15004, 0xA15006 "halt?", 0xA15008 unknown "interrupt" pulse). svpdoc lists **0xA1500A, 0xA1500C and 0xA1500E as "?" with a reset value of 0xFFFF**, so three registers have no known function. A bank or mode register would most plausibly live there. Probe them first: write each one, then re-read ROM, DRAM and the 0x200000 window from both CPUs. PicoDrive decodes only `(a & ~0x0f) == 0xa15000`, so **0xA15010–0xA150FF and the odd offsets are unmapped in emulation and unprobed on hardware.**
* **0xA130F1–0xA130FF** (the standard Sega /TIME bank-register window used by SSF2-style mappers) has never been tried on the VR cart. If the SVP latches anything there, it could be a bank select. Test: write values, then re-read ROM at 0x080000+ from both the 68k and the SSP.
* 0xA15006: svpdoc says VR "writes 0xa" before DMA from DRAM and 0 afterwards, "probably done to prevent SVP accessing DRAM". Fully map what each bit does, because halting the SSP cleanly is required for safe DRAM handoff in §C1.

### A7. Undocumented external-register behaviours [FACT from wiki, uses SPECULATION]

From the wiki's "External registers behaviour":

* **EXT1 writes with UP0/UP1 clear "serve another function (still unknown)".**
* **EXT2 reads with UP0/UP1 clear return "random values".** Possibly a bus/open-bus read, which might expose whatever is on an internal bus. Log many reads and check for correlation with ROM addresses being fetched, or with DRAM contents.
* **EXT5 is a write-only PM** (not used by VR at all).
* **XST "can be programmed as PMAR"** (notaz, only seen in test-mode code), and EXT0 acts as a PM with ST=0x20 (VR INT2 handler, wiki "Virtua Racing test code").
* **UP1 alone (ST=0x40) changes nothing observed.** Its purpose is unknown, and VR never uses it alone.
* **USR0 toggles on every ST read.** USR1 has never been seen active.

That gives up to **six simultaneously programmed memory streams** (PM0, PM1, PM2, XST, PM4, EXT5) with no reprogramming overhead, which matters a lot for the multi-stream decompressors in §C1. The unknown EXT1/EXT2 behaviours could also turn out to be extra address or bank control.

---

## B. Space currently wasted inside the 2MB (recoverable today)

### B1. The "SSP code below 0x20000, 68k code after" convention wastes ~124KB [FACT]

* `samples/SVP_mem_reader/main.asm:55-69` (and every other sample): `DCB.b 0x1FFF4-SVP_PaddingStart,0x00` zero-fills to 0x20000 "as Virtua Racing did", then puts the 68k code above it. The samples' SSP code uses a few hundred words of that 64K-word window.
* Only SSP **code that executes from ROM** has to live at words 0x0400–0xFBFF. 68k code, 68k data and any SSP data (read through PMs, which reach the full 2MB) can go anywhere. Packing 68k code and data into the unused part of 0x800–0x1F7FF recovers **up to ~124KB (≈6% of the ROM)**.
* This needs a linker-style flow. Today the 68k binary is the base and `ssp16asm` writes on top of it (§D1/D2 show how fragile that is).

### B2. The IROM-shadowed 2KB (ROM bytes 0x1F800–0x1FFFF) [FACT]

* SSP program fetches from words 0xFC00–0xFFFF hit the internal ROM, not the cart ROM (wiki "Internal ROM", and the `DEADBEEF` test at `samples/SVP_mem_reader/main.asm:70-72`). Those 2KB of cart ROM are **invisible to SSP instruction fetch but fully readable by the 68k and by PM reads**. Put 68k data there instead of padding.
* Similarly, ROM bytes 0x000–0x7FF are shadowed by IRAM for SSP fetch. They're already used for the 68k vectors and header.
* `samples/SVP_speed_test/tests.svp:253` assembles a copy of the IROM routine at word 0xFC8F. Those words are dead in ROM because the SSP executes the IROM's own copy. Harmless, but don't rely on anything ORG'd at 0xFC00+.

### B3. SSP code isn't limited to the first 128KB [FACT]

* The VR and IROM pattern (`loc_FC4B` in the IROM and VR's `0xC8DA` loader): read code from ROM with `((r0)+!)` or a PM, and write it to IRAM through PM4 with mode `0x081C` at `0x8000`.
* Code read through a **PM** can come from anywhere in the 2MB (the `((r0))` program-read form is limited to 64K words). So SSP code can be stored as **IRAM overlays anywhere in ROM**, and it also runs ~5× faster there: 216ns per instruction from ROM vs ~44ns from IRAM (wiki "Timings").
* Only a small resident dispatcher needs to stay below 0xFC00. This removes the 128KB code ceiling and frees B1's window for data.

### B4. Reuse the internal ROM's routines and sine table [FACT]

* The IROM (1K words) contains fill routines (`FC4B`, `FC58`, …), 8.24 fixed-point add/mul (`FC8A/FC8F`, `FC9C`), sin/cos lookup (`FD51/FD55`), and vertex-transform routines (`FD79`, …), plus a sine LUT at `0xFEE3` (`img/svp_sine_table_*.jpg`).
* Calling them (return address in `r6`, as the IROM expects) costs zero cart ROM, and they run at IRAM speed. The saving is small, a few hundred words, but it is free.

---

## C. Making 2MB hold more: effective-capacity techniques (no unknowns required)

### C1. SVP as a streaming decompressor into DRAM [INFERENCE, strongest practical route]

* **Pipeline:** compressed assets in ROM → SSP reads them through PMs (any of the 2MB) → decompresses in IRAM-resident code → writes to DRAM through PM4 → 68k / VDP DMA pulls from 0x300000.
* **Throughput budget:** IRAM code runs at ~22 M instructions/s (≈44ns each). Even at ~20 instructions per output word that's about 1M words/s, roughly 20K words (40KB) per frame. That's more than a frame's worth of VDP DMA bandwidth, so decompression would never be the bottleneck. (In the wiki's mixed real-life benchmark, IRAM code ran at 146% of the speed of the same routine run from ROM; wiki "Timings".)
* **Use the multi-PM trick (A7):** keep literal, match and control streams on separate PMs (PM1/PM2/XST for reads, PM4/EXT5 for writes) so the hot loop never reprograms PMC.
* **The DRAM becomes a 128KB cache.** Typical ratios of 2–4× on MD graphics and audio turn 2MB into an effective 4–8MB of assets, with no hardware change.
* **Caveat to verify:** VDP DMA from the SVP needs the source address set **2 bytes higher** than intended. svpdoc §4: "the code always sets DMA source address value larger by 2 … This is even true for DMAs from ROM, as it's probably hooked through SVP's memory controller." PicoDrive's `PicoSVPDma` returns `source - 2` to model it. `samples/SVP_basic_gfx/main.asm:178-190` DMAs from 0x302000 without that compensation, so check the first word of the tiles on hardware.

### C2. "Cell arrange" views: store once, read in three layouts [FACT]

* The 68k sees DRAM linearly at 0x300000 and through two address-scrambling windows at 0x390000 and 0x3A0000. The exact bit permutations are in VR's own test code (wiki "Virtua Racing test code", `sub_1BDC6` / `sub_1BD70`). Cell arrange 1: `(a & 0xF001) | ((a & 0x3E) << 6) | ((a & 0xFC0) >> 5)`. Cell arrange 2: `(a & 0xF801) | ((a & 0x1E) << 6) | ((a & 0x7E0) >> 4)`.
* Effect: a linear bitmap rendered or decompressed into DRAM can be DMA'd to VRAM *as tiles* with no conversion pass. You never need to store tile-ordered copies of bitmap data, or bitmap copies of tile data.
* Per svpdoc, cell arrange only maps the **first 64KB** of DRAM (0x300000–0x30FFFF), so use that half as the "render here, DMA as tiles" buffer.
* The SSP side has a matching write mode, "special increment" (`s` bit, mode `0x4018`): +1 on even addresses, and on odd addresses +32 according to svpdoc but +31 according to PicoDrive and the wiki's hardware test. The wiki measurement is the one taken on hardware, but settle it before relying on it. Combined with `d`, the wiki confirms the decrement variant works too.

### C2b. PMC nibble-rotate read: free dither and pattern variants [FACT]

* svpdoc §2.2.7: reading PMC while it's in "waiting for mode" state returns the word **rotated by 4 bits** (nibbles swapped). VR uses this on words such as `abab` to get `baba`, for chessboard dithering.
* One stored 4bpp pattern gives its pixel-shifted and checkerboard-alternated variant for free, so you can store half as many dither and stipple patterns.

### C3. Overwrite mode: hardware nibble transparency [FACT]

* PM write mode bit `v` (0x0400) writes only the non-zero nibbles (`overwrite_write` in `ssp16.c`, tested by VR test 5 "DSP DRAM OVER WRITE"). That's exactly 4bpp transparent-pixel compositing.
* Store sprite parts, layers and decals once and composite them in DRAM, instead of storing pre-composited frames. Big savings for animation.

### C4. Strided, decrementing and cell-order *ROM reads* [SPECULATION, untested on ROM]

* The emulator only implements ROM reads with **inc = 1** (`(mode & 0xfff0) == 0x0800`, commented "verified to be correct"). The `nnn` increment field (0,1,2,4,8,16,32,128), the `d` (decrement) bit and the `s` (cell) bit have only been exercised against DRAM and IRAM.
* If they work on ROM reads, the SSP gets **free asset transforms at read time**:
  * decrement = horizontally mirrored rows;
  * stride 2/4/8 = subsampled (half- or quarter-size) versions;
  * cell order = transposed or tiled reads.
* Then one master asset can stand in for its mirrored and scaled variants. Test with A1's probe using modes `0x8800`, `0x1000`, `0x1800`, `0x4800`, etc.

### C5. Procedural generation and an entropy source [INFERENCE]

* The SSP's single-cycle MAC plus the IROM sine table is ideal for generating textures, terrain, audio wavetables and geometry at runtime. That's what VR itself does: it stores polygon data, not frames.
* **Entropy:** EXT2 reads with UP0/UP1 clear return "random values", and USR0 toggles on each ST read (§A7). Either could seed a PRNG for procedural content without storing seeds or tables. Characterise the randomness before relying on it.

### C6. Polyglot / dual-use data [INFERENCE]

* The same ROM words are visible to the 68k (bytes, big-endian), to the SSP program bus and to the PMs (words). Tables can be dual-purpose, for example a sine LUT also used as a gradient tile row or palette ramp, or SSP immediate operands that double as 68k data. This is fiddly, but it's the most "indirect" form of reuse available.
* At a smaller scale: VR-style routines take parameters as data words that follow the call site (`0xC8DA` loader). The same idea works for compact bytecode or command lists that the SSP interprets, which is denser than native code. An SSP-hosted interpreter for game logic data is a real size win.

---

## D. Toolchain bugs and quirks (verified by building `ssp16asm` and running test inputs)

### D1. Silent truncation of SSP code beyond the base file [FACT, BUG]

`tools/ssp16asm/src/assembler/assembly.rs:489-494`: without `--fill`, the output is resized to the **base file's size**, discarding anything assembled past it with no error.

Repro: a 4KB base plus `org C000` code gives a 4096-byte output, and the code at byte 0x18000 is gone. This doesn't bite the samples only because their 68k binaries are always larger than 128KB.

Fix: resize to `max(file_size, max_address)`.

### D2. Labels wrap at 0xFFFF but emitted bytes don't [FACT, BUG]

The pass that builds the symbol table uses a `u16` address with `wrapping_add` (`assembly.rs:26,292`). The pass that emits code uses a `u64` (`:330`).

Repro: `org FFFE` followed by two instructions. The label for the second one is `0000`, but its bytes land at file offset 0x20000+. A `bra @wrapped` branches to IRAM 0.

Side effect worth knowing: `DW` streams that cross word 0xFFFF *do* keep emitting linearly into the next 128KB. That's an accidental way to place SSP-readable data above 128KB from the assembler, but labels on it are wrong.

### D3. An instruction can't directly follow a `DW` [FACT, BUG]

`DW` stays the "current operation" until another macro arrives, so `x: dw 0x2` followed by `y: ld a, …` fails with `Invalid token: Mnemonic(...) for instruction Some(Macro(Dw))`. Workaround: insert an `org`. This pushes people towards layouts that waste space.

### D4. No 2MB guard by default [FACT]

`--fill` defaults to 4MB, and a base file of 2MB+1 bytes is accepted without warning (`out5.bin` = 2,097,153 bytes).

Emulators make this worse [INFERENCE]: PicoDrive's SVP setup (`pico/carthw/svp/memory.c`) only overrides the DRAM, cell-arrange and I/O ranges, so the 68k ROM mapping is the generic whole-image one, so a >2MB SVP image can "work" in an emulator while the extra data simply doesn't exist on real hardware. **Always test with `--2meg`.**

### D5. `ORG` is 16-bit [FACT, design limit]

Data for PM reads above 128KB can't be placed with `ssp16asm`. It has to be baked into the 68k base binary. A 24-bit `ORG` (or a `DATA_ORG`) would let SSP data live anywhere in the 2MB.

### D6. `svptool` [FACT, minor]

* `-k 0` panics in `data.chunks(0)` (`tools/svptool/src/lib.rs:148`).
* A non-numeric or >65535 `-k` silently falls back to 512KB (`:28-40`).
* An odd-length input gets a padding byte placed *before* the last byte during the swap (`swap_endianness`, which pushes `0` then the byte). That's intended, but worth knowing when splitting images.

---

## E. Documentation and sample inconsistencies (can mislead memory-map work)

* svpdoc §3 still says "There were reports that SVP has internal ROM, but fortunately they were wrong." This repo's `SVP_mem_reader` and the wiki's "Internal ROM" page show there **is** a 1K-word internal ROM at 0xFC00–0xFFFF. svpdoc's program-memory table (ROM at 0x400–0xFFFF) is outdated on this point.
* The special-increment step is +32 in svpdoc and +31 in PicoDrive and on hardware (wiki). See §C2.

* `samples/SVP_basic_gfx/svp_gfx.svp:26` says "0x181000-0x1807FF" (end below start). The buffer actually starts at word 0x181000 = 68k 0x302000.
* `samples/SVP_basic_gfx/main.asm:178` says "Source - tile data in DRAM: 0x301000", but `svp.asm:39` uses `0x302000`.
* `svp_gfx.svp:59-71`: whether the blind write itself increments the PM address is "unconfirmed in hardware". This matters for every PM-addressed layout in §C. Resolve it with a two-word write test on the EPROM board.
* `samples/SVP_basic_gfx/init.asm:100,112`: MiSTer detects the SVP by the version string `GM MK-1229 -00`. Homebrew must keep it. Emulator SVP detection is header-based, not hardware-based.
* Wiki "Interrupts": the SVP "resets itself after a few seconds" of continuous interrupts. Relevant if you use IE1 (HSYNC) to pace a streaming decompressor. Prefer polling.

---

## F. Board-level observations

* **F1.** `svp-adapter-board` leaves ROM pins **1 and 44** unrouted (`svp-adapter-board.net:251-254`). On 44-pin Sega/Macronix-style 32Mbit mask ROMs, pin 1 is typically the next address line. If A1/A4 find a live A20, the adapter board is the place it would need routing. The EPROM board (27C160) would need a 4MB part with a different pinout.
* **F2.** `/BYTE#` is routed (net 23). Byte mode only reorganises the same 16Mbit, so it gives no capacity. Listed only to rule it out.
* **F3.** The VR prototype boards reportedly used **four 8-bit 120ns EPROMs** (wiki "Timings"). How those were decoded (on the board, or by an SVP chip-select) is worth checking if anyone has prototype board photos. A second chip-select pad on the SVP would tie in with A1/A4.

---

## G. Out of scope (needs extra hardware), listed for completeness

* Mega-CD (Mode 1) or 32X as a data source.
* A bank-switching CPLD or flash-cart passthrough (the "cartridge slot" board planned in `boards/README.md`).
* A larger ROM plus a bodge wire for a discovered A20 (A4/F1). This is a small mod, but still hardware.

---

## H. Suggested test order (cheapest and most decisive first)

1. **Visually read the VR PCB**: whether pads 72/74/75/98/100 have traces (A4). Also note the DRAM part number for the record (A2 is now low priority).
2. **A1 probe**: SSP reads at mode `0x0810–0x0817`, compared with `0x0800`. Also let a `0x0800` read increment across word 0xFFFFF, since svpdoc says the carry goes into bit 20.
3. **0xA1500A/C/E**: write and read these, and watch for any change in ROM or DRAM views (A6).
4. **A3/A2 confirmation**: a quick 68k dump of 0x200000+ and 0x320000+ to confirm svpdoc (internal-bus echo, DRAM mirrors).
5. **C4**: ROM reads with decrement, stride and cell modes.
6. **A5**: sweep the PM mode "?" bits (combinations).
7. **A6**: 0xA130F1–FF writes, and a 0xA15000–0xA150FF register scan.
8. **A7**: characterise EXT1 writes, EXT2 "random" reads and UP1.
9. Then build §C1 (IRAM decompressor + multi-PM streams + cell-arrange DMA), which pays off regardless of what 1–8 find.

Write every result back to the wiki. Even a clean "mirror/open bus" for A1–A3 is useful, because it closes the question for everyone.
