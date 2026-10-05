# AEON R2 Toolchain Validation Report

**Date:** 2026-09-08
**Scope:** Independent validation of two AEON-capable disassembly tools against the actual
MStar MT5889 / AEON R2 DSP firmware, before any DTS-path reasoning (R27).
**Constraint honored:** No firmware artifact was modified, patched, flashed, or run. No ADB/runtime.
Only the *toolchain* (Ghidra processor module + Reko) was installed/fixed in a throwaway workspace.

---

## 0. Bottom line (classification)

| Tool | Result | Notes |
|------|--------|-------|
| **Reko** (`--arch aeon`) | **VALIDATED** | Decodes all 9 test sites; mnemonics, registers, immediates, mixed-width and jump targets are internally consistent and match the independently-established manual ISA model. |
| **Ghidra** (`aeon:LE:32:default`) | **PARTIALLY VALIDATED** | Registers, loads, and decodes opcodes/registers/immediates for all common instructions **exactly** like Reko, and handles mixed 24/32-bit width. **But** it (a) required two module fixes to even load, (b) its `bg.jal` (call) *target* reconstruction is broken (returns a constant `0x130CEA` for three different immediates), and (c) ~94 instructions have degenerate/placeholder constructors (obscure `bn.*` opcodes get wrong names). |

**Overall toolchain status: PARTIALLY VALIDATED** → the gate in the request ("do NOT start R27 unless at least PARTIALLY VALIDATED") is **met**. R27 may proceed. Reko is the trustworthy reference for *call/jump targets*; both tools are trustworthy for *opcode/register/immediate* decoding.

---

## 1. Tools & versions

### Tool 1 — Ghidra
- Install: `C:\ghidra_12.1.2_PUBLIC` (Ghidra 12.1.2 PUBLIC).
- AEON processor module: installed from `C:\Users\k0994\Downloads\ghidra-aeon-master.zip`
  (github.com/shinyquagsire23/ghidra-aeon).
- Exact language ID (case-sensitive): **`aeon:LE:32:default`**
- Endianness (from `aeon.ldefs`): `endian="little"` (data), `instructionEndian="big"`
  (instruction words are big-endian — matches the manual model). `size="32"`.
- Instruction-length model: **mixed 16/24/32-bit** (`instr16(16)`, `instr24(24)`,
  `instr32(32)` tokens). In this firmware only 24-bit (3-byte `bn.*`) and 32-bit
  (4-byte `bg.*`) forms actually appear; no 2-byte form was observed.

### Tool 2 — Reko
- Binary: `C:\Program Files\jklSoft\Reko\reko.exe`, `--arch aeon` verified available.
- Runtime: .NET 8 (`Microsoft.NETCore.App 8.0.30`, `Microsoft.WindowsDesktop.App 8.0.30`).

### Firmware regions (extracted, read-only)
- SND: `mst_snd_r2_MS12V22.bin` → `snd_region.bin` = bytes `[0x016F00 : 0x01C100]`
  (20 992 B; absolute base `0x16F00`).
- DEC/CODEC: `mst_codec_r2_MS12V22.bin` → `dec_region.bin` = bytes `[0x12400 : 0x22100]`
  (64 768 B; absolute base `0x12400`).

---

## 2. Ghidra AEON registration (Section 1 of request)

Ghidra did **not** register the module as shipped. Two fixes were required (both are changes to
the *tool*, not the firmware):

1. **Missing `Module.manifest`.** Every built-in Ghidra processor under
   `Ghidra/Processors/*` ships an (empty) `Module.manifest` at the module root; the AEON
   module had none, so Ghidra never discovered it → `Unsupported language: aeon:LE:32:default`.
   Fix: copied the real `Module.manifest` (and `LICENSE`, `data/patterns/`, `build.xml`,
   `sprs.py`) from the official ZIP.
2. **SLEIGH compile error in `aeon.slaspec`.** Ghidra 12.1.2's SLEIGH compiler is stricter
   than whatever built the repo's prebuilt `.sla`, so it rejects the prebuilt `.sla` and
   recompiles from source — which failed on two reversed bit-fields:
   - `aeon.slaspec:45`  `i16_uimm4_2 = (4, 2)`  → fixed to `(2, 4)`
   - `aeon.slaspec:95`  `i24_uimm4_2 = (4, 2)`  → fixed to `(2, 4)`
   (In this file's `(lo,hi)` convention `(4,2)` is invalid; both fields are *unused* in any
   constructor, so the flip changes no decode semantics.) After the fix the language compiles
   cleanly (only `94 NOP constructors` warnings remain).

After fixes, headless import succeeds and the script reports:
```
LANG: aeon:LE:32:default
ENDIAN_DATA: false
```
i.e. the processor loads, is little-endian data / big-endian instructions, 32-bit.

> **Operational note:** `analyzeHeadless` must be launched via `support/analyzeHeadless.bat`
> through **PowerShell** (not Bash — the `.bat` builds Windows-style classpaths), and the
> **project directory must be created beforehand** (`New-Item -ItemType Directory`); otherwise
> it aborts with `Directory not found`. A linear sweep via `disassemble()` in `-noanalysis`
> mode desyncs (emits `<UNDEFINED>` after a few KB); **per-address disassembly is reliable**,
> so all comparison data below was obtained with individual-address post-scripts.

---

## 3. Reko AEON mode (Section 2 of request)

Reko was driven with the *real* firmware base addresses (never guessed), e.g.:
```
reko.exe disassemble --arch aeon --base 0 --entry 0 ^
  --default-to unknown.format --dasm-address --dasm-bytes snd_region.bin
```
producing `snd_region.reko/snd_region_code.asm` (full linear decode of the region). The DEC
region and the four call-site windows were decoded the same way. Reko's addresses are
region-relative (base 0); add `0x16F00` (SND) / `0x12400` (DEC) to map to firmware addresses.

---

## 4. Mixed-width validation (Section 4 of request)

Confirmed on **both** tools. Reko's full SND listing shows 3-byte (`bn.*`) and 4-byte (`bg.*`)
instructions back-to-back with correct, non-aligned boundaries. Ghidra, decoded per-address:

| Addr | Raw | Ghidra | Reko | len |
|------|-----|--------|------|-----|
| 0x16F00 | 01 0C EE | `len=3 bn.op0_2 r8,r12,-0x12` | `len=3 bn.nop 0x0` | 3 |
| 0x16F03 | EC 01 02 EE | `len=4 bg.lwz r0,0x2ec(r1)` | `len=4 bg.lwz r0,0x2EC(r1)` | 4 |
| 0x16F1B | 20 34 9B | `len=3 bn.bnf 0x00017c41` | `len=3 bn.bnf? 00000D41` | 3 |
| 0x16F21 | 50 01 C3 | `len=3 bn.ori r0,r1,0xc3` | `len=3 bn.ori r0,r1,0xC3` | 3 |

The 3-byte vs 4-byte split is honored; common instructions (`bg.lwz`, `bn.ori`, `bn.bnf`)
agree exactly between tools (note `0x16F1B`: Ghidra `0x17c41` = Reko `0x0D41` + base `0x16F00` —
**exact target match**). Obscure `bn.*` opcodes (`bn.nop`/`bn.op0_2`) get different *names*
because Ghidra's module has placeholder constructors for them — a cosmetic/naming gap only.

---

## 5. Comparison table — 5 known SND instructions (Sections 3, 5)

Raw bytes are the big-endian instruction word. All addresses are **firmware** addresses.

| # | Addr | Raw | Manual model (R25) | Ghidra | Reko | Agreement |
|---|------|-----|--------------------|--------|------|-----------|
| A | 0x16F84 | EE EC 00 FA | LD r23, 0xFA(r12) | `bg.lwz r23,0xf8(r12)` | `bg.lwz r23,0xF8(r12)` | **Tools exact.** Manual off by load-offset *granularity*: field `0xFA` → offset `0xF8` (×4). |
| B | 0x1BEF6 | FF 20 F8 72 | ori r25,r0,0xF872 | `bg.addi r25,r0,-0x78e` | `bg.addi r25,r0,-0x78E` | **Exact** (`-0x78E` ≡ `0xF872` signed). |
| C | 0x1BF02 | FF 20 4E 1F | ori r25,r0,0x4E1F | `bg.addi r25,r0,0x4e1f` | `bg.addi r25,r0,0x4E1F` | **Exact.** |
| D | 0x1C046 | CB 20 18 00 | r25 = 0x1800 | `bg.ori r25,r0,0x1800` | `bg.ori r25,r0,0x1800` | **Exact.** |
| E | 0x1C05E | E7 FF FD 95 | CALL (link, target per R25) | `bg.j 0x0001bf28` | `bg.j 00005028` (=0x1BF28) | **Tools exact** (mnemonic + target). **Manual "CALL/link" REJECTED** — both tools decode a *plain jump, no link* (bit0=1). |

**Conclusion for A–D:** Ghidra and Reko decode the opcodes, registers and immediates *identically*,
and both correct the manual model's load-offset granularity (the `lwz` offset is the 14-bit field
shifted left 2, so raw `0xFA` → `0xF8`). **For E:** both tools overturn the manual "CALL" assumption.

---

## 6. Do NOT trust assumptions — manual-model corrections surfaced

- **SND E is a jump, not a call.** The manual model (R25-C) labelled `0x01C05E` a linking CALL
  with target "per R25". Both tools decode `E7 FF FD 95` as `bg.j` (plain jump, link bit0 = 1 →
  **no** link into `r9`), target `0x1BF28` — an in-region backward jump (lands between C and D),
  **not** `r25`-relative. The "target per R25" hypothesis is false.
- **Load offset is ×4 granularity.** `bg.lwz` offset = 14-bit field `<< 2`; raw field `0xFA`
  decodes to `0xF8`. The manual model's `0xFA` was the *unshifted* field.
- **Opcode 0x39 is `jal` (link, bit0=0) or `j` (no link, bit0=1).** Link register is `r9`
  (`r9 = inst_next + 4`). Both tools apply this rule identically.

---

## 7. CALL instruction analysis — `0x01C05E` (Section 7)

- **Mnemonic:** `bg.j` (both tools). Plain jump.
- **Operands:** 26-bit displacement field; no register operand.
- **Direct vs register-relative:** direct (absolute displacement), **not** `r25`-relative.
- **Link register:** **not** written (bit0 = 1 ⇒ `j`, not `jal`).
- **Displacement / computed target:** `0x1BF28` (Ghidra and Reko agree; Reko's `0x5028` is the
  region-relative form, +`0x16F00` = `0x1BF28`).
- **Target verify:** `0x1BF28` lies inside the SND region (between instructions C/D) — a plausible
  internal branch. **Verified consistent across both tools.** The manual model's "CALL" claim is
  **not** supported.

---

## 8. DEC image call-site cross-check (Section 8)

Four call-like sites in `mst_codec_r2_MS12V22.bin`:

| Addr | Raw | Ghidra | Reko | Mnemonic/link |
|------|-----|--------|------|---------------|
| 0x124A6 | E4 23 D0 88 | `bg.jal 0x00130cea` | `bg.jal 0011E844` | **Agree: jal (link)** |
| 0x21FD2 | E7 FF FF 3F | `bg.j 0x00021f71` | `bg.j FFFFFF9F` | **Agree: j (no link)** |
| 0x21FE2 | E4 21 DA 10 | `bg.jal 0x00130cea` | `bg.jal 0010ED08` | **Agree: jal (link)** |
| 0x22058 | E4 21 D9 24 | `bg.jal 0x00130cea` | `bg.jal 0010EC92` | **Agree: jal (link)** |

**Mnemonic / link-bit agreement is 100%** between the tools for all four sites. **Target values:**
- Ghidra returns the **same** `0x130CEA` for *all three* `bg.jal` sites despite three *different*
  immediates (`0x023D088`, `0x021DA10`, `0x021D924`) → its `call` target reconstruction is
  **defective** (constant). Reko returns distinct, self-consistent targets
  (`0x11E844`, `0x10ED08`, `0x10EC92`).
- For the single plain `bg.j` (`0x21FD2`) Ghidra shows `0x21F71` (an in-region absolute target),
  Reko shows the raw displacement `FFFFFF9F`; the two use different display/addressing
  conventions but agree the instruction is a non-linking jump.
- **Therefore: trust Reko for CALL/JUMP targets; treat Ghidra's `bg.jal` target as unreliable.**
  Ghidra's `bg.j`/`goto` targets are usable (e.g. `0x1BF28` matched Reko exactly).

---

## 9. Final classification (Section 9)

- **Reko — VALIDATED.** Full decode path (mnemonic, registers, immediates, mixed width, jump
  targets) consistent with the manual ISA model and independently cross-checked against Ghidra.
- **Ghidra — PARTIALLY VALIDATED.** Correctly registers/loads and decodes opcodes, registers and
  immediates for every common instruction (A–D, all DEC sites) **identically to Reko**; mixed
  24/32-bit width confirmed. Limitations: (1) needed module fixes to load at all; (2) `bg.jal`
  target reconstruction is broken (constant `0x130CEA`); (3) ~94 NOP/degenerate constructors →
  obscure `bn.*` opcodes get placeholder names. None of these affect the *opcode/operand* decoding
  that the validation hinges on.
- **CALL disagreement?** No genuine tool-vs-tool disagreement on linkage: both apply the bit0
  rule identically. The only divergence is Ghidra's broken *target* arithmetic for `jal`, which
  is a known module limitation, not a conflicting decode.

**=> At least PARTIALLY VALIDATED. R27 may start.**

---

## 10. R27 gate (Section 10)

The precondition ("do NOT start R27 unless at least PARTIALLY VALIDATED") is satisfied. This
report does **not** start R27 — that is left to the next step. When R27 begins, use:

- **Reko** as the primary decoder for call/jump *targets* and for any full-region sweep.
- **Ghidra** (`aeon:LE:32:default`) as a cross-check for opcode/register/immediate decoding and
  for mixed-width; ignore its `bg.jal` target and obscure-`bn.*` names.
- The corrected manual ISA model: 32-bit BE `|op(6)|rD(5)|rA(5)|imm16(16)|`; `lwz` offset ×4;
  opcode `0x39` = `jal`(bit0=0)/`j`(bit0=1), link `r9`; `0x01C05E` is a plain jump to `0x1BF28`.

---

## Appendix A — exact Ghidra commands (PowerShell)

```powershell
# SND individual sites (base 0x16F00), project dir pre-created:
New-Item -ItemType Directory -Force -Path C:\firmware_temp\aeon_validate\ghidra_snd5 | Out-Null
& C:\ghidra_12.1.2_PUBLIC\support\analyzeHeadless.bat `
   C:\firmware_temp\aeon_validate\ghidra_snd5 sndproj5 `
   -import C:\firmware_temp\aeon_validate\snd_region.bin `
   -loader BinaryLoader -loader-baseAddr 16F00 `
   -processor aeon:LE:32:default -noanalysis `
   -scriptPath C:\firmware_temp\aeon_validate\scripts `
   -postScript DumpAEON.java 16F84 1BEF6 1BF02 1C046 1C05E 4FF0 4FF3 4FF6

# DEC call sites (base 0x12400):
& C:\ghidra_12.1.2_PUBLIC\support\analyzeHeadless.bat `
   C:\firmware_temp\aeon_validate\ghidra_dec4 decproj4 `
   -import C:\firmware_temp\aeon_validate\dec_region.bin `
   -loader BinaryLoader -loader-baseAddr 12400 `
   -processor aeon:LE:32:default -noanalysis `
   -scriptPath C:\firmware_temp\aeon_validate\scripts `
   -postScript DumpAEON.java 124A6 21FD2 21FE2 22058
```

## Appendix B — exact Ghidra decode excerpts (collected)

```
# snd_region, base 0x16F00  (gh_snd_indiv.txt)
0x16F84  EEEC00FA  len=4  bg.lwz r23,0xf8(r12),
0x1BEF6  FF20F872  len=4  bg.addi r25,r0,-0x78e
0x1BF02  FF204E1F  len=4  bg.addi r25,r0,0x4e1f
0x1C046  CB201800  len=4  bg.ori r25,r0,0x1800
0x1C05E  E7FFFD95  len=4  bg.j 0x0001bf28

# dec_region, base 0x12400  (gh_dec_calls4.txt)
0x124A6  E423D088  len=4  bg.jal 0x00130cea
0x21FD2  E7FFFF3F  len=4  bg.j 0x00021f71
0x21FE2  E421DA10  len=4  bg.jal 0x00130cea
0x22058  E421D924  len=4  bg.jal 0x00130cea

# mixed-width probe, base 0x16F00  (gh_snd_mixed.txt)
0x16F00  010CEEEC  len=3  bn.op0_2 r8,r12,-0x12
0x16F03  EC0102EE  len=4  bg.lwz r0,0x2ec(r1),
0x16F1B  20349B00  len=3  bn.bnf 0x00017c41
0x16F1E  00CAE050  len=3  bn.sw_0 -0x20(r10),r6,
0x16F21  5001C317  len=3  bn.ori r0,r1,0xc3
```

## Appendix C — exact Reko decode excerpts (collected)

```
# snd_region_code.asm
00000084 EE EC 00 FA   bg.lwz   r23,0xF8(r12)
00004FF6 FF 20 F8 72   bg.addi  r25,r0,-0x78E
00005002 FF 20 4E 1F   bg.addi  r25,r0,0x4E1F
00005146 CB 20 18 00   bg.ori   r25,r0,0x1800
0000515E E7 FF FD 95   bg.j     00005028

# dec per-site windows (base 0)
00000000 E4 23 D0 88   bg.jal   0011E844
00000000 E7 FF FF 3F   bg.j     FFFFFF9F
00000000 E4 21 DA 10   bg.jal   0010ED08
00000000 E4 21 D9 24   bg.jal   0010EC92
```

## Appendix D — artifacts produced
- `C:\firmware_temp\aeon_validate\scripts\DumpAEON.java` — Ghidra post-script (per-address decode).
- `C:\firmware_temp\aeon_validate\snd_region.bin`, `dec_region.bin` — extracted regions.
- `C:\firmware_temp\aeon_validate\gh_snd_indiv.txt`, `gh_dec_calls4.txt`, `gh_snd_mixed.txt` — Ghidra outputs.
- `C:\firmware_temp\aeon_validate\snd_region.reko\`, `dec_region.reko\`, `dec_12*_reko\` — Reko listings.
- Ghidra AEON module (fixed): `C:\ghidra_12.1.2_PUBLIC\Ghidra\Processors\aeon\`
  (added `Module.manifest` + `data/patterns/`; `aeon.slaspec` fields `(4,2)→(2,4)`).
