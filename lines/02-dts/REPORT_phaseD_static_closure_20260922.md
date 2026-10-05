# REPORT — Phase D Static Closure: ARM AUTH → DSP License Transport → DSP License State → DTS Output Producer

**Date:** 2026-09-22
**Scope:** Static-only (read-only). No device/ADB contact. No flash/patch/load. All new findings only.
**Success criteria:** D-A … D-E (below).

---

## 1. Executive conclusion

The static causal chain from ARM-side IPAuth (AV box +0x440/0x444 → CheckHashkey → ApplyHashkey → SET_IPAUTH_GROUP → AbsWriteByte to SE IDMA 0x112A82–85) through the DSP-side DEC gate (`*(u16*)0xB000001E == 5` at 4 sites) to DTS output production is now closed as a static corpus, with one unresolved hop (the SE → DSP mailbox content of 0x112A82/83) and one boot-time hop (DM record creation) still open for runtime confirmation.

Key confirmations this session:
- **Gate immediate is 5.** Ghidra `dec06_out.txt` proves `bg.beqi r26/r27, 0x5` at all 3 checkable gate sites. Our branch targets matched Ghidra exactly; only `aeon_decode`'s printed immediate was wrong. True immediate = bits 16–20 of the instruction word (`i32_uimm16_5 = (16, 20)` / `#uimm16_5 = (16, 5)` in `aeon_ORBIS32.sinc:486`).
- **m6-init verdict switch is on 2/3/1, not 3410/3250/1178.** Applying the same bit-field fix to the `bg.beqi` instructions at 0x1e3fb/0x1e3ff/0x1e403/0x1e44f/0x1e453/0x1e457 yields immediates 2, 3, 1, 2, 3, 1 — VERIFIED instruction-level corroboration of the prior "verdict class == 2" claim.
- **`lic[%d,%d,%d]` format string located at 0x1427CE** (`arm->r2 %d: state:%x... lic[%d,%d,%d], `), and the diagnostic printer at 0x1c93e/0x1ca37 loads **+0x2E0 and +0x2E4** into varargs before `jal 0x130CEA` (printf). This is the A→C bridge: the SHM38 license fields are passed into the DSP diagnostic log that ends in `lic[]`.
- **Writers of +0x2E4/+0x2E6 EXIST** (contradicting the earlier "writer never found"): a regular `bg.sbz` byte-store family at 0xae288–0xae298 stores r0 to +0x2D3, +0x2EA, +0x2E4, +0x2E5, +0x2E6; plus word/half stores at 0x21cda, 0x279ad/bd, 0x27c52/56, 0x408e5, 0x72531, 0xce66b, 0x129128.
- **`+0x10C` is not a config-apply instruction displacement** — 0 hits with disp 0x10C in 0x1E000–0x24000. Prior's "+0x10C verdict" claim is reclassified as a static-record-layout coordinate (data-scan methodology), not a decoded load. Verdict class field position within the record at instruction level remains UNPROVEN; verdict value not static in blob is VERIFIED (0 multi-2 candidates at +0x10C with stride 0x290).

---

## 2. Input artifacts and hashes

| Artifact | Identity | md5[:8] | Role |
|---|---|---|---|
| `utpa2k_stock.ko` | 25381336 B | 2fc6e9fc | D1 primary (`.text` off 0xd654, size 0x6287dc) |
| `mik_stock.ko` | | c1421040 | D7 variant |
| `utpa2k_installed.ko` | | f44ad0a4 | D7 installed |
| `snd_full.bin` | 0x1c1330 | | D2 target (0xB544A inside) |
| `dec_full.bin` | 0x1e401c | | D3–D6 target |
| `dec_work\MASTER_FORENSIC_REPORT_20260919.md` | | | D6 source (lines 44/106/108/163/279) |
| `REPORT_GLM53_full_audit_and_dts_repair.md` | | | D3 prior claims (lines 156–158/199–200/212–213/345/396–398/491) |
| `FINDING_codec5_vs_9_host_differential.md` | | | DM base 0x1C010000, struct at 0x1C012734 |
| `BACKUP_20260914_0717\md5_manifest.txt` | | | D7 source (4 aucode bins) |
| Scripts (all RAN) | `phaseD_d1_part5.py`, `d2.py`, `d3.py`, `d4.py`, `d4b.py`, `d5.py`, `d6.py`, `d7.py`, `d3d5_close.py`, `d4d5_writer.py` | | this session |
| Outputs | `phaseD\*.txt` (`d1_part5`, `d2_disasm`, `d3_boot_dm`, `d4_gate_2e6`, `d4b_more`, `d5_lic`, `d6_b1b2`, `d7_variants`, `d3d5_close`, `d4d5_writer`) | | this session |

Decoder stack: `aeon_decode.py` (`decode_at(data, addr, base=0)` → `(mnem, ops, len, bytes)`; AEON words BIG-ENDIAN); Ghidra `aeon_ORBIS32.sinc` / `aeon.slaspec`; `dec06_out.txt` for gate byte truth.

---

## 3. D1 — ARM AUTH chain closure

**CLAIM:** `MDrv_AUTH_IPCheck` ids → AV+0x440/0x444 → `MDrv_AUDIO_CheckHashkey 0x423494` → `ApplyHashkey 0x424B90` → `SET_IPAUTH_GROUP 0x424BD8` → `AbsWriteByte` into SE IDMA 0x112A82–85.
**EVIDENCE:** `phaseD_d1_part5.py` → `d1_part5.txt`.
**STATUS: VERIFIED.**

Sequence (byte-verified):
1. `SET_IPAUTH_GROUP`: 0x112AC0 mask 0xFFF0,0 → 0x112A7E,1,1 → poll 0x112A80 ≤0xC8 iters → 0x112A84=0x83/0x84 → 0x112A85=0x1F → 0x112A82/83.
2. Sibling pusher @0x423298 (codec IDs 0x1F83/0x1F84, site 0x423310; bulk 0x42d7e8).
3. ApplyHashkey callers: 0x3e24a4, 0x405b58.
4. AbsWriteByte physical = addr − 0x100000.
5. IPCheck id→bit map: 0x11→bit0, 0x83→bit18, 0x84→bit7.
6. mik/DEC/SND have NO 0x112Axx refs.

**Open (sub-D1):** garbled `strb` decodes @0x424c60–0x424c90 leave exact 0x112A82/83 payload bytes ambiguous → **UNPROVEN** (receiver ID needs D3 boot/DM mapping or runtime).

---

## 4. D2 — SND composer / C-Path byte verification

**CLAIM:** 0x1F5FD `bg.jal 0xB544A` (r9=0x1F605); 0x1F67B `bg.jal 0xB6D8F` (r9=0x1F683); 0x1F67F `bg.andi r11,r11,0x200`; 0x1F685 `bn.beqi r11,0→0x1F6DA`; 0x1F6DA `bg.bnei r14,1294,0xA1→0x1F77B`; 0x1F44F = `bn.addi r1,r,r,-48` (composer entry).
**EVIDENCE:** `phaseD_d2.py` v2 → `d2_disasm.txt`.
**STATUS: VERIFIED** (all byte-checked). 0x1F683 remains an encoding gap (`89c32161` fails both width gates) → decoder limitation, not a corpus contradiction.

---

## 5. D3 — Boot stub + config-apply + verdict layout

### 5.1 Stride 0x290 and DM base

**CLAIM:** DM records stride 0x290, base 0x1C010000.
**STATUS: STRONGLY SUPPORTED.** 30 `bg.muli …,656 (0x290)` sites including 0x22e75 (config-apply) and **0x1c949** (diagnostic printer). DM base literal 0x1C010000 @file 0x1D1F20. Struct at 0x1C012734 per `FINDING_codec5_vs_9_host_differential.md`.

### 5.2 config-apply walk 0x22DB0–0x22ED8

**CLAIM (prior):** verdict class at +0x10C is read by config-apply.
**STATUS: SUPERSEDED (instruction level).** Full walk (`d3d5_close.txt`):
- `0x22db0 muli r25,r3,120`; `movhi r12,0xE0; addi r12,7112` → staging 0xE01BC8.
- HASH reads via r23 ori 0x838/0x81C/0x83C (match data fields 0x0001081C/838/83C).
- lhz disps 522/526/528/530 (0x20A–0x212); lwz disps 4/8/12/**132(0x84 licensee)**/536/540/544/572/612/616.
- Stores at end: `0x22ec8 sw r30,288(r10)` (=+0x120) where r30 = r3·0x290 (**stride·idx, NOT verdict**), `0x22ecc sw r29,292(r10)` (+0x124), +0x128, +0x12C.
- **disp == 0x10C instruction hits in 0x1E000–0x24000: 0.**
- Static word==2 at X+0x10C stride 0x290 BE multi-2 candidates: **0** → verdict NOT static in blob → runtime-created (matches prior "ARM/loader-delivered").

Reclassification: prior's "+0x10C" is a **static-record-layout coordinate from data-scan methodology**, not a decoded load immediate. Field position at instruction level: **UNPROVEN**. Verdict not static: **VERIFIED**.

### 5.3 Boot stub 0x0–0x220

**CLAIM (prior, fragmentary):** r6=0xD0000, r3=0x00010000; local window 0x00010000+N ↔ blob 0xD0000+N.
**STATUS: STRONGLY SUPPORTED (prior) / full copy-table UNPROVEN this session.**
Structure confirmed by hexdump (`d3d5_close.txt`):
- 0x000–0x0FF: zeros (padding/exception vectors).
- 0x100–0x17F: SPR init (`ori r7,0x4800/0x4802`; `mtspr`) + address-table-like constants (`001ca0 001cc0 001ce0 …` ascending) + `c060 0001 c863 d000 8469` at 0x174–0x17F (**only D000 byte-pattern in region**).
- 0x180–0x1FF: zeros.
- 0x200+: stack entry `bg.sw r1,132(r0); movhi r1,0x4B; ori r1,r1,0x8450; addi r1,r1,-300` (stack at 0x4B8450−300) then `mfspr` then `jal 0x40255F` (walk has gap at 0x218 before it — target 0x40255F is plausible image entry but phase-drift unconfirmed).

Full DATA copy-table extraction (aligned decode 0x100–0x180) does not yield a clean `movhi`+`addi` pair for 0xD0000/0x10000 — remains **UNPROVEN** pending either a better boot decode window or runtime.

### 5.4 Verdict-class static presence

Static word==2 at X+0x10C stride 0x290: 0 BE multi-2 candidates → **VERIFIED not static.**

---

## 6. D4 — DEC gate and +0x2E6 / +0x2E4 readers/writers

### 6.1 Gate immediate

**CLAIM:** `*(u16*)0xB000001E == 5`.
**STATUS: VERIFIED (instruction level).**
- Sites: 0x21AC3, 0x21B19, 0x21F0E, 0x21F5C.
- Ghidra `dec06_out.txt`: `021AC3 bg.beqi r26,0x5 [D34500B2]→0x00021ad9`; `021B19 bg.beqi r27,0x5 [D3650092]→0x00021b2b`; `021F0E bg.beqi r26,0x5 [D34500B2]→0x00021f24`.
- Our branch targets matched Ghidra exactly; only printed immediate wrong.
- Bit-field: `(word >> 16) & 0x1F` → 0xD345&0x1F=5. Sinc form `:bg.beqi i32_rD,i32_uimm16_5,i32_rel_simm3_13` at `aeon_ORBIS32.sinc:486`.
- Fail path: read 0xB000001D `andi 0x7f` → 0x22031 error; print jal 0x130CEA @0x21FE2 with r3=0x144904 (`Invalid Spatif license:%d, output_spdifSz:%d`).

**DECODER DEFECT (documented):** `aeon_decode.py` prints low16 as `bg.beqi` immediate (178/3410/etc). Real imm = bits 16–20. Branch targets from decoder are correct.

### 6.2 m6-init verdict switch (same defect, now fixed)

Raw at sites (`d4d5_writer.txt`):
| Addr | raw | decoded imm (bug) | true imm `(w>>16)&0x1F` |
|---|---|---|---|
| 0x1e3fb | d2e20d52 | 3410 | **2** |
| 0x1e3ff | d2e30cb2 | 3250 | **3** |
| 0x1e403 | d2e1049a | 1178 | **1** |
| 0x1e44f | d2e205fa | 1530 | **2** |
| 0x1e453 | d2e30992 | 2450 | **3** |
| 0x1e457 | d2e108f2 | 2290 | **1** |

→ verdict class comparison **==2/3/1 VERIFIED**. Prior "verdict=2" claim corroborated at instruction level.

### 6.3 +0x2E6 / +0x2E4 readers (prior 5 sites)

Byte-verified (`d4d5_writer.txt` exact decodes):
- 0x27a73: `bg.lhz r7,742(r10)`
- 0x280ce: `bg.lhz r14,742(r11)`
- 0x2830b: `bg.lhz r28,742(r10)`
- 0xafc86: `bg.lbz r23,742(r10)`
- 0x157327: `bg.lhs r31,742(r27)` (entry walk drifts — reader only)

**STATUS: VERIFIED.**

### 6.4 +0x2E4 / +0x2E6 writers — NEW (prior said never found)

**CLAIM (prior):** writer for +0x2E4 never found.
**STATUS: DISPROVEN / UPDATED — writers found.**

Bounded store scan 0x20000–0x160000 for disp 740/742 (`d4d5_writer.txt`):
```
21cda: edcb02e4 bg.sw  r14,740(r11)
279ad: ef2302e5 bg.sw  r25,740(r3)
279bd: ef2302e7 bg.sh  r25,740(r3)     ; halfword store to +0x2E4
27c52: ef8a02e5 bg.sw  r28,740(r10)
27c56: ef6a02e7 bg.sh  r27,740(r10)    ; halfword store to +0x2E4
408e5: ec0b02e4 bg.sw  r0,740(r11)     ; clear +0x2E4
72531: eef202e4 bg.sw  r23,740(r18)
ce66b: ee2102e4 bg.sw  r17,740(r1)
129128: ee2102e4 bg.sw r17,740(r1)
```

Plus a regular **byte-clear family** at 0xae288–0xae298 (raw bytes match at aligned decode):
```
ae288: f80a02d3 bg.sbz 723(r10),r0    ; +0x2D3
ae28c: f80a02ea bg.sbz 746(r10),r0    ; +0x2EA
ae290: f80a02e4 bg.sbz 740(r10),r0    ; +0x2E4
ae294: f80a02e5 bg.sbz 741(r10),r0    ; +0x2E5
ae298: f80a02e6 bg.sbz 742(r10),r0    ; +0x2E6
```
Three consecutive byte stores r0 → +0x2E4/+0x2E5/+0x2E6 = clear of the 3-byte license field. Anchored walk from prologue 0xadeba (`bn.addi r1,r,-32`) drifts into `bg.op2E` stream — function identity of the clearer still **UNPROVEN**, but raw byte patterns are regular and intentional-looking.

**STATUS: STRONGLY SUPPORTED** (writers exist; exact function context partially drifted).

---

## 7. D5 — `lic[]` format-string xrefs and +0x2E4 → printf bridge

### 7.1 Strings

| String | VA | Notes |
|---|---|---|
| `arm->r2 %d: state:%x(chg:%d,req=%d), … lic[%d,%d,%d], ` | **0x1427CE** | full diagnostic format; starts mid-run after `, fifo:%05x, ` @0x1427C0 |
| `lic[%d,%d,%d]` | 0x142892 | inside the above |
| `ARM_AUDIO_INIT` | 0x1428A2 | |
| `Invalid Spatif license:%d, output_spdifSz:%d` | 0x144904 | |
| `output_spdifSz` | 0x144843 / 0x14491E | |
| `dd_licensee` | 0x145089 | |
| `licensee` | 0x14508C+ | |
| `dts m6` | 0x146A8A+ | |

### 7.2 Whole-image `addi` imm hunt (step-decode, `d4d5_writer.txt`)

Only 3 positive-imm leas found to target strings:
- `0x21fde: bg.addi r3,r3,18692` → **0x144904** Invalid Spatif (single xref image-wide — VERIFIED)
- `0x2958c / 0x295ae: bg.addi r3,r3,27274` → **0x146A8A** dts m6
- imm 10352 (0x142870), 10386 (0x142892), 18563, 18718, 20617, 20620: **0 hits** (those substrings are only reached via the 0x1427CE base + interior format parsing, not dedicated leas)

`0x1c97e: bg.addi r3,r3,10087` → 0x142767 (related interior point; leas at 0x1d9c6/ca, 0x1dacd/d9 also → 0x142767).

### 7.3 The `lic[]` printer — A→C bridge

Function **0x1c93e** (`d4d5_writer.txt` walk):
```
1c93e: bn.addi r1,r,-120
1c941: bg.movhi r23,0x46
1c945: bg.muli r25,r3,120
1c949: bg.muli r26,r3,656          ; stride 0x290 AGAIN (near print entry)
1c94d: bg.muli r24,r3,28
1c958: bg.muli r23,r3,848
...
1c9b6: bg.lwz r30,128(r10)         ; +0x80
1c9ba: bg.lwz r29,132(r10)         ; +0x84 licensee
1c9be: bg.lwz r28,136(r10)         ; +0x88
1c9c2: bg.lwz r27,120(r10)         ; +0x78
...
```

At the printf site **0x1ca12–0x1ca37** (prior + current walk):
```
1ca12: bg.lwz r25,736(r13)         ; SHM +0x2E0 → r25
1ca16: bg.lwz r24,740(r13)         ; SHM +0x2E4 → r24
1ca1c: bg.addi r3,r3,10190         ; r3 = 0x140000+10190 = 0x1427CE (format), r3 held from earlier movhi 0x14
1ca28..1ca34: save r28,r27,r26,r25,r24 to stack (varargs)
1ca37: bg.jal 0x130CEA             ; printf
```

**CLAIM:** +0x2E4 (and +0x2E0) are passed as printf varargs into the `arm->r2 … lic[%d,%d,%d]` diagnostic.
**EVIDENCE:** phase-anchored walks in `d4d5_close.txt` + `d4d5_writer.txt`; string body at 0x1427CE.
**STATUS: STRONGLY SUPPORTED.** The `lic[]` slots in the log are fed from SHM38 +0x2E0/+0x2E4 (and likely a third slot — exact 3rd register not cleanly isolated this session).

`lic[3]` ↔ +0x2E6 exact mapping: **UNPROVEN** (third vararg not cleanly decoded; +0x2E6 is read as halfword at 5 sites and cleared as byte at 0xae298).

Other D5 notes:
- 0x12921E = stack epilogue word-restore (lwz 736..764(r1), `addi r1,r1,776`) — NOT lic[] array (prior misread corrected).
- "Invalid Spatif" NOT a contiguous standalone string beyond 0x144904 line.
- 0x144904 print tail r3 VERIFIED @0x21FE2.

---

## 8. D6 — B1/B2 prepare/emit verdict

**Naming map (prior reports do not use B1/B2 labels):**
- **B1 = prepare/parser = `F_45FD1`** (MASTER_FORENSIC_REPORT_20260919.md:106)
- **B2 = emit/output = `F_04618E`** (line 108)

**CLAIM:** 0x45FD1 prepares/decodes input into the descriptor; 0x4618E emits output; call structure `0x41283→0x45FD1`, `0x41289→0x4618E`, r12-gate `0x4128D→0x411C9`, desc at `root+0x14CCC`, F0=0 @0x461AB, N=ctx+0x38, bit-reader `0x64369` @0x46210/0x4621E, upstream `0x3EF64`.
**EVIDENCE:** MASTER_FORENSIC_REPORT_20260919.md:44,106,108,163,279; `.workbuddy-ai\memory\2026-09-12.md:393-411`, `2026-09-13b.md:6`; `phaseD_d6.py` v2 → `d6_b1b2.txt` (drifted decode of 0x45f91–0x4618e; `0x45ff1 bg.beqi r11,…→0x46080` — immediate display-buggy, true imm = `(w>>16)&0x1F`).
**STATUS: STRONGLY SUPPORTED** (prior report chain + targeted window runs; full clean decode of both bodies not achieved due to linear-sweep drift).

Unresolved sub-questions (desc fields +0x28/+0xF0/+0xF8/+0xFC/+0x16C): **UNPROVEN** this session — left as runtime/measurement items.

---

## 9. D7 — Variant hash table and known-site diff

From `phaseD_d7.py` → `d7_variants.txt` (hashes verified):

| Variant | md5[:8] | Known sites / notes |
|---|---|---|
| STOCK utpa2k | **2fc6e9fc** | baseline; hashkey stock (efe6bf, ffffff) |
| STOCK mik | **c1421040** | |
| K-1 | **58d9e8d0** | ARM "17 B" |
| K-2 | **5d2f2652** | |
| P-BYP | **2e251c98** | 0x44524C bne→NOP, 0x44D1DC, 0x446A78; mik 0x977d4/0x9A2C8 |
| P-BYP2/3/4/6 | **f8513553** | derived |
| installed utpa2k | **f44ad0a4** | |

md5_manifest aucode bins: adec `51392f23`, adec_MS12V22 `2a279c8b`, asnd `42c1cb79`, asnd_MS12V22 `b2a7e246`.

**STATUS: VERIFIED** (hashes recomputed this session). K-1 loader truth unchanged: `HAL_AUDSP_DspLoadCode 0x46f5a8` memcpy's embedded arrays from utpa2k.ko `.data`; `/vendor/.../audio_bin/*.bin` DORMANT → all /vendor DEC patches **INERT-SURFACE**.

No dedicated K-2/P-BYP build scripts found in corpus (gap noted, not blocking).

---

## 10. Consolidated causal graph (Phase D additions only)

```
[ARM] IPCheck ids → AV+0x440/0x444
        → CheckHashkey 0x423494 → ApplyHashkey 0x424B90
        → SET_IPAUTH_GROUP 0x424BD8 → AbsWriteByte → SE IDMA 0x112A82/83   [D1 VERIFIED]
              │
              │  (hop UNPROVEN: SE mailbox → DSP view of 0xB000001E)
              ▼
[DSP] boot stub 0x0–0x220 (stack @0x4B81AC, SPR init, D000/10000 window)
        → DM records @0x1C010000+idx·0x290 (runtime-created, not static)   [D3 STRONGLY SUPPORTED / copy-table UNPROVEN]
              │
              ▼
        DEC gate  *(u16*)0xB000001E == 5   @0x21AC3/21B19/21F0E/21F5C      [D4 VERIFIED]
              │ fail → 0xB000001D&0x7f → 0x22031 error
              │ pass
              ▼
        m6-init  bg.beqi …, 2 / 3 / 1      @0x1e3fb/ff/03/4f/53/57         [D4 VERIFIED]
        config-apply 0x22DB0–0x22ED8 (HASH→staging 0xE01BC8→DM; stores +0x120=idx·0x290, +0x124..+0x12C)
              │  licensee @DM+0x84
              ▼
        SHM38 / stack  +0x2E0, +0x2E4, +0x2E6  (read @5 sites; cleared @0xae290 family;
                                                 written @0x21cda/279ad/bd/27c52/56/408e5/…)
              │
              ▼
        printf 0x1ca37 → format 0x1427CE "… lic[%d,%d,%d]"  (+0x2E0/+0x2E4 as varargs)
        print 0x21FE2 → format 0x144904 "Invalid Spatif license:%d, output_spdifSz:%d"
              │
              ▼
[D6] B1=F_45FD1 prepare → desc root+0x14CCC → B2=F_04618E emit
        (callers 0x41283/0x41289, gate 0x4128D→0x411C9, upstream 0x3EF64)  [STRONGLY SUPPORTED]
              │
              ▼
[DTS/DTS-passthrough output]  (runtime observable)
```

Permanent closures carried forward unchanged: Phase A/B/C graph E1–E13; K-1 loader truth; all /vendor DEC patches INERT-SURFACE.

---

## 11. Exactly VERIFIED (this session or prior, re-checked)

1. ARM AUTH chain D1 (byte-verified sequence + id→bit map + AbsWriteByte phys formula).
2. SND composer/C-Path byte hits D2 (except encoding-gap 0x1F683).
3. Gate `== 5` at 0x21AC3/0x21B19/0x21F0E (Ghidra bytes + sinc field + matching branch targets).
4. `bg.beqi` true immediate = bits 16–20 (decoder defect root-caused).
5. m6-init verdict switch immediates 2/3/1.
6. +0x2E6 readers at 5 sites; +0x2E4/+0x2E6 writers exist (store family).
7. `lic[]` format at 0x1427CE; Invalid Spatif single xref 0x21FDE→0x144904; dts m6 leas 0x2958C/0x295AE.
8. +0x2E0/+0x2E4 loaded into varargs before printf @0x1CA37.
9. Stride 0x290 at 30+ sites including diagnostic printer 0x1C949; DM base literal 0x1C010000.
10. Verdict class NOT static in blob (0 multi-2 candidates).
11. config-apply full walk structure (HASH reads, staging 0xE01BC8, stores +0x120/+0x124/+0x128/+0x12C, licensee +0x84).
12. D7 variant hashes (all 7 + 4 aucode bins).
13. K-1 loader truth + INERT-SURFACE on /vendor DEC patches.

---

## 12. Exactly UNPROVEN / SUPERSEDED / OPEN

| Item | Status |
|---|---|
| Exact bytes written to 0x112A82/83 (garbled strb @0x424c60–0x424c90) | UNPROVEN |
| SE mailbox → DSP `0xB000001E` delivery hop | UNPROVEN (needs runtime) |
| Boot-stub full DATA copy-table (0xD0000/0x10000 window) | UNPROVEN (prior STRONGLY SUPPORTED, not re-extracted cleanly) |
| config-apply "+0x10C verdict read" as instruction | **SUPERSEDED** — 0 hits; reclassified as static-layout coordinate |
| Verdict-class field byte-offset within record at instruction level | UNPROVEN |
| Writer function identity at 0xae290 family (entry 0xadeba drifts) | UNPROVEN (writers themselves STRONGLY SUPPORTED) |
| `lic[3]` ↔ +0x2E6 exact mapping (3rd printf vararg) | UNPROVEN |
| 0x1F683 SND instruction (encoding gap 89c32161) | decoder limitation |
| D6 desc fields +0x28/+0xF0/+0xF8/+0xFC/+0x16C | UNPROVEN |
| ApplyHashkey payload disambiguation between two strb sequences | UNPROVEN |
| Runtime values (AV+0x4D8/0x440/0x444, gIpAuthVars, +0x2E6 SHM38) | deferred (static-only rules) |
| `func_at` STT_FUNC filter returns giant `mst_codec_r2_MS12V22` for all addrs | tooling defect — use known symbols |

Tooling constraints permanently in force: bounded region scans only (full-image decode sweeps time out at 300 s); AEON big-endian; no `struct.unpack('<I')`; linear sweeps drift — use phase-anchored walk-back to `bg/bn.addi r1,r1,-N` prologues; avoid glob storms and whole-tree grep on 65536-byte records.

Unread prior reports (cite as limitation if skipped): `REPORT_DTS_phase3_final.md`, `REPORT_SPDIF_audio_path.md`, `REPORT_ms12v22_profile.md`, `REPORT_kodi22_minimal_patch.md`.

---

## 13. Runtime measurements still necessary (NOT authorized this phase)

Do **not** execute until Phase D report accepted and a separate runtime phase is explicitly authorized.

1. Read `*(u16*)0xB000001E` and `*(u8*)0xB000001D` live at each gate site under passthrough-on vs off.
2. Read SE IDMA 0x112A82–85 after SET_IPAUTH_GROUP to capture exact payload bytes.
3. Dump DM records at 0x1C010000+idx·0x290 (esp. +0x84 licensee, +0x10C-or-actual verdict field, +0x120/+0x124) with idx=0..N.
4. Dump SHM38 stack/heap +0x2E0/+0x2E4/+0x2E6 at the printf 0x1CA37 breakpoint / sample point.
5. Capture the `arm->r2 … lic[]` log line and `Invalid Spatif license` line during a passthrough session.
6. Read AV+0x440/0x444/0x4D8 and gIpAuthVars after IPCheck.
7. Single-step boot stub to confirm 0xD0000/0x10000 window and record creation timing.
8. D6 descriptor fields +0x28/+0xF0/+0xF8/+0xFC/+0x16C and N=ctx+0x38 bit-reader state during B1→B2.
9. Break on the 0xae290 byte-clear family to identify calling function.
10. Compare live `lic[]` values against dumped +0x2E0/+0x2E4/+0x2E6 to prove the mapping.

---

## Success criteria

| ID | Criterion | Result |
|---|---|---|
| **D-A** | ARM AUTH → SE IDMA chain statically reconstructed with byte evidence | **MET** (D1 VERIFIED; payload bytes still open sub-item) |
| **D-B** | DSP license state (DM stride/base, gate==5, verdict switch==2, SHM +0x2E0/4/6) statically bounded | **MET** (gate/verdict VERIFIED; DM base/stride STRONGLY SUPPORTED; writers/readers found; boot copy-table open) |
| **D-C** | DSP-visible license state → DTS output producer path identified | **MET as static graph** (printf bridge STRONGLY SUPPORTED; B1/B2 STRONGLY SUPPORTED from prior; runtime confirmation deferred) |
| **D-D** | Variant/build ledger complete and hash-verified | **MET** (D7 VERIFIED) |
| **D-E** | Permanently-closed hypotheses preserved; new claims labeled with status vocabulary | **MET** (this report §11–§12) |

**Phase D verdict:** Static closure achieved for the requested chain at the corpus level. The two hops that require silicon (SE payload bytes; 0xB000001E live value / boot record creation) and the exact `lic[3]`↔+0x2E6 mapping are explicitly deferred to an authorized runtime phase. No Phase D action modified any prior artifact.

---
*End of report.*
