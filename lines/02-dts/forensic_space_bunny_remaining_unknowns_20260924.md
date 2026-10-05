# SPACE BUNNY — REMAINING UNKNOWNS CLOSURE PASS

**Date:** 2026-09-24  
**Mode:** static/corpus-only, targeted follow-up  
**Scope:** Sections 24–25 unknowns from the prior full reconstruction only  
**Runtime:** none; no ADB, device, playback, runtime memory, patch, flash, reboot, or module reload  
**TCL/T615:** not used; no required blocker was resolved by TCL/T615 evidence  
**Preservation:** no prior artifact was modified

## Executive Summary

This pass attacks the ten unresolved items individually rather than repeating the prior broad audit.

The strongest reductions are:

- **U1:** the MI first argument is a decoder-open return propagated through a function-pointer callback, not a statically fixed codec value. A known callback path suggests a small slot/ID range, but the module binding and live value remain unresolved.
- **U2:** `AV+0x4D0/0x4D8` are written by `MDrv_AUDIO_CheckHashkey`; `AV+0x524/0x528` are explicit latch-slot words with multiple path-dependent writers; SHM33/SHM38/SHM4D are query inputs whose live producers are not closed in TD98.
- **U3:** `0x3847E` definitely has a reachable `r11` clobber. The supplied SLEIGH does not model `bt.swst`/`bt.inst16_4`, so the post-helper value is not safely classified.
- **U4:** no upstream constructor for `[B0+0x1414C]` was found. The normal fill zeroes it temporarily, then `0x3DF79` writes a helper-dependent value.
- **U5:** no later path was found that merges `D_P` and `D_B`; the tracks remain distinct unless `P==B0`.
- **U6:** the F8/FC-to-`S` edge remains blocked. No visible writer publishes `D+0xF8/0xFC` into `*(R+0x174)`.
- **U7:** the `F_45FD1` table is now largely resolved. All 14 entries are local scalar/control labels; none is an A/B or TX consumer.
- **U8:** `F_029867 → F_28A33 → F_13602/F_13B08` is a real delayed value/ring track, but its source `S=*(R+0x174)` is not the DEC F8/FC object.
- **U9:** SND AC3/DTS local output-buffer copies are resolved, but the next physical transport edge remains open.
- **U10:** the `+0x672000` ring is a software dump/monitor surface with no proven producer link to DEC or SND output.

The smallest causal set is therefore:

```text
U1/U2: host handle + AV/latch/SHM acceptance state
U3/U4/U5/U6: V/P/B0 identity and the F8/FC→S handoff
U8/U9: separate delayed/SND output tracks
```

U7 is now mostly disconnected from the A/B question: it is descriptor scalar/control dispatch, not a hidden A/B consumer.

## Unknown Closure Matrix

| ID | Unknown | Result | Status | Evidence | Next dependency |
|---|---|---|---|---|---|
| U1 | MApi first argument / `+0x10` | Open callback/slot range; `+0x10=0xFF` source verified, gate reachability unresolved | `CONDITIONALLY VERIFIED` / `BLOCKED` | `MI_AUDIO_Start 0xA29F4/0xA2AC0/0xA4850/0xA4858`; `MDrv_AUDIO_OpenDecodeSystem 0x405CC8`; `HAL_AUDIO_OpenDecodeSystem`; libutopia/kernel symbol split | loader/export binding and live callback value |
| U2 | AV/SHM provenance | AV+4D0/4D8 and latch fields have writers; SHM live producers unresolved | `PARTIALLY VERIFIED` / `BLOCKED` | `MDrv_AUDIO_CheckHashkey`; `HAL_AUDIO_SetDecoder12_Type`; `HAL_AUDIO_SetSystem2`; `HAL_MAD_GetAudioInfo2` | DSP/SE producer and live state |
| U3 | `0x3847E` r11 semantics | Reachable clobber proven; return-visible r11 unresolved | `CONDITIONALLY VERIFIED` / `BLOCKED` | `0x384CB/0x384DE/0x38519`; empty `swst/inst16_4` SLEIGH | primary ISA epilogue semantics |
| U4 | `1414C` constructor | No constructor; conditional fill + helper-dependent store | `BLOCKED` | `0x3DF2D/0x3DF35/0x3DF79`; no upstream direct writer | external constructor/loader or opaque state |
| U5 | D_P/D_B merge | No merge found; conditional only if `P=B0` | `REMAIN DISTINCT` / `CONDITIONAL` | F3E961 one-action state flow; F0411A6 `0x41275`; F03E0DD `0x3E325` | U3/U4 identity |
| U6 | F8/FC → S | No publication edge found | `BLOCKED` | F029867 `S=*(R+0x174)`; F03E0DD table/fields; no writer | global descriptor/copy publication |
| U7 | F45FD1 indirect targets | Table words and local targets resolved; no A/B target | `VERIFIED` for local dispatch; A/B edge `DISPROVEN` | table `0x14B7FC`, 14 LE words, `0x46060/0x46063` | none for A/B; control semantics only |
| U8 | Delayed S/output chain | S → byte-order copy → ring/cache verified; link to F8/FC blocked | `CONDITIONALLY VERIFIED` / `BLOCKED` at F8→S | F029867, F28A33, F13602, F13B08 | U6 |
| U9 | SND buffer → transport | Local copy fields verified; next physical consumer unresolved | `BLOCKED` | AC3/DTS `0x1078A9` calls; shared SND helpers | downstream SND hardware-facing reader |
| U10 | TX ring producer/consumer | Monitor ring construction/read verified; no producer link | `OUTPUT MONITOR` / `BLOCKED` physically | `MDrv_AUDIO_Dump_SpdifNpcm_Monitor 0x369408` | absolute writer/consumer outside monitor |

## U1 — MApi first argument / `+0x10`

### QUESTION

What value can actually reach the first argument of the kernel `MApi_AUDIO_SetDecodeSystem` call, and is `+0x10=0xFF` the value that reaches the internal `_MApi` gate?

### PRIMARY EVIDENCE

```text
MI_AUDIO_Open:
0xA29F4  call MApi_AUDIO_OpenDecodeSystem
0xA2A04  capture return in r4
0xA2AC0  store return to [MI+0xADC]

MI_AUDIO_Start:
0xA44BC  write 0xFF to start struct
0xA47E0  copy into struct+0x10
0xA4850  load [MI+0xADC]
0xA4858  call MApi_AUDIO_SetDecodeSystem
```

The `mik.ko` file has undefined `MApi_AUDIO_OpenDecodeSystem` and `MApi_AUDIO_SetDecodeSystem`. The same names also exist as user-space exports in `libutopia.so`, but those are not automatically the kernel binding target. The stock `utpa2k` object retains local function labels and function-pointer thunks, while the exported symbol/loader binding is not closed by the static corpus.

The known kernel-side `MDrv_AUDIO_OpenDecodeSystem @0x405CC8` is a thunk:

```text
0x405CC8..0x405CE0  load callback and bx through it
```

The callback body eventually returns a decoder/slot result through `AU_GetDecID`/table logic. The known static callback path is consistent with a small slot/index range and `-1` failure sentinel, but the exact live return and loader binding are not proven.

### TRACE

```text
OpenDecodeSystem return
  → MI+0xADC
  → MI_AUDIO_Start reload
  → exported MApi wrapper
  → Utopia/kernel binding
  → internal _MApi gate
```

The exported Utopia wrapper decompilation directly shows:

```text
MApi_AUDIO_OpenDecodeSystem:
  UtopiaOpen(0x80000034,...)
  UtopiaIoctl(...,0x3F,...)

MApi_AUDIO_SetDecodeSystem:
  UtopiaOpen(0x80000034,...)
  UtopiaIoctl(...,0x3C,...)
```

The exact kernel `AUDIOIoctl` case that maps the local wrapper payload into the `+0x10` structure is not closed in the primary corpus.

The internal `_MApi` branch itself is verified:

```text
0x3E534C..0x3E5354  arg0==5/-1 alternate path
0x3E53B8..0x3E53C8  otherwise reject +0x10==0xFF/0
```

### RESULT

- `+0x10=0xFF` source in `MI_AUDIO_Start`: **VERIFIED**.
- First argument identity as a fixed codec `5` or `9`: **DISPROVEN**.
- Static possible range: **small decoder/slot range plus failure sentinel, conditionally supported; exact range/live value BLOCKED**.
- The `_MApi` gate being the exact function reached by the MI external call: **BLOCKED by module/export binding**.
- AC3/DTS branch selection at this gate: **BLOCKED**.

### STATUS

`CONDITIONALLY VERIFIED` for the return being a slot/index and `+0x10` source; `BLOCKED` for the live value and exact gate reachability.

**Exact opaque edge:**

```text
mik.ko undefined MApi_AUDIO_OpenDecodeSystem/SetDecodeSystem
  → loader/export binding
  → exact internal _MApi implementation and payload
```

## U2 — AV / SHM provenance

### QUESTION

Can the static corpus close the values and lifecycle of `AV+0x4D0`, `AV+0x4D8`, `AV+0x524`, `AV+0x528`, SHM33, SHM38, and SHM4D?

### PRIMARY EVIDENCE

#### AV+0x4D0 / AV+0x4D8

`MDrv_AUDIO_CheckHashkey` writes both fields at the end of its state-building sequence:

```text
0x424870  [AV+0x4D0] = r6
0x424878  [AV+0x4D8] = r8
```

The values are assembled through multiple `MDrv_AUTH_IPCheck` result branches. The code proves path-dependent writes and static constants in the surrounding branches; it does not prove the live value on stock DTS.

- `0x4D0`: used by image/profile selection and the SetSystem2 profile branch.
- `0x4D8`: used by DTS SetSystem2 byte selection and the DTS Decoder_Type range test.

#### AV+0x524 / AV+0x528

Direct setters:

```text
0x44CC9C  [AV+0x524] = r4
0x44CD4C  [AV+0x528] = r4
```

Additional SetSystem2 writes include:

```text
0x44D2B8  [AV+0x528] = 1
0x44D6F0  [AV+0x528] = r1
0x44D714  [AV+0x524] = r4
```

The reader/dispatch side is concrete:

```text
HAL_MAD_GetAudioInfo2 @0x45E34C
0x45EA6C  read primary latch from AV+0x524
secondary path reads AV+0x528
0x45F1C8  table base, indexed by latch-2
```

These are decoder-latch slot words. The exact selected slot is path-dependent; no single static value is valid for all starts.

#### SHM33 / SHM38 / SHM4D

The query paths are direct:

```text
AC3/latch 3:
0x45F234..0x45F23C  SHM33
0x45F250..0x45F258  SHM4D

DTS/latch 0xA:
0x45F6C0..0x45F6C4  SHM38
0x45F6D0..0x45F6DC  AV+0x4D8 range
0x45F6E0..0x45F6E8  SHM38 range
```

No primary TD98 producer was found that maps an ARM/SHM write directly to the live SHM33/SHM38/SHM4D values consumed here.

### TRACE

```text
CheckHashkey → AV+0x4D0/0x4D8
SetDecoder12_Type/SetSystem2 → AV+0x524/0x528
GetAudioInfo2(type=0x30) → latch table
latch 3 → SHM33/SHM4D
latch 0xA → SHM38 and AV+0x4D8 checks
```

### RESULT

- `AV+0x4D0/0x4D8` writer and consumers: **VERIFIED**.
- Exact live values: **RUNTIME-ONLY / BLOCKED**.
- `AV+0x524/0x528` latch-slot role and writers: **VERIFIED**.
- SHM33/SHM38/SHM4D live producer: **BLOCKED**.
- `AV+0x582`: no direct edge in this targeted pass; not promoted into the DTS causal set.

### STATUS

`PARTIALLY VERIFIED` for host-side field provenance; `BLOCKED` for live SHM/AV values and DSP-side producer identity.

**Exact opaque edges:**

```text
CheckHashkey branch result → live AV+0x4D8
DSP/SHM producer → SHM33/SHM38/SHM4D
```

## U3 — `0x3847E` r11 semantics

### QUESTION

What is `r11` entering, during, and after `0x3847E`, and can `r11` be known at `0x3DF79` and `0x3E19E`?

### PRIMARY EVIDENCE

```text
0x3847E  helper entry
0x384C8  loop condition uses r12
0x384CB  r11 = r1
0x384CF  r3 = [r11]
0x384DA  fill
0x384DE  r11 = r11+4
0x384E0  loop
0x38519  return
```

The `r12<=0` path skips the `0x384CB` clobber. The data-positive path definitely writes `r11`.

The local SLEIGH gives empty bodies to `bt.swst`, `bt.nop`, and `bt.inst16_4`. The epilogue encoding therefore cannot be interpreted from the supplied semantics.

### TRACE

```text
r11 entering 0x3847E: inherited from F_03DF03, nominally V_in unless
                    an earlier helper has changed it

r12>0 path:
  r11 = r1
  r11 advances in loop

r12<=0 path:
  explicit 0x384CB clobber skipped

return at 0x38519:
  r11 restoration is not modeled by local SLEIGH
```

The F_03DF03 call sequence has two `0x3847E` calls before `0x3DF79`:

```text
0x3DF3B  0x3847E(...)
0x3DF4D  0x3847E(...)
0x3DF79  store r11
```

### RESULT

- “0x3847E never clobbers r11”: **DISPROVEN**.
- `r11` at `0x3DF79`: **helper/path-dependent; not proven V_in**.
- `r11` at `0x3E19E`: **epilogue-dependent**.
- A definitive `P=V_in/B0/another pointer` result is not available from local semantics.

### STATUS

`CONDITIONALLY VERIFIED` for a reachable clobber; `BLOCKED` for return-visible liveness.

**Exact opaque edge:**

```text
0x3847E/0x38519 epilogue (bt.swst + bt.inst16_4)
  → r11 at 0x3DF79
  → P at 0x3E19E
```

## U4 — `1414C` constructor / initialization

### QUESTION

Who first creates or fills `[B0+0x1414C]`?

### PRIMARY EVIDENCE

```text
0x3DF2D  V_in = [B0+0x1414C]
0x3DF35  0x130D25(B0,0,0x16F00)
...
0x3DF79  [B0+0x1414C] = r11
```

The normal `0x130D25` path zeroes the field; its global-mode branch does not have equivalent full-range semantics. No constructor, pool assignment, pointer-table publication, or direct `B0` assignment was found upstream.

### TRACE

```text
B0 object
  → conditional broad fill at 0x3DF35
  → helper-dependent r11 operations
  → field store at 0x3DF79
  → later state-machine reads
```

The fixed `B0=0xBA0` call path still goes through the same helper; it does not create a distinct self-pointer constructor.

### RESULT

- Temporary zero fill: **CONDITIONALLY VERIFIED**.
- Final field value: **BLOCKED**.
- Explicit `V_in=B0`: **NOT FOUND**.
- Upstream pool/constructor/workspace source: **NOT FOUND**.

### STATUS

`BLOCKED`.

**Exact opaque boundary:**

```text
[B0+0x1414C] before 0x3DF35
  → 0x3DF35/0x3DF3B/0x3DF4D/0x3DF59/0x3DF79
  → field after 0x3DF79
```

## U5 — D_P / D_B merge

### QUESTION

Can a later state-machine path merge `D_P=P+0x14CCC` and `D_B=B0+0x14CCC`?

### PRIMARY EVIDENCE

`F_3E961` is one-action-per-entry. On the direct path:

```text
0x3EF64  F_0411A6(B0)
0x3F1AE  later F_03E0DD(...)
```

`F_0411A6` always constructs:

```text
0x41275  D_B=B0+0x14CCC
```

`F_03E0DD` constructs:

```text
0x3E341  D_P=P+0x14CCC
```

No global pointer publication, descriptor copy, or table store was found that establishes `D_P=D_B` after a nonzero P difference. A later F0411A6 can read D_B fields, but that is not a merge proof.

### TRACE

```text
P-track: P → D_P → F45DE0 → D_P+F8/FC
B-track: B0 → D_B → F45FD1 → F4618E(D_B)
```

### RESULT

- Merge on all paths: **DISPROVEN**.
- Conditional merge if `P=B0`: **POSSIBLE, not proven**.
- Later global publication/copy edge: **NOT FOUND**.

### STATUS

`REMAIN DISTINCT` / `CONDITIONAL` on `P=B0`.

**Dependency:** U3/U4 must close before U5 can be upgraded.

## U6 — F8/FC → delayed S

### QUESTION

Does any pointer or copy publish `D+0xF8/0xFC` into `S=*(R+0x174)`?

### PRIMARY EVIDENCE

The delayed consumer begins:

```text
0x29FE9  S = *(R+0x174)
0x29FF0  read *S
0x2A04F  sync path
0x2A1B3  F_28A33(dst=R+0x140,src=S,len=R+0x178)
0x2A4ED  alternate dst=R+0xFC
```

The DEC path constructs and fills local A/B pointers. No direct writer in F375BE/F3E961/F0411A6 installs:

```text
[R+0x174] = D+0xF8
[R+0x174] = D+0xFC
[R+0x174] = A
[R+0x174] = B
```

The possible numeric relations are not pointer provenance.

### TRACE

```text
D_P/D_B → F8/FC
F8/FC → [unproven publication] → S
S → F28A33 → R+0x140/R+0xFC
```

### RESULT

- F8/FC local processing: **VERIFIED**.
- `R+0x174` is a separate source pointer: **VERIFIED**.
- F8/FC→S publication: **NOT FOUND**.
- Same-offset/same-helper equivalence is insufficient.

### STATUS

`BLOCKED`.

**Exact opaque edge:**

```text
[D+0xF8/0xFC] → *(R+0x174)
```

No source/target register publication is identified.

## U7 — F_45FD1 indirect targets

### QUESTION

Can the `F_45FD1` indirect table be resolved, and does any target consume A/B/F8/FC?

### PRIMARY EVIDENCE

```text
0x46053  r23=0x150000
0x46057  r23-=0x4804
         table base=0x14B7FC
0x4605B  r3<<=2
0x4605E  r3+=r23
0x46060  r23=*(table)
0x46063  jr r23
```

The selector is bounded `0..13` by `0x4604F`. The 14 LE table words are:

```text
index  value
0      0x00046000
1      0x00046144
2      0x0004613C
3      0x00046134
4      0x00046000
5      0x00046000
6      0x0004612C
7      0x00046124
8      0x0004611C
9      0x00046000
10     0x00046000
11     0x00046114
12     0x0004610C
13     0x00046065
```

These are local `F_45FD1` labels. The `0x46065` path writes descriptor scalar fields and rejoins the epilogue; the `0x4610C..0x46144` paths write scalar constants/control fields and rejoin `0x46069`. No target loads `D+0xF8/0xFC`, passes A/B, calls SND, or submits a ring/DMA object.

### TRACE

```text
selector 0..13
  → table 0x14B7FC
  → local scalar/control label
  → rejoin F_45FD1
```

### RESULT

- Table base and 14 entries: **VERIFIED**.
- Selector bound: **VERIFIED**.
- Targets are local control labels: **VERIFIED**.
- Hidden A/B/F8/FC consumer through table: **DISPROVEN**.

### STATUS

U7 is substantially closed. It is causally relevant to descriptor control, not to the A/B→S question.

## U8 — F_029867 delayed consumer chain

### QUESTION

What is the complete static path from `S=*(R+0x174)` to a ring or output submission?

### PRIMARY EVIDENCE

```text
0x29FE9..0x2A04F  load S and recognize sync signatures
0x2A1AF/0x2A1B3  F_28A33(dst=*(R+0x140),src=S,len=*(R+0x178))
0x2A4E9/0x2A4ED  alternate F_28A33(dst=*(R+0xFC),src=S,len=*(R+0x178))
```

`F_13602` resolves a ring object at `*(R+0x1E4)` through `F_13041`; it copies supplied data into the ring base at the write index, advances/wraps `ring+0x10`, and updates owner counter `*(R+0x9C)`.

`F_13B08` mirrors this through `F_1318B` using ring object `*(R+0x1D4)` and owner counter `*(R+0x94)`.

A separate call at `0x29D1B` enters `F_28AFE(context,1,config,ptr,R+0xF8)`, which chooses `F_14F63` or `F_1476A` based on mode and converges on `F_142F3`, where queue indexes and MMIO-facing state are updated.

### TRACE

```text
S → F28A33 → R+0x140 or R+0xFC
  → F13602/F13B08 → ring objects

separate:
R+F8 → F28AFE → F14F63/F1476A → F142F3
```

The two tracks are not connected by a proven pointer/value identity.

### RESULT

- S→byte-order copy: **VERIFIED**.
- Ring/cache object and wrap/index updates: **VERIFIED**.
- S track to F28AFE/F14F63 track: **NOT PROVEN**.
- F8/FC to S: **BLOCKED** (U6).

### STATUS

`CONDITIONALLY VERIFIED` for the S/ring path; `BLOCKED` at its connection to DEC F8/FC and output submission.

## U9 — SND buffer → transport

### QUESTION

What happens after the AC3/DTS `0x1078A9` copies?

### PRIMARY EVIDENCE

AC3:

```text
0x1EDC8  source = *(r10+0x4860)
0x1EDCF  length = 0xA00
0x1EDD6  0x1078A9(dst=r14+8,src,len)
```

DTS:

```text
0x1F8BC  source = r13
0x1F8BF  length = 0x7F0
0x1F8C5  0x1078A9(dst=r14+8,src=r13,len)
```

`0x1078A9` is a generic word/byte copy loop. The SND output manager is shared/interleaved: both AC3 and DTS paths reach common helpers such as `0xCC9B`/`0xCB50`; DTS additionally reaches `0xCF86`/`0xCAB4`. Those calls do not by themselves establish codec-exclusive ownership.

No exact downstream reader of the `r14+8` payload was found that carries it to a physical transmitter.

### TRACE

```text
SND source → r14+8 local buffer
  → generic copy/helper
  → downstream consumer unresolved
```

### RESULT

- AC3/DTS source, destination, and length differences: **VERIFIED**.
- Local output-buffer construction: **VERIFIED**.
- Direct physical transport: **BLOCKED**.
- F8/FC source identity: **NOT PROVEN**.

### STATUS

`BLOCKED` at the transport boundary; local SND path `VERIFIED`.

## U10 — TX ring producer/consumer

### QUESTION

Is `HAL_AUDIO_GetDspMadBaseAddr(1)+0x672000` an output producer, monitor, or internal buffer?

### PRIMARY EVIDENCE

`MDrv_AUDIO_Dump_SpdifNpcm_Monitor @0x369408` constructs:

```text
ring = HAL_AUDIO_GetDspMadBaseAddr(1)+0x672000
span = 0x66000
```

It reads status/control bytes and maps/copies portions of the ring to a file through `MsOS_MPool_PA2KSEG1` and `MDrv_AUDIO_FileWrite`. No static writer tying this exact ring to `F_029867`, `F_28A33`, `F_13602`, `F_13B08`, DEC F8/FC, or SND `0x1078A9` was found.

The DSP ring objects used by `F_13602/F_13B08` are derived from separate object-relative ring bases; their identities are not proven equal to the monitor’s `+0x672000` object.

### TRACE

```text
monitor function
  → DSP MAD base +0x672000
  → status/control
  → file dump
```

### RESULT

- Ring construction and monitor read path: **VERIFIED**.
- Producer from DEC/SND output: **NOT FOUND**.
- Physical transmitter identity: **UNPROVEN**.

### STATUS

`OUTPUT MONITOR` / `INTERNAL DEBUG DUMP`; physical relation `BLOCKED`.

## Standard vs MS12V22

The targeted comparison did not establish one-to-one identity for the unresolved U3–U10 functions. The stock profile loader and SetSystem2 profile branch are verified, but common names/strings and nearby control flow do not prove identical function bodies or identical runtime behavior.

```text
MDrv_OpenDecodeSystem thunk/callback:       match not established
0x3847E equivalent:                        match not established
1414C constructor area:                     match not established
A108 family:                                match not established
F_45FD1/F_4618E counterparts:             match not established
F_029867/SND/TX counterparts:              match not established
```

The only verified standard/MS12 difference relevant here is profile/image selection and the conditional DTS engine byte `0x97`. No “STANDARD fixes DTS” conclusion is made.

## Targeted TCL Differential

TCL/T615 was not used. The remaining blockers are concrete TD98 structural edges:

- kernel MApi export/loader binding;
- `0x3847E` epilogue semantics;
- `[B0+0x1414C]` upstream constructor;
- F8/FC→S publication;
- SND `r14+8` downstream reader.

None was closed by a TCL string, name, or address comparison, so no TCL evidence was introduced.

## Causal Dependencies Between Unknowns

```text
U1 MApi handle/gate
  ↓
U2 AV/latch/SHM acceptance
  ↓
Decoder_Type / SetSystem2 state
```

```text
U3 r11 epilogue semantics
  ↓
U4 final 1414C value
  ↓
U5 P/B0 and D_P/D_B identity
  ↓
U6 F8/FC → S publication
  ↓
U8 delayed S/ring path
  ↓
U9 SND/transport path
```

```text
U7 F45FD1 local dispatch
  → descriptor scalar/control state
  → not currently a proven A/B consumer
```

## Minimal DTS Success Preconditions

Static evidence supports this reduced set:

```text
C1  DTS parser selection: format 0x0B/0x0C -> parser "dts"
C2  codec ID 9 reaches MI
C3  MI decoder-system 0xB
C4  the exported MApi/SetDecodeSystem binding and its handle/+0x10 gate accept the call
C5  SetSystem2 writes the DTS engine/latch state expected by the selected path
C6  Decoder_Type query has latch 0xA, AV+0x4D8 in {1,2,3}, and SHM38 in [1,7]
C7  F45FD1 selects the DTS data-type branch when the descriptor selector reaches 0x200
C8  the selected SND frame path constructs the 0x0B/0x3F80 local fields and copies them
C9  if the claim depends on DEC F8/FC, an additional F8/FC→S publication edge is required
C10 physical output requires a downstream reader/transport edge not yet present in the static corpus
```

Unknown but causally relevant:

```text
U1, U2, U3, U4, U5, U6, U8, U9
```

Currently disconnected or weakly connected:

```text
U7: local descriptor control dispatch, not an A/B consumer
U10: monitor/dump ring, no proven producer link
```

## Updated End-to-End Graph

```text
format
  -> parser
  -> codec ID
  -> MI/MApi handle and +0x10 gate       [U1]
  -> SetSystem2 / AV state              [U2]
  -> Decoder_Type / SHM acceptance     [U2]
  -> F45FD1 local dispatch              [U7 closed]
  -> F45DE0/F4618E                     [U3/U4/U5]
  -> F8/FC                              [U5]
  -> S publication                     [U6 blocked]
  -> F029867/F28A33/ring              [U8]
  -> SND local copy                    [U9]
  -> physical transport                [blocked]
  -> monitor ring                      [U10 separate]
```

## First Remaining Opaque Boundary

The first causally meaningful unresolved edge is:

```text
F8/FC A/B pointees
  → [publication/copy/descriptor]
  → S=*(R+0x174)
```

The direct F4618E caller boundary remains:

```text
0x4128D
```

but the delayed S track continues after that return. The U3/U4 upstream boundary remains:

```text
0x3DF03 return → 0x3E19E
```

with `r11` not closed by local SLEIGH semantics.

## Corrections

1. U1 must not treat `MI+0xADC` as codec 5/9; it is an OpenDecodeSystem return propagated through a function-pointer/export boundary.
2. `+0x10=0xFF` is verified as the MI start-struct source, but the exact `_MApi` gate branch for AC3/DTS is not yet proven through the module binding.
3. U3’s “no clobber” interpretation is disproven by the reachable `0x384CB/0x384DE` path; return-visible `r11` remains blocked.
4. U7’s indirect table is `0x14B7FC`, not the earlier `0x107FC` annotation.
5. U7 is local scalar/control dispatch, not a hidden A/B consumer.
6. U6 must not equate the separate `S` pointer with D+0xF8/0xFC merely from downstream calls.
7. SND `0x1078A9` is a generic copy, not a physical queue or transmitter.
8. The TX ring remains a software monitor/dump surface; no physical-sink claim is made.

## Highest-Value Next Step

The smallest next static step is to close the two edges that determine whether the reconstructed chain can reach output:

1. resolve the kernel `MApi` export/loader binding and live first-argument range;
2. resolve the F8/FC→S publication edge, while separately closing the `0x3847E` epilogue and `[B0+0x1414C]` constructor.

If those edges remain blocked, the correct final result is not a patch or license theory: the TD98 corpus has reached a static object-identity/loader boundary before physical output.

## Preservation Statement

This is a new artifact. No previous report, raw log, binary, Ghidra project, script, hash, or historical correction was modified. No runtime, ADB, device, playback, memory inspection, patch, flash, reboot, module reload, configuration change, or TCL/T615 evidence was used.
