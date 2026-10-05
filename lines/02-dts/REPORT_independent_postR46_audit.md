# Independent Post-R46 Audit — AEON (R2) `snd_full.bin`
## DTS passthrough failure vs AC3 passthrough success (MStar MT5889 / Thundeal TD98 Pro / Android 11)

**Audit scope:** R47–R52 (produced by a *different* agent), plus independent re-validation.
**Auditor posture:** adversarial — every later conclusion re-tested against raw evidence; no conclusion accepted merely because it was documented.
**Device safety:** all runtime work used ONLY the previously validated `read_dsp_sram_type=1` mirror. No patch, no module, no config/property/EDID change, no MMIO write, no `/dev/malloc`, no `/dev/mem`, no `type=2`, no new memory primitive.
**Date:** 2026-09-11.

---

## 1. Executive conclusion

R47–R52 established one thing **solidly** and built a large structure of **unsupported inference** on top of it.

**Solid:** the AC3-vs-DTS divergence is real, sustained, and independently reproduced. During confirmed playback, the DSP status block `0x0C00-0x0C07` is **repeatedly non-zero for AC3** and **repeatedly all-zero for DTS**, while the decoder-liveness region `0x0FE8-0x0FEC` is live in **both**. Classification: **VERIFIED BY RUNTIME**, reproduced by this audit with address-verified reads.

**The structural problems, in one line each:**
- **R50's gate premise is factually wrong.** It claimed `mem[0x1c(r10)]` at `0x1F605` is a snapshot of `mem[0x8(r10)]`. Ground truth: the preceding `jal 0xB544A` **clobbers `r3`**, so `0x1F605` stores that call's **return value**; and the actual gate predicate `r14` at `0x1F6DA` is the **return value of a later call (`jal 0xB6D8F` at `0x1F67B`)**. R50's chain is broken at link one, and its headline "the gate tests `mem[0x8]`" is **INCORRECT**.
- **R48's "bits 17|19 = AC3/DTS selector" is an OVERCLAIM.** R46d itself recorded those bits' semantics as UNKNOWN; R48 upgraded correlation to causation without a mechanism.
- **R51/R52 converted a negative search into architecture.** "0 static callers found" was rewritten as "the dispatcher is runtime-mediated." The defensible statement is only: *the dispatcher/context construction could not be recovered from the statically decoded image.* **OVERCLAIMED.**
- **R47e's load-bearing "0 writers of `0x0C00-0x0C07`" came from the Python interpreter**, which standing rules deem non-authoritative. This audit re-ran it in **Ghidra** and it **holds** (0 stores found among all identifiable base constructions) — but it was not evidence *as originally sourced*.
- **Two methodological artifacts** (stale `dmesg | tail` counting in R47c; a 4-byte-step disassembly in R48's `callers_predA.txt`) were present in the evidence chain. R47c's *conclusion* survives (the DTS status block is genuinely, sustainedly zero), but its per-sample counting was inflated.

**Net position:** the failure is localised to the **DTS-specific output/arm path downstream of a shared, decoder-live upstream**, and the first *proven* AC3/DTS divergence is the status block `0x0C00-0x0C07`. The next discriminating field (`r10+0x8`, and even the corrected predicate) is **not observable** through the validated mechanism.

**Final decision: C — CURRENT SAFE OBSERVATION ENVELOPE EXHAUSTED.** (See §8 for the one-line justification and what a *B* would require.)

---

## 2. Audit of R47–R52 — per-phase verdicts

Classification key: **VERIFIED BY BINARY** · **VERIFIED BY RUNTIME** · **STRONG INFERENCE** · **HYPOTHESIS** · **INCORRECT-OVERCLAIMED** · **UNRESOLVED**

### R47 — runtime capture (status block divergence)
| # | Original claim | Evidence offered | My classification | Correction / note |
|---|---|---|---|---|
| R47-a | DTS decoder is running | DEC region live for both | **VERIFIED BY RUNTIME** | Independently reproduced (DEC `0x0FE8-0x0FEC` live in every sample, both codecs). |
| R47-b | HAL/EDID are excluded as the cause | logcat shows identical Android→HAL pipeline (`format 5` vs `format 9`) | **STRONG INFERENCE** | Acceptable: the *pipeline* is identical; this does not by itself exclude a later codec-dependent branch. Not upgraded to "excluded". |
| R47-c | `0x0C00-0x0C07` is non-zero for AC3, all-zero for DTS | dense 20-sample dumps | **VERIFIED BY RUNTIME** (conclusion) / **method flawed** | Confirmed independently (§5). BUT the driver's `dmesg \| grep \| tail -N` re-prints stale cumulative lines, inflating the apparent sample count. `sort -u` on timestamps gives 20 distinct reads/cell — same conclusion, honest framing. |
| R47-d | "DTS decoded output is correct" | decoder liveness | **INCORRECT-OVERCLAIMED** | Liveness ≠ correct output. Only "the DEC block is live" is supported. |
| R47-e | 0 firmware writers of `0x0C00-0x0C07` ⇒ status is a symptom, not the cause | Python abstract interpreter | **VERIFIED BY BINARY** *(after re-test)* / **source was non-authoritative** | Re-run in Ghidra (this audit): **0 stores, 10 loads**. Claim holds; original sourcing did not meet the project's own ground-truth rule. |

### R48 — static semantics of `0xC49D` vs `0xCB50`
| # | Original claim | Evidence offered | My classification | Correction / note |
|---|---|---|---|---|
| R48-A | bits 17\|19 of `0x854` are **the AC3-vs-DTS selector** | Predicate A reads `0x854 & 0xA0000`; `0xCB50` sets those bits | **INCORRECT-OVERCLAIMED** | R46d §D3 recorded these bits' semantics as **UNKNOWN**. "AC3 helper sets them / DTS helper clears them" is a robust *correlation*; calling them the *selector* asserts a causal role never demonstrated. Downgrade to **STRONG INFERENCE (correlation)**. |
| R48-§3 | per-caller TRUE/FALSE branch table | `callers_predA.txt` | **artifact defective** | That artifact was decoded with a **4-byte step** and contains `<UNDEFINED>` garbage; only the first instruction per caller is trustworthy. R48's report admits the bug was later fixed, but the artifact in the evidence chain predates the fix. |
| R48-§3 | caller `0x1F75C` labelled "⇒ AC3" on both branches | — | **INCORRECT (internal inconsistency)** | The two branches cannot both yield AC3. Derived from the defective artifact. |

### R49 — re-interpretation as a state latch
| # | Original claim | Evidence offered | My classification | Correction / note |
|---|---|---|---|---|
| R49-B | `0xC49D` is a **state-normalization latch**, not a selector | `0x1F754 jal 0xCC9B` precedes `0x1F75C jal 0xC49D`; CC9B clears 17\|19 ⇒ P2 always FALSE ⇒ `0xCB50` force-runs | **STRONG INFERENCE** | The **instruction ordering is VERIFIED BY BINARY** (byte-exact: `0x1F754 jal CC9B`, `0x1F75C jal C49D`, `0x1F760 bnei r3,0x0,0x1F6F2`, `0x1F773 jal CB50`). The *conditional* — that `0xCC9B` actually reaches its clear store on this path — depends on its internal guard (`0xCCAF bgtsi`), which R49 did not fully discharge. Better-supported than R48, but not proven. |
| R49-B | "the AC3/DTS decision is therefore upstream" | inference | **HYPOTHESIS** | Plausible and consistent with R47's status divergence, but it is an inference from an inference. |

### R50 — codec-9 dispatch and the `0x1F6DA` gate
| # | Original claim | Evidence offered | My classification | Correction / note |
|---|---|---|---|---|
| R50-C | `mem[0x1c]` @`0x1F605` is a **snapshot of `mem[0x8(r10)]`** | register reading of `0x1F5D4 lwz r15,0x8` … `0x1F5FB mov r3,r15` … `0x1F605 sw 0x1c(r10),r3` | **INCORRECT** | **Decisive correction.** The sequence is `0x1F5FB mov r3,r15` → `0x1F5FD jal 0xB544A` → `0x1F601 movhi r14,0x4E` → `0x1F605 sw 0x1c(r10),r3`. The call **clobbers `r3`**; `0xB544A` returns `r3 = r10(callee) = arg3` (`0xB56CA mov r3,r10` → `0xB56F3 jr r9`; args at the call site are `r4=mem[0xc]`, `r5=mem[0x10]`, so the return is the **arg3 pointer derived from `mem[0x10]`**, or 0 via the `0xB54A4 movi r10,0x0` path). Hence `mem[0x1c]` = that **return value**, NOT `mem[0x8]`. |
| R50-C | the gate `0x1F6DA bnei r14,0x0` tests `mem[0x8]` | chain from the above | **INCORRECT** | Even setting link one aside: `r14` is (re)assigned at `0x1F683 mov r14,r3`, where `r3` is the **return value of `jal 0xB6D8F` @`0x1F67B`** (which received `mem[0x1c]` as arg1). So the **actual gate predicate is the return value of `0xB6D8F`** — an entirely different provenance from `mem[0x8]`. |
| R50-C | DTS slots `0x30`/`0x34` cleared unconditionally, populated conditionally | `0x1F618`/`0x1F61B` (`sw ...,r0`), `0x1F910`/`0x1F891` | **VERIFIED BY BINARY** | Independently confirmed in the ground-truth dump (`0x1F618 sw 0x30(r10),r0`; `0x1F61B sw 0x34(r10),r0`). This part stands. |
| R50-C | "Codec 9 reaches the DTS-slot code unconditionally" | dispatch at `0x1F70C-0x1F718` | **HYPOTHESIS** | The dispatch comparison is verified; "unconditionally" is not — it ignores the `0x1F6DA` gate and the `0x1F6E8`/`0x1F6F2` interlocks. |

### R51 / R52 — "dispatcher cannot be found" → "runtime-mediated"
| # | Original claim | Evidence offered | My classification | Correction / note |
|---|---|---|---|---|
| R51-C | `0x1F575` has **0 static callers** | `getReferencesTo` empty; `getFlows` scan of 590,024 insns = 0; one literal hit @`0x12AF20` | **VERIFIED BY BINARY** (negative) | Independently reproduced: 590,024 scanned, **0 flow hits**, of which 0 call/jmp (R53c). |
| R51-C | therefore `struct+0x8` is "framework/runtime state" | inference | **INCORRECT-OVERCLAIMED** | The negative is about *static decodability*, not about the runtime. Correct statement: *no statically decodable caller was found.* Also moot: `struct+0x8` is not the gate predicate (§R50). |
| R52 | no code constructs the mstsound table base; **the dispatcher is runtime-mediated or in an UNDEFINED region** | 0 hits for base `[0x12AF00,0x12B00]`; 0 hits for stored pointers to `0x12AF0C`/`0x12AF50` | **HYPOTHESIS** for the scan; **OVERCLAIMED** for the conclusion | The scans are **VERIFIED BY BINARY (negative)**. "Runtime-mediated" adds an architectural claim the scans cannot support. Defensible: *the dispatcher/context construction could not be recovered from the statically decoded image* — consistent with the ~20.3% UNDEFINED-instruction caveat in the project SKILL.md. |

**Scoreboard:** 5 claims INCORRECT/OVERCLAIMED (R47-d, R48-A, R48-§3, R50 gate ×2, R51/R52 architectural leap); 4 STRONG INFERENCE; 4 VERIFIED BY RUNTIME; 3 VERIFIED BY BINARY. No claim required **withdrawal**, but two required **material correction** (R50's gate chain; R48's selector role).

---

## 3. Independent DTS path — compact control/data-flow

Reconstructed from byte-exact ground truth (`0x1F575` body), *not* from the later reports' narrative.

```
  caller-side state of the format/codec handler  (r10 = struct pointer, r3 = arg)
  ---------------------------------------------------------------------------
  0x1F591  ori    r23,r23,0x814        ; base = 0xB000_0814
  0x1F58A  ori    r24,r23,0xC05        ; -> 0xB000_0C05   [codec/format byte]
  0x1F59B  lbz    r23,0x0(r24)         ; r23 = format byte  (r26 = it)
  0x1F5A1  mov    r10,r3               ; struct base for this call
  0x1F5D1  lwz    r14,0xC(r10)         ; field +0xC
  0x1F5D4  lwz    r15,0x8(r10)         ; field +0x8   (NOT the gate predicate — see below)
  0x1F5D7  lwz    r13,0x10(r10)        ; field +0x10  (passed as arg3)
        ...
  0x1F5ED  jal    0x10786E             ; (helper)
  0x1F5F1  mov    r4,r14               ; arg2 = +0xC
  0x1F5F3  mov    r5,r13               ; arg3 = +0x10
  0x1F5F5  addi   r6,r1,0x4            ; arg4 = stack buffer
  0x1F5F8  ori    r7,r0,0x20           ; arg5 = 0x20
  0x1F5FB  mov    r3,r15               ; arg1 = +0x8
  0x1F5FD  jal    0x0B544A             ; *** returns r3 = arg3 pointer (or 0) ***
  0x1F601  movhi  r14,0x4E             ; r14 clobbered to 0x4E0000
  0x1F605  sw     0x1C(r10),r3         ; mem[0x1C] = return value of 0xB544A   <-- NOT mem[0x8]
  0x1F608  addi   r3,r14,0x6AD4
  0x1F60C  jal    0x0CA42              ; (sub-init; 0xCA42 itself writes its own r10+0x8)
  0x1F618  sw     0x30(r10),r0         ; DTS slot 0x30 cleared   (unconditional)
  0x1F61B  sw     0x34(r10),r0         ; DTS slot 0x34 cleared   (unconditional)
  0x1F622  mov    r3,r10
  0x1F624  jal    0x1F0A0
  0x1F66B  addi   r13,r10,0x3050      ; large struct: +0x3050 region
  0x1F66F  lwz    r3,0x1C(r10)        ; r3 = mem[0x1C]  (TODAY: the 0xB544A return)
  0x1F672  addi   r4,r10,0x3038
  0x1F676  mov    r5,r13
  0x1F678  addi   r6,r10,0x14
  0x1F67B  jal    0x0B6D8F            ; *** r3 := return of THIS call ***
  0x1F67F  andi   r11,r11,0x200       ; test bit 9
  0x1F683  mov    r14,r3              ; r14 = return of 0xB6D8F   <-- GATE PREDICATE
  0x1F685  beqi   r11,0x0,0x1F6DA     ; if bit9 clear -> straight to the gate
       ...                            ; (clock/rate bookkeeping 0x1F688-0x1F6D6 uses spr 0x2808)
  0x1F6DA  bnei   r14,0x0,0x1F77B     ; *** GATE: r14 != 0 -> SKIP the DTS block ***
  ---------------------------------------------------------------------------
  DTS block: 0x1F6DE and onward (opcode_2E / beqi dispatch to 0x1F725 / 0x1F78B)
             with slot clears at 0x1F6EF (0x30) and 0x1F6FA (0x34)
  ---------------------------------------------------------------------------
  Downstream output-config:
    0xCB50 -> 0xCBFB  : 0xB000_0854  |= bits 2,6,17,19 | rate   [AC3 helper]
    0xCC9B -> 0xCD15  : 0xB000_0854  &= 0xF5FFFF (clear 17,19)  [DTS helper]
    0xC49D            : r3 = ((0x854 & 0xA0000) == 0xA0000)      [Predicate A]
    0x1F754 jal 0xCC9B -> 0x1F75C jal 0xC49D -> 0x1F760 bnei (skip)
                       -> 0x1F773 jal 0xCB50   [AC3 re-arm, fall-through only]
```

**Correction summary vs the later reports:** the chain `0x1F575` → `r10+0x30/0x34` → `0x1F6DA` is real, but the *value* flowing into the gate is `mem[0x1C]` (the `0xB544A` return), which is **then transformed by `0xB6D8F`** whose return is the actual predicate. Neither is `mem[0x8]`.

---

## 4. First proven AC3/DTS divergence

**Runtime, address-verified, this audit:**

| Observation | AC3 | DTS | Status |
|---|---|---|---|
| Kodi playback gate (`state=3`) | confirmed (poll 5) | confirmed (poll 5) | equal |
| DEC liveness `0x0FE8-0x0FEC` | live, varying | live, varying | **equal** |
| Status block `0x0C00-0x0C07` (address-verified reads) | **8 of 22 non-zero** (bursty), 14 zero | **0 of 23 non-zero** (all zero) | **DIVERGENT** |
| Output-config `0x0854/0x0858` | `0x000000` | `0x000000` | unobservable (equal-by-absence) |

Representative AC3 non-zero status values (address-verified): `0x0C00`=0x6B5A00 / 0x3C7800 / 0x7CF900; `0x0C01`=0x9F3E00 / 0xB6DB00; `0x0C02`=0x1E3E00 / 0x7CF900 / 0x50B500; `0x0C03`=0x6A6E00; `0x0C05`/`0x0C06`=0x3E7C00 / 0x8F1E00; `0x0C07`=0xCD6B00.
DTS status: **23 of 23 verified readings `0x000000`** (plus 6 earlier rounds × 8 cells from a second driver), during confirmed playback, across two independent drivers.

**Honest qualification:** AC3's status block is **bursty** — it is not non-zero in every sample (14 of 22 verified
readings are zero), because the cells cycle as the stream progresses. The meaningful asymmetry is not
"AC3 always non-zero" but **"AC3 writes the block at some point during playback; DTS never does"**. The
divergence direction is nonetheless unambiguous: **no DTS reading — 23 here plus the earlier dense runs — ever
produced a non-zero value.** (Blank/unverified readings, where the mirror failed to emit the intended cell, are
excluded from these counts, not counted as zeros.)

**This is the first proven point of divergence.** It lies **downstream of decoder liveness** (both live) and is expressed in a peripheral-status region whose runtime content tracks codec. It therefore localises the failure to the **DTS-specific output/arm path**, not to decode.

**Static counterpart (VERIFIED BY BINARY, correlation only):** `0xCC9B` clears bits 17|19 of `0xB000_0854` on the DTS path and `0xCB50` sets them on the AC3 path. Because `0x854` is **not observable** at runtime (always reads 0), this remains **STRONG INFERENCE**, never "proven selector".

---

## 5. Runtime results (AC3 and DTS, separately)

**Mechanism characterisation (new in this audit) — important for interpreting ALL prior captures:**
The `read_dsp_sram_type=1` mirror's addressing is **deterministic and offset**, and the *format of the
address you supply changes which convention is used*:
- With `addr=0x<hex>`: the emitted cell is **exactly `requested+1`** — measured **8/8**. Anchored against the live DEC region (req `0x0FE8` ⇒ `DM[0x0FE9]`, which is the live DEC counter), so the emitted cell is a real cell, just offset by one.
- With `addr=<hex>` (no prefix): the emitted cell is **almost never the request** — measured **0/8 exact, 0/8 +1, 8/8 other**; during **AC3** it frequently collapses to `DM[0x0000]`/`DM[0x0001]` regardless of the requested address.
- With `len=2` etc.: emits the pair starting at `0x0000`/`0x0001`/`0x0002`, unrelated to the request.
- Consequence: any single reading may be off-target, and **opposite conventions were used in different phases**.
  **Only address-verified, repeated readings carry weight** — hence §5's readings were accepted only when the
  emitted address matched the intended cell, otherwise retried and finally reported as `UNSTABLE`.

**AC3 (address-verified, 3 rounds):** playback confirmed; status `0x0C00-0x0C07` **8/22 verified readings non-zero** (bursty) and changing; DEC `0x0FE8-0x0FEC` live; output-config `0x0854/0x0858` zero.

**DTS (address-verified, 3 rounds + 6 earlier rounds):** playback confirmed; status `0x0C00-0x0C07` **0/23 verified readings non-zero** — every verified reading zero; DEC live; output-config zero.

**Device integrity:** no reboot. `uptime` advanced monotonically through the audit (9601 → 10714 s); `ro.boot.bootreason` remained `watchdog` (unchanged). No forbidden operation was issued.

---

## 6. Remaining uncertainty — by evidential class

**PROVEN**
- AC3/DTS status-block divergence `0x0C00-0x0C07` (runtime, reproducible, address-verified).
- Decoder liveness in both codecs (runtime).
- `0x1F5FD jal 0xB544A` precedes `0x1F605 sw 0x1C(r10),r3`, and `0xB544A` returns via `0xB56CA mov r3,r10` / `0xB56F3 jr r9` (binary). ⇒ `mem[0x1C]` is the call's return, **not** `mem[0x8]`.
- The `0x1F6DA` predicate `r14` is assigned at `0x1F683 mov r14,r3` from `jal 0xB6D8F` (binary).
- 0 statically decodable callers of `0x1F575` (binary; 590,024 instructions scanned).
- 0 stores to `0xB000_0C00..0x0C07` among all identifiable base constructions (binary; 10 loads instead).

**STRONGLY SUPPORTED**
- The AC3/DTS split is decided downstream of decode, in the output/arm path.
- `0x854` bits 17|19 differ by codec (set by the AC3 helper, cleared by the DTS helper) — **correlation**, role unproven.

**UNRESOLVED**
- **The actual gate predicate value at runtime.** The first runtime objective (`r10+0x8`, and now the corrected predicate = return of `0xB6D8F`) is **RUNTIME-UNOBSERVABLE** through the validated mechanism: the mirror requires an explicit DM `addr=`, whereas `r10` is a runtime-supplied struct pointer whose address is not derivable from the static image. No address can be named, so no read can be made. This is **not** evidence that the value is zero or non-zero.
- Whether AC3 and DTS share their upstream state before the divergence (**P-a**), or diverge only in the DTS-specific output path (**P-b**). The observed status divergence is consistent with **both**; absence of observability is recorded as UNRESOLVED, per the decision rule.
- The semantic meaning of `0x854` bits 17|19 (never classified in R46d).
- Whether the `0xB6D8F` return is codec-dependent.

---

## 7. Methodology review — the previous agent's mistakes

1. **Storing a return value as if it were an input.** R50 read `mov r3,r15` … `jal` … `sw 0x1C(r10),r3` and concluded `mem[0x1C] = mem[0x8]`, **ignoring that the intervening `jal` overwrites `r3`**. This is the single most consequential error: it mis-named the gate predicate, and R51/R52 then spent three phases resolving "`struct+0x8`" — the wrong field.
2. **Escalating correlation to role.** R48 turned "bits 17|19 differ by codec" into "bits 17|19 are the selector", despite the project's own R46d recording the bits as semantically unknown.
3. **Converting a negative into architecture.** R51/R52 rewrote "no static caller found" as "runtime-mediated dispatcher." The correct phrasing is bounded by the ~20.3% UNDEFINED-instruction caveat already documented in the project SKILL.
4. **Using non-authoritative tooling for a load-bearing fact.** R47e's "0 writers" rested on the Python interpreter, contrary to the project rule that Ghidra is ground truth. (The claim survived re-testing, but the sourcing was wrong.)
5. **Undisciplined sampling.** R47c's `dmesg | grep | tail -N` re-printed stale lines, inflating counts. The conclusion held, but the numbers did not mean what they appeared to.
6. **Shipping a defective artifact.** R48's `callers_predA.txt` was decoded on a 4-byte step (garbage `<UNDEFINED>` lines). The report's own note says the bug was fixed later, but the artifact entered the evidence chain unfixed.
7. **Internal inconsistency left uncaught.** R48 §3 labels both branches of the `0x1F75C` caller as "⇒ AC3".
8. **Not characterising the measurement instrument.** Six phases of runtime work were built on `read_dsp_sram_type=1` without ever establishing its address semantics; this audit found it is codec-dependently *mis-addressing*. That does not overturn the sustained divergence, but it should have been established before drawing conclusions from single reads.

**Positive:** R47's core runtime observation was correct and reproducible; R51/R52's scans were correct *as scans*; the project's safety discipline (no patching, no MMIO) was maintained throughout.

---

## 8. Final decision

### **C — CURRENT SAFE OBSERVATION ENVELOPE EXHAUSTED**

**Justification.** The failure is localised as far as the permitted envelope allows: the AC3/DTS split is proven to lie **downstream of a shared, decoder-live upstream** and manifests in the DSP peripheral-status region `0x0C00-0x0C07`. Advancing to the *gate predicate itself* requires reading a field of a runtime-supplied struct pointer (`r10`), for which the validated mechanism — an **explicit-address** DM reader — cannot form an address. The only route to that value would be either (a) a **new memory/pointer-resolving primitive**, or (b) **instrumentation or patching** of the firmware — both **explicitly forbidden** by the standing constraints, and the previous attempt at the former class (`/dev/malloc`) caused a device reboot.

**What decision B would require (recorded for completeness, not attempted):**
- An already-observable cell whose value is provably a function of the `0xB6D8F` return value, **or**
- A validated, read-only way to resolve the runtime value of `r10`, **or**
- A repeatable correlation between a **known DM cell** and the DTS block's zero/non-zero state. None is currently available.

**Analytical note (no patch deployed, none proposed):** the single most informative *future* discriminator, if instrumentation were ever authorised, would be the return value of `0xB6D8F` at the `0x1F6DA` gate — **not** `mem[0x8]`, which the later reports wrongly identified as the gate source.

---

*Auditor's summary line: the previous work reached the right neighbourhood by a partially wrong route. The runtime divergence is real and now independently reproduced; the gate predicate it named is wrong; the dispatcher claim exceeds its evidence. Within the authorised, non-invasive envelope, the root cause is localised to "the DTS-specific output/arm path, downstream of decode" — and no further safe discriminator remains.*
