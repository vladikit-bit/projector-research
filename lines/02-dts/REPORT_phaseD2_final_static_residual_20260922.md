# REPORT — Phase D.2 Final Static Residual Closure

**Date:** 2026-09-22
**Scope:** Static read-only only. No device/ADB, no runtime, no breakpoints, no patches, no firmware writes, no state changes. New analysis scripts/logs/reports only. Historical artifacts not modified.
**Parents:** `REPORT_phaseD_static_closure_20260922.md` (Phase D), `REPORT_phaseD1_residual_static_20260922.md` (Phase D.1).
**Deliverable:** this file.

---

## 0. Epistemic rules and scope (applied throughout)

```
instruction verified  ≠  data-flow verified  ≠  causal relation verified
```

Status vocabulary: `VERIFIED | STRONGLY SUPPORTED | LIKELY | UNPROVEN | DISPROVEN | SUPERSEDED | ARTIFACT/ERROR | INERT-SURFACE`, plus `OPAQUE BOUNDARY` and `BLOCKED BY DECODER`.

**D.2 scope rule (D.2-H):** no web research, new firmware extraction, patch, runtime, broad reread, or re-checking old dead ends. Authoritative artifacts only: `utpa2k_stock.ko`, `mik_stock.ko`, `dec_full.bin`, `snd_full.bin`, Ghidra AEON listing `dec32_clean.txt`, `aeon_ORBIS32.sinc`, existing outputs, Ghidra ARM dumps under `spdif_audio_investigation\tools\`.

**Special requirements carried forward:**
- Do NOT mix structure instances: same disp `+0x2E4` with different base (r3/r10/r11/r18/r25) = separate provenance.
- `r1+740` = STACK forever; `0xce66b` and `0x129128` permanently excluded from structural +0x2E4 writers.
- `0x1CA37`: format string VERIFIED, printf VERIFIED, +0x2E0/+0x2E4 loads VERIFIED, semantics UNPROVEN, `lic[3] ↔ +0x2E6` DISPROVEN FOR THIS PRINTER — do not revive.
- Do NOT follow the old `mask[7:0]/[15:8]/[23:16]` byte-order assumption as a prior — derive order only from `ubfx/uxtb/mov/add/AbsWriteByte` instruction order (done in D.2-A).
- Never call `*(r4+16)` a license flag without source/consumer evidence.

**Scripts/log evidence this phase:** `phaseD2_followup.py` → `phaseD2_followup.txt` (Capstone ARM, raw-hit contexts, listing windows); `phaseD2_cdg.txt` (AEON walk D.2-C/D/E/F/G); `phaseD2_b_bridge.txt` (literal scans); Ghidra dump `dd_MDrv_AUDIO_ApplyHashkey.txt` / `dd_ApplyHashkey.txt`; listing `dec32_clean.txt`.

---

## 1. Executive conclusion — seven-item status table

| Item | Question | Status |
|---|---|---|
| **D.2-A** | Exact SE payload bytes (0x112A82–85 construction order) | **VERIFIED** (instruction sequence + byte order from ubfx/uxtb; overwrite/FIFO-vs-final still open as sub-question) |
| **D.2-B** | SE IDMA → DSP `0xB000001E` hop (code writer of the gate word) | **OPAQUE BOUNDARY** (all raw hits classified as data, not code; no ARM/DEC code writer found) |
| **D.2-C** | `*(r4+16)` identity, r4 provenance, +0x10 producer | **Mixed:** r4 provenance + F_45FD1 gate data-flow **VERIFIED**; `+0x10==1` producer **OPAQUE BOUNDARY**; `*(r4+16)` as license flag **UNPROVEN** (do not name) |
| **D.2-D** | r14 producer → desc+0xF0 | **STRONGLY SUPPORTED** — r14 = `*(desc+0x38)` (element count N) on the non-trivial path; store at `0x461ab` **VERIFIED**; early path stores 0 |
| **D.2-E** | F8/FC alloc + fill | **VERIFIED** — alloc/memset in `F_45DE0`; content fill loops + IEC constants in `F_4618E` |
| **D.2-F** | Boot DM creation / stride 0x290 / base 0x1C010000 | **STRONGLY SUPPORTED** — `muli *,656` + `movhi/addi` base + `jal 0xDFB1` allocator **VERIFIED as instructions**; printed movhi imm `0xe0` vs raw `1c01` is decoder discrepancy (imm **0x1C01** under bit-field correction → base **0x1C0126B0**); boot-time identity of 0xDFB1 **OPAQUE BOUNDARY** |
| **D.2-G** | SHM +0x2E0/+0x2E4/+0x2E6 semantics | **Mixed:** buffer→field data-flow **VERIFIED** (`0x27986–0x279f8`); printf consumption of +0x2E0/+0x2E4 **VERIFIED** (D.1); **license semantics UNPROVEN**; `lic[3]↔+0x2E6` remains **DISPROVEN** for printer |

**Phase-level call:** every D.2-A…G target now has a bounded status backed by primary evidence. Residual open items are named `OPAQUE BOUNDARY` (static cannot cross them in this corpus/phase) or explicitly `UNPROVEN` semantic labels.

**Declaration: STATIC EXHAUSTED — RUNTIME JUSTIFIED** for the seven D.2 questions as posed. Runtime is **not** entered in this phase (requires explicit user authorization).

No edge below is promoted to a full causal license-gate closure.

---

## 2. D.2-A — SE payload exact bytes

**Question:** exact instruction-level construction of IDMA words at 0x112A82–85 in `SET_IPAUTH_GROUP`, and whether a sibling pusher reuses the same base.

**Evidence:** `dd_MDrv_AUDIO_ApplyHashkey.txt` (Ghidra ARM @0x424BD8); Capstone window `phaseD2_followup.txt` lines 178–247; sibling/bulk windows lines 2–176; symbols `HAL_AUDIO_AbsWriteByte/MaskReg/MaskByte/Reg` confirmed in `.ko` string table (lines 271–274).

### 2.1 SET_IPAUTH_GROUP @0x424BD8 — VERIFIED sequence

```
0x424bd8 push {r4,r5,r6,r7,fp,lr}
0x424bdc mov  r4, r1                 ; r4 = mask (AV+0x440/0x444), preserved
0x424be0 cmp  r0, #0
0x424bf4 mov  r5, #0x84              ; r0==1 → dspId 0x84
0x424bfc mov  r5, #0x83              ; r0==0 → dspId 0x83
0x424c44 movw r6, #0x2a82
0x424c4c movt r6, #0x11              ; r6 = 0x112A82
0x424c48 movw r1, #0xfff0
0x424c50 add  r0, r6, #0x3e          ; [0x112AC0] AbsWriteMaskReg 0xFFF0 ← 0
0x424c5c sub  r0, r6, #4             ; [0x112A7E] ← 1,1 ; poll #8/#0x10
0x424c8c add  r0, r6, #2 ; mov r1,r5 ; AbsWriteByte  [0x112A84] = dspId (0x83/0x84)
0x424c98 add  r0, r6, #3 ; mov r1,#0x1f ; AbsWriteByte [0x112A85] = 0x1F
0x424ca4 ubfx r1, r4, #8, #8  ; mov r0, r6      ; AbsWriteByte [0x112A82] = mask[15:8]
0x424cb0 add  r5, r6, #1
     ubfx r1, r4, #0x10, #8   ; mov r0, r5      ; AbsWriteByte [0x112A83] = mask[23:16]
0x424cc0 uxtb r1, r4          ; mov r0, r6      ; AbsWriteByte [0x112A82] = mask[7:0]  (overwrites)
0x424ccc mov  r0, r5 ; mov r1, #0               ; AbsWriteByte [0x112A83] = 0          (overwrites)
0x424cd8 mov  r0, #0x10 ; bl poll ; return
```

**Byte order DERIVED from actual `ubfx/uxtb` operand order** (allowed — not the forbidden prior assumption). Final architectural state if AbsWriteByte is a plain store: `[0x112A82]=mask[7:0]`, `[0x112A83]=0`, `[0x112A84]=dspId`, `[0x112A85]=0x1F`.  
**Sub-question OPEN:** whether AbsWriteByte is FIFO/port (each write observed by SE) vs final-state only — **instruction sequence VERIFIED; hardware semantics OPAQUE BOUNDARY**.

### 2.2 Sibling and bulk pushers — VERIFIED same base, different selector

| site | role | payload difference |
|---|---|---|
| `0x423298` sibling | codec-id check `r0∈{0x1F83,0x1F84}` && `r2==0` → special paths; else same `movw/movt r8,#0x2a82/0x11`, mask-reg clear, poll, `add r0,r8,#2; uxtb r1,r6` writes **full r6 low byte to 0x112A84**; also `lsr r1,r6,#8` at 0x42336c | not the SET_IPAUTH dspId/0x1F/`mask` triple |
| `0x42D7E8` bulk | same base `r6=0x112A82`; `uxtb r1,r4 → [base+2]`, `ubfx r1,r4,#8,#8 → [base+3]`, then loop over array `[sl+r0<<2]` writing `ubfx` byte lanes into base/+1 | array-driven bulk, not AV mask r4 |

**STATUS D.2-A: VERIFIED** for exact SET_IPAUTH payload construction and byte order; sibling/bulk **VERIFIED** as co-tenants of 0x112A82 with different semantics. FIFO-vs-final **OPAQUE BOUNDARY**.

---

## 3. D.2-B — SE → DSP `0xB000001E` bridge

**Question:** find a **code** writer that produces gate word `0xB000001E` (or the SE mailbox content that becomes it) statically.

**Evidence:** `phaseD2_b_bridge.txt` (full-image LE32/BE32 scans); `phaseD2_followup.txt` lines 314–346 (raw-hit contexts + AEON decode at dec 0x178d41).

### 3.1 Hit classification

| file | encoding | file offs | context classification |
|---|---|---|---|
| `dec_full.bin` | LE32 `0xB000001E` | **0x178d41 only** | surrounded by float-like dword stream (`…9d1e0000 b0050000…`); AEON decode at 0x178d30–0x178d6c is **nonsensical data** (`bn.op0_*`, `sw 84(r0)`), **not code** |
| `dec_full.bin` | BE32 0x1E | 0 hits at full address | — |
| `utpa2k_stock.ko` | BE32 `1E000000`/`1D000000` pattern | 0x12e40a9, 0x12e4c79, 0x12e46e9 | structured tables (`…000a00…` records), **data/reloc-like, not ARM code constructing 0xB000001E** |
| `utpa2k_stock.ko` | LE32 | 0x6360c1, 0x9cbfa1, 0xbd9eed | float/LEA streams, **data** |
| `mik_stock.ko` | BE/LE | 0x5edc11, 0x64f12e, 0x7b1fe5 | address tables / data |

### 3.2 ARM forward path after SET_IPAUTH

Capstone tail `0x424cd8–0x424d04`: final poll `#0x10`, load status word, `pop`, branch-to-lr. **No store of `0xB000001E`, no mailbox kick of a DSP gate address** appears in the dumped SET_IPAUTH tail. Kick may live in `AbsWriteByte` body or SE firmware (outside `utpa2k` `.text` dump scope for this phase).

**STATUS D.2-B: OPAQUE BOUNDARY.** Static corpus contains **zero code writers** of `0xB000001E`. The hop SE→DSP remains uncrossed without (a) AbsWriteByte/SE IDMA side-effect knowledge or (b) runtime DSP-memory observation — both out of scope for D.2.

---

## 4. D.2-C — `r4+16` / F_45FD1 gate / +0x10

**Question:** provenance of r4 at `0x4127d`, body of `F_45FD1`, identity of `*(r4+16)`, and who produces `+0x10==1`.

**Evidence:** listing window `phaseD2_followup.txt` lines 347–510; `dec32_clean.txt` L82443–82525, L89044–89104, L86815–86844; +0x10 catalog lines 512–540.

### 4.1 Caller r4 provenance — VERIFIED (path-specific)

```
0411b0 lwz r23, 0x290(r3)          ; entry r3 = base object
0411b4 mov r10, r3
...
04120e movhi r23, 0x1               ; r23 = 0x10000   (dominating def on path to 0x4127d)
041210 add  r11, r3, r23            ; r11 = base + 0x10000
041213 lwz  r12, -0x5edc(r11)
04121a lwz  r23, 0x4ee4(r11)
04121e beqi r23, 1 → 0x41275        ; only edge into 0x4127d region
041275 movhi r23, 1 ; 041277 ori r23, 0x4CCC ; 04127b add r10, r23
                                         ; r10 = base + 0x14CCC (desc root)
04127d lwz  r4, -0x5ef8(r11)        ; r4 = *(base + 0x10000 - 0x5EF8) = *(base+0xA108)
041281 mov  r3, r10                 ; desc = base+0x14CCC
041283 jal  0x45FD1                 ; F_45FD1(desc, r4)
041287 mov  r3, r10
041289 jal  0x4618E                 ; F_4618E(desc)
```

Path dominance: `0x4127d` is reachable only via `0x4121e→0x41275`; `r11` set at `0x41210`. Separate def at `0x411c3` (`r11=r3+0x10000` on other branch) is same formula but **not mixed** — this call site uses the `0x41210` def.  
**r4 provenance VERIFIED** for this instance: `r4 = *(base+0xA108)`. Object **named** at `base+0xA108` not resolved to a prior writer in bounded walk → identity of the pointed-to struct **OPAQUE BOUNDARY**.

### 4.2 F_45FD1 body — VERIFIED gate data-flow

```
045fdb movi r23, 1
045fdd sw   0x28(r3), r23           ; desc+0x28 ← 1 (default set)
045fe0 lwz  r23, 0(r4)              ; *r4
045fe3 sw   0x34(r3), r0            ; desc+0x34 ← 0
045fe6 sw   0x38(r3), r0            ; desc+0x38 ← 0
045fe9 mov  r10, r3
045feb beqi r23, 1 → 0x4600C         ; *r4==1 → success path (uses r4+0x28)
045fee lwz  r11, 0x10(r4)           ; *(r4+16)
045ff1 beqi r11, 1 → 0x46080        ; *(r4+16)==1 → alt path (uses r4+0x38)
045ff5 sw   0x28(r3), r0            ; desc+0x28 ← 0  ONLY if *r4!=1 && *(r4+16)!=1
04600c lwz  r4, 0x28(r4) ; jal 0x64149 ; …
046080 lwz  r4, 0x38(r4) ; jal 0x64149 ; sw 0x2c(r10), r11 ; …
04608e addi r11,1; slli r11,5; sw 0x38(r10), r11 ; memset r10+0x16c size 8 via 0x130d25
```

- **desc+0x28 clear condition VERIFIED:** `=0` iff `*r4!=1 && *(r4+16)!=1`.
- `*(r4+16)==1` **skips the clear** (keeps +0x28=1) and takes the `r4+0x38` path — consumer **VERIFIED** as a gate bit, **not** named license (UNPROVEN semantics).

### 4.3 `+0x10` store / readers

| kind | sites | notes |
|---|---|---|
| store | **only** `04462f sw 0x10(r11), r23` | part of fill `r23 = movhi -8; ori 0x18` → **0xFFF80018** written to `r11+0x00..0x28` — sentinel/error constant fill, **not** a `=1` flag write; base r11 ≠ r4’s object |
| readers | `044721 lwz r13,0x10(r10); bnei 1`; `044c8f … bnei 0`; `045fee lwz r11,0x10(r4); beqi 1`; `04665d lwz r23,0x10(r4)` | value `1` is expected by consumers |

**Producer of `+0x10==1` for the `r4` object: NOT FOUND** in bounded search → **OPAQUE BOUNDARY**.  
**Do not call `*(r4+16)` a license flag.** Status: gate data-flow VERIFIED; +0x10=1 producer OPAQUE; license label UNPROVEN.

---

## 5. D.2-D — r14 producer → desc+0xF0

**Evidence:** `dec32_clean.txt` L89190–89304 (full `F_4618E`); prior note `forensic_trace_autonomous_20260918_v4.md` (“r14 = N”).

```
0461a1 lwz  r23, 0x28(r3)      ; load desc+0x28
0461a4 mov  r10, r3
0461a6 movi r14, 0             ; r14 ← 0
0461a8 beqi r23, 1 → 0x461C4   ; if +0x28==1 skip store, load N
0461ab sw   0xf0(r10), r14     ; store r14 → desc+0xF0
… epilogue (addi r1,r1,0x20; jr r9)   ; path A returns after storing 0
0461c4 lwz  r14, 0x38(r3)      ; r14 ← *(desc+0x38)  = N (element count)
0461c7 lwz  r3, 0xf8(r3)       ; F8 ptr
… jal 0x130d25 memset-style with size r14<<2 on F8 then FC
… loops using r14 as trip count (ble/bgt/addi r23,r14,1 — r14 not reassigned)
046259 lwz  r23, 0x2c(r10)
04625c bnei r23, 1 → 0x461ab    ; re-enter store with r14 still N
04629c j    0x461ab             ; after sync writes, re-enter store
```

**Path A** (`+0x28!=1`): stores **0** to +0xF0 and returns — VERIFIED.  
**Path B** (`+0x28==1`): r14 loaded from `desc+0x38` (=0 from F_45FD1 init, or set at `046097 sw 0x38(r10),r11` with `r11=(prev+1)<<5`); never reassigned before later `0x461ab` stores → **r14 at store = `*(desc+0x38)` (count/size class), not a license payload**.

**STATUS D.2-D: STRONGLY SUPPORTED** (instruction-level producer `*(desc+0x38)` + store site VERIFIED; semantic role = count/size, UNPROVEN beyond that). Not DISPROVEN; not OPAQUE — the producer is found.

---

## 6. D.2-E — F8/FC allocation and fill

**Evidence:** `dec32_clean.txt` L88860–88892 (`F_45DE0`); L89190–89304 (`F_4618E` fill/emit); D.1 report §D.1-D (carry-forward).

### 6.1 Alloc — VERIFIED

```
F_45DE0:
  ori r5, 0x174 ; r4=0 ; jal 0x130d25          ; pre-clear
  mov r10, r3 ; r13=r4 (input table)
  r4=0; r5=0x130; r3=r10+0x3c; jal 0x130d25    ; memset desc+0x3C size 0x130
  lwz r3, 0(r13) ; sw 0xf8(r10), r3            ; +0xF8 ← table[0]
  r4=0; r5=0x200; jal 0x130d25                  ; memset size 0x200
  lwz r3, 4(r13) ; sw 0xfc(r10), r3            ; +0xFC ← table[1]
  r4=0; r5=0x200; jal 0x130d25                  ; memset size 0x200
  … +0xEC ← 6
```

### 6.2 Fill — VERIFIED (content path)

`F_4618E` after r14=N:
- memset-style `jal 0x130d25` with `r5 = r14<<2` on `*F8` and `*FC` (zero-fill length N*4).
- Sign-extend loop: `lwz` sample from F8/FCS, `sll/sra r12`, `sw` back (`04620a–0x4624b`).
- IEC 61937-style constants: `0x7FFE/0x8001` (`04627b/046285`), `0x1FFF/0xE800` (`0462e2/0462ed`), halfword copy from `+0x16C` family via `lhz` (`0461e5`, `046266`, `0462a8–0x462bf`).
- Guards: `desc+0x28`, `desc+0x2C`, `lhz +0x16C` select which emit path runs.

**STATUS D.2-E: VERIFIED** — allocation and fill mechanisms closed at instruction level. Buffer *content meaning* (license vs codec bitstream) **UNPROVEN** without a consumer that interprets it as license (none found that does so by name).

---

## 7. D.2-F — DM creation / stride 0x290 / base 0x1C010000

**Evidence:** `dec32_clean.txt` L28769–28783 (`0x18800` walk); `phaseD2_cdg.txt` movhi tags; prior Phase D §5.1 (data-scan literal 0x1C010000 @file 0x1D1F20; struct 0x1C012734).

### 7.1 Allocator site — VERIFIED instructions

```
018800 lwz  r12, 0(r3)            ; index / count
018803 movhi r11, imm             ; raw word c1601c01 — printed "0xe0", raw embeds 1c01
018807 muli r23, r12, 0x290       ; index * 656
01880b addi r11, r11, 0x26b0
01880f add  r11, r23              ; r11 = base + 0x26B0 + index*0x290
018811 mov  r3, r11
018813 ori  r4, r0, 0x290         ; size argument = 0x290
018817 jal  0xDFB1                ; allocation/record-create call
01881b lwz  r23, 0(r11)
01881e beqi r23, 2 → 0x187c0
018821 muli r12, r12, 0x78        ; companion stride 120
018825 movhi r23, imm ; addi r23, 0x1bc8 ; add r12, r23   ; second pool base+0x1BC8
```

### 7.2 Immediate discrepancy (decoder)

- Ghidra/`aeon_decode` print `bg.movhi rN, 0xe0` for words whose bytes contain `1c01` (e.g. `c1601c01`).
- Separate listing form `bn.movhi_2 r23, 0x1C01` exists (`0x163340` raw `06fc01` in cdg) — **1C01 is a real movhi immediate class**.
- Applying the same class of bit-field correction used for `beqi` (immediate not the low-printed field): **imm = 0x1C01** → base high = **0x1C01**, record base = **0x1C0126B0**, consistent with DM region **0x1C010000** and struct **0x1C012734** (offset 0x84 from 0x1C0126B0 is adjacent layout, not required to match exactly for stride math).
- If one instead trusted printed `0xe0`, base would be `0xE026B0` — **inconsistent** with the data-scan literal `0x1C010000` already VERIFIED in Phase D. Prefer **0x1C01** under correction; note both.

### 7.3 What is not proven

- `0xDFB1` symbol identity / boot-time-only behavior: **OPAQUE BOUNDARY** (no symbol hit in followup; runtime would confirm first-call timing).
- Full copy-table of boot stub: still open from Phase D.

**STATUS D.2-F: STRONGLY SUPPORTED** — stride 0x290 allocation sequence VERIFIED; DM base 0x1C01xxxx STRONGLY SUPPORTED (instruction + prior data literal); boot-only creation mechanism OPAQUE BOUNDARY.

---

## 8. D.2-G — SHM +0x2E0 / +0x2E4 / +0x2E6 semantics

**Evidence:** `dec32_clean.txt` L48355–48401 (`0x27982–0x27a16` full walk); D.1 writer table; D.1-C printer analysis.

### 8.1 Buffer → field data-flow — VERIFIED (instance r3)

```
027982 lwz  r24, 0x32c(r3)
027986 lwz  r23, 0x2d4(r3)       ; +0x2D4 holds buffer pointer (724)
02798a lwz  r26, 0(r3)
02798d add  r23, r24             ; r23 = ptr(+0x2D4) + ptr(+0x32c)  (base+off)
02798f lbz  r25, 0x10(r23)
…
0279a1 lbz  r25, 5(r23) ; 0279a4 lbz r27, 4(r23)
0279a7 slli r25, 8 ; 0279aa or r25, r27          ; BE u16 bytes[5:4]
0279ad sw   0x2e4(r3), r25                       ; +0x2E4 ← BE16(buf[5:4])
0279b1 lbz  r25, 0x1e(r23) ; lbz r27, 0x1d ; pack
0279bd sh   0x2e4(r3), r25                       ; halfword second write same field (path/drift)
0279e8 sw   0x2dc(r3), r26                       ; BE32 assemble bytes[0x16..0x18]
0279ec lbz  r26, 0x1a ; lbz r27, 0x19 ; pack
0279f8 sw   0x2e0(r3), r26                       ; +0x2E0 ← BE16(buf[0x1A:0x19])
027a08 sw   0x2f0(r3), r26 ; 027a0e sw 0x2e8(r3), r26
```

- Source = in-struct buffer via **`r3+0x2D4` / `r3+0x32c`**, not a global `lic[]` and not `SHM+i`.
- Two sequential stores to `+0x2E4` (word then half from different byte lanes) — path/drift ambiguity retained from D.1; mechanism STRONGLY SUPPORTED.
- Separate bases (`r10+0x2E4` at `0x27c52`, `r11+0x2E4` at `0x408d9`, clear family `r10+0x2E4/5/6` at `0xae290/94/98` with **src r0**) are **different instances** — not merged.

### 8.2 Consumers

| field | consumer | status |
|---|---|---|
| +0x2E0 | printer `0x1CA37` → `r25` → stack+80 → `jal 0x130CEA` | VERIFIED (D.1) |
| +0x2E4 | same printer → `r24` → stack+84 | VERIFIED (D.1) |
| +0x2E6 | **no load in printer**; only clear `0xae298 sbz r0` and other byte writers | `lic[3]↔+0x2E6` **DISPROVEN** for this printer (carry-forward) |
| all three as “license state” | no consumer names them license; printf format is diagnostic | **UNPROVEN** |

**STATUS D.2-G: VERIFIED** for buffer→SHM field copies and printf consumption of +0x2E0/+0x2E4; **UNPROVEN** for license semantics; **DISPROVEN** only for `lic[3]↔+0x2E6` via printer (unchanged).

---

## 9. VERIFIED graph (static, this phase + carry-forward)

```
ARM: MDrv_AUTH_IPCheck → AV+0x440/0x444
   → MDrv_AUDIO_CheckHashkey 0x423494
   → ApplyHashkey 0x424B90
   → SET_IPAUTH_GROUP 0x424BD8          [D.2-A VERIFIED byte order / payload]
        AbsWriteMaskReg [0x112AC0] 0xFFF0←0
        AbsWriteByte    [0x112A7E]←1
        poll #8/#0x10
        AbsWriteByte    [0x112A84]←dspId(0x83/0x84)
        AbsWriteByte    [0x112A85]←0x1F
        AbsWriteByte    [0x112A82]←mask[15:8] then mask[7:0] (overwrite)
        AbsWriteByte    [0x112A83]←mask[23:16] then 0     (overwrite)
   → (sibling 0x423298 / bulk 0x42D7E8 share base 0x112A82)  [VERIFIED co-tenant]

DEC combining path (instance base r3):
   0x4127d r4 ← *(base+0xA108)                     [VERIFIED]
   0x41283 F_45FD1(desc=base+0x14CCC, r4)
        desc+0x28←1; +0x34←0; +0x38←0
        *r4==1 → 0x4600C path
        *(r4+16)==1 → 0x46080 path                 [consumer VERIFIED, label UNPROVEN]
        else desc+0x28←0                           [VERIFIED clear condition]
   0x41289 F_4618E(desc)
        r14←0; if +0x28!=1: +0xF0←0; return        [VERIFIED]
        else r14←*(desc+0x38); memset F8/FC (N*4);
             fill loops; +0x16C-family sync;
             rejoin 0x461ab with r14=N             [STRONGLY SUPPORTED]

F_45DE0: +0xF8/+0xFC ← input table; memsets 0x130/0x200  [VERIFIED]

DM: muli *,0x290 ; movhi 0x1C01 (corrected) ; addi 0x26B0 ;
    jal 0xDFB1 with r4=0x290                       [STRONGLY SUPPORTED]

SHM: buffer via +0x2D4/+0x32c → BE-assemble → +0x2E0/+0x2E4  [VERIFIED]
     +0x2E0/+0x2E4 → printf varargs (0x1CA37)                 [VERIFIED, D.1]
     clear +0x2E4/5/6 ← r0 (0xae290 family)                   [VERIFIED zero]

DSP gates (4 sites): movhi 0xB000 → ori 0x1E → lh → beqi #5    [VERIFIED, D.1]
  fed by OPAQUE hop from SE                                      [D.2-B OPAQUE]
```

---

## 10. OPAQUE graph (named static boundaries)

| ID | Boundary | Why static cannot cross in D.2 |
|---|---|---|
| O1 | **SE IDMA → DSP 0xB000001E** | No code writer of the word; AbsWriteByte body / SE firmware side-effect outside decoded ARM tail; all raw hits are data |
| O2 | **AbsWriteByte FIFO vs final-state** | Hardware/SE semantics not in instruction stream |
| O3 | **Identity of `*(base+0xA108)` object** (r4’s pointee) | Bounded walk did not hit a naming writer |
| O4 | **Producer of `+0x10==1` on r4’s object** | Only found fill of 0xFFF80018 on a different base r11 |
| O5 | **`0xDFB1` boot-time identity** | No symbol; timing is runtime |
| O6 | **License *semantics* of +0x2E0/+0x2E4/F8/FC content** | No consumer labels them license; diagnostic printf ≠ license verdict |
| O7 | **Which AbsWriteByte stores the SE actually samples** | Depends on O2 |

---

## 11. Corrections / supersessions this phase

1. **Sibling@0x423298 is not a second SET_IPAUTH encoder** — it selects on codec ids 0x1F83/0x1F84 and writes uxtb(r6) / lane bytes differently; same base only. Prior “sibling pusher” wording refined: **co-tenant, not duplicate payload path**.
2. **dec LE hit 0x178d41 is DATA, not a garbled gate writer** — float table; do not re-chase as code.
3. **`0x4462f +0x10` store is a 0xFFF80018 region fill, not a flag=1 write** — cannot be cited as license-flag producer.
4. **movhi printed `0xe0` vs raw `1c01`** — prefer corrected **0x1C01** when reconciling with DM base literal 0x1C010000; record both prints (decoder discrepancy, same class as historical beqi bug).
5. **D.1 “r14 producer open” → found:** `*(desc+0x38)` / early 0. Not a license word.
6. **D.1 F8/FC “fill not found” → found:** `F_4618E` memset + sample/sync loops.
7. **Capstone WAS available** in the followup run (earlier session note that it was missing is superseded for tooling; Ghidra dumps remain authoritative for cited SET_IPAUTH lines).

Unchanged forever: stack 740(r1) exclusions; `lic[3]↔+0x2E6` DISPROVEN for printer; `0xae290` = byte-clear src r0 not “license writer”; `desc=root|0x4CCC`; gate==5; m6 2/3/1; SHM33/EDID/Kodi/0x97/0x81/N-series/T615/patch design out of scope.

---

## 12. Runtime questions (for a future authorized phase — NOT started here)

1. After SET_IPAUTH AbsWriteByte sequence, what does DSP `*(u16*)0xB000001E` become, and when (O1)?
2. Does SE IDMA present all four byte writes as a burst, or last-write-wins (O2)?
3. What structure is loaded from `base+0xA108`, and who stores `1` to its `+0x10` (O3/O4)?
4. On first boot, is `0xDFB1` the sole creator of 0x1C01xxxx records; stride/size observed live (O5)?
5. Do live `+0x2E0/+0x2E4` bytes correlate with an external license oracle, or only with the diagnostic `lic[]` log (O6)?
6. F8/FC pointed buffers: hexdump at emit time vs IEC 61937 sync — confirm fill path (D.2-E) on device.

---

## 13. DO-NOT list (carry-forward + D.2 additions)

**Carry-forward DO-NOT:** SHM33, EDID, Kodi, 0x97, 0x81, N-series, T615, patch design, /vendor inertness re-proof, full-corpus sweeps, +0x10C verdict as config-apply load, “0xae290 = license writer”, `lic[i]=SHM+i`, `desc+0x28=0` as license gate without `*(r4+16)` identity, 740(r1) as field writes, any device/ADB/flash in this phase, re-opening `lic[3]↔+0x2E6` for printer 0x1CA37, assuming mask byte order without ubfx/uxtb evidence, mixing disp+0x2E4 across bases, decoding ARM with aeon_decode as sole tool, full-image decode sweeps (300s timeout), linear AEON walk without phase anchors, calling `*(r4+16)` a license flag, citing `0x4462f` as =1 producer, citing 0x178d41 as code.

**D.2-H DO-NOT:** web research, new firmware extraction, patch, runtime, broad reread, re-checking DO-NOT dead ends — until user explicitly authorizes a runtime phase.

---

## Appendix — artifact index (this phase)

| Path | Role |
|---|---|
| `REPORT_phaseD2_final_static_residual_20260922.md` | this deliverable |
| `C:\Users\k0994\AppData\Local\Temp\opencode\phaseD2_followup.txt` | Capstone + hit contexts + listing windows |
| `…\phaseD2_followup.py` | script (Capstone available at run time) |
| `…\phaseD2_cdg.txt` | AEON D.2-C/D/E/F/G walks |
| `…\phaseD2_b_bridge.txt` / `.py` | literal bridge scans |
| `spdif_audio_investigation\tools\dd_MDrv_AUDIO_ApplyHashkey.txt` | authoritative SET_IPAUTH dump |
| `spdif_audio_investigation\tools\dd_ApplyHashkey.txt` | ApplyHashkey dump |
| `spdif_audio_investigation\tools\closure_utpa2k.txt` | AbsWriteByte xrefs L183–214 |
| `dec_work\dec32_clean.txt` | AEON listing (oracle for all AEON cites) |
| `dec_work\REPORT_phaseD_static_closure_20260922.md` | Phase D |
| `dec_work\REPORT_phaseD1_residual_static_20260922.md` | Phase D.1 |
| `patch_baseline\utpa2k_stock.ko` (2fc6e9fc), `mik_stock.ko` | D.2-B sources |
| `dec_work\dec_full.bin`, `r27_work\snd_full.bin` | D.2-B targets |
| `aeon_validate\scripts\aeon_decode.py` | AEON only |
| `aeon_ghidra_public\aeon\data\languages\aeon_ORBIS32.sinc` | beqi line 486; movhi layout |

**End of Phase D.2.** Static exhausted for D.2-A…G as posed; runtime justified for O1–O7 only under separate explicit authorization. No runtime entered.
