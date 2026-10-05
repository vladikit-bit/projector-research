# SPACE BUNNY — AEON 0x20 FAMILY / R11 RESTORATION CLOSURE

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device, ADB, playback, breakpoint, memory read, patch, or flash  
**TCL/T615:** not used  
**Preservation:** new artifact only; no existing report, SLEIGH, ReKo project, Ghidra project, binary, or raw JSONL was modified

## Closure result

The previously blocked AEON `0x20/0x21` family has now been independently decoded. ReKo's validated AEON backend supplies the semantics that the local Ghidra SLEIGH leaves empty.

This upgrades the earlier statement:

```text
r11 restoration at 0x38510: STRONGLY INFERRED
```

to:

```text
r11 restoration at 0x38510: PROVEN
```

The proof is byte-level and does not depend on a vendor ISA manual.

## 1. Independent decoder evidence

The corpus already records ReKo as a validated AEON decoder:

- `aeon_validate/REPORT_AEON_TOOLCHAIN_VALIDATION.md`
- ReKo executable: `C:\Program Files\jklSoft\ReKo\reko.exe`
- ReKo `--arch aeon` was previously validated on the corpus test sites.

The local Ghidra SLEIGH has empty bodies for the relevant `bt.*` forms. ReKo instead renders the same halfwords as explicit stack operations:

```text
bt.swst <slot*4>(r1),rN
bt.lwst rN,<slot*4>(r1)
```

The 0x20-family field layout independently derived from ReKo and raw bytes is:

```text
bits 15..10 = 0x20       stack-prefix opcode
bits 9..5   = rD         register number
bits 4..1   = slot       frame slot; byte offset = slot * 4
bit 0       = direction  0 = store, 1 = load
```

The layout was checked against 1,987 ReKo renderings with zero mismatches and independently reproduced on a second AEON binary (`snd_packer.bin`). The 0x21 family also agrees across ReKo, SLEIGH, and raw field decoding (`0x8529` is `bt.jr r9`).

## 2. Exact helper frame

`0x3847E` has a `0x40`-byte frame, i.e. 16 word slots. The prologue stores registers `r9..r18` as follows:

| Register | Prologue halfword | Slot | Frame offset |
|---:|---:|---:|---:|
| r10 | `0x815C` | 14 | `+0x38` |
| r13 | `0x81B6` | 11 | `+0x2C` |
| r15 | `0x81F2` | 9 | `+0x24` |
| r16 | `0x8210` | 8 | `+0x20` |
| r17 | `0x822E` | 7 | `+0x1C` |
| r9  | `0x813E` | 15 | `+0x3C` |
| r11 | `0x817A` | 13 | `+0x34` |
| r12 | `0x8198` | 12 | `+0x30` |
| r14 | `0x81D4` | 10 | `+0x28` |
| r18 | `0x824C` | 6 | `+0x18` |

The epilogue is the exact reverse-direction load map:

| Register | Epilogue halfword | Slot | Frame offset |
|---:|---:|---:|---:|
| r9  | `0x813F` | 15 | `+0x3C` |
| r18 | `0x824D` | 6 | `+0x18` |
| r17 | `0x822F` | 7 | `+0x1C` |
| r16 | `0x8211` | 8 | `+0x20` |
| r15 | `0x81F3` | 9 | `+0x24` |
| r14 | `0x81D5` | 10 | `+0x28` |
| r13 | `0x81B7` | 11 | `+0x2C` |
| r12 | `0x8199` | 12 | `+0x30` |
| r11 | `0x817B` | 13 | `+0x34` |
| r10 | `0x815D` | 14 | `+0x38` |

All ten register-to-slot pairs match exactly. The epilogue then executes:

```text
038516  bn.addi r1,r1,0x40
038519  bt.jr r9
```

Therefore the loop writes at `0x384CB/0x384DE` are inside the saved-register window and are overwritten by the epilogue load:

```text
0x38510: r11 = [r1+0x34]
```

This dominates every normal return path from the helper.

## 3. Consequence for `U` and `A108`

The caller dataflow is now:

```text
0x3DF2D  r11 = [B0+0x1414C]       ; V_in
0x3DF3B  call 0x3847E             ; r11 restored
0x3DF4D  call 0x3847E             ; r11 restored
0x3DF79  [B0+0x1414C] = r11
```

On the traced `F_03DF03` path, the returned/stored value is therefore:

```text
U = V_in
A108 = V_in + 0x7C4
```

The `r11 = r1+4*k` stack value remains a real intermediate value inside the loop, but it is no longer the return-visible value. The `U ∈ {V_in, stack address}` uncertainty from the previous addendum is closed for this helper.

The separate conditional-base issue in `F_3E961` is not closed by this result: its multiple `r10` definitions still prevent treating every `-0x5EF8(r10)` site as the same `B0+0xA108` without path-specific proof.

## 4. Standard-image corroboration

The standard/ALT counterpart at `0x43275` has the same prologue and epilogue byte structure. It corroborates the decoder result but is not needed for the proof and is not used for semantic transfer.

## 5. What is now closed and what remains

| Item | Status |
|---|---|
| AEON 0x20 stack-slot field layout | **Proven by ReKo/raw cross-check** |
| `0x3847E` r11 clobber inside loop | **Proven** |
| `0x3847E` return-visible r11 | **Proven restored to entry value** |
| `U` at `0x3DF79` | **`V_in` on the traced path** |
| `A108 = U+0x7C4` | **Proven on the traced path** |
| `F_3E961` all-sites base identity | Still conditional |
| `V_in` constructor/lifetime | Still unresolved |
| F8/FC → `R+0x174` | Still unresolved |
| SND external hardware consumer | Still unresolved |
| Physical SPDIF endpoint | Still unresolved |

The earlier AEON addendum remains preserved as historical evidence; this document supersedes its “r11 restoration not proven” verdict.

No runtime, device, TEE, TCL/T615, or binary-modification activity was used.
