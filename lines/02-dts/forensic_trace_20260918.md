# Forensic trace — ctx → output buffers → downstream (read-only static)
Date: 2026-09-18. Sources: dec32_clean.txt (byte-matched dec_full.bin), aeon_ORBIS32.sinc, utpa2k_stock.ko (.data embedded images).

## 1. Geometry branch 0x460C2 — CORRECTED polarity
- Listing `dec32_clean.txt:89121`: `0460c2 bg.ble r25,r26,0x000460f3 | d73a0189`
- Predecessors `89118-89120`: `r24=ctx+0x08 (L)`, `r23=ctx+0x38 (N)`, `r26=r24+0x80`, `r25=r23<<5`.
- Decoder `aeon_ORBIS32.sinc:514-515` (`Verified on hw`, opcode 0x35): `bg.ble rD,rA,tgt = if (rD <= rA unsigned) goto tgt`.
- VERIFIED single polarity: taken to `0x460F3` iff `uint32(N<<5) <= uint32(L+0x80)`. Fall-through to `0x460C6` builder-dispatch checks iff `>`.
- Previous `<= пропускає builder` was inverted. Do NOT name it capacity/overflow/DTS/PCM gate.
- Adjacent gate `0x460B1 bnei r3,0,0x4614C` (r3 = ctx+0x24 via helper 0x64354) is the true builder-bypass: `!=0 → skip N/L gate + all builder calls → 0x460F3` via `0x4614C: jal 0x6434F`.
- Output loop guard `0x461FD bg.ble r14,r16,0x46259` (r16=0): `N<=0 → skip loop`. Not a DTS gate.

## 2. Helpers (VERIFIED addresses)
- `0x64354:130304 = lwz r3,0x24(r3); jr` — returns ctx+0x24.
- `0x6434F:130302 = sw 0x24(r3),r0; jr` — clears ctx+0x24. Called at `0x4614E` (bypass path).
- `0x64149:130121-130140` — 0x28-byte memcpy `*r3 = *r4` (0x00..0x24).
- `0x64369:130312+` — bit-reader-shaped: reads ctx+0x08/0x00/0x04, writes word to `(r5)`. LIKELY payload writer; exact bit semantics BLOCKED (decoder gaps, indirect logic).
- `0x130D25:410711+` — called with `r4=0, r5=len, r3=dest` from output (`0x461D2/0x461DE`) and parser. Zero-fill-shaped (writes r4 to 0x00..0x1C loop) but contains `opcode_2B`, muls, branches — semantics NOT certified.

## 3. Provenance (local chain only; whole-image 0x24=1234 hits, 0x28=1082 hits → no global uniqueness claim)
- ctx+0x24: no write in `[0x45FD1,0x4618E)` except clear via 0x6434F on bypass path. Read via 0x64354 at `0x460AA → sw 0x30`. Value arrives with ctx. BLOCKED at function-input boundary.
- ctx+0x28: `0x45FDD sw 0x28(r3),1` init; `0x45FF5 sw 0x28(r3),0` on invalid-input branch (falls to `0x46000` early-return). Readers: `0x4600C (input+0x28 load, different object), 0x46016/0x46033/0x46079/0x461A8 (ctx+0x28==1 gates)`. VERIFIED local lifecycle.
- ctx+0x30: writer `0x460AE sw 0x30(r10),r3` (r3=ctx+0x24). Readers `0x461E2/0x46260`. VERIFIED derived.
- ctx+0x34: writer `0x46105 sw 0x34(r10),r3` (r3 from `0x642DC`). No reader in `[0x45FD1,0x4661E)`. Reader BLOCKED.
- ctx+0x38 (N): clear `0x45FE6 sw 0x38,0`; set `0x46097 sw 0x38,r11` where `r11=(parsed+1)<<5` (`0x4608E/090`). Readers `0x460B8 (L gate), 0x461C4 (output N), 0x461FD loop bound`. VERIFIED.
- ctx+0xF8/FC (A/B): writers `0x45E17 sw_0 0xF8(r10),r3 | ec6a00f8`, `0x45E2F sw_0 0xFC(r10),r3 | ec6a00fc` in func `0x45DE0` region (ctx = r10/r11 there). Readers: output `0x461C7/0x461D6/0x46201/0x46214/0x46222/0x46226/0x4626A/0x462A0/0x462DE` + parser-helper `0x45E52: 0x45EFB/0x45EFF`. Instance identity across functions NOT proven (different r10/r11 bases) → LIKELY same struct family, BLOCKED same-instance.
- ctx+0xF0: only write in `[0x4618E,0x4661E)` is `0x461AB sw_0 0xF0(r10),r14 | edca00f0` with r14=0 on `ctx+0x28!=1` path. No `F0=N` store in output. CORRECTION of old `end F0=r14` claim. Other writer `0x45E52:0x45EC5 sw_0 0xF0(r11),r16`. Readers: `0x3EF?` none in verified post-call; whole-image many → BLOCKED.
- Caller alias PROOF: caller `r11=r3+0x10000` (`0x411C1/0x411C3`), `sw_0 0x4DBC(r11),r0` at `0x411C9/0x41221` = `*(r3+0x14DBC)`. Descriptor `r10=r3+0x14CCC` (`0x41275/0x41277/0x4127B`), so `descriptor+0xF0 = r3+0x14DBC`. Same address. VERIFIED.
- r12: single load `0x41213 lwz r12,-0x5EDC(r11)` where `r11=r3+0x10000` → `r12=*(r3+0xA124)`. No write inside caller. Checked at `0x41225 bnei r12,1→0x411C9` and `0x4128D bnei r12,1→0x411C9` (post-output cleanup). Provenance BLOCKED at caller-input boundary. Semantics UNKNOWN — do not name.

## 4. A/B downstream — last verified sink = memory
- Output fills `A/B` via `0x130D25` (fill), header copy `0x462A0+` (`lh/lhz/sw` 8 bytes `Pa/Pb/Pc/Pd`), loop `0x64369` + shift/store, sync-word restores `0x7FFE/0x8001` or `0x1FFF/0xE800`.
- Caller after `0x4618E` return: only `0x4128D r12-gate → clear F0 alias or j 0x41228 return-1`. No `F8/FC` load.
- Upstream `0x3EF64 jal 0x411A6; mov r17,r3; bnei r3,1→0x3EABD; sw_0 -0x5EE4(r10),r0; lwz r3,-0x5EF8(r10); jal 0x37D2D` — no F8/FC/F0 read. VERIFIED boundary.
- Whole-image F8=400 / FC=364 hits, but register-base identity unproven → NO downstream consumer VERIFIED. First opaque boundary = caller return → upstream memory-resident buffers.

## 5. SHM33
NO PROVEN RELATION. DSP ctx offsets are small struct fields (<0x200); SHM33 host offsets are `B+0xE02044/04`. No shared layout, no translation code, no `0x1044/0x3C0/0x33` structural link in verified chain. Numeric equivalence not used.

## 6. Standard vs MS12V22 (byte compare from utpa2k_stock.ko .data)
- Builder 217B `MS12:0x45D07 == STD:0x50AFE` identical. `dec_full.bin` builder identical to MS12.
- Gate `0x460B5/MS12 == 0x50EAC/STD` 32B identical; prep `0x46156/0x50F4D` identical; copy `0x462A0/0x51097` 64B identical. Parser/output wholes NOT identical → behavior equivalence not claimed.

## 7. AEON discipline
Byte-match proves provenance, not decode correctness. `sw_1/sh/lhz/bt.inst16_4/opcode_2B/opcode_2E/cMov` lanes/packing/indirect targets (`jr r23` at `0x46063`, `jr r9` returns) are limitations. No inference to fill gaps.
