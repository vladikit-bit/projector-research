# SPACE BUNNY DTS/SPDIF — STATIC `V → B` IDENTITY CLOSURE

**Date:** 2026-09-24  
**Mode:** static/corpus-only, read-only  
**Parent audit:** `forensic_space_bunny_full_corpus_audit_20260924_independent.md`  
**Prior downstream note:** `aeon_validate/dec_work/forensic_downstream_closure_20260924.md`  
**Runtime:** none; no ADB, device, playback, memory read, breakpoint, module reload, reboot, patch, or flash  
**TCL/T615:** not searched, extracted, compared, or used

---

## 1. Executive result

The requested identity closure does **not** produce an unconditional `V == B` proof.

For the `F_03E0DD` input base `B0`:

```text
V = *(B0 + 0x1414C)
```

The only direct store to the `B0`-derived field is a read/write round trip inside `0x3DF03`:

```text
0x3DF2D  V = *(r10 + 0x414C)
...
0x3DF79  *(r10 + 0x414C) = V
```

It does not assign `B0` to the field and does not initialize a previously absent pointer. The decoded corpus contains no direct instruction that explicitly stores `B0` into `[B0+0x1414C]`.

The result by requested path is:

| Path | `V == B0` result | Strongest static statement |
|---|---|---|
| `F_0375BE` direct `F_03E0DD` at `0x37810` | `UNPROVEN` | `V` is inherited from the field; no visible preceding initializer |
| `F_0378A5` first `F_03E0DD` at `0x37986` | `UNPROVEN` | Visible fills do not establish the field value |
| `F_0378A5` second `F_03E0DD` at `0x37A61` | `UNPROVEN` | First call self-writes the old `V`; no visible intervening assignment |
| `F_03E961` nested `F_03E0DD` at `0x3F1AE` | `UNPROVEN` | `F_03E961` writes `B0+0xA108 = B0+0x7C4`, then `F_03E0DD` may replace that slot with `V+0x7C4` |
| `F_03F44F` / `0x3F7D2` call site | `UNPROVEN` | No direct field writer in the preceding bounded interval |
| Separate `0x3DF03` call at `0x37ACE` | Not the `F_0378A5` `B0` field on the visible path | It operates on a separately computed base `r16` |

Therefore the only valid same-instance statement is conditional:

```text
If V == B0, then
  F_03E0DD → F_45DE0 → D+0xF8/0xFC → F_4618E
describes one descriptor/buffer instance.

If V != B0, the code uses a V-based descriptor and V-based buffer
addresses, while the B0-based table slots are a separate object.
```

The static corpus does not justify choosing either branch for a live state.

---

## 2. Scope, notation, and evidence boundary

### 2.1 Notation

`B0` is the second argument received by `F_03E0DD` at entry. It is deliberately not called `B` in the derivation, because `B` is also used below for the second F8/FC buffer.

```text
B0 = F_03E0DD.r4 at entry
V  = *(B0 + 0x1414C)
```

The helper receives `B0` in `r3` and constructs `r10 = B0 + 0x10000`; therefore the listing displacement `0x414C(r10)` is exactly:

```text
(B0 + 0x10000) + 0x414C = B0 + 0x1414C
```

This uses the authoritative AEON listing and the load/store displacement semantics, not a raw suffix alone.

### 2.2 Primary evidence

| Evidence | Use |
|---|---|
| `aeon_validate/dec_work/dec32_clean.txt` | authoritative DEC instruction listing |
| `aeon_ghidra_public/aeon/data/languages/aeon.slaspec` | immediate/load/store field interpretation |
| `aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc` | instruction semantics |
| `aeon_ghidra_public/aeon/data/languages/aeon.cspec` | call-visible register preservation |
| `spdif_audio_investigation/libs/libutopia.so` | TX monitor disassembly and symbol metadata |
| raw DEC listing and SLEIGH cross-checks | disambiguation of candidate writers |

The phrase “every writer” below means every **statically identifiable writer in the checked decoded corpus**, including the bounded direct/indirect call paths leading to each `F_03E0DD` call. A dynamic constructor, opaque allocator, external firmware component, or data-mediated write not represented by those instructions cannot be proven absent.

---

## 3. Exhaustive direct-field reference inventory

A full search of the authoritative listing found 20 textual `0x414C` references. They divide into the following classes.

### 3.1 `B0`-derived field reads

These use the same `r10 = B0 + 0x10000` form as `0x3DF03`:

```text
0x3D714  bg.lwz  r3,0x414c(r10)
0x3DEDB  bg.lwz  r3,0x414c(r10)
0x3DF2D  bg.lwz  r11,0x414c(r10)
0x3F78E  bg.lwz  r23,0x414c(r10)
0x3F923  bg.lwz  r3,0x414c(r10)
0x3F93D  bg.lwz  r3,0x414c(r10)
0x3F948  bg.lwz  r23,0x414c(r10)
0x3FC06  bg.lwz  r3,0x414c(r10)
0x3FC86  bg.lwz  r3,0x414c(r10)
```

They read the field or read `V` and then use `V` as an object pointer. None stores `B0` into the field.

Observed post-load uses include `V+0xF4`, `V+0xA2`, and `V+0xB0` in the later state-machine code. That is evidence that `V` is treated as a pointer to an object; it is not evidence that the object is `B0` itself.

### 3.2 The only direct `B0`-derived field store

```text
0x3DF2D  bg.lwz  r11,0x414c(r10)
...
0x3DF79  bg.sw_0 0x414c(r10),r11
```

The source register is the same field value loaded at `0x3DF2D`. This is a self-round-trip:

```text
V_old → r11 → field
```

It is not a constructor and cannot prove `V == B0`.

### 3.3 Negative-displacement references at other bases

The listing also contains:

```text
0x0D2154  bg.lwz r24,-0x414c(r11)
0x0E8DF5  bg.lwz r23,-0x414c(r26)
0x0E8E9E  bg.lwz r23,-0x414c(r25)
0x0E9222  bg.lwz r23,-0x414c(r13)
0x0E9278  bg.lwz r23,-0x414c(r13)
0x0E9378  bg.lwz r23,-0x414c(r13)
0x1086F9  bg.lwz r25,-0x414c(r23)
```

Their base registers are not the `F_03E0DD` `r10 = B0+0x10000` expression. They are separate object/offset candidates and are not promoted to writers of `[B0+0x1414C]` without a separate base-identity proof.

### 3.4 Absolute-address candidate at `0x10832D`

The only other direct store containing the negative `0x414C` displacement is:

```text
0x108329  movhi r23,0x4          ; r23 = 0x40000
0x10832B  add   r3,r23          ; r3 = 0x40000
0x10832D  sw    -0x414c(r3),r10 ; [0x3BEB4] = 0
```

This writes absolute address `0x3BEB4`. It would target the requested field only under the special numeric condition:

```text
B0 + 0x1414C = 0x3BEB4
B0 = 0x27D68
```

No relevant `F_03E0DD` path in the checked call graph supplies the fixed value `0x27D68`; the relevant bases are input-derived pointers plus `0xBA0`, `0x10000`, or other small offsets. The store is therefore retained as a **numeric-alias candidate**, not counted as a proven relevant-path writer.

### 3.5 Non-memory constant

```text
0x1CE40D  bg.muli_maybe r20,r11,0x414c
```

This is a multiply immediate, not a load or store. It is not a field writer.

### 3.6 Direct writer conclusion

```text
Direct B0-field stores found:                  1
That store assigns B0:                         NO
That store initializes from a new value:       NO
Other absolute/negative-base stores linked to
the relevant B0 object:                       NOT PROVEN
```

---

## 4. `0x3DF03` is a user, not a constructor, of `V`

`0x3DF03` begins with `r3` as its object base. In the `F_03E0DD` call, `r3 = B0`:

```text
0x3E0E6  mov r3,r4              ; r3 = B0
0x3E101  jal 0x3DF03
```

Inside the helper:

```text
0x3DF0E  add  r10,r3,r23       ; r10 = B0 + 0x10000
0x3DF2D  lwz  r11,0x414c(r10)  ; r11 = V
...
0x3DF79  sw   0x414c(r10),r11  ; write V back
```

The intervening `0x130D25` and `0x3847E` operations use the loaded object pointers, including `V`; they do not assign `B0` to the field. The helper therefore has the static form:

```text
initialize/reformat object pointed to by an already-existing V
write the same V pointer back
```

That is different from:

```text
V = B0
```

No direct constructor instruction for the field was found.

---

## 5. The second `0x3DF03` call is a different-base operation

The complete direct-call search finds only two calls to `0x3DF03`:

```text
0x37ACE  in F_0378A5
0x3E101  in F_03E0DD
```

At `0x37ACE`, the argument is `r3 = r16`, where `r16` is built in the surrounding `F_0378A5` state from the input-derived `r12`, a helper return, and fixed offsets. The later `F_03E0DD` base is built separately as:

```text
r15 = ALIGN(r12,4)
B0  = r15 + 0xBA0
```

The `0x37ACE` path does not assign `r16` to `B0`, and no intervening instruction creates that equality. Under the usual constant return of `0x37D12`, the computed pointer is far beyond the `B0` expression; even without relying on that return value, the two expressions have no static equality edge.

Status for this call:

```text
writer of [r16+0x1414C]:       VERIFIED
writer of [B0+0x1414C] on the
visible F_0378A5 path:          NOT PROVEN; expressions are separate
```

---

## 6. Initialization search

### 6.1 What was searched

The search covered:

1. every direct `0x414C` load/store in the DEC listing;
2. every direct caller of `0x3DF03`;
3. every direct caller of `F_03E0DD`;
4. direct `0x130D25` and `0x130E0D` calls in the relevant function intervals;
5. the B0-derived A108 construction and the later V-based overwrite;
6. the separate absolute candidate at `0x3BEB4`.

### 6.2 What was found

The first static read of `V` on the `F_03E0DD` path is:

```text
0x3DF2D  r11 = *(B0 + 0x1414C)
```

There is no preceding direct instruction in the `F_03E0DD` prologue that defines `r11` as `B0` or stores `B0` into the field.

The first visible write after that read is the self-store at `0x3DF79`.

No direct `B0 → field` assignment exists.

### 6.3 What remains opaque

A value may have been placed in the field by:

- an earlier object constructor not present in the bounded call path;
- an indirect allocator or callback;
- a data-mediated copy whose source provenance is not present in the decoded caller;
- an external firmware/boot component.

Those possibilities explain why the result is `OPAQUE`, not “V is zero” and not “V equals B0.”

---

## 7. Intervening-call and state-machine path matrix

### Path P1 — `F_0375BE` direct path

```text
F_0375BE(R, ...)
  ├─ F_03E961(R, W=R+0xBA0)       [0x37687]
  ├─ 0x3D600 / 0x3D62C / 0x3D25C  [0x377AC..0x377F2]
  └─ F_03E0DD(R, W=R+0xBA0)       [0x37810]
```

`B0 = R+0xBA0`.

The helper calls manipulate fields derived from the context and the A108 family, but no direct store to `[B0+0x1414C]` is present. The first `V` load remains inside `0x3DF03`.

**Result:** `V == B0` is `UNPROVEN`.

### Path P2 — `F_0378A5` first initialization call

```text
F_0378A5(...)
  ├─ 0x130D25(r15, 0, 0x7B40)
  ├─ allocator/filler helpers
  ├─ 0x40269(r15)                ; visible 0xBA0 workspace operation
  └─ F_03E0DD(r15, r15+0xBA0, r15) [0x37986]
```

Here `r15 = ALIGN(r12,4)` and `B0 = r15+0xBA0`.

The visible first zero fill ends at `r15+0x7B40`, while the requested field is:

```text
B0 + 0x1414C = r15 + 0x14EEC
```

so that first visible fill does not establish the field value. The later visible fills use external/derived buffer pointers rather than a proven B0 range.

**Result:** `V == B0` is `UNPROVEN`.

### Path P3 — `F_0378A5` second initialization call

The first `F_03E0DD` call can only self-write the already-read `V` at `0x3DF79`. Between the two calls, the visible operations are allocator/filler calls, a `0x537C6` helper, and another workspace operation. No direct field store is visible.

The second call therefore still begins with the inherited `V`, unless an opaque helper changes it.

**Result:** `V == B0` is `UNPROVEN`; the visible self-store is not an initialization.

### Path P4 — `F_03E961` nested path

At `F_03E961` entry:

```text
r11 = B0
r10 = B0 + 0x10000
r14 = outer R
```

Before the nested `F_03E0DD` call, the function explicitly writes:

```text
0x3EA38  [r10-0x5EF8] = r28
```

With `r28 = r11+0x7C4`, this is:

```text
[B0+0xA108] = B0+0x7C4
```

At `0x3F1AE`, `F_03E0DD` later loads `V` and overwrites the same slot with:

```text
[B0+0xA108] = V+0x7C4
```

This is important: the code does not force the A108 pointer to remain `B0+0x7C4` across the installer. It permits a different V-derived pointer.

**Result:** `V == B0` is `UNPROVEN`; the A108 overwrite is direct evidence that the implementation carries two possible pointer identities.

### Path P5 — `0x3F7D2` call site

The preceding bounded interval contains state-machine helpers but no direct `0x414C` store and no direct generic fill/copy call whose destination is proven to be `[B0+0x1414C]`.

**Result:** `V == B0` is `UNPROVEN`.

### Path P6 — helper-only `0x37ACE`

As described in Section 5, this call uses a separate computed base `r16`, not the later `B0` expression.

**Result:** it does not close or disprove the `F_03E0DD` V/B identity on the relevant path.

---

## 8. Explicit `V = B0` assignment search

No decoded direct store has the form:

```text
[B0+0x1414C] = B0
```

or an equivalent generic copy with a statically resolved source of `B0`.

The only direct field store is:

```text
[B0+0x1414C] = V_old
```

Therefore:

```text
Explicit V = B0 assignment:                 NOT FOUND
V == B0 by self-store:                     NO
V == B0 by initialization constructor:    NOT FOUND
V == B0 on any relevant path:              UNPROVEN
```

`NOT FOUND` is a bounded static result, not a universal claim about dynamically generated or external code.

---

## 9. A/B and descriptor alias closure

### 9.1 `F_03E0DD` V-based construction

At the table construction site:

```text
0x3E325  ori   r23,r14,0x4EE8
0x3E329  add   r23,r11
0x3E32B  addi  r24,r23,0x800
0x3E32F  ori   r4,r14,0x4E40
0x3E333  ori   r14,r14,0x4CCC
0x3E337  sw    0x4E40(r10),r23
0x3E33B  sw    0x4E44(r10),r24
0x3E33F  add   r4,r11
0x3E341  add   r3,r11,r14
0x3E344  jal   0x45DE0
```

With `r11 = V` after `0x3DF03` and `r10 = B0+0x10000`:

```text
D_V      = V + 0x14CCC
T_V      = V + 0x14E40
A_V      = V + 0x14EE8
B_V      = V + 0x156E8

[B0+0x14E40] = A_V
[B0+0x14E44] = B_V
```

The two stores are B0-based, but the values written into them are V-based.

### 9.2 `F_0411A6` B0-based construction

`F_0411A6` receives `r3 = B0` on the direct `F_03E961` path and constructs:

```text
D_B = B0 + 0x14CCC
T_B = B0 + 0x14E40
A_B = B0 + 0x14EE8
B_B = B0 + 0x156E8
```

### 9.3 Same-offset alias matrix

| Address expression | Meaning in the F03E0DD path | Meaning in the F0411A6 path | Alias condition |
|---|---|---|---|
| `B0+0x14CCC` | outer B0 descriptor candidate | `D_B` | equals `V+0x14CCC` iff `V=B0` |
| `B0+0x14E40` | table slot receiving `A_V` | `T_B` | equals `V+0x14E40` iff `V=B0` |
| `B0+0x14E44` | table slot receiving `B_V` | second `T_B` word | equals `V+0x14E44` iff `V=B0` |
| `B0+0x14EE8` | — | `A_B` | equals `V+0x14EE8` iff `V=B0` |
| `B0+0x156E8` | — | `B_B` | equals `V+0x156E8` iff `V=B0` |
| `D+0x174` | `V+0x14E40` in `F_03E0DD` | `B0+0x14E40` in `F_0411A6` | same base condition |
| `D+0x178` | `V+0x14E44` in `F_03E0DD` | `B0+0x14E44` in `F_0411A6` | same base condition |
| `D+0xF8` | `D_V+0xF8 = A_V` after `F_45DE0` | not the same descriptor unless bases alias | `V=B0` required |
| `D+0xFC` | `D_V+0xFC = B_V` after `F_45DE0` | not the same descriptor unless bases alias | `V=B0` required |

The pointee accesses `A+4*i`, `B+4*i`, and the corresponding descriptor fields inherit these same base conditions.

### 9.4 Conditional F8/FC closure

`F_45DE0` receives `D_V` and `T_V`. It installs:

```text
D_V+0xF8 = T_V[0]
D_V+0xFC = T_V[4]
```

`F_4618E` later reloads those fields and transforms the pointed-to buffers.

Therefore:

```text
V == B0:
  F_03E0DD, F_45DE0, F_4618E, and F_0411A6
  can be described as one B0/V descriptor-buffer instance.

V != B0:
  F_03E0DD installs into D_V/T_V and F_4618E consumes D_V fields.
  F_0411A6 separately constructs D_B from B0.
  The two chains must not be merged.
```

The V-based buffers represent a separate object/alias candidate selected by the pointer at `[B0+0x1414C]`. The static corpus does not identify its allocation, lifetime, or physical role. It is not safe to call them “B0’s A/B buffers” without the identity proof.

---

## 10. TX dump-ring revisit after alias closure

The TX monitor was revisited only after the B/V alias analysis.

### 10.1 Static function

`libutopia.so` exports:

```text
MDrv_AUDIO_Dump_SpdifNpcm_Monitor @ 0x369408
```

The ARM function calls:

```text
HAL_AUDIO_GetDspMadBaseAddr(1)
HAL_MAD2_Read_DSP_sram(0x1114, 1)
MsOS_MPool_PA2KSEG1(...)
MDrv_AUDIO_FileWrite(...)
```

The base and span are constructed as:

```text
r4 = HAL_AUDIO_GetDspMadBaseAddr(1)
r2 = r4 + 0x72000
r8 = r2 + 0x600000 = r4 + 0x672000
```

The file-write lengths are bounded by:

```text
0x66000
```

including the wrap/remaining case:

```text
0x3694E4  rsb r2,r5,#0x66000
```

### 10.2 Relation to B0/V

The monitor receives no A/B pointer, D pointer, V pointer, or B0 pointer as an argument. It uses:

- a DSP MAD base returned by a separate helper;
- a global file/control pointer;
- status bytes at offsets `0x55C` and `0x55D`;
- fixed ring base and span arithmetic.

No direct static edge connects `r4` to `V` or `B0`.

If, hypothetically, `V` or `B0` equaled the returned DSP MAD base, their derived A/B addresses (`+0x14EE8` and `+0x156E8`) would still be far below the monitor’s `+0x672000` ring base. That arithmetic is only a conditional non-overlap observation; it is not a proof of the runtime bases.

### 10.3 Ring verdict

```text
TX monitor ring base/span:                  VERIFIED
A/B or V/B pointer reaches monitor:        NOT FOUND
TX ring is the F8/FC descriptor:            DISPROVEN as a static identity claim
TX ring is a physical SPDIF sink:            UNPROVEN
TX ring relation after V/B closure:         SEPARATE/UNPROVEN
```

The monitor remains a software-labelled dump surface. The V/B uncertainty does not turn it into a consumer of the DEC F8/FC buffers.

---

## 11. Evidence ledger

| ID | Claim | Primary evidence | Status |
|---|---|---|---|
| V1 | `V = *(B0+0x1414C)` | `dec32_clean.txt:78362` plus `0x3E0E6/0x3E0EE` | `VERIFIED` |
| V2 | Only direct B0-field store is self-round-trip | `dec32_clean.txt:78362,78385` | `VERIFIED` |
| V3 | No direct store assigns B0 to the field | full `0x414C` store search | `NOT FOUND` |
| V4 | `0x3DF03` uses V as an object pointer and writes it back | `dec32_clean.txt:78362-78385` | `VERIFIED` |
| V5 | `0x37ACE` is a separate-base helper call | `dec32_clean.txt:69678-69734`, `69837-69843` | `VERIFIED` |
| V6 | Absolute `0x3BEB4` store is a numeric-alias candidate | `dec32_clean.txt:353712-353714` | `VERIFIED` candidate; link `UNPROVEN` |
| V7 | No relevant path has a static V=B0 proof | path matrix Sections 7–8 | `UNPROVEN` |
| V8 | `F_03E0DD` installs V-based D/T/A/B | `dec32_clean.txt:78680-78689` | `VERIFIED` |
| V9 | `F_0411A6` constructs B0-based D | `dec32_clean.txt:82507-82520` | `VERIFIED` |
| V10 | Same descriptor requires V=B0 | Sections 9.1–9.4 | `VERIFIED` conditional relation |
| V11 | TX ring base is DSP MAD base + `0x672000`, span `0x66000` | Capstone disassembly of `libutopia.so` | `VERIFIED` |
| V12 | TX monitor receives A/B/V/B0 pointer | function arguments and calls | `NOT FOUND` |
| V13 | Physical SPDIF relation of TX ring | no static edge | `UNPROVEN` |

---

## 12. Final static conclusion

The V/B question is closed to the following honest boundary:

1. `V` is read from `[B0+0x1414C]`.
2. The only direct B0-field write preserves the value already read.
3. No relevant static path explicitly sets `V = B0`.
4. No relevant static path disproves `V = B0`; the field’s upstream constructor is opaque.
5. `F_03E0DD` uses V-based descriptor/table/buffer addresses.
6. `F_0411A6` uses B0-based descriptor addresses.
7. The two are the same instance only under the unproved condition `V == B0`.
8. The TX dump monitor is a separate fixed-offset DSP-MAD dump surface and does not close that identity.

Accordingly, the prior unconditional F8/FC same-instance narrative is **not promoted**. The only defensible closure is:

```text
conditional closure: V == B0
unresolved branch:   V-based descriptor/buffers versus B0-based descriptor
```

No runtime action is required or permitted for this report. Any further proof would need a static constructor/loader trace that supplies the missing value of `[B0+0x1414C]`, or a separately authorized runtime observation outside this audit.

---

## 13. Preservation statement

- This is a new report.
- No previous report, raw log, binary, Ghidra project, or historical artifact was modified.
- No runtime, ADB, device connection, playback, memory read, breakpoint, patch, firmware write, flash, or reboot was performed.
- TCL/T615 evidence was not used.
