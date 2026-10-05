# R31 — Static Reconstruction of the DTS Gate / Output-State Path

**Module:** `mst_snd_r2_MS12V22.bin` (extracted `snd_full.bin`) — MStar/MedTek AEON SND audio firmware
**CPU:** `aeon:LE:32:default` · `flow[0]` authoritative · `opcode_2E` = MMIO-touch
**Prior phases:** R29 (differential static trace → R29-B) · R30 (runtime: gate-block during DTS → R30-B, runtime-VERIFIED)
**This phase:** R31 — **pure static analysis only.** No runtime captures, no device changes, no patches.
**Mode:** Read-only static reconstruction from Ghidra `flow[0]` decodes (`r29_work/`).
**R30 usage:** R30 runtime observations are used ONLY as *directional evidence*; R31 conclusions are derived
from static code, never promoted into causal proof.
**Deliverable date:** 2026-09-09

---

## 0. Headline classification

> ### R31 = **R31-C** — the gate/transition is clearly output-related, but exact necessity remains unresolved
> The `0x25EE1` gate is an **edge-triggered descriptor-population** routine: when its predicate `r11`
> (bit7 of `byte@0xB000_0C06`) rises 0→1, it calls `0x13292`, which fills the global struct `0x252AD0`
> (a buffer/rate descriptor) consumed by the DTS output path. The output-enable MMIO (`0x870`/`0x0854`)
> is written **unconditionally** by the DTS helper *before* the gate, so `r11=0` does **not** kill all output —
> it only suppresses the `0x252AD0` population, for which the consumer has a default fallback.
> Whether that suppressed population is *required* for valid DTS output is **UNRESOLVED** at the static level
> (needs `+0x18` semantics, the `0x109595` consumer's decision, `word@0x2512C4` value, and the hardware meaning
> of `0xB000_0C06` bit7).

**Discipline note:** R30-B (runtime) showed `r11=0` during DTS playback. This R31 does **not** conclude "failure"
from that alone. Per the R31 brief, B is *not* chosen merely because the branch skips `0x13292`.

---

## 1. Evidence labels (used throughout)

- **VERIFIED STATIC** — directly demonstrated by decoded instructions / `flow[0]` control flow.
- **STRONG STATIC INFERENCE** — very likely from multiple static observations, but semantic mapping incomplete.
- **UNRESOLVED** — cannot be determined from the available image.

---

## 2. Phase A — Reconstruction of the `0x25EE1` state machine (VERIFIED STATIC)

Source: `decoded_25EE1.txt` (74 instr, `flow[0]`). The function takes `r3 = this` (SND state object `0x4D17C0`),
sets `r10 = r3` (0x25F24), and operates on fields `+0x54 / +0x58 / +0xB8 / +0xBC` (plus `+0xB0`, `+0x94`, `+0x50`).

### 2.1 Predicate (input)
```
r12 = byte@0xB000_0C06 ; r11 = r12 >> 7            # input #1 = bit7(0x0C06)
r23 = signed byte@0xB000_0x0015 ; if (r23 < 0) r11 = 0   # input #2
r24 = word@0x2512C4 ; if (r24 == 0) r11 = 0        # input #3 (unreadable at runtime; 16-bit truncation)
```

### 2.2 Field transitions (every static WRITE/READ inside `0x25EE1`)
| Offset | READ | WRITE | Value / bitmask written | Meaning (no semantic name assigned) |
|--------|------|-------|------------------------|--------------------------------------|
| `+0xBC` | 0x25F17 `lwz r23,0xbc(r3)` | 0x25F32 `sw_0 0xbc(r10),r11` | **= r11** (predicate) | cached copy of `r11` (old value read for comparison) |
| `+0xB8` | — | 0x25F5A `sw_0 0xb8(r10),r11` | **= r11** | second cached copy of `r11` |
| `+0x54` | 0x25F36 `lwz r23,0x54(r10)` | 0x25F42 `sw 0x54(r10),r12` | **= byte@0x0C06 & 0x40** (bit6, NOT bit7) | cached status bit6 |
| `+0x58` | — | 0x25F2F / 0x25F45 / 0x25F71 `sw 0x58(r10),1` | **= 1** | "changed/dirty" flag set when +0x54 or +0xBC changes |
| `+0xB0` | — | 0x25F91 `sw_0 0xb0(r3),r0` | **= 0** | set 0 **only on the gate-PASS path** |
| `+0x94` | 0x25F48 `lwz r23,0x94(r10)` | — | used as table index | index into jump table at `0x18CCC` |
| `+0x50` | 0x25F68 `lwz r23,0x50(r10)` | — | checked `== 0` | secondary guard |

### 2.3 Control flow / state transitions
1. `old = *(+0xBC)` (0x25F17).
2. `if (old == r11) goto 0x25F36` (0x25F26) → **no transition** (merge/state-sync only).
3. `if (r11 != 0) goto 0x25F91` (0x25F2A) → **gate PASS**.
4. gate-PASS `0x25F91`: `*(+0xB0) = 0`; set up args; `jal 0x13292`; then `j 0x25F2D`.
5. `r11 == 0` fall-through `0x25F2D`: `*(+0x58)=1; *(+0xBC)=r11` (=0).  *(No `0x13292` call.)*
6. Merge `0x25F36`: `*(+0x54) = bit6(byte@0x0C06)`; `*(+0x58)=1` if changed; then `+0x94` table lookup.

### 2.4 Critical reconstruction result
- **`+0xBC` is a cached copy of `r11`, not independent state.** It is read as `old` and compared at 0x25F26
  (`beq r23, r11`); the comparison is *"has the predicate changed since last call?"* → **edge detection.**
- The decision `0x25F26` (skip if `old==r11`) means: **only a CHANGE in `r11` triggers action.** Combined with
  `0x25F2A` (`r11 != 0` → gate pass), the `0x13292` call fires **only on the rising edge (0→1) of `r11`**.
  - old=0, r11=1 → rising edge → `0x13292` called.
  - old=1, r11=1 → steady → skipped (no repeat).
  - old=0, r11=0 → steady → skipped (no action).
  - old=1, r11=0 → falling edge → `0x25F2D` only (cache updated, no `0x13292`).

**Answer to Phase A key question:** `+0xBC` is **a cached copy of `r11`** (also `+0xB8` duplicates it). Its *mismatch*
with the current `r11` is meaningful — it is the edge-trigger that gates the `0x13292` call. It is not an
independent state whose value carries separate meaning. **VERIFIED STATIC.**

---

## 3. Phase B — Uses of the DTS state object `0x4D17C0` (STRONG STATIC INFERENCE)

Object base: `0x4D17C0` (built `bg.movhi rX,0x4d ; bg.addi rX,rX,0x17c0` at `0xCEC8` in `0xCC9B`, `0xD2C4` in `0xCF86`;
passed as `r3` into `0x25EE1`). It is the **shared SND output-state object**, used by far more than the two DTS helpers:

| Function (region) | Accesses to object fields | Role |
|-------------------|--------------------------|------|
| `0x25EE1` (gate) | +0xBC r/w, +0xB8 w, +0x54 r/w, +0x58 w, +0xB0 w (pass), +0x94 r | gate/state machine |
| `0x26060` (0x25) | +0xBC r/w, +0xB8 w, +0x54 r/w, +0x58 w, +0xB0 w | **parallel gate-like** state machine (same object) |
| `0x2619F`–`0x26621` (0x25) | +0xB0 w, +0x58 w, +0xB8 w, +0x54 w, +0x94 r | state setters |
| `0x26D32` / `0x26D57` (0x25) | +0x54 r, +0x58 r | read cached status |
| `0x27C79`/`0x27C8F`/`0x27E58` (0x27) | +0x54 r, +0x58 r, +0x94 r | higher-level output/enable decision |
| `0x27F5B`/`0x27F86`/`0x27F9F`/`0x27FB9` (0x27) | +0xB0 r, +0xB4 r, +0xB8 r, +0xBC r → each passed to `jal 0x109595` | **consumer of gate's cached predicate** |

**Conclusion (Phase B):** the gate's cached predicate (`+0xBC`, `+0xB8`) and pass-flag (`+0xB0`) are **read by
higher-level decision functions** (`0x27xxx` → `0x109595`). So the gate state *participates* in output/enable
decisions. What `0x109595` does with `+0xBC=0` specifically (disable vs. benign) is **UNRESOLVED** (that callee
was not decoded this phase). STRONG STATIC INFERENCE that the gate state is consumed by output logic; necessity UNRESOLVED.

---

## 4. Phase C — `0x13292` producer / consumer (STRONG STATIC INFERENCE)

Treat `0x252AD0` as a **data structure** (not assumed DMA descriptor).

### 4.1 What `0x13292` writes (struct at `r10`, arg `r3=0x252AD0` from `0x25FA3`)
Constants passed in (`0x25F95`–`0x25FB4`): `r4=0x10F000`, `r5=0x96000`, `r6=8`, `r7=0x100`, `r8=4`.
Fields written (`decoded_13_115.txt`, `0x13292` body):
| Offset | Value written | Source |
|--------|---------------|--------|
| +0x00 | `0x10F000` | saved r12 (=r4) |
| +0x04 | `0x1A5000` | r11 + r12 |
| +0x08 | `0x96000` | saved r11 (=r5) |
| +0x0C | `0x10F000` | saved r12 |
| +0x10 | `0x96000` | saved r11 |
| +0x14 | `0` | |
| +0x18 | **`0x100`** | saved r25 (r7) |
| +0x1C | `0x100` | r7 |
| +0x20 | **`4`** | saved r8 |
| +0x24 | `8` | r6 |
| +0x28 | (computed) | |
| +0x2C | (computed) | |
| +0x30 | **`0x8000`** | `ori r23,0x8000` |
| +0x34 | **`0x100`** | `ori r23,0x100` |
| +0x38 | `0` | |
| +0x3C | `0` | |

Note: `0x13292` calls `0x10786E` at 0x13338/0x133A7 with **different args** (`r3=0x128A7`/`0x128419`, NOT the struct),
so `0x13292` does **not** pass `0x252AD0` into `0x10786E`. The struct is populated for **later** consumption.

### 4.2 Who reads `0x252AD0` (consumers) — binary + dump cross-reference
`bg.addi rX,rX,0x2ad0` (builds `0x252AD0`) appears at:
- `0x25FA3` — inside `0x13292` (the **producer**).
- `0xCF64` — inside DTS helper `0xCC9B` (reads the struct).
- `0xD323` — inside DTS helper `0xCF86`: `r23=0x252AD0; r24=*(+0x18)`; if `0` → set `+0x18=8` (default); `r25=*(+0x20)`; if `0` → `+0x20=4`. Then proceeds to fixed-point math + write to a `0xA868` global + cache flush.
- `0x261AA`, `0x26233`, `0x263CB`, `0x263DA`, `0x263EF` — 0x26-region functions building `0x252AD0` and reading it.

**Conclusion (Phase C):** `0x13292` prepares the global struct `0x252AD0` (buffer/rate descriptor with constants
`0x8000`/`0x100`/`0x100`/`4`/`8`); it is **consumed by the DTS output path itself** (`0xCF64`/`0xD323` inside the
DTS helpers) and by 0x26-region routines. The consumer `0xD323` uses a **default** (`+0x18=8`) when the struct is
unpopulated. This is a **buffer/rate/state descriptor**, not proven to be a DMA descriptor. STRONG STATIC INFERENCE.

**Answer to Q4:** consumers = `0xCF64` (0xCC9B), `0xD323` (0xCF86), and 0x26-region funcs `0x261AA/0x26233/0x263CB/0x263DA/0x263EF`.

---

## 5. Phase D — Shared finalizer usage (VERIFIED STATIC for topology; genericity VERIFIED)

Caller matrix (from `flow[0]`):
| Function | Codec/path | Conditions | Purpose |
|----------|-----------|------------|---------|
| `0x10786E` | **AC3** (0xCBEC `jal`, 0xCB4C `j`) AND **DTS** (0x13338, 0x133A7 inside `0x13292`) AND many 0x13–0x115 / 0x25 funcs | unconditional wrapper | calls `0x107860` then `0x115693` |
| `0x107860` | via `0x10786E` only | — | if r3≠0 clear byte@(0x1BE77); `j 0x107379` |
| `0x107379` | via `0x107860` only | — | final finalizer |
| `0x115693` | **only via `0x10786E`** | — | generic "commit output" |

- `0x115693` is **generic / shared**: it is reached only through the shared wrapper `0x10786E`, which both codecs
  (and many other routines) call. It performs a buffer copy/flush loop and writes `0xB000_0020 = 0x2` and
  `0xB000_003E = <val>` (MMIO `sh_1` stores). It is **not** DTS-specific and is **not** exclusively tied to
  output *activation* — it is a commit step used by the common path.
- Therefore the DTS-specific code does **not** live in `0x115693`/`0x107379`; those are shared.

**Answer to Q5:** `0x115693` establishes a buffer copy/flush + writes `0xB000_0020=0x2`, `0xB000_003E=<val>` — a
generic output-commit. VERIFIED STATIC (topology + MMIO stores visible in `decoded_13_115.txt`).

---

## 6. Phase E — Writers of `0xB000_0C06` (VERIFIED STATIC — binary-wide scan)

**Method:** scanned the entire `snd_full.bin` (0x1C1330 bytes) for the instruction forms that build `0xB000_0C06`
(`bg.ori rX,rY,0xc06` = bytes `CB ?? 0C 06` BE; `bg.addi rX,rY,0xc06` = `FC ?? 0C 06` BE). Then classified each
reference's follow-up instruction (read `lbz` op 0x11 vs. store `sb`/`sh`/`sw`/`sw_0`).

**Result: exactly 3 references to `...0C06` exist in the whole image:**
1. `0x25EF2` (`bg.ori r24,r23,0xc06`) → next `bn.lbz r12,0x0(r24)` → **READ** (the gate's read). VERIFIED read.
2. `0x262E1` (`bg.ori r25,r23,0xc06`) → address built, then `bg.sw_0 0x94(r10),r26` (stores into `this+0x94`,
   **not** into `0xc06`) → **READ** (consumer feeding the gate's `+0x94` table index). VERIFIED read.
3. `0x114C23` (`bg.addi r4,r4,0xc06`) → r4 is a **function argument** (base caller-dependent); next instruction is
   `bg.movhi` (unrelated); no store to `r4` in vicinity. Even if it resolves to `0xB000_0C06`, it is a **read**
   (no `sb`/`sh`/`sw`/`sw_0` to the address anywhere in the image).

**No occurrence is a store to `0x...0C06`.** No `sb`/`sh`/`sw`/`sw_0`/`opcode_2E` writes `0xB000_0C06` anywhere.

> **Conclusion (Phase E):** **No firmware writer to bit7 of `0xB000_0C06` was found in the searched image.**
> `0xB000_0C06` is a **read-only status register**; its bit7 is produced by **hardware** (or another subsystem)
> and merely *read* by firmware. VERIFIED STATIC (entire-image scan).

---

## 7. Phase F — DTS sequence before/after the gate (VERIFIED STATIC + STRONG INFERENCE)

Full path reconstructed from `decoded_CF86.txt` (`0xCF86` DTS helper) and `decoded_25EE1.txt`:
```
0xCF86 (DTS private helper)
  ├─ 0xD09A: writes 0xB000_0870  (output-enable)  ← UNCONDITIONAL, BEFORE the gate
  ├─ 0xCFFA: read-modify-write 0xB000_0854 / clear 0xB000_0850  ← UNCONDITIONAL, BEFORE the gate
  ├─ 0xD2C0: r3 = 0x4D17C0 (state object)
  ├─ 0xD2C8: jal 0x25EE1            (THE GATE)
  │     r11 = bit7(byte@0xB000_0C06) AND (signed byte@0x0015≥0) AND (word@0x2512C4≠0)
  │     r11=0 → no 0x13292 (state-sync only)
  │     r11≠0 → 0x13292  (fills 0x252AD0)  → 0x10786E → 0x115693
  ├─ 0x25EE1 returns r3 = (word@0x2512C4 != 0) ? 1 : 0   ← driven by input #3, NOT by r11
  ├─ 0xD2CC: bnei r3,0 → 0xD31F (consumer) ; else 0xD2CF (alt path, 0x255CA8)
  └─ 0xD31F: r23=0x252AD0; reads +0x18 (default 8 if unpopulated), +0x20 (default 4);
            fixed-point math → writes 0xA868 global + cache flush (0xD4C8)
```

**What differs between the two gate paths:** the gate-PASS path additionally runs `0x13292` (populating `0x252AD0`
with `+0x18=0x100`, etc.); the `r11=0` path skips it and the consumer uses default/stale values. The
output-enable MMIO (`0x870`/`0x0854`) is written in **both** paths.

**Answer to Q6 discipline:** the `r11=0` path is the **edge-trigger "no transition"** path, not an explicit error
path. It is only a problem if the rising edge (bit7(0x0C06) 0→1) never occurs — which, per Phase E, depends on
hardware status, and per R30 runtime was observed as 0 during DTS playback.

---

## 8. Phase G — DTS vs AC3 equivalent mechanisms (STRONG STATIC INFERENCE)

| Step | AC3 private helper | DTS private helper |
|------|-------------------|--------------------|
| output-enable write | `0x0854`/`0x850` (in helper, before any wrapper) | `0x870`/`0x86C` (in helper, before gate) |
| gate | **none** — goes straight to wrapper | `0x25EE1` (gate) |
| descriptor population | **none** (`0x252AD0` not populated by AC3 path) | `0x13292` fills `0x252AD0` (only if gate passes) |
| shared finalizer | `0x10786E` → `0x107860`→`0x107379` + `0x115693` | same `0x10786E`→`0x115693` |
| `0x252AD0` consumed? | not via this path | yes (`0xCF64`/`0xD323`/`0x26-region`) |

**Answer to Q7:** The functionality DTS obtains **only** through its extra gated transition is the population of
the `0x252AD0` buffer/rate descriptor (via `0x13292`). AC3 reaches the shared finalizer directly and does not use
`0x252AD0` on this path. So DTS's output path depends on a DTS-specific descriptor that AC3 does not need (or obtains
elsewhere). STRONG STATIC INFERENCE.

---

## 9. Answers to the 8 required R31 questions

**Q1. What exactly is `r11` controlling in `0x25EE1`?**
`r11` is the gate predicate `bit7(byte@0xB000_0C06) AND (signed byte@0x0015≥0) AND (word@0x2512C4≠0)`. It controls
whether `0x13292` (struct `0x252AD0` population) is executed. The comparison `*(+0xBC)==r11` makes the action
**edge-triggered** (fires on rising edge 0→1). VERIFIED STATIC.

**Q2. What is the semantic relationship between `r11` and `this+0xBC`?**
`+0xBC` (and `+0xB8`) is a **cached copy of `r11`**, written each call, read back as `old` for the
`beq r23,r11` skip-test. Its *mismatch* is meaningful (edge detection); it carries no independent state. VERIFIED STATIC.

**Q3. What does `0x13292` actually prepare?**
It fills the global struct `0x252AD0` with fixed constants (`+0x30=0x8000`, `+0x34=0x100`, `+0x18=0x100`,
`+0x20=4`, `+0x24=8`, `0x10F000/0x96000/0x1A5000` pointers/sizes). It is a **buffer/rate/state descriptor**, not
proven to be a DMA descriptor. STRONG STATIC INFERENCE.

**Q4. Who consumes the state prepared by `0x13292`?**
`0xCF64` (0xCC9B), `0xD323` (0xCF86) — the DTS helpers themselves — plus 0x26-region functions
`0x261AA/0x26233/0x263CB/0x263DA/0x263EF`. The consumer `0xD323` uses a default (`+0x18=8`) when unpopulated.
STRONG STATIC INFERENCE.

**Q5. What does `0x115693` actually establish?**
A generic output-commit: buffer copy/flush loop + `sh_1` writes `0xB000_0020=0x2` and `0xB000_003E=<val>`.
Shared by both codecs (reached only via `0x10786E`). VERIFIED STATIC (topology + MMIO).

**Q6. Who writes `0xB000_0C06` bit7?**
**No firmware writer** to `0xB000_0C06` exists in the entire image (3 references total, all reads). bit7 is
**hardware-produced status**, only read by firmware. VERIFIED STATIC (binary-wide scan).

**Q7. What concrete functionality differs between AC3 and DTS?**
DTS obtains `0x252AD0` descriptor population (via gated `0x13292`) that AC3 bypasses; AC3 reaches the shared
finalizer directly. STRONG STATIC INFERENCE.

**Q8. Is `r11=0` demonstrably a failure state, a normal synchronized state, or still unresolved?**
**Still unresolved → R31-C.** `r11=0` is the edge-trigger "no transition" path; it suppresses `0x13292` (struct
population) but the output-enable MMIO is written unconditionally, and the consumer has a default fallback. Whether
the suppressed population is *required* for valid DTS output depends on `+0x18`/`+0x20` semantics, what `0x109595`
does with the cached `+0xBC`, the value of `word@0x2512C4`, and the hardware meaning of `0xB000_0C06` bit7 — all
UNRESOLVED statically. R30 runtime (bit7=0 during DTS) is directional only and not promoted to causal proof.

---

## 10. Final classification

**R31-C** — *The gate/transition is clearly output-related, but exact necessity remains unresolved.*

Rationale:
- The gate is demonstrably **output-related**: it populates `0x252AD0` (consumed by the DTS output path) and its
  cached predicate (`+0xBC`) is read by higher-level output-decision functions (`0x27xxx`→`0x109595`). VERIFIED/STRONG.
- But selecting **B** ("demonstrably suppresses a *required* DTS operation") is not justified: the suppressed
  operation (`0x13292`→`0x252AD0`) has a **default fallback** in its consumer, and the output-enable MMIO is written
  regardless. The *requirement* is UNRESOLVED.
- Selecting **A** ("demonstrably normal/steady, gate is not the blocker") is also not justified: the gate's predicate
  bit7 is hardware status (Phase E) observed 0 during DTS (R30), so the rising edge provably never occurs and
  `0x252AD0` is provably never populated on the DTS path — which *could* be the defect, but the necessity is unproven.
- Selecting **D** is too weak: the *state semantics* of `+0xBC` (cached r11) and the gate topology ARE resolved;
  only the *consequential necessity* is unresolved.

Hence **R31-C** is the disciplined classification.

---

## 11. Residual / UNRESOLVED items (for a possible R32)

1. `0x109595` — what it does with the gate's cached `+0xBC`/`+0xB8`/`+0xB0` (disable vs. benign). Decode required.
2. `0x252AD0` field semantics — is `+0x18=0x100` vs default `8` a buffer size / channel count / rate flag that
   actually changes emitted output? Needs datasheet or further trace of the `0xA868` global consumer + `0x26-region`.
3. `word@0x2512C4` (gate input #3) — value during DTS (16-bit-truncation-unreadable at runtime); determines the
   post-gate consumer dispatch (`0xD31F` vs `0xD2CF`).
4. `0x114C23` reference — confirm base register (r4 is a function argument); binary scan shows no store regardless.
5. `0x26060` — the parallel gate-like function on the same object; its trigger/relationship to `0x25EE1`.

---

## Appendix A — Files used (all static, from R29 work)
- `r29_work/decoded_25EE1.txt` — `0x25EE1` body (74 instr).
- `r29_work/decoded_13_115.txt` — `0x13292` (106), `0x10786E` (20), `0x115693` (108).
- `r29_work/decoded_107860.txt`, `decoded_10786E.txt`, `decoded_CF86.txt` — helper/finalizer bodies.
- `r29_work/ghidra_dump_25.txt`, `ghidra_dump_13_115.txt`, `ghidra_dump_CC00_F00.txt` — `flow[0]` region dumps.
- `aeon_validate/r27_work/snd_full.bin` — full binary (0x1C1330) for the Phase E binary scan.

## Appendix B — Reproduce the Phase E binary scan
```python
data=open('aeon_validate/r27_work/snd_full.bin','rb').read()   # 0x1C1330 bytes, BE-stored
hits=[i for i in range(len(data)-3) if data[i] in (0xCB,0xFC) and data[i+2]==0x0C and data[i+3]==0x06]
# then classify the next instruction (lbz=0x11 read; store opcodes = write)
```
Result: 3 hits, 0 stores → no firmware writer to `0xB000_0C06`.
