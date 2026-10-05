# REPORT — Phase D.1 Residual Static Closure

**Date:** 2026-09-22
**Scope:** Static read-only only. No device/ADB. No firmware/binary/Ghidra/historical artifact modifications. New scripts/logs/reports only.
**Parent:** `REPORT_phaseD_static_closure_20260922.md` (Phase D).

---

## 0. Epistemic rules (applied throughout)

```
instruction verified  ≠  data-flow verified  ≠  causal relation verified
```

- **VERIFIED** = direct static evidence of the instruction/data-flow in the corpus.
- **STRONGLY SUPPORTED** = multiple independent static observations, no contradiction.
- **UNPROVEN** = plausible but no direct static proof.
- **OPAQUE BOUNDARY** = named gap that static analysis cannot cross in this phase.
- **SUPERSEDED** = prior claim replaced by better evidence.
- Diagnostic formatting is never treated as semantic proof of license meaning.

### Status carry-in (Phase D, unchanged)

Already VERIFIED: ARM AUTH → SET_IPAUTH_GROUP; `beqi #5`; m6-init compares 2/3/1; readers/writers for +0x2E4/+0x2E6 exist; `lic[]` format + printf site; +0x2E0/+0x2E4 into printf varargs (STRONGLY SUPPORTED); B1/B2 machinery exists; `/vendor/audio_bin` inert; historical N/L2/D1 reclassified.

Not VERIFIED entering D.1 (and still gated by evidence below): `SE IDMA → DSP gate input`; `+0x2E4/+0x2E6 = license state`; `lic[3] = +0x2E6`; `verdict = +0x10C`; `writer 0xae290 = license initializer`; `desc+0x28 = 0` via license gate; B2 receives DTS output descriptor; ARM AUTH state reaches DEC state.

---

## 1. Executive conclusion

Phase D.1 closed the four requested static gaps to the limit of the corpus:

| Task | Result |
|---|---|
| **D.1-A** writer family | **Classification table complete.** True struct writers separated from **stack-spill false positives** (740(r1)). `0xae290..98` = **byte-clear of +0x2E4/5/6 with source r0** → role **reset/init of a 3-byte field**, not proven license write. Best semantic sample: `0x279ad/bd` assemble big-endian halfword from buffer bytes into +0x2E4. |
| **D.1-B** gate input | **All 4 gates VERIFIED** as full MMIO data-flow: `movhi rN,0xB000 → ori #0x1E → lh (16-bit load) → beqi #5`. Fail path separately loads `0xB000001D` (`lbz & 0x7F`). |
| **D.1-C** `lic[]` args | **Two SHM inputs confirmed** (`+0x2E0→r25→stack+80`, `+0x2E4→r24→stack+84` immediately before `jal 0x130CEA`). **No load of +0x2E6 anywhere in the printer.** Third `lic` slot **does not come from +0x2E6 via this function** (static). Register args r4/r5/r6 = `*(r10+52/48/24)` preserved to the call — candidate early format args, **not** proven as `lic[0..2]`. |
| **D.1-D** B2 desc | `desc = root \| 0x4CCC` at `0x41261` (ori). `F_45FD1` **conditionally zeros desc+0x28** (`sw 40(r3),r0`) when `*(r4+16) != 1`. `F_4618E` writes **desc+0xF0 = r14** at entry (size). `+0xF8/+0xFC` written in `F_45DE0` from alloc results (sizes 0x130/0x200) and RMW’d in emit. `+0x16C` **byte-level construction of 0xF872 and 0x4E1F** (IEC 61937 sync/end constants) before F_45DE0. License→`desc+0x28=0` path **NOT established**. |

**No edge below was promoted to VERIFIED causal closure.** Opaque hops named in §7.

---

## 2. D.1-A — `+0x2E4/+0x2E6` writer family

Scripts: `phaseD1_a_writers.py` → `phaseD1_a_writers.txt`; `phaseD1_followup.py` → `phaseD1_followup.txt`.

### 2.1 Critical correction

**Not every `disp=740` store is a field write at +0x2E4.**  
`base == r1` ⇒ stack frame slot (callee-save spill). Those are **excluded**.

| address | function / anchor | dst | width | src | provenance | likely role | confidence |
|---|---|---|---|---|---|---|---|
| **0x279ad** | walk-hit entry **0x2780d** (`addi r1,-24`) | `r3+0x2E4` | `sw` (word) | r25 | `lbz r25,5(r23); lbz r27,4(r23); slli 8; or` — BE halfword from buffer `r23=lwz 724(r3)` | **field copy/assemble** buffer → desc | STRONGLY SUPPORTED |
| **0x279bd** | same entry | `r3+0x2E4` | `sh` (half) | r25 | same pattern from `bytes[30],[29]` of `r23` | second halfword store to same offset (path/drift ambiguity) | STRONGLY SUPPORTED (mechanism), LIKELY same field update |
| **0x27c52** | same func region | `r10+0x2E4` | `sw` | r28 | preceded `lhz r4,728(r10)`; `or r28,r28,r29` | struct field write, base `r10` | STRONGLY SUPPORTED |
| **0x27c56** | same | `r10+0x2E4` | `sh` | r27 | near above | halfword companion | STRONGLY SUPPORTED |
| **0x408d9** | no prologue in ±0x600 | `r11+0x2E4` | `sw` | r17 | window drifted | non-zero field write | LIKELY struct write; role unknown |
| **0x408e5** | same | `r11+0x2E4` | `sw` | **r0** | constant zero | **clear +0x2E4** | STRONGLY SUPPORTED (zero store) |
| **0x21cda** | prologue candidate 0x217a3 (walk drift) | `r11+0x2E4` | `sw` | r14 | `addi r11,…` chain only | struct write | LIKELY; provenance weak |
| **0x72531** | prologue 0x7229b (`-80`) | `r18+0x2E4` | `sw` | r23 | `r18 = r10+r19` | struct write through computed base | LIKELY |
| **0x72587** | same | `r18+0x2E4` | `sw` | r21 | same base | struct write | LIKELY |
| **0xa5a79** | — | `r5+0x2E4` | `sw` | r24 | — | struct write | LIKELY (base≠r1) |
| **0xa615c** | — | `r3+0x2E4` | `sw` | r24 | — | struct write | LIKELY |
| **0xae290** | prologue **not found** within 0x600; backward chain drifts | `r10+0x2E4` | `sbz` byte | **r0** | constant zero | **clear byte +0x2E4** | STRONGLY SUPPORTED (zero) / role **reset**, NOT “license writer” |
| **0xae294** | same family | `r10+0x2E5` | `sbz` | **r0** | zero | clear +0x2E5 | STRONGLY SUPPORTED |
| **0xae298** | same family | `r10+0x2E6` | `sbz` | **r0** | zero | clear +0x2E6 | STRONGLY SUPPORTED |
| **0xb003d** | — | `r25+0x2E4` | `sbz` | **r24** | **non-zero** | byte write of computed/arg value | LIKELY non-clear write |
| **0xb0051** | — | `r10+0x2E5` | `sbz` | **r0** | zero | clear +0x2E5 | STRONGLY SUPPORTED |
| **0x129932** | — | `r19+0x2E4` | `sbz` | **r6** | non-zero | byte write | LIKELY |
| **0x13602a** | — | `r10+0x2E4` | `sw` | r24 | — | word write | LIKELY |
| **0xce66b** | entry **0xce5f5** `addi r1,-776` | **`r1+740`** | `sw` | r17 | stack save block `sw r11..r22 to 720..764(r1)` | **STACK SPILL — not field** | VERIFIED (false positive) |
| **0x129128** | entry **0x129110** `addi r1,-776` | **`r1+740`** | `sw` | r17 | same spill pattern | **STACK SPILL — not field** | VERIFIED (false positive) |

### 2.2 `0xae290` family — what is proven

```
0xae290: f80a02e4  bg.sbz 740(r10),r0   → byte[+0x2E4] = 0
0xae294: f80a02e5  bg.sbz 741(r10),r0   → byte[+0x2E5] = 0
0xae298: f80a02e6  bg.sbz 742(r10),r0   → byte[+0x2E6] = 0
```

- **Source:** register **r0 (zero)** only → **clear/reset**, not copy, not computed assign.
- **Function identity:** OPEN (no prologue within 0x600; end-chain drifts into `bg.op*` stream).
- **Do not label “license initializer.”** It clears a 3-byte region that *overlaps* SHM fields the printer also reads — overlap ≠ proven semantic license object.

### 2.3 Best non-zero data-flow example (0x279ad)

```
27986: lwz r23, 724(r3)          ; source buffer in struct
279a1: lbz r25, 5(r23)
279a4: lbz r27, 4(r23)
279a7: slli r25, r25, 8
279aa: or  r25, r25, r27          ; r25 = BE(u16) from bytes[5:4]
279ad: sw  r25, 740(r3)           ; store to +0x2E4
279b1: lbz r25, 30(r23)
279b4: lbz r27, 29(r23)
279b7: slli/or                    ; second pair
279bd: sh  r25, 740(r3)           ; halfword store same offset
```

**Role:** extract/assign a 16-bit (possibly 32→16 narrowing) field from an in-struct buffer into offset +0x2E4. **Not zero, not stack.** Whether that field is “license” remains **UNPROVEN** without a consumer that interprets it as license.

---

## 3. D.1-B — DEC `==5` input data-flow (ALL FOUR GATES)

Scripts: `phaseD1_b_gate.py`, `phaseD1_b2_refine.py` → `phaseD1_b_gate.txt`, `phaseD1_b2_refine.txt`.

### 3.1 Deliverable table

| site | read instruction | source MMIO | register | width | transform | compare | branch target | status |
|---|---|---|---|---|---|---|---|---|
| **0x21AC3** | `21aa4 movhi r24,0xB000; 21aa8 ori r25,r24,#0x1E; 21aab bn.lh r26,0(r25)` | **0xB000001E** | r26 | **16-bit** (`lh`) | none before cmp (address formed by movhi+ori) | `bg.beqi r26, #5` | 0x21AD9 | **VERIFIED** |
| **0x21B19** | `21b0f movhi r24,0xB000; 21b13 ori r27,r24,#0x1E; 21b16 bn.lh r27,0(r27)` (end-chain exact) | **0xB000001E** | r27 | **16-bit** | — | `bg.beqi r27, #5` | 0x21B2B | **VERIFIED** |
| **0x21F0E** | `21eef movhi r24,0xB000; 21ef3 ori r25,r24,#0x1E; 21ef6 bn.lh r26,0(r25)` | **0xB000001E** | r26 | **16-bit** | — | `bg.beqi r26, #5` | 0x21F24 | **VERIFIED** |
| **0x21F5C** | `21f52 movhi r24,0xB000; 21f56 ori r27,r24,#0x1E; 21f59 bn.lh r27,0(r27)` (end-chain exact) | **0xB000001E** | r27 | **16-bit** | — | `bg.beqi r27, #5` | 0x21F6E | **VERIFIED** |

Evidence quality: every-offset multi-align decode + instruction-end chain (`decode_end`) confirming the `lh` sits immediately before each `beqi`. Fail path independently verified:

```
ori r24, r24, #0x1D     ; r24 still 0xB000 base → 0xB000001D
lbz r23, 0(r24)         ; byte load
andi r23, r23, #0x7F
beqi r23, #0 → error-path continuation
```

### 3.2 Interpretation

- Gate value is a **16-bit MMIO read** at `0xB000001E`, compared to **#5** with **no intervening mask/shift** on the loaded value (address arithmetic only).
- `0xB000001D` is a **separate** byte-wide status read on the fail path (`& 0x7F`) — not the gate source.
- Decoder note: printed imm `178/146` is the known low16 bug; true imm = `(word>>16)&0x1F = 5` (Phase D sinc proof). Branch targets match Ghidra.

**This closes instruction+data-flow for “what is compared to 5.”**  
It does **not** prove who *writes* `0xB000001E` (SE/DSP producer) — that hop stays **OPAQUE** (§7).

---

## 4. D.1-C — printf `lic[]` argument provenance

Scripts: `phaseD1_c_printf.py`, `phaseD1_b2_refine.py`, `phaseD1_followup.py`.

### 4.1 What is established

| fact | evidence | status |
|---|---|---|
| printf site `0x1ca37 → jal 0x130CEA` | byte `e4228566`; target prologue `addi r1,-32` | **VERIFIED** |
| format lea `addi r3,r3,10190` at `0x1ca1c` → `0x1427CE` **if** r3=0x140000 | unique imm 10190 image-wide (1 hit); string at 0x1427CE contains `lic[%d,%d,%d]` | **STRONGLY SUPPORTED** (r3 high via earlier `movhi r3,0x14` only seen on drifted gap path — see blockers) |
| `+0x2E0 → r25` | `0x1ca12: lwz r25, 736(r13)` | **VERIFIED** load |
| `+0x2E4 → r24` | `0x1ca16: lwz r24, 740(r13)` | **VERIFIED** load |
| r25/r24 pushed to outgoing stack before call | `1ca31: sw 80(r1),r25`; `1ca34: sw 84(r1),r24` then `1ca37: jal` | **STRONGLY SUPPORTED** = printf inputs |
| **no `lwz/lh 742(r13)` (or any 742) in 0x1c93e..0x1ca60** | dedicated disp scan | **VERIFIED absence in this function** |
| r4 last def `1c9ad: lwz r4,52(r10)`; r5 `1c9aa: lwz r5,48(r10)`; r6 `1c9a7: lwz r6,24(r10)`; **no redef** before jal | def scan 1c9d0–1ca37 | **VERIFIED** live at call |
| r7 clobbered at `1c9fe: addi r7,r14,-4862` | def scan | **VERIFIED** |

Stack save block immediately pre-call (clean multi-align):

```
64(r1) ≤ r29
68(r1) ≤ r28
72(r1) ≤ r27
76(r1) ≤ r26
80(r1) ≤ r25   ← +0x2E0 value
84(r1) ≤ r24   ← +0x2E4 value
```

### 4.2 Argument-order reasoning (not yet proof)

- If AEON follows r3=fmt, r4..r7=first four variadic words, then stack slots 64+ = subsequent words: **r4/r5/r6 are live early structure loads**; **lic[] is late in a ~28-conversion format string** (Phase D string body) → stack slots 64–84 alone cannot hold all conversions.
- Therefore either (a) more outgoing slots exist outside the decoded window, (b) a different calling convention packs differently, or (c) this jal’s register/stack setup is partially hidden by encoding gaps (`888b` at 0x1ca1a).
- **We do not assign `lic[0]=r4`, `lic[1]=r5`, `lic[2]=r6` without explicit order proof.**

### 4.3 Third `lic` slot

| hypothesis | static result |
|---|---|
| `lic[2]` ← `+0x2E6` | **DISPROVEN for this printer** — no load of offset 742 in the function. |
| `lic[2]` ← some other SHM word | **UNPROVEN** — no third `73x/74x(r13)` load found. |
| `lic[*]` ← r4/r5/r6 from `r10+52/48/24` | **CANDIDATE only** — live at call, semantics of those fields unknown. |

**Status labels (mandatory):**

```
format string        = VERIFIED
printf site          = VERIFIED
+0x2E0 = printf input = STRONGLY SUPPORTED (load+stack+adjacent jal)
+0x2E4 = printf input = STRONGLY SUPPORTED
+0x2E6 = third lic    = UNPROVEN  (no static load in printer)
lic[i] semantic map   = UNPROVEN  (printf order not proven)
```

---

## 5. D.1-D — B2 descriptor / output producer (static reaudit)

Scripts: `phaseD1_d_b2desc.py`, `phaseD1_followup.py`.

### 5.0 Descriptor identity

| item | finding | status |
|---|---|---|
| raw `00 01 4C CC` LE/BE in image | **0 hits** | — |
| `ori r3,r3,#0x4CCC` at **0x41261** (and 0x4124d, 0x41277 on r3/r23) | constructs `root\|0x4CCC` when high bits already `0x1`… | **STRONGLY SUPPORTED** `desc = root+0x14CCC` |
| callers | `0x41283 jal 0x45FD1`; `0x41289 jal 0x4618E`; `0x4128d bnei r12 → 0x411C9` | **VERIFIED** |
| pre-inits | `jal 0x45E52` at 0x4126d / 0x41259 | STRONGLY SUPPORTED (init before B1/B2) |

### 5.1 `desc+0x28` (disp 40)

**Writers (struct-relevant):**

| addr | inst | value | context |
|---|---|---|---|
| **0x45ff5** | `bn.sw 40(r3),r0` | **0** | inside `F_45FD1` after `45ff1: beqi r11, TI=1 → 0x46080` — **zeros +0x28 when r11 ≠ 1** |
| 0x44641 | `sw 40(r11),r23` | r23 | non-zero writer outside F_45FD1 |
| 0x39e1d / 0x3a599 / 0x3db21 / … | various | various | other modules (0x30xxx) |

**Readers:** `0x4600c lwz 40(r4)`; `0x46016/33/79 lwz 40(r10)` + `0x46019 bnei r23,#1 → 0x45FFA`.

**Condition chain (F_45FD1):**

```
45fee: lwz r11, 16(r4)          ; input object field
45ff1: beqi r11, #1 → 46080     ; if ==1 skip zeroing
45ff5: sw  40(r3), r0           ; desc+0x28 = 0
```

Also nearby `sw 52(r3),r0` / `sw 56(r3),r0` (adjacent clears).

| question | answer | status |
|---|---|---|
| writers identified? | yes (zero @45ff5; value @44641; …) | **VERIFIED** |
| readers + compare-to-1? | yes | **VERIFIED** |
| codec-specific? | unknown field at `*(r4+16)` | **UNPROVEN** |
| license-specific? | **no static path DTS/license → +0x28=0** | **UNPROVEN — do not assert** |
| simply init/reset? | zero-store is conditional on `*(r4+16)!=1` — **conditional clear**, not unconditional init | STRONGLY SUPPORTED as conditional clear |

**Main question `DTS → license → desc+0x28=0`: NOT closed (UNPROVEN).** Treat `*(r4+16)` as opaque input.

### 5.2 `desc+0xF0` (disp 240) — output size

| item | finding | status |
|---|---|---|
| structure writer in 0x40000–0x48000 | **only** `0x461ab: bg.sw r14, 240(r10)` in `F_4618E` (post-prologue) | **VERIFIED** (single writer in window) |
| stack false positive | `0x41003/0x41025: 240(r1)` | excluded |
| r14 provenance inside emit entry | **no def** in 0x4618e–0x461b5 → **incoming** (arg/caller-saved) | **OPAQUE at emit boundary** |
| branch on size==0 after store | **none found** on the stored +0xF0 in emit body | **UNPROVEN** (size written early; zero/nonzero decision not visible as `if size==0` on that field) |
| relation to desc+0x28 | no static data-flow between 45ff5 and 461ab | **UNPROVEN** |

Prior claim “output size can become 0”: consistent with `r14=0` possible, but **r14’s producer is an OPAQUE BOUNDARY** until caller/`0x45E52` path is traced (left open — see §10).

### 5.3 `desc+0xF8 / +0xFC` (248 / 252)

**Writers (F_45DE0):**

```
45dfd: ori r5, r0, #0x130          ; size 304
45e04: jal 0x130D25                ; alloc/copy-like
45e08: lwz r3, 0(r13)              ; pointer from table r13
45e17: sw  r3, 248(r10)            ; desc+0xF8 = ptr_A

45e1d: ori r5, r0, #0x200          ; size 512
45e21: jal 0x130D25
45e25: lwz r3, 4(r13)
45e2f: sw  r3, 252(r10)            ; desc+0xFC = ptr_B
```

**Readers / use (F_4618E):**

```
46222: lwz r5, 248(r10)            ; ptr_A
46226: lwz r23, 252(r10)           ; ptr_B
4622a: add r24, r5, r11            ; A + index
4622d: lwz r25, 0(r24)             ; load through A
46238: sw  0(r24), r25             ; RMW through A
4623b: lwz r24, 0(r23)             ; load through B
46246: sw  0(r23), r24             ; RMW through B
```

| question | answer | status |
|---|---|---|
| pointer origins | post-`jal 0x130D25` results from `r13[0]` / `r13[1]`, sizes 0x130 / 0x200 | **STRONGLY SUPPORTED** |
| same descriptor instance? | both use **r10** base in F_45DE0 and F_4618E; caller chain 45E52→45FD1→4618E | **STRONGLY SUPPORTED** |
| point into generated A/B buffers? | RMW `ptr+index` pattern consistent with dual buffers; buffer fill not in this window | **LIKELY** (generation site not fully walked) |
| copied outward? | no external `desc+0xF8` consumer found in window | **UNPROVEN** |

### 5.4 `+0x16C..+0x173` (364..371)

**Byte-level construction (before F_45DE0, window 0x45d00–0x45de0):**

```
45d3f: addi r24, r0, #-1934        ; r24 = 0xFFFFF872 → low16 = 0xF872  IEC sync
45d4c: sw  r24, 364(r3)            ; +0x16C word
45d50: addi r24, r0, #19999        ; r24 = 0x4E1F                         IEC end/IF
45d54: sh  r5,  368(r3)            ; +0x170 half
45d58: sh  r24, 364(r3)            ; +0x16C half
45d5c: sw  r23, 368(r3)            ; +0x170 word
; pattern repeats: 45d92/96, 45da9/b1/b5, 45dd2/d6, 45d8a, 45dca …
```

Constants observed: **`0xF872`** (IEC 61937 sync) and **`0x4E1F`** (common IEC end/DTS-related). Also `ori r23,r0,#0x10C` / `#0x111` / `#0x20D` / `#0x311`… as small mode words in the same block (**unrelated to the superseded “verdict +0x10C” claim** — do not reintroduce that reading).

**Reader in emit:** `0x461e5 lhz r24,364(r10)`; `0x462b5 lhz r25,368(r10)`; `0x462bf lhz r24,370(r10)`.

| question | answer | status |
|---|---|---|
| exact writes? | yes, listed above | **VERIFIED** |
| IEC preamble metadata? | **values 0xF872 / 0x4E1F match IEC 61937 sync/end constants**; stored at +0x16C/+0x170 | **STRONGLY SUPPORTED as IEC-like preamble constants** — not labeled solely by location |
| init before/after +0xF0? | construction @0x45d3f… **before** F_45DE0/F_45FD1; +0xF0 written inside F_4618E later | **STRONGLY SUPPORTED order** |
| DTS-specific values constructed? | 0x4E1F used in DTS/IEC contexts; no full DTS substream frame built in this window | **CANDIDATE** (not fully proven as complete DTS preamble assembly) |
| base r3 vs readers’ r10 | writers use r3, readers r10 — **same object only if r3 held desc during construction** | **LIKELY / not bit-proven** (no `r3==r10` tie in window) |

### 5.5 `F_4618E` reconstructed skeleton (partial)

```
4618e: addi r1, r1, -32                 ; prologue VERIFIED
461a3: jal 0x2EAE6                      ; helper (opaque)
461a8: beqi r23, #1 → 461C4             ; prepared/flag gate (r23 origin OPAQUE here)
461ab: sw  r14, 240(r10)                ; desc+0xF0 = size (r14 incoming)
461e5: lhz r24, 364(r10)                ; load +0x16C (IEC word)
461ee: sfeqi r25, r0                    ; zero-check r25 (from lwz 48(r10) at 461e2) → field+0x30 NULL/zero
461f7: bnei r24, … → 462A0              ; branch on preamble/type half
4621e: jal 0x64369                      ; bit-reader (prior B1/B2 report)
46222/46226: load F8/FC pointers
4622a..46246: RMW through A and B
462b5/462bf: lhz +0x170/+0x172
… type dispatch: beqi r24 TI∈{0,2,4,8} and many bnei steps …
```

**Observed predicates:** `r23==1` (early), `r25==0` (null-ish), `r24` type switch (0/2/4/8…), `r23==0` at 46376.  
**Not observed:** `if desc+0xF0 == 0`, `if prepared==0` with named field, `if buffer==NULL` as a single clean `beqi` on F8/FC (closest: `sfeqi r25,r0` on **+0x30**, and RMW assumes non-NULL — a NULL F8 would be an unchecked fault path unless guarded earlier).

**B2 “really receives DTS output descriptor”:** machinery reads/writes the same r10 object that holds IEC-like words and A/B pointers — **STRONGLY SUPPORTED** as output descriptor handling; **full causal “this is THE DTS output” remains one OPAQUE hop** (fill of A/B buffers + caller meaning of r14).

---

## 6. VERIFIED graph

Only edges with direct static data-flow proof in-corpus:

```
ARM IPCheck → AV+0x440/0x444 → CheckHashkey → ApplyHashkey → SET_IPAUTH_GROUP
        │                    [Phase D D1 — VERIFIED instructions]
        ▼
AbsWriteByte → SE IDMA 0x112A82/83
        │                    [VERIFIED write instructions]
        ▼
   ═══ OPAQUE: SE → DSP 0xB000001E producer ═══

*(u16*)0xB000001E   ← movhi/ori/lh chain @ 21aa8/21aab, 21b13/21b16,
                       21ef3/21ef6, 21f56/21f59          [D.1-B VERIFIED]
        ▼
beqi #5  @ 0x21AC3 / 0x21B19 / 0x21F0E / 0x21F5C          [VERIFIED]
        ▼ fail
*(u8*)0xB000001D & 0x7F                                   [VERIFIED]

m6-init beqi #2/#3/#1 @ 0x1e3fb…                          [Phase D VERIFIED]

F_45FD1:
  *(u16*)(src+16)==1 ? skip : desc+0x28 = 0               [D.1-D VERIFIED store/branch]
  readers lwz +0x28; bnei #1                              [VERIFIED]

F_45DE0:
  alloc(0x130) → desc+0xF8                                [VERIFIED store]
  alloc(0x200) → desc+0xFC                                [VERIFIED store]
  r24=0xF872 / 0x4E1F → desc+0x16C/+0x170 construction    [VERIFIED inst+imm]

F_4618E:
  prologue; desc+0xF0 = r14                               [VERIFIED store]
  lhz/lwz +0x16C/+0xF8/+0xFC; RMW *F8/*FC                 [VERIFIED loads/RMW]
  type beqi on r24                                        [VERIFIED branches]

Printer 0x1c93e:
  lwz +0x2E0 → r25; lwz +0x2E4 → r24; sw to 80/84(r1); jal 0x130CEA
                                                          [VERIFIED loads/stack/jal]
  format lea addi #10190 → 0x1427CE                       [VERIFIED imm; r3-high STRONGLY SUPPORTED]

Struct +0x2E4/5/6:
  zero byte-clear family ae290/94/98 (src r0)             [VERIFIED clear]
  BE halfword assemble 279ad/bd from buffer               [VERIFIED data-flow]
  stack spills ce66b/129128 (r1+740)                      [VERIFIED as spills, not fields]
```

---

## 7. Candidate / opaque graph

```
ARM AUTH … SET_IPAUTH_GROUP … SE IDMA
        │ VERIFIED
        ▼
   ??? SE receiver / mailbox implementation
        │ OPAQUE BOUNDARY  (no static writer of 0xB000001E found)
        ▼
DSP 0xB000001E == 5
        │ VERIFIED read+compare          [D.1-B closed]
        ▼
gate pass → m6-init 2/3/1 → config-apply
        │ VERIFIED compares; config-apply walk VERIFIED
        ▼
DM record @ 0x1C010000+idx*0x290 (runtime-created)
        │ STRONGLY SUPPORTED layout; content OPAQUE statically
        ▼
SHM +0x2E0/+0x2E4  ──printf──►  format 0x1427CE lic[]
        │ STRONGLY SUPPORTED as inputs
        │ semantic license meaning OPAQUE
        ▼
   +0x2E6 third lic slot: UNPROVEN (no printer load)

desc+0x28 = 0  ◄── *(src+16)!=1 in F_45FD1     [VERIFIED clear]
        │ license/DTS cause: UNPROVEN
        ▼
F_4618E size=F8/FC/IEC fields → type switch → buffers
        │ machinery VERIFIED; “DTS output descriptor” LIKELY, causal hop OPAQUE
        ▼
SPDIF/HDMI sink path (prior phases)
```

---

## 8. Corrections to Phase D

| Phase D statement | D.1 correction |
|---|---|
| Gate `==5` VERIFIED via Ghidra/sinc | **Now also VERIFIED with full in-image MMIO data-flow** (movhi+ori+lh+beqi) at all 4 sites — stronger than immediate-only proof. |
| Writers of +0x2E4/+0x2E6 exist | **Split:** real struct writers vs **stack false positives** at `740(r1)` (0xce66b, 0x129128). Family at 0xae290 = **zero byte-clear**, role **reset**, not “license writer.” |
| `+0x2E0/+0x2E4` → printf STRONGLY SUPPORTED | **Confirmed load+stack adjacency**; **third lic input not +0x2E6** (no load). `lic[]`↔SHM index map remains UNPROVEN. |
| `desc+0x28` zero path open | **Store/branch located** (`45ff1/45ff5`); **license cause still UNPROVEN**. |
| `+0xF0` output size | **Single struct writer `461ab: sw r14,240(r10)`**; r14 origin OPAQUE; no `if size==0` branch shown. |
| `+0x16C` “IEC preamble?” location-only | **Upgraded:** explicit **0xF872 / 0x4E1F** immediates constructed into +0x16C/+0x170 — STRONGLY SUPPORTED IEC constants (still not a full DTS frame proof). |
| `record +0x10C = verdict` | **Remains SUPERSEDED.** Incidental `ori #0x10C` in the 0x45d83 preamble block is a **mode constant**, not a verdict field — do not revive the old claim. |
| `0x14CCC` as raw addi | **0 raw hits**; only `ori #0x4CCC` construction sites. |

---

## 9. Remaining static blockers (after D.1)

1. **OPAQUE:** producer/writer of `0xB000001E` (and relationship of SE IDMA 0x112A82/83 to that MMIO) — no static writer in dec corpus.
2. **OPAQUE:** DM record *contents* (verdict/license words) — runtime-created (Phase D static multi-2 scan = 0).
3. **UNPROVEN:** semantic identity of SHM +0x2E0/+0x2E4/+0x2E6 as license vs telemetry (printer proves *consumption as `%d`*, not meaning).
4. **UNPROVEN:** printf argument index mapping to `lic[0..2]`; **+0x2E6 not loaded** in printer.
5. **UNPROVEN:** `*(r4+16)` semantics in F_45FD1 (license? codec? mode?).
6. **OPAQUE:** r14 → `desc+0xF0` producer (caller / 0x45E52 path).
7. **UNPROVEN:** A/B buffer fill path into F8/FC pointees (alloc verified; write-into-buffer site not fully walked).
8. **BLOCKED (decoder):** exact function boundary of `0xae290` family (no prologue ±0x600; `STT_FUNC` giant-symbol bug); bytes `0x1ca1a` gap (`888b`) before format lea; F_45FD1 entry byte at 0x45FD1 itself `<gap>` (walk starts 0x45FF1…).
9. **UNPROVEN:** `r3 == desc` during +0x16C construction (writers on r3, readers on r10).
10. **UNPROVEN:** ARM AUTH state reaches DEC state end-to-end (blocked by #1).

---

## 10. Exact runtime questions remaining after D.1

Single future session (not started now). Ordered by edge:

1. Who writes `0xB000001E`? Read it and `0xB000001D` live at each gate; correlate with SE IDMA 0x112A82/83 writes.
2. Dump DM record `0x1C010000+idx*0x290` including the real verdict field offset (stop calling it +0x10C until seen).
3. At `jal 0x130CEA`, capture r3,r4,r5,r6,r7 and stack 64..N → map to `lic[%d,%d,%d]` and to +0x2E0/+0x2E4/+0x2E6.
4. Read SHM +0x2E0/+0x2E4/+0x2E6 during passthrough; compare to printer line.
5. Break `F_45FD1` at 0x45ff1: what is `*(r4+16)` when +0x28 is zeroed? Is DTS selected?
6. Break `F_4618E` at 0x461ab: value of r14; watch +0xF0 over a frame.
7. Dump desc at `root+0x14CCC`: +0x28/+0xF0/+0xF8/+0xFC/+0x16C..+0x173 during DTS on/off.
8. Identify function owning `0xae290` clear (symbol/backtrace).
9. Fill A/B buffers: confirm F8/FC pointees receive generated frames (size 0x130/0x200).
10. Confirm `0xF872/0x4E1F` lands in the emitted SPDIF/DTS byte stream.

---

## 11. Explicit DO-NOT-REINVESTIGATE list

Do **not** reopen in static phase or “while waiting” for runtime:

- SHM33, EDID, Kodi, 0x97, 0x81, old N-series, T615 porting, new firmware patch design, new experiments.
- Re-proving `/vendor/audio_bin` inertness or K-1/K-2/P-BYP Exp-B results (Phase D / D7 hash tables stand).
- Full-corpus re-sweeps, glob storms, giant linear decode sweeps (300s timeout).
- Reviving `record+0x10C = verdict` without a new decoded load.
- Calling `0xae290` a “license writer” without source≠0 or a consumer proof.
- Calling `lic[i] = SHM+i` from printf order alone.
- Calling `desc+0x28=0` a license gate without `*(r4+16)` identity.
- Calling `+0x16C` “IEC preamble” **only** by location (now we cite 0xF872/0x4E1F; still stop short of full DTS frame proof).
- Treating stack `740(r1)` stores as field writes.
- Any device/ADB/flash until an explicit runtime authorization.

### Tooling rules (unchanged)

AEON big-endian; `bg.beqi` true imm = `(word>>16)&0x1F`; bounded regions; end-chain + multi-align instead of pure linear sweep; Ghidra `dec06_out.txt` / `.sinc` as immediate oracle; `aeon_decode.py` known gaps (`89c32161`, F_45FD1 entry, 0x1ca1a); no STT_FUNC boundaries from `func_at`.

---

## 12. Artifacts (new only)

| path | role |
|---|---|
| `Temp\opencode\phaseD1_b_gate.py` | D.1-B first pass |
| `Temp\opencode\phaseD1_b2_refine.py` | D.1-B all-4-gate proof |
| `Temp\opencode\phaseD1_a_writers.py` | D.1-A writers |
| `Temp\opencode\phaseD1_c_printf.py` | D.1-C printf |
| `Temp\opencode\phaseD1_d_b2desc.py` | D.1-D descriptor |
| `Temp\opencode\phaseD1_followup.py` | ABI/size/writer classification |
| `Temp\opencode\phaseD1_*.txt` | all logs above |
| `dec_work\REPORT_phaseD1_residual_static_20260922.md` | this report |

No prior report, binary, firmware, or Ghidra project was modified.

---

*End of Phase D.1. Static limits reached; runtime session remains unauthorized until explicitly ordered.*
