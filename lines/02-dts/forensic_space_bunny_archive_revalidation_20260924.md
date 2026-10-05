# SPACE BUNNY — ARCHIVE REVALIDATION AND ADDRESS-CLASSIFICATION PASS

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Device/runtime:** none; no ADB, device connection, playback, breakpoint, runtime memory read, module reload, reboot, settings change, patch, firmware write, or flash  
**TCL/T615:** not used in this pass  
**Preservation:** no previous report, raw JSONL, firmware binary, Ghidra project, or historical archive file was modified

## Executive verdict

The backup folder contains material primary-device-init evidence that was inherited by the historical session but had not been fully rechecked against the current TD98 corpus. The recheck closes the **provenance** of the decoder base and the **numeric configuration input**, but it also requires a stricter address classification:

1. `MI_SYS_GetMmapLayout(0x13B, ...)` is the direct producer path for the decoder-ID-2 base request. The raw module normalizes `0x13B` to table index **`0x44` (hex; decimal 68)**, whose name-table entry is `MI_MAD_ADV_BUF`.
2. The extracted TD98 configuration gives `MI_MAD_ADV_BUF_ADR = 0x0022500000`, length `0x001B00000`, and `CMA_HID = 0`. Thus `B = 0x22500000` is supported for this configuration/init path. It is a configuration value, not a literal address proved inside `mik_stock.ko`.
3. `HAL_AUDIO_SetDspBaseAddr(2, ...)` masks the supplied low word with `& ~0xF`, writes it to `AudioVars+0xA0`, and writes zero to `AudioVars+0xA4`; `HAL_AUDIO_GetDspMadBaseAddr(2)` reads that pair.
4. The SHM33 destination is computed as `MsOS_PA2KSEG1(B + 0xE01000) + 0x1044 + 0x3C0*slot`. The arithmetic candidates are `0x23302044` and `0x23302404` for slots 0 and 1 **before accounting for the return value of `MsOS_PA2KSEG1`**. They must not be called live CPU or DSP addresses from static code alone.
5. The TEE loader descriptor is rechecked exactly, but OPTEE dispatch is conditional on `AudioVars+0x24F0 == 2`, and the secure invocation ends at external `TEEC_InvokeCommand` / `tee_client_invoke_func` symbols. Neither `B` nor `B+0x45D07` is thereby proved to be the DSP PC.

This pass changes the confidence and wording of the base/SHM33 edges. It does not close the DSP-visible SHM mapping, live selector, secure loader implementation, or physical transport boundary.

## 1. Archive coverage and preservation boundary

The following archive material was read directly:

- `C:\firmware_temp\Бекап розслідування\GLM_archive_20260918\SESSION_INDEX.md`
- `C:\firmware_temp\Бекап розслідування\GLM_archive_20260918\ARCHIVE_SOURCES.md`
- Targeted final evidence blocks in:
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_fc9171c8_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_7280ff60_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_c15b682b_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_b1eab034_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_4a1b4982_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_ba3dda29_001.md`
  - `SUBAGENT_TRAJECTORIES\SUBAGENT_9f05c8e9_001.md`

The archive was **not** treated as a complete substitute for raw analysis. The seven rendered transcript chunks, every raw JSONL record, and every DB message were not exhaustively reread in this pass. The targeted traces were cross-checked against the current binaries and extracted configuration below. No JSONL normalization or archive rewrite was performed.

## 2. `0x13B` → `MI_MAD_ADV_BUF` → decoder base

### 2.1 Direct ARM producer path

`MI_AEXTIN_Init` in `patch_baseline\mik_stock.ko` contains the following `.text`-relative ARM instructions:

```text
05e430  movw    r0, #0x13b
05e434  mov     r1, r5             ; layout buffer = sp+0x50
05e438  bl      MI_SYS_GetMmapLayout
...
05e450  ldr     r5, [sp, #0x60]     ; layout+0x10
05e458  ldr     r4, [sp, #0x64]     ; layout+0x14
...
05e4a8  mov     r0, #2
05e4ac  mov     r2, #0
05e4b4  str     r5, [sp]
05e4b8  str     r4, [sp, #4]
05e4bc  bl      MDrv_AUDIO_SetDspBaseAddr
```

The same physical-word-to-setter pattern is independently present in the audio initialization paths, including the `_MI_PCM_HwInit` diagnostic region. This proves that the setter receives the layout's physical address pair; it does not prove that the pair is a DSP execution address.

### 2.2 Exact ID normalization and table entry

The current `mik_stock.ko` implementation of `mi_sys_GetMmapLayout` (`.text+0x2806cc`) normalizes the high ID range as follows:

```text
2807a4  cmp     r4, #9
2807a8  sub     r0, r4, #0x100
2807ac  cmp     r0, #0x8d
2807b4  sub     r3, r4, #0xf7
2807b8  add     sl, r3, r3, lsl #3
2807bc  ldr     r2, [r7, #0x20]
2807c8  add     r2, r2, sl, lsl #3
```

For `r4 = 0x13B`:

```text
r3 = 0x13B - 0xF7 = 0x44
entry offset = 0x44 * 9 * 8 = 0x1320
```

The table stride is `0x48`, so `0x1320 / 0x48 = 0x44`. The companion ID-to-name path uses the same `id - 0xF7` normalization and indexes `_aszMmapLayoutString` at `0x110b20 + 0x44*4`. That entry resolves to:

```text
MI_MAD_ADV_BUF
```

The hexadecimal qualification matters: `0x44` is decimal 68, not decimal 44. The nearby name-table entry at decimal index 44 is a different name and must not be substituted.

The caller then reads the returned layout fields at `+0x10` and `+0x14`; those are the words passed to `MDrv_AUDIO_SetDspBaseAddr` in the producer chain above.

### 2.3 Configuration value for `B`

The extracted TD98 MT5889 configuration contains:

```text
C:\firmware_temp\display_edid_investigation\extracted\cusdata\common\MMAP_MI.h:242
/* MI_MAD_ADV_BUF   */
#define MI_MAD_ADV_BUF_AVAILABLE               0x0022500000
#define MI_MAD_ADV_BUF_ADR                     0x0022500000
#define MI_MAD_ADV_BUF_LEN                     0x0001B00000
#define MI_MAD_ADV_BUF_CMA_HID                 0
```

The same address/length values occur in the extracted 2G and 3G MT5889 MMAP headers. The configuration file is part of the current extracted TD98 corpus; it is not a runtime read in this pass.

Therefore, for the stock configuration and the demonstrated initialization path:

```text
B = 0x22500000
```

This is a static configuration result, not proof that every possible allocation or replacement path must return the same value. A raw byte search for the little-endian `0x22500000` pattern in `mik_stock.ko` produced a hit in `__ksymtab` metadata, not in the mmap initializer; that hit was discarded as runtime-base evidence.

### 2.4 Setter normalization

The current `utpa2k_stock.ko` `HAL_AUDIO_SetDspBaseAddr` region contains:

```text
44c920  and     r1, r4, #0xf
44c940  bic     r2, r4, #0xf
44c954  str     r2, [r0, #0xa0]
44c95c  str     fp, [r0, #0xa4]       ; fp = 0 on this path
44c968  ldr     r4, [r0, #0x88]
44c970  ldr     sb, [r0, #0x8c]
44c97c  ldr     r4, [r0, #0xa0]
44c984  str     sb, [r0, #0xa4]
```

`HAL_AUDIO_GetDspMadBaseAddr(2)` later reads the `+0xA0/+0xA4` pair. This closes the host-side algebra:

```text
B_low  = layout[0x10] & ~0xF
B_high = 0
```

It does not establish any relationship between `B` and a DSP instruction pointer.

## 3. SHM selector `0x33`: exact arithmetic and address classification

### 3.1 Raw setter path

`HAL_DEC_R2_Set_SHM_PARAM` starts at `.text+0x45A434`. The selector dispatch reaches the `0x33` case (hexadecimal 51, not decimal 33) through the table at `0x45A4C0` and the case block at `0x45ACEC`:

```text
45acec  rsb     r0, fp, fp, lsl #4      ; 15 * slot
45acf0  movw    r1, #0x1044
45acf4  b       0x45b348
45b348  add     r0, r6, r0, lsl #6      ; shm + 0x3c0 * slot
45b34c  str     sl, [r0, r1]
45b350  bl      MsOS_FlushMemory
```

The function validates slot values 0 and 1 before the relevant case. Its lazy base initialization repeats:

```text
45a460  mov     r0, #2
45a464  bl      HAL_AUDIO_GetDspMadBaseAddr
45a468  movw    r2, #0x1000
45a46c  movt    r2, #0xe0
45a470  adds    r0, r0, r2
45a474  adc     r1, r1, #0
45a478  bl      MsOS_PA2KSEG1
45a47c  mov     r6, r0
45a484  str     r0, [g_virDecR2shm]
```

`HAL_DEC_R2_init_SHM_param` has the same `GetDspMadBaseAddr(2) + 0xE01000 -> MsOS_PA2KSEG1 -> g_virDecR2shm` construction at `0x458454..0x458478`.

### 3.2 What the numbers mean

The static equation is:

```text
P_shm = MsOS_PA2KSEG1(B + 0xE01000)
D0    = P_shm + 0x1044
D1    = P_shm + 0x1044 + 0x3C0
```

For `B = 0x22500000`:

```text
B + 0xE01000              = 0x23301000
numeric slot-0 source     = 0x23302044
numeric slot-1 source     = 0x23302404
```

The last two values are arithmetic results after applying the offsets to the **pre-mapping physical source**. `MsOS_PA2KSEG1` is a mapping wrapper; the current static chain does not prove that its return value is numerically identical to its physical input. Therefore:

- `0x23302044` and `0x23302404` are valid **backing-address candidates / pre-mapping arithmetic values** under the stock `B` hypothesis;
- they are not, by themselves, verified live CPU virtual destinations;
- they are not DSP addresses;
- they do not close the SHM33 live-value producer or the DSP-visible mapping.

This is a classification correction to any inherited wording that called the two values unconditionally “host physical locations.” The underlying arithmetic is verified; the address domain after `MsOS_PA2KSEG1` is not.

## 4. TEE loader recheck

### 4.1 Descriptor construction

The `utpa2k_stock.ko` loader region at `.text+0x46FCB8..0x46FCF4` constructs the command-3 descriptor:

```text
46fcb8  str     r6, [sp, #4]          ; +0x00 = 0
46fcbc  bl      HAL_AUDIO_GetDspMadBaseAddr
46fcc0  str     r0, [sp, #8]          ; +0x04 = B low word
46fcc4  mov     r1, #0x700000
46fccc  str     r1, [sp, #0xc]         ; +0x08 = 0x700000
46fcd8  ldr     r0, [r0, #0x4d0]      ; selector
46fce0  cmp     r0, #4
46fce4  str     r5, [sp, #0x10]       ; +0x0c = selected image size
46fce8  movwne  r0, #3
46fcec  str     r0, [sp, #0x1c]       ; +0x18 = 3 or 4
46fcf0  mov     r0, #3
46fcf4  bl      MDrv_AUDIO_TEE_R_Send_Cmd
```

The descriptor therefore contains the same conditional image type discriminator previously identified, but the caller's branch alone does not prove that TEE is active.

### 4.2 Conditional dispatch and external boundary

`MDrv_AUDIO_TEE_R_Send_Cmd` at `.text+0x422294` checks `AudioVars+0x24F0`:

```text
4222c4  ldr     r1, [r0, #0x24f0]
4222cc  cmp     r1, #1
4222d4  mvn     r0, #0
4222d8  cmp     r1, #2
4222e0  ...
4222ec  b       MDrv_AUDIO_OPTEE_R_Send_Cmd
```

The OPTEE wrapper's command-3 case stores the descriptor pointer and length into the TEEC operation (`descriptor`, `0x1C`, parameter type `7`) and calls `MDrv_SYS_TEEC_InvokeCmd`. The local implementation reaches version-3 conversion helpers, while the actual command invocation is represented by unresolved external symbols including:

```text
TEEC_InvokeCommand       relocation at .text+0x1AD04
tee_client_invoke_func   relocations at .text+0x1AD5C and .text+0x1AF00
```

No secure-world implementation is present in the TD98 corpus. Consequently, the static evidence proves a conditional host-side descriptor/OPTEE dispatch, not the secure-side address translation, image placement, or DSP execution base.

## 5. Selector evidence from the archive

The targeted `CheckHashkey` trajectory is consistent with the current TD98 findings:

- `MDrv_AUDIO_CheckHashkey` writes its computed selector to `AudioVars+0x4D0` at `.text+0x424870`.
- The stock code contains local assignments for selector values `0`, `2`, `3`, and `4`; the value-4 paths include AUTH checks associated with IP values `0x54`, `0x08`, `0x66`, `0x09`, `0x0A`, and `0x7D`.
- The finalization path checks the hash result before the commit, and the `+0x503` latch can cause an early return.
- The actual AUTH returns, latch state, initialization state, and final live selector are not recoverable from static code alone.

The loader's `AudioVars+0x4D0 == 4` test therefore remains a conditional image-selection fact, not proof that the MS12V22 image was active on a particular run.

## 6. Effect on the current causal matrix

| Edge | Result after archive revalidation | Confidence / remaining limit |
|---|---|---|
| `0x13B` → mmap name | `0x13B - 0xF7 = 0x44`; `_aszMmapLayoutString[0x44] = MI_MAD_ADV_BUF` | **VERIFIED** in current module and extracted config |
| mmap layout → `B` | `MI_SYS_GetMmapLayout(0x13B)` returns the layout pair; caller passes it to setter | **VERIFIED** producer path |
| numeric `B` | `0x22500000`, length `0x1B00000`, `CMA_HID=0` in TD98 extracted config | **VERIFIED for this config/init path**, not universal allocation behavior |
| setter normalization | low word `& ~0xF`, high word zero, stored at `AV+0xA0/A4` | **VERIFIED** |
| SHM33 arithmetic | `P_shm + 0x1044 + 0x3C0*slot`, slots 0/1 | **VERIFIED** |
| `0x23302044/0x23302404` | arithmetic candidates before `MsOS_PA2KSEG1` return translation | **VERIFIED arithmetic; live CPU address not closed** |
| TEE descriptor | B low, `0x700000`, image size, type 3/4, command 3 | **VERIFIED** |
| TEE active path | only if `AV+0x24F0 == 2` | **CONDITIONAL** |
| secure translation / DSP PC | unresolved at external TEE and hardware mapping boundary | **BLOCKED** |

## 7. Final static boundary

The archive review materially improves the provenance chain but does not make the DSP execution path complete. The strongest defensible chain is now:

```text
MMAP_MI.h: MI_MAD_ADV_BUF_ADR = 0x22500000
    -> MI_SYS_GetMmapLayout(0x13B)
    -> layout[0x10] passed to MDrv_AUDIO_SetDspBaseAddr(2, ...)
    -> HAL_AUDIO_SetDspBaseAddr: B & ~0xF, high = 0
    -> HAL_AUDIO_GetDspMadBaseAddr(2) + 0xE01000
    -> MsOS_PA2KSEG1(...) -> g_virDecR2shm
    -> SHM selector 0x33 stores at +0x1044 + 0x3C0*slot
```

The following remain outside static closure:

- whether the mapped `P_shm` is numerically equal to `0x23301000` on the active configuration;
- the live values of SHM33/SHM38/SHM4D and the DSP-visible alias;
- the actual value of `AudioVars+0x4D0` and the selected image at runtime;
- the secure-side handling of TEE command 3;
- the relationship between the host base, the DEC image, and a DSP PC;
- the later consumer and physical transport of the SND destination.

No runtime conclusion is drawn from this pass.
