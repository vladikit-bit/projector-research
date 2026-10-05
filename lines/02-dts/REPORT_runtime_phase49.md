# R49 — Predicate A (0xC49D): codec selector or state normalizer?

**Image:** `snd_full.bin` (AEON `aeon:LE:32:default`, MStar MT5889)
**Tooling:** Ghidra `analyzeHeadless` (project `r27_work/ghidra_r27f` / `sndr27`, `-noanalysis`) + `scripts/DumpR49Upstream.java`
**Date:** 2026-09-10
**Constraint compliance:** No patch, no device modification, no re-open of AUTH/CheckHashkey, no EDID re-analysis, no full-image MMIO survey, no new generic decoder. Upstream-only, 4 canonical callers only.

---

## 0. Proven fact retained from R48 (NOT extended)

> **Predicate A @ 0xC49D** reads `*(0xB000_0854)`, masks `0xA0000` (bits 17|19),
> returns `r3 = ((v & 0xA0000) == 0xA0000)`.
> That boolean controls, inside each canonical caller, whether **0xCB50** (sets bits 17|19) executes.

The R48 *interpretation* that "0x854 bits 17|19 = AC3/DTS selector" is **deliberately not retained** per task instructions. R49 tests it directly.

**Helpers (Ghidra ground truth, writer identities from R46d/R48):**
- `0xCB50` — writes bits 17|19 of 0x854 **SET**.
- `0xCC9B` — writes bits 17|19 of 0x854 **CLEAR**.
- `0xCA42` / `0xC55E` / `0xC5C5` / `0xCF86` — sibling output-config helpers (the "other codec" helpers in the traces below).

---

## 1. Method note — why "what function calls the block" is answered from disassembly, not symbols

The `-noanalysis` project has **no function boundaries** and **no reference graph** on arbitrary offsets:
- `getFunctionContaining(blk)` → null for all four blocks.
- `getReferencesTo(blk)` → empty for 0x19486, 0x1EDF0, 0x265F3.
- `getReferencesTo(blk)` → **one** direct reference: `0x1F8E1  (UNCONDITIONAL_JUMP) → 0x1F75C`.

`findEntry(blk)` (bounded backward prologue scan, `addi r1,r1,-N` + `bt.jr r9` within 0x800) returns inner prologues, all reached by **fall-through/jump**, so `getReferencesTo(entry)` is empty in every case. Therefore Q1 ("what function/context calls the block") is answered from the **control/data flow visible in the upstream window**, not from a recovered C name or caller list. The inferred entries are recorded for completeness:

| canonical blk | inferred entry | reached by |
|---|---|---|
| 0x19486 | 0x19486 (no prologue within 0x800) | fall-through |
| 0x1EDF0 | 0x1EC54 (`bn.addi r1,r1,-0x40`) | fall-through |
| 0x1F75C | 0x1F575 (`bn.addi r1,r1,-0x38`) | jump from 0x1F8E1 + fall-through |
| 0x265F3 | 0x26270 (`bn.addi r1,r1,-0x48`) | fall-through (inside R47 gate 0x25EE1) |

Each block sits in a **large audio-output configuration / format-dispatch state machine** (spans thousands of bytes; confirmed by the jump-target clustering in the windows).

---

## 2. Per-caller upstream traces (Ghidra ground truth)

Notation: **P1** = first `CALL 0xC49D` in the block (reads bits 17|19 **before** any CC9B); **P2** = the *canonical* `CALL 0xC49D` (the four addresses under study). `*+0x14` = the value compared against `0x2FFF` before CB50 runs (a size/threshold guard).

### 2.1 Caller 0x19486  (output-config function, ~0x176D6–0x19A5D)

```
upstream:
  r10+0x108  read @0x193FB ; if ==2 → 0x189D3 (skip whole sub-case)
  global init flag  r11-0x5944  read @0x19403
     !=0 → 0x1942E (already-init shortcut)
     ==0 → CA42 @0x19412 ; C55E @0x1941A ; set flag ; → 0x1942E
branch:
  P1 @0x19434  (pre-CC9B)  → if bits17|19 set → 0x19A5D (SKIP sub-block)
  global re-check @0x1946A (==0 / !=0) → both converge 0x19482
CALL 0xC49D (P2, canonical) @0x19486  → r12 = P2 (reads ENTRY bit state)
CALL 0xCC9B                 @0x19490  → clears bits 17|19
predicate result: r12 = P2 = (bits were SET on entry)
CB50 / skip:
  @0x19494  if r12!=0 → 0x194AB  SKIP CB50   (CC9B already left bits CLEARED)
  else check *(r13+0x14) <= 0x2FFF → if true CB50 @0x194A7 (SET) else skip
final 0x854 write: bits17|19 = SET iff (entry was CLEAR) AND (size<=0x2FFF); else CLEARED
```

### 2.2 Caller 0x1EDF0  (AC3 output setup, entry 0x1EC54, ~0x1EC54–0x1EF67)

```
upstream:
  r10+0x486c (AC3 slot)  read @0x1ED7D ; if ==0 → 0x1EF2C (skip)
  (DTS-adjacent slot r10+0x4870 handled @0x1EE49 → C5C5 @0x1EE5B when !=0)
  P1 @0x1ED8F (pre-CC9B) → if bits17|19 set → 0x1EF15 SKIP
branch:
  @0x1ED85  if r10+0x486c==0 → 0x1EF2C  (=> block entered when AC3 slot !=0)
CALL 0xC49D (P2, canonical) @0x1EDF0  → r14 = P2 (reads ENTRY bit state)
CALL 0xCC9B                 @0x1EDFA  → clears bits 17|19
predicate result: r14 = P2 = (bits were SET on entry)
CB50 / skip:
  @0x1EDFE  if r14!=0 → 0x1ED00 SKIP CB50
  else check *(r12+0x14) <= 0x2FFF → if true CB50 @0x1EE12 (SET) else skip
final 0x854 write: bits17|19 = SET iff (entry was CLEAR) AND (size<=0x2FFF); else CLEARED
```

### 2.3 Caller 0x1F75C  (format/codec dispatch, entry 0x1F575; jump from 0x1F8E1)

```
upstream:
  r26 = codec/format ID ; dispatch @0x1F70C/0x710/0x714/0x718  (r26 ∈ {9,8,5,2})
  r10+0x30 (DTS slot)  read @0x1F725 ; if ==0 → 0x1F8FC (skip)
  alt path: r11==1 @0x1F6EC → 0x1F78B reads r10+0x34 (DTS slot) → C5C5 @0x1F79C, 0xCF86 @0x1F7BA
  P1 @0x1F736 (pre-CC9B) → if bits17|19 set → 0x1F8E5 SKIP
branch:
  @0x1F6EC  if r11==1 → 0x1F725 ;  @0x1F72C if r10+0x30==0 → 0x1F8FC
  (=> block entered when DTS slot r10+0x30 !=0)
CALL 0xCC9B                 @0x1F754  → clears bits 17|19   *** CC9B BEFORE P2 here ***
CALL 0xC49D (P2, canonical) @0x1F75C  → r3 = P2 = (bits just CLEARED by CC9B) = FALSE always
predicate result: r3 = FALSE (CC9B ran first)
CB50 / skip:
  @0x1F760  if r3!=0 → skip  (never taken)
  else check *(r11+0x14) <= 0x2FFF → if true CB50 @0x1F773 (SET) else skip
final 0x854 write: bits17|19 = SET iff (size<=0x2FFF); else CLEARED   (force-SET for valid DTS)
```

### 2.4 Caller 0x265F3  (DTS gate 0x25EE1, entry 0x26270, ~0x26270–0x26773)

```
upstream:
  r10+0x3c read @0x26574 ; r10+0x44 @0x2657B (<=0xBB80) ; r10+0x60 read @0x26586
  if r10+0x60 !=0 → CA42 @0x26594 ; C55E @0x2659C ; set r10+0x60   (OTHER helper path)
  else fall to 0x265AF
  P1 @0x265B5 (pre-CC9B) → if bits17|19 set → 0x26773 SKIP
branch:
  @0x2658D  if r10+0x60!=0 → 0x265AF (other helper) ; ==0 → continue
  @0x26574/@0x2657B/@0x26582  (r10+0x3c / r10+0x44 <= 0xBB80)
CALL 0xC49D (P2, canonical) @0x265F3  → r14 = P2 (reads ENTRY bit state)
CALL 0xCC9B                 @0x265FD  → clears bits 17|19
predicate result: r14 = P2 = (bits were SET on entry)
CB50 / skip:
  @0x26601  if r14!=0 → 0x26451 SKIP CB50
  else check *(r11+0x14) <= 0x2FFF → if true CB50 @0x26615 (SET) else skip
final 0x854 write: bits17|19 = SET iff (entry was CLEAR) AND (size<=0x2FFF); else CLEARED
```

---

## 3. Cross-comparison of the four traces

| Aspect | 0x19486 | 0x1EDF0 (AC3) | 0x1F75C (DTS) | 0x265F3 (DTS gate) |
|---|---|---|---|---|
| Upstream gate (entry to block) | `r10+0x108 != 2` + global init flag | `r10+0x486c (AC3) != 0` | `r10+0x30 (DTS) != 0` | `r10+0x60 == 0` (else other helper) |
| "Other codec" helper in alt sub-case | CA42/C55E (init) | C5C5 (0x4870 path) | C5C5/0xCF86 (r12==1 path) | CA42/C55E (r10+0x60 path) |
| P1 (pre-CC9B) role | skip-if-armed guard | skip-if-armed guard | skip-if-armed guard | skip-if-armed guard |
| CC9B position vs P2 | **after** P2 | **after** P2 | **before** P2 | **after** P2 |
| P2 reads | entry bit state | entry bit state | CLEARED state (→ FALSE) | entry bit state |
| CB50 condition | P2==0 AND size<=0x2FFF | P2==0 AND size<=0x2FFF | size<=0x2FFF (P2 always FALSE) | P2==0 AND size<=0x2FFF |
| Final bits 17|19 | reconciled latch | reconciled latch | force-SET | reconciled latch |

**What is identical in all four:**
1. A **prior per-channel/per-format state slot** (not 0x854) selects whether the block is even entered — and *which* codec: `r10+0x486c` = AC3, `r10+0x30`/`r10+0x60` = DTS, plus the `r26` codec/format dispatch at 0x1F75C.
2. In the *alternate* sub-case (slot set for the "other" codec) a **different helper** (CA42/C55E/C5C5/0xCF86) is invoked — i.e. the codec choice is already made upstream by the slot value.
3. **P1 (pre-CC9B) is a "skip-if-already-armed" validation gate**: if bits 17|19 are already set, the whole sub-block is bypassed.
4. The `CC9B → P2 → CB50` triplet is a **clear / re-read / conditionally-set latch re-arm**, gated on the *prior* latch state plus a size threshold — not on codec identity.

**What differs:** only the CC9B/P2 ordering. At 0x1F75C (DTS) CC9B precedes P2, so P2 is hard-wired FALSE and CB50 force-runs; at the other three, P2 precedes CC9B so a pre-set latch makes P2 TRUE and CB50 is skipped (leaving the bits CLEARED). This asymmetry is itself evidence the sequence is a **state reconciler**, not a codec selector — if 0xC49D selected the codec, the AC3 and DTS callers would not disagree on ordering of the very same clear/set pair.

---

## 4. Resolution of the self-referential concern (the user's central objection)

> `old 0x854 state ↓ Predicate A ↓ CC9B clears bits 17|19 ↓ if predicate TRUE: skip CB50 else: execute CB50`

The objection was that this looks like it merely normalizes/toggles existing state rather than selecting the codec. **This is confirmed.**

- **CC9B and CB50 appear together** in every one of the four canonical callers (they are a paired clear-then-set, not codec-specific writers used in isolation). The R48 labels "CC9B = DTS-clear", "CB50 = AC3-set" described *contexts*, not a selector — both codecs run the same pair.
- **Predicate A is evaluated against a state that CC9B (or the upstream slot logic) has already shaped.** At 0x1F75C CC9B clears *before* P2, so P2 is structurally FALSE. At the other three, P2 reads the entry latch and is used only to decide *whether to re-arm*, not *which codec*.
- Therefore the **final 0x854 bits 17|19 are a deterministic function of (entry latch state, size threshold, CC9B/P2 ordering)** — i.e. a configuration/armed latch — and are **independent of AC3 vs DTS**.

**Direct answer to the key question:** `0xC49D` is **not** deciding the codec/output mode. It is operating on a state (`0xB000_0854` bits 17|19) that upstream logic — the per-channel slots (`r10+0x486c` AC3, `r10+0x30`/`r10+0x60` DTS) and the `r26` format dispatch — has **already selected**. Predicate A validates/normalizes that latch (skip-if-armed, re-arm on clear), exactly as R49-B predicts.

---

## 5. Classification

# **R49-B — State normalization / validation**

Retained proven fact (R48): Predicate A reads 0x854 bits 17|19 and its result gates CB50.
**Rejected:** the R48 interpretation that this is an AC3/DTS *selector*. The bit pair is a re-armable "configured" latch; the AC3/DTS decision lives entirely upstream of 0xC49D.

---

## 6. Implication for the DTS-passthrough bug & next step

Because bits 17|19 of 0x854 are now **ruled out as the AC3/DTS divergence point** (both codecs run the identical CC9B→CB50 re-arm, and the user's injunction against inferring "bit = AC3 / bit = DTS" from differing final states is honored — the final states are driven by latch/size, not codec), the actual DTS-vs-AC3 failure must originate **upstream**:

1. The **per-channel slots** `r10+0x486c` (AC3) vs `r10+0x30`/`r10+0x34` (DTS) and `r10+0x60`: if DTS fails, check whether `r10+0x30` is ever non-zero when DTS input is present (0x1F75C is skipped when it is 0 — `0x1F72C`).
2. The **`r26` codec/format dispatch** at 0x1F70C–0x1F718 (values 9/8/5/2 → writes `r3+0x28` = 2/4/5): confirm DTS's format ID actually reaches this dispatch and the DTS slot gets set.
3. The **R47 gate 0x25EE1** (DTS-path-exclusive TX-arm): its entry conditions (`bit7(0x0C06)`, `sign(0x0015)>=0`, `0x2512C4 != 0`) remain the most likely DTS-specific failure locus and are untouched by R49.
4. **Other bits of 0x854** are still unmapped by R49 (only 17|19 were in scope). A true AC3/DTS selector, if encoded in 0x854 at all, would have to be a *different* bit field — that remains open but is explicitly outside this narrow R49 pass.

No patch was prepared or is recommended from R49; this is a read-only static finding.

---

## 7. Evidence pointer

- `r49_work/r49_upstream.txt` — raw Ghidra dump: 4 upstream windows + inferred entries + xref reports (this run, post-Edit-2 `DumpR49Upstream.java`).
- `scripts/DumpR49Upstream.java` — the dumper (reuses `DumpPostCall.java` `getInstructionAt` walk; not a generic decoder).
