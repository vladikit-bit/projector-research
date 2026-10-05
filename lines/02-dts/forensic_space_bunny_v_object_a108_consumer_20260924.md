# SPACE BUNNY DTS/SPDIF — V OBJECT, A108 FAMILY, AND FIRST CONSUMER

**Date:** 2026-09-24  
**Pass:** static/corpus-only, read-only  
**Prior V/B report:** `forensic_space_bunny_v_to_b_identity_20260924.md`  
**Primary DEC listing:** `aeon_validate/dec_work/dec32_clean.txt`  
**Primary DEC image:** `aeon_validate/dec_work/dec_full.bin`  
**Runtime:** none; no ADB, device, playback, runtime memory, breakpoint, patch, flash, reboot, or module reload  
**TCL/T615:** not searched or used

---

## Executive summary

This pass found two primary-evidence corrections to the previous V/B report.

1. The field `[B0+0x1414C]` has a relevant **generic range writer** that the earlier literal-displacement pass missed:

   ```text
   0x3DF35: 0x130D25(B0, 0, 0x16F00)
   ```

   On the normal fill path this covers both `[B0+0x1414C]` and `[B0+0xA108]`. The helper has a separate global-mode branch, so the range write is conditional on the global fill mode.

2. The source register used by the unconditional store at `0x3DF79` is **not statically proven to be the original V**. `0x3847E`, called before `0x3DF79`, has a reachable path that writes `r11`. The supplied SLEIGH leaves `bt.swst` and `bt.inst16_4` semantics empty, while `aeon.cspec` describes `r11` as callee-preserved. Therefore:

   ```text
   V_in = value loaded from [B0+0x1414C] at 0x3DF2D
   U    = r11 at the 0x3DF79 store
   P    = r11 observed by F_03E0DD at 0x3E19E
   ```

   `U` and `P` are not safely interchangeable with `V_in`. The previously stated V-based D/T/A/B equations remain a **conditional decoded-flow track**, not an unconditional primary fact.

The A108 family is now clearer:

```text
0x3EA38: [B0+0xA108] = B0+0x7C4       (verified)
0x3E1A6: [B0+0xA108] = P+0x7C4       (conditional on P)
```

The first direct `F_0411A6` handoff in the `F_03E961` path receives the verified B0 variant because `0x3EA38` executes before the `F_0411A6` call at `0x3EF64`. Later re-entry is last-writer-dependent.

The downstream result is also bounded:

```text
F_45DE0 receives D_P/T_P
F_0411A6 constructs D_B and calls F_45FD1(D_B,A108)
F_0411A6 calls F_4618E(D_B)
```

There is no direct `F_4618E(D_P)` call. No A/B pointer is formed from A108. The first post-`F_4618E` A/B consumer is not found; the exact loss-of-provenance boundary is `0x4128D`.

---

## 1. Alias-normalized `0x1414C` writer inventory

The search used effective-address forms, not only the literal `0x414C` displacement. The following symbols are kept separate:

```text
B0     = F_03E0DD entry r4
V_in   = *(B0+0x1414C) loaded at 0x3DF2D
P      = r11 used by F_03E0DD at 0x3E19E
U      = r11 used by the 0x3DF79 store
```

### 1.1 Relevant B0-derived writers

| Site | Effective destination | Base provenance | Status |
|---|---|---|---|
| `0x3DF35` → `0x130D25` | normal mode: `[B0,B0+0x16F00)`; includes `[B0+0x1414C]` and `[B0+0xA108]` | `r3=B0`, `r4=0`, `r5=0x16F00` | `VERIFIED RELEVANT WRITER` on normal path; `BLOCKED` on global-mode path |
| `0x3DF79` | `[B0+0x1414C]` | `r10=B0+0x10000`, displacement `0x414C`; source is `r11` after intervening helpers | `VERIFIED RELEVANT WRITER`; source `OPAQUE`, not proven `V_in` |

The normal `0x130D25` range is large enough to cover the target:

```text
0x1414C < 0x16F00
```

The global-mode branch is not equivalent:

```text
0x130D25:
  0x130D29  load global flag at 0x1DC594
  0x130D2D  if flag != 0 -> 0x130DD4
  0x130DD4..0x130DE0  one-byte store path
```

The image initially contains zero at the flag address, but static code also contains later flag writers, including `0x17659`, `0x17C40`, and `0x17FF0`. Runtime/global state is not being inferred. The range writer is therefore conditional, not unconditional.

### 1.2 Related V-object writer

At `0x3DF59`:

```text
0x130D25(V_in, 0, 0xAC6C)
```

On the normal path this zero-fills the object pointed to by `V_in`, including object offsets such as `V_in+0xF4`, `V_in+0xA2`, `V_in+0xB0`, and `V_in+0x7C4`. It does not itself initialize the pointer field `[B0+0x1414C]`; the later `0x3DF79` store is the direct field writer.

This call is also subject to the same `0x130D25` global-mode branch.

### 1.3 Generic helper candidates with unproven base aliases

The following calls can write a range that would cover `[B0+0x1414C]` if their runtime pointer fields satisfy an additional offset condition. The conditions are not proven:

| Call | Helper destination | Required unresolved condition | Status |
|---|---|---|---|
| `0x3DF3B` → `0x3847E` | `[P1,P1+0xB338)`, `P1=*(B0+0xA128)` | `P1` in the interval ending at `B0+0x1414C` | `BLOCKED` |
| `0x3DF4D` → `0x3847E` | `[P2,P2+0xB338)`, `P2=*(B0+0xA148)` | same form | `BLOCKED` |
| `0x3DF47` → `0x130D25` | `[P3,P3+0xAC6C)`, `P3=*(B0+0xA12C)` | `P3` range contains target | `BLOCKED` |
| `0x3DF65` → `0x130D25` | `[P4,P4+0x7E60)`, `P4=*(B0+0x14CC8)` | `P4` range contains target | `BLOCKED` |

The calls at `0x3E23A`, `0x3E38A`, `0x3FA18`, `0x3FE84`, and `0x3FF23` have runtime-derived or unrelated bases. Their small fill/copy ranges do not prove an alias to the target.

### 1.4 Arithmetically excluded helpers on the B0 path

The following visible destinations do not include `[B0+0x1414C]`:

```text
0x3E123: [B0+0x148, B0+0x278)
0x3E12F: [B0, B0+0x130)
0x3E2B7: starts at B0+0x142BC       (above target)
0x3E2D1: starts at B0+0x14560       (above target)
0x3E303: [B0+0x1481C, B0+0x14B04)  (above target)
0x3E344: F_45DE0 descriptor/buffer region above target
0x3E8B2: B0-based copy of length 0x130
```

The `0x3E2B7`/`0x3E2D1` destinations are above the target, not below it.

### 1.5 Absolute and compensating-address candidates

The following are retained as numeric-alias candidates, not verified B0 writers:

| Site(s) | Effective form | Reason not promoted |
|---|---|---|
| `0x10832D` | absolute `0x3BEB4` (`0x40000-0x414C`) | would require `B0=0x27D68`; no relevant path supplies that fixed base |
| `0x2D755`, `0x2D7EB`, `0x2D8A9`, `0x2D8D1`, `0x2D8F1`, `0x2D91C`, `0x2D94F` | `*(rbase+0x10)+0x1414` | requires an unrelated `rbase` to equal `B0+0x38` |
| `0x87E72`, `0x87F61` | `rbase+0x1414` | requires `rbase=B0+0x38`; base provenance unrelated |
| `0x9FB7E`, `0x9FBD3` | `rbase+0x1414` | same unresolved compensating-base condition |
| `0x1574AD` | `r0-0x1414` | requires `r0=B0+0x15560`; unrelated base |
| `0xFD1C3`, `0xFE61D` | constructs `rbase+0x1414` without a proven consuming store | `BLOCKED`/dead-value candidates |

Classification for all rows above: `NUMERIC ALIAS CANDIDATE` or `BLOCKED`, not `VERIFIED RELEVANT WRITER`.

### 1.6 Same-offset accesses that are not B0-field accesses

The exact `0x414C` loads at:

```text
0x3D714, 0x3DEDB, 0x3DF2D,
0x3F78E, 0x3F923, 0x3F93D, 0x3F948,
0x3FC06, 0x3FC86
```

are readers of a field/object selected by their respective bases. They are not additional writers.

Negative-displacement references at `0x0D2154`, `0x0E8DF5`, `0x0E8E9E`, `0x0E9222`, `0x0E9278`, `0x0E9378`, and `0x1086F9` use other bases. They are `UNRELATED SAME-OFFSET ACCESS` or `BLOCKED`, not proven B0-field accesses.

`0x1CE40D` is `muli #0x414C`, not a memory operation.

### 1.7 Writer conclusion

```text
Direct effective writer to [B0+0x1414C]:       0x3DF79
Conditional normal-mode range writer:            0x3DF35 -> 0x130D25
Direct assignment of B0 to the field:           NOT FOUND
Direct assignment of the original V_in:         NOT PROVEN
Unresolved indirect/external writers:            BLOCKED
```

The natural upstream opaque boundary is the source provenance of `U`/`P` after the helper sequence, not the existence of the destination store.

---

## 2. V provenance and the `V_in → U → P` correction

### 2.1 Exact helper sequence

At `F_03E0DD`:

```text
0x3E0E6  r3 = B0
0x3E101  jal 0x3DF03
```

Inside `0x3DF03`:

```text
0x3DF2D  r11 = *(B0+0x1414C) = V_in
0x3DF35  0x130D25(B0, 0, 0x16F00)
0x3DF3B  0x3847E(...)
0x3DF47  0x130D25(...)
0x3DF4D  0x3847E(...)
0x3DF59  0x130D25(V_in, 0, 0xAC6C)
0x3DF65  0x130D25(...)
0x3DF79  *(B0+0x1414C) = r11
```

The normal fill path transiently changes the field and A108. The source register at the final store is not safely the original `V_in`.

### 2.2 Reachable `r11` clobber in `0x3847E`

The helper contains:

```text
0x384CB  r11 = r1
0x384CF  r11 used as a loop cursor
0x384DE  r11 = r11+4
...
0x38519  return
```

The branch is data-dependent, but the call site does not statically prove that the non-clobber path is taken. Thus `r11` at `0x3DF79` is:

```text
U = unknown helper-dependent value
```

not proven `V_in`.

The helper’s prologue/epilogue also contains `bt.swst`, `bt.nop`, and `bt.inst16_4`, whose supplied SLEIGH bodies are empty. `aeon.cspec` lists `r11` as unaffected, but that contract is not sufficient to erase the raw reachable clobber before the store.

### 2.3 Caller-side register ambiguity

`F_03E0DD` uses `r11` after the call:

```text
0x3E19E  r23 = r11+0x7C4
0x3E1A6  [B0+0xA108] = r23
```

The possible symbolic values are:

```text
P = V_in       if the helper leaves the loaded pointer in r11
P = B0         if the epilogue restores the callee-saved r11
P = U/other    if a helper-produced value survives
```

The primary corpus does not select one branch.

### 2.4 Upstream initialization verdict

No direct instruction assigns a new pointer value to `[B0+0x1414C]` before `0x3DF2D`. The only relevant operations are:

- conditional range zero-fill;
- helper-dependent source value at `0x3DF79`;
- unrelated/indirect candidate writes.

Therefore:

```text
V_in initialization/provenance:       OPAQUE BOUNDARY
V_in == B0:                          UNPROVEN
V_in != B0:                          UNPROVEN
P == V_in:                           CONDITIONAL
P == B0:                             CONDITIONAL
U == V_in:                           NOT PROVEN
```

The prior V-based equations are retained only as the branch where `P=V_in`.

---

## 3. V object field map

This table describes direct accesses to the object value loaded from `[B0+0x1414C]`. It does not assume that the later caller register `P` is the same object.

| Field | Reader(s) | Writer(s) | Evidence-based role/status |
|---|---|---|---|
| `V_in+0x0000` / `+0x0004` | passed through `0x682C4` at `0x3D714`, `0x3DEDB` | `0x682C4` may clear these words under its branch | neutral state/object fields; exact branch role unresolved |
| `V_in+0x00F4` | `0x3F792`, `0x3F95F` | `0x693DF` writes `low16(r24)`; `0x3DF59` normal fill zeroes the object | read/multiply use; exact write value at `0x693DF` is `OPAQUE` because `0x693D4` is unmodeled `inst16_4` |
| `V_in+0x00A2` | `0x3F950`, `0x6884D`, `0x68B3A`, `0x6937A` | no typed nonzero writer found; normal `0x3DF59` fill zeroes it | scalar/status-like field only by access shape; no semantic name assigned |
| `V_in+0x00B0` | `0x689CA` | `0x3F929` writes `1` | flag/control-shaped field; no semantic name assigned |
| `V_in+0x07C4` | A108 target when `P=V_in`; `F_45FD1` then reads `P+0,+0x10,+0x28,+0x38` | `0x3E1A6` conditionally writes `P+0x7C4` into A108 | pointer-family handoff; not an A/B pointer |
| `V_in+0x14CCC` | constructed as `D_V` only if `P=V_in` | `0x3E341` / `F_45DE0` path | descriptor candidate, conditional on P |
| `V_in+0x14E40` | constructed as `T_V` only if `P=V_in` | table slot/read path | table candidate, conditional on P |
| `V_in+0x14E44` | second table word only if `P=V_in` | table slot/read path | table candidate, conditional on P |
| `V_in+0x14EE8` | `A_P` only if `P=V_in` | pointer installation/fill path | buffer address candidate, conditional on P |
| `V_in+0x156E8` | `B_P` only if `P=V_in` | pointer installation/fill path | buffer address candidate, conditional on P |

The normal `0x3DF59` fill is the only verified broad object initializer in this pass. It does not identify who originally supplied `V_in`.

---

## 4. A108 pointer-family provenance

### 4.1 Effective A108 address

On the `F_03E961`/`F_03E0DD` paths:

```text
r10 = B0+0x10000
A108 = r10-0x5EF8 = B0+0xA108
```

The positive store at `0x3E7B7` uses a different base expression (`r10` is set from another function argument); it is not promoted to A108 without a base-identity proof.

### 4.2 Writers

| Site | Effective write | Source | Status |
|---|---|---|---|
| `0x3DF35` | normal mode zeroes `[B0+0xA108]` as part of `[B0,B0+0x16F00)` | fill value `0` | `VERIFIED CONDITIONAL RELEVANT WRITER` |
| `0x3EA38` | `[B0+0xA108] = B0+0x7C4` | `r28=B0+0x7C4` | `VERIFIED` |
| `0x3E1A6` | `[B0+0xA108] = r11+0x7C4` | `r11=P` after `0x3DF03` | `VERIFIED destination; source conditional` |

The `0x3DF35` range write can transiently clear A108 before the later `0x3E1A6` write. If the global fill mode is nonzero, the range behavior is not statically equivalent.

### 4.3 Readers and propagation

The key direct readers are:

```text
0x3EF72  r3 = *(B0+0xA108)
0x3EF76  jal 0x37D2D(P_A108)

0x4127D  r4 = *(B0+0xA108)
0x41281  r3 = D_B
0x41283  jal F_45FD1(D_B, P_A108)
```

`0x37D2D` reads `P_A108+0x24` and returns a boolean/test result. It does not load an A/B pointer.

The equivalent-displacement sweep also found the following `-0x5EF8` reader sites in the DEC state-machine regions:

```text
0x3D639, 0x3D750,
0x3EA8B, 0x3EB8A, 0x3EB95, 0x3EC38,
0x3EE33, 0x3EE9C, 0x3EED9, 0x3EF72, 0x3EF7E,
0x3EF9B, 0x3EFC3, 0x3F017, 0x3F029,
0x3F12E, 0x3F136, 0x3F145, 0x3F184,
0x3F22A, 0x3F281, 0x3F2B6, 0x3F2D4,
0x3F535, 0x3F674, 0x3F806,
0x40D60, 0x40D76, 0x40DD0, 0x40F30,
0x410B5, 0x41111, 0x4127D
```

The `0x3E...`/`0x3F...` group is tied to the B0-derived `r10` in the relevant state-machine functions. The `0x40...` group uses other bases and is not promoted to A108 without a separate base-identity proof. The positive `0x5EF8` store at `0x3E7B7` is likewise a different-base candidate.

`F_45FD1` reads only status/control fields of `P_A108`:

```text
0x45FE0  r23 = *(P_A108+0)
0x45FEB  if r23 == 1 -> 0x4600C
0x4600C  r4 = *(P_A108+0x28)
0x4600F  0x64149 copy 0x28 bytes to D_B

0x45FEE  r11 = *(P_A108+0x10)
0x45FF1  if r11 == 1 -> 0x46080
0x46080  r4 = *(P_A108+0x38)
0x46083  0x64149 copy 0x28 bytes to D_B
```

No A/B address is formed from `P_A108`.

### 4.4 A108 first-handoff ordering

On the direct `F_03E961` path:

```text
0x3EA38  write A108 = B0+0x7C4
0x3EF64  call F_0411A6
...
0x3F1AE  later call F_03E0DD
0x3F7D2  another F_03E0DD call site
```

Therefore the **first** `F_0411A6` handoff receives the B0 variant unless an unmodeled callback mutates the field. A later invocation can receive the `P+0x7C4` variant after `0x3E1A6`.

Status:

```text
A108 -> F_45FD1:              VERIFIED status/control handoff
A108 -> A/B/F8/FC identity:   BLOCKED; no such edge found
A108 first direct handoff:    B0+0x7C4 VERIFIED
A108 later source:            P+0x7C4, P not closed
```

---

## 5. V/B0 descendant alias matrix

The corrected downstream track uses `P`, not an assumed V:

```text
D_P = P+0x14CCC
T_P = P+0x14E40
A_P = P+0x14EE8
B_P = P+0x156E8

D_B = B0+0x14CCC
T_B = B0+0x14E40
A_B = B0+0x14EE8
B_B = B0+0x156E8
```

| Pair | Same-address condition | Current status |
|---|---|---|
| `D_P` / `D_B` | `P=B0` | `CONDITIONAL` |
| `T_P` / `T_B` | `P=B0` | `CONDITIONAL` |
| `A_P` / `A_B` | `P=B0` | `CONDITIONAL` |
| `B_P` / `B_B` | `P=B0` | `CONDITIONAL` |
| `D+0x174`, `+0x178` | same base condition | `CONDITIONAL` |
| `D+0xF8`, `+0xFC` | same base condition | `CONDITIONAL` |
| A108 `B0+0x7C4` / `P+0x7C4` | `P=B0` | `CONDITIONAL` |
| cross-offset A/B coincidences | require explicit `P-B0` offsets such as `±0x800`; no such relation is proven | `UNPROVEN` |

If `P=V_in`, the P-track becomes the old provisional V-track. If `P=B0`, it becomes the B0-track. The corpus does not choose between them.

The fixed-base branch is a separate concrete case, but the two paths must not be conflated:

```text
0x37992  r16 = literal 0xBA0
0x37A61  F_03E0DD(..., B0_fixed=0xBA0,...)

[B0_fixed+0x1414C] = 0x14CEC
[B0_fixed+0xA108]  = 0xACA8
```

The later `0x37ACE` helper call is reached from the success branch at `0x37986`, where `r16` still carries the earlier workspace-derived `B0_B`; it is not the fixed `0xBA0` call. Thus:

```text
0x37A61: B0_fixed path
0x37ACE: B0_B path
```

The helper still does not provide a direct `V=B0` constructor on either path.

---

## 6. F8/FC continuation: separate P and B0 tracks

### 6.1 P-based installer

The table construction in `F_03E0DD` uses the post-helper register `r11`:

```text
0x3E325  A_P = P+0x14EE8
0x3E32B  B_P = P+0x156E8
0x3E337  [B0+0x14E40] = A_P
0x3E33B  [B0+0x14E44] = B_P
0x3E33F  T_P = P+0x14E40
0x3E341  D_P = P+0x14CCC
0x3E344  F_45DE0(D_P,T_P)
```

`F_45DE0` then installs:

```text
D_P+0xF8 = T_P[0]
D_P+0xFC = T_P[4]
```

This is a verified construction **conditional on the identity of `P`**. It is not evidence that `P=V_in`.

### 6.2 B0-based consumer

`F_0411A6` constructs:

```text
0x41275  r10 = B0+0x14CCC = D_B
0x4127D  r4 = *(B0+0xA108) = P_A108
0x41283  F_45FD1(D_B,P_A108)
0x41289  F_4618E(D_B)
```

There is no direct call `F_4618E(D_P)` in the decoded corpus.

Therefore:

```text
P == B0:
  D_P == D_B; F8/FC tracks may merge, subject to field contents.

P != B0:
  D_P and D_B remain separate.
  F_45DE0 installs into D_P.
  F_4618E consumes D_B.
  The source of D_B+0xF8/0xFC is not established by the V/P installer.
```

`F_45FD1` copies status/control data into `D_B`; it does not install A/B pointers from A108.

### 6.3 F8/FC field operations

The direct `F_4618E` operations are local:

```text
0x461C7/0x46201/0x46222/0x4626A/0x462A0/0x462DE
  loads D_B+0xF8 and writes/reads its pointee

0x461D6/0x46214/0x46226/0x46271/0x462A4/0x462E6
  loads D_B+0xFC and writes/reads its pointee
```

`0x130D25` fills the loaded pointers; `0x64369` writes extracted values to them. Neither helper propagates an A/B pointer to a later descriptor.

---

## 7. First post-`F_4618E` consumer

The only direct call is:

```text
0x41289  jal F_4618E(D_B)
```

Return sites:

```text
0x461C2  early return when D+0x28 != 1
0x462FE  prepared-path return
```

The direct caller resolves the return continuation to `0x4128D`:

```text
0x4128D  if r12 != 1 -> 0x411C9
0x411C9  [B0+0x14DBC] = 0
0x411CF  r3 = 1
0x411D9  return

r12 == 1 -> 0x41291 -> 0x41228
0x4122E  r3 = 0
0x41234  return
```

The `B0+0x14DBC` write is `D_B+0xF0`; it is a status/F0 cleanup, not an A/B operation.

No post-return instruction reloads:

```text
D+0xF8
D+0xFC
D+0x174
D+0x178
```

or passes A/B to another function.

The adjacent `F_45FD1` indirect branch is before `F_4618E`:

```text
0x46053  r23 = 0x15000
0x46057  r23 -= 0x4804       ; table base 0x107FC
0x4605B  r3 <<= 2
0x46060  r23 = *(0x107FC + selector*4)
0x46063  jr r23
```

The selector is bounded to `0..13`; the exact target is `BLOCKED` because the table bytes overlap mixed-width/code-like data. It is descriptor-state dispatch, not an A/B call.

**First real post-F4618E A/B consumer:** `BLOCKED`.  
**First exact control/provenance boundary:** `0x4128D`, `VERIFIED`.

---

## 8. TX dump-ring relation

The existing static result remains:

```text
MDrv_AUDIO_Dump_SpdifNpcm_Monitor
ring base = HAL_AUDIO_GetDspMadBaseAddr(1)+0x672000
span      = 0x66000
```

The function receives no `D`, `P`, `B0`, `V`, `A`, `B`, `F8`, or `FC` pointer. It uses a DSP-MAD base, global status bytes, a file/control pointer, and fixed ring arithmetic.

No new direct producer/control edge to the V/A108/F8/FC tracks was found.

```text
TX ring construction:       VERIFIED
TX -> V/A108/D/F8/FC edge: BLOCKED
TX identity with A/B:       SEPARATE
TX physical SPDIF claim:    UNPROVEN
```

The ring is not used to merge the P and B0 tracks.

---

## 9. Evidence-backed corrections

1. **Missed range writer:** the prior literal-displacement audit omitted `0x3DF35 -> 0x130D25(B0,0,0x16F00)`, which normally covers the target field and A108. It is conditional because `0x130D25` has a global-mode branch.

2. **V self-roundtrip correction:** `0x3DF79` is a verified destination writer, but its source is not proven to be the original `V_in`; `0x3847E` has a reachable `r11` clobber before the store.

3. **A108 correction:** the unconditional statement `[B0+0xA108]=V+0x7C4` is replaced by `[B0+0xA108]=P+0x7C4`, with `P` unresolved. The first direct `F_0411A6` handoff remains verified as `B0+0x7C4`.

4. **F8/FC correction:** `F_45DE0` uses the post-helper `P` track. The only decoded `F_4618E` call receives `D_B`; the two tracks merge only if `P=B0`.

5. **Post-consumer correction:** no real A/B consumer is proven after `F_4618E`; the exact boundary is `0x4128D`, not a queue/DMA/TX handoff.

6. **Fixed-base arithmetic correction:** `B0_fixed=0xBA0` gives target field `0x14CEC` and A108 slot `0xACA8`.

No prior report or artifact was edited.

---

## 10. First opaque boundary

The primary opaque boundary for this pass is:

```text
0x3DF03 return → F_03E0DD:0x3E19E
```

Evidence immediately before it:

```text
0x3DF2D  V_in = *(B0+0x1414C)
0x3DF35  conditional range fill
0x3DF3B/0x3DF4D  reachable r11-clobbering helpers
0x3DF79  [B0+0x1414C] = U
0x3E19E  r23 = r11+0x7C4
```

The static corpus does not close `U` or the epilogue-visible `P` to `V_in`, `B0`, or a concrete allocator return.

A second, downstream boundary is:

```text
0x4128D after F_4618E(D_B)
```

where A/B pointer provenance is lost and only status/F0 handling remains.

---

## 11. Highest-value next static targets

1. Resolve the `0x3847E`/`bt.swst`/`bt.inst16_4` register semantics using a primary AEON ISA description or a matching compiler-generated static call pattern. The exact question is whether `P` at `0x3E19E` is `V_in`, `B0`, or a helper-produced value.

2. Trace the upstream constructor/allocator for `[B0+0x1414C]` and the `V_in+0x7C4` subobject, without using runtime or the old license/SHM directions.

---

## Preservation statement

- This is a new report.
- No existing report, raw log, binary, Ghidra project, or historical artifact was modified.
- No runtime, ADB, device connection, playback, memory read, breakpoint, patch, firmware write, flash, or reboot was performed.
- TCL/T615 evidence was not used.
