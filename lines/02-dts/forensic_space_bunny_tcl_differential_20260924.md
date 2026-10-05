# SPACE BUNNY — TARGETED TCL/T615 DIFFERENTIAL

**Date:** 2026-09-24  
**Mode:** static/corpus-only, targeted differential  
**TD98 authority:** TD98 remains primary for every TD98 conclusion  
**TCL/T615:** used only for the exact structural questions below  
**Runtime:** none; no ADB, device, playback, memory inspection, patch, flash, reboot, or module reload  
**TCL extraction:** `C:\Users\k0994\AppData\Local\Temp\opencode\tclaudio\` (read-only)

## Executive Summary

The TCL/T615 differential reduced or corroborated only a subset of the remaining unknowns:

- **U1 MApi binding:** TCL independently confirms the Utopia command structure and the four-gate `_MApi` logic, including the `+0x10` test. It supports a slot/handle interpretation, not a codec-ID interpretation. TD98’s live handle and kernel export binding remain unresolved.
- **U2 AV/SHM:** TCL shows the same broad callback/latch/SHM architecture, but with different object offsets and AArch64 field layout. It does not provide TD98 live values.
- **U3 r11 epilogue:** no valid structural counterpart was found. TCL is AArch64 and uses ordinary `stp/ldp` callee-save patterns; this does not resolve TD98 ARM32 `bt.swst`/`bt.inst16_4` semantics.
- **U4 1414C/A108 constructor:** TCL has object-construction/state-publication analogues, but no proven field-to-subobject relationship equivalent to TD98 `[B0+0x1414C] → P+0x7C4`.
- **U5 D_P/D_B:** unchanged; TCL provides no identity bridge.
- **U6 F8/FC→S:** unchanged; no TCL publication edge analogous to the missing TD98 edge was found.
- **U7 F_45FD1:** TD98 table resolution remains the primary result. TCL did not reveal a hidden A/B consumer.
- **U8 delayed S/ring:** TCL monitor code is not a structural equivalent of TD98’s `F_029867→F_28A33→ring` path.
- **U9 SND transport:** TCL SPDIF functions inspected are configuration/control paths; no comparable payload-copy-to-transport chain was proven.
- **U10 TX ring:** TCL has a separate SPDIF/PCM monitor symbol, but no proof that it is the same ring or a physical sink.

The differential therefore confirms architecture and codegen patterns, but does not invent a TD98 constructor, publication edge, or physical output path.

## Corpus / Evidence Rule

TCL primary artifacts inspected:

```text
C:\Users\k0994\AppData\Local\Temp\opencode\tclaudio\lib_modules_utpa2k.ko
  AArch64, unstripped, targeted symbols

C:\Users\k0994\AppData\Local\Temp\opencode\tclaudio\lib_modules_mik.ko
  AArch64, unstripped

C:\Users\k0994\AppData\Local\Temp\opencode\tclaudio\lib_libutopia.so
  ARM32, unstripped, MApi/Utopia/output symbols
```

Classification vocabulary:

- `TCL CORROBORATION`: TCL independently confirms a structural mechanism already visible in TD98.
- `TCL STRUCTURAL ANALOGUE`: TCL has a related but not equivalent object/codegen/data-flow shape.
- `TCL CONTRADICTION`: primary TCL evidence conflicts with a TD98 structural claim.
- `TCL UNRELATED`: no structural relation is established.
- `BLOCKED`: the requested TD98 edge remains open even after the targeted TCL search.

No TCL result is transferred semantically to TD98 merely because names, offsets, or strings match.

## TCL Target 1 — TD98 `0x3847E` r11/epilogue

### TD98 construct

```text
0x3847E  helper entry
0x384C8  data-dependent loop
0x384CB  r11 = r1
0x384DE  r11 += 4
0x38519  return
```

TD98 local SLEIGH leaves `bt.swst` and `bt.inst16_4` semantics empty. The data-positive path provably clobbers `r11`; return-visible `r11` is not closed.

### TCL construct

Relevant AArch64 functions in `lib_modules_utpa2k.ko` show ordinary compiler-generated save/restore patterns:

```text
MDrv_AUDIO_OpenDecodeSystem @0x417F98
  stp x29,x30
  str x19,[sp,#0x10]
  ...
  ldr x19,[sp,#0x10]
  ldp x29,x30
  ret

MDrv_AUDIO_SetDecodeSystem @0x434EC0
  stp x29,x30
  stp x22,x21
  stp x20,x19
  ...
  ldp x20,x19
  ldp x22,x21
  ldp x29,x30
  ret

HAL_AUDIO_SetSystem2 @0x45D16C
  stp x29,x30
  stp x22,x21
  stp x20,x19
  ...
  ldp x20,x19
  ldp x22,x21
  ldp x29,x30
  ret
```

A raw search for the TD98 halfword encodings did not produce a phase-anchored AArch64 counterpart. The AArch64 functions do not save x11 in these prologues; that is an ABI/codegen fact, not a mapping of TD98 r11.

### Structural relation

Same broad compiler pattern: persistent locals are saved/restored while a scratch register is reused. No direct function identity, instruction identity, or caller/callee register correspondence is established.

### Exact supporting addresses

```text
TCL MDrv_OpenDecodeSystem: 0x417F98
TCL MDrv_SetDecodeSystem:  0x434EC0
TCL HAL_SetSystem2:        0x45D16C
TD98 target helper:        0x3847E
TD98 clobber sites:       0x384CB, 0x384DE
```

### Difference

TD98 is ARM32 with custom/unknown stack instructions; TCL is AArch64 with standard `stp/ldp` codegen. The register classes are not equivalent.

### TD98 implication

U3 is not resolved. The “TCL restores TD98 r11” conclusion would be invalid.

### Status

`TCL STRUCTURAL ANALOGUE` for codegen; `TCL UNRELATED` for exact epilogue semantics.

## TCL Target 2 — 1414C/A108 construction lifecycle

### TD98 construct

```text
[B0+0x1414C] → V
0x3DF35 conditional broad fill
helper operations
0x3DF79 final field store
0x3E19E P+0x7C4
0x3E1A6 [B0+0xA108]=P+0x7C4
```

### TCL construct

TCL `HAL_AUDIO_OpenDecodeSystem @0x4507E0` reads an input object at `+0xC`, selects a table based on that value, calls a capability/ID helper, and stores a returned result into object/global fields such as `+0x18` and `g_AudioVars2+0x1C4`.

TCL `HAL_AUDIO_SetSystem2 @0x45D16C` reads `g_AudioVars2+0x548/+0x550/+0x558` and writes latch/state fields at `+0x5A4/+0x5A8` through several paths.

TCL `HAL_SND_R2_init_SHM_param @0x6DE570` performs an explicit object-style initialization sequence:

```text
allocate/obtain an SND object
store its pointer into a global
write object fields including +0x14C,+0x154,+0x158,+0x15C,+0x174
```

### Structural relation

TCL demonstrates the general MStar pattern of callback-selected state, object allocation, field initialization, and a later status/control handoff. It does not contain a proven `+0x1414C`/`+0xA108` relationship, and its field offsets/layout differ.

### Exact supporting addresses

```text
TCL HAL_OpenDecodeSystem: 0x4507E0
TCL HAL_SetSystem2:       0x45D16C
TCL HAL_SND_R2_init:      0x6DE570
TCL CheckHashkey:         0x43347C
```

### Difference

The TCL SND object is a different component and architecture. No pointer chain of the form:

```text
field → subobject at +0x7C4-like offset → descriptor consumer
```

is established.

### TD98 implication

U4 is not resolved. TCL corroborates that constructor-like setup can exist, but cannot supply TD98’s missing `[B0+0x1414C]` initializer.

### Status

`TCL STRUCTURAL ANALOGUE`; no TD98 identity or value transfer.

## TCL Target 3 — F8/FC → downstream publication

### TD98 construct

```text
D_P/D_B +0xF8/0xFC
  → A/B pointer loads
  → F_45E52/F_4618E local transforms
  → no proven *(R+0x174) publication
```

### TCL construct

The targeted TCL functions contain descriptor/table and state writes, but no inspected function establishes a field-relative analogue of:

```text
[D+0xF8/0xFC] → [R+0x174]
```

TCL `HAL_SND_R2_init_SHM_param` initializes an SND object, while TCL `HAL_AUDIO_SetSystem2` writes latch/state words. Neither is a direct consumer of a DEC-style F8/FC pointer pair.

### Structural relation

Both firmwares contain descriptor/state objects and downstream helpers, but the missing publication edge is not reproduced.

### TD98 implication

U6 remains blocked. TCL does not justify adding a TD98 F8/FC→S edge.

### Status

`TCL UNRELATED` for the requested F8/FC→S edge; no structural publication found.

## TCL Target 4 — SND output buffer → transport

### TD98 construct

```text
TD98 AC3: source *(r10+0x4860) at 0x1EDC8,
          length 0xA00 at 0x1EDCF,
          0x1078A9 at 0x1EDD6 → r14+8
TD98 DTS: source r13 at 0x1F8BC,
          length 0x7F0 at 0x1F8BF,
          0x1078A9 at 0x1F8C5 → r14+8
```

### TCL construct

TCL `HAL_AUDIO_SPDIF_ApplySetting @0x6CD3A0` reads configuration/state fields such as `+0x4F0`, `+0x4F4`, and `+0x4F8`, then calls a configuration helper. It does not show the TD98 AC3/DTS source/length copy sequence.

TCL `HAL_SND_R2_init_SHM_param @0x6DE570` initializes an SND object and SHM-related fields, but no inspected direct read of a post-copy destination payload was tied to a DMA/MIU/physical transport object.

### Structural relation

TCL confirms separate SPDIF configuration and SND object initialization layers. It does not confirm TD98’s local copy destination or a downstream reader.

### TD98 implication

U9 remains blocked. The fact that TCL has SPDIF APIs does not prove the TD98 `r14+8` buffer is transmitted.

### Status

`TCL UNRELATED` for the exact copy/transport chain; `TCL STRUCTURAL ANALOGUE` only for configuration/SND object separation.

## TCL Target 5 — delayed S path

### TD98 construct

```text
S=*(R+0x174)
  → sync recognition
  → F_28A33 byte-order copy
  → F_13602/F_13B08 ring/cache
```

### TCL construct

TCL `MDrv_AUDIO_Dump_SpdifNpcm_Monitor @0x4AC800` is a monitor loop over an object/table of up to `0x200` entries. It loads indexed pointers/values, calls helper routines, and updates/clears monitor state.

No inspected TCL function contains a matching `S` source field, the TD98 sync-signature comparisons, and the `F_28A33` byte-order transformation followed by the TD98 ring index updates.

### Structural relation

Both paths are monitor-like and pointer-driven, but the producer/consumer architecture is not matched.

### TD98 implication

U8 is not reduced by TCL. The TD98 S/ring path remains a separate static track unless a direct TD98 alias is later found.

### Status

`TCL UNRELATED` for the exact delayed S/ring architecture; `TCL STRUCTURAL ANALOGUE` only for “monitor over runtime objects.”

## TCL Target 6 — physical output

### TD98 construct

TD98 has a verified local SND copy and a separate delayed S/ring path, but no direct static edge to a physical transmitter.

### TCL construct

TCL SPDIF/output APIs inspected are configuration, monitor, and object-control functions. No targeted function was found that takes a TD98-like generated buffer, wraps it in a proven DMA/MIU descriptor, and reaches a hardware register with a corresponding completion path.

### TD98 implication

The physical-output blocker remains. TCL SPDIF naming and API presence cannot be used as a physical sink proof.

### Status

`BLOCKED`; no TCL physical sink edge established.

## TCL Target 7 — host MApi binding

### TD98 construct

```text
MI_AUDIO_Open
  → OpenDecodeSystem return
  → MI+0xADC
  → MI_AUDIO_Start
  → +0x10=0xFF
  → MApi_SetDecodeSystem
```

The exact TD98 kernel/userspace module binding and live first-argument value are unresolved.

### TCL construct

The TCL `lib_libutopia.so` provides a strong independent mechanism comparison:

```text
_MApi_AUDIO_SetDecodeSystem @0x33C05C
MApi_AUDIO_SetDecodeSystem   @0x349F84
_MApi_AUDIO_OpenDecodeSystem @0x3341C0
MApi_AUDIO_OpenDecodeSystem  @0x34A29C
```

The disassembly of `_MApi_AUDIO_SetDecodeSystem` contains the same structural gate sequence:

```text
0x33C0FC  cmp r4,#5
0x33C100  cmnne r4,#1
0x33C104  bne fail
0x33C1A0  ldr r0,[r5,#0x10]
0x33C1A4  cmp r0,#0xFF
0x33C1A8  cmpne r0,#0
0x33C1AC  bne continue
```

The TCL wrapper functions also use the matching Utopia command split:

```text
MApi_AUDIO_OpenDecodeSystem → UtopiaIoctl command 0x3F
MApi_AUDIO_SetDecodeSystem  → UtopiaIoctl command 0x3C
```

TCL `MDrv_AUDIO_OpenDecodeSystem @0x417F98` is a callback thunk, and `HAL_AUDIO_OpenDecodeSystem @0x4507E0` uses a table/capability path before writing object state.

### Structural relation

The command IDs, gate polarity, callback/open structure, and payload organization are directly corroborated across the TCL/TD98 lineage. The TCL result supports a slot/handle interpretation of the first argument; it does not prove the live TD98 return value or that the TD98 kernel export resolves to the same binary object in every boot.

### TD98 implication

U1 is reduced:

- MApi first argument is not a codec ID: `TCL CORROBORATION` plus TD98 data flow.
- `+0x10=0xFF` is a valid gate value, not a universal killer: `TCL CORROBORATION`.
- exact TD98 handle range/live value: still `BLOCKED`.

### Status

`TCL CORROBORATION`, not semantic transfer.

## Updated U1–U10 Matrix

| Unknown | TCL effect | Updated status |
|---|---|---|
| U1 MApi binding | Confirms Utopia cmd split, four-gate structure, and slot/handle lineage | `REDUCED`; live TD98 binding/value still `BLOCKED` |
| U2 AV/SHM | Confirms callback/latch/SHM architecture with different offsets | `UNCHANGED` for live TD98 values |
| U3 r11 epilogue | AArch64 save/restore pattern only; no exact counterpart | `UNCHANGED` / `BLOCKED` |
| U4 1414C constructor | Object-construction analogues exist, no equivalent field chain | `UNCHANGED` / `BLOCKED` |
| U5 D_P/D_B | No TD98 identity bridge | `UNCHANGED` |
| U6 F8/FC→S | No comparable publication edge | `UNCHANGED` / `BLOCKED` |
| U7 F45FD1 | No hidden A/B consumer revealed; TD98 table result remains authoritative | `UNCHANGED` |
| U8 delayed S/ring | TCL monitor is not structurally equivalent | `UNCHANGED` |
| U9 SND→transport | TCL config/object functions do not close destination reader | `UNCHANGED` / `BLOCKED` |
| U10 TX ring | TCL has a monitor symbol but no shared producer/consumer proof | `UNCHANGED` / `SEPARATE` |

## Causal Effect of TCL Evidence

```text
TCL MApi gate/callback comparison
  → reduces uncertainty about U1 architecture
  → does not close TD98 live handle

TCL AArch64 codegen comparison
  → shows ordinary callee-save mechanics
  → does not close U3 r11 epilogue

TCL object/latch initialization
  → confirms constructor-like patterns exist
  → does not supply U4 1414C provenance

TCL monitor/config functions
  → do not provide U6/U8/U9/U10 identity edges
```

The first TD98 opaque edges remain:

```text
U3/U4: 0x3DF03 return → 0x3E19E
U6:    D+0xF8/0xFC → *(R+0x174)
U9:    SND r14+8 destination → transport
```

## Final TCL Differential Verdict

The TCL/T615 differential is useful for mechanism corroboration, especially the Utopia/MApi gate and the existence of callback/object initialization patterns. It does not justify transferring TD98 runtime semantics.

The smallest unresolved TD98 set after TCL comparison is:

```text
U1: exact live MApi handle and binding
U2: live AV/SHM values
U3: return-visible r11 after 0x3847E
U4: upstream initializer of [B0+0x1414C]
U6: F8/FC publication into S
U9: SND destination consumer/transport
```

`U7` is reduced by TD98 itself, not by TCL. `U5`, `U8`, and `U10` remain separate/blocked tracks.

## Preservation Statement

This is a new report. No previous report, raw log, binary, Ghidra project, script, hash, or TCL/T615 artifact was modified. No runtime, ADB, device, playback, memory inspection, patch, flash, reboot, module reload, or configuration change was performed.
