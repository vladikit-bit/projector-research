# SPACE BUNNY — POST-TCL STATIC CLOSURE PASS

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Purpose:** close additional TD98 unknowns after the targeted TCL differential  
**Runtime:** none; no ADB, device, playback, memory inspection, patch, flash, reboot, or module reload  
**TCL/T615:** used only as the previously bounded differential reference  
**Preservation:** no previous artifact was modified

## Executive Summary

This pass found two additional TD98 primary-evidence results:

1. **MApi/Utopia binding is now statically closed inside the TD98 module pair.**  
   `mik.ko` has undefined `MApi_AUDIO_OpenDecodeSystem` / `MApi_AUDIO_SetDecodeSystem`; `utpa2k_stock.ko` exports the corresponding implementations. The exported wrappers pack the arguments into an ioctl payload (`0x3F` for Open, `0x3C` for Set), and the internal `_MApi` functions apply the four gate checks. The first argument is a decoder slot/ID returned by the Open path, not codec `5` or `9`. The exact live slot value remains conditional, but the module binding and payload flow are no longer opaque.

2. **The immediate post-copy SND chain does not directly read the copied destination.**  
   After both AC3 and DTS `0x1078A9` calls, the next helper calls use different bases (`r13+0x6AD4` or the enclosing `r11/r12` object), not the `r14+8` / `r11+8` destination passed to the copy helper. This is a concrete negative result for the immediate downstream edge, although a later asynchronous reader remains possible.

The following remain unresolved:

- TD98 `0x3847E` return-visible `r11` semantics;
- upstream constructor/lifetime of `[B0+0x1414C]`;
- `P == V_in`, `P == B0`, or `D_P == D_B`;
- F8/FC publication into `S=*(R+0x174)`;
- eventual physical transport of the SND destination.

### TD98 anchors retained for this pass

```text
MI_AUDIO_OpenDecodeSystem call: 0xA29F4
MI+0xADC store:                    0xA2AC0
MI_AUDIO_Start reload:             0xA4850
MApi_SetDecodeSystem call:         0xA4858
TX monitor ring base/span:         0x672000 / 0x66000
```

## 1. TD98 MApi export/loader closure

### QUESTION

Can the `mik.ko` undefined `MApi_*` calls be tied to the stock `utpa2k_stock.ko` implementations and their ioctl payloads?

### PRIMARY EVIDENCE

`patch_baseline/mik_stock.ko` contains:

```text
UND MApi_AUDIO_OpenDecodeSystem
UND MApi_AUDIO_SetDecodeSystem
```

`patch_baseline/utpa2k_stock.ko` exports:

```text
MApi_AUDIO_OpenDecodeSystem  @0x3F32EC
MApi_AUDIO_SetDecodeSystem   @0x3F2FD4
_MApi_AUDIO_OpenDecodeSystem @0x3E56E0
_MApi_AUDIO_SetDecodeSystem  @0x3E52A8
```

### Open payload

At `0x3F32EC`:

```text
0x3F32F4..0x3F3314  prepare local payload
0x3F336C            r2 = &payload
0x3F3374            r1 = 0x3F
0x3F3378            bl UtopiaIoctl
0x3F3398            reload return field
```

The wrapper passes the Open argument into the payload and returns the ioctl result. The local Open structure is then passed to `_MApi_AUDIO_OpenDecodeSystem`.

### Set payload

At `0x3F2FD4`:

```text
0x3F2FF4  store arg0 at sp+1
0x3F3004  store arg1 at sp+5
0x3F3058  r2 = sp
0x3F3060  r1 = 0x3C
0x3F3064  bl UtopiaIoctl
```

The exported wrapper therefore binds directly to the kernel MDrv-side implementation. The same structure and command split is independently present in TCL, but TCL is not needed to establish TD98 module binding.

### Internal Set gate

At `_MApi_AUDIO_SetDecodeSystem @0x3E52A8`:

```text
0x3E534C  cmp r4,#5
0x3E5350  cmnne r4,#1
0x3E5354  bne reject/alternate
0x3E53B8  ldr r0,[r5,#0x10]
0x3E53BC  cmp r0,#0xFF
0x3E53C0  cmpne r0,#0
0x3E53C4  bne continue
...
0x3E5448  AV[r4*0x28+0xFA] check
0x3E5468  tail-call MDrv_AUDIO_SetDecodeSystem(r4,r5)
```

The first argument is therefore a decoder/slot selector. `+0x10=0xFF` is structurally rejected on the ordinary path, not universally killed before the alternate `5/-1` path.

### Open return and slot publication

`_MApi_AUDIO_OpenDecodeSystem @0x3E56E0` calls `MDrv_AUDIO_OpenDecodeSystem` at `0x3E58B0`, receives `r4`, and then:

```text
0x3E593C  pass r4 to MDrv_AUDIO_SetDecodeSystem
0x3E5940..0x3E5958  compute slot address r4*0x28+0xF9
0x3E5964..0x3E5968  write slot validity marker at +0xFA
```

This proves the return is a slot/session value propagated into the internal API, not a codec type. The exact live slot set is still determined by the registered Open callback and runtime state.

### RESULT

- TD98 `mik.ko` → `utpa2k_stock.ko` MApi binding: **VERIFIED**.
- Wrapper command split `0x3F/0x3C`: **VERIFIED**.
- First argument is not codec 5/9: **VERIFIED**.
- Exact live slot value: **CONDITIONAL / RUNTIME-DEPENDENT**.
- AC3/DTS branch through the MApi gate: **still conditional**, not proven failure point.

### STATUS

`VERIFIED` for the static loader/payload path; `BLOCKED` only for the live slot and selected branch.

## 2. TD98 AV/SHM field provenance refinement

### PRIMARY EVIDENCE

`MDrv_AUDIO_CheckHashkey` writes:

```text
0x4234D8  [AV+0x440] = 0
0x4234DC  [AV+0x444] = 0xFFFFFF
0x4234E0  [AV+0x582] = 0
...
0x424870  [AV+0x4D0] = r6
0x424874  [AV+0x4D4] = r5
0x424878  [AV+0x4D8] = r8
```

The values `r5/r6/r8` are assembled by multiple `MDrv_AUTH_IPCheck` branches. Static code proves path dependence and the writer, not one universal value.

Latch setters:

```text
0x44CC9C  [AV+0x524] = r4
0x44CD4C  [AV+0x528] = r4
```

Additional `SetSystem2` branches write these slots, and `HAL_MAD_GetAudioInfo2` reads them for the type-`0x30` table. SHM33/38/4D are read through the SHM-info helpers; no direct TD98 host writer of their live values was found in the relevant setup path.

### RESULT

- AV+0x4D0/0x4D8 writer and downstream readers: **VERIFIED**.
- AV+0x524/0x528 latch-slot role: **VERIFIED**.
- Exact live values: **RUNTIME-DEPENDENT**.
- SHM producer: **BLOCKED**.

## 3. TD98 immediate SND post-copy closure

### QUESTION

Does the first call after `0x1078A9` read the destination buffer that was just filled?

### AC3 sequence

```text
0x1EDC8  source = *(r10+0x4860)
0x1EDCF  length = 0xA00
0x1EDD3  destination = r14+8
0x1EDD6  call 0x1078A9
0x1EDDA  next helper base = r13+0x6AD4
0x1EDDE  next helper length = 0xC00
0x1EDE2  call 0xC8CA
0x1EDEF  call 0xC49D
```

### DTS sequence

```text
0x1F8BC  source = r13
0x1F8BF  length = 0x7F0
0x1F8C2  destination = r11+8
0x1F8C5  call 0x1078A9
0x1F8C9  next helper base = r11
0x1F8CB  next helper length = 0x400
0x1F8CF  call 0xC8CA
0x1F8DD  call 0xC49D
```

Neither immediate post-copy call takes the copied destination as its data pointer. The next helpers operate on a separate context object and control/status fields.

### Helper observations

`0xC8CA` loads object fields and calls `0xD4C8`; it is not a direct `r14+8` buffer reader in these call sites. `0xC49D` reads a different runtime/control structure. The immediate chain therefore does not prove copy-to-transport.

### RESULT

- AC3/DTS copy construction: **VERIFIED**.
- Immediate next helper receives copied destination: **DISPROVEN at these call sites**.
- Later asynchronous reader of the destination: **NOT FOUND / BLOCKED**.

This narrows U9: the first post-copy function is not the missing consumer, and the eventual consumer remains opaque.

## 4. Reassessment of U1–U10

| Unknown | New static result | Status |
|---|---|---|
| U1 MApi binding | TD98 export/loader and payload path closed; live slot/branch remains conditional | `REDUCED` |
| U2 AV/SHM | Direct writers/readers closed; live values and SHM producer unresolved | `PARTIALLY CLOSED` |
| U3 r11 | No new epilogue proof | `UNCHANGED / BLOCKED` |
| U4 1414C | No constructor found | `UNCHANGED / BLOCKED` |
| U5 D_P/D_B | No merge/publication found | `UNCHANGED` |
| U6 F8/FC→S | No new publication edge | `UNCHANGED / BLOCKED` |
| U7 F45FD1 | TD98 local dispatch remains resolved; no A/B target | `UNCHANGED` |
| U8 delayed S | No new identity edge | `UNCHANGED` |
| U9 SND→transport | Immediate post-copy destination edge disproven; later reader unknown | `REDUCED` |
| U10 TX ring | Still separate monitor/dump surface | `UNCHANGED` |

## 5. Causal impact

The static pass changes the order of investigation:

```text
U1: exact live MApi slot and branch selection
  ↓
U2: live AV/SHM state
  ↓
U4/U5: V/P/D identity
  ↓
U6: F8/FC publication
  ↓
U8/U9: delayed S and SND destination consumer
```

U9 is now more precise: after the SND copy, the immediate helper chain is **not** a direct destination consumer. The next unresolved edge is a later reader or a different descriptor/publication path.

## 6. Targeted TCL status

The TCL differential remains:

- `TCL CORROBORATION` for MApi/Utopia command/gate architecture.
- `TCL STRUCTURAL ANALOGUE` for callback/object/latch initialization.
- `TCL UNRELATED` for the exact TD98 `0x3847E`, `1414C`, F8/FC→S, and SND transport edges.
- No TCL semantic transfer was made.

## 7. Exact remaining opaque edges

```text
MApi live slot/branch:
  exported MApi @0x3F32EC/0x3F2FD4
  → _MApi @0x3E56E0/0x3E52A8
  → runtime callback/AV state

V/P:
  [B0+0x1414C]
  → 0x3DF03 helper sequence
  → r11 at 0x3DF79/0x3E19E

DEC publication:
  D+0xF8/0xFC
  → *(R+0x174)

SND transport:
  r14+8/r11+8 destination
  → later reader
  → DMA/MIU/physical output
```

## 8. Next static target

The highest-value next static target is now:

```text
search all writes to R+0x174 / F029867 source objects
and all later readers of r14+8/r11+8 after 0x1078A9,
including asynchronous state-machine entries.
```

The second target is resolving the TD98 `0x3847E` epilogue through a primary ISA/compiler-pattern source.

## Preservation Statement

This is a new report. No previous report, raw log, binary, Ghidra project, script, hash, or TCL/T615 artifact was modified. No runtime, ADB, device, playback, memory inspection, patch, flash, reboot, module reload, or configuration change was performed.
