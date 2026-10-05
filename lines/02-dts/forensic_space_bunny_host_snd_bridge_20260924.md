# SPACE BUNNY — HOST SND SHM / RING BRIDGE AUDIT

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device, ADB, playback, breakpoint, runtime memory read, patch, or flash  
**TCL/T615:** not used  
**Preservation:** new artifact only; no existing report, binary, Ghidra/ReKo project, or raw JSONL was modified

## Verdict

There is **no statically proven pointer or descriptor alias** between the host-side `g_virSndR2shm` control area and the SND firmware runtime ring at `0xD72000`.

What is proven is narrower and more precise:

```text
SND image base = B + 0x700000
reserved SND image window = [SND+0xA000, SND+0xAA00)
g_virSndR2shm = MsOS_PA2KSEG1(B + 0x70A000)
```

The host control area occupies a statically reserved hole in the SND image head. The `0xD72000` ring belongs to the SND image/runtime domain. No code in `utpa2k_stock.ko` constructs `0xD72000` or stores that address into the SHM.

## 1. `g_virSndR2shm` construction

The same construction is present at ten host-module sites, including:

```text
HAL_SND_R2_init_SHM_param
HAL_SND_R2_Set_SHM_PARAM
HAL_SND_R2_Get_SHM_PARAM
HAL_SND_R2_Get_SHM_INFO
HAL_SND_R2_Get_SIGNED_SHM_INFO
HAL_SND_R2_Set_SHM_COMMOM_PARAM
HAL_SND_R2_Get_SHM_COMMOM_PARAM
HAL_SND_R2_Get_SHM_INFO_64bit
HAL_AUR2_BackupShareMemory
HAL_AUR2_RestoreShareMemory
```

Representative code:

```text
4582c4  bl      HAL_AUDIO_GetDspMadBaseAddr(2)
4582c8  movw    r2, #0xa000
4582cc  movt    r2, #0x70          ; 0x70A000
4582d0  adds    r0, r0, r2
4582d8  bl      MsOS_PA2KSEG1
4582e4  str     r0, [g_virSndR2shm]
```

`HAL_AUDIO_GetDspMadBaseAddr(2)` reads the window-2 base from `AudioVars+0xA0/+0xA4`. The host module therefore proves a mapped SND control pointer, not a flat numeric address equal to `B+0x70A000`.

## 2. SND image hole and loader geometry

`HAL_AUDSP_DspLoadCode` copies the selected SND image at:

```text
46fd34  r7 = B + 0x700000
46fd48  source = selected mst_snd_r2 image
46fd4c  length = 0xA000
46fd50  memcpy
46fd64  clear offset = 0xAA00
46fd68  clear length = 0x6F5600
46fd70  memset
46fd88  source = selected SND image + 0xAA00
46fd8c  length = r8
46fd90  memcpy
```

The arithmetic is exact:

```text
0xAA00 + 0x6F5600 = 0x700000
```

The image loader leaves the interval:

```text
[SND+0xA000, SND+0xAA00)
```

uncopied and uncleared. `g_virSndR2shm` begins at `SND+0xA000`, and `HAL_SND_R2_init_SHM_param` clears `0x8B8` bytes there, leaving additional room inside the reserved interval.

This is a **containment/layout relationship**, not proof that the host pointer and the SND ring are the same object.

## 3. No pointer is written through `HAL_SND_R2_Set_SHM_PARAM`

All 23 call sites of `HAL_SND_R2_Set_SHM_PARAM` were inventoried. The arguments are small control values:

- selector/enum values;
- one-byte fields;
- boolean/clz-derived values;
- a bounded 24-bit composite built in the `0x46D930` region.

The successful handlers use ordinary loads/stores such as:

```text
[r6+0x70]
[r6+0x88]
[r6+0x90]
[r6+0x94]
[r6+0x98]
[r6+0xE0..0xF8]
[r6+0x118..0x174]
[r6+0x180..0x1A8]
[r6+0x328..0x348]
```

No call site supplies:

- `0xD72000`;
- a host virtual address;
- a DSP MAD address;
- a pointer to a ring manager;
- a pointer to a queue or DMA descriptor.

The function has no MMIO or DMA instruction in its write handlers. Successful paths converge on `MsOS_FlushMemory` after the shared-memory store.

## 4. Ring address is absent from the host module

A whole-file and instruction-construction search found no code construction of `0xD72000` in `utpa2k_stock.ko`. The few raw-byte occurrences are relocation metadata, not executable operands.

The same calibrated scan finds the host SND offset `0x70A000` and DEC SHM offset `0xE01000`, confirming that the absence of `0xD72000` is not a search-method failure.

The SND image's `0xD72000` ring constant must therefore be analyzed in the SND image, not inferred from host-module code.

## 5. Nearest hardware boundary

The closest host-side hardware mechanisms are:

```text
MDrv_Write_DSP_sram       IDMA/SRAM byte/word port
_s32AUDIOMutexIDMA        IDMA mutex
0x112A82                  IDMA control window
HAL_AUDSP_CheckSeIdmaReady
```

The MIO aperture is reached through `HAL_AUDIO_AbsWriteMaskByte/Reg`, using `_gMIO_MapBase` and register-address translation.

`HAL_SND_R2_EnableR2` itself only writes hardware control bytes/registers, including the `0x112CB2` and `0x163080` families; it does not dereference the SND SHM or a ring pointer.

No BDMA copy path was found in this audio loader chain. The only located `MDrv_BDMA_CopyHnd` caller belongs to flash/PCLRC initialization, not audio transport.

## 6. Debug monitor is not an alias proof

`MDrv_AUDIO_Dump_SNDR2_Log_Monitor` computes a mapped SND log region from `MDrv_AUDIO_GetAudioInfo2` selectors `0x68`, `0x69`, and `0xA`/`0x6A` and `B+0x700000`, then writes the result to a debug file.

This proves host observability of a mapped SND R2 region. It does not prove that the monitor's address is the firmware ring at `0xD72000`, because the geometry producer is behind the external function pointer `g_FuncPrt_Hal_GetAudioInfo2`.

## 7. Corrected domain boundary

The following must remain separate:

```text
B / AudioVars+0xA0       DSP base/window domain
MsOS_PA2KSEG1(...)       CPU mapping result
g_virSndR2shm             mapped host control pointer
SND+0xA000               reserved image/control window
0xD72000                 SND runtime ring constant
0xB00008xx               SND mailbox/status family
IDMA/MIO                 hardware access boundary
```

The old host/SND ring address references must not be relocated into `utpa2k_stock.ko`; they belong to the separate SND image analysis.

## 8. Remaining gaps

Static evidence still does not prove:

- whether `0xD72000` is `B+0xD72000`, an SND-local address, or another translated form;
- whether the SND ring has AC3 and DTS aliases;
- the identity of the external `g_FuncPrt_Hal_GetAudioInfo2` geometry provider;
- the hardware/DSP consumer that drains the ring;
- the physical SPDIF endpoint.

The host-side bridge is now bounded at **mapped SND control region + IDMA/MIO hardware boundary**, not at a proven ring pointer or DMA descriptor.

No previous artifact was modified.
