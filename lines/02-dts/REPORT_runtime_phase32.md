# R32 — Static Resolution of the `0x25EE1` Return Path, `0x252AD0` Consumers, and Gate-State Propagation

**Module:** `mst_snd_r2_MS12V22.bin` (extracted `snd_full.bin`) — MStar/MedTek AEON SND audio firmware
**CPU:** `aeon:LE:32:default` · `flow[0]` authoritative · `opcode_2E` = MMIO-touch
**Prior phases:** R29 (R29-B) · R30 (runtime: gate-block during DTS → R30-B) · R31 (static: edge-triggered descriptor gate → R31-C)
**This phase:** R32 — **pure static analysis only.** No runtime, no patch, no device/EDID/AUTH/settings changes.
**R30 usage:** treated as *directional evidence only*; never promoted to causal proof.
**Deliverable date:** 2026-09-09

---

## 0. Headline classification

> ### R32 = **R32-C** — the actual important DTS decision is the **separate `0x25EE1` return-value path**, not the `r11 → 0x13292` transition.

The corrected architecture (per R31 + R32) is:

```text
DTS helper (0xCC9B / 0xCF86)
    ↓
write output-enable 0x086C / 0x0870          (UNCONDITIONAL, before the gate)
    ↓
jal 0x25EE1  ── produces TWO INDEPENDENT outputs:
    │
    ├─(A) SIDE EFFECT:  r11 = bit7(0x0C06) & (0x0015≥0) & (0x2512C4≠0)
    │       edge-detected via *(+0xBC); rising edge 0→1  →  jal 0x13292
    │                                                     →  fills 0x252AD0
    │
    └─(B) RETURN VALUE: r3 = (table[*(this+0x94)+0x12]==1) AND (word@0x2512C4≠0) ? 1 : 0
            ↓
        caller branch:
            r3 != 0  →  0x252AD0 consumer   (0xCF60 in 0xCC9B / 0xD31F in 0xCF86)
            r3 == 0  →  0x255CA8 alternate  (0xCED4 in 0xCC9B / 0xD2CF in 0xCF86)
```

**(B) is the higher-level decision**: it selects *which configuration structure* drives the DTS computation.
**(A)** only determines whether `0x252AD0` holds the "rich" populated descriptor or a default — and it only
matters *inside* the `r3 != 0` branch. If `r3 == 0`, the caller never reads `0x252AD0` at all, making (A) moot.

---

## 1. Evidence labels

**VERIFIED STATIC** · **STRONG STATIC INFERENCE** · **UNRESOLVED** (as defined in R31).

---

## 2. Phase A — The return value of `0x25EE1` (VERIFIED STATIC)

### 2.1 Exact return expression
From `decoded_25EE1.txt`:
```
0x25F63  bt.movi r3,0x0                          ; r3 = 0   (default return)
0x25F48  bg.lwz r23,0x94(r10)                    ; idx = *(this+0x94)
0x25F54  bn.addi r23,r23,0x12                    ; idx += 0x12
0x25F57  bn.slli r23,r23,0x2                     ; idx <<= 2
0x25F5E  bt.add r23,r24                          ; + table base (0x18CCC)
0x25F60  bn.lwz r23,0x0(r23)                     ; r23 = table[*(this+0x94)+0x12]
0x25F65  bn.beqi r23,0x1,0x25f83                 ; if (table==1) goto 0x25F83
...
0x25F83  bg.lwz r23,0x130(r13)                   ; r23 = word@0x2512C4   (r13 = 0x251194, +0x130)
0x25F87  bn.sfnei r23,0x0                        ; if (word@0x2512C4 != 0)
0x25F8A  bn.cmovsi_2 r3,r0,0x1                   ;   r3 = 1
0x25F8D  bg.j 0x25f68
```

> **RETURN = `r3` = ( `table[*(this+0x94)+0x12] == 1` ) AND ( `word@0x2512C4 != 0` ) ? 1 : 0**
> **VERIFIED STATIC.** Note: the return is driven by `word@0x2512C4` (gate input #3) and the `+0x94` table
> index — **NOT** by `r11` and **NOT** by `bit7(0xB000_0C06)`.

### 2.2 Caller 1 — `0xCF86` (DTS helper #2)
```
0xD2C0  bg.movhi r3,0x4d
0xD2C4  bg.addi  r3,r3,0x17c0        ; r3 = 0x4D17C0 (state object)
0xD2C8  bg.jal 0x25ee1               ; THE GATE CALL
0xD2CC  bn.bnei r3,0x0,0xd31f        ; return != 0  ->  0xD31F   (0x252AD0 consumer)
0xD2CF  (return == 0)  bg.movhi r23,0x25
0xD2D3  bg.addi r23,r23,0x5ca8       ; r23 = 0x255CA8  (ALTERNATE config struct)
0xD2D7  bn.lwz r24,0x18(r23)         ; read 0x255CA8 +0x18
0xD2E8  bt.movi r25,0x2              ; default +0x20 = 2
0xD2ED  bn.sw 0x20(r23),r25
0xD2F0  ... (common fixed-point math)
```

### 2.3 Caller 2 — `0xCC9B` (DTS helper #1)
```
0xCEC4  bg.movhi r3,0x4d
0xCEC8  bg.addi  r3,r3,0x17c0        ; r3 = 0x4D17C0
0xCECC  bg.jal 0x25ee1               ; THE GATE CALL
0xCED0  bg.bnei r3,0x0,0xcf60        ; return != 0  ->  0xCF60   (0x252AD0 consumer)
0xCED4  (return == 0)  bg.movhi r23,0x25
0xCED8  bg.addi r23,r23,0x5ca8       ; r23 = 0x255CA8  (ALTERNATE config struct)
0xCEDC  bn.lwz r24,0x18(r23)         ; read 0x255CA8 +0x18
0xCEE2  bt.movi r24,0xa              ; default +0x18 = 10
0xCEED  bt.movi r25,0x2              ; default +0x20 = 2
0xCEF5  ... (common fixed-point math)
```

### 2.4 Branch semantics (VERIFIED STATIC)
| Return | `0xCC9B` | `0xCF86` | Config struct used | Reads `0x252AD0`? |
|--------|----------|----------|--------------------|-------------------|
| **!= 0** | `0xCF60` → `0xCF64: r23=0x252AD0` | `0xD31F` → `0xD323: r23=0x252AD0` | **`0x252AD0`** | **YES** |
| **== 0** | `0xCED4` → `0x255CA8` | `0xD2CF` → `0x255CA8` | **`0x255CA8`** | **NO** |

**Answer to Phase A's question:** *Yes* — the return value controls a **more consequential** decision than the
`r11 → 0x13292` transition. The return value **selects the entire configuration path** (`0x252AD0` vs `0x255CA8`).
The `r11 → 0x13292` transition only supplies data *within* the `return != 0` branch. **VERIFIED STATIC.**

Consequence for the R30 model: `r11 = 0` (bit7(`0xB000_0C06`) = 0) only matters if `word@0x2512C4 != 0`
(return = 1). If `word@0x2512C4 == 0`, the caller takes the `0x255CA8` path and `0x13292` is irrelevant.

---

## 3. Phase B — Resolution of `0xD323` (the `0x252AD0` consumer) (STRONG STATIC INFERENCE)

Reconstructed from `decoded_CF86.txt` (`0xD31F` → `0xD340` → `0xD2F0` → `0xD15E`):

```
0xD31F  bg.movhi r23,0x25
0xD323  bg.addi  r23,r23,0x2ad0      ; r23 = 0x252AD0
0xD327  bn.lwz r24,0x18(r23)         ; r24 = *(+0x18)
0xD32A  bn.bnei r24,0x0,0xd332       ; if != 0 skip
0xD32D  bt.movi r24,0x8              ; DEFAULT: +0x18 = 8
0xD332  bn.lwz r25,0x20(r23)         ; r25 = *(+0x20)
0xD335  bn.bnei r25,0x0,0xd2f0       ; if != 0 skip
0xD338  bt.movi r25,0x4              ; DEFAULT: +0x20 = 4
0xD340  bg.j 0xd2f0
        --- common math ---
0xD2F0  bn.muls r24,r24,r25          ; r24 = (+0x18) * (+0x20)
0xD2F3  bn.lwz r23,0x14(r23)         ; r23 = *(+0x14)
0xD2F6  bg.muli_maybe r24,r24,0x30   ; r24 = product * 0x30   (x48)
0xD2FA  bg.lwz r26,0x114(r10)        ; r26 = *(0x114(r10))
        ... division r25 = r23 / r24 (operand order per AEON divu) ...
        ... r24 = *(0x253364) + r26 + *(0x118(r10)) + r25 ...
0xD31B  bg.j 0xd15e
0xD15E  bg.sw_0 0x868(r23),r24       ; r23 = 0x1 - 0x6000 = 0xA000  ->  WRITE 0xA868 = r24
0xD174  bg.j 0xd4c8                  ; cache flush (0xD4C8)
```

### 3.1 Populated vs default values
| Field | Populated by `0x13292` | Default (not populated) |
|-------|------------------------|--------------------------|
| `+0x18` | **`0x100`** (256) | **`8`** |
| `+0x20` | `4` | `4` (identical either way) |
| `+0x14` | `0` (written by `0x13292` at `0x132E5`) | stale/uninitialized |

### 3.2 Effect on arithmetic
- Divisor term = `((+0x18) × (+0x20)) × 0x30`:
  - populated: `(0x100 × 4) × 0x30 = 0x400 × 0x30 = **0xC000**` (49 152)
  - default: `(0x8 × 4) × 0x30 = 0x20 × 0x30 = **0x600**` (1 536)
  - → **32× difference in the divisor**, hence 32× difference in the quotient term.
- The quotient feeds the value written to **`0xA868`** (a data global, followed by a cache-flush `0xD4C8` —
  i.e. a value intended for another agent/DMA/ISR to observe).

### 3.3 Honest caveat (do NOT overclaim)
The dividend is **`+0x14`**, which `0x13292` explicitly writes to **0** (`0x132E5 bn.sw 0x14(r10),r0`).
If `+0x14 == 0` at consumption time, the quotient is `0` regardless of the divisor, and the 32× divisor
difference produces **no** difference in `0xA868`. Whether `+0x14` is subsequently set by other code
(between `0x13292` and consumption) is **UNRESOLVED**.
→ **STRONG STATIC INFERENCE** that `+0x18`/`+0x20` are buffer/timing parameters feeding a computed value
written to `0xA868`; **UNRESOLVED** whether the populated-vs-default difference actually changes emitted audio.

**Semantic labels are deliberately withheld** (per Phase B instruction) until the role is established: the
fields behave as **multiplicative scale factors** into a divisor of a timing/buffer computation whose result
is published to a shared global (`0xA868`) — consistent with a buffer-size/period parameter, but **not proven**.

---

## 4. Phase C — Resolution of `0xCF64` (STRONG STATIC INFERENCE)

`0xCF64` is `0xCC9B`'s `0x252AD0` consumer (`decoded_CC9B.txt`):
```
0xCF60  bg.movhi r23,0x25
0xCF64  bg.addi  r23,r23,0x2ad0      ; r23 = 0x252AD0
0xCF68  bn.lwz r24,0x18(r23)         ; read +0x18
0xCF6B  bn.bnei r24,0x0,0xcf73
0xCF6E  bt.movi r24,0x8              ; DEFAULT +0x18 = 8
0xCF73  bn.lwz r25,0x20(r23)         ; read +0x20
0xCF76  bg.bnei r25,0x0,0xcef5
0xCF7A  bt.movi r25,0x4              ; DEFAULT +0x20 = 4
0xCF82  bg.j 0xcef5                  ; -> common math
```
Structurally **identical** to `0xD323`. One per-helper difference: `0xCC9B` publishes its result to
**`0xA86C`** (`0xCE26 bg.sw_0 0x86c(r23),r24` with `r23 = 0xA000`), whereas `0xCF86` publishes to
**`0xA868`** (`0xD15E`). So the two DTS helpers write **different global slots** (`0xA868` vs `0xA86C`),
consistent with two output instances/ports.

**Data-path difference (`0x13292` executed vs not):** identical to `0xD323` — `+0x18` is `0x100` vs `8`
(default), `+0x20` is `4` either way; the divisor differs 32×; the published global differs accordingly.
**STRONG STATIC INFERENCE** (same `+0x14` caveat as §3.3).

---

## 5. Phase D — The remaining `0x252AD0` consumers (`0x26` region)

Decoded `0x261AA`, `0x26233`, `0x263CB`, `0x263DA`, `0x263EF` (`decoded_26region.txt`):

| Function | Relationship to `0x252AD0` | Role |
|----------|---------------------------|------|
| `0x261AA` | `bg.addi r3,r3,0x2ad0` → builds `0x252AD0` | higher-level orchestrator |
| `0x26233` | `bg.addi r3,r3,0x2ad0` → builds `0x252AD0` | higher-level orchestrator |
| `0x263CB` | `bg.addi r3,r14,0x2ad0` → builds `0x252AD0` | orchestrator; callees include `0xCC9B`, `0xCF86`, `0x10786E`, `0x1078A9`, `0x107C09`, `0xC55E`, `0xC731` |
| `0x263DA` | `bg.addi r3,r14,0x2ad0` → builds `0x252AD0` | same family |
| `0x263EF` | `bg.addi r14,r14,0x2ad0` → builds `0x252AD0` | same family |

All five construct the `0x252AD0` base and are **DTS output-related orchestrators** (their callee sets
include the DTS private helpers `0xCC9B`/`0xCF86`, the shared finalizer `0x10786E`, and the output
reset/enable helpers `0xC55E`/`0xC731`). Within their decoded bodies the visible `+0x20` accesses are via
`r11` (`bn.lwz r23,0x20(r11)` / `bn.sw 0x20(r11),r23`), i.e. the descriptor is reached through a register
alias rather than a direct literal — so the exact field mapping here is **STRONG STATIC INFERENCE**, not
fully traced (per Phase D instruction: stop once the relationship is established).

**Established:** `0x252AD0` is a **shared DTS output-configuration descriptor** referenced by the gate helper,
both DTS private helpers, and the `0x26`-region orchestrators. **STRONG STATIC INFERENCE.**

---

## 6. Phase E — `0x109595` (the gate-state consumer) — PARTIALLY UNRESOLVED

### 6.1 What is established (VERIFIED STATIC — call sites)
The `0x27xxx` higher-level decision functions read the SND state object's gate fields and pass each into
`0x109595` as **`r3`**:
- `0x27F5B  bg.lwz r3,0xb0(r11)` → `jal 0x109595`   (`+0xB0` — the gate-PASS flag, set 0 on pass)
- `0x27F86  bg.lwz r3,0xb4(r11)` → `jal 0x109595`   (`+0xB4`)
- `0x27F9F  bg.lwz r3,0xb8(r11)` → `jal 0x109595`   (`+0xB8` — cached `r11`)
- `0x27FB9  bg.lwz r3,0xbc(r11)` → `jal 0x109595`   (`+0xBC` — cached `r11`, the edge-detection field)

So `0x109595` **receives the gate's cached predicate and pass-flag** from the higher-level output-decision layer.

### 6.2 What is NOT resolved
`0x109595` resides at **`0x109595`**, inside the region **`0x108000–0x12FFF`**, which is a **gap in the
available text dumps** (`ghidra_dump_107.txt` ends at `0x107FFF`; `ghidra_dump_13_115.txt` starts at
`0x13000`). Attempts to generate a fresh dump were blocked in this environment:
- `analyzeHeadless -import` on a new project → `No load spec found for import file` (raw binary needs the
  AEON loader the original project had).
- Pre-existing `ghidra_snd4/5/6` projects are **empty shells** (no imported program: `Requested project
  program file(s) not found`).
- `-loader RawLoader` → `Invalid loader name`; `-loader "Raw Binary Loader"` → `Invalid loader name`.

### 6.3 Answer to Phase E's key question
> *Does `+0xBC = 0` eventually suppress an actual DTS output operation?*

**UNRESOLVED.** The gate's cached predicate **is** propagated (`+0xB8`/`+0xBC` → `0x109595`) into a
higher-level decision helper, so it *participates* in output decision-making — but whether `+0xBC = 0`
suppresses an encoder/SDO/TX operation cannot be established without decoding `0x109595`'s body.
**Do not infer "suppresses" from the argument alone.** (Phase E instruction: "Do not infer from the argument name.")

---

## 7. Phase F — `0xB000_0C06` revisited conservatively

### 7.1 Rephrased result (as instructed)
Previously phrased as "no firmware writer exists anywhere". The correct, defensible statement is:

> **No direct firmware store to `0xB000_0C06` was found among the searched address-construction forms.**
> A whole-image scan of `snd_full.bin` (0x1C1330 bytes) for the instruction forms that build
> `0x...0C06` (`bg.ori …,0xc06` = `CB ?? 0C 06`; `bg.addi …,0xc06` = `FC ?? 0C 06`, big-endian storage)
> returned **exactly 3 occurrences** — `0x25EF2`, `0x262E1`, `0x114C23` — and **none** is followed by a
> `sb`/`sh`/`sw`/`sw_0` store to that address.

### 7.2 Indirect / computed MMIO check
Scanned the whole image for any `bg.opcode_2E rX` (**0xBA/0xBB**) occurring within 16 bytes *after* a
`…0C06` address build (i.e. a possible indirect/computed write):
**0 candidates.** No evidence of an indirect write path.

### 7.3 `0x114C23` (cheap resolution attempted)
`0x114C23` is `bg.addi r4,r4,0xc06` — **`r4` is a function argument** (no `movhi` to `r4` with `0xB000` or
`0x23xxxx` was found in the preceding window), so the effective address is caller-dependent. No store to
`r4` occurs in the vicinity. Either way it is **not a store to `0xB000_0C06`**.
Classified as a read (consistent with the other two); its exact base is **UNRESOLVED** (cheap resolution
attempted; not worth further effort).

### 7.4 Net position
Given: (i) only 3 address-construction occurrences, none a store; (ii) 0 indirect-write candidates; and
(iii) all MMIO addresses in this firmware are built with immediate `bg.ori`/`bg.addi` off a `0xB000` base
(no global-loaded address indirection observed anywhere):

> **STRONG STATIC INFERENCE:** `0xB000_0C06` is a **read-only status register**; its bit7 is **not set by
> firmware** but produced externally (hardware / sink / another subsystem). Not classified as VERIFIED,
> because a fully indirect path (address loaded from a data global) cannot be excluded by static scan alone.

---

## 8. Phase G — Role of `0x2512C4` in the return path

### 8.1 Reads (VERIFIED STATIC)
`word@0x2512C4` is read at exactly two sites, both inside `0x25EE1`, via **base + offset**
(`r13 = 0x251194`, offset `0x130` → `0x2512C4`):
- `0x25F10  bg.lwz r24,0x130(r13)` — as gate **input #3** into the `r11` predicate.
- `0x25F83  bg.lwz r23,0x130(r13)` — to compute the **return value** (`sfnei r23,0` → `cmovsi_2 r3,r0,1`).

### 8.2 Immediate-form scan (VERIFIED STATIC)
A whole-image scan for `bg.ori`/`bg.addi …,0x12c4` returned **0 occurrences** — confirming the address is
never built as an immediate, only via the `0x251194` base + `0x130` offset.

### 8.3 Writer / initialization
**UNRESOLVED.** Because the access is base+offset, a simple immediate scan cannot locate the writer, and the
initialization routine (which would populate the `0x251194` structure) lies outside the dumped regions or
uses the same base-register form. No write to `word@0x2512C4` was located in the searched dumps.

### 8.4 Influence on the DTS path
- `word@0x2512C4 == 0` → return = 0 → caller takes the **`0x255CA8` alternate** path (`0xCED4` / `0xD2CF`).
- `word@0x2512C4 != 0` (and `table[+0x94+0x12] == 1`) → return = 1 → caller takes the **`0x252AD0` consumer**
  (`0xCF60` / `0xD31F`).
- It *also* feeds `r11` (gate input #3), so it can independently zero the `r11` predicate.

Semantic role: **UNRESOLVED.** It behaves as a **state/capability flag** that selects the DTS configuration
path and gates the descriptor transition. It is **not** assumed to be a license value.

---

## 9. Phase H — Reclassification of the R30 hypothesis

**R30 claim:** *"r11 = 0 blocks DTS output."*

### Verdict: **the R30 claim is REJECTED as stated — it is an over-simplification, not a demonstrated cause.**

Reasoning (all static; R30 runtime used only directionally):
1. `r11 = 0` suppresses **only** `0x13292` (population of `0x252AD0`). It does **not** suppress the
   output-enable MMIO writes (`0x086C`/`0x0870`), which the DTS helpers perform **unconditionally before**
   the gate. So `r11 = 0` is **not** a master output kill. **VERIFIED STATIC.**
2. Whether `0x13292` even matters depends on the **return value**: if `word@0x2512C4 == 0`, the caller uses
   the `0x255CA8` path and never reads `0x252AD0` — so `r11 = 0` is **irrelevant** on that branch.
   **VERIFIED STATIC.**
3. On the `return != 0` branch, `r11 = 0` changes the `0x252AD0` `+0x18` from `0x100` to `8`, producing a
   **32× divisor difference** in the computation published to `0xA868`/`0xA86C`. But if `+0x14 == 0`
   (as `0x13292` itself initializes it), the quotient is 0 either way and the difference is nil.
   **STRONG STATIC INFERENCE; the net effect is UNRESOLVED.**
4. `0x109595` — the one consumer that receives the gate's cached predicate — could not be decoded
   (**UNRESOLVED**), so the "gate state suppresses output" chain is **not** established.

Therefore (per the R32 option set):

> ### **R32-C** — the actual important DTS decision is the separate `0x25EE1` **return-value path**.
>
> The return value (`word@0x2512C4` + `+0x94` table) selects **which configuration structure drives DTS**
> (`0x252AD0` vs `0x255CA8`). The `r11 → 0x13292` transition is a **secondary, data-population step inside
> one of those two branches**, and its necessity remains **UNRESOLVED** (§3.3 caveat + §6 UNRESOLVED).
> R32-A vs R32-B cannot be distinguished statically yet.

---

## 10. Answers to the 6 required R32 questions

**Q1. What does the return value of `0x25EE1` control?**
It selects the caller's configuration path: **!= 0** → `0x252AD0` consumer (`0xCF60`/`0xD31F`);
**== 0** → `0x255CA8` alternate (`0xCED4`/`0xD2CF`). Expression: `(table[*(this+0x94)+0x12]==1) AND
(word@0x2512C4 != 0) ? 1 : 0` — driven by `0x2512C4`, **not** by `r11`. **VERIFIED STATIC.**

**Q2. What does `0x13292` prepare that `0xD323`/`0xCF64` consume?**
It populates `0x252AD0` with `+0x0=0x10F000`, `+0x4=0x1A5000`, `+0x8=0x96000`, `+0x18=0x100`, `+0x1c=0x100`,
`+0x20=4`, `+0x24=8`, `+0x30=0x8000`, `+0x34=0x100`, `+0x14=0`. `0xD323`/`0xCF64` consume `+0x18` and `+0x20`
as multiplicative scale factors. **STRONG STATIC INFERENCE** (buffer/timing descriptor; not proven DMA).

**Q3. Functional difference: populated vs default `0x252AD0`?**
`+0x18` = `0x100` vs `8` → divisor `0xC000` vs `0x600` (**32×**) in the computation published to
`0xA868` (`0xCF86`) / `0xA86C` (`0xCC9B`). Caveat: the dividend `+0x14` may be `0`, which would nullify the
difference. **STRONG STATIC INFERENCE; net audio effect UNRESOLVED.**

**Q4. What does `0x109595` do with `+0xBC`/`+0xB8`/`+0xB0`?**
It **receives** them as `r3` from the `0x27xxx` decision functions (`0x27F5B` `+0xB0`, `0x27F86` `+0xB4`,
`0x27F9F` `+0xB8`, `0x27FB9` `+0xBC`). Its internal behavior is **UNRESOLVED** (body in `0x108000–0x12FFF`,
not decodable in this environment). Not inferred from the argument.

**Q5. Does any of this directly reach an actual DTS encoder/SDO/TX operation?**
Partly: the chain reaches **output-enable MMIO** (`0x086C`/`0x0870`, written unconditionally) and the
**shared finalizer** `0x10786E → 0x115693` (which writes `0xB000_0020`/`0xB000_003E`) — **both shared with
AC3**. The **DTS-specific** part ends at the **data global `0xA868`/`0xA86C`** (published with a cache flush
for another agent). Whether that global drives an encoder/SDO/TX operation is **UNRESOLVED**.
**STRONG STATIC INFERENCE** that it is a shared output descriptor; the final consumer is not established.

**Q6. Is the R30 "gate blocks DTS output" hypothesis supported, weakened, or rejected?**
**Rejected as stated** (over-simplified). `r11 = 0` is not a master output kill; it is a secondary
data-population skip whose relevance is itself conditioned on the **return-value path** (`0x2512C4`) and
whose net effect is unresolved. R30's runtime observation (`bit7(0x0C06) = 0`) remains valid *directional*
evidence but is **not** causal proof of "DTS disabled". Classification: **R32-C**.

---

## 11. Residual UNRESOLVED items (for a possible R33)

1. **`0x109595` body** — needs a dump covering `0x108000–0x12FFF` (gap). Requires an AEON raw-binary load
   spec that the current projects lack.
2. **`+0x14` of `0x252AD0`** — actual value at consumption time (determines whether the 32× divisor
   difference matters at all).
3. **`0x2512C4` writer/init** — base+offset access; writer not located.
4. **`0xA868`/`0xA86C` final consumer** — which agent (DMA/SDO/TX/encoder) reads them after the cache flush.
5. **`0x255CA8`** — the alternate config struct's semantics and initialization.

---

## Appendix — Artifacts used / produced (all static)
- `r29_work/decoded_25EE1.txt` — gate body (return expression, field transitions).
- `r29_work/decoded_CC9B.txt` — **new this phase** (`0xCC9B`: `0xCECC` gate call, `0xCED0` branch, `0xCF64` consumer).
- `r29_work/decoded_CF86.txt` — (`0xD2C8` gate call, `0xD2CC` branch, `0xD323` consumer, `0xD15E` → `0xA868`).
- `r29_work/decoded_26region.txt` — **new this phase** (`0x261AA`/`0x26233`/`0x263CB`/`0x263DA`/`0x263EF`).
- Whole-image scans of `r27_work/snd_full.bin` for `…0C06` (3 hits, 0 stores) and `…12C4` (0 hits).
- Decode tool: `scripts/r29_decode.py` (recursive descent over Ghidra `flow[0]` dumps).

---
---

# R32 — Part II: Resolution of `0x255CA8`, `0xA868`/`0xA86C` consumers, `+0x14` writers, and `divu` semantics

*(Added after the first R32 pass; supersedes the provisional caveats in §3.3 and §10 where noted.)*

## II.1 Priority 4 — AEON `divu` semantics (VERIFIED STATIC — authoritative)

Source: `ghidra-aeon-master/ghidra-aeon-master/data/languages/aeon_ORBIS32.sinc`, line 274:
```
:bn.divu i24_rD, i24_rA, i24_rB is i24_uimm0_3 = 1 & i24_rD & i24_rA & i24_rB { i24_rD = i24_rA / i24_rB; }
```
(for reference, line 273: `:bn.divs … { i24_rD = i24_rA s/ i24_rB; }`)

> **`bn.divu rD, rA, rB`  ⇒  `rD = rA / rB`** — **rA = dividend, rB = divisor** (unsigned).
> **VERIFIED STATIC** (SLEIGH definition, not inferred).

### Corrected arithmetic at `0xD2F0` (removes the §3.3 ambiguity)
```
0xD2F0  bn.muls       r24,r24,r25      ; r24 = (+0x18) * (+0x20)
0xD2F3  bn.lwz        r23,0x14(r23)    ; r23 = *(+0x14)
0xD2F6  bg.muli_maybe r24,r24,0x30     ; r24 = r24 * 0x30
0xD2FA  bg.lwz        r26,0x114(r10)
0xD2FE  bn.divu       r25,r23,r24      ; ** r25 = r23 / r24 **  = (+0x14) / [((+0x18)*(+0x20))*0x30]
        ... r24 = *(0x253364) + r26 + *(0x118(r10)) + r25 ...
0xD31B  bg.j 0xd15e
0xD15E  bg.sw_0 0x868(r23),r24         ; r23=0xA000  ->  0xA868 = r24
```
| Case | `+0x18` | `+0x20` | divisor `((+0x18)*(+0x20))*0x30` | quotient `r25 = (+0x14)/divisor` |
|------|---------|---------|------------------|-------------------------------|
| populated (`0x13292` ran) | `0x100` | `4` | `0x400*0x30 = **0xC000**` (49 152) | `+0x14 / 0xC000` |
| default (`0x13292` skipped) | `8` | `4` | `0x20*0x30 = **0x600**` (1 536) | `+0x14 / 0x600` (**32× larger**) |

The quotient `r25` is **added** into the value published to `0xA868` (`0xD31A bt.add r24,r25`).
**The ambiguity in §3.3 is now resolved: dividend = `+0x14`, divisor = the `+0x18*+0x20*0x30` product.**

## II.2 Priority 3 — ALL writers to `0x252AD0 + 0x14` (decisive)

Whole-decode search for `0x14(r…)` in every function touching `0x252AD0`:

| Address | Function | Instruction | Value written | Non-zero capable? |
|---------|----------|-------------|---------------|-------------------|
| `0x132E5` | `0x13292` (producer) | `bn.sw 0x14(r10),r0` | **`0`** (init) | no |
| `0xD031` | `0xCF86` | `bn.sw 0x14(r3),r0` | **`0`** | no |
| **`0xD1D3`** | **`0xCF86`** | **`bn.sw 0x14(r3),r23`** | **`r23 = (+0x10) − (+0xc)`** | **YES** |
| `0xCD34` | `0xCC9B` | `bn.sw 0x14(r3),r0` | **`0`** | no |
| **`0xCCDE`** | **`0xCC9B`** | **`bn.sw 0x14(r3),r23`** | **`r23`** (computed) | **YES** |
| `0xD2F3` | `0xCF86` | `bn.lwz r23,0x14(r23)` | *(read — the divisor's dividend)* | — |
| `0xCEF8` | `0xCC9B` | `bn.lwz r23,0x14(r23)` | *(read)* | — |
| `0xD0F8`, `0xD06B`, `0xCD41`, `0xCDBF` | both helpers | `bn.lwz …,0x14(r…)` | *(reads, other contexts)* | — |

**Conclusion:** `+0x14` is **not** permanently `0`. It is initialised to `0` by `0x13292`/`0xD031`/`0xCD34`, but
**`0xD1D3` (0xCF86) and `0xCCDE` (0xCC9B) write it with a computed delta `(+0x10) − (+0xc)`**, which is
non-zero-capable. **VERIFIED STATIC** that a non-zero write path exists.

**Net effect (STRONG STATIC INFERENCE):** because `+0x14` can be non-zero at consumption, the **32× divisor
difference does propagate** into the quotient and therefore into the value written to `0xA868`/`0xA86C`.
The §3.3 "difference may be nil" caveat is **downgraded but not eliminated**: it is nil *only* on paths where
`+0x14` is still `0` (i.e. where `0xD1D3`/`0xCCDE` did not run or computed `0`). Whether that is the case at
runtime is **UNRESOLVED**.

*(Note: `0xCF86`/`0xCC9B` access `+0xc`, `+0x10`, `+0x14`, `+0x18`, `+0x1c`, `+0x30`, `+0x34`, `+0x38` on their
instance register `r3` — the same field layout `0x13292` writes to `0x252AD0` — indicating the DTS helpers
operate **on `0x252AD0` itself** as their instance object. STRONG STATIC INFERENCE.)*

## II.3 Priority 2 — Readers/consumers of `0xA868` and `0xA86C`

### II.3.1 Known producers (VERIFIED STATIC)
| Global | Written by | Instruction | Base |
|--------|-----------|-------------|------|
| **`0xA868`** | `0xCF86` | `0xD15E bg.sw_0 0x868(r23),r24` | `r23 = 0x1 − 0x6000 = 0xA000` |
| **`0xA86C`** | `0xCC9B` | `0xCE26 / 0xCE72 / 0xCF48 bg.sw_0 0x86c(r23),r24` | `r23 = 0xA000` |

Each is followed by a cache-flush (`bg.j 0xD4C8`) — i.e. the value is published for **another agent**
(DMA/ISR/other core) to observe.

### II.3.2 Whole-image scan for offset `0x868` / `0x86C` (34 + 21 raw hits; load/store-family filtered)
Offset `0x868`: `0xD15E` (EF, known write), **`0x16470` (ED)**, **`0xFFDB1` (EE)**
Offset `0x86C`: `0xCD72` (CB = `bg.ori`, MMIO `0xB000_086C`), `0xCE26/0xCE72/0xCF48` (EF, known writes),
**`0x16464` (ED)**, **`0xFFDAD` (EE)**

*(Remaining raw hits have opcodes outside the load/store family — `0x00/0x01/0x02/0x03/0x05/0x10/0x48/0x58/0x77/0x79/0x8d/0xab/0xb0/0xb4/0xd0/0xd3/0xd7/0xe0/0xe9/0xf0/0xf2/0xfa` — and are data/false positives.)*

### II.3.3 The two localized consumers — a contiguous array, not isolated scalars
Raw bytes reveal **block** accesses to consecutive 32-bit slots:

**Site A — `0x16454…0x16474`:**
```
0x16464  edaa086c   ; offset 0x86C
0x16468  ed8a0870   ; offset 0x870
0x1646C  ed6a0874   ; offset 0x874
0x16470  ed6a0868   ; offset 0x868
```
**Site B — `0xFFD9D…0xFFDB1` (strictly DESCENDING):**
```
0xFFD9D  ede1087c   ; 0x87C
0xFFDA1  ee010878   ; 0x878
0xFFDA5  ee210874   ; 0x874
0xFFDA9  ee410870   ; 0x870
0xFFDAD  ee61086c   ; 0x86C
0xFFDB1  ee810868   ; 0x868
```

> **STRUCTURAL FINDING (STRONG STATIC INFERENCE):** `0x868…0x87C` form a **contiguous 6-slot 32-bit array**
> (`0x868, 0x86C, 0x870, 0x874, 0x878, 0x87C`). `0xA868`/`0xA86C` are **slots 0 and 1** of that array, and at
> least two downstream functions access the whole array **as a block** (Site B is a textbook descending
> save/restore or block-copy sequence).
> **The values written by the DTS helpers therefore DO reach downstream block consumers.**

### II.3.4 Answer to the priority-2 question
> *Do `0xA868`/`0xA86C` reach a real DTS output/encoder path?*

**STRONG STATIC INFERENCE — yes, they reach a downstream consumer**, and that consumer treats them as part of a
contiguous output-slot array. **UNRESOLVED**: the exact identity/semantics of Sites A (`0x16454…0x16474`) and B
(`0xFFD9D…0xFFDC9`), and whether the base is `0xA000` (data array) or `0xB000` (MMIO output bank
`0xB000_0868…0xB000_087C`) — both dumps that would cover them (`ghidra_dump_13_115.txt`, range `0x13000…0x115FFF`)
are **sparse**: `grep -c '^0x164'` = 0 and `grep -c '^0x(ffd|FFD)'` = 0, i.e. those addresses were never dumped.
Decoding them requires a dump of the missing ranges (blocked in this environment — see §6.2).

## II.4 Priority 1 — Resolution of `0x255CA8` (the alternate configuration object)

### II.4.1 All reference sites (whole-image scan for immediate `0x5CA8`: 15 raw hits, 9 real)
Real `bg.addi …,0x5ca8` sites (opcodes `0xFC/0xFE/0xFF`):
`0xCED8` (`0xCC9B`), `0xD2D3` (`0xCF86`), **`0xED4C`, `0xEFD2`, `0x16089`, `0x16243`, `0x16A8B`, `0x18109`, `0x18111`**
(remaining hits — opcodes `0x81/0xa3/0x10/0x01/0x04` — are data/false positives).

> `0x255CA8` is used by **at least 9 functions**, i.e. it is **not** a one-off fallback for the two DTS helpers;
> it is a **widely-used configuration object** in its own right. **STRONG STATIC INFERENCE.**

### II.4.2 Fields accessed (from the two decodable sites)
| Site | Fields read | Fields written | Defaults |
|------|-------------|----------------|----------|
| `0xCC9B` `0xCED4` | `+0x18` (`0xCEDC`), `+0x20` (`0xCEE7`), `+0x18` (`0xCEEA`), `+0x14` (`0xCEF8`) | `+0x18` (`0xCEE4`), `+0x20` (`0xCEF2`) | `+0x18 = 0xa` (10); `+0x20 = 0x2` |
| `0xCF86` `0xD2CF` | `+0x18` (`0xD2D7`), `+0x20` (`0xD2E2`) | `+0x20` (`0xD2ED`) | `+0x20 = 0x2`; (`+0x18` default not reached in decoded path) |

Both then fall into the **same common math** (`0xCEF5` / `0xD2F0`) as the `0x252AD0` path — i.e. the identical
`(+0x18)*(+0x20)*0x30` divisor computation, then the publish to `0xA86C` (`0xCC9B`) / `0xA868` (`0xCF86`).

### II.4.3 Is it a real alternate DTS configuration structure?
**STRONG STATIC INFERENCE — yes.** Evidence: (i) it is selected by the `0x25EE1` **return==0** branch as the
configuration source; (ii) it exposes the **same field offsets** (`+0x14`, `+0x18`, `+0x20`) consumed by the same
arithmetic; (iii) it is referenced by 9 functions. It is a **second instance of the same descriptor type**
(field-compatible with `0x252AD0`), chosen when `word@0x2512C4 == 0`.

**Semantic labels are withheld** (no evidence for "rate"/"channels"/"buffer"): the fields are demonstrably
**multiplicative scale factors** into a divisor of a computed value published to a shared output-slot array.

**UNRESOLVED:** the other 7 sites (`0xED4C`, `0xEFD2`, `0x16089`, `0x16243`, `0x16A8B`, `0x18109`, `0x18111`)
are outside the available dumps, so their reads/writes are not enumerated.

## II.5 Updated classification (superseding/qualifying §10)

**Headline remains R32-C** — the primary DTS decision is the **`0x25EE1` return-value path** (`0x255CA8` vs
`0x252AD0`). But Part II materially sharpens the secondary question:

- **R32-A is NOT supported.** The `0x13292` absence is **not** a harmless lazy/optional init: with `+0x14`
  non-zero-capable, the `0x100 → 8` change in `+0x18` alters the divisor **32×** and therefore alters the
  quotient that is added into `0xA868`/`0xA86C` — a value that **is** consumed downstream as part of a
  contiguous output-slot array. → the "optional/lazy" reading is rejected. **STRONG STATIC INFERENCE.**
- **R32-B is partially supported.** The absence of `0x13292` **does change an output/timing/buffer parameter**
  that propagates to a consumed global. The qualifier "*required*" (i.e. that the change is what breaks DTS
  audio) remains **UNRESOLVED** — the downstream consumer (Sites A/B) is not decoded, and the runtime value of
  `word@0x2512C4` (which decides whether the `0x252AD0` branch is taken at all) is unknown.
- Therefore **R32-C** stands: the *important* decision is the return-value path, while the
  `r11 → 0x13292` transition is a **real but secondary** parameter-changing step **inside** the `return != 0`
  branch.

**Consolidated model:**
```
0x25EE1
├─ r11  → edge-triggered → 0x13292 → populates 0x252AD0 (+0x18=0x100, +0x14=0)
└─ return (word@0x2512C4 + table[+0x94+0x12])
        ├─ == 0  →  0x255CA8  (+0x18=10/2, +0x20=2)  ─┐
        └─ != 0  →  0x252AD0  (+0x18=0x100 | default 8) ┤
                                                        ↓
                        common math:  (+0x14) / [((+0x18)*(+0x20))*0x30]
                                                        ↓
                              publish → 0xA868 (0xCF86) / 0xA86C (0xCC9B) + cache flush
                                                        ↓
                        downstream block consumer of array 0x868…0x87C  (Sites 0x164xx / 0xFFDxx)
```

## II.6 Corrected Q3 answer (replaces the §10 Q3 wording)
Populated (`0x13292` ran): `+0x18 = 0x100` → divisor `0xC000`.
Default (skipped): `+0x18 = 8` → divisor `0x600` (**32× smaller → 32× larger quotient**).
The quotient is summed into the published value. Because `+0x14` is written non-zero-capable by
`0xD1D3`/`0xCCDE`, the difference **does** propagate (not nil in general). **STRONG STATIC INFERENCE**;
whether it alters audible output is **UNRESOLVED**.
