# SPACE BUNNY — AUDIO-INFO PROVIDER CORRECTION V2

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device/ADB, patch, flash, or module changes  
**TCL/T615:** not used  
**Preservation:** new correction artifact; previous reports were not edited

## Why this correction exists

The first host-bridge correction used the wrong base for the `HAL_SND_R2_Get_SHM_INFO` jump table. The `add r2,pc,#0` instruction is at `.text+0x458E80`, so ARM PC semantics make the table base `.text+0x458E88`, not `0x458E8C`.

## 1. Correct case table

The table at `0x458E88` resolves:

```text
SND info 0x49 -> handler 0x458F4C -> shm+0x858
SND info 0x4A -> handler 0x458FA4 -> shm+0x88C
SND info 0x4B -> handler 0x458FB0 -> shm+0x890
SND info 0x4C -> handler 0x458FB8 -> shm+0x894
```

The handlers converge at:

```text
459038  bl      MsOS_FlushMemory
459048  ldr     r6, [r6]
459074  mov     r0, r6
```

Thus `HAL_SND_R2_Get_SHM_INFO` returns the 32-bit value from the selected SHM tail field.

## 2. Correct `GetAudioInfo2` mapping

`HAL_MAD_GetAudioInfo2` cases used by the SND log monitor are:

| `GetAudioInfo2` selector | SND info selector | SHM field |
|---:|---:|---:|
| `0x68` | `0x4A` | `g_virSndR2shm+0x88C` |
| `0x69` | `0x4B` | `g_virSndR2shm+0x890` |
| `0x6A` | `0x4C` | `g_virSndR2shm+0x894` |

The previous v1 correction’s `0x4A -> +0x890`, `0x4B -> +0x894`, `0x4C -> default` mapping is rejected.

## 3. Correct descriptor bridge

`MDrv_AUDIO_Dump_SNDR2_Log_Monitor` now has a statically closed local geometry chain:

```text
A    = *(u32 *)(g_virSndR2shm + 0x88C)
lim  = *(u32 *)(g_virSndR2shm + 0x890)
len  = *(u32 *)(g_virSndR2shm + 0x894)

ring/log base = DSP_MAD2 + A + 0x700000
```

The calls are local through:

```text
MDrv_AUDIO_GetAudioInfo2
 -> g_FuncPrt_Hal_GetAudioInfo2
 -> HAL_MAD_GetAudioInfo2
 -> HAL_SND_R2_Get_SHM_INFO
```

`HAL_MAD_Init` installs the local provider at `0x45C4E4`:

```text
g_FuncPrt_Hal_GetAudioInfo2 = HAL_MAD_GetAudioInfo2
```

This is a descriptor/value bridge, not a direct host pointer alias to `0xD72000`.

## 4. Initialization correction

`HAL_SND_R2_init_SHM_param` clears `0x8B8` bytes. Therefore:

```text
0x88C < 0x8B8
0x890 < 0x8B8
0x894 < 0x8B8
```

All three fields are inside the zero-initialized SHM block. The previous statement that they were “past” the `0x8B8` clear was arithmetically wrong.

The ordinary host `HAL_SND_R2_Set_SHM_PARAM` handlers reach only approximately `+0x348`, so they do not write these tail fields. They are populated after initialization by the SND/DSP side or another shared-memory producer.

## 5. What this proves

The host-side SND log monitor does not depend on an unexplained external geometry function in the normal initialized module. It reads three firmware-populated SHM words and uses them to construct a mapped log range.

This is a stronger bridge than the earlier “external provider” description, but it still does not prove:

```text
the fields contain 0xD72000
the fields are absolute pointers
the host mapping aliases the SND ring
the physical consumer is identified
```

If the SND runtime ring is `0xD72000` and the image base arithmetic is `B+0x700000`, the implied offset would be `0x672000`; the actual contents of the three SHM words remain firmware/runtime state.

## 6. Corrected status

| Claim | Status |
|---|---|
| Local `HAL_MAD_GetAudioInfo2` provider | **Verified** |
| Cases 0x68/0x69/0x6A | **Verified** |
| SHM fields +0x88C/+0x890/+0x894 | **Verified** |
| Fields are inside `0x8B8` clear | **Verified; previous “past” statement rejected** |
| Firmware writes descriptor values | **Strongly supported; producer still unresolved** |
| Direct alias to `0xD72000` | **Not proved** |
| External hardware consumer | **Still unresolved** |

This v2 correction supersedes the field mapping in `forensic_space_bunny_audio_info_provider_correction_20260924.md`.
