# MUSE TASK — the B-latch object: who writes `+0x0` and `+0x10`, and is the write codec-conditional

**Date:** 2026-09-30 (evening) · **Priority:** highest — this is the narrowest open gate
**Files:**
- **`dec_work/dec33_realcode.txt`** (504 216 real instructions). Never `dec32_clean.txt`.
- DEC image `dec_work/dec_full.bin` (file offset == fw addr)
- SND realcode `snd_work/snd33_realcode.txt`
- Read first: `FINDING_dts_es_to_spdif_20260930.md` (incl. the ADDENDUM) — the verified chain
  is there; do not re-derive it.
- Prior session report with the latch-chain work: `MUSE_TASK_ring_remaining_20260930.md`
  (the "[0x0] B-latch 0 for DTS" finding and the "4-gate latch chain" live there).

**Do not delete any Ghidra project — create new folders/projects. Static only, no flashing.**

---

## Where the chain stands (verified, so you do not redo it)

Everything downstream of decode keys off ONE object — call it the B-latch — whose pointer
`r4` reaches `F_45FD1`, where:

```
0x45FDB movi r23,0x1 / 0x45FDD sw 0x28(r3),r23   ; prologue: [+0x28] initialized to 1
0x45FE0 lwz  r23,0x0(r4)      ; B = [+0x0]
0x45FEB beqi r23,0x1,0x4600C  ; B==1  -> copy arm   -> 0x46013 sw 0x2c(r10),r0  (=0)
0x45FEE lwz  r11,0x10(r4)     ; [+0x10]
0x45FF1 beqi r11,0x1,0x46080  ; ==1  -> 0x46080 arm -> 0x46087 sw 0x2c(r10),r11 (=1)
0x45FF5 sw 0x28(r3),r0        ; neither -> zero [+0x28], return: output stage dead
```

The IEC61937 packer (`0x46259`, Pc tags `0x0B/0x10C/0x20D` at `0x45DA4/0x45D83/0x45DC3`,
real Pa/Pb `0xF872/0x4E1F` at `0x45D7F/0x45D8E`) runs only when `+0x2C==1`, i.e. only on the
`0x46080` arm. The whole block is invoked from the pump `F_410A3` via
`0x4121E beqi r23,0x1,0x41275`, where `r23 = [r11+0x4EE4]` is the **pump progress latch**:
`0x3E919` writes 1 after a pass where the pump returned non-zero (`0x3E82D bnei r3,0x0`),
`0x3E833` writes 2 after a return of 0. Range gate `0x253C7` is wide open for DTS (Pc `0x0B`
passes directly; the divert `0x2549D blesi r24,0x8` rejoins for Pc ≤ 8). SND-side whitelist
and `0x0B`-scan exonerated.

**Under DTS the B-latch reads `[+0x0] = 0` (your own earlier ring-report finding), so every
path hinges on `[+0x10] == 1`.** Its writer is the question.

## One correction that scopes this task

`dump_es` taps the decoder's **INPUT** ES (AC-3 dump head `0b 77` = AC-3 syncword `0x0B77`;
`dump_pcm` is the separate output dump). The 2.88 MB DTS "ES" is the framed source arriving,
not DSP output. **The decoder's output toward SND has never been observed live** — the static
chain is the primary evidence. Do not treat the ES dump as proof any downstream stage ran.

## Q1 (identity). Is there one B-latch object or two?

The object pointer is loaded at two sites with **different offsets** off the pump's base:

* pump head: `0x410B5 lwz r3,-0x5EF8(r10)` where `r10 = r3+0x1000` → field `[r3-0x4EF8]`,
  immediately passed to `0x410BD jal 0x37D18`;
* call site: `0x4127D lwz r4,-0x5EF8(r11)` where `r11 = r3` → field `[r3-0x5EF8]`, passed to
  `0x41283 jal 0x45FD1`.

Determine: are `[r3-0x4EF8]` and `[r3-0x5EF8]` the same object (one latch) or two? Find where
**each field** is written. This decides whether `F_37D18` and `F_45FD1` see the same state.

## Q2 (decisive). The writer of `[+0x10]` and its codec-conditionality

`0x10(rN)` is a ubiquitous offset — a blind grep is worthless. Scope it by **pointer
provenance**:

1. Establish the base `r3` identity: `F_410A3` is called at `0x3E829` with `r3 = r11` of its
   caller (find that caller's head — the loop lives around `0x3E7D0..0x3E95D`) and
   self-recursively at `0x41238` with `r3 = r10`. Walk up to a named entry and note every
   aliasing (`add`/`mov` of the base) so the sweep cannot pick a false positive from another
   function's `+0x10`.
2. With the base provenance fixed, sweep the full corpus for stores through that base at
   `+0x0` and `+0x10`, and for the fields at `-0x4EF8`/`-0x5EF8` of the pump's caller struct.
3. For each writer: is it reachable under DTS, and what decides the value? Re-derive your
   "4-gate latch chain" with these new endpoints — if one writer is codec-conditional
   (stream type 4 vs 5, the `0x19862` tag word, a caps/format flag), **that is the block**.

## Q3. What makes the pump return 0 (the DTS-stall hypothesis)

Map `F_410A3`'s return paths: `0x411CD` (returns 1), `0x41234`/`0x41205` regions,
`0x4123C beqi r3,0x0,0x41295` (recursion returned 0 → `0x41295` → `0x411E7`), and the
`r4`/`r12` gates (`0x41217 bnei r4,0x0`, `0x41225 bnei r12,0x1,0x411c9`, where
`r12 = [r11-0x5EDC]` — the same word read at the pump head `0x410EA`). If under DTS the pump
can never return non-zero, the latch sticks at 2 and the output stage never runs — that is a
complete mechanism for "TX silent". Name the condition that would have to flip.

## Q4. `F_37D18` — what the pump asks the latch

`0x410BD jal 0x37D18` receives the object and its result feeds `0x410C6..0x410E7`
(sign-trick into `cmov`). Determine what it computes and whether it is codec-conditional.

## Method traps (each has cost real time here)

* Branch targets are 8 hex digits; mnemonics carry suffixes and trailing commas; offsets are
  lowercase; strip the mnemonic field. `jr r9` can end a fragment — verify with a branch in.
* `0x10(rN)` and `0x28(rN)` are common offsets: sweep only with provenance fixed, else you
  will report 600 false hits.
* Counters read once will be re-derived wrongly. Every claim cites a realcode address or a
  binary offset. A clean negative is a result — say what you searched.

## Deliverable

A ranked list of the writers of the latch `+0x10` (and `+0x0`), each with: address, the value
written, reachability under DTS, and codec-conditionality. If exactly one write decides
`[+0x10]==1` under DTS, say so loudly with the deciding branch — that is the one-byte patch
candidate this project has been narrowing toward for a month. If instead the latch is
codec-blind and the stall comes from the pump's return value, say that plainly: it moves the
A ranked list of the writers of the latch `+0x10` (and `+0x0`), each with: address, the value
written, reachability under DTS, and codec-conditionality. If exactly one write decides
`[+0x10]==1` under DTS, say so loudly with the deciding branch — that is the one-byte patch
candidate this project has been narrowing toward for a month. If instead the latch is
codec-blind and the stall comes from the pump's return value, say that plainly: it moves the
target to Q3.

---

## RESULTS (2026-09-30, static only, `dec33_realcode.txt`)

ADDENDUM read and independently verified where it matters (caller `0x3E827/
0x3E829`, latch writers `0x3E919/0x3E833`, arms, Pc tags, `0x253C7`-rejoin,
scanner adjacency, const absences — all confirmed byte-exact in `dec33`).
Two corrections below stand even against the ADDENDUM (§Q1 field expression,
§Q3 the `-0x5EDC` paradox). Traps respected: dual-pattern sweeps
(offset-first `bn.sw` + reg-first `bg.lwz`/`bg.sw_0`), branch-into verified
entries, no blind common-offset greps for attribution.

### Q1. ONE latch object (three readers) — with one field-expression correction

The task's `[r3-0x4EF8]` does not exist in the listing. The head reads
`0x410B5 bg.lwz r3,-0x5ef8(r10)` with `r10 = entry_r3 + K`
(`0x410A6 bt.movhi r23,0x1; 0x410AC bn.add r10,r3,r23`; K's numerics
unresolved — `bt.movhi` shift semantics unproven — and **identity-irrelevant**,
since every site uses the same `movhi-0x1` idiom). The "0x1000" is
unverified: flagged, not adopted.

Same-object evidence (STRONGLY SUPPORTED, one residual):
1. Identical field offset `-0x5EF8` at all three reads: head→`F_37D18`
   (`0x410B5`), body-reload→direct B/`[0x10]` tests
   (`0x41111 lwz r23,-0x5ef8(r10)` → `0x41115/0x4111B` → copy arms
   `0x4118C`/`0x41199`), call-site→`F_45FD1` (`0x4127D lwz r4,-0x5ef8(r11)`).
2. Pointer installers unanimous: `-0x5EF8` has exactly TWO writers FW-wide
   (complete dual-pattern sweep) — `0x3E1A6` (`r23 = r11+0x7C4`) and
   `0x3EA38` (`r28 = r11+0x7C4`, `0x3EA04`) — both value `r11+0x7C4`, both
   unconditional-once-reached (straight-line windows `0x3E173+`/`0x3EA00+`).
   (The `0x3E7B7 +0x5EF8` store is a different field — sign differs;
   explicitly excluded.)
3. Functional coherence: all three readers implement the same trichotomy
   (`B==1`→copy-ish, `[0x10]==1`→arm-ish, else dead) — head via `F_37D18`,
   body direct, output stage `F_45FD1`.
4. ADDENDUM's independent convergence (same conclusion, different route).
- Residual R1: base flow `r10_head == r11_callsite` has one unmapped link
  (branch into the `0x411A8`-block; `r3` provenance at `0x41207`, whose sole
  entry is branch `0x411BA`). Everything else in the chain is call-verified.

### Q2. Ranked writers of `+0x10` / `+0x0` (with provenance)

Base provenance: `F_410A3` gets `entry_r3 = caller.r11`
(`0x3E827 mov r3,r11; 0x3E829 jal`, sole external caller; + self-recursion
`0x41238`). Latch pointer = `caller.r11+0x7C4` (both installers). Content
writes land on `F_3D952` descs (`r10 = entry_r7`, `+0xA4` stride array from
the `F_3E961` loop via sole caller `0x3F6B6`) — desc≡latch linkage residual
R2 (`F_3E961` r7↔r11 relation; continue at `0x3F6B6` arg setup).

1. **Pointer installers — codec-blind, value-unanimous.** `0x3E1A6`,
   `0x3EA38`: no compares in-window, pure `r11`-arithmetic. DTS/AC-3 can
   differ only via `r11` itself (per-stream base, runtime). The "which
   installer runs for DTS" question dissolves (same value).
2. **`+0x0` (B): `0x3DE7C` latch (`=1`)** iff the 4-gate chain holds —
   re-derived with endpoints: `[r12+0xC]∈{1,2}` (`0x3DA55/0x3DA59`) →
   `[r10+0x18]` (`0x3DDE8` via `F_37EC2`, or `0x3DE2C` via `F_37ED6`) →
   `0x3DA6B` → `F_37EDC(r13)==[[r13+0x54]+0x68]==1` (`0x3DE71`;
   `F_37EDC` = 3-insn double-deref `037edc..037ee2`, sole caller `0x3DE6D`)
   → `[r10+0x4]==1` (`0x3DE78`; `+0x4 ← 0x3DA68` = `F_37F25` return).
   Init default `+0x0 ← 0x294A6` (`F_37535` query). No codec immediates
   anywhere in-chain (full windows verified).
3. **`+0x10`: `0x3DB62` (= `F_37F54` return)**; init default `0x294E0`.
   Both blind query+store; values descriptor-fed (runtime).
4. **The loud answer the brief asked for: no single codec-conditional
   WRITE exists.** Every writer above is branch-free on codec; the DTS
   decision lives in query RESULTS and gate COMPARES. The patchable points
   are the 4 latch compares (`0x3DA55/0x3DA59`, `0x3DA6B`, `0x3DE71`,
   `0x3DE78`) and the `F_45FD1` entry branches — with the failing one
   statically unknowable. No one-byte writer candidate exists; do not
   propose one.

### Q3. Pump return map → the DTS-stall is a return-0, four possible arms

Every `jr` in `[0x410A3,0x41360]` with return value (r3 at `jr`):
- return-1: `0x411D9`, `0x41205`, `0x41234`, `0x41228`, status arms
  `0x412E9+` (each `movi r3,1`), the `0x411C9`-path (→`0x411CD` return-1).
- **return-0: `0x4110C`** (r3 = head-predicate: fork/pred-0 or `-0x5EDC`≠1
  path), **`0x4116D`** (remaining-predicate 0), **`0x4118A`** (ALWAYS 0 —
  the refill exit: derive+refill then `movi r23,0`), **`0x412C8`**
  (classifier fallthrough, `movi r3,0`).
- Recursion `0x41238`: `0x4123C beqi-0 → 0x41295 → j 0x411E7` (ends
  return-1!). So a nested 0 still surfaces as 1 unless an outer frame hits
  a return-0 arm. Caller `0x3E82D bnei r3,0 → 0x3E904` else latch=2:
  **latch sticks at 2 ⟺ the pump persistently exits via a return-0 arm.**
  The flip conditions (narrowest first):
  1. **`[r10-0x5EDC]≠1` → immediate return-0 at `0x4110C`
     (`0x410EA lwz; 0x410FA beqi-1 → continue`) — before ANY output code.
     PARADOX (stated plainly): complete dual-pattern sweep finds ZERO
     explicit writers of `-0x5EDC` FW-wide (many readers: `0x3DEB9`,
     `0x3E87A`, `0x3F0E2+`, `0x40C5C/0x40EE2/0x40F30`, `0x410EA/0x41213`).
     Value must come from bulk-init/DMA/aliasing (residual R3 with teeth:
     watch the word live). AC-3 passes it (plays) ⟹ runtime value IS 1
     for AC-3; DTS's value unknown. Narrowest stall word, writer-unknown.
  2. Head conjunction (all must hold or `r23` zeroes out via `cmov_0`
     chain `0x410E7/0x410F1/0x410F7` → return-0 at `0x4110C`):
     `F_37D18≠0` AND `[r10-0x5F28]≤0x7F0` (`0x410DF sfleui`) AND
     `[r24+0xA34]≠0` (`0x410EE`) AND `[r24+0xA20]≠0` (`0x410F4`),
     `r24=[r11+0x290]` — four further word-conditions (writers unmapped;
     small-offset trap respected by not blind-grepping).
  3. Remaining-predicate 0 (`0x4115F`: `(remaining<<5)≠0x1E0` i.e.
     remaining≠15 → 0) and fork-path 0 (`0x410FD`).
  4. Refill-path exit `0x4118A` (taken when `[0x14]−[0xC] > 0x200`):
     refills then returns 0 — a DTS that keeps refilling never progresses
     the latch.
  5. Classifier fallthrough `0x412C8`.

### Q4. `F_37D18` = `([+0x0]==1) || ([+0x10]==1)`, codec-blind

Body `037d18..037d2b`: `[0x0]==1 → return 1` (`0x37D1B/0x37D1D/0x37D29`);
else `[0x10]`-cmov → return. Adopted OR-semantics, dual-cited: (a) the
archive's independent decode ("params->0x0==1 || params->0x10==1");
(b) body-shape + cross-function coherence (head use `0x410C3 beqi-0 →
plain` mirrors `F_45FD1`'s neither-arm; `cmovsi_2` polarity micro-residual
noted but outcome-equivalent under both witnesses). No type compares in
the entry (`0x37D18–0x37D2B` full read) → codec-blind VERIFIED. Extra
entries attributed, not conflated: `0x37D2D` (`+0x24`-test leaf, 6 setup
callers `0x3D808+`) and `0x37D38` (struct-copy leaf, 2 callers) — separate
leaves with own callers. Head use `0x410C6..0x410E7`: nonzero →
`op16`/`opcode_2E` dispatch on `r12<<1` → negate → sign-extract (`srli
0x1F`, the sign-trick) → folded with the descriptor words into the
return-0-vs-continue verdict above.

### Ranked deliverable (writer → value → DTS-reachability → conditionality)

1. Pointer installers `0x3E1A6` / `0x3EA38` → `r11+0x7C4` → always (once
   reached) → blind. Codec enters only via `r11`.
2. `+0x0`: `0x3DE7C` (=1, 4-gate chain §Q2) → DTS fails ≥1 gate (prior:
   never latches) → data-driven, no immediates. Init `0x294A6` (query).
3. `+0x10`: `0x3DB62` (query return) → runtime per-desc → blind.
   Init `0x294E0`.
4. Stall words (Q3): `[r10-0x5EDC]` (writer-unknown! narrowest), head
   conjunction (`[r10-0x5F28]`, `[r24+0xA34/A20]`, `F_37D18`), remaining/
   fork predicates, refill exit, classifier fallthrough.
5. Progress latch `[r11+0x4EE4]` (`0x3E919`=1 / `0x3E833`=2): NOT a codec
   flag (ADDENDUM-verified) — consequence of pump-return, not cause.
   Do not patch it.
- **One-byte patch candidate: NONE as a writer.** If the stall is latch
  content, the targets are the 4 compares (§Q2); if it is pump-return,
  the targets are the Q3 arms — and the failing arm is runtime-known
  only. The target moves to Q3 exactly as the brief anticipated: the
  latch is codec-blind, and the stall is a return-0.

### Residuals (address-named, none blocking the watchlist)

- R1: `r3` provenance into the `0x411A8`-block (sole entry branch
  `0x411BA`); K-numerics (`bt.movhi`) irrelevant to identity, still open.
- R2: desc≡latch linkage (`F_3E961` r7↔r11; continue at `0x3F6B6` args +
  `0x3E827`-r11 origin above `0x3E7D0`).
- R3: `-0x5EDC` value source (bulk-init/DMA/aliasing; live-watch closer);
  `-0x5F28/+0xA34/+0xA20` writers (trap-respecting skip).
- R4: `cmovsi_2` polarity micro-proof (outcome-equivalent per dual witnesses).
- R5: SND side untouched this session (prior exoneration stands).

Scripts: `b_q.py` (heads/returns/`F_37D18`/loop/installer sweeps),
`b_q2.py` (all `jr`+values, installer gates, corrected field sweeps,
entries), `b_q3.py` (installer gates, reload, `0x41207`-entry).
