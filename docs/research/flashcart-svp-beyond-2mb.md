# SVP homebrew beyond 2MB on Mega EverDrive Pro and MegaSD

Companion to `2mb-limit-notes.md`. That file covers the real VR cartridge. This one covers the FPGA re-implementations the project actually targets.

Sources inspected (Nov 2026):

* **MiSTer Genesis core** (`MiSTer-devel/Genesis_MiSTer` @ `adc0c42`): `rtl/SVP/SVP.vhd`, `rtl/system.sv`, `Genesis.sv`.
* **Krikzz MEGA-PRO dev repo** (`krikzz/MEGA-PRO` @ `7064f45`, MIT): `fpga/mapper/map_svp/*`, `fpga/mapper/lib_base/*`, `edio/everdrive.[ch]`, `extended-ssf.md`.
* **MegaSD:** closed firmware. Terraonion's 1.02R01 release notes say the SVP support uses "SSP160x core used with permission from Sergey Dvodnenko" (srg320), the same author as the MiSTer core.

**All three products use srg320's SVP/SSP160x VHDL.** The Pro's `SVP.vhd` and MiSTer's differ only in packaging; the decode logic quoted below is identical.

---

## 1. The DSP (SSP) side is limited to 2MB in the core itself

`SVP.vhd` (MiSTer and Pro copies):

```vhdl
ROM_A : out std_logic_vector(20 downto 1);          -- 20-bit word address = 2MB
...
if MEM_ADDR(20) = '0' then                           -- ROM 000000-1FFFFF
    ROM_ADDR <= MEM_ADDR(19 downto 0);
elsif MEM_ADDR(20 downto 16) = "11000" then          -- DRAM
elsif MEM_ADDR(20 downto 15) = "111001" then         -- IRAM
end if;                                              -- anything else: no request
```

The core differs from real hardware in a few ways. All of them are FPGA facts, read from the source:

* A PM aimed at word 0x100000–0x17FFFF (the "unused" window from the hardware notes) **issues no memory request at all**. The PM keeps returning its previous data, so the A1 probe can't reach anything on these carts.
* Auto-increment only adds into `MA(19 downto 0)`, so the carry **does not** propagate into bit 20. svpdoc says it does on hardware.
* Increment code `nnn = 7` uses a per-PM **custom displacement** `CD`, not 128. `CD` is loaded by a blind read of EXT7 (`SSP_EA = "111"`), taking its value from PMC's low word. This is undocumented elsewhere, and probably a srg320 reverse-engineering result. It's usable for arbitrary strides (C4 in the main notes) on these carts.
* Special increment is +1/+31 (matching PicoDrive and the wiki, not svpdoc's +32).
* The cell-arrange formulas are the same as VR's test code.

**Conclusion: no software trick on the SSP gets past 2MB on any of the three.** SSP code and every asset the SSP reads must live in the first 2MB of the image, unless the mapper is changed (§3B).

---

## 2. The 68k side: depends on the wrapper, not the core

### MiSTer: already more than 2MB [FACT from RTL]

`rtl/system.sv`:

```verilog
else if(SVP_QUIRK && MBUS_A[23:20] == 3) ...          // 0x300000-0x3FFFFF -> SVP
else if (MBUS_A < ROMSZ) ...                           // everything else below ROM size -> ROM
...
if (ROMSZ > 'h200000) begin  // SSF2 banking: not gated by SVP_QUIRK
    BANK_REG[MBUS_A[3:1]] <= MBUS_DO[4:0];
assign ROM_ADDR = BANK_ROM ? {BANK_REG[MBUS_A[21:19]], MBUS_A[18:1]} : MBUS_A;
```

* SVP mode is enabled by the serial at 0x180: `"MK-1229 "` or `"G-7001  "` (`Genesis.sv:998-999`).
* With a ROM larger than 2MB, the 68k sees **0x000000–0x2FFFFF as 3MB of linear ROM**.
* The SSF2 bank registers (0xA130F3–0xA130FF, 512KB pages, 5-bit index) work **at the same time as the SVP**. That gives up to 16MB through windows 1–5. Windows 6–7 (0x300000+) are shadowed by the SVP decode.
* The SSP still sees only physical 0–2MB, with no banking (`ROM_ADDR2` is a separate, unbanked port).
* VDP DMA source addresses get the SVP −1-word fix on every ROM read in SVP mode (`MBUS_A - 1'd1` when `msrc == MSRC_VDP`).

So MiSTer is a good development target today, and shows how the other two should behave.

### Mega EverDrive Pro, stock SVP mapper: hard 2MB [FACT from RTL]

`fpga/mapper/map_svp/map_svp_v2.sv`:

```verilog
assign dout = svp_reg_ce ? svp_do : abus[21] == 0 ? rom_do_smd : ram_do_smd;
wire rom_oe_smd = abus[23:21] == 0 & (!oe | !as);
.mem_addr_a(abus[20:1]),          // 68k ROM port: 20 bits
.mem_addr_b(svp_rom_a[20:1]),     // SSP ROM port: 20 bits
```

* The memory ports are already `[21:1]` wide and the cart has 2×8MB PSRAM (`everdrive.h: ADDR_FCI_ROM … (2x8MB PSRAM)`), but only 20 address bits are wired in.
* 68k reads of 0x200000–0x2FFFFF return the DRAM data path, which is stale or mirrored. There's no ROM behind it.
* `topcfg.sv` builds the SVP mapper with `` `define MDP_OFF `` (MD+ CD audio disabled), `` `define MCD_OFF ``, and save states and cheats off.
* `hub.sv` instantiates the SVP core only when the menu's `map_idx == MAP_SVP`, falling back to the normal mapper otherwise.

### MegaSD: unknown wrapper

* The core is the same, so the SSP side is 2MB.
* MegaSD does implement the Extended-SSF style mapper (header "SEGA SSF" at 0x100) for normal games. **Whether "SEGA SSF" at 0x100 combined with an SVP serial at 0x180 gives both is untested.**
* This test is cheap: build a 4MB image with distinct markers per 512KB bank and that header combination, then check that the SVP still runs and that bank writes to 0xA130F2–0xA130FE switch what the 68k sees.

---

## 3. Ways past 2MB on the Mega EverDrive Pro (no extra hardware)

### 3A. Runtime overlays from the SD card through EverDrive I/O [strong lead, needs a test]

The Pro's system layer (`lib_base/everdrive.sv`, `base_io.sv`) is compiled into **every** mapper, including SVP. Its register block at 0xA130D0 (`base_io.sv:34`, decoded on `/TIME`, not gated by game mode) is the MCU mailbox that `edio/everdrive.c` uses. The relevant commands:

| Command | Effect |
|---|---|
| `CMD_F_FOPN` / `CMD_F_FPTR` | open and seek a file on the SD card |
| `CMD_F_FRD_MEM (0xCB)` | **the MCU reads a file straight into cart memory** at a given address |
| `CMD_MEM_WR (0x1A)` | write arbitrary bytes into cart memory |

Cart memory map from the MCU's view (`everdrive.h:97-103`): `0x000000` is ROM PSRAM, 2×8MB.

* In the SVP mapper, **`rom0` is the 2MB ROM that both CPUs see**.
* `rom1` is the SVP's 128KB DRAM (`MemBus svp_ram(mout.rom1, …)`, address `[16:1]`), which most likely sits at MCU address 0x800000. That last part is an inference from the wiring; check it.

So with the **stock** SVP mapper, the 68k can replace any part of the 2MB ROM image (or fill the SVP DRAM directly) with data streamed from files on the SD card. The amount of content is effectively unlimited: bank or level overlays, loaded at transition points.

Mechanics and caveats:

* `ed_run_dma()` → `ed_halt()` runs a small routine **copied to 68k work RAM** (`halt_app_ram[512]`) and spins there while the MCU owns the memory bus. Your loader has to be RAM-resident in the same way.
* Keep the SSP off cart ROM during a transfer: park it in an IRAM loop, or hold it via 0xA15006. The SVP core's ROM port and the MCU DMA share the PSRAM controller (`mem_ctrl … .dma_req(dma.req_rom0)`), and arbitration under SVP load hasn't been tested.
* Overlay regions must not contain code that's executing. Keep the resident 68k code, the vectors, the SSP's resident dispatcher (words 0x400–0xFBFF) and the IRAM loader outside them.
* Throughput is bounded by SD and MCU speed. The 68k is halted for the duration, so this suits loading screens, not per-frame streaming.
* **First test:** call `ed_init()` and then `ed_cmd_file_open` + `CMD_F_FRD_MEM` from a ROM running under the SVP mapper. Confirm that the MCU accepts commands in game mode and that the SVP sees the new data.

### 3B. Custom SVP mapper with banking [strongest, needs Quartus]

Custom mappers are an officially supported feature (`fpga/README.md`: "put `mega-pro.rbf` in the same folder as the ROM"; they can also be loaded over USB for testing). `fpga/fpga_pro/map_svp` is a complete Quartus project for exactly this mapper, MIT-licensed by Krikzz. srg320's SVP core is separately licensed and permission-based, so check its terms before distributing a modified bitstream.

Minimal changes to `map_svp_v2.sv`:

1. **68k banking.** Add eight 5-bit bank registers written at 0xA130F0–0xA130FE, as in `extended-ssf.md`. Feed `mem_addr_a` with `{bank[abus[21:19]], abus[18:1]}` and extend `rom_oe_smd` to cover 0x200000–0x2FFFFF (WIN4/WIN5). That gives up to 16MB visible from the 68k through 512KB windows, compatible with how MiSTer already behaves.
2. **SSP banking.** Choose one:
   * Prepend a bank register (written by the 68k, e.g. at 0xA15010, or by the SSP through a spare EXT write) to `svp_rom_a` for PM data reads only. That needs a small `SVP.vhd` change to tell `MAS_ROM_RD` (PM data) apart from `MAS_PROM_RD` (instruction fetch), so the code space stays fixed.
   * Or decode `MEM_ADDR(20 downto 19) = "10"` (the unused word window 0x100000–0x17FFFF) as "banked ROM". The SSP then gets a second 1MB window that the 68k pages, with no change to how normal VR code behaves.
3. Make `hub.sv` select `map_svp` unconditionally, so the menu's mapper choice doesn't matter.
4. Optional: rebuild with MD+ enabled (remove `MDP_OFF`) and move music to CD-audio tracks, which takes audio out of ROM entirely. Check that it fits in the FPGA first.

This is the only route that gives the **SSP** access to more than 2MB.

### 3C. Effective-capacity techniques

These work on all three carts. The FPGA cores already implement them:

* SSP decompression into DRAM;
* cell-arrange views;
* overwrite mode;
* strided PM reads (on these cores, include the custom displacement `nnn=7` / EXT7);
* the PMC nibble-rotate read.

See §C of `2mb-limit-notes.md`.

---

## 4. Recommended portable layout

1. **First 2MB:** 68k vectors and resident code; SSP resident dispatcher at words 0x400–0xFBFF; every asset the SSP reads.
2. **Above 2MB:** 68k-only assets (tiles, maps, audio, compressed SSP overlays that the 68k copies into DRAM for the SSP to unpack).
3. **One loader interface, three backends:**
   * **MiSTer:** linear 0x200000–0x2FFFFF plus SSF2 banks. Works today according to the RTL.
   * **Pro:** 3A (stock mapper, SD overlays) or 3B (custom mapper with banks).
   * **MegaSD:** SSF banks if the header-combination test in §2 passes. Otherwise, ask Terraonion, or stay at 2MB and lean on §3C.
4. Keep `"GM MK-1229 -00"` at 0x180 so all three detect the SVP.
