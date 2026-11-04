# Toolchain notes: ssp16asm, svptool, and building ROMs

Everything here was verified by building `tools/ssp16asm` (`cargo build --release`) and assembling test inputs. Encodings were checked word by word.

## 1. Build and use

```
cd tools/ssp16asm && cargo build --release      # binary: target/release/ssp16asm
ssp16asm -b output.bin code.svp rom_svp.bin      # assemble SSP code on top of a 68k binary
ssp16asm --fill --2meg ...                       # pad to 2MB (the default --fill size is 4MB)
```

Sample workflow (`samples/*/README.md`):

1. Assemble the 68k side with `SNASM68K.EXE /p main.asm,output.map,output.lst,output.bin`.
2. Run `ssp16asm -b output.bin <file>.svp rom_svp.bin`.
3. Optionally run `svptool -e -s -k 1024` to byte-swap and split the image for EPROMs.

## 2. Syntax essentials

* Numbers are always hex. `0x` and `h` are ignored. 1–2 digits make a byte, 3–4 digits a word (`ld a, 0x02` and `ld a, 0x0002` can encode differently).
* A label is defined as `name:` and referenced as `@name`. Constants: `name: EQU 1234` (word), `name: EQUB 12` (byte).
* `ORG` takes a **word** address. `DW` emits words.
* `#` starts a comment.
* Register names: `- x y a st stack pc p ext0..ext7 r0..r7`.
  * **`ext7` is the accumulator low word `AL`** (register 15). There's no `al` keyword.
  * VR disassembly names: PM0=ext0, PM1=ext1, PM2=ext2, XST=ext3, PM4=ext4, PMC=ext6.
* RAM-bank direct addressing: `A[0x10]` / `B[0x10]`. Only `ld A[nn], a`, `ld a, B[nn]` and `OP a, A[nn]` exist.
* Mode-word reminder: `ld ext6, <addr>` then `ld ext6, <mode>`.

## 3. Verified encodings (and what the FPGA core does with them)

| Source | Words | FPGA behaviour |
|---|---|---|
| `mod always, shr` / `shl` / `neg` / `abs` | 9002 / 9003 / 9006 / 9007 | ok |
| `mod always, ror` / `rol` / `inc` / `dec` | 9000 / 9001 / 9004 / 9005 | NOP |
| `mod z=0, shr` / `mod n=0, neg` | 9052 / 9076 | ok |
| `mod z=1, shr` / `mod n=1, neg` | 9152 / 9176 | **ADD A,0** (nothing) |
| `mod f, setie` | 9405 | sets IE only |
| `mod f, resie/setop/resop/setl/resl/set/res` | 9404/9409/9408/9403/9402/940F/940E | NOP |
| `bra z=1, x` / `call n=0, x` | 4D50 / 4870 + addr | ok |
| `bra l=0` / `bra ov=1` / `bra gpi0=1` | 4C40 / 4D60 / 4D20 + addr | never taken |
| `bra diof=1` | 4D80 + addr | always taken |
| `ld r3, 0x05` / `ld r7, 0x05` | 1B05 / 1F05 | NOP |
| `ld r3, a` / `ld a, r3` | 1433 / 1233 | spare byte register |
| `ld ext4, ext1` | 00C9 | **corrupts**: writes EXT1's write channel |
| `ld a, p` / `add a, p` | 0037 / 8007 | 32-bit |
| `ld (r3\|10), a` | 043B | RAM-A[2] |
| `ld a, ((r0+!))` | 0A34 | the RAM word increments; r0 is unchanged |
| `ld x, (a)` | 4A10 | program read at A[31:16] |
| `ld -, ext1` | 0009 | blind read (same as VR) |
| `ld -, ext7` | 000F | blind AL access: resets PMC, [FPGA] loads CD |
| `ld ext7, 0x0000` | 08F0 0000 | **clears A[15:0]** |
| `ld ext4, a` / `ld ext3, a` | 00C3 / 00B3 | |

## 4. Bugs and quirks (verified)

| # | Issue | Where | Effect | Workaround |
|---|---|---|---|---|
| T1 | Output truncated to the base-file size | `src/assembler/assembly.rs:489-494` | SSP code assembled past the end of the `-b` file is **silently dropped** | make the 68k base binary larger than every ORG'd address, or use `--fill` |
| T2 | Labels wrap at 0xFFFF but emitted bytes don't | `assembly.rs:26,292` vs `:330` | code at `org FFFE` + 2 words: the second label = 0x0000, its bytes land at 0x20000 | never cross word 0xFFFF |
| T3 | An instruction can't follow a `DW` directly | the DW state persists | "Invalid token … Some(Macro(Dw))" | put an `org` after DW blocks |
| T4 | No 2MB guard | default `--fill` = 4MB; a 2MB+1 base is accepted | emulators accept >2MB images that real hardware can't use | always use `--2meg` for hardware |
| T5 | `ORG` is 16-bit | design | can't place SSP data above 128KB | put it in the 68k base binary |
| T6 | `svptool -k 0` panics; a bad `-k` silently means 512 | `tools/svptool/src/lib.rs:28-40,148` | | |
| T7 | Odd-length input to `svptool -e` | `swap_endianness` | the last byte becomes `00 xx` | pad images to an even size |
| T8 | No `al` register name | `asm/registers.rs` | must write `ext7` | |

## 5. Sample-code corrections (in `samples/`)

* `SVP_basic_gfx/svp_gfx.svp:26` comment says "0x181000-0x1807FF" (the end is below the start). The loop actually writes 0x400 words, words 0x181000–0x1813FF (68k 0x302000–0x3027FF), while `main.asm` DMAs 0x1000 bytes (0x800 words) from 0x302000.
* `SVP_basic_gfx/main.asm:178` comment says 0x301000; the code (`svp.asm:39`) uses 0x302000.
* `svp_gfx.svp:59-71` blames a "blind-write increment" for its −1 PM address. The FPGA core doesn't increment on blind writes, so the −1 actually lines up with the **DMA one-word latency**.
* `SVP_speed_test/tests.svp`: `bra l=0` (carry) is never taken on the FPGA. `org 0xFC8F` places a copy of the internal-ROM routine into cart ROM. That's dead on hardware, but it's exactly what the FPGA executes.
* `SVP_mem_reader`: on the FPGA, "internal ROM" pages show cart ROM (the 0xDEADBEEF markers at 0x1FFF4 become visible).
* Every sample zero-pads bytes 0x800–0x1FFF4 and starts the 68k code at 0x20000. That isn't required, and it wastes ~124KB.
* `ld ext7, 0x0000` throughout the samples clears A[15:0]. It's harmless where it's used, but it isn't a register "reset".
