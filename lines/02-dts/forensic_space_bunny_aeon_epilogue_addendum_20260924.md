# SPACE BUNNY — AEON EPILOGUE / R11 ADDENDUM

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none  
**TCL/T615:** not used  
**Preservation:** new artifact only; no existing report, binary, Ghidra project, ReKo project, or raw JSONL was modified

## Purpose

This addendum records the final bounded recheck of MS12 DEC helper `0x3847E` after the broader static continuation pass. It preserves the exact evidence and the remaining decoder boundary without changing the conclusions in the earlier reports.

## 1. Helper extent and direct entry

Primary artifact: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (`0x1E401C`, SHA-256 `530bfa684bc363e8513951b731cd42ccae97c96accef3c1d51eb431b61a53f5b`) and the authoritative `dec32_clean.txt` listing.

The helper extends over:

```text
0x3847E..0x38519
```

The frame is exact:

```text
03847e  bn.addi r1,r1,-0x40
...
038516  bn.addi r1,r1,0x40
038519  bt.jr r9
```

There are exactly two direct call sites:

```text
0x3DF3B -> 0x3847E
0x3DF4D -> 0x3847E
```

No pointer-table entry for `0x3847E` was found.

## 2. Proven reachable `r11` clobber

The loop path is byte-level visible:

```text
0384c8  bn.blesi r12,0,0x384e4
0384cb  bt.mov   r11,r1
0384de  bt.addi  r11,0x4
0384e0  bg.bgts  r12,r10,0x384cf
```

The compiled SLEIGH confirms that the `bt.addi` body is `rD = rD + simm5`; the field name `i16_rA` in the source is misleading for this case.

When `r12 > 0`, the post-loop value is a stack-derived address:

```text
r11 = r1 + 4 * iterations
```

It is not `V_in` and not `B0` on that path. When `r12 <= 0`, the first write is skipped. The old statement that `0x3847E` never clobbers `r11` is therefore disproven.

## 3. Exact epilogue boundary

The return-visible block is:

```text
038500  81 3f
038502  98 61    ; real 2-byte bt.movi r3,1
038504  82 4d
038506  82 2f
038508  82 11
03850a  81 f3
03850c  81 d5
03850e  81 b7
038510  81 99
038512  81 7b
038514  81 5d
038516  1c 21 40 ; frame teardown
038519  85 29    ; return through r9
```

A six-32-bit interpretation is structurally impossible because it would swallow the teardown and return. A mixed 16/32-bit interpretation and an all-16-bit interpretation remain possible from the local bytes.

## 4. Structural inference, not proof

Across four related functions, the prologue and epilogue `0x20`-family halfwords have a stable mirrored relationship:

| Function | Prologue slots | Epilogue slots | Paired operand relation |
|---|---:|---:|---|
| `0x3847E` | 10 | 10 | `R+B = 24` |
| `F_03DF03` | 7 | 7 | `R+B = 16` |
| `F_03E0DD` | 10 | 10 | `R+B = 22` |
| `F_03DF92` | 3 | 3 | `R+B = 14` |

The paired words differ systematically in bit0, and the operand fields form contiguous ranges consistent with saved-register indices. If the fields are register indices, the epilogues restore the saved registers, including `r11`. That would make:

```text
U = V_in
A108 = V_in + 0x7C4
```

at the direct `F_03E0DD` path.

This is a medium-confidence structural inference. It is not a local instruction decode. The operand could instead be a slot index, in which case the register mapping is not recoverable from these bytes alone.

The local `aeon.cspec` cannot arbitrate the question because it describes an unaffected-register contract that is inconsistent with reachable clobbers in the same functions.

## 5. Corrected DEC dataflow details

The second memset base in the `F_03DF03` chain is:

```text
[B0+0xA12C]
```

at `0x3DF25`, not `[B0+0xA148]`.

The `r11` chain into the `A108` store passes through:

```text
0x3847E (twice)
 -> F_03DF03 epilogue
 -> F_03DF92 epilogue
 -> 0x67366 (r11-clean leaf)
 -> 0x3E19E: r23 = r11+0x7C4
 -> 0x3E1A6: [B0+0xA108] = r23
```

`F_03DF92` itself writes `r11` at `0x3DF9B`; `0x67366` writes only `r3`/`r23` and returns through `r9`.

## 6. Standard-image comparison

The standard/ALT counterpart at `VERIFY_adec_r2_alt.bin+0x43275` is structurally identical in the helper prologue and all 26 epilogue bytes. Only call displacements differ. This is byte-level code identity, not an independent semantic decode and not an automatic transfer of behavior.

## 7. Final bounded verdict

```text
r11 clobber on the loop path:       PROVEN
r11 restoration at the epilogue:   STRONGLY INFERRED, NOT PROVEN
U = V_in:                          CONDITIONAL
A108 = U+0x7C4:                    VERIFIED AS AN EXPRESSION,
                                    CONDITIONAL ON U
```

The remaining blocker is the semantics of the AEON `0x20/0x21` instruction family. The local SLEIGH leaves `bt.trap`, `bt.nop`, `bt.swst`, and `bt.inst16_4` with empty bodies. A primary MStar/AEON ISA description or an independent R2 decoder is required to close the epilogue definitively.

No runtime, device, TEE, TCL/T615, or binary-modification activity was used.
