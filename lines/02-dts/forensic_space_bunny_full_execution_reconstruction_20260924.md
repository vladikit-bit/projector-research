# SPACE BUNNY — FULL STATIC EXECUTION RECONSTRUCTION

**Date:** 2026-09-24  
**Mode:** static/corpus-only, autonomous forensic reconstruction  
**Runtime:** none; no ADB, device, playback, runtime memory, breakpoint, patch, flash, reboot, module reload, or configuration change  
**TCL/T615:** not used; no TD98 static question required a TCL differential reference  
**New artifact only:** no prior report, binary, raw log, Ghidra project, script, or hash was modified

## Executive Summary

This pass reconstructs the static execution/data-flow chain from the stock TD98 audio input through the furthest downstream software boundary currently visible in the corpus.

The main result is not a single proven “DTS license failure.” Static evidence proves several distinct codec-specific forks and two separate downstream data paths:

```text
audio_config.format
  -> parser-name selection
  -> codec ID 5 / 9
  -> MI decoder-system value 3 / 0xB
  -> SetSystem2 engine/latch configuration
  -> Decoder_Type acceptance query
  -> DEC descriptor/F45FD1/F4618E local processing
  -> a separate delayed S-buffer/output path
```

The first **intentional** AC3/DTS divergence is parser selection at `0x230B8 -> 0x230BC -> 0x28258 -> 0x230C0`. The first **stateful hardware-facing** divergence is `SetSystem2` at `0x44D1DC` versus `0x44D228`. The first **output-acceptance condition** divergence is the latch-specific `Decoder_Type` path at `0x45F230` versus `0x45F6BC`.

A separate, later frame-field divergence is directly visible in MS12V22 SND:

```text
AC3: data type 0x0001 at 0x1EDBD..0x1EDBF, length<<3 at 0x1EDC1
DTS: data type 0x000B at 0x1F8B1..0x1F8B4, literal burst field 0x3F80 at 0x1F8B6..0x1F8BA
```

These values enter a local SND output buffer and the generic copy helper `0x1078A9`; a physical SPDIF sink is not proven.

The DEC F8/FC path is separate:

```text
F_03E0DD -> F_45DE0 -> D+0xF8/0xFC -> F_45E52/F_4618E
```

The direct `F_4618E` call receives `D_B`, while the installer track is parameterized by unresolved post-helper register `P`. A delayed consumer at `F_029867` reads a separate pointer `S=*(R+0x174)`, recognizes IEC sync signatures, byte-swaps/copies through `0x28A33`, and continues through ring/cache and DMA helpers. No static edge proves that the F8/FC A/B arrays are `S`, enter that delayed path, or reach a physical transmitter.

The report therefore distinguishes:

```text
VERIFIED codec recognition and state forks
VERIFIED local frame-field construction
VERIFIED delayed output-buffer processing on a separate source pointer
UNPROVEN F8/FC -> delayed S/TX/physical-output identity
```

## 1. Corpus / Evidence Boundary

### Primary static evidence

| Area | Primary artifact |
|---|---|
| Userspace stream/parser path | `spdif_audio_investigation/libs/audio.primary.mt5889.so` |
| MI host layer | `spdif_audio_investigation/k_mik_decomp.c`, `k_Start.asm`, `k_MI_AUDIO_Start.asm` |
| MAPI/MDrv/HAL ARM path | `patch_baseline/utpa2k_stock.ko`, `spdif_audio_investigation/k_utpa2k_decomp.c`, `k_ut_setdecsys.c` |
| DEC DSP image | `aeon_validate/dec_work/dec_full.bin`, `dec32_clean.txt` |
| SND DSP image | `aeon_validate/r27_work/snd_full.bin`, Ghidra/Reko listings |
| Software TX monitor | `spdif_audio_investigation/libs/libutopia.so` |
| Historical captures | hypotheses only; never primary proof in this report |

The actual ELF symbol for the userspace decoder function is:

```text
_Z15mi_decoder_openP16mstar_stream_outP12audio_config
symbol value: 0x2F409; function start: 0x2F408
```

Some older dumps use a shifted `0x3F...` address. The report uses the primary ELF/raw address `0x2F408` unless explicitly marked as a shifted dump.

### Status vocabulary

- `VERIFIED`: direct bytes, instruction, and data flow establish the claim.
- `CONDITIONAL`: the branch exists, but a required runtime value/identity is unresolved.
- `UNPROVEN`: plausible interpretation without a complete edge.
- `BLOCKED`: a required source, target, or semantic edge is absent from the bounded corpus.
- `DISPROVEN`: primary evidence contradicts the claim.
- `COSMETIC/DIAGNOSTIC`: observed output does not feed the claimed control path.
- `CONTROL-FLOW RELEVANT`: the value changes a branch/state transition.
- `BUFFER-DATA RELEVANT`: the value changes data written or copied.
- `OUTPUT-ACCEPTANCE RELEVANT`: the value affects an acceptance/query branch, but physical transport is not proven.

Historical “AC3 works / DTS fails” observations are used only to select the comparison. They do not prove any static state value, branch, or output edge.

## 2. End-to-End Execution Graph

```text
INPUT / CONFIG
  audio_config.format
  [0x22D74 load fp; 0x22E98..0x22EA8 select format; 0x22EB2 store]
        |
        v
PARSER RECOGNITION
  0x230B8 load stream_out+0x180
  0x230BC utils_get_audio_parser_name
  0x28258 AC3: 0x09/0x0A -> "ac3"
          DTS: 0x0B/0x0C -> "dts"
  0x230C0..0x230C8 Parser_Open
  return <0 -> 0x22F12
        |
        v
CODEC-ID SELECTION
  mi_decoder_open @0x2F408
  AC3: 0x2F4E8 r1=5
  DTS: 0x2F528 r1=9
  0x2F52E store r1 -> stream_out+0xDC
  0x2F5D4 MI_AUDIO_Start handoff
        |
        v
MI / HOST START
  MI_AUDIO_Start @0xA437C
  codec store / mapper / decSys
  case 5  -> decSys 3   @0xA3568/0xA356C
  case 9  -> decSys 0xB @0xA3748/0xA374C
  start struct +0x10 = 0xFF @0xA44BC, copied @0xA47E0
  MApi_SetDecodeSystem @0xA4858
        |
        v
MAPI / MDrv / HAL
  _MApi_SetDecodeSystem first-argument and +0x10 gate
  MDrv_SetDecodeSystem @0x424E78
  CheckHashkey / mapped SE writes
  HAL_AUDIO_SetDecodeSystem @0x4407F4
  HAL_AUDIO_SetSystem2 @0x44D0B0
        |
        +--> DTS r7=0xB: engine 0x04 or 0x97, latch 0xA
        |
        +--> AC3 r7=3: engine r4|1, latch 3
        |
        v
DECODER-TYPE / ACCEPTANCE QUERY
  MI_AUDIO_GetCodecType @0xB3BE8
  GetAudioInfo2 type 0x30 @0xB3D74
  HAL_MAD_GetAudioInfo2 @0x45E34C
  latch 3  -> SHM33 / SHM4D path
  latch 0xA -> SHM38 / AV+0x4D8 range path
        |
        v
DEC DESCRIPTORS / LOCAL BUFFERS
  F_03E0DD
  0x1414C load / helper / A108 / table
  F_45DE0 -> D+0xF8/0xFC
  F_0411A6 -> F_45FD1 -> F_4618E
        |
        +--> local A/B transforms and sync writes
        |
        +--> no proven A/B -> S/queue/TX edge
        |
        v
SEPARATE DELAYED OUTPUT TRACK
  F_029867
  S=*(R+0x174)
  sync-signature recognition
  0x28A33 byte-order copy
  0x13602 / 0x13B08 ring/cache helpers
  F_28AFE -> F_14F63 / F_1476A
        |
        v
PHYSICAL OUTPUT
  no static edge to a named physical SPDIF sink
```

## 3. AC3 Working Path

The AC3 path below is a static path through the working control surface. Historical success is not re-proven here.

| Stage | AC3 static path | Status |
|---|---|---|
| Input format | `0x09000000` and `0x0A000000` map to parser name `ac3` at `0x28258/0x28290/0x28296/0x28298` | `VERIFIED` |
| Parser open | `0x230C0..0x230C8`; negative return branches to `0x22F12` | `VERIFIED` |
| Codec ID | `r1=5` at `0x2F4E8`; common store `0x2F52E` | `VERIFIED` |
| MI decoder system | case 5 writes `3` at `0xA3568/0xA356C` | `VERIFIED` |
| SetSystem2 | `r7=3` reaches `0x44D228`; engine value `r4|1`; latch `3` at `0x44D25C/0x44D358` | `VERIFIED` |
| Profile-dependent engine value | `r4=0` if `AV+0x4D0==4`, otherwise `0x7F`; raw AC3 value is `0x01` or `0x7F`, not an inferred `0x81` | `VERIFIED` |
| Decoder_Type arm | latch 3 -> `0x45F230`; SHM33 then SHM4D | `VERIFIED` |
| F45FD1 data-type branch | selector `0x400` -> `F45D07` with `r4=1`; data type `0x10C` | `VERIFIED` on branch |
| F4618E | local descriptor buffer processing; sync constants depend on `D+0x2C` and `D+0x30` | `VERIFIED` local behavior |
| SND AC3 frame field | `0x1EDBD..0x1EDBF` writes data type `0x0001`; `0x1EDC1` writes `length<<3`; Pa/Pb writes at `0x1EDA6/0x1EDAF`; copy at `0x1EDD6` | `VERIFIED` local output-buffer construction |
| Physical output | no direct static edge to a named physical sink | `UNPROVEN` |

The AC3 path is therefore statically complete through a local SND output-buffer copy, not through a proven physical-output endpoint.

## 4. DTS Failing Path

The DTS path is reconstructed in parallel. “Failing” means the historical comparison selected this branch; the static corpus does not identify one field as the proven cause of silence.

| Stage | DTS static path | Status |
|---|---|---|
| Input format | `0x0B000000` and `0x0C000000` map to parser name `dts` at `0x28276/0x2827C/0x282A2/0x2828C` | `VERIFIED` |
| Parser open | same `Parser_Open` return gate as AC3 | `VERIFIED` |
| Codec ID | `r1=9` at `0x2F528`; common store `0x2F52E` | `VERIFIED` |
| MI decoder system | case 9 writes `0xB` at `0xA3748/0xA374C` | `VERIFIED` |
| SetSystem2 | `r7=0xB` reaches `0x44D1DC`; engine `0x04`, or `0x97` if `AV+0x4D8==3`; latch `0xA` | `VERIFIED` |
| Decoder_Type arm | latch `0xA` -> `0x45F6BC`; reads SHM38; requires `AV+0x4D8∈{1,2,3}` and `SHM38∈[1,7]` | `VERIFIED` conditions; live values `BLOCKED` |
| F45FD1 data-type branch | selector `0x200` -> `F45D07` with `r4=0`; data type `0x0B` | `VERIFIED` on branch |
| F4618E | local descriptor processing and `7FFE/8001` or `1FFF/E800` branches | `VERIFIED` local behavior |
| SND DTS frame field | `0x1F8B1..0x1F8B4` writes `0x000B`; `0x1F8B6..0x1F8BA` writes `0x3F80`; Pa/Pb and `0x1078A9` copy path | `VERIFIED` local output-buffer construction |
| DEC F8/FC -> DTS payload | no direct edge from `D_P+0xF8/0xFC` to SND or `S` | `BLOCKED` |
| Physical output | no direct static edge to a named physical sink | `UNPROVEN` |

The DTS path has a real local frame-construction branch; static evidence does not prove that the compressed DTS payload reaches that branch or that the branch is rejected downstream.

## 5. First AC3/DTS Divergence

There are several firsts, and they must not be conflated.

### 5.1 Earliest intentional parser divergence — `VERIFIED`

```text
0x22D4C  adev_open_output_stream
0x22D74  load audio_config pointer
0x22E98..0x22EA8 select format field
0x22EB2  stream_out+0x180 = format
0x230B8  load format
0x230BC  utils_get_audio_parser_name
0x28258  AC3/DTS name split
0x230C0  Parser_Open
```

This is a real causal fork, but it is intended recognition, not a failure cause.

### 5.2 First codec-ID divergence — `VERIFIED`

```text
AC3: 0x2F4E8 movs r1,#5
DTS: 0x2F528 movs r1,#9
0x2F52E: store r1,[stream_out+0xDC]
0x2F5D4: MI_AUDIO_Start handoff
```

Classification: `CONTROL-FLOW RELEVANT`, intentional.

### 5.3 First stateful/hardware-facing divergence — `VERIFIED`

```text
DTS: SetSystem2 0x44D1DC, latch 0xA
AC3: SetSystem2 0x44D228, latch 3
```

Classification: `CONTROL-FLOW RELEVANT` and configuration/data-flow relevant. It is the earliest stateful split proven from static code. Its causal role in stock DTS silence remains `UNPROVEN`.

### 5.4 First output-acceptance query divergence — `VERIFIED CONDITIONS`

```text
AC3/latch 3: 0x45F230 -> SHM33, then SHM4D
DTS/latch 0xA: 0x45F6BC -> SHM38, AV+0x4D8 range, SHM38 range
```

Classification: `OUTPUT-ACCEPTANCE RELEVANT` for the query path; no direct physical-transport consequence is proven.

### 5.5 First codec-specific IEC frame-field divergence — `VERIFIED`

```text
AC3: SND +4 = 0x0001 at 0x1EDBD..0x1EDBF
DTS: SND +4 = 0x000B at 0x1F8B1..0x1F8B4
```

This is the strongest directly observed AC3/DTS output-preparation difference. It still does not prove that the bytes reach a physical sink or that the compressed DTS payload is correct.

### Failure-root-cause verdict

No single static root cause is proven. The earliest **verified** fork is parser/codec selection; the earliest **failure-relevant unresolved condition set** is the SetSystem2/Decoder_Type/live-handle state. A later F8/FC-to-output handoff remains blocked.

## 6. Host → DSP Control Flow

### 6.1 MI start argument provenance

`MI_AUDIO_Start` does not pass codec `5` or `9` as the first API argument.

Static flow:

```text
MI_AUDIO_Open:
  MApi_AUDIO_OpenDecodeSystem return
  [MI instance + 0xADC] = return       (0xA2AC0 in stock flow)

MI_AUDIO_Start:
  codec / stream parameters are processed
  start struct +0x10 = 0xFF           (0xA44BC)
  start struct copy                    (0xA47E0)
  r0 = [MI instance + 0xADC]           (0xA4850)
  MApi_AUDIO_SetDecodeSystem(handle,&struct) (0xA4858)
```

The `+0xADC` value is an OpenDecodeSystem return/handle, not a statically fixed codec number. The old statement that the first `_MApi` argument is simply codec `5` or `9` is `UNPROVEN` and is not used here.

### 6.2 `_MApi_AUDIO_SetDecodeSystem` gate

Primary stock raw control flow:

```text
0x3E534C  compare first argument with 5
0x3E5350  compare first argument with -1
0x3E5354  special path if 5/-1

otherwise:
0x3E53B8  load param2+0x10
0x3E53BC  reject 0xFF
0x3E53C0  reject 0
0x3E53C4..0x3E53C8 no-dispatch/rejection path
otherwise:
0x3E5448  AV[first_arg*0x28+0xFA] check
0x3E546C  MDrv_AUDIO_SetDecodeSystem dispatch
```

The source decompilation at shifted address `0x4028CC` agrees with the branch structure. The host `r4` and DEC `F_45FD1` `r4` are different ABIs and are not merged.

The start path statically supplies `+0x10=0xFF` for both codec values, but the first argument is a handle whose live value is not statically fixed. Therefore:

```text
MApi +0x10 gate:                 VERIFIED structure
AC3 vs DTS gate divergence:      CONDITIONAL/UNPROVEN
```

### 6.3 MDrv/HAL path

After accepted dispatch:

```text
MDrv_AUDIO_SetDecodeSystem @0x424E78
  -> CheckHashkey / mapped AUTH-related writes
  -> pFuncPtr_SetSystem
  -> HAL_AUDIO_SetDecodeSystem @0x4407F4
  -> HAL_AUDIO_SetSystem2 @0x44D0B0
  -> HAL_AUDIO_SPDIF_SetMode configuration call
```

`HAL_AUDIO_SPDIF_SetMode` does not statically branch on a codec-derived `eAudioSource` value. It loads mode from shared state (`g_AudioVars2+0x1C4`) before dispatch. A historical source=3/4 correlation is not a static AC3/DTS branch.

### 6.4 Utopia/ioctl boundary

The stock wrapper decompilation shows:

```text
MApi_AUDIO_OpenDecodeSystem:
  UtopiaOpen(0x80000034,...)
  UtopiaIoctl(_pInstantAudio,0x3F,...)

MApi_AUDIO_SetDecodeSystem:
  UtopiaOpen(0x80000034,...)
  UtopiaIoctl(_pInstantAudio,0x3C,...)
```

The exact payload propagation from the local stack/object fields through the kernel ioctl into final DSP/shared state is not closed by the checked static corpus.

```text
Utopia command existence:     VERIFIED
payload-to-DSP field edge:    BLOCKED
```

## 7. DTS Decoder Preconditions

| Condition/state | AC3 static value/path | DTS static value/path | Required for DTS? | Status |
|---|---|---|---|---|
| Input parser name | `ac3` for `0x09/0x0A` | `dts` for `0x0B/0x0C` | Yes, for DTS parser path | `VERIFIED` |
| Codec ID | `5` at `0x2F4E8` | `9` at `0x2F528` | Yes, for selected MI path | `VERIFIED` |
| MI decoder-system value | `3` | `0xB` | Yes, for corresponding SetSystem2 branch | `VERIFIED` |
| `_MApi` first argument | handle, not proven codec | handle, not proven codec | Must pass 5/-1 or +0x10 condition | `CONDITIONAL` |
| Start struct `+0x10` | statically `0xFF` | statically `0xFF` | Must satisfy branch or handle special case | `VERIFIED` source; live gate `BLOCKED` |
| `AV+0x4D8` | profile-dependent but not directly the SetSystem2 AC3 branch | `1/2/3` required by DTS Decoder_Type arm; `3` also selects `0x97` | Yes for DTS query path | `VERIFIED` condition; live value `BLOCKED` |
| `AV+0x4D0` | selects profile and AC3 engine byte | selects profile / image branch | Profile-dependent | `VERIFIED`; no fix proof |
| DTS decoder latch | `3` | `0xA` | Yes, for DTS Decoder_Type arm | `VERIFIED` |
| SHM33 / SHM4D | AC3 arm reads these | not the DTS arm | AC3-specific acceptance query | `VERIFIED` |
| SHM38 | not the AC3 arm | `1..7` required | Yes for DTS query success | `VERIFIED` condition; live value `BLOCKED` |
| `AV+0x524/0x528` | latch slot lookup | latch slot lookup | Required to select query table | `VERIFIED`; exact live slot `CONDITIONAL` |
| `AV+0x582` | no direct execution edge established here | no direct execution edge established here | Not claimed | `UNPROVEN` |
| `0x112E98/9A/9B` | AC3 SetSystem2 writes differ | DTS writes `0x04/0x97` and latch-related bytes | Yes for DSP-visible setup | `VERIFIED`; transport meaning not proven |
| F45FD1 selector | `0x400` branch -> `0x10C` | `0x200` branch -> `0x0B` | Branch-dependent | `VERIFIED`; live selector source conditional |
| SND frame data type | `0x0001` | `0x000B` | Yes for local output frame field | `VERIFIED` |
| A/B -> delayed S | not proven | not proven | Required for any downstream claim | `BLOCKED` |

## 8. Decoder_Type / SetSystem2 / SHM Flow

### 8.1 SetSystem2 codec-specific branches

```text
HAL_AUDIO_SetSystem2 @0x44D0B0
common engine base / profile read: 0x44D104..0x44D150

DTS r7=0xB:
  0x44D1DC
  reads AV+0x4D8
  writes 0x04, or 0x97 if AV+0x4D8==3
  stores decoder latch 0xA

AC3 r7=3:
  0x44D228
  writes r4|1
  r4=0 if AV+0x4D0==4, else 0x7F
  stores decoder latch 3
```

This is the earliest verified stateful AC3/DTS divergence. It is a configuration fork, not proof of failure.

### 8.2 Decoder_Type query

```text
MI_AUDIO_GetCodecType @0xB3BE8
  -> GetAudioInfo2 type 0x30 @0xB3D74
  -> HAL_MAD_GetAudioInfo2 @0x45E34C
```

AC3/latch 3:

```text
0x45F230
0x45F234..0x45F23C  SHM33
0x45F250..0x45F258  SHM4D
```

DTS/latch `0xA`:

```text
0x45F6BC
0x45F6C0..0x45F6C4  SHM38
0x45F6D0..0x45F6DC  AV+0x4D8 ∈ {1,2,3}
0x45F6E0..0x45F6E8  SHM38 ∈ [1,7]
```

The query is a real control/output-acceptance path. It is not a direct physical-output call.

### 8.3 SetMode correction

`HAL_AUDIO_SPDIF_SetMode @0x448A10` does not statically branch on the `eAudioSource` argument in the raw body. It loads mode from shared state and overwrites the source register before mode dispatch. Any AC3=3 / DTS=4 SetMode divergence is `UNPROVEN` and is not used as the first failure point.

## 9. F_03E0DD State Machine

The corrected symbolic state is:

```text
B0    = F_03E0DD entry r4
V_in  = *(B0+0x1414C)
U     = r11 at 0x3DF79
P     = post-helper r11 used at 0x3E19E
```

Relevant helper sequence:

```text
0x3DF03  helper entry with r3=B0
0x3DF2D  r11=V_in
0x3DF35  0x130D25(B0,0,0x16F00)
0x3DF3B  0x3847E(...)
0x3DF47  0x130D25(...)
0x3DF4D  0x3847E(...)
0x3DF59  0x130D25(V_in,0,0xAC6C)
0x3DF65  0x130D25(...)
0x3DF79  [B0+0x1414C]=r11
0x3E19E  r23=P+0x7C4
0x3E1A6  [B0+0xA108]=P+0x7C4
```

`0x3847E` has a reachable `r11` clobber path at `0x384CB/0x384DE`; the supplied `swst/inst16_4` semantics are incomplete. Therefore `P` is not promoted to `V_in` or `B0`.

The downstream construction is symbolic:

```text
D_P = P+0x14CCC
T_P = P+0x14E40
A_P = P+0x14EE8
B_P = P+0x156E8
```

## 10. 1414C Provenance

The normal fill at `0x3DF35` covers `[B0+0x1414C]`, but the fill is followed by helper-dependent source value `U` at `0x3DF79`. The possible post-helper values are:

```text
U/P = V_in       conditional
U/P = B0         conditional on epilogue restoration
U/P = another pointer / helper value   not excluded
```

The initial image flag at `0x1DC594` is initially zero, but the helper has a global-mode branch and later flag writers. Static source cannot select the live branch.

```text
1414C zero-filled:             CONDITIONAL on 0x130D25 mode
1414C final value:             OPAQUE
explicit V=B0 assignment:      NOT FOUND
explicit V_in constructor:     NOT FOUND
```

## 11. V / P / B0 Identity Matrix

| Object/address | Track | Merge condition | Status |
|---|---|---|---|
| `[B0+0x1414C]` | input field / V_in | none | `VERIFIED` field |
| `P+0x14CCC` | D_P | `P=B0` for D_P=D_B | `CONDITIONAL` |
| `P+0x14E40` | T_P | `P=B0` | `CONDITIONAL` |
| `P+0x14EE8` | A_P | `P=B0` | `CONDITIONAL` |
| `P+0x156E8` | B_P | `P=B0` | `CONDITIONAL` |
| `P+0x7C4` | A108 source candidate | `P=B0` | `CONDITIONAL` |
| F45FD1/F4618E input | D_B | direct call only | `VERIFIED` on direct call |
| F8/FC pointer installation | D_P | `D_P=D_B` | `MERGE CONDITIONAL` |
| cross-offset A/B coincidences | require explicit P-B0 delta | none proven | `UNPROVEN` |

No unconditional V/B merge is permitted.

## 12. A108 Last-Writer Matrix

| Reader/path | Last verified writer | Value | Status |
|---|---|---|---|
| `F_03E961` before `0x3EF64` F0411A6 | `0x3EA38` | `B0+0x7C4` | `VERIFIED` |
| Later F0411A6 after F03E0DD | `0x3E1A6` | `P+0x7C4` | `CONDITIONAL` |
| A108 after normal `0x3DF35` fill, before `0x3E1A6` | fill | `0` transiently | `CONDITIONAL` |
| `0x3EF72 -> 0x37D2D` | current A108 field | `P_A108+0x24` predicate | `VERIFIED` |
| `0x4127D -> F_45FD1` | current A108 field | status/control pointer | `VERIFIED` |
| `0x3E7B7` positive store | different base | not A108 without base proof | `BLOCKED` |

The first direct F0411A6 handoff is the B0 variant on the direct F03E961 path. Later state transitions are last-writer-dependent.

## 13. F8/FC P-track

```text
P
 -> D_P=P+0x14CCC
 -> T_P=P+0x14E40
 -> A_P/B_P
 -> F_45DE0
 -> D_P+0xF8/0xFC
```

`F_45DE0` installs the table words and fills local pointer regions. This track is `VERIFIED` as a construction, but the identity of `P` is unresolved.

A later F0411A6 invocation may call `F_45E52` at `0x411F5/0x41259/0x4126D` and reload `D_P+0xF8/0xFC` at `0x45EFB/0x45EFF/0x45F53/0x45F57`, but only if `D_P=D_B`. Otherwise it reads D_B’s own/stale fields.

## 14. F8/FC B0-track

```text
B0
 -> D_B=B0+0x14CCC
 -> F_45FD1(D_B,A108)
 -> F_4618E(D_B)
 -> D_B+0xF8/0xFC
```

`F_4618E` locally fills and transforms the A/B pointees. No direct downstream pointer propagation is proven after the call.

```text
D_P == D_B:       MERGE CONDITIONAL on P=B0
P != B0:          REMAIN DISTINCT
D_B F8/FC source:  BLOCKED when P!=B0
```

## 15. Generated DTS Byte Flow

### 15.1 DEC F8/FC writes

`F_4618E` writes local buffers at the following kinds of addresses:

```text
A + 4*i
B + 4*i
A + derived offset
B + derived offset
```

It writes sync/header values including:

```text
0x7FFE / 0x8001
0x1FFF / 0xE800
```

These are verified local memory writes, not proven queue insertions.

### 15.2 MS12V22 SND codec-specific writes

AC3:

```text
0x1EDBD..0x1EDBF  data type 0x0001
0x1EDC1            burst field length<<3
0x1EDA6/0x1EDAF    Pa/Pb
0x1EDD6            0x1078A9 copy
```

DTS:

```text
0x1F8B1..0x1F8B4  data type 0x000B
0x1F8B6..0x1F8BA  burst field 0x3F80
0x1F745..0x1F8A0  branch/setup and Pa/Pb
0x1F8C5            0x1078A9 copy
```

These are the strongest direct AC3/DTS output-frame-field divergences. The compressed DTS payload source and physical sink remain unresolved.

The SND output manager is shared/interleaved: the AC3 slot path also reaches `0xCC9B`/`0xCB50`, while the DTS slot path reaches `0xCC9B`, `0xCB50`, `0xCF86`, and `0xCAB4`. Those helper calls are not proven AC3-exclusive or DTS-exclusive. The direct source/length edges are codec-specific:

```text
AC3: 0x1EDC8 source = *(r10+0x4860)
     0x1EDCF length = 0xA00
     0x1EDD6 0x1078A9(dst=r14+8, src, len=0xA00)

DTS: 0x1F8BC source = r13
     0x1F8BF length = 0x7F0
     0x1F8C5 0x1078A9(dst=r14+8, src=r13, len=0x7F0)
```

These are `BUFFER-DATA` divergences. Neither source is statically tied to DEC `F8/FC`, and neither is a physical-transport proof.

## 16. F_4618E Downstream

Return sites:

```text
0x461C2  early return
0x462FE  prepared return
```

The direct caller at `0x41289` continues at `0x4128D`, where only `r12` status is consumed. The path clears/returns status and does not reload A/B fields.

A later state-machine entry may call F45E52/F4618E again, but that is a new invocation, not a direct post-return consumer.

## 17. Delayed / Indirect Consumers

`F_029867` is an indirect/asynchronous top-level entry. It provides a separate downstream value track:

```text
S = *(R+0x174)
0x29FE9  load S
read *S
compare IEC sync signatures
0x2A04F  sync-hit path
0x2A1B3  0x28A33(R+0x140,S,R+0x178)
or       0x2A4ED alternate R+0xFC destination
0x28A33  byte-order transformation
0x13602 / 0x13B08 ring/cache helpers
F_28AFE -> F_14F63 / F_1476A
```

The sync signatures include the observed `0x7FFE8001`, `0x80017FFE`, `0x1FFFE800`, `0xE8001FFF`, and related forms.

The delayed S track is `VERIFIED` as a value/format consumer. The identity

```text
F8/FC A/B -> S
```

is `BLOCKED/UNPROVEN`. No visible writer in F375BE/F3E961/F0411A6 installs S from the D/F8/FC addresses.

The `F_45FD1` indirect dispatch is also delayed/control-related:

```text
0x46053  r23=0x150000
0x46057  subtract 0x4804 -> 0x14B7FC
0x4605B..0x46060  indexed table load
0x46063  jr r23
```

The selector is bounded to `0..13`; exact target resolution is `BLOCKED`. The old `0x107FC` table-base annotation is corrected to `0x14B7FC`.

## 18. TX Ring Relation

The static TX monitor remains:

```text
MDrv_AUDIO_Dump_SpdifNpcm_Monitor @0x369408
ring = HAL_AUDIO_GetDspMadBaseAddr(1)+0x672000
span = 0x66000
```

It receives no D, P, B0, V, A, B, F8, FC, SND, or IEC source pointer. Its producer/control structure is separate.

```text
TX monitor construction:                 VERIFIED
TX monitor -> DTS A/B/F8/FC edge:         NOT FOUND
TX monitor -> S/F029867 edge:             NOT FOUND
TX monitor -> physical SPDIF:             UNPROVEN
```

The ring is not used to merge the DEC and SND tracks.

## 19. Standard vs MS12V22

### Image selection

`HAL_AUDSP_DspLoadCode @0x46F5A8` reads `AV+0x4D0` at `0x46FB28` and conditionally selects base/MS12 image pointers and sizes at `0x46FB38..0x46FB68`.

Status: image/profile selection `VERIFIED`; live selected profile `BLOCKED`.

### SetSystem2 profile branch

`SetSystem2` profile logic at `0x44D140..0x44D150` changes the table/byte selection. DTS `0x97` is conditional on `AV+0x4D8==3`; AC3 uses `0x01` or `0x7F` according to profile.

Status: control-flow/data-flow difference `VERIFIED`; “MS12V22 fixes DTS” is `UNPROVEN`.

### Function identity

Both image families contain common DTS packer names/structures, but exact one-to-one address identity is not established from string presence alone.

```text
STANDARD vs MS12V22 same address/behavior:  NOT PROVEN
Profile selection difference:               VERIFIED
DTS repair claim:                           BLOCKED
```

## 20. Conditions Required for DTS Output

The static corpus supports the following necessary-condition set. It does not prove that all are simultaneously satisfied in a live stock session.

| ID | Condition | Where set | Where consumed | AC3 comparison | DTS requirement | Status |
|---|---|---|---|---|---|---|
| C1 | Input format selects DTS parser | `0x22EB2/0x230B8/0x28258` | `Parser_Open 0x230C0` | `ac3` name | `dts` name | `VERIFIED` |
| C2 | Codec ID 9 reaches MI | `0x2F528/0x2F52E` | `MI_AUDIO_Start` | ID 5 | ID 9 | `VERIFIED` |
| C3 | MI decSys 0xB | `0xA3748/0xA374C` | SetSystem2 | decSys 3 | decSys 0xB | `VERIFIED` |
| C4 | Host handle passes MApi gate | `_MApi 0x3E534C..0x3E53C8` | MDrv dispatch | handle-dependent | handle-dependent | `CONDITIONAL` |
| C5 | DTS SetSystem2 configuration | `0x44D1DC..0x44D21C` | DSP setup | AC3 branch at `0x44D228` | latch 0xA, byte 0x04/0x97 | `VERIFIED` |
| C6 | DTS Decoder_Type conditions | `0x45F6BC..0x45F6E8` | query acceptance | latch 3 / SHM33 / SHM4D | AV+4D8 and SHM38 ranges | `VERIFIED` conditions; live values `BLOCKED` |
| C7 | P/D identity | `0x3DF03..0x3E344` vs `0x41275` | F45E52/F4618E | not applicable as merge | `P=B0` for same descriptor | `BLOCKED` |
| C8 | F45FD1 data-type branch | `0x460C6/0x460CD` | F45D07/F4618E | selector 0x400 -> 0x10C | selector 0x200 -> 0x0B | `VERIFIED` branch |
| C9 | Local F8/FC processing | `0x45E08..0x462FE` | local buffers | descriptor-dependent | descriptor-dependent | `VERIFIED` local |
| C10 | SND IEC frame field | `0x1EDBD` / `0x1F8B1` | local SND buffer | type 1, length field | type 0xB, 0x3F80 | `VERIFIED` |
| C11 | F8/FC -> S/transport | no proven edge | `F_029867/F_28A33/ring/DMA` | no direct proof | required for any downstream claim | `BLOCKED` |
| C12 | Physical sink | no static edge | transmitter/peripheral | not proven | not proven | `UNPROVEN` |

The static corpus does not justify inventing additional C13/C14 conditions merely to make the table look complete.

## 21. Verified Failure Point

The strongest honest statement is:

```text
First verified intentional divergence:
  parser/codec selection at 0x22D4C / 0x28258 / 0x2F4E8-0x2F528

First verified stateful divergence:
  SetSystem2 at 0x44D228 (AC3) vs 0x44D1DC (DTS)

First verified output-acceptance condition divergence:
  Decoder_Type at 0x45F230 (AC3) vs 0x45F6BC (DTS)

First verified frame-field divergence:
  SND +4 at 0x1EDBD (AC3) vs 0x1F8B1 (DTS)
```

A single stock-failure point is not proven. The first unresolved point where a DTS-specific execution can no longer be shown to continue is the F8/FC-to-delayed-output/transport identity, not a proven license field.

## 22. Corrections to Earlier Reports

1. The userspace symbol is `mi_decoder_open` at primary ELF address `0x2F408`; shifted `0x3F...` addresses from older dumps are not used.
2. The earliest AC3/DTS fork is parser-name selection at `0x22D4C/0x230B8/0x28258`, before `mi_decoder_open`.
3. The `_MApi` first argument is an OpenDecodeSystem return/handle, not statically proven codec 5/9.
4. The first AC3 byte is raw `0x01` or `0x7F`, not an inferred `0x81`.
5. `0xB` is the DTS decSystem/branch value; the DSP decoder latch written by SetSystem2 is `0xA`.
6. `HAL_AUDIO_SPDIF_SetMode` has no proven codec-source branch; historical source=3/4 correlation is not a static edge.
7. The F45FD1 indirect table base is `0x14B7FC`, not `0x107FC`.
8. F4618E’s direct caller boundary is `0x4128D`, but delayed output processing continues on a separate S track; `0x4128D` is not automatically the physical-output boundary.
9. F8/FC and the delayed S pointer must not be merged without an alias proof.
10. Historical wire captures remain hypotheses and are not evidence that stock DTS output succeeded.

## 23. First Opaque Boundary

The first end-to-end opaque boundary is:

```text
F_45DE0/F_4618E local A/B writes
        |
        | no proven pointer/data edge
        v
S = *(R+0x174) consumed by F_029867
```

Evidence immediately before the boundary:

```text
0x45E08/0x45E17/0x45E25/0x45E2F  install D+0xF8/0xFC
0x461C7..0x462FE                  local A/B fills/transforms/sync writes
0x41289                           F_4618E(D_B)
0x4128D                           status/F0 return handling
```

The separate S-side evidence begins at:

```text
0x29FE9  S=*(R+0x174)
0x2A1B3  0x28A33 copy/byte-order path
```

The exact A/B-to-S relation is `BLOCKED`.

## 24. Remaining Unknowns

1. Live `MApi` first argument and whether the `+0x10=0xFF` rejection branch is taken for stock DTS.
2. Live values of `AV+0x4D0`, `AV+0x4D8`, `AV+0x524/0x528`, SHM33, SHM38, and SHM4D.
3. Whether the `0x3847E`/stack epilogue leaves `P` as `V_in`, `B0`, or another pointer.
4. Whether any constructor upstream of `[B0+0x1414C]` supplies a non-zero stable pointer.
5. Whether D_P and D_B merge in a later state-machine invocation.
6. Whether F8/FC bytes ever become `S=*(R+0x174)` or another queue/ring source.
7. The exact `F_45FD1` indirect targets at table base `0x14B7FC`.
8. Which SND/DEC output buffer reaches any physical transmitter.
9. Whether the STANDARD or MS12V22 profile is active in a given stock attempt.
10. The exact producer and consumer relation for the `0x672000` software dump ring.

## 25. Highest-Value Next Investigation

1. Statically close the host gate and live-state prerequisites: exact OpenDecodeSystem return for DTS, `+0x10`, `AV+0x4D0/0x4D8`, `AV+0x524/0x528`, and SHM33/38/4D.
2. Statically close the P/B0 identity and the A/B-to-delayed-S edge, including the `0x3847E` epilogue and any constructor/workspace initializer.
3. Resolve the `F_45FD1` table targets at `0x14B7FC` and the subsequent `F_029867 -> F_28A33 -> F_28AFE/F_14F63` producer chain.
4. Only after those edges exist can the physical-output boundary be classified; the current corpus does not prove a physical SPDIF sink.

## Preservation Statement

This report is a new artifact. No existing report, raw log, binary, Ghidra project, script, hash, or historical correction was modified. No runtime, ADB, device, playback, memory inspection, patch, flash, reboot, module reload, configuration change, or TCL/T615 evidence was used.
