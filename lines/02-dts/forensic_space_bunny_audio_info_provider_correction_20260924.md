# SPACE BUNNY — AUDIO-INFO PROVIDER / SND TAIL CORRECTION

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device/ADB, patch, flash, or module changes  
**TCL/T615:** not used  
**Preservation:** new correction artifact only

## Correction

The earlier host-SND bridge statement that the `g_FuncPrt_Hal_GetAudioInfo2` geometry provider was necessarily external is too broad. The current `utpa2k_stock.ko` contains and installs a local provider.

## 1. Function-pointer installation

`HAL_MAD_Init` explicitly writes:

```text
45c4e4  movw    r0, g_FuncPrt_Hal_GetAudioInfo2
45c4e8  movw    r1, HAL_MAD_GetAudioInfo2
45c4ec  movt    r0, g_FuncPrt_Hal_GetAudioInfo2
45c4f0  movt    r1, HAL_MAD_GetAudioInfo2
45c4f4  str     r1, [r0]
```

`MDrv_AUDIO_Free` later clears the pointer at `0x427020`.

`MDrv_AUDIO_GetAudioInfo2` at `0x427910` is a wrapper that dispatches through the pointer. In the normal initialized module path, the installed implementation is the local `HAL_MAD_GetAudioInfo2@0x45E34C`.

## 2. Cases used by the SND log monitor

The local `HAL_MAD_GetAudioInfo2` jump table maps:

```text
case 0x68 -> HAL_SND_R2_Get_SHM_INFO(0x4A, channel)
case 0x69 -> HAL_SND_R2_Get_SHM_INFO(0x4B, channel)
case 0x6A -> HAL_SND_R2_Get_SHM_INFO(0x4C, channel)
```

The SND `Get_SHM_INFO` table maps:

```text
0x4A -> g_virSndR2shm + 0x890
0x4B -> g_virSndR2shm + 0x894
0x4C -> default/return path
```

For the implemented cases, the function loads the value, flushes shared memory, and returns the word. The default path returns the current error/default value; it is not an external geometry computation.

## 3. Consequence for `MDrv_AUDIO_Dump_SNDR2_Log_Monitor`

The log monitor's apparent geometry is now bounded locally:

```text
A = value(g_virSndR2shm + 0x890)
B = value(g_virSndR2shm + 0x894)
SND_log_base = DSP_MAD2 + 0x700000 + A
observed_end = SND_log_base + B
```

The monitor then maps the resulting range and writes a debug file.

The `g_virSndR2shm` initialization clears `0x8B8` bytes, so `+0x890` and `+0x894` are the final words of the host-visible SND control block. The ordinary `HAL_SND_R2_Set_SHM_PARAM` handlers only reach offsets through approximately `+0x348`; they do not write these tail fields. The tail values are therefore DSP/shared-memory-populated state in the normal path, not values invented by an external host geometry provider.

## 4. What this closes and what it does not

This closes the provenance of the host monitor’s numeric offsets:

```text
GetAudioInfo2(0x68/0x69)
    -> local HAL_MAD_GetAudioInfo2
    -> local HAL_SND_R2_Get_SHM_INFO
    -> g_virSndR2shm tail fields +0x890/+0x894
```

It does **not** prove that these values are pointers to `0xD72000`, nor that the host mapping and SND ring alias. It only proves that the monitor reads a DSP-populated tail of the SND control block and maps a computed log range.

## 5. Corrected host-bridge statement

Replace:

```text
The geometry provider is outside the module.
```

with:

```text
The normal function-pointer provider is local HAL_MAD_GetAudioInfo2; the unresolved boundary is the producer/semantics of the SND tail fields +0x890/+0x894 and their relationship to the firmware ring.
```

The nearest hardware boundary remains IDMA/MIO and the external ring consumer.

No existing artifact was modified.
