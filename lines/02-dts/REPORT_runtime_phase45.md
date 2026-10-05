# R45 — Recover exact values & control flow of DTS-only `0xB000_0868 / 0xB000_086C / 0xB000_0870`

**Phase chain:** R29–R44. Carried-over CLOSED branches (do NOT revisit): `/dev/malloc` mmap, MIU probes,
`A868/A86C` runtime observation, `DspMadBase`, `DM[]` reads, SHM-reader search.
**This phase (R45):** pure static value/control-flow recovery of the three registers named in the R45 brief,
with one iron rule reinforced by the user: **SND SHM `+0x868 / +0x86C` ≠ MMIO `0xB000_0868 / 0xB000_086C`.**
No patching. No device access. No `/dev/malloc` / `DM[]` / `A868` runtime work.

**Evidence base (re-read, base-registers verified this pass):**
- `r29_work/decoded_CC9B.txt` (DTS helper `0xCC9B`, 230 instrs)
- `r29_work/decoded_CF86.txt` (DTS helper `0xCF86`, 369 instrs, includes the cache-flush leaf `0xD4C8`)
- `r29_work/decoded_all6.txt` (all 6 helpers; used for AC3 re-check + full-evidence grep for `0x868` base)
- Tool: `r45_work/analyze_stores.py` (register-tracking store classifier — verifies the base of every store)

---

## 0. HEADLINE CORRECTION to R44 (this is the main R45 result)

R44 §2/§3 claimed DTS writes MMIO `0xB000_086C` **three times** (`0xCE26`, `0xCE72`, `0xCF48`) and MMIO
`0xB000_0868` once (`0xD15E`). **That classification was wrong.** Those four stores were identified only by the
store *offset* (`0x86c` / `0x868`) and the base register was **never verified** — the exact failure mode the
skill warns about ("an offset-only scan is NOT conclusive. Always verify the BASE REGISTER").

When the base register of each store is actually traced, all four turn out to be **SHM writes, not MMIO:**

| Store PC | Instruction | Base reg at store | Base value | **Effective address** | Class |
|----------|-------------|-------------------|-----------|----------------------|-------|
| `0xCE26` | `bg.sw_0 0x86c(r23),r24` | `r23` = `movhi r23,0x1; addi r23,r23,-0x6000` | `0xA000` | **`0xA86C` = SHM `+0x86C`** | SHM |
| `0xCE72` | `bg.sw_0 0x86c(r23),r24` | same | `0xA000` | **`0xA86C` = SHM `+0x86C`** | SHM |
| `0xCF48` | `bg.sw_0 0x86c(r23),r24` | same | `0xA000` | **`0xA86C` = SHM `+0x86C`** | SHM |
| `0xD15E` | `bg.sw_0 0x868(r23),r24` | same | `0xA000` | **`0xA868` = SHM `+0x868`** | SHM |

**Bulletproof corroboration — the cache-flush argument equals the written address.** Each of these four stores
is immediately followed by a tail-call `bg.j 0xD4C8` (the SHM publish cache-flush leaf, `flush_invalidate` +
`syncwritebuffer` + `mtspr 0x11`). The flush argument `r3` is built as `movhi r3,0x1; addi r3,r3,-0x5794/0x5798`:

| Store | Flush-arg build | `r3` at `0xD4C8` | Matches written cell? |
|-------|-----------------|------------------|------------------------|
| `0xCE26` | `addi r3,r3,-0x5794` | `0xA86C` | **YES** (flush publishes the just-written SHM `+0x86C`) |
| `0xCE72` | `addi r3,r3,-0x5794` | `0xA86C` | **YES** |
| `0xCF48` | `addi r3,r3,-0x5794` | `0xA86C` | **YES** |
| `0xD15E` | `addi r3,r3,-0x5798` | `0xA868` | **YES** (flush publishes the just-written SHM `+0x868`) |

A DSP only cache-flushes a line it just wrote to make it visible to the **host CPU** (the SHM consumer).
That is the definitive signature of a **shared-memory handoff field**, not a hardware register. The MMIO
stores below have **no** cache flush — they are direct writes into the `0xB000_08xx` peripheral.

**The genuine MMIO writes** in this DTS path are:

| MMIO register | Store PC | Instruction | Base build | Class |
|---------------|----------|-------------|-----------|-------|
| `0xB000_086C` | `0xCD76` | `bn.sw 0x0(r26),r25` | `0xCD68 movhi r24,0xb000` → `0xCD72 ori r26,r24,0x86c` | **MMIO (1 store)** |
| `0xB000_0870` | `0xD0A1` | `bn.sw 0x0(r26),r24` | `0xD093 movhi r23,0xb000` → `0xD09A ori r26,r23,0x870` | **MMIO (1 store)** |

R44's "0x870 store not proven" is therefore itself **wrong** — the store at `0xD0A1` is real and proven.

**`0xB000_0868` (MMIO) has NO proven store anywhere.** In `decoded_all6.txt` the only `0x868` reference is the
SHM store `0xD15E`; there is **no** `movhi 0xb000` + `ori/addi …0x868` construction in any of the six helpers
(grep for `ori …0x868` / `addi …0x868` returns nothing). So MMIO `0xB000_0868` is **UNRESOLVED** as a register
written by this DTS path (it may be written by an unexamined function, or not at all).

---

## 1. Per-store analysis (the four SHM stores + the two real MMIO stores)

### 1.1  SHM `+0x86C` — `0xCE26` (base `r23 = 0xA000`)

- **Bytes:** `EF17086C` (`bg.sw_0 0x86c(r23),r24`, len 4)
- **Effective address:** `0xA000 + 0x86C = 0xA86C` = SHM `+0x86C`. (base `r23` set at `0xCE1C`/`0xCE22`)
- **Source `r24` provenance (preceding computes):** r24 is a **computed ring-buffer write pointer / byte-offset**:
  it is the sum of (a) a timer-derived quotient (`(timer>>3) − sign`, from `0xCE04`/`0xCE0E`/`0xCE11`, magic
  `0x2AAA_AAAB` ≈ division by 3), (b) `r28` (a value loaded via `bg.opcode_2E r28` at `0xCD9E` — an MMIO read),
  (c) `r25 = [0x25194+0x118]` (`0xCE14`), and (d) `r26 = [0x25194+0x118]` / `[0xA118]`-class load
  (`0xCE20`). Net: **not a constant, not a bitfield — a computed offset/position handed to the host.**
- **Condition/reachability:** reached in the DTS working path; the preceding `0xCCDA` block falls through from the
  `0xCCB2` equality-check path when the buffer delta is non-negative (`0xCCD6 blesi r23,-0x1,0xceab`).
- **Next op:** `0xCE2A` build flush arg `r3=0xA86C` → `0xCE3A bg.j 0xD4C8` (publish).

### 1.2  SHM `+0x86C` — `0xCE72` (base `r23 = 0xA000`)

- **Bytes:** `EF17086C` (`bg.sw_0 0x86c(r23),r24`, len 4)
- **Effective address:** `0xA86C` (SHM `+0x86C`). (base `r23` re-set at `0xCE4A`/`0xCE4E`)
- **Source `r24` provenance:** sum of `r24 = [0xA128]` (`0xCE5A lwz r24,0x128(r23)`, r23=0xA000), `addi r24,-0x20`
  (`0xCE65`), `opcode_2E r25` result (`0xCE61`), `r28` (`0xCE6A`), `r26 = [0xA118]` (`0xCE70`). Again a
  **computed buffer offset**, not a constant.
- **Condition:** reached via the `0xCE3E` `opcode_2E r25` branch dispatch (`beqi r25,0x1 → 0xCEC4`, `beqi r25,0x2 →
  0xCF1E`); `0xCE72` is the fall-through/`r25==0` path of that dispatch (`0xCE42`/`0xCE46`).
- **Next op:** build flush arg `r3=0xA86C` → `bg.j 0xD4C8`.

### 1.3  SHM `+0x86C` — `0xCF48` (base `r23 = 0xA000`)

- **Bytes:** `EF17086C` (`bg.sw_0 0x86c(r23),r24`, len 4)
- **Effective address:** `0xA86C` (SHM `+0x86C`). (base `r23` set at `0xCF1E`/`0xCF22`)
- **Source `r24` provenance:** sum of `r24 = [0xA128]` (`0xCF2A lwz r24,0x128(r23)`), `addi r24,-0x60`
  (`0xCF34`), `opcode_2E r24` (`0xCF3C`), `r26 & 0xFFF` (`0xCF40`), `r26 = [0xA118]` (`0xCF46`), `r25 =
  [0xA118]` (`0xCF44`). Computed offset, same family as §1.1/§1.2.
- **Condition:** reached via `0xCF1E` dispatch (the `r25==2` path, `0xCE46 beqi r25,0x2,0xcf1e`).
- **Next op:** build flush arg `r3=0xA86C` → `bg.j 0xD4C8`.

### 1.4  SHM `+0x868` — `0xD15E` (base `r23 = 0xA000`)

- **Bytes:** `EF170868` (`bg.sw_0 0x868(r23),r24`, len 4)
- **Effective address:** `0xA000 + 0x868 = 0xA868` = SHM `+0x868`. (base `r23` set at `0xD154`/`0xD15A`)
- **Source `r24` provenance:** reached from three tail-call sites (`0xD238`, `0xD2BC`, `0xD31B`) all jumping to
  `0xD15E`. r24 is a **computed offset** = `[0xA128]` (`0xD213 lwz r24,0x128(r23)`) ± timer-derived terms
  (`0xD21E addi r24,-0x20`, `0xD2A3 addi r24,-0x60` on the other paths) + `opcode_2E` result + `[0xA118]`-class
  globals. Not a constant.
- **Condition:** this is the **shared DMA/packer timing tail** of `0xCF86` (all three `opcode_2E`-gated branches
  funnel here), so `0xA868` is written on every DTS packet-timing pass.
- **Next op:** build flush arg `r3=0xA868` → `0xD174 bg.j 0xD4C8` (publish).

### 1.5  MMIO `0xB000_086C` — `0xCD76` (base `r26 = 0xB000_086C`)  **← the real DTS-specific HW write**

- **Bytes:** `0F3A00CB` (`bn.sw 0x0(r26),r25`, len 3)
- **Effective address:** `r26 = 0xB000_086C` (base built `0xCD68 movhi r24,0xb000` → `0xCD72 ori r26,r24,0x86c`).
- **Source `r25` provenance:** `r25 = (sample_global + timer) >> 11 − sign`, then masked `& 0xFFFF`:
  - `0xCD41 lwz r23,[0x25194+0x14]` (a global, = sample-rate / period field),
  - `0xCD4C muls r25,r23,0xA57EB503` (magic mul — **then immediately overwritten** by `0xCD53 add r25,r23,r26`,
    so the net arriving value is the add+shift, not the mul),
  - `0xCD4F mfspr1 r26,0x2808` (timer/counter),
  - `0xCD53 add r25 = r23 + r26`, `0xCD56 srai r25,0xb` (>>11, arithmetic), `0xCD59 srai r26,r23,0x1f` (sign),
    `0xCD5C sub r25 = r25 − sign`,
  - `0xCD6F and r25,r25,r26` with `r26=0xFFFF` → **16-bit masked timing/period quotient**.
- **Value semantics:** **16-bit scaled-division timing/period quotient** (sample-rate-derived). Not a constant,
  not an enable bitfield, not a buffer address.
- **Condition:** reached on the **normal DTS path** — branch `0xCCAF bgtsi r25,-0x1,0xcd04` is taken when
  `MMIO[0x814]<<10 >= 0` (the usual case), flowing `0xCD04 → 0xCD39 → 0xCD68 → 0xCD72 → 0xCD76`. (The
  not-taken path re-joins via `0xCCDA` and still reaches `0xCD76`.) So the write is **live**.
- **Next op:** `0xCD79` build `0xB000_0814`, read+mask; `0xCD87` build `0xB000_0818`; `opcode_2E` MMIO reads at
  `0xCD91`/`0xCD9E` (SDO/packer-status region). No read-back (poll) of `0x86C` itself.

### 1.6  MMIO `0xB000_0870` — `0xD0A1` (base `r26 = 0xB000_0870`)  **← the real DTS-specific HW write (R44 missed it)**

- **Bytes:** `0F1A00CB` (`bn.sw 0x0(r26),r24`, len 3)
- **Effective address:** `r26 = 0xB000_0870` (base built `0xD093 movhi r23,0xb000` → `0xD09A ori r26,r23,0x870`).
- **Source `r24` provenance:** `r24 = (timing_div_result + sample_quot) & 0xFFFF`:
  - `0xD028 lwz r24,[r3+0x1c]` (input-struct field),
  - `0xD03D muls r27,r24,0x10624DD3` (magic), `0xD047 mfspr1 r28,0x2808` (timer), `0xD04B srai r24,r24,0x1f`
    (sign), `0xD04E srai r25,r28,0x6` (timer>>6), `0xD051 sub r24 = r25 − sign`, `0xD054 slli r24,0x2`,
    `0xD060 divs r23,r23,r24` (scaled division by a timer-derived value),
  - `0xD06B–0xD086` recompute a sample-rate quotient `r26 = [0x25194+0x14]` (same magic `0xA57EB503`, >>11),
  - `0xD08C add r24 = r23 + r26`; `0xD097 sw [r3+0x38]=r24`; `0xD09E and r24,r24,r28` with `r28=0xFFFF` →
    **16-bit masked timing/length quotient**.
- **Value semantics:** **16-bit scaled-division timing/length quotient** (sample-rate-derived). Same family as
  `0x86C`.
- **Condition:** reached in the main timing path of `0xCF86` after the codec-type switch (`0xD054`/`0xD05D`
  etc.); it is the per-packet timing configuration write.
- **Next op:** `0xD0A4` build `0xB000_0814`, read+mask; `0xD0B2`/`0xD0B9` build `0x81C`/`0x818`; `opcode_2E`
  reads at `0xD0C3`/`0xD0C7`. No read-back of `0x870` itself.

---

## 2. Per-register classification

| Register | Real MMIO write? | Where | Verified value | SHM counterpart |
|----------|------------------|-------|---------------|-----------------|
| **`0xB000_086C`** | **YES — 1 store** (`0xCD76`, `0xCC9B`) | base `r26=0xB000_086C` | 16-bit sample-rate/timing quotient `(samp+timer)>>11−sign` | SHM `+0x86C` (`0xA86C`) written ×3 (§1.1–1.3) — **different cell** |
| **`0xB000_0868`** | **NO proven store** | — | **UNRESOLVED** (no `movhi 0xb000; ori/addi …0x868` anywhere in all6) | SHM `+0x868` (`0xA868`) written ×1 (`0xD15E`, §1.4) — the "0x868 MMIO store" of R44 was this SHM cell |
| **`0xB000_0870`** | **YES — 1 store** (`0xD0A1`, `0xCF86`) | base `r26=0xB000_0870` | 16-bit timing/length quotient `(div + samp_quot)` | (no SHM `+0x870` examined) |

**Bottom line:** the DTS-specific **hardware** divergence is **two MMIO timing registers** — `0xB000_086C` and
`0xB000_0870` — each written once with a 16-bit sample-rate-derived quotient. The three `+0x86C` and one `+0x868`
"stores" R44 attributed to MMIO are in fact **SHM handoff fields** (`0xA86C`, `0xA868`), cache-published to the
host. `0xB000_0868` is **not** written by this path (UNRESOLVED).

---

## 3. Value semantics reconstruction (what the written bits mean)

- **Both MMIO values are data, not control:** they are 16-bit **scaled-division results** of a sample-rate /
  timer global (`[0x25194+0x14]`) and the `0x2808` timer. The arithmetic is a classic "multiply by magic constant
  + shift − sign" signed integer division producing a **small timing/period/length number** (typical of SPDIF
  IEC61937 burst spacing / pause-period / burst-length registers in a transmitter).
- **Not** a constant, codec enum, buffer pointer, size, counter-increment, or state bitfield. Both are written
  **write-only** (no poll, no reset, no read-back in the code).
- **Output-facing relationship:** both are **configuration / mode-timing registers** for the SDO/non-PCM
  hardware block. They are written once during output setup; the surrounding `opcode_2E` reads (`0xCD91`/`0xCD9E`
  in `0xCC9B`, `0xD0C3`/`0xD0C7` in `0xCF86`) are the SDO/packer-status MMIO touches. They are **not** poll/reset/
  enable bits (those live in `0x854`/`0x850`/`0x84C`).
- **SDO/IEC61937 link:** STRONG INFERENCE that `0x86C`/`0x870` are SDO/IEC61937 burst-timing registers (DTS-
  specific, in the `0xB000_08xx` block, written with burst-timing values). **NOT PROVEN** — R44 §5 found no
  static link (dispatch table / function pointer / shared struct) to `DTSX_CORE2_API_SDO_Packer` /
  `DTSDecSDOPacker_API_Process` / `Mstar_DTS_Hdmi_Packer`, and no datasheet exists.

---

## 4. AC3 comparison — re-checked correctly

Re-grep of `decoded_all6.txt` (all four AC3 helpers `0xCB50`, `0xCAB4`, `0xC49D`, `0xC5C5`):
- AC3 constructs **only** `0x854 / 0x850 / 0x84C / 0x814 / 0x80C / 0x810` (R44 §1 table, re-confirmed).
- AC3 does **not** construct MMIO `0x86C` or `0x870` (the only `ori …0x86c` / `ori …0x870` in all6 are in
  `0xCC9B`/`0xCF86`). → AC3 never writes the two DTS-specific MMIO timing registers.
- AC3 does **not** write SHM `+0x868`/`+0x86C` (the only `0x868`/`0x86c(r23)` SHM stores are in the DTS
  helpers; AC3's SHM slots are the different `0x486c`/`0x4870`-on-`r10` pair per skill §7).
- **Conclusion:** the divergence is clean and confirmed — AC3 (known-working) programs **neither** the DTS-specific
  MMIO timing block (`0x86C`/`0x870`) **nor** the DTS-specific SHM `+0x868`/`+0x86C` fields.

---

## 5. Evidence classification

### VERIFIED
- Base registers of all six target stores traced; the four `0x86c(r23)`/`0x868(r23)` stores resolve to
  `0xA000` (SHM) and the two `0x0(r26)` stores resolve to `0xB000_086C` / `0xB000_0870` (MMIO). (§0, §1)
- The four SHM stores are cache-published: the `0xD4C8` flush argument `r3` equals the written cell (`0xA86C` /
  `0xA868`). Definitive SHM signature. (§0)
- `0xB000_086C` written once at `0xCD76`; `0xB000_0870` written once at `0xD0A1`; both 16-bit masked quotients of
  sample-rate/timer globals. (§1.5, §1.6)
- `0xB000_0868` has **no** MMIO store in the examined helpers (full-evidence grep). (§0, §2)
- AC3 re-check: AC3 writes neither the DTS MMIO timing block nor the DTS SHM `+0x868`/`+0x86C`. (§4)
- R44 §2's "MMIO `0x86C` ×3" and "MMIO `0x868` ×1" attributions are **corrected**; R44's "0x870 store not
  proven" is **corrected** (it is proven).

### STRONG INFERENCE
- `0xB000_086C`/`0xB000_0870` are SDO/IEC61937 burst-timing / period / length registers (data, not control).
- The SHM `+0x86C`/`+0x868` fields are DSP→host ring-buffer write-pointer / byte-offset handoffs (consistent
  with R33–R37).
- The DTS-only MMIO timing block is the first hardware-facing divergence between working AC3 and failing DTS.

### UNRESOLVED
- Exact bit-level meaning of `0x86C` / `0x870` (no datasheet).
- Whether `0xB000_0868` is written by an unexamined DTS function (not found in all6).
- Whether the `0x86C`/`0x870` values differ between working AC3 (which doesn't write them) and failing DTS in a
  way that **causes** the failure — needs a *safe* runtime comparison, which remains blocked (MMIO writes are
  invisible to `DM[]`; `/dev/malloc` is forbidden per the STOP instruction).
- Causal role of the divergence (controlled experiment, not more static work).

---

## 6. R45 decision (A / B / C / D)

- **R45-A** (proven divergent HW register + proven causal role) — **NO**: causal role unproven.
- **R45-B** (proven divergent HW register, plausible but causal role unproven) — **PARTIAL FIT**: the two MMIO
  timing registers are proven divergent and plausible blockers.
- **R45-C** (proven divergence, semantics unproven) — **SELECTED**: we have a *corrected, verified* divergence
  (DTS writes `0xB000_086C` + `0xB000_0870` with sample-rate timing quotients; AC3 writes neither), but the exact
  register semantics and causal role are unproven. This supersedes the over-broad R44-C with a precise, base-
  verified picture.
- **R45-D** (no divergence) — **NO**.

**Verdict: R45-C** — with the explicit correction that the "DTS MMIO block" is two timing registers, not three,
and that the prior three "`0x86C` MMIO stores" were SHM.

---

## 7. Patch decision

**Do NOT create a patch.** The values written to `0x86C`/`0x870` are *computed* sample-rate timing quotients that
are correct-by-construction for the DTS stream; there is no evidence they are wrong, only that they are
**DTS-specific**. A speculative patch (e.g. forcing these registers, or copying AC3's behaviour) is unjustified
and risks breaking SPDIF timing. The next step that could change the classification is a **safe** runtime
comparison of `0xB000_086C` and `0xB000_0870` during working AC3 vs failing DTS — but (a) MMIO writes are
invisible to `DM[]`, (b) `/dev/malloc` mmap is forbidden, (c) no kernel-sanctioned MMIO-peek exists. So the
experiment is **blocked**; the honest conclusion stands: *the DTS-specific MMIO timing block is identified and
base-verified, but its causal role for the DTS failure is unresolved.*

---

## 8. Reusable tooling

`r45_work/analyze_stores.py` — register-tracking store classifier. Parses the decoded helper bodies, tracks each
register, and reports every store's **effective address** + base class (MMIO / SHM / OTHER). This is the check
that should have been run in R44 to avoid the base-register conflation; use it for any future "which address does
this store hit?" question.
