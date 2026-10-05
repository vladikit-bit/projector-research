# SPACE BUNNY — DTS FOLLOW-UP STATIC PASS

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device/ADB, playback, breakpoint, memory read, patch, flash, or module reload  
**TCL/T615:** not used  
**Preservation:** new report only; no prior artifact was modified

## Scope and limitation

This follow-up directly rechecks the current `utpa2k_stock.ko`, DEC listing/raw image, standard/MS12 SND images, and selected driver/library artifacts. The intended parallel agents were rate-limited, so the findings below are bounded direct raw checks rather than a claim of an exhaustive whole-corpus proof.

## 1. DTS state matrix: verified code conditions

### `CheckHashkey` final commit

`MDrv_AUDIO_CheckHashkey` ends with the following stores:

```text
0x424870  [AudioVars+0x4D0] = r6
0x424874  [AudioVars+0x4D4] = r5
0x424878  [AudioVars+0x4D8] = r8
0x42487C  [AudioVars+0x43E] = r7
0x424880  [AudioVars+0x43D] = sl
```

The store is reached only after:

```text
0x42485C  HAL_AUDIO_CheckHashkeyDone
0x424860  cmn r0,#0x16
0x424864  beq failure-path
```

and a non-null AudioVars object. The already-latched path at `+0x503` can return before recomputation.

The raw assignment contexts verify the following `r6` selector values:

| `r6` / `AV+0x4D0` | AUTH/check contexts observed |
|---:|---|
| `0` | fallback/default paths |
| `2` | IP `0x50`, `0x73` paths |
| `3` | IP `0x52`, `0x75`, `0x51`, `0x74`, `0x79` paths |
| `4` | IP `0x54`, `0x08`, `0x66`, `0x09`, `0x0A`, `0x7D` paths |

`r5` is observed with values `8` and `9`; `r8` is observed with values `0`, `1`, `2`, and `3`. The exact semantic names of all `r5/r8` branches are not fully rederived in this bounded pass. Their final storage and the selector conditions above are static facts.

### `SetSystem2` DTS-relevant reads

`HAL_AUDIO_SetSystem2` reads:

```text
0x44D144  [AudioVars+0x4D0]
0x44D148  cmp selector, #4
0x44D150  mvneq r4, #0x7f
...
0x44D1DC  [AudioVars+0x4D8]
0x44D1E4  cmp state, #3
0x44D1EC  movweq r1, #0x97
0x44D1F0  HAL_AUDIO_AbsWriteByte
```

Thus the code-level engine distinction is:

```text
AV+0x4D8 == 3 -> command value 0x97
otherwise     -> command value 0x04
```

`AV+0x4D0 == 4` changes the `r4` control value path; non-4 sets `r4=0x7F` in the shown branch. The function has a 0x1E-entry switch on `r7-1`, with output values including `0`, `3`, `4`, `6`, `7`, `8`, `9`, `0xA`, `0xB`, `0xC`, `0xD`, `0xE`, `0xF`, `0x10`, `0x13`, `0x14`, `0x15`, `0x16`, and `0x17`. These are system-command branches, not all DTS license states.

A concrete caller path in `HAL_MAD_SetAudioParam2` is:

```text
if input == 0x200:
    r7 = 0x0C
else:
    r7 = 0xFF
r0 = decoder/class value
HAL_AUDIO_SetSystem2(r0, r7)
```

The return value is checked against `1` before later state updates.

### SHM38 / SHM4D / SHM33

The static readers remain separated:

- DTS `GetAudioInfo2`/decoder-type paths read SHM38 and related AV state.
- AC3 latch-3 paths read SHM33 and SHM4D.
- The current host `Set_SHM_PARAM` writers are scalar/control writers; no live DTS acceptance value is inferable without runtime state.

The code conditions are therefore known, but the actual values of `AV+0x4D0`, `AV+0x4D8`, SHM38, and the latches remain runtime-dependent.

## 2. F8/FC and `R+0x174`: bounded direct result

The known DEC descriptor path remains:

```text
F_45DE0:
    D+F8 = *(arg4+0)
    D+FC = *(arg4+4)
    clear each 0x200 bytes

F_45E52:
    writes header/transformed words through F8/FC

F_4618E:
    later zero-fill and local processing
```

A full raw scan of `dec32_clean.txt` finds many unrelated uses of byte offsets `0xF8/0xFC`; offset text alone cannot establish object identity. The known descriptor block is the only currently proven F8/FC setup path.

`R+0x174` has independent state writers and consumers (`F_029867`, `F_28A33`, rotation/accumulation regions). No direct store of `D+F8`, `D+FC`, or their pointees into `R+0x174` was found in the bounded alias-normalized search.

Current result:

```text
F8/FC -> R+0x174: not found
R+0x174 -> F_28A33: proven
object identity between them: unresolved
```

This is a bounded negative result, not a claim that every indirect/data-only path in the entire image has been mathematically enumerated.

## 3. Standard DTS manager: direct raw comparison

The standard SND image contains distinct construction roots:

```text
standard AC3 manager = 0x4C6380
standard DTS manager = 0x4D561C
```

At the corresponding raw construction site, the MS12 bytes use `0x4E0000 + 0x6AD4 = 0x4E6AD4`; the standard bytes use `0x4C0000 + 0x6380 = 0x4C6380`. A separate standard DTS construction uses `0x4D0000 + 0x561C = 0x4D561C`.

No equivalent initialization call for standard `0x4D561C` was found in the bounded direct search. Its base/limit/span remain unresolved.

## 4. External consumer search

The corpus-wide raw search found no new executable consumer that reads the MS12 ring payload and advances its read cursor.

`spdif_audio_investigation/kmods/dtv_driver.ko` contains many textual `0xD72000`/mailbox byte hits, but the inspected hits are in `.symtab`, `.rel.text`, or `.rel.ARM.exidx` metadata rather than direct executable constants. Its relevant exported functions are audio-capture/PCM paths such as `MTGADEC_MTAUD_Record2PCM_read`, which call capture APIs and copy to user buffers; they are not a proven SND ring drainer.

`libmi3.so`/related libraries contain mailbox constants in data/relocation material but no independently verified ring consumer in the inspected symbols.

The nearest verified boundary remains:

```text
SND firmware ring
 -> 0xB00008xx mailbox/status handshake
 -> external IDMA/MIO/hardware agent not identified in the corpus
```

## 5. Host audio-info provider correction

The normal `g_FuncPrt_Hal_GetAudioInfo2` provider is local, not external:

```text
HAL_MAD_Init:
0x45C4E4..0x45C4F4
    g_FuncPrt_Hal_GetAudioInfo2 = HAL_MAD_GetAudioInfo2
```

The local jump table is based at `0x458E88` (not `0x458E8C`):

```text
GetAudioInfo2 0x68 -> SND info 0x4A -> g_virSndR2shm+0x88C
GetAudioInfo2 0x69 -> SND info 0x4B -> g_virSndR2shm+0x890
GetAudioInfo2 0x6A -> SND info 0x4C -> g_virSndR2shm+0x894
```

`HAL_SND_R2_Get_SHM_INFO` flushes and loads the selected word. These fields are inside the `0x8B8` initialized SND SHM block. The host setter does not write them; their producer is the SND/DSP side.

`MDrv_AUDIO_Dump_SNDR2_Log_Monitor` therefore computes a local log geometry from three firmware-populated words:

```text
A   = *(g_virSndR2shm+0x88C)
lim = *(g_virSndR2shm+0x890)
len = *(g_virSndR2shm+0x894)
mapped region = DSP_MAD2 + A + 0x700000 ... + lim
```

This closes the provider provenance, but not the numeric values or direct alias to `0xD72000`.

## 6. Net DTS status

| Question | Result |
|---|---|
| DTS selector conditions | Code branches verified; live values unknown |
| `AV+0x4D8` engine choice | `0x97` iff state 3, otherwise `0x04` |
| SHM38/SHM33 live values | Not statically knowable |
| F8/FC → `R+0x174` | Not found in bounded direct search |
| Standard DTS manager | Root `0x4D561C` found; initialization unresolved |
| MS12 ring topology | Shared `0x4E6AD4`; standard roots separate |
| SND firmware payload consumer | Not found; mailbox boundary remains |
| Host audio-info geometry provider | Local provider and SHM tail fields resolved |
| Physical transport | Still unresolved |

The static audit has narrowed the DTS uncertainty substantially, but it still does not identify the final hardware consumer or prove the complete F8/FC-to-transport edge.
