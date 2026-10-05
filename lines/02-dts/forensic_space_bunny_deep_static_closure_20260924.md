# SPACE BUNNY — DEEP STATIC CLOSURE

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no ADB, device connection, playback, breakpoint, runtime memory read, module reload, reboot, settings change, patch, firmware write, or flash  
**TCL/T615:** not used  
**Preservation:** this is a new report; no previous report, raw JSONL, firmware binary, Ghidra project, ReKo project, SLEIGH file, or historical archive file was modified

## Executive conclusion

This pass materially changes the static confidence in several places and corrects two errors in the preceding continuation report.

### Newly closed or upgraded

1. **AEON `0x20` stack family and `r11` restoration are now proven by ReKo plus raw-byte cross-check.**
   - `bt.swst`/`bt.lwst` are real stack-slot store/load operations, not empty `trap/nop/inst16_4` artifacts.
   - `0x3847E` restores `r11` from its saved frame slot on every normal return.
   - On the traced `F_03DF03` path, `U = V_in` and `A108 = V_in+0x7C4`.

2. **MS12 and standard SND images have different ring topology.**
   - MS12 AC3 and DTS both use the manager rooted at `0x4E6AD4`, base `0xD72000`, limit `0xDD8000`, span `0x66000`.
   - Standard AC3 and DTS use different manager roots (`0x4C6380` and `0x4D561C`); the DTS standard manager’s initialization remains unresolved.
   - This is a structural behavioral difference, not yet a proven DTS root cause.

3. **The host SND SHM is a reserved control window, not a proven ring alias.**
   - `g_virSndR2shm = MsOS_PA2KSEG1(B+0x70A000)`.
   - The loader leaves `[SND+0xA000,SND+0xAA00)` untouched for this host-owned window.
   - No host `Set_SHM_PARAM` call writes a pointer, ring address, host VA, or DMA descriptor into that SHM.
   - No `0xD72000` construction exists in `utpa2k_stock.ko`.

4. **The SPDIF/NPCM monitor is a debug dump path, not a playback consumer.**
   - It observes DSP SRAM/IDMA state and writes a file from `DSP_MAD1+0x672000`.
   - It does not establish the normal transport consumer or physical SPDIF endpoint.

### Corrected errors

- The earlier statement that MS12 loader `r8=0x176930` was wrong. The current raw instruction sequence includes `movteq r8,#0x1B`; the correct value is **`0x1B6930`**.
- Therefore the claimed MS12 `0x40000` copy shortfall and omitted-tail theory are rejected.
- “F8/FC have exactly two writers” was too broad. `F_45DE0` establishes the pointers, while the `F_45E52` block writes the first header/transformed words through those pointers.

## 1. Evidence corpus and precedence

Primary raw artifacts:

| Artifact | SHA-256 |
|---|---|
| `aeon_validate/dec_work/dec_full.bin` | `530bfa684bc363e8513951b731cd42ccae97c96accef3c1d51eb431b61a53f5b` |
| `aeon_validate/r27_work/snd_full.bin` | `1da795fa47aec5aec2aa4cdf78d52028d62abe911f6993dac664206b9210c22d` |
| `patch_baseline/utpa2k_stock.ko` | `8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da` |
| `patch_baseline/mik_stock.ko` | `44b0b3b66b7255bd5c91c88fef7e9688737dee1c1a9c91667095e4a98d91e70d` |
| extracted `MMAP_MI.h` | `45d306cbf63efbda1181c4eccd0623cce49e3d491e7aa8ceeb741af9c2d4d756` |

The historical archive was mined across rendered transcript chunks, raw DB extracts, raw JSONL copies, and subagent trajectories. It is treated as evidence of raw bytes and historical claims, not as an oracle. The archive itself contains no `V_in`, `U`, `[B0+0x1414C]`, or `R+0x174` interpretation; those conclusions come from the later raw DEC analysis.

## 2. AEON decoder closure and `r11`

### 2.1 ReKo is an independent decoder

The corpus contains a previously validated ReKo AEON backend. ReKo renders the empty-SLEIGH `0x20` halfwords as:

```text
bt.swst <slot*4>(r1),rN
bt.lwst rN,<slot*4>(r1)
```

The recovered fields are:

```text
bits 15..10 = 0x20       stack-prefix opcode
bits 9..5   = register number
bits 4..1   = frame slot; byte offset = slot*4
bit 0       = 0 store, 1 load
```

The layout was checked against 1,987 ReKo renderings with zero mismatches and independently reproduced on a second AEON binary. The `0x21` family also agrees across ReKo, SLEIGH, and raw fields.

### 2.2 `0x3847E` frame

The helper has a `0x40` frame. ReKo/raw decoding gives the exact register/slot map:

```text
prologue stores:  r9,r10,r11,r12,r13,r14,r15,r16,r17,r18
epilogue loads:   r9,r18,r17,r16,r15,r14,r13,r12,r11,r10
```

The `r11` pair is:

```text
prologue: 0x817A  store r11 at [r1+0x34]
epilogue: 0x817B  load  r11 from [r1+0x34]
```

The loop writes at `0x384CB` and `0x384DE` are therefore temporary values inside the saved window. The epilogue load at `0x38510` dominates every normal return, followed by:

```text
0x38516  frame teardown
0x38519  jr r9
```

### 2.3 Dataflow consequence

`F_03DF03` loads:

```text
0x3DF2D  r11 = [B0+0x1414C] = V_in
```

then calls `0x3847E` twice and stores the restored value at `0x3DF79`. Thus, on the traced path:

```text
U = V_in
A108 = V_in + 0x7C4
```

This closes the former U3/r11 ambiguity. It does **not** identify the constructor or lifetime of `V_in`; only the value’s preservation and expression are now proven.

The standard-image counterpart at `0x43275` has the same structure and corroborates the result, but no semantic transfer is needed.

## 3. DEC descriptor and `R+0x174`

### 3.1 `F8/FC` pointer establishment

`F_45DE0` establishes two `0x200`-byte buffers:

```text
0x45E17  [D+0xF8] = *(arg4+0)
0x45E21  clear(F8,0,0x200)
0x45E2F  [D+0xFC] = *(arg4+4)
0x45E39  clear(FC,0,0x200)
```

The descriptor itself is cleared for `0x174` bytes. The fourth-argument source is conditionally:

```text
B0+0x14E40 / B0+0x14E44
```

### 3.2 Additional first-word writes

The `F_45E52` block reads the pointers and writes the staged header words:

```text
0x45EFB  load F8
0x45EFF  load FC
0x45F14  [F8+index*4] <- ctx+0x16C
0x45F20  [FC+index*4] <- ctx+0x16E
0x45F2C  [F8+4]       <- ctx+0x170
0x45F34  [FC+4]       <- ctx+0x172
```

It then performs the transformation loop through `0x45FBB`. Therefore the accurate statement is:

```text
F_45DE0 establishes the pointer fields;
F_45E52 writes through the pointers;
F_4618E performs a later local processing pass.
```

The absence of a direct `F8/FC -> R+0x174` publication remains unchanged.

### 3.3 `R+0x174`

`F_029867` reads `S=*(R+0x174)` and later uses `S` in the `F_28A33` path. Multiple state writers exist, including rotation/accumulation and reset paths, but no writer receives `D+F8/FC`.

The current static statement is:

```text
R+0x174 consumer path: proven
R+0x174 object provenance: unresolved
F8/FC -> R+0x174: not found
```

## 4. Host loader and address domains

### 4.1 DEC/SND image placement

For the direct non-TEE path:

```text
B = HAL_AUDIO_GetDspMadBaseAddr(2)
dst = MsOS_PA2KSEG1(B)
memset(dst, 0, 0x700000)
dst = MsOS_PA2KSEG1(B)
memcpy(dst, selected_DEC_image, selected_DEC_size)
```

The SND image is copied at `B+0x700000`, and `HAL_SND_R2_init_SHM_param` maps `B+0x70A000` into `g_virSndR2shm`.

### 4.2 Correct loader lengths

Current raw bytes prove:

```text
standard r8 = 0x173F48 = 0x17E948 - 0xAA00
MS12     r8 = 0x1B6930 = 0x1C1330 - 0xAA00
```

The `movt #0x17`, `movweq #0x6930`, `movteq #0x1B` sequence must be decoded together. The earlier `0x176930` reading omitted the final high-half write and is rejected.

The reserved window:

```text
SND+0xA000 .. SND+0xAA00
```

is not copied or cleared by the image-copy sequence and is occupied by the host SND control mapping. This is strong structural evidence of a reserved control hole, not proof of design documentation.

### 4.3 Mapping-domain rule

`MsOS_PA2KSEG1` is table-based. It returns a mapping result derived from `mpool_info`, not a proven identity function. Therefore:

```text
0x23302044 / 0x23302404
```

remain pre-mapping arithmetic candidates for DEC SHM slots, not proven live CPU/DSP addresses.

## 5. SND ring topology

### 5.1 MS12 shared ring

The MS12 SND image constructs the same manager for AC3 and DTS:

```text
AC3 root = 0x4E0000 + 0x6AD4 = 0x4E6AD4
DTS root = 0x4E0000 + 0x6AD4 = 0x4E6AD4
```

The initializer fields are:

```text
base        0xD72000
limit       0xDD8000
span        0x66000
read cursor 0xD72000 on reset
write cursor0xD72000 on reset
```

`limit-base=span` exactly. MS12 AC3 and DTS records therefore compete for one cursor-managed window.

### 5.2 Standard separation

The standard image has distinct construction roots:

```text
standard AC3 manager = 0x4C6380
standard DTS manager = 0x4D561C
```

The DTS standard manager’s initialization/base is not closed by the current pass.

### 5.3 Consumer boundary

The ring subsystem appends records and updates write/fill state. No read-cursor payload load was found in the ring subsystem. The only observed cursor writes after initialization are reset-to-base operations. The external consumer is not present in the SND firmware image.

Mailbox/status addresses include:

```text
0xB000080C
0xB0000814
0xB000084C
0xB0000854
```

The strongest interpretation is an out-of-image hardware/DSP consumer. The exact DMA descriptor and physical output remain unresolved.

## 6. Host SND control plane versus ring

The host module does not construct `0xD72000` and does not store a ring pointer into `g_virSndR2shm`.

All 23 `HAL_SND_R2_Set_SHM_PARAM` call sites pass scalar/control values. No host VA, DSP MAD address, ring pointer, queue pointer, or DMA descriptor is written.

The nearest host hardware boundary is:

```text
MDrv_Write_DSP_sram / SE IDMA
_s32AUDIOMutexIDMA
0x112A82 control window
MIO aperture via AbsWriteMaskByte/Reg
```

BDMA is not in the audio loader path; the located BDMA caller belongs to flash/PCLRC initialization.

The host SND log monitor is observability only: it obtains offsets through an external function-pointer geometry provider and writes a debug file. It does not prove ring aliasing.

## 7. SPDIF monitor boundary

`MDrv_AUDIO_Dump_SpdifNpcm_Monitor` reads DSP SRAM address `0x1114` through the SE IDMA path and maps:

```text
DSP_MAD1 + 0x72000 + 0x600000
= DSP_MAD1 + 0x672000
```

It writes the observed range to a file under debug-state gates. This is not the normal SPDIF transport.

`MDrv_AUDIO_Dump_SNDR2_Log_Monitor` similarly observes mapped SND R2 log regions around `DSP_MAD2+0x700000`, again through a file dump.

Neither monitor is a playback consumer.

## 8. Current authoritative matrix

| Edge | Current status |
|---|---|
| `MI_MAD_ADV_BUF` → `B` | Verified for extracted TD98 config |
| `B` → host DSP base setter | Verified |
| Direct DEC image placement | Closed as `MsOS_PA2KSEG1(B)` |
| Direct SND image placement | Closed as mapped `B+0x700000` |
| `g_virSndR2shm` | Closed as mapped `B+0x70A000` control region |
| MS12/standard SND ring topology | **Closed: MS12 shared, standard separate manager roots** |
| Host SHM → SND ring pointer alias | **No alias proven; no pointer writes found** |
| SND firmware payload consumer | Not present in ring subsystem; external identity blocked |
| AEON 0x20 stack family | **Proven via ReKo/raw cross-check** |
| `0x3847E` return-visible `r11` | **Proven restored** |
| `U` / `A108` on traced path | **`U=V_in`, `A108=V_in+0x7C4`** |
| F8/FC pointer establishment | Closed |
| F8/FC first-word writes | Closed in `F_45E52` path |
| F8/FC → `R+0x174` | Not found |
| `R+0x174` consumer | Exists; object provenance unresolved |
| SPDIF monitor → physical transport | Monitor only; not transport |
| Secure TEE/DSP PC | Still blocked |
| Physical SPDIF endpoint | Still blocked |

## 9. Remaining static work

The highest-value remaining checks are:

1. Identify the constructor/lifetime of `V_in=[B0+0x1414C]`.
2. Resolve `F_3E961`’s conditional `r10` base identities.
3. Find the object/data path connecting `F8/FC` to `R+0x174`, if one exists.
4. Resolve the standard DTS manager initialization and prove whether standard AC3/DTS can ever converge.
5. Locate external hardware/DSP metadata or code that drains the MS12 ring.
6. Resolve the `g_FuncPrt_Hal_GetAudioInfo2` geometry provider used by host monitors.
7. Identify the secure-side TEE consumer and the actual DSP execution/image placement semantics.
8. Identify the physical SPDIF/DMA endpoint outside the firmware image.

## 10. Preservation and correction record

This report is new. Earlier reports were not edited. Where an earlier report is superseded, the correction is explicit:

- `forensic_space_bunny_aeon_r11_closure_20260924.md` supersedes the earlier “r11 restoration inferred” verdict.
- `forensic_space_bunny_archive_corrections_20260924.md` supersedes the earlier `r8=0x176930` and “F8/FC exactly two writers” wording.
- `forensic_space_bunny_snd_ring_comparison_correction_20260924.md` records the raw-byte rejection of the MS12 copy-shortfall claim while retaining the verified ring-topology result.
- `forensic_space_bunny_host_snd_bridge_20260924.md` records the no-alias host SHM result.
- `forensic_space_bunny_spdif_monitor_static_20260924.md` records the debug-monitor boundary.

No runtime, device, TCL/T615, or binary-modification activity was used.
