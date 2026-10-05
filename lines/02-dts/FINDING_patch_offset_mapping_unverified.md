# FINDING — 2026-09-30 (night): the patch offset mapping is UNVERIFIED — which reopens the whole patch line

**This invalidates a load-bearing historical conclusion of this project. Read before building
any further patch.**

## The problem

The static listings (`dec32_clean.txt` → `dec33_realcode.txt`) print **byte addresses**
(`046013`, `045ff1`, …) that are *not* uniformly 4-aligned (`0x45FE0`→`0x45FEB` is +11,
`0x45FEE`→`0x45FF1` is +3, `0x45FF5`→`0x45FF8` is +3) — consistent with AEON's mixed 16/32-bit
encoding, but it means an address is **not** a 4-byte-word index into `dec_full.bin`.

I attempted to derive the address→file-offset mapping three ways, all against
`dec_work/dec_full.bin` (md5 `4b7e9509…`, 1 982 492 B, the same md5 as the device's
`/data/local/tmp/dec_stock.bin`):

1. **The session's "verified" 32-bit encoder**
   `enc(rD,rA,imm,sub) = (7<<29)|(0x3B<<26)|(rD<<21)|(rA<<16)|(imm&~3)|sub` — searching for
   `sw 0x2c(r10),r0 @0x46013`, `sw 0x2c(r10),r11 @0x46087`, `sw 0x0(r10),r23 @0x3DE7C`,
   `sw 0xc(r12),r23 @0x19862`, `sw 0x28(r3),r23 @0x45FDD`: **0 hits** (LE and BE).
2. **Six field-order permutations** of the same encoder (rD/rA swap, opcode at 23/24, sub-field
   variants): **0 hits** for all five probe instructions simultaneously.
3. **The 24-bit forms from `aeon_ORBIS32.sinc`** (`bn.sw = i24_opcode 0x03 & uimm0_2 = 0`,
   decoder field 5): **0 hits** for `sw`/`lwz` probes in either endianness.

So either the encodings are still wrong, or the listing was produced from a *different* image
than `dec_full.bin`, or the file is a container (header/compressed section) rather than a flat
code stream at those addresses.

Side note: the Ghidra language file `aeon_ghidra_public/aeon/data/languages/aeon.slaspec`
is a **placeholder for OpenRISC** ("Specification for the 32-bit OpenRISC instruction set",
`define endian=big`) — the real mnemonics come from `aeon_ORBIS32.sinc`. Anything derived from
the slaspec alone is suspect.

## Why this matters

Four patches were flashed in earlier sessions and all were inert:

| patch | site | result |
|---|---|---|
| H1 | `0x460B1` fork | null |
| H3′ | `0x21AD3` licence clobber | null |
| type tag | `0x19861` `E4`→`E5` | null |
| SND Pc | SND image | null |

Those nulls were interpreted as proof that **the block is not in code** — the basis for
telling ourselves not to patch. If the offsets were wrong, the patches never touched the
intended instructions, and that conclusion collapses. The device currently carries **stock**
images, so nothing needs undoing; the images simply need to be re-derived and re-tested.

## The bounded task to fix this (do this before any new patch)

1. Disassemble `dec_full.bin` at file offset `0x3DA4C` and at `0x46013` with a tool that
   starts at a known offset (ReKo `disassemble --arch aeon` on a 64-byte slice, or Ghidra
   headless import at base 0), and compare the printed instructions with the listing lines
   `3da4c lwz r24,0xc(r12)` and `46013 sw 0x2c(r10),r0`.
2. Find the constant delta between listing addresses and file offsets, and confirm it is
   constant over a range (e.g. 0x3D000–0x47000). If no constant delta exists, the listing
   addresses are indices into a different unit — establish that unit first.
3. Only then rebuild the two narrowest candidates and re-test:
   * `0x45FF1` (`beqi r11,1 → 0x46080` — the arm that sets `+0x2C=1`, the IEC61937 packer arm),
   * `0x4121E` (`beqi r23,1 → 0x41275` — the output-stage gate on the pump latch).
4. Rollback procedure is already proven (patched images load; stock image restore + reboot).

The patch logic itself is unchanged and still justified by the live evidence in
`FINDING_host_exonerated_20260930.md` (host exonerated, block is DSP-side). Only the byte
addresses were ever in question.
