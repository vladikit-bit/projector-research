# FORENSIC DOWNSTREAM CLOSURE — F8/FC

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Baseline:** `C:\firmware_temp\forensic_space_bunny_full_corpus_audit_20260923.md`  
**New report only:** no existing artifact was edited, renamed, deleted, or overwritten  
**Runtime:** none; no ADB, device, memory reads, breakpoints, settings, reboot, or patch actions

## Executive Summary

The F8/FC investigation now closes several previously open static questions:

1. The installer and the `F_4618E` consumer use the **same descriptor instance** in the direct `F_3E961` call chain.
2. The exact descriptor layout on that path is:

```text
R       = root object
W       = R + 0xBA0
D       = W + 0x14CCC
T       = D + 0x174
A       = T[0] = W + 0x14EE8
B       = T[4] = W + 0x156E8
D+0x174 = A
D+0x178 = B
D+0xF8  = A
D+0xFC  = B
```

3. The exhaustive typed-reader result is:

```text
F8 descriptor-field loads: 8
FC descriptor-field loads: 8
F8/FC field stores:         1 each
```

All eight loads of each field are inside `F_45E52` or `F_4618E`. No later typed field read or pointer propagation is present after `F_4618E` returns.

4. A and B are passed as destinations only to:

- the fill helper `0x130D25`; and
- the bit/extraction helper `0x64369`.

The remaining operations are direct A/B memory reads and writes inside `F_45E52` and `F_4618E`. No caller consumes a returned A/B value as a new pointer, no later pointer is stored into a new structure, and no queue/DMA/TX object is populated with the A/B address.

5. The first concrete downstream object is therefore **not a queue or DMA descriptor**. It is the pair of memory buffers `A` and `B` themselves. The first unresolved boundary is after `F_4618E` returns:

```text
early unprepared return:            0x461C2  bt.jr r9
prepared-path final write branch:   0x462F1 -> 0x462F4/0x462F7 -> 0x46293/0x46299
prepared-path normal return:        0x462FE  bt.jr r9
caller status branch:               0x4128D  bg.bnei r12,1,...
```

At `0x4128D`, the branch consumes only `r12`; `r10=D` and the pre-call `r11=W+0x10000` are visible at the call site, but `F_4618E` reuses `r11` as a temporary, so post-call `r11` liveness is unproven. No subsequent typed reload of D+0xF8, D+0xFC, A, or B is proven. The static data-flow stops there.

6. The `+0xF8`/`+0xFC` accesses in the later `F_29867` region use a separate base candidate. Its identity relative to D is **UNPROVEN/BLOCKED**, not a proven different object and not a proven alias. Its `0x13602`/`0x13B08` calls are structural ring/cache candidates, but no A/B pointer identity is established.

7. No exact DEC-to-SND edge from A/B to SND `+0x250C` was found. Identical Pa/Pb constants are not an execution-path proof.

8. The standard DEC image contains near/exact counterparts of the F8/FC machinery. The standard SND image does not contain the MS12V22 `+0x250C` function or an exact shared helper edge.

9. The secondary SE-IDMA target remains bounded: ARM stores are now decoded down to mapped MIO byte stores, but no static producer for the DSP-side `0xB000001E` read is identified. The first opaque boundary is the hardware/SE sampling side of the MIO store, not an inferred software bridge.

**Primary answer:** F8/FC do not have a statically provable downstream consumer after `F_4618E`; they are populated local memory buffers whose pointer provenance terminates at the return/status boundary.

## Scope and Preservation Rules

### Scope

This report covers only the requested downstream continuation:

```text
F_45DE0
  -> D+0xF8 / D+0xFC
  -> A/B memory contents
  -> all statically reachable consumers
  -> pointer propagation
  -> queue/ring/DMA/TX candidates
  -> first opaque boundary
```

The secondary static SE-IDMA → DSP target is included as a bounded comparison, not as a reopening of the general Phase D/D.2 audit.

### Preservation rules honored

- No existing artifact was changed.
- No device or runtime action was performed.
- No patch, proposed patch, bypass, flash, or remediation is included.
- No old report was used to override raw bytes or SLEIGH semantics.
- The previous Space Bunny report is treated as the evidence baseline, not as an authority over contradictory primary evidence.
- Already-closed narratives were not reopened without new primary evidence.

### Evidence hierarchy

1. Raw DEC/SND/ARM bytes.
2. `aeon_validate/dec_work/dec32_clean.txt` and other authoritative listings.
3. AEON SLEIGH load/store and branch semantics.
4. Direct def-use data flow.
5. Historical captures only where needed for execution-surface context.
6. Prior reports and agent results.

### Status vocabulary

- `VERIFIED`: direct raw/instruction/data-flow proof.
- `STRONGLY SUPPORTED`: multiple consistent primary observations.
- `LIKELY`: plausible but not required to make the final chain claim.
- `UNPROVEN`: semantic or causal claim without a complete edge.
- `OPAQUE`: the available static corpus cannot cross the boundary.
- `BLOCKED`: a candidate exists, but a necessary identity/provenance edge is missing.
- `NOT FOUND`: bounded negative search result, not universal absence.
- `DISPROVEN`: directly contradicted by primary evidence.

### Image identities

| Image | Size | MD5 | SHA-256 |
|---|---:|---|---|
| `aeon_validate/dec_work/dec_full.bin` | 1,982,492 | `4b7e9509b4358fd3a130bd4d3b9cbe0a` | `530bfa684bc363e8513951b731cd42ccae97c96accef3c1d51eb431b61a53f5b` |
| `aeon_validate/r27_work/snd_full.bin` | 1,839,920 | `eb879cdc07f510722f19db6d18d77d3c` | `1da795fa47aec5aec2aa4cdf78d52028d62abe911f6993dac664206b9210c22d` |
| `spdif_audio_investigation/r2img/mst_codec_r2.bin` | 2,736,372 | `e26ce887c6d3573d6fd5c243881cb448` | `f733e96921e330495a537701ecb771f307da0b1743ebc90b31106bf8fbb913ef` |
| `spdif_audio_investigation/r2img/mst_snd_r2.bin` | 1,567,048 | `42c1cb7991e88c8a97bf926bfb47884e` | `f5bdb7c2ef93caa406ca8a9c6c38ff9aeb8868275828c4004b9011f5b9d4f2f5` |

The current static execution-surface proof is the embedded MS12V22 image selected by the loader when its branch conditions are satisfied. The current runtime selector value is not inferred here.

## F8 Provenance

### Function boundaries

| Function | Address range used here | Role |
|---|---:|---|
| `F_45DE0` | `0x45DE0–0x45E52` | Installs A/B pointers into D+0xF8/+0xFC |
| `F_45E52` | `0x45E52–0x45FD1` | Reads D+0xF8/+0xFC and prepares A/B contents |
| `F_45FD1` | `0x45FD1–0x4618E` | Prepares descriptor fields; indirect dispatch later |
| `F_4618E` | `0x4618E–0x46300` | Reads D+0xF8/+0xFC and transforms A/B contents |
| `F_411A6` | `0x411A6–0x4139A` | Builds D and calls the F8/FC consumers |
| `F_3E0DD` | entry `0x3E0DD` | Builds D/T and installs A/B pointer table |
| `F_3E961` | entry `0x3E961` | Calls F_411A6 and F_3E0DD on state-machine paths |

Evidence: `aeon_validate/dec_work/dec32_clean.txt:88860-89037`, `:89181-89306`, `:82438-82522`, `:78680-78689`, `:79190-79645`.

### D identity and T origin

The direct caller chain is now closed:

```text
F_375BE(root, W=R+0xBA0)
  -> F_3E961(root, W)
       -> F_411A6(W)
       -> F_3E0DD(root, W)
```

Evidence:

- `F_375BE` forms `W = R+0xBA0`: `dec32_clean.txt:69420-69472`.
- `F_3E961` preserves `r11=W` and `r14=root`: `dec32_clean.txt:79190-79214`.
- `F_411A6(W)` constructs `D=W+0x14CCC`: `dec32_clean.txt:82461-82520`.
- `F_3E0DD(root,W)` constructs the same `D=W+0x14CCC`: `dec32_clean.txt:78680-78689`.

At `F_3E0DD`, the relevant `r14=0x10000` value is established at `0x3E29F`; the arithmetic then becomes:

```text
r23 = W + 0x14EE8       = A
r24 = W + 0x156E8       = B
D+0x174 = A
D+0x178 = B
T = D+0x174
F_45DE0(D,T)
```

The exact instructions are:

```text
03e325  ori   r23,r14,0x4ee8
03e329  add   r23,r11
03e32b  addi  r24,r23,0x800
03e32f  ori   r4,r14,0x4e40
03e333  ori   r14,r14,0x4ccc
03e337  sw_0  0x4e40(r10),r23
03e33b  sw_0  0x4e44(r10),r24
03e33f  add   r4,r11
03e341  add   r3,r11,r14
03e344  jal   0x45de0
```

Raw listing: `aeon_validate/dec_work/dec32_clean.txt:78642`, `:78680-78689`.

The pointer table is therefore not an unknown object on this path. It is `D+0x174`, and its two words are A and B.

### F8 field store

`F_45DE0` loads `T[0]` and stores it into `D+0xF8`:

```text
045e08  lwz   r3,0x0(r13)     ; r3 = T[0] = A
045e17  sw_0  0xf8(r10),r3     ; D+0xF8 = A
```

Evidence: `dec32_clean.txt:88875`, `:88880`.

The only other F8 field store found in the typed descriptor chain is absent: the F8 slot is installed once by `F_45DE0`; later operations store A/B contents, not the A pointer itself.

### F8 pointer arithmetic

After installation, the F8 pointer is reloaded only in the two consumer functions:

- `r24 = *(D+0xF8)` at `0x45EFB`;
- `r29 = *(D+0xF8)` at `0x45F53`;
- `r3 = *(D+0xF8)` at `0x461C7`;
- `r5 = *(D+0xF8)` at `0x46201`, `0x46222`, `0x4626A`, `0x462A0`, and `0x462DE`.

The resulting A-derived addresses are:

```text
A + 0
A + 4*i
A + 4*(i+1)
A + 4*(i+2)
```

After the one-time installation at `D+0xF8`/`D+0xFC`, no later A/B-derived pointer is stored into a new descriptor, queue, DMA, or SND structure. The installation stores themselves are the two field stores listed above.

## FC Provenance

### FC origin

`F_45DE0` loads `T[4]` and stores it into `D+0xFC`:

```text
045e25  lwz   r3,0x4(r13)     ; r3 = T[4] = B
045e2f  sw_0  0xfc(r10),r3     ; D+0xFC = B
```

Evidence: `aeon_validate/dec_work/dec32_clean.txt:88884`, `:88887`.

### FC field reads

The exhaustive FC field reads are:

```text
045eff  F_45E52: r23 = *(D+0xFC) = B
045f57  F_45E52: r30 = *(D+0xFC) = B
0461d6  F_4618E: r3  = *(D+0xFC) = B
046214  F_4618E: r5  = *(D+0xFC) = B
046226  F_4618E: r23 = *(D+0xFC) = B
046271  F_4618E: r23 = *(D+0xFC) = B
0462a4  F_4618E: r23 = *(D+0xFC) = B
0462e6  F_4618E: r23 = *(D+0xFC) = B
```

The FC pointer is used for the same kinds of local buffer operations as F8, but no FC pointer is propagated to a later function.

### FC pointer arithmetic

Observed B-derived accesses include:

```text
F_45E52 header path: B + 4*i, B + 4*(i+1), B + 4*(i+2)
F_45E52 bulk path:   B + 4*i and B + 4*(i+1)
F_4618E sync path:   B + r16, where the exact origin of r16 is unresolved at 0x461E9
```

All are local to `F_45E52` or `F_4618E`. The FC field itself is not passed to `F_45D07`, `F_13602`, `F_13B08`, or a SND function on the proven D chain.

## All F8/FC Consumers

### Descriptor-field readers versus pointee reads

The following are **descriptor-field reads**:

```text
*(D+0xF8) or *(D+0xFC)
```

They are not themselves reads of A/B contents.

The following are **pointee reads/writes**:

```text
*(A + offset)
*(B + offset)
```

They occur only after a field read has loaded the pointer.

### F8 reader table

| Site | Function | Instruction | Source → destination | Direct use | Propagation |
|---:|---|---|---|---|---|
| `0x45EFB` | `F_45E52` | `lwz r24,0xF8(r11)` | `D+F8 → r24=A` | Header staging, `A+4*i` writes | None |
| `0x45F53` | `F_45E52` | `lwz r29,0xF8(r11)` | `D+F8 → r29=A` | Bulk staging, `A+4*i` writes | None |
| `0x461C7` | `F_4618E` | `lwz r3,0xF8(r3)` | `D+F8 → r3=A` | `0x130D25` fill, `N*4` | Fill helper only |
| `0x46201` | `F_4618E` | `lwz r5,0xF8(r10)` | `D+F8 → r5=A` | `A+4*i` to `0x64369` | Bit reader writes A |
| `0x46222` | `F_4618E` | `lwz r5,0xF8(r10)` | `D+F8 → r5=A` | Direct A word RMW | None |
| `0x4626A` | `F_4618E` | `lwz r5,0xF8(r10)` | `D+F8 → r5=A` | `A+r16` sync/header write | None |
| `0x462A0` | `F_4618E` | `lwz r5,0xF8(r10)` | `D+F8 → r5=A` | A header-field copy | None |
| `0x462DE` | `F_4618E` | `lwz r5,0xF8(r10)` | `D+F8 → r5=A` | `0x1FFF` branch | None |

Evidence: `dec32_clean.txt:88964`, `:88993`, `:89206`, `:89225`, `:89237`, `:89261`, `:89278`, `:89297`.

### FC reader table

| Site | Function | Instruction | Source → destination | Direct use | Propagation |
|---:|---|---|---|---|---|
| `0x45EFF` | `F_45E52` | `lwz r23,0xFC(r11)` | `D+FC → r23=B` | Header staging | None |
| `0x45F57` | `F_45E52` | `lwz r30,0xFC(r11)` | `D+FC → r30=B` | Bulk staging | None |
| `0x461D6` | `F_4618E` | `lwz r3,0xFC(r10)` | `D+FC → r3=B` | `0x130D25` fill, `N*4` | Fill helper only |
| `0x46214` | `F_4618E` | `lwz r5,0xFC(r10)` | `D+FC → r5=B` | `B+4*i` to `0x64369` | Bit reader writes B |
| `0x46226` | `F_4618E` | `lwz r23,0xFC(r10)` | `D+FC → r23=B` | Direct B word RMW | None |
| `0x46271` | `F_4618E` | `lwz r23,0xFC(r10)` | `D+FC → r23=B` | `B+r16` sync/header write | None |
| `0x462A4` | `F_4618E` | `lwz r23,0xFC(r10)` | `D+FC → r23=B` | B header-field copy | None |
| `0x462E6` | `F_4618E` | `lwz r23,0xFC(r10)` | `D+FC → r23=B` | `0xE800` branch | None |

Evidence: `dec32_clean.txt:88965`, `:88994`, `:89211`, `:89232`, `:89238`, `:89263`, `:89279`, `:89299`.

### Pointer-derived helper calls

| Helper | A/B argument | Call site | Helper behavior | Pointer propagation |
|---|---|---|---|---|
| `0x130D25` | A/B destination | `0x45E21`, `0x45E39`, `0x461D2`, `0x461DE` | Fill/copy bytes through a local cursor; `r3` is restored from the helper's local `r25` cursor/destination state | Callers do not consume it as a new pointer |
| `0x64369` | `A+4*i` or `B+4*i` | `0x46210`, `0x4621E` | Writes extracted word to supplied address | No propagated pointer |
| `0x45D07` | D, not A/B | `0x45FC9` | Header metadata preparation | No A/B pointer |

The fill helper's direct store sites are listed in `dec32_clean.txt:410711-410789`. Its return setup is at `dec32_clean.txt:410766-410769`; `r25` is a local destination/cursor that can advance on tail paths, and callers do not consume the returned value as a new pointer. The bit reader's direct write sites are `0x643B7` and `0x643D5`, `dec32_clean.txt:130312-130351`.

### Exhaustive-search qualification

The complete decoded listing contains many unrelated `+0xF8`/`+0xFC` fields. The count above is exhaustive for the typed D chain because all eight F8 and eight FC loads are tied to D by the caller/field def-use chain. Computed addresses and untyped structure aliases remain a bounded negative result, not a universal absence claim.

The raw listing search found no separate typed D+0xF8/FC reader after `0x462FE`. The direct `D` pointer is local to `F_411A6`; it is not returned or stored by the caller.

## Pointer/Data-flow Chains

### F8 chain

```text
R
  -> W = R+0xBA0
  -> D = W+0x14CCC
  -> T = D+0x174
  -> A = T[0] = W+0x14EE8
  -> D+0xF8 = A                         VERIFIED
  -> F_45E52 loads D+0xF8                VERIFIED
       -> A+4*i header/bulk writes        VERIFIED
  -> F_4618E loads D+0xF8                VERIFIED
       -> 0x130D25(A,N*4)                VERIFIED
       -> 0x64369(A+4*i)                 VERIFIED
       -> A word/header/sync writes       VERIFIED
  -> F_4618E return (early or prepared)  VERIFIED
  -> status branch 0x4128D                VERIFIED
  -> later A consumer                    NOT FOUND / OPAQUE
```

### FC chain

```text
R
  -> W = R+0xBA0
  -> D = W+0x14CCC
  -> T = D+0x174
  -> B = T[4] = W+0x156E8
  -> D+0xFC = B                         VERIFIED
  -> F_45E52 loads D+0xFC                VERIFIED
       -> B+4*i/B+4*(i+1)/B+4*(i+2) staging VERIFIED
  -> F_4618E loads D+0xFC                VERIFIED
       -> 0x130D25(B,N*4)                VERIFIED
       -> 0x64369(B+4*i)                 VERIFIED
       -> B word/header/sync writes       VERIFIED
  -> F_4618E return (early or prepared)  VERIFIED
  -> status branch 0x4128D                VERIFIED
  -> later B consumer                    NOT FOUND / OPAQUE
```

### Same-descriptor correction

The previous uncertainty about whether `F_45DE0` and `F_411A6` used different descriptor instances is resolved for the direct `F_375BE → F_3E961` chain:

```text
D_installer = W+0x14CCC
D_consumer  = W+0x14CCC
```

This proves object identity. It does not prove that every state-machine invocation calls the installer immediately before the consumer; the two calls occur on separate state-machine branches and can be revisited.

## Post-F_4618E Transforms

### Header staging before F_4618E

`F_45E52` uses A/B as output buffers for descriptor header words:

- A receives values derived from `D+0x16C` and `D+0x170`.
- B receives values derived from `D+0x16E` and `D+0x172`.
- A/B are also filled from a separate source at `r27 = r12+0x6E2C` in the bulk path.

Evidence: `dec32_clean.txt:88964-89006`.

This is a data transformation into A/B. It is not a pointer propagation into another structure.

### Initial fill in F_4618E

When `D+0x28 == 1`:

```text
r14 = D+0x38
r11 = r14 << 2
0x461D2: fill(A, 0, r11)
0x461DE: fill(B, 0, r11)
```

Evidence: `dec32_clean.txt:89205-89214`.

The helper is a byte-fill implementation, not an allocator and not a queue submit.

### Loop transform

For each `i < N`:

```text
0x46201/0x46214  load A/B
0x46205/0x46219  compute 4*i
0x46210/0x4621E  call 0x64369(A+4*i or B+4*i)
0x4622D/0x4623B  reload word
0x46238/0x46246  write sign/width-adjusted word back
```

The loop counter is `N = D+0x38`; no external index or DMA descriptor is introduced.

### Sync/header branches

- If `D+0x28 != 1`, `F_4618E` takes the early path, stores zero to `D+0xF0`, and returns at `0x461C2` without the prepared A/B loop.
- If `D+0x16C == 0` and `D+0x2C==1`, the path can reach the `0x7FFE/0x8001` writes at `0x4627B/0x46285` when `D+0x30==0`.
- If `D+0x16C != 0`, `0x461F7` branches to the header-copy path at `0x462A0`; this path writes through `0x462D1/0x462D7` and rejoins the loop at `0x461FD`.
- If the prepared path reaches `D+0x30 != 0`, it writes `0x1FFF/0xE800` at `0x462EA/0x462F1`, shifts the cursor at `0x462F4`, and rejoins the common round-trip at `0x462F7 -> 0x4628C`.

Evidence: `dec32_clean.txt:89190-89214`, `:89215-89304`.

### Last proven writes and returns

The last write is path-dependent:

```text
early unprepared return:       0x461C2  bt.jr r9
prepared/header path:         0x462D1/0x462D7  A/B header writes, then loop rejoins
D+0x30 != 0 path:             0x462F1  B[...] = 0xE800
                               0x462F4/0x462F7 -> 0x4628C
                               0x46293/0x46299  common A/B round-trip writes
prepared normal return:        0x462FE  bt.jr r9
```

Thus `0x462F1` is not universally the last write, and `0x462FE` is not the only return. Both returns reach the same caller status boundary.

Evidence: `dec32_clean.txt:89190-89204`, `:89278-89306`.

No direct A/B write occurs in `F_411A6` after `0x41289` calls `F_4618E`; the next relevant instruction is the status branch at `0x4128D`.

### Unresolved local operands

Two narrow decoder issues remain inside the closed direct flow:

- `0x45F7B` is decoded as an unresolved opcode affecting later register state in the bulk loop.
- `0x461E9` is an unresolved short instruction before the `r8`/`r16` sync-offset use.

Neither creates a new A/B pointer or downstream object. They are local data-value uncertainties, not evidence of a hidden consumer.

## Queue / Ring / DMA Evidence

### No A/B queue submission

No instruction in the proven A/B chain passes A or B to a queue/DMA/TX submit helper. The only helper calls receiving A/B are `0x130D25` and `0x64369`, both local buffer operations.

### F_29867 candidate has a separate, unproven base

The later `F_29867` region has same-numeric `+0xF8`/`+0xFC` accesses. Its base is not statically tied to D:

```text
0x29A6D  r25 = *(base298+0xFC)
0x29D0E  r7  = *(base298+0xF8)
0x2A244  r4  = *(base298+0xFC)
0x2A29E  r4  = *(base298+0xFC)
0x2A4CB  r4  = *(base298+0xFC)
0x2A4E9  r3  = *(base298+0xFC)
0x2A50B  r4  = *(base298+0xFC)
0x2A568  r4  = *(base298+0xFC)
```

Evidence: `dec32_clean.txt:50903-51900`.

For the D chain, the corresponding absolute expressions are:

```text
D       = R+0xBA0+0x14CCC = R+0x1586C
D+0xF8  = R+0x15964
D+0xFC  = R+0x15968
```

The `F_29867` base is a separate candidate object, but the available call/data flow does not prove either equality or inequality with D. Its F8/FC fields therefore cannot be promoted to downstream consumers. Status: `UNPROVEN/BLOCKED`.

### Structural ring/cache helpers

`F_13602` and `F_13B08` contain structural ring/cache-like behavior:

- state fields at `r10+0x1D4` / `r10+0x1E4`;
- counter updates at `r10+0x9C` / nearby fields;
- length/stride arithmetic;
- calls to `0x132001`, `0xDFB1`, `0x13041`, or `0x1318B`;
- conditional branches and index calculations.

Evidence: `dec32_clean.txt:22169-22321`, `:22577-22739`.

However, the `F_29867` call sites pass the `base298+0xFC` value as scalar arguments to these helpers. The helpers do not establish that the value is A or B, and no call receives the D-derived A/B pointer. Status: `STRUCTURAL CANDIDATE`, edge `BLOCKED/UNPROVEN`.

### No DMA/FIFO conclusion

No A/B-derived value is shown to reach:

- a DMA address/length pair;
- a paired producer/consumer ring index;
- an MMIO write with a pointer payload;
- a submit/kick/trigger sequence;
- the SND `+0x250C` object.

The absence is bounded to the decoded corpus and does not exclude external hardware or dynamically computed code.

## DEC → SND Relations

### No exact cross-image edge

The MS12V22 SND IEC preparation function writes:

```text
SND+0x250C  Pa
SND+0x250E  Pb
SND+0x2510  Pc
SND+0x2512  Pd
SND+0x2514  payload
```

Evidence: `aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm:17739-17784`.

No A/B pointer, D pointer, or D+0xF8/FC value is passed to this function. The exact key sequence is absent from both DEC images.

The standard SND image has isolated Pa/Pb instruction pairs but no exact `+0x250C` function or field sequence. The SND fill helper has an exact standard-SND counterpart, but no exact counterpart in either DEC image; the DEC fill helper is likewise absent from SND.

Therefore:

```text
DEC A/B -> shared helper -> SND +0x250C
```

is `NOT EVIDENCED`, not “proved absent” at hardware level.

### Identical constants are insufficient

The following facts are verified independently:

- DEC contains Pa=`0xF872`, Pb=`0x4E1F`.
- SND contains Pa=`0xF872`, Pb=`0x4E1F`.
- Standard SND contains isolated Pa/Pb pairs.

These facts do not establish a shared structure, helper, pointer, or call edge.

## Indirect Call Targets

### F_45FD1 dispatch

`F_45FD1` contains an indirect dispatch:

```text
0x46060  load dispatch table base
0x46063  jr r23
```

Evidence: `aeon_validate/dec_work/dec32_clean.txt:89086-89092`.

The table entries are local function targets in the decoded corpus. This dispatch is descriptor-state control, not an A/B pointer call. No F8/FC value is used as the target.

### F_4618E path

The A/B path uses direct calls to:

- `0x130D25`;
- `0x64369`;
- descriptor/bit helpers in the surrounding function.

No indirect target receives A or B. The return `jr r9` is a normal AEON return register, not a function-pointer edge.

### F_3E961 state machine

`F_3E961` uses saved return continuations and `jr r9` for state-machine control. Its direct call to `F_411A6(W)` and `F_3E0DD(root,W)` is resolved. No A/B pointer is carried across the return.

### F_29867 helpers

The `F_13602`/`F_13B08` calls from `F_29867` are direct `jal` calls. Their ring/cache behavior does not resolve the A/B identity question because the input base is the separate `base298` candidate, with no proven identity to D.

### Indirect-call verdict

| Candidate | Target status | A/B relevance |
|---|---|---|
| `F_45FD1` `jr r23` | Local dispatch targets resolved | None |
| `F_4618E` calls | Direct targets resolved | Local A/B fill/transform only |
| `F_3E961` continuation | State-machine return, not data pointer | None |
| `F_13602/13B08` | Direct targets; semantic role structural | Separate F29867 base candidate; identity to D unproven |
| External/dynamic A/B target | `OPAQUE` | No static evidence |

## SE-IDMA → DSP Static Bridge

### ARM software side is now deeper than the prior baseline

The stock ARM sequence remains:

```text
MDrv_AUDIO_ApplyHashkey
  -> SET_IPAUTH_GROUP @0x424BD8
  -> AbsWriteByte calls
```

Call evidence: `spdif_audio_investigation/tools/closure_utpa2k.txt:178-216`.

A fresh raw ARM decode of `patch_baseline/utpa2k_stock.ko` resolves `HAL_AUDIO_AbsWriteByte` at `0x441CDC` to a mapped byte-store path (store at `0x441E7C`):

```text
logical address - 0x100000
mapped offset = 2*(logical address - 0x100000) - (address & 1)
strb to [_gMIO_MapBase + offset]
```

The four normal `SET_IPAUTH_GROUP` addresses map to:

```text
0x112A82 -> _gMIO_MapBase + 0x25504
0x112A83 -> _gMIO_MapBase + 0x25505
0x112A84 -> _gMIO_MapBase + 0x25508
0x112A85 -> _gMIO_MapBase + 0x25509
```

The boot initialization table (`AudioInitTbl_0+0x678` through `+0x748` exclusive in `patch_baseline/utpa2k_stock.ko`) also contains data-driven records targeting `0x112A82..0x112A85`; the table executor passes each record address to `HAL_AUDIO_AbsWriteMaskByte` (`0x442430` executor, call at `0x4424D4`).

### Polling is not a bridge

`HAL_AUDSP_CheckSeIdmaReady` polls `0x112A80` through `AbsReadByte`, with timeout handling. This proves a status wait, not a producer edge to `0xB000001E`.

Other static co-writers exist:

- `MDrv_Write_DSP_sram` has another `0x112A82..85` writer.
- `HAL_Copy` has a bulk writer.
- `HAL_MAD2_SetDspIDMA` writes status `0x112A80`.

None is statically joined to the `SET_IPAUTH_GROUP` data transaction as a common object.

### DSP read side

The four DEC reads remain:

```text
0x21AAB/0x21B16/0x21EF6/0x21F59  lh 0xB000001E
0x21AC3/0x21B19/0x21F0E/0x21F5C  compare #5
```

Evidence: `aeon_validate/dec_work/dec32_clean.txt:40659-41034`.

The SND image has a separate 8-bit handshake using the same numeric address:

```text
0xC253  write 0xE3 to 0xB000002E
0xC259  read byte 0xB000001E
0xC260  repeat until byte == 0xF3
0xC26E  clear 0xB000002E
```

Evidence: `aeon_validate/r27_work/ghidra_dump_C000_E00.txt:606-644`.

This is a shared numeric read location, not a producer edge.

### First static bridge boundary

The earliest opaque edge is after the ARM MIO store:

```text
0x424C94/0x424CA0/0x424CAC/0x424CD4
  -> HAL_AUDIO_AbsWriteByte
  -> strb to mapped MIO
  == OPAQUE: SE/hardware sampling and IDMA protocol ==
  ... no identified receiver state ...
  == OPAQUE: producer of DSP 0xB000001E ==
  -> DEC/SND reads
```

No static code writer of `0xB000001E` was found in the bounded executing DEC/SND images or stock ARM `.text`. The correct status is `OPAQUE`, not “no writer exists.”

## Standard vs MS12V22

### DEC counterpart comparison

Raw byte comparison of the standard and MS12V22 DEC images gives:

| MS12V22 function | Standard counterpart | Match | Classification |
|---|---:|---:|---|
| `F_45D07` | `0x50AFE` | `217/217` | Exact |
| `F_45DE0` | `0x50BD7` | `102/114` | Near; call relocation differences |
| `F_45E52` | `0x50C49` | `380/383` | Near; one call relocation |
| `F_45FD1` | `0x50DC8` | `438/445` | Near |
| `F_4618E` | `0x50F85` | `1162/1168` over extended `[0x4618E,0x4661E)` | Near; call relocation differences |
| `F_411A6` | `0x4BF9D` | `500/500` | Exact |
| `F_3E961` | `0x49758` | `2798/2798` | Exact |
| `F_375BE` | `0x423B5` | `594/594` | Exact |
| `F_3E0DD` | `0x48ED4` | `678/693` | Near |
| `F_29867` region | `0x344AF` | `3291/3394` over the extended downstream region | Near; base identity remains unproven |

Raw comparison sources are `spdif_audio_investigation/r2img/mst_codec_r2.bin` and `spdif_audio_investigation/r2img/mst_snd_r2.bin`; MS12V22 sources are `aeon_validate/dec_work/dec_full.bin` and `aeon_validate/r27_work/snd_full.bin`. The standard DEC preserves the F8/FC field instructions exactly in the corresponding functions, including the F8/FC load/store encodings. The `F_29867` count is an extended-region comparison: the broader `[0x29867,0x2A5A9)` region maps to standard `[0x344AF,0x351F1]` with `3291/3394` matching bytes; the earlier `[0x29867,0x29E00)` subwindow is `1415/1433`. It is not a single 683-byte function comparison. The call-target differences do not create a shared SND edge.

### SND comparison

The standard SND image has no exact counterpart of the MS12V22 `+0x250C` function/access sequence. It has isolated Pa/Pb pairs only. The standard SND fill helper has a separate exact counterpart in standard SND, not in DEC.

### Execution-surface qualification

The embedded MS12V22 DEC/SND pair is the image selected by the stock loader branch when its conditions are met. The standard pair is a static comparison artifact here. No claim is made about the current runtime selector value because runtime inspection is prohibited.

The `/vendor/audio_bin` files are not used as execution evidence.

## First Opaque Boundary

### Exact boundary

The first concrete downstream object is the A/B memory itself:

```text
A = W+0x14EE8
B = W+0x156E8
```

The last proven data operations are path-dependent inside `F_4618E`:

```text
early unprepared return:       0x461C2  bt.jr r9
prepared/header path:         0x462D1/0x462D7  A/B header writes, then loop rejoins
D+0x30 != 0 path:             0x462F1  B[...] = 0xE800
                               0x462F4/0x462F7 -> 0x4628C
                               0x46293/0x46299  common A/B round-trip writes
prepared normal return:        0x462FE  bt.jr r9
```

The first caller instruction that loses A/B provenance is:

```text
0x4128D  bg.bnei r12,1,0x411C9
```

Immediately after the `F_4618E` call at `0x41289`, the branch consumes only `r12`; `r10=D` and the pre-call `r11=W+0x10000` are visible at the call site, but `F_4618E` reuses `r11` as a temporary, so post-call `r11` liveness is unproven. No later typed reload of D+0xF8, D+0xFC, A, or B is proven. The subsequent state-machine code uses other fields and does not carry A/B.

### Boundary classification

```text
A/B origin:                         VERIFIED
A/B contents and transforms:        VERIFIED
A/B pointer passed to local helpers: VERIFIED
A/B -> queue descriptor:             NOT FOUND
A/B -> DMA/FIFO/TX submission:       NOT FOUND
A/B -> SND +0x250C:                  NOT FOUND
First downstream object:             A/B memory buffers
First opaque boundary:               0x4128D after F_4618E return
```

The boundary is not evidence that hardware does not consume A/B. It is the first point at which the decoded software corpus loses the pointer and provides no direct propagation edge.

## Corrections to Previous Reports

| Prior statement | Correction/status |
|---|---|
| F8/FC installer instance was unresolved | `DISPROVEN` as an identity gap on the direct `F_375BE → F_3E961` chain; D identity is now `VERIFIED` |
| A/B could be passed to a later queue/helper | `NOT FOUND` in the typed chain; only fill and bit-reader helpers receive them |
| F8/FC may be a queue/DMA/TX object | `UNPROVEN`; no structural queue/DMA evidence follows A/B |
| `F_29867` `+F8/+FC` fields are downstream F8/FC | `UNPROVEN/BLOCKED`; the base is separate in the listing, but no equality/inequality identity proof closes the edge |
| Same Pa/Pb constants imply DEC→SND path | `DISPROVEN` as an execution-path inference |
| Standard and MS12V22 F8/FC logic are unrelated | `SUPERSEDED`; standard DEC has exact field instructions and near function counterparts |
| F8/FC content is proven compressed DTS | `UNPROVEN`; transforms and sync constants are verified, semantic payload identity is not |
| A/B pointer survives F_4618E | `DISPROVEN`; the proven local pointer is not reloaded after return |
| F8/FC are license fields | `UNPROVEN`; no license semantic edge is present, so the license naming is unsupported rather than disproved by this trace |
| F8/FC are allocated by F_45DE0 | `DISPROVEN`; F_45DE0 installs pointers and fills buffers |
| Physical SPDIF relation follows from header constants | `UNPROVEN`; no A/B-to-peripheral edge exists |

No old `0x97`, `r4=codec ID`, `+0x10=license`, `AE290=license`, `+0x2E6`, `0xDFB1=allocator`, `/vendor DEC`, or historical `Pc=0x000B` narrative is used as a new conclusion here.

## Evidence Ledger

| ID | Claim | Primary evidence | Status |
|---|---|---|---|
| F01 | F_45DE0 installs two pointers | `dec32_clean.txt:88860-88890` | `VERIFIED` |
| F02 | F8 source is T[0] | `dec32_clean.txt:88875-88880` | `VERIFIED` |
| F03 | FC source is T[4] | `dec32_clean.txt:88884-88887` | `VERIFIED` |
| F04 | T is D+0x174 | `dec32_clean.txt:78680-78689` | `VERIFIED` |
| F05 | A = W+0x14EE8 | `dec32_clean.txt:78680-78686` | `VERIFIED` |
| F06 | B = W+0x156E8 | `dec32_clean.txt:78680-78686` | `VERIFIED` |
| F07 | F411A6 and F3E0DD use same D | `dec32_clean.txt:69420-69472`, `:79190-79645`, `:78680-78689` | `VERIFIED` |
| F08 | Eight F8 field reads | `dec32_clean.txt:88964-89297` | `VERIFIED` |
| F09 | Eight FC field reads | `dec32_clean.txt:88965-89299` | `VERIFIED` |
| F10 | A/B source loads and fill-helper calls | `dec32_clean.txt:88875-88890`, `:89205-89214` | `VERIFIED` |
| F11 | A/B passed to bit reader | `dec32_clean.txt:89225-89236` | `VERIFIED` |
| F12 | Fill helper writes; return register reflects local cursor/destination state, and callers ignore it | `dec32_clean.txt:410711-410789` | `VERIFIED` |
| F13 | Bit reader writes, does not return pointer | `dec32_clean.txt:130312-130351` | `VERIFIED` |
| F14 | F4618E direct A/B transforms | `dec32_clean.txt:89237-89304` | `VERIFIED` |
| F15 | F4618E early return at 0x461C2 and prepared return at 0x462FE | `dec32_clean.txt:89190-89204`, `:89305-89306` | `VERIFIED` |
| F16 | Caller checks only r12 after return | `dec32_clean.txt:82518-82522` | `VERIFIED` |
| F17 | No later typed F8/FC field read | Full decoded def-use search | `NOT FOUND` |
| F18 | No A/B pointer propagation after return | Direct call/data-use trace | `NOT FOUND` |
| F19 | F29867 uses a separate base candidate; identity relative to D | `dec32_clean.txt:50903-51900` | `UNPROVEN/BLOCKED` |
| F20 | F13602/13B08 have ring-like structure | `dec32_clean.txt:22169-22739` | `VERIFIED` structure; edge `UNPROVEN` |
| F21 | F13602/13B08 receive A/B on D path | Direct argument trace | `DISPROVEN` |
| F22 | No DEC→SND +250C edge | `aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm:17739-17784`; raw cross-image comparison | `NOT FOUND` |
| F23 | Standard DEC preserves F8/FC field instructions | `spdif_audio_investigation/r2img/mst_codec_r2.bin`; raw counterpart comparison | `VERIFIED` |
| F24 | Standard SND lacks exact +250C function | `spdif_audio_investigation/r2img/mst_snd_r2.bin`; raw counterpart comparison | `VERIFIED` |
| F25 | ARM writes reach mapped MIO bytes | `spdif_audio_investigation/tools/closure_utpa2k.txt:178-216`; fresh raw decode of `patch_baseline/utpa2k_stock.ko` | `VERIFIED` software side |
| F26 | ARM MIO write → DSP 0xB000001E producer | Full bounded static scan | `OPAQUE` |
| F27 | Four DEC 16-bit gate reads | `dec32_clean.txt:40659-41034` | `VERIFIED` |
| F28 | SND 8-bit same-address handshake | `aeon_validate/r27_work/ghidra_dump_C000_E00.txt:606-644` | `VERIFIED`; relation `UNPROVEN` |

## Final Causal Graph

```text
R
 |
 v
W = R+0xBA0                                      VERIFIED
 |
 v
D = W+0x14CCC                                    VERIFIED
 |
 +--> T = D+0x174 = W+0x14E40                   VERIFIED
 |       |
 |       +--> A = T[0] = W+0x14EE8              VERIFIED
 |       +--> B = T[4] = W+0x156E8              VERIFIED
 |
 +--> D+0x174=A, D+0x178=B                      VERIFIED
 |
 +--> F_45DE0
 |       |
 |       +--> D+0xF8=A                          VERIFIED
 |       +--> D+0xFC=B                          VERIFIED
 |       +--> fill A/B                          VERIFIED
 |
 +--> F_45E52
 |       |
 |       +--> header/bulk transforms in A/B     VERIFIED
 |
  +--> F_4618E
  |       |
  |       +--> N*4 fill A/B                      VERIFIED
  |       +--> 0x64369(A/B element)              VERIFIED
  |       +--> A/B sample/header/sync writes     VERIFIED
  |       +--> early/prepared returns            VERIFIED
 |
 +--> F_411A6
         |
         +--> status branch @0x4128D            VERIFIED
         +--> A/B reload                        NOT FOUND
         +--> queue/DMA/TX propagation          NOT FOUND
         +--> downstream consumer              FIRST OPAQUE BOUNDARY

Separate F29867 base candidate:
base298+0xF8 / base298+0xFC
  -> F_29867
  -> scalar arguments to F_13602/F_13B08
  -> structural ring/cache candidate
  -> no identity proof to D+0xF8/D+0xFC

SE secondary:
ARM SET_IPAUTH_GROUP
  -> mapped MIO byte stores                      VERIFIED
  -> SE/hardware sampling                       OPAQUE
  -> producer of DSP 0xB000001E                 OPAQUE
  -> four DEC 16-bit gate reads                 VERIFIED
```

**Final status:** the requested graph extension is complete to the first honest boundary. A/B are real, distinct memory buffers with verified local transforms, but no statically provable queue, DMA, FIFO, TX, or SND consumer exists after `F_4618E` in the decoded corpus.

**No runtime action was performed. No existing artifact was modified.**
