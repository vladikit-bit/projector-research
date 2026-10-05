# R46 — FINAL SUMMARY (consolidated handover)

**Subject:** Thundeal TD98 Pro / MStar MT5889 (Android 11) — AEON (R2) audio DSP
**Image:** `snd_full.bin` (SND R2 DSP firmware, 1,839,920 bytes, base 0)
**Symptom under investigation:** DTS passthrough over SPDIF/HDMI-ARC produces no output; AC3 passthrough works.
**Method:** static-only. **No patch. No device access. No runtime MMIO.** (R43 stop respected throughout.)
**This document consolidates** the full phase report `REPORT_runtime_phase46.md` (§0–§20 + appendices) **and** `REPORT_runtime_phase46b.md` (§21–§24: Ghidra-only writer verification + AC3/DTS post-quotient differential). Where they differ, **this summary and R46b are authoritative** — the phase report contains superseded statements kept for provenance.

> **R46b closes the writer-count risk register (R1/R3/R4) with Ghidra-only evidence** (§21), corrects the "write-only / never read back" wording (§21.3), identifies the **single structurally-supported AC3-vs-DTS asymmetry** (§22: DTS writes the quotient to `0x868`/`0x86C`/`0x870`; AC3 discards it and leaves hardware defaults), and locates the **firmware transmit interface** (§23: control register `0x85C` used as bit-field read-modify-write, plus the 16-bit command pair `0x0020`/`0x003E`).
>
> **R46c (exact formulas) → classification C (UNRESOLVED) — but narrowed by §36.** The DTS timing formulas are structurally recovered. **§36 then resolved two of the three original blockers**: `bg.mfspr1 rX,0x2808` is a **multiply-high transfer** (96.8% of its 823 reads immediately follow a multiply, and the `/1000` magic identity closes *exactly*: `high(rate × 0x10624DD3) >> 6 = rate/1000` for 48k/44.1k/96k/192k), so the three "discarded" multiplies are **not dead** and the arithmetic is **exact, not broken**. `0x870`'s divisor is therefore a genuine **`rate/1000`** (kHz) term. ⇒ Classification stays **C solely because the register unit is unknown**; **H1 can no longer be argued from "the arithmetic looks broken"**.
>
> **R46c §37–§38 (provenance of the state input):** `word@0x2511A8` has exactly **4 writers**, and three compute it as a **difference of two global words** (`0x2511A4 − 0x2511A0`). Those two globals are a **rolling counter**: `0x11674–0x1168D` does `global[0x2511A0] += 0x1080` (4224) and wraps to `global[0x251194]` when it exceeds `global[0x251198]`. ⇒ **`0x86C`/`0x870` are interval-derived values** — the first *semantic* grounding for these registers that does not rest on their address. **§38.1 CORRECTION:** §37.2's "the helpers arm the timestamps" claim is **withdrawn** — the helpers write fields of their *argument* structure (`0x4E6AD4`/`0x25332C`), not the global `0x251194` one.
>
> **§39 — the counter is a 64-step cycle:** `BASE = 0x7D7800` (8,222,720) and `LIMIT = 0x819800` (8,492,032) are **compile-time constants** with `LIMIT − BASE = 0x42000 = 270,336 = 64 × 4224`; the step is `0x1080` (4224). So the counter advances 4224 per step and wraps after 64 steps.
>
> **Concrete numeric prediction (§38.3/§39):** the state is a multiple of 4224, so **`0x86C = 0x870 = 2`** for one step and **`217`** for a full 64-step cycle (before the additive term `T`) — a falsifiable runtime check. The registers therefore hold **small integers** (≈2…217), a further reason to doubt any "large timing quotient" reading. **The single remaining unknown is the meaning of one step (`0x1080` = 4224)** — `BASE`/`LIMIT` are arbitrary offsets in that unit. One unexplained code detail remains: the zeroed `0x870` numerator. Corrections from R46c: **`0x86C` has a path-dependent additive term** (§19.1 was incomplete); **`0x870`'s division numerator is provably 0** on the examined path (so `0x870 = 0x86C` there); **`0x868` is counter-driven (0..67)×48, NOT sample-rate-derived**; and the next AC3-vs-DTS control divergence is **`0x854`**. The "hardware performs the framing" claim is downgraded to STRONG INFERENCE.

---

## 1. Verdict

**R46-B** for the register question, **R46-FINAL-C** for the root-cause question.

- **R46-B (register inventory — stands):** `0xB000_086C` / `0xB000_0870` / `0xB000_0868` are **DTS-path-exclusive MMIO registers** in the SPDIF/HDMI-ARC output block (`0xB000_0800–0x08FF`) — each with **exactly one writer in the whole image (Ghidra-verified, R46b §21.2)** and **no functional reader** (one telemetry reader each, R46b §21.3). **AC3 writes none of them.**
- **Their *semantic* role is NOT established.** The earlier label "sample-rate-derived timing registers" is **withdrawn** (R46c §27.3, §40): `0x868` is counter-driven, and the `0x86C`/`0x870` input is a **ring-buffer byte offset**, not a time. Treat their function as **UNRESOLVED**.
- **R46-FINAL-C:** the DTS-path gate `0x25EE1` is real and DTS-specific, **but the claim "gate FAIL ⇒ no physical SPDIF TX" is NOT PROVEN and is withdrawn.** The gate's downstream callee `0x13292` is a **generic arithmetic helper**, not a transmitter, and is **not gate-exclusive**.
- **No patch.** Resolution paths (if any) are runtime/config/licence, or a corrected timing value — both outside static reach.

---

## 2. What is VERIFIED (Ghidra ground truth where marked "G")

| # | Finding | Evidence |
|---|---|---|
| 1 | AC3 helpers (`0xCB50`, `0xCAB4`, `0xC49D`, `0xC5C5`) touch **only** `0x854` (R/W), `0x850` (wr 0), `0x84C` (wr 0), `0x814` (rd), `0x80C` (rd) — **never** `0x86C`/`0x870`/`0x868` | G (`ghidra_ac3.txt`) |
| 2 | DTS writes `0x86C` (`0xCD76`, helper `0xCC9B`), `0x870` (`0xD0A1`, helper `0xCF86`), `0x868` (`0x178B9`) | G |
| 3 | The `0x86C`/`0x870` writers compute **the same base quantity** `((word@0x2511A8 + SPR(0x2808)) >> 11) − sign`, masked to 24 bits | G (`ghidra_timing.txt`) |
| 4 | The `0x870` writer **dispatches on the sample rate** read from `0xB000_0814`, comparing against literals **88200 / 128000 / 176400 / 192000** | G |
| 5 | The **AC3** helper `0xCB50` computes the **identical** quotient from the same state word and the same SPR `0x2808`, and dispatches on **88200 / 44100 / 64000 / 32000** | G |
| 6 | The `0xB290` dispatch calls the `0x86C`/`0x870` writers at `0xB368`/`0xB37B`, **conditionally** (`bn.beqi …,1` can skip each) | G |
| 7 | Gate `0x25EE1` predicate: `r11 = bit7(byte@0xB000_0C06) AND (signed byte@0xB000_0015 <= −1) AND (word@0x2512C4 ≠ 0 ⇒ clear)` | G |
| 8 | The gate **returns** (`0x25F7E` epilogue → `0x25F81 bt.jr r9`) and returns a status in `r3` that **all four callers branch on** | G |
| 9 | Gate call sites: exactly **4** — `0xCECC`(∈`0xC91A`), `0xD2C8`, `0xEC82`(∈`0xE9E5`), `0xEF5E`(∈`0xED6E`); none is an AC3 helper | G |
| 10 | `0xB000_0C06` and `0xB000_0015` have **no writer among decodable instructions** | Python scan (see risk R1) |
| 11 | `word@0x2512C4` is written once, at `0x172FD`, in the DTS-X SDO-packer cluster | Python scan |
| 12 | `0x13292` is **shared integer arithmetic / structure-store code** — remainder idioms, `×0x30` modulo, struct stores at `r10+0x0..0x2c` — with **no MMIO / DMA descriptor / FIFO / trigger** | G (`ghidra_txchain.txt`) |
| 13 | `0x10786E` is a **generic wrapper** (1919 call sites); `0x115693` is a **critical-section + fixed-point** helper (writes SR via SPR `0x11`, stores 0 → `0x9000_0001`) — neither is an audio transmitter | G |
| 14 | The gate body contains **no call to the timing writers**, and there is **no caller/callee edge** between the gate and `0xB290` in either direction | G + Python |
| 15 | `0xA57EB503` multiply in both timing writers is **dead code** (immediately overwritten by `bn.add`) | G |

---

## 3. Hypotheses

### 3.1 Ranking (final)
- **H1 — DTS writes a wrong/unmatched timing value.** *Live.* Now better-founded **semantically** (§3.2) but its **value correctness is UNRESOLVED** (needs datasheet or runtime capture).
- **H2 — the gate blocks TX.** *Live, but its mechanism is NOT established.* What is proven is that the gate is DTS-path-only and calls `0x13292` on PASS; what is **not** proven is that this call (or anything gated by it) performs physical TX.

**R46b update (§22.5) — H1 is now the only *structurally-supported* asymmetry:** AC3 computes the same quotient but **discards it**, writing only a **rate code** to `0x854` and leaving `0x868`/`0x86C`/`0x870` at hardware defaults; DTS **writes the quotient** into those registers. This asymmetry alone is sufficient to explain "AC3 works, DTS fails", so **H1 no longer needs H2** to be invoked. H2 is neither confirmed nor refuted by it.

**R46b update (§23.3) — transmit interface located:** the firmware's transmit-side interface is the shared control register **`0xB000_085C`** (used as bit-field **read-modify-write** at ten paired `lwz`→`sw` sites) plus the **16-bit command/value pair `0xB000_0020`/`0x003E`**. The DTS timing registers sit outside both: they are **write-once hardware parameters** with no firmware consumer. ⇒ byte-push / IEC61937 framing is performed by **hardware**; the firmware only configures it.
- **H3 — AC3 uses different configuration.** **Ruled out** for `0x86C`/`0x870`/`0x868` (AC3 never constructs them).

H1 and H2 are **co-equal and may co-occur**. (An earlier ranking that put H2 first, and before that H1 first, is superseded.)

### 3.2 Why the timing-register classification was upgraded — **and then corrected by R46c**
Both AC3 and DTS compute the **same** base quotient from `word@0x2511A8` + `SPR(0x2808)`, and both dispatch on literal sample-rate constants. That moved the classification of `0x86C`/`0x870` toward "strongly corroborated sample-rate-derived timing".

**R46c corrections (§27):**
- **`0x868` is NOT sample-rate-derived.** It is driven by a **wrapping counter (range 0..67) × 48**. The "all three carry sample-rate-derived timing quotients" statement is **wrong for `0x868`**.
- **`0x86C` has a path-dependent additive term** `T = (SPR>>5) − sign(ctx[0x10]−ctx[0x0c])`, present only when `(word@0x814 << 10) >= 0`. The earlier `0x86C = Q` was incomplete.
- **`0x870`'s division numerator is zeroed** (`bt.movi r23,0x0` at `0xD034`), so on the examined path `0x870 = 0x86C`.
- **Three "magic" multiplies are discarded** (`0xA57EB503`, `0x2AAAAAAB`, `0x10624DD3`) — compiler output does not normally contain three distinct dead magic multiplies, so the printed arithmetic is **not certifiable** (candidate explanation: the multiply form writes a MAC register pair; UNRESOLVED).
- `SPR(0x2808)` is **unnamed** in the authoritative SPR definitions (the tick timer is `SPR 0x5000/0x5001`).

⇒ The classification of these registers is **weaker** than §3.2 originally claimed: their role is plausible but their **unit and numeric correctness are UNRESOLVED (classification C)**.

---

## 4. Refuted / withdrawn

| Claim | Status |
|---|---|
| `0x13292` is the SPDIF/IEC output **DMA** | **REFUTED** — it is a generic arithmetic helper with no hardware access (G) |
| Variant A / Variant B (gate→`0x13292`→… = TX arm) | **REFUTED** |
| "gate `0x25EE1` FAIL ⇒ zero physical SPDIF activity" | **WITHDRAWN** — unproven |
| "H2 is the definitive root cause" | **WITHDRAWN** — H1/H2 co-equal |
| `r31` is a threshold operand in the gate predicate | **REFUTED** — `0x25F14` is `bn.sflesi r23,-0x1`, an **immediate** |
| "`bg.op2E` = MMIO-touch op" | **REFUTED** — `op2E` is a real but unmodelled DSP op (5× in straight-line code) |
| "the gate never returns" | **REFUTED** — `bt.jr r9` at `0x25F81` |
| "`0xB290` is called by `0xC226` only" | **REFUTED** — ≥6 call sites (see R2) |

---

## 4b. Superseded-claims register (what NOT to believe from the older text)

Several statements in `REPORT_runtime_phase46.md` (§0–§20) and `REPORT_runtime_phase46b.md` were corrected by R46c. Both reports now carry a banner pointing here. **Do not quote the stale form.**

| Stale claim | Status | Corrected in |
|---|---|---|
| `0x86C`/`0x870` are **write-only**, never read back | **DISPROVED** — one telemetry reader each (`0xE674`/`0xE68D`/`0xE6A6`) | R46b §21.3 |
| timer operand is **`SPR(r5,16)`** | **WRONG** — it is `bg.mfspr1 rN,0x2808` | R46c §18.3 |
| `SPR 0x2808` is a **timer / time base** | **WRONG** — it is a **multiply-high transfer** | R46c §36.4 |
| the three registers are **sample-rate-derived timing** | **WITHDRAWN** — `0x868` is counter-driven; the `0x86C`/`0x870` input is a ring-buffer byte offset | R46c §27.3, §40 |
| the `×0xA57EB503` multiply is **dead code** | **WRONG** — its high word is consumed; arithmetic is exact (`= 64/99`) | R46c §36 |
| `0x86C = Q` and `0x870 = (0xDD8000/D) + Q` | **INCOMPLETE** — `0x86C` has a path term `T`; `0x870`'s numerator is zeroed | R46c §27.1/§27.2 |
| `func 0xB290` is called by `0xC226` **only** | **WRONG** — ≥6 call sites | R46c §20.3 |
| the byte-push / IEC61937 framing is **done by hardware** | **DOWNGRADED** to STRONG INFERENCE | R46c §32 |
| `0x86C`/`0x870` are **interval-derived timing** | **WITHDRAWN** (superseded within R46c itself) | R46c §40 |
| Python-walk **caller counts** | **LOWER BOUNDS** — the walk drifts out of phase and misses sites | R46c §20.4 |

---

## 5. Risk register — claims that are COVERAGE-LIMITED (lower bounds)

The Python decoder (`scripts/aeon_decode.py`) has **three proven defects**, all of which can corrupt negative/exclusivity claims:

1. **20.3% of the image is UNDEFINED** (471,745 / 591,620 instruction starts decode). Every "no writer found" is "not found among decodable instructions".
2. **It renders some immediates as registers** (e.g. `bn.sflesi rD,imm5` → printed as `r31`), which manufactured a phantom threshold.
3. **Its linear walk drifts out of phase and misses call sites** — proven by `0xB290`: the scan reported 1 caller function; Ghidra shows ≥6 call sites.

Consequently, the following are **lower bounds, not proofs**, and should be re-verified with Ghidra before being relied on:

- ~~**R1 — "`0xB000_0C06` / `0xB000_0015` have zero writers"**~~ → **CLOSED (R46b §21.2): VERIFIED BY GHIDRA** — 0 writers each, 4 readers each.
- **R2 — caller counts** (still OPEN): `0xB290` (≥6, Ghidra-confirmed), `0x13292` (≥12 / ≥9 functions), the timing writers `0xCC9B`/`0xCF86`, the gate `0x25EE1` (4). Not re-verified exhaustively.
- ~~**R3 — "`0x86C`/`0x870` have exactly ONE writer each in the whole image."**~~ → **CLOSED (R46b §21.2): VERIFIED BY GHIDRA** — exactly one writer each (`0xCD76`, `0xD0A1`), confirmed by two independent decoder-free methods.
- ~~**R4 — "`0x868` is written by exactly one non-DTS function (`0x178B9`)."**~~ → **CLOSED (R46b §21.2): VERIFIED BY GHIDRA** — exactly one writer (`0x178B9`).

**New finding that corrects a previous statement (R46b §21.3):** the registers are **not** literally "write-only / never read back". Each has **exactly one reader** — `0xE674` / `0xE68D` / `0xE6A6` — and all three are part of a **straight-line telemetry dump** (`0xE60C–0xE702`) that reads `0x858…0x87C` at +4 and feeds the generic logging helper `0x10786E`. ⇒ **no functional firmware consumer**; the corrected wording is *"no functional reader; one telemetry reader each"*.

**Method used (no Python decoder):** new Ghidra postscript `scripts/FindMMIO.java` (587,030 instructions swept; 106 MMIO stores / 281 MMIO loads) cross-checked against a decoder-free big-endian word-pattern scan (`r46_work/r46_bytescan.py`). Both agree: exactly three writers in total.

---

## 6. Remaining unknowns (all runtime/datasheet-bound)

1. Semantics of `op2A` / `op2B` / `op2E` / `op2F` (real but unmodelled DSP ops).
2. Runtime values of `bit7@0xB000_0C06` and `byte@0xB000_0015` during DTS vs AC3.
3. Correctness of the `0x86C` / `0x870` / `0x868` quotients for the DTS sample rate / IEC61937 framing.

## 7. Decisive experiment (specified; BLOCKED by the R43 device-access stop)

On **real Kodi DTS passthrough** (the only path that engages the gate — `probe7` does not):
1. capture `bit7@0xB000_0C06` and `byte@0xB000_0015` → tests H2;
2. capture `0x86C` / `0x870` / `0x868` and compare against the IEC61937 burst-period/pause-interval for the DTS rate, and against AC3's defaults → tests H1;
3. capture `0xB000_0814` (the status that drives the rate dispatch and the conditional timing writes).

**Blocked because:** `/dev/malloc` MMIO read panics the device (R43); `DM[]` does not reflect hardware MMIO writes; no safe vendor dump exposes the SHM/MMIO region.

---

## 8. Tooling (reproducible)

| Tool | Purpose |
|---|---|
| `scripts/aeon_decode.py` | Python AEON decoder — **known-defective** (see §5); use only with the caveats above |
| `r46_work/r46_scan.py` | Full-image linear disassembler + backward register resolver (Python) |
| `r46_work/r46_txchain2.py` / `r46_txchain3.py` | Caller sets, caller contexts, hardware footprint (corrected parsers) |
| `r46_work/r46_verify_sites.py` | Re-derives call sites from raw bytes + decode-density control |
| `r46_work/r46_opcodes.py` | Slaspec cross-check; opcode/decoder-field histogram |
| `scripts/DumpAEON_R29.java` | **Ghidra headless ground-truth dumper** (`SWEEP:START:LEN`) — the authoritative tool |
| Ghidra slaspec | `aeon_ghidra_public/aeon/data/languages/aeon.slaspec` |
| Ground-truth dumps | `r46_work/ghidra_txchain.txt`, `ghidra_timing.txt`, `ghidra_ac3.txt`, `ghidra_c380.txt`, `ghidra_c2f0.txt` |

**Ghidra recipe:** see `aeon-snd-static-trace` skill §1 (env `JAVA_HOME`/`APPDATA`/`HOME`, reuse project `r27_work/ghidra_r27f` / `sndr27`, `-process snd_full.bin -noanalysis`, bare-hex `SWEEP:` args).

---

## 9. Correction chain R44 → R46 (provenance)

| Phase | Claim then | Status now |
|---|---|---|
| R44 | `0x86C`/`0x868`/`0x870` are DTS-only hardware registers (offset-only match) | Partly right; base verification was missing |
| R45 | Corrected: base-verified; `0x86C`/`0x870` = MMIO, `0xA86C`/`0xA868` = SHM | Holds |
| R46 | "exactly one writer each; write-only; DTS-exclusive" | **Lower bound** (R3) |
| R46 §13 | gate does not gate the timing write; `0xB290` called only from `0xC226` | Conclusion holds; **caller detail wrong** (R2) |
| R46 §14 | "gate gates DTS TX (DMA)" | **Refuted** — callee is not DMA |
| R46 §15 | "H2 is the definitive root cause" | **Withdrawn** — H1/H2 co-equal; zero-writer claim is R1 |
| R46 §16 | timing formulas (with `SPR(r5,16)`) | Corrected: **SPR `0x2808`**; `+r25` term; divisor `<<2` twice |
| R46 §17 | `0x13292` semantics unresolved; not gate-exclusive | Confirmed, and resolved in §18 |
| R46 §18 | `0x13292` = arithmetic helper; gate returns; no `r31` | Holds (Ghidra) |
| R46 §19 | §16 validated; sample-rate dispatch found | Holds (Ghidra) |
| R46 §20 | AC3 differential confirmed; shared quotient; caller count corrected | Holds (Ghidra) |

**Bottom line:** every load-bearing conclusion has been re-grounded on Ghidra ground truth, **except** the writer-exclusivity claims (R1/R3/R4), which remain lower bounds from the defective walk and are the recommended next verification.
