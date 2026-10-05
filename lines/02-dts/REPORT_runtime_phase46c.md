# REPORT — R46c — Exact DTS timing-writer formulas, input provenance, and IEC61937 comparison

**Subject:** Thundeal TD98 Pro / MStar MT5889 (Android 11) — AEON (R2) audio DSP
**Image:** `snd_full.bin` (1,839,920 bytes, base 0)
**Method:** **Ghidra only** (authoritative `aeon.slaspec`). Every instruction quoted below is Ghidra ground truth; two independent decoders were used to cross-check the one suspicious instruction.
**Constraints honoured:** no patch, no device access, no runtime experiment, no new full-image MMIO survey.
**Versioning:** new report. `REPORT_runtime_phase46.md`, `R46_FINAL_SUMMARY.md` and `REPORT_runtime_phase46b.md` are preserved as prior evidence; where they conflict, **this report supersedes them**.
**Evidence labels used throughout:** VERIFIED / STRONG INFERENCE / UNRESOLVED.

---

## §26 — Scope and the headline results

Asked: are the DTS-specific values written to `0x868`/`0x86C`/`0x870` *mathematically correct* for IEC61937/DTS output timing?

Three results, all in §30:
1. **The formulas are recoverable in structure but NOT convertible to any physical unit** → classification **C (UNRESOLVED)**.
2. **Two previous descriptions were wrong** and are corrected here: the `0x86C` formula was missing a path-dependent additive term (§27.1), and **`0x868` is not sample-rate-derived at all** — it is driven by a **wrapping counter capped at 67** (§27.3).
3. **A new structural anomaly**: on the examined path the `0x870` division numerator is **provably zero** (§27.2), so `0x870` receives the same value as `0x86C`. This is stated as a finding about the code, **not** as proof that anything is wrong for IEC61937.

---

## §27 — Exact formulas

### §27.1 `0x86C` — helper `0xCC9B`, store `0xCD76` (VERIFIED, corrected)

`0xCC9B` has **two entry paths**; both join at `0xCD39`.

```
0xCCA5/CCAC  r24 = word@0xB000_0814 ; r25 = r24 << 10
0xCCAF bn.bgtsi r25,-0x1,0xCD04      → if (0x814 << 10) >= 0  → PATH B (0xCD04)
                                     else                     → PATH A (0xCCB2)
-- PATH A (0xCCB2 .. 0xCD00) --
0xCCB2/CCB6/CCBC  r24 = word@0xB000_080C ; r23 = word@0xB000_080C
0xCCC3/CCC6  r24 &= 0xFFFFFF ; r23 &= 0xFFFFFF
0xCCC9 bg.beq r24,r23,0xCE96         → NOTE: both loads use the SAME base r23, so
                                       this compare is degenerate (see §27.4)
0xCCCD/CCD0  r23 = ctx[0x0c] ; r24 = ctx[0x10]
0xCCD3 bn.sub r23,r24,r23            → r23 = ctx[0x10] − ctx[0x0c]
0xCCD6 bg.blesi r23,-0x1,0xCEAB      → if r23 <= −1 → 0xCEAB
0xCCDA/CCDE  ctx[0x14] = r23
0xCCE8/CCEC  r24 = 0x2AAAAAAB        (magic for /3)
0xCCF0 bn.muls r24,r23,r24           → product DISCARDED (dead — see §27.4)
0xCCF3 bg.mfspr1 r25,0x2808          → r25 = SPR(0x2808)
0xCCF7 bn.srai r23,r23,0x1f          → sign(r23)
0xCCFA bn.srai r24,r25,0x5           → r24 = SPR >> 5
0xCCFD bn.sub r24,r24,r23            → r24 = (SPR>>5) − sign(ctx[0x10]−ctx[0x0c])
0xCD00 bg.j 0xCD39
-- PATH B (0xCD04 .. 0xCD37) --
0xCD04/08  r25 = word@0xB000_0854 ; r25 &= 0xF5FFFF ; write 0x854 = r25
0xCD18/1C  write 0xB000_084C = 0
0xCD1F/2B  r24 = ctx[0x2c] + 1 ; ctx[0x2c] = r24
0xCD2E/31/34  ctx[0x10] = ctx[0x0c] = 0xD72000 ; ctx[0x14] = 0
0xCD37 bt.movi r24,0x0               → r24 = 0
-- JOIN (0xCD39) --
0xCD39/3D  r10 = 0x251194
0xCD41 bn.lwz r23,0x14(r10)          → r23 = word@0x2511A8          [STATE]
0xCD44/48  r25 = 0xA57EB503
0xCD4C bn.muls r25,r23,r25           → product DISCARDED (dead — §27.4)
0xCD4F bg.mfspr1 r26,0x2808          → r26 = SPR(0x2808)
0xCD53 bn.add r25,r23,r26            → r25 = STATE + SPR
0xCD56 bn.srai r25,r25,0xb           → r25 >>= 11
0xCD59 bn.srai r26,r23,0x1f          → r26 = sign(STATE)
0xCD5C bn.sub r25,r25,r26            → Q = ((STATE + SPR) >> 11) − sign(STATE)
0xCD62 bt.add r25,r24                → r25 = Q + r24        ← PATH-DEPENDENT TERM
0xCD6C bn.sw 0x30(r3),r25            → ctx[0x30] = r25
0xCD6F bn.and r25,r25,0xFFFFFF
0xCD76 bn.sw 0x0(0xB000_086C),r25
```

**Formula (VERIFIED):**

```
Q     = ((word@0x2511A8 + SPR(0x2808)) >> 11) − sign_extend(word@0x2511A8, 31)
0x86C = (Q + T) & 0x00FFFFFF
        T = (SPR(0x2808) >> 5) − sign(ctx[0x10] − ctx[0x0c])   if (word@0x814 << 10) >= 0
        T = 0                                                  otherwise
```

> **Correction to R46 §19.1 / §16.1:** those stated `0x86C = Q` (i.e. `T = 0`). That is only **PATH B**. On PATH A there is an additional `SPR>>5` term. §19.1 was produced before the helper's second path was examined.

### §27.2 `0x870` — helper `0xCF86`, store `0xD0A1` (VERIFIED, with a new anomaly)

```
0xCF92/CF96  r23 = word@0xB000_0814 ; r23 &= 0xFFFFFF
0xCFA3 bn.lwz r26,0x34(r3)           → r26 = ctx[0x34]
0xCFA6 bg.opcode_2E r24              → UNMODELLED op on r23's value
0xCFAC bg.beq r24,r26,0xD23C         → path select
0xCFB2 bn.lwz r24,0x1c(r3)           → r24 = ctx[0x1c]   [SAMPLE RATE]
   rate dispatch:  88200 (0x15888)→code 1 ; 128000 (0x1F400)→0xe ;
                   176400 (0x2B110)→0xd ; 192000 (0x2EE00)→0xc ; else→3
                   with sub-branches at 0xD178 (rate<=88200) and 0xD250 (rate<=128000)
0xCFF3 bn.andi r23,r23,0xf           → r23 = word@0x814 & 0xF  (currently-programmed code)
0xCFF6 bg.beq r23,r25,0xD19F         → if already programmed → SKIP the write
0xCFFA/D00F  r25 = word@0xB000_0854 & 0x5FFFFF ; write 0x854 = r25
0xD012/D016  write 0xB000_0850 = 0
0xD019/1F/25  r24 = ctx[0x30] + 1 ; ctx[0x30] = r24      [ctx[0x30] = the 0x86C value]
0xD028 bn.lwz r24,0x1c(r3)           → r24 = ctx[0x1c]   [SAMPLE RATE]
0xD01C/21/2B/2E  r23 = 0xDD8000 ; ctx[0x10] = ctx[0x0c] = 0xDD8000
0xD031  ctx[0x14] = 0
0xD034 bt.movi r23,0x0               → r23 = 0            ← NUMERATOR ZEROED
0xD036 bn.lwz r26,0x34(r3)
0xD039 bg.beqi r24,0x0,0xD1E1        → if rate == 0 → 0xD1E1
0xD03D/40  r27 = 0x10624DD3          (magic for /1000)
0xD044 bn.muls r27,r24,r27           → product DISCARDED (dead — §27.4)
0xD047 bg.mfspr1 r28,0x2808          → r28 = SPR(0x2808)
0xD04B bn.srai r24,r24,0x1f          → sign(rate)
0xD04E bn.srai r25,r28,0x6           → r25 = SPR >> 6
0xD051 bn.sub r24,r25,r24            → r24 = (SPR>>6) − sign(rate)
0xD054 bn.slli r25,r24,0x2           → r25 = r24 << 2
0xD057 bn.sfnei r26,0x0
0xD05A bn.cmov_0 r24,r25,r24         → select r24 = r25 or r24
0xD05D bn.slli r24,r24,0x2           → r24 <<= 2
0xD060 bn.divs r23,r23,r24           → r23 = 0 / divisor  = 0
0xD06B bn.lwz r25,0x14(r10)          → r25 = word@0x2511A8
0xD079 bg.mfspr1 r27,0x2808
0xD07D bn.add r26,r25,r27 ; 0xD080 bn.srai r26,r26,0xb ; 0xD086 bn.sub r26,r26,r27
                                     → r26 = Q   (identical to §27.1's Q)
0xD08C bn.add r24,r23,r26            → r24 = 0 + Q = Q
0xD097 bn.sw 0x38(r3),r24            → ctx[0x38] = Q
0xD09E bn.and r24,r24,0xFFFFFF
0xD0A1 bn.sw 0x0(0xB000_0870),r24
```

**Formula (VERIFIED):**

```
0x870 = ((0 / D) + Q) & 0x00FFFFFF  =  Q & 0x00FFFFFF      on the examined path
        D = (((SPR(0x2808) >> 6) − sign(rate)) << 2 [cond. select] << 2)
```

> **New finding (VERIFIED as code behaviour):** the numerator is zeroed at `0xD034` (`bt.movi r23,0x0`; encoding verified against the slaspec: `i16_opcode = 0x26`, `rD=23`, `simm5=0`) and nothing between `0xD034` and `0xD060` writes `r23`. Two independent decoders agree on `0xD060 bn.divs r23,r23,r24`. ⇒ **on this path `0x870` receives exactly the same value as `0x86C`.** The `0xDD8000` constant is not lost — it is stored to `ctx[0x0c]`/`ctx[0x10]` (`0xD02B`/`0xD02E`), i.e. consumed elsewhere. Whether the zeroed numerator is intentional or a defect is **UNRESOLVED** (it is *not* evidence of an IEC61937 timing error, because the register's unit is unknown — §30).

### §27.3 `0x868` — `0x178B9` (VERIFIED, **classification corrected**)

```
0x17868 bg.lwz r13,0xb70(r10)        → r13 = word@(r10 + 0xb70)   [PRE-increment value]
0x17874 bg.muli_maybe r13,r13,0x30   → r13 *= 48
0x17880 bt.add r13,r25               → r13 += r25
0x1788A bt.mov r3,r13
0x17890 bg.jal 0x1078A9              → helper call
0x17894 bg.lwz r23,0xb70(r10)        → re-load the same counter
0x1789B bt.addi r23,0x1              → r23 += 1
0x1789D bn.sflesi r23,0x43           → compare r23 <= 0x43 (67)
0x178A0 bn.cmov_0 r23,r23,r0         → if r23 > 67 then r23 = 0     [WRAP]
0x178A6 bg.sw_0 0xb70(r10),r23       → store the counter back
0x178B2 bn.and r24,r13,0x00FFFFFF
0x178B9 bn.sw 0x0(0xB000_0868),r24
```

**Formula (VERIFIED):**

```
0x868 = ((word@(r10+0xb70) × 48) + r25) & 0x00FFFFFF
where word@(r10+0xb70) is a WRAPPING COUNTER with range 0..67 (cap 0x43, wraps to 0)
```

> **Correction to R46 §19.3 / §12.1.6 / §16.3:** `0x868` is **not** sample-rate-derived. Its dominant term is a **bounded counter (0..67) × 48**, i.e. an index/marker that sweeps a bounded range, not a rate- or timer-derived quotient. Any statement that all three registers carry "sample-rate-derived timing quotients" is **wrong for `0x868`**.

### §27.4 Anomaly: three "dead" magic multiplies (UNRESOLVED — limits the arithmetic's certifiability)

Three multiplication results are computed and then never read:

| site | constant | magic for | result |
|---|---|---|---|
| `0xCD4C` | `0xA57EB503` | (unknown; ≈0.6468 × 2³²) | overwritten by `bn.add` at `0xCD53` |
| `0xCCF0` | `0x2AAAAAAB` | division by 3 | overwritten by `bn.srai` at `0xCCFA` |
| `0xD044` | `0x10624DD3` | division by 1000 (2³⁸/1000) | never read |

Compiler output does not normally contain three distinct discarded magic multiplies. Two explanations are possible and **neither can be settled with the available tooling**:
- (a) the AEON multiply form used here writes a **MAC register pair** rather than `rD` (the slaspec models `bn.muls rD,rA,rB` as `rD = rA*rB`, which may be a simplification), so the products *are* consumed by a later MAC read;
- (b) the decode of the immediately following instruction is wrong.

⇒ **STRONG INFERENCE that the arithmetic as printed is not the whole story.** This is a hard limit on §30's certifiability.

---

## §28 — Input provenance

| Input | What is provable | Label |
|---|---|---|
| `SPR(0x2808)` | Read via `bg.mfspr1 rN,0x2808`. The slaspec defines the SPR field as **bits 5–20** (`i32_uimm5_16`), so `0x2808` is the correct field value (the old Python reading `16` was wrong). **`0x2808` is NOT named in the available SPR definitions:** the file's group 5 (`offset 0xa000` = SPR `0x2800`) lists only indices 0–6 (`_`,`MACLO`,`MACHI`,`FPMADDLO`,`FPMADDHI`,`VMACLO`,`VMACHI`); the tick timer is elsewhere (**SPR `0x5000`/`0x5001` = `TTMR`/`TTCR`**, group 10). Used only as `SPR>>3`, `SPR>>4`, `SPR>>5`, `SPR>>6` and added to a state word → consistent with a **monotonic counter/time base**, but its identity and units are **UNRESOLVED**. | VERIFIED (encoding) / UNRESOLVED (meaning) |
| `word@0x2511A8` | Read as `r10 = 0x251194` then `bn.lwz rN,0x14(r10)` → `0x2511A8`. It is a **data** word (not MMIO). Its writer was **not** re-established in this pass (would need a data-address writer sweep, out of the agreed scope). It is added to the SPR and shifted, i.e. used as an **accumulated state/timestamp**. | VERIFIED (address) / UNRESOLVED (writer) |
| `word@0xB000_0814` | **Read-only** (0 stores, 24 loads, R46b §23.1). Bit 21 (`<<10` sign test) selects the `0x86C` path; bits 0–3 hold the **currently-programmed rate code** compared at `0xCFF6`. | VERIFIED (read-only + usage) |
| `word@0xB000_080C` | **Read-only** (7 loads, 0 stores). Read twice into two registers in the `0x86C` path. | VERIFIED (read-only) |
| sample rate | `ctx[0x1c]` (context field), compared against `88200 / 128000 / 176400 / 192000` in `0xCF86` and `88200 / 44100 / 64000 / 32000` in AC3's `0xCB50`. ⇒ **the sample rate is a software context field, not an MMIO read.** | VERIFIED |
| `word@(r10+0xb70)` | A **software counter** incremented once per call and wrapped to 0 when it exceeds `0x43` (67) — a **bounded index**, not a rate. | VERIFIED |
| `ctx[0x30]` | Holds the `0x86C` value (`0xCD6C` writes it); `0xCF86` then reads it, adds 1, and writes it back ⇒ used as a **sequence counter**. | VERIFIED |
| `r25` in the `0x868` writer | Function-local; its origin is inside the SDO-packer cluster and was not traced further (out of scope). | UNRESOLVED |

**Units:** no unit can be assigned to any of the three register values. The only dimensional hints are `SPR>>k` (a counter divided by 8/16/32/64) and the sample-rate magic multiplies — but §27.4 shows those multiplies are discarded, so **the dimensional analysis does not close**.

---

## §29 — IEC61937 / DTS expected quantities (for comparison)

Expressed symbolically, because the constants must come from the DTS/IEC61937 specification, not from the firmware:

| Quantity | Expression | At 48 kHz |
|---|---|---|
| audio frame period (= IEC61937 burst repetition period) | `T_burst = N_frame / fs` | `N_frame = 512` → 10.667 ms; `N_frame = 1024` → 21.333 ms |
| burst payload | the DTS core frame (`N_frame` samples of coded audio) | — |
| IEC61937 preamble | 4 × 16-bit words (Pa, Pb, Pc, Pd) then the payload | — |
| inter-burst pause | `T_pause = T_burst − T_burst_payload` | — |
| sample-rate-dependent output timing | any register that encodes burst/pause timing must scale as `1/fs` (or as a count of a **known** clock per burst) | — |

**Note on `N_frame`:** the DTS core frame length in samples must be taken from the DTS specification for the active rate; both 512 and 1024 appear in the literature for different modes. This report therefore **does not assert a single number** — it states the two candidates and the resulting periods.

---

## §30 — Comparison and classification

**Can the recovered formulas be checked against §29?** No, for three independent reasons:

1. **Unknown unit of the register.** Nothing in the firmware converts the register value to samples, bytes, ticks or µs; there is no firmware reader at all (R46b §21.3), so the unit must come from the register datasheet.
2. **Unresolved inputs.** The path selection depends on read-only hardware status (`0x814` bit 21, `0x80C`), and the time base `SPR(0x2808)` is unnamed. Both are needed to evaluate any expression numerically.
3. **Non-certifiable arithmetic.** §27.4 shows three discarded magic multiplies; the printed arithmetic is therefore not trustworthy as the complete computation.

**Classification: C — UNRESOLVED.** *(See §36.6: after the follow-up work, this now rests on a single reason — the register unit — because the arithmetic was shown to close exactly.)*
The formulas are clear in *structure*, but the unit/meaning of the hardware registers **cannot be determined without datasheet or runtime evidence**. Therefore:
- **H1 is NOT confirmed** — no numeric value can be shown to be wrong. Per the explicit instruction, "H1 is the root cause" is **not** claimed.
- **H1 is NOT refuted** either. In particular, `0x868` is now known to be **counter-driven, not rate-driven** (§27.3), which *removes* it from the "timing quotient" family and is a candidate defect — but it cannot be called wrong without knowing what the register expects.

**What would settle it:** (a) the MMIO register map naming `0x868`/`0x86C`/`0x870` and their units; or (b) a runtime capture of the three values plus `0x814`/`0x80C`/`SPR(0x2808)` during DTS passthrough, compared against AC3's defaults. Both are outside the current constraints.

---

## §31 — The next control divergence after the timing registers

Using only the already-generated dumps (no new survey), the shared-control writes differ as follows:

| register | AC3 (`0xCB50`) | DTS (`0xCC9B` / `0xCF86`) |
|---|---|---|
| `0x854` | **ORs in a rate code** (`0x1000`/`0x4000`/`0x2000`/`0x5000`/`0x3000`) plus flags `0xA0044` and the low 12 bits of its prior value (`0xCBF4`/`0xCBFB`) | `0xCC9B` PATH B: `0x854 &= 0xF5FFFF` (clears bits 17,19). `0xCF86`: `0x854 &= 0x5FFFFF` (clears bits 20,21) |
| `0x850` | written 0 (`0xCAE2`) | `0xCF86` writes 0 (`0xD016`) |
| `0x84C` | written 0 (`0xCC24`, `0xC5B0` region) | `0xCC9B` PATH B writes 0 (`0xCD1C`) |
| `0x85C` | read-modify-write (shared) | read-modify-write (shared) |
| `0x0020`/`0x003E` | not touched by the AC3 helpers | not touched by the DTS helpers |

**⇒ The next concrete divergence after the timing registers is `0x854`:** AC3 **sets a rate-code field**, while DTS **clears specific high bits** (bits 17/19 via `0xF5FFFF`, bits 20/21 via `0x5FFFFF`) without setting a rate code in these helpers. That is a genuine, previously unrecorded difference in the shared output-control register.

**Label: VERIFIED** (instruction-level), with the *meaning* of those bits **UNRESOLVED**.

---

## §32 — Correction of a claim made in R46b §23.3

R46b stated that "the actual byte-push / IEC61937 framing is done by **hardware**". That over-reached. The statically supported statement is narrower:

- **VERIFIED:** the three timing registers have **no functional firmware reader** (the only reads are a telemetry dump, R46b §21.3).
- **VERIFIED:** the firmware's transmit-side interface that *is* exercised is the shared control register `0x85C` (bit-field read-modify-write) plus the 16-bit command pair `0x0020`/`0x003E`.
- **STRONG INFERENCE (not proof):** because nothing in the firmware consumes the timing values, they are either consumed by the output hardware **or** they are unused/vestigial. The available evidence cannot distinguish these two.

---

## §33 — Verdict and residual

**§30 classification: C — UNRESOLVED.** The DTS timing formulas are structurally recovered (§27) but cannot be shown to be numerically right or wrong for IEC61937, because (i) the register unit is unknown, (ii) required inputs (read-only HW status, the unnamed `SPR 0x2808`) are unknown at runtime, and (iii) three discarded magic multiplies make the printed arithmetic non-certifiable.

**New, self-standing findings (independent of the classification):**
1. `0x86C = (Q + T) & 0xFFFFFF` with a **path-dependent** `T` — the earlier "`0x86C = Q`" was incomplete (§27.1).
2. On the examined path `0x870`'s division numerator is **0**, so `0x870 = 0x86C` there (§27.2).
3. `0x868` is **counter-driven (0..67)×48**, **not** sample-rate-derived — the previous "all three are sample-rate timing quotients" claim is corrected (§27.3).
4. `SPR(0x2808)` is **unnamed** in the authoritative SPR definitions; the tick timer is `SPR 0x5000/0x5001` (§28).
5. The next AC3-vs-DTS control divergence is **`0x854`** (AC3 sets a rate code; DTS clears bits 17/19 and 20/21) (§31).
6. The "hardware performs the framing" claim is downgraded to STRONG INFERENCE (§32).

**Residual (all require datasheet or runtime capture):** the register units; the runtime values of `0x814`, `0x80C`, `SPR(0x2808)`, `word@0x2511A8`; whether the zeroed `0x870` numerator is a defect; whether the discarded multiplies are MAC-accumulate forms; the semantics of `op2A/2B/2E/2F`.

**No patch. No device access. No runtime experiment. No new full-image survey.**

---

## §35 — Follow-up on §27.4: the discarded multiplies (MAC hypothesis refuted; conclusion unchanged)

### §35.1 Hypothesis tested
That the AEON multiply form writes a **MAC register pair** (`MACHI`/`MACLO`) instead of `rD` — which would explain three products that appear dead.

### §35.2 Result 1 — MAC hypothesis **REFUTED** (VERIFIED)
The authoritative slaspec *defines* the macros `mul64s`, `mul64z`, `getmac`, `setmac` (which read/write `MACHI`/`MACLO`), but **no instruction in the model uses them** — a `grep` over `aeon_ORBIS32.sinc` finds the four names only at their own definitions (lines 26/32/38/42). No modelled instruction reads or writes `MACHI`/`MACLO`. ⇒ The MAC explanation is not supported.

### §35.3 Result 2 — `bn.muls rD,rA,rB → rD = rA*rB` is **CONFIRMED** (VERIFIED)
Whole-image Ghidra sweep of **5394** multiply instructions (`scripts/MulUse.java`):

| outcome | count |
|---|---|
| destination subsequently **READ** (result used) | **2087** |
| destination **OVERWRITTEN** before any read | **2703** |
| unresolved (no read/overwrite within 24 insns) | 604 |

A large population of genuinely-read results confirms the `rD` semantics. (Method note: the first run of this script had a parser bug — the `r1` inside mnemonics such as `bg.mfspr1` polluted register detection; after the fix the counts moved by <1%, and `0xD044` was correctly re-classified as overwritten. Both runs are kept: `ghidra_muluse.txt`, `ghidra_muluse2.txt`.)

### §35.4 Result 3 — the three §27.4 sites are dead in the printed stream, and the idiom **recurs**
- `0xCD4C bn.muls r25,r23,r25` → overwritten by `0xCD53 bn.add r25,r23,r26`
- `0xCCF0 bn.muls r24,r23,r24` → overwritten by `0xCCFA bn.srai r24,r25,0x5`
- `0xD044 bn.muls r27,r24,r27` → overwritten by `0xD079 bg.mfspr1 r27,0x2808`

The same idiom — `muls rX,rA,<magic>` → `mfspr1 rT,0x2808` → `add/srai rX,…,rT` — recurs at many other sites (`0x0B86B`, `0x0BB66`, `0x0BCC6`, `0x0D076`, `0x0E0E4`, `0x1011B`, …). ⇒ **systematic, not a one-off.**

### §35.5 Residual hypothesis (UNRESOLVED)
A plausible reading is that **`bg.mfspr1 rX,0x2808` is not a plain SPR read** but an unmodelled instruction that transfers a multiply result — which would turn the idiom into a standard **magic-number division** (`rA + high(rA × magic)`). This is consistent with (a) the slaspec being demonstrably incomplete elsewhere (`op28–0x2F` placeholders; the `op2E` trap model, §19.6), and (b) SPR `0x2808` being unnamed. **It cannot be settled with the available model.** Note the tension: if this reading is right, `SPR(0x2808)` is **not** a timer, and §28's "monotonic counter/time base" reading would be wrong.

### §35.6 Effect on the verdict
**None — classification C (UNRESOLVED) stands.** §27.4's caveat keeps its force (the printed arithmetic is not certifiable), but its scope is now better bounded: the MAC explanation is **excluded**, the multiply semantics are **confirmed**, and the open question is narrowed to the meaning of `bg.mfspr1 rX,0x2808`.

---

## §36 — Resolving §27.4 and §28: `bg.mfspr1 rX,0x2808` is a **multiply-high** transfer

### §36.1 Method
New Ghidra sweep `scripts/FindSPR.java` enumerating **every** `mfspr`/`mfspr1`/`mtspr`/`mtspr1` in the image, with read/write classification and the three preceding instructions. Output `r46_work/ghidra_sprs.txt`.

### §36.2 Test A — adjacency (VERIFIED)
- SPR `0x2808`: **823 reads, 1 write**; **797 of 823 (96.8%) are immediately preceded by a multiply** (`bn.muls`/`bn.mulu`).
- The companion encodings `0x2809`, `0x280A`, `0x280B`, `0x280C`, `0x2821`–`0x2824` each carry **28–64 reads** used in the same way — a whole family of "SPR" encodings behaving as a **multi-lane multiply-result file**, not as system SPRs.

### §36.3 Test B — the arithmetic identity closes **EXACTLY** (VERIFIED, decisive)
In the `0x870` writer:
```
0xD03D/40 r27 = 0x10624DD3
0xD044 bn.muls r27,r24,r27      ; 64-bit product = rate * 0x10624DD3
0xD047 bg.mfspr1 r28,0x2808     ; r28 = HIGH(product)
0xD04E bn.srai r25,r28,0x6      ; r25 = high >> 6
```
`0x10624DD3` is the magic constant for **division by 1000 with a 38-bit shift**; the code applies `>>32` (implicitly, via the multiply-high) then `>>6` = `>>38`. Verified numerically:

| rate | `high(rate × 0x10624DD3)` | `>>6` | `rate/1000` |
|---|---|---|---|
| 48000 | 3072 | **48** | 48.0 |
| 44100 | 2822 | **44** | 44.1 |
| 96000 | 6144 | **96** | 96.0 |
| 192000 | 12288 | **192** | 192.0 |

⇒ **`r25 = rate / 1000` exactly.** The identity closes for every rate, which **proves** the multiply-high semantics — this is not merely an adjacency correlation.

### §36.4 Corrections
- **§27.4 RESOLVED.** The three multiplies are **not dead**: their **high words are consumed** by the immediately following `bg.mfspr1 rX,0x2808`. The "dead multiply" appearance was an artefact of the slaspec modelling that instruction as a system-SPR read.
- **§28 CORRECTED.** `SPR(0x2808)` is **not** a monotonic counter / time base. It is the **multiply-result transfer** encoding. (One `mtspr1 r30,0x2808` write exists at `0x0AF13`; a single site, itself unexplained, and it does not change the reading.)
- **§35.5 CONFIRMED** — the residual hypothesis was correct.

### §36.5 Revised, now-complete formulas
```
high_M(x) := (x * M) >> 32      where M is the constant of the immediately preceding multiply

0x86C:  Q = ((state + high_A57E(state)) >> 11) − sign(state)      state = word@0x2511A8
        0x86C = (Q + T) & 0xFFFFFF          (T as in §27.1)
0x870:  D = (rate/1000) × 4 or × 16          (rate = ctx[0x1c])
        0x870 = ((0 / D) + Q) & 0xFFFFFF  =  Q & 0xFFFFFF
0x868:  ((counter(0..67) × 48) + r25) & 0xFFFFFF
```
Note: `0xA57EB503 / 2³² = 0.646464646… = 64/99 **exactly**` ⇒ the constant is a **rational Q32** value and the arithmetic is **exact**, not approximate. The `0x86C` base is therefore `state × (1 + 64/99) = state × 163/99`, then `>>11` with a sign correction.

### §36.6 Effect on the classification (§30)
§30's classification C rested on three reasons:
1. register unit unknown — **still true**;
2. required inputs unknown — **largely removed**: the "unknown timer" `SPR(0x2808)` is now identified as the multiply-high transfer, and the divisor is a plain **`rate/1000`** term;
3. arithmetic non-certifiable — **REMOVED**: the arithmetic now closes exactly.

⇒ **Classification remains C, but for a single narrow reason: the unit of the register value is unknown.** Positive consequence: the DTS timing values are **shown to be well-formed fixed-point quantities derived from the accumulated state and the sample rate (via `rate/1000`)** — so **H1 can no longer be argued from "the arithmetic looks broken"**. The one remaining unexplained code detail is the **zeroed numerator** in the `0x870` path (§27.2).

---

## §37 — Provenance of the "state" input: `word@0x2511A8` is an **elapsed time delta**

§28 left `word@0x2511A8` untraced, which is what kept the register **unit** unknown. This section closes it.

### §37.1 All four writers of `word@0x2511A8` (VERIFIED, Ghidra)
New sweep `scripts/FindDataWriters.java` (same register tracking as `FindMMIO`, targets = data range `0x251180–0x251300`). Output `r46_work/ghidra_datawriters.txt`. Stores to `0x2511A8`:

| site | value stored |
|---|---|
| `0xF50F` | `bn.lwz r4,0x10(r10)` − `bn.lwz r23,0xc(r10)` ⇒ **`ctx[0x10] − ctx[0xc]`** |
| `0xF613` | same delta, or **`(ctx[0x10] − ctx[0xc]) + ctx[0x8]`** when the delta is non-negative |
| `0x10B25` | **`0`** (`bn.sw 0x14(r10),r0`) — an explicit reset |
| `0x18176` | **the constant `0x1B8D74`** (1,807,732) — written together with `ctx[0x8]`, `ctx[0xc]`, `ctx[0x10]`, `ctx[0x14]`, `ctx[0x18]` in the SDO-packer cluster |

Also confirmed: `word@0x2512C4` has exactly two writers, `0x10C29` (writes 0) and `0x172FD` (the DTS SDO packer — matching R46 §15).

### §37.2 The DTS helpers themselves set the two timestamps (VERIFIED)
`ctx[0xc]` and `ctx[0x10]` — the fields whose **difference** becomes the state word — are written by the DTS timing helpers:
- `0xCC9B` PATH B: `ctx[0x10] = ctx[0xc] = 0xD72000` (`0xCD2E`, `0xCD31`)
- `0xCF86`: `ctx[0x10] = ctx[0xc] = 0xDD8000` (`0xD02B`, `0xD02E`)

⇒ **The DTS path writes the timestamps and then measures their difference** — a coherent **measurement-pair structure**: one helper arms a time reference, the other measures the elapsed interval and writes it out. (Consistent detail: immediately after arming, `ctx[0x10] − ctx[0xc] = 0`, i.e. the delta starts at zero and grows.)

### §37.3 Consequence — the registers are INTERVAL-DERIVED (STRONG INFERENCE)
```
state = ctx[0x10] − ctx[0xc]        (an ELAPSED INTERVAL, optionally + ctx[0x8])
0x86C = ((state × 163/99) >> 11) − sign(state) + T          (T as §27.1)
0x870 = Q = ((state + high(state × 0xA57EB503)) >> 11) − sign(state)
```
The `0x86C`/`0x870` values are therefore **derived from an elapsed time interval**, not from a sample index or an absolute counter. **This is direct structural support for the "output timing" classification** (and hence for H1's premise), and it is the first *semantic* grounding obtained for these registers that does not rest on their address.

### §37.4 The unit reduces to one question (still datasheet-bound)
The scale factor is `163/99 / 2048 = 0.0008039`. So the register value = `elapsed_ticks × 0.0008039`, and the only remaining unknown is **what `ctx[0xc]`/`ctx[0x10]` count**. The literals available as clues are `0xD72000` (14,098,432), `0xDD8000` (14,516,224) and the initialiser `0x1B8D74` (1,807,732). No static step can convert these to a physical unit without the datasheet or a runtime sample.

### §37.5 Effect on the classification
- The **semantics** of the three registers is now well-supported: `0x86C`/`0x870` are **interval/timing** values; `0x868` remains **counter-driven** (§27.3).
- Classification remains **C**, but the residual is now a single, sharply-defined question: *the unit of the timestamp literals*. H1's premise (DTS writes timing that could be wrong) is **structurally supported**, while H1's *conclusion* still **cannot be confirmed or refuted** without a numeric reference.

---

## §38 — Correction to §37.2, and the real feeders of the global timestamps

### §38.1 CORRECTION — §37.2's "measurement pair" claim is WITHDRAWN
§37.2 asserted that the DTS timing helpers themselves set the two timestamps whose difference becomes the state word. **That was wrong: it conflated two different structures.**

- The **state word** is read by the helpers from the **global** structure: `0xCD3D`/`0xD067` build `r10 = 0x251194`, then `bn.lwz rN,0x14(r10)` → `0x2511A8`. (VERIFIED)
- But the helpers' writes to `0xc(r3)`/`0x10(r3)` (`0xCCCD`/`0xCCD0` in `0xCC9B`; `0xD02B`/`0xD02E` in `0xCF86`) use **`r3`, their argument** — and the callers pass **`0x4E6AD4`** (`0xB290` @`0xB368`) and **`0x25332C`** (`0xB290` @`0xB37B`), which are **different structures**, not `0x251194`.

⇒ The helpers write fields of *their own argument structure*; they do **not** arm the global timestamps. The "one helper arms a reference, the other measures the interval" reading is **withdrawn**. §37.1 and §37.3 (the state word is a difference of two global words) are unaffected.

### §38.2 The global "timestamps" are a **rolling counter** (VERIFIED, Ghidra)
`0x11674–0x1168D`:
```
0x11674 bn.lwz r26,0xc(r10)          → r26 = global[0x2511A0]
0x11677 bn.lwz r23,0x4(r10)          → r23 = global[0x251198]
0x1167A bg.addi r26,r26,0x1080       → r26 += 0x1080   (4224)
0x1167E bn.sw 0xc(r10),r26           → global[0x2511A0] = r26          ← INCREMENT
0x11681 bg.bgt r23,r26,0x11690       → if global[0x251198] > r26, done
0x11685/89 r26 = global[0x251194]    → else reload the BASE
0x1168D bn.sw 0xc(r10),r26           → global[0x2511A0] = base          ← WRAP
```
⇒ `global[0x251194]` = **base**, `global[0x251198]` = **limit**, `global[0x2511A0]` = **current**, and the counter advances by **`0x1080` (4224) per step** and wraps to the base when it exceeds the limit. Other writers of the pair are `0xF674`/`0x10AD5` (`0x2511A4`), `0x10B08`/`0x1167E`/`0x1168D` (`0x2511A0`) and the initialiser `0x1816A–0x18179` (all set to `0x1B8D74`).

### §38.3 A concrete numeric prediction (for a future runtime check)
The state word is normally one step of that counter, i.e. **`state = 0x1080 = 4224`**:
```
high(4224 × 0xA57EB503) = (4224 × 274877907) >> 32 = 270
0x86C = ((4224 + 270) >> 11) − 0 = 4494 >> 11 = 2
0x870 = Q                                        = 2
```
⇒ **on the normal path `0x86C = 0x870 = 2`** (before the §27.1 additive term `T`). This is a **falsifiable prediction**: a runtime capture of `0xB000_086C` during DTS passthrough should read `2 + T`. It also means the register values are **small integers**, which is a further reason to doubt any "large timing quotient" reading.

### §38.4 Status
- The **semantics** chain is now: rolling counter (step `0x1080`) → snapshot difference → `state` → `0x86C`/`0x870`. That is an **interval-derived** quantity, so the "output timing" classification keeps its structural support.
- The **unit** remains unknown: `0x1080` (4224) per step is the only concrete magnitude, and whether it is samples, bytes, ticks or something else cannot be decided without the datasheet or a runtime sample.
- **Classification remains C.** The residual is unchanged and now sharper: *what does one counter step (`0x1080`) represent?*

---

## §39 — The counter's base and limit are compile-time constants: a **64-step cycle**

`0x10A50–0x10B10` (the routine that initialises the global structure):
```
0x10A71 bn.movhi_2 r25,0x81
0x10A74 bn.movhi_2 r26,0x7d
0x10A7B bg.ori r25,r25,0x9800      → r25 = 0x819800 = 8,492,032
0x10A7F bg.ori r26,r26,0x7800      → r26 = 0x7D7800 = 8,222,720
0x10AD5 bn.sw 0x10(r10),r26        → global[0x2511A4] = 0x7D7800
0x10B01 bg.sw_0 0x1194(r11),r26    → global[0x251194] (BASE)  = 0x7D7800
0x10B05 bn.sw 0x4(r10),r25         → global[0x251198] (LIMIT) = 0x819800
0x10B08 bn.sw 0xc(r10),r26         → global[0x2511A0] (CURRENT) = 0x7D7800
0x10B0B bn.sw 0x54(r10),r25        → and again to +0x54
0x10B0E bn.sw 0x60(r10),r25        → and again to +0x60
```

**Constants (VERIFIED):**
```
BASE  = 0x7D7800 = 8,222,720
LIMIT = 0x819800 = 8,492,032
LIMIT − BASE = 0x42000 = 270,336 = 64 × 4224
STEP  = 0x1080  = 4,224
```
⇒ **The counter is a 64-step cycle**: it advances by `4224` per step and wraps to `BASE` after **64** steps (`270,336 = 64 × 4224`). Both `BASE` and `LIMIT` are **compile-time constants**, and `CURRENT` is initialised to `BASE` (so the first delta is one step).

**Refined numeric prediction (§38.3):** the state is a multiple of `4224`, so
```
one step  : state = 4,224  ⇒ 0x86C = 0x870 = 2      (high(4224×0xA57EB503) = 270; (4224+270)>>11 = 2)
full cycle: state = 270,336 ⇒ 0x86C = 0x870 = 217   ((270336+174795)>>11 = 217)
```
⇒ the registers hold a value in roughly **2 … 217** depending on how many counter steps elapsed between the two snapshots.

**Note:** `BASE`/`LIMIT` straddle `2²³ = 0x800000` but not symmetrically (`−0x28800` / `+0x19800`), so no obvious "physical" anchor is implied by the constants themselves.

### §39.1 Status
- The full chain is now: **compile-time `BASE`/`LIMIT` + step `0x1080` → modulo-64 counter → snapshot difference → `state` → `0x86C`/`0x870`**.
- The **single remaining unknown is the meaning of one step (`0x1080` = 4224)**. `BASE`/`LIMIT` are arbitrary offsets in that unit, so they carry no independent information.
- **Classification remains C**, unchanged. This is now the narrowest possible statement of the residual.

---

## §34 — Tooling and artifacts

| Artifact | Content |
|---|---|
| `r46_work/ghidra_formulas.txt` | Ghidra ground truth: full `0xCC9B` (0xCC9B–0xCE6B), `0xCF86` (0xCF86–0xD286), `0x868`-writer context (0x17830–0x17900) |
| `r46_work/ghidra_writers.txt` | (R46b) full-image MMIO inventory used for §28/§31 |
| `aeon_ghidra_public/.../aeon_SPRs.sinc` | authoritative SPR names — established that `0x2808` is unnamed and that `TTCR` = SPR `0x5001` |
| `scripts/FindMMIO.java` | (R46b) Ghidra full-image MMIO sweep |
| `scripts/MulUse.java` | Ghidra full-image multiply-usage analysis (§35) — `OUT:<path>`; outputs `r46_work/ghidra_muluse2.txt` |
| `scripts/FindSPR.java` | Ghidra full-image SPR-access enumeration (§36) — `OUT:<path>`; outputs `r46_work/ghidra_sprs.txt` |
| `scripts/FindDataWriters.java` | Ghidra full-image writer/reader search for the data words `0x2511A8` / `0x2512C4` / `0x251194` (§37) — `OUT:<path>`; outputs `r46_work/ghidra_datawriters.txt` + `ghidra_state.txt` |
| `aeon_ghidra_public/.../aeon_ORBIS32.sinc` | authoritative ISA — established that the MAC macros are unused (§35.2) |
