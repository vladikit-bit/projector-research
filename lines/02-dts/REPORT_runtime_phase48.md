# REPORT — R48: Predicate A (0xC49D / 0xB000_0854 bits 17|19) caller & output-control trace

**Firmware:** `snd_full.bin` (AEON / MStar MT5889, Thundeal TD98 Pro)
**Question:** Does the boolean produced by Predicate A — `(0xB000_0854 & 0xA0000) == 0xA0000`
(bits 17|19 both set) — reach a real hardware-output / SDO / non-PCM decision,
or is it merely a capability/status field?

**Classification: R48-A — PROVEN OUTPUT-CONTROL GATE (VERIFIED, Ghidra ground truth).**

---

## Method (Ghidra ground truth only — no Python decoder, no broad reinvestigation)

- Reused existing project `r27_work/ghidra_r27f` / `sndr27`, `analyzeHeadless -noanalysis`.
- **xref API validated against a known callee:** `getReferencesTo(0x10786E)` = **1919**
  references (the `bg.jal 0x10786E` packer called ~12× in `ghidra_854.txt`). So the
  Ghidra reference machinery is live in this project.
- `getReferencesTo(0xC4E0)` = **0** — expected, because `0xC4E0` is *mid-function*
  (`0xC4C0` is `<UNDEFINED>`, the 2nd word of a `movhi` that starts at `0xC4BC`); the
  true function entry is `0xC49D`. Searching `0xC4E0` alone (as an earlier throwaway
  scan did) returns 0 callers — a wrong-target artifact, not death code.
- Full-image scan (587,030 instructions, the proven `FindMMIO`/`DumpAEON` walk) for call
  targets in `[0xC380,0xC4F0]` → **11 callers, every one targeting `0xC49D`.**
- Post-call windows obtained via Ghidra `getInstructionAt` **following real instruction
  boundaries** (AEON insns are 2/3/4 bytes — the 4-byte step bug that produced the
  `<UNDEFINED>` artifacts is fixed).

---

## 1. Predicate A reconstruction

Function entry `0xC49D` (return at `0xC4EE`, `bt.jr r9`). Body (Ghidra disassembly):

```
0xC49D  bg.movhi r23,0xb000
0xC4A1  bg.ori   r24,r23,0x814
0xC4A5  bn.lwz   r24,0x0(r24)        ; r24 = *(0xB000_0814)
0xC4A8  bn.slli  r25,r24,0xa
0xC4AB  bn.bgtsi r25,-0x1,0xc4f0     ; early-out guard
0xC4AE  bg.ori   r23,r23,0x80c
0xC4B2  bn.lwz   r24,0x0(r23)        ; r24 = *(0xB000_080C)
0xC4B5  bn.movhi_2 r26,0xff
0xC4B8  bn.lwz   r23,0x0(r23)        ; r23 = *(0xB000_080C)
0xC4BF  bn.and    r24,r24,r26
0xC4C2  bn.and    r23,r23,r26
0xC4C5  bg.beq    r24,r23,0xc527     ; guard compare
   ...  (0xC4C9–0xC4D5: 0xC527/0xC53B sub-paths) ...
0xC4D8  bg.movhi r23,0xb000
0xC4DC  bg.ori   r23,r23,0x854
0xC4E0  bn.lwz   r23,0x0(r23)        ; r23 = *(0xB000_0854)
0xC4E3  bt.movhi r26,0xa            ; r26 = 0xA0000 (bits 17|19)
0xC4E5  bn.and    r23,r23,r26        ; r23 = v & 0xA0000
0xC4E8  bn.sfne   r23,r26            ; flag = (v&0xA0000) != 0xA0000
0xC4EB  bn.cmovsi_2 r3,r0,0x1       ; r3  = flag ? 0 : 1
0xC4EE  bt.jr     r9                 ; return r3
```

**Boolean:** `r3 = 1` iff `(v & 0xA0000) == 0xA0000` (bits 17|19 BOTH SET);
`r3 = 0` iff not both set.  (Predicate result per the agreed R48 definition:
`(0xB000_0854 & 0xA0000) == 0xA0000`.)

---

## 2. Complete caller set (11) — Ghidra ground truth

| # | Caller addr | Target | Consumes r3? |
|---|-------------|--------|--------------|
| 1 | `0x19434` | `0xC49D` | yes (branch) |
| 2 | `0x19486` | `0xC49D` | yes (gates `0xCB50`) |
| 3 | `0x1E6AB` | `0xC49D` | no (r3 dead/overwritten in window) |
| 4 | `0x1ED8F` | `0xC49D` | yes (branch) |
| 5 | `0x1EDF0` | `0xC49D` | yes (gates `0xCB50`) |
| 6 | `0x1F0BB` | `0xC49D` | no (r3 dead/overwritten in window) |
| 7 | `0x1F736` | `0xC49D` | yes (branch + nested call) |
| 8 | `0x1F75C` | `0xC49D` | yes (gates `0xCB50`) |
| 9 | `0x1F8DD` | `0xC49D` | no (tail-forward to `0x1F750`) |
|10 | `0x265B5` | `0xC49D` | yes (branch + nested call) |
|11 | `0x265F3` | `0xC49D` | yes (gates `0xCB50`) |

All 11 confirmed by full-image Ghidra scan; **none** call `0xC4E0` directly.

---

## 3. TRUE / FALSE branch behaviour (per caller)

Predicate result `r3 = 1` ⇔ bits 17|19 BOTH SET (TRUE); `r3 = 0` ⇔ not (FALSE).

```
caller 0x19434
 → CALL 0xC49D
 → next: bn.lwz r23,0x14(r13)
 → branch/select: 0x1943B bg.bnei r3,0x0,0x19a5d   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x19a5d (skip store-increment + bg.jal 0xd4c8 block)
 → FALSE (r3==0): fall-through 0x1943F..0x19462 (store/increment, bg.jal 0xd4c8)

caller 0x19486
 → CALL 0xC49D
 → next: bt.mov r12,r3 ; bg.addi r3,r14,0x6ad4 ; bg.jal 0xCC9B   (DTS-clear: writes 0x854 clearing 17|19)
 → branch/select: 0x19494 bn.bnei r12,0x0,0x194ab   (taken if TRUE)
 → TRUE  (r3≠0, bits set): jump 0x194ab ⇒ SKIP 0xCB50 ⇒ only DTS-clear ran ⇒ bits 17|19 CLEARED ⇒ DTS/non-PCM output
 → FALSE (r3==0, bits not set): fall-through ⇒ 0x194A7 bg.jal 0xCB50 (AC3-set: writes 0x854 setting 17|19) ⇒ bits SET ⇒ AC3 output

caller 0x1E6AB
 → CALL 0xC49D
 → next: bn.lwz r8,0x0(r10) ; bn.lwz r23,0x2c(r10) ... (r3 not branched; overwritten 0x1E6D4 bg.lwz r3,0x90(r4))
 → branch/select: NONE on r3 in window (branches on r23 @0x1E6B7/0x1E6D0; r3 reloaded before 0x1E6D8)
 → boolean appears DEAD in this caller body (passed up / consumed beyond window)

caller 0x1ED8F
 → CALL 0xC49D
 → next: bt.mov r14,r3 ; bn.lwz r23,0x14(r12)
 → branch/select: 0x1ED98 bg.bnei r3,0x0,0x1ef15   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x1ef15
 → FALSE (r3==0): fall-through load/store sequence

caller 0x1EDF0
 → CALL 0xC49D
 → next: bt.mov r14,r3 ; bg.addi r3,r13,0x6ad4 ; bg.jal 0xCC9B   (DTS-clear)
 → branch/select: 0x1EDFE bn.bnei r14,0x0,0x1ed00   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x1ed00 ⇒ SKIP 0xCB50 ⇒ DTS-clear only ⇒ bits CLEARED ⇒ DTS
 → FALSE (r3==0): fall-through ⇒ 0x1EE12 bg.jal 0xCB50 (AC3-set) ⇒ bits SET ⇒ AC3

caller 0x1F0BB
 → CALL 0xC49D
 → next: bn.lwz r24,0x0(r15) ; bn.lwz r23,0x18(r15) ... (r3 not branched; overwritten 0x1F0D5 bg.lwz r3,0x3038(r15))
 → branch/select: NONE on r3 in window (branch on r23 @0x1F0C7)
 → boolean appears DEAD in this caller body

caller 0x1F736
 → CALL 0xC49D
 → next: bn.lwz r23,0x14(r11)
 → branch/select: 0x1F73D bg.bnei r3,0x0,0x1f8e5   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x1f8e5
 → FALSE (r3==0): fall-through ⇒ 0x1F750 bg.addi r3,r14,0x6ad4 ; bg.jal 0xCC9B (DTS-clear) ; bg.jal 0xC49D (nested)

caller 0x1F75C
 → CALL 0xC49D
 → next: (r3 live)
 → branch/select: 0x1F760 bn.bnei r3,0x0,0x1f6f2   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x1f6f2 ⇒ SKIP 0xCB50
 → FALSE (r3==0): fall-through ⇒ 0x1F773 bg.jal 0xCB50 (AC3-set) ⇒ bits SET ⇒ AC3

caller 0x1F8DD
 → CALL 0xC49D
 → next: 0x1F8E1 bg.j 0x1f750   (UNCONDITIONAL tail-forward; boolean discarded)
 → branch/select: NONE on r3 here (boolean recomputed inside 0x1F750, which re-calls 0xC49D)
 → not a consuming site itself (forwarder)

caller 0x265B5
 → CALL 0xC49D
 → next: bn.lwz r23,0x14(r11)
 → branch/select: 0x265BC bg.bnei r3,0x0,0x26773   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x26773
 → FALSE (r3==0): fall-through store/increment + bg.jal 0xd4c8 ; nested bg.jal 0xC49D @0x265F3

caller 0x265F3
 → CALL 0xC49D
 → next: bt.mov r14,r3 ; bg.addi r3,r12,0x6ad4 ; bg.jal 0xCC9B   (DTS-clear)
 → branch/select: 0x26601 bn.bnei r14,0x0,0x26451   (taken if TRUE)
 → TRUE  (r3≠0): jump 0x26451 ⇒ SKIP 0xCB50 ⇒ DTS-clear only ⇒ bits CLEARED ⇒ DTS
 → FALSE (r3==0): fall-through ⇒ 0x26615 bg.jal 0xCB50 (AC3-set) ⇒ bits SET ⇒ AC3
```

**Canonical output-control pattern (callers 2, 5, 8, 11):** `0xCC9B` (DTS-clear, writes
`0xB000_0854` clearing bits 17|19) always runs; `0xCB50` (AC3-set, writes `0xB000_0854`
setting bits 17|19) runs **iff the predicate is FALSE** (bits not already set). The
boolean therefore decides the final `0x854` bit state → AC3 vs DTS (non-PCM) output.

---

## 4. Exact first output-related consumer

- **`0xCC9B`** — DTS-clear helper; writes `0xB000_0854` clearing bits 17|19. Executed
  **unconditionally** in the canonical callers (e.g. `0x19490`, `0x1EDFA`, `0x1F754`,
  `0x265FD`), i.e. on the TRUE path.
- **`0xCB50`** — AC3-set helper; writes `0xB000_0854` setting bits 17|19. Executed on the
  **FALSE path**, gated by the predicate branch (`0x194AB`/`0x1ED12`/`0x1F773`/`0x26615`).
- Both are the `0x854` output-config writers established in R46d (AC3 `0xCBFB` sets
  bits 2,6,17,19; DTS `0xCC9B`/`0xCF86` clears 17|19). `0x854` is the SDO/non-PCM
  output-control register (R46/R47). The predicate's boolean **directly gates** `0xCB50`.

---

## 5. Classification: VERIFIED

- Ghidra ground truth (not the Python decoder): full-image scan of 587,030 instructions,
  xref API validated against a known callee (1919 refs).
- 11 callers enumerated, all → `0xC49D` (true entry, not the mid-function `0xC4E0`).
- 8/11 callers branch on `r3`; 4 of those gate the `0x854` output writer `0xCB50`.
- The 3 non-consuming sites (`0x1E6AB`, `0x1F0BB`, `0x1F8DD`) are dead/forwarder, not
  counter-evidence (e.g. `0x1F8DD` forwards to `0x1F750` which re-calls `0xC49D`).

---

## 6. Decision: R48-A — PROVEN OUTPUT-CONTROL GATE

The boolean produced by Predicate A **does reach a hardware-output decision.** It gates
whether the firmware writes `0xB000_0854` (the SDO/non-PCM output-control register) with
the AC3-set helper `0xCB50`. Bits 17|19 on `0x854` are **not merely a capability/status
field** — they are actively written by codec-specific output helpers and the predicate's
result selects between the DTS-clear (`0xCC9B`) and AC3-set (`0xCB50`) writes.

### Minimal reversible patch candidate (R48-A) — **NOT DEPLOYED, HYPOTHETICAL**
At canonical caller `0x19486`, the `0xCB50` (AC3-set) gate is the single instruction
`0x19494 bn.bnei r12,0x0,0x194ab`. To force the codec-specific writer deterministically
(remove the self-referential `0x854`-state guard), the minimal reversible change is to
neutralise this one branch (e.g. replace with `bn.beqi r12,0x0,0x194ab` so AC3-set always
runs, or NOP the branch). **Reversible:** record the original 4 bytes (`20 35 88` + the
opcode word) to restore.

**Caveats (important):**
1. This patch is **speculative and unverified** — included only because R48-A is proven.
2. It is **NOT deployed** (per the standing no-patch / no-device-modification constraint).
3. The boolean is **self-referential**: Predicate A *reads* `0x854`, then gates a *write*
   to `0x854`. It maintains the current output mode rather than making a fresh decision.
4. The **actual DTS-passthrough failure is downstream of Predicate A** — gate `0x25EE1`
   (DTS TX-arm) requires hardware status `0x0C06` bit7, which never sets at runtime (R47).
   This Predicate-A patch therefore will **not by itself restore DTS output**; it only
   addresses the boolean→writer gating, not the hardware arm gate.

---

## Scope / exclusions honoured
- No CheckHashkey/AUTH, EDID, `0x13292`, `0x868/0x86C/0x870` investigation.
- No Python-decoder caller claims (Ghidra ground truth used throughout).
- No re-investigation of the `0x854` register map (R46d bit-map reused as established fact).
- No full-image MMIO re-survey; only the 11 already-known callers inspected.
