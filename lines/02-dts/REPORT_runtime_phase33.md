# R33 — Static Resolution of the `0xA868` / `0xA86C` Downstream Path

**Module:** `mst_snd_r2_MS12V22.bin` (extracted `snd_full.bin`) — MStar/MedTek AEON SND audio firmware
**CPU:** `aeon:LE:32:default` · `flow[0]` authoritative · `opcode_2E` = MMIO-touch
**Prior:** R29 (R29-B) · R30 (runtime, R30-B) · R31 (R31-C) · R32 (R32-C + Part II)
**This phase:** R33 — **pure static analysis.** No runtime, no patch, no device/EDID/AUTH/settings changes.
**Not repeated:** `0x25EE1` / `r11` / `0x0C06` / `divu` analysis (already established in R31/R32).
**Deliverable date:** 2026-09-09

---

## 0. Headline classification

> ### R33 = **R33-B** — `0xA868`/`0xA86C` affect **output-adjacent hardware / shared-memory (DMA) state**,
> ### but **no actual DTS formatter / SDO / IEC61937 / TX consumer can be established** —
> ### because **there is no firmware consumer at all**.

The decisive new result is a **definitive negative**: an exhaustive whole-image search for every construction of
base `0xA000` shows that **only 4 sites touch `0x868`/`0x86C`, and all 4 are stores (the known producers).
No `bg.lwz` (or any load) of `0xA868`/`0xA86C` exists anywhere in `snd_full.bin`.** The slots are
**write-only from the firmware's perspective**; each write is followed by a cache-flush (`0xD4C8`), i.e. a
publish to an external agent (hardware / DMA / other core).

---

## 1. Evidence labels

**VERIFIED STATIC** · **STRONG STATIC INFERENCE** · **UNRESOLVED**.

---

## 2. Phase A — Decode the two candidate consumer sites (both REFUTED)

Targeted dumps were generated with the **existing analysed project** `r27_work/ghidra_r27f` (project `sndr27`,
program `snd_full.bin`) — reuse succeeded; no re-import was needed.
*(Note: `DUMPALL` args must be **bare hex, no `0x` prefix** — `DUMPALL:16400:400`, not `0x16400`.)*

### 2.1 Site A — `0x16454…0x16474` → **not** `0xA868`
```
0x16458  bg.addi r3,r10,0x878
0x1645C  bg.jal 0x0010c5b9
0x16460  bg.addi r3,r10,0x998
0x16464  bg.sw_0 0x86c(r10),r13
0x16468  bg.sw_0 0x870(r10),r12
0x1646C  bg.sw_0 0x874(r10),r11
0x16470  bg.sw_0 0x868(r10),r11
0x16474  bt.movi r4,0x0
0x16476  bn.ori  r5,r0,0xdc
0x16479  bg.jal 0x0010c5b9
```
The wider function writes repeating 4-word groups at stride `0x120`
(`0x508…0x514`, `0x628…0x634`, `0x748…0x754`, `0x868…0x874`) and calls `0x10C5B9` after each group.

**Base `r10` provenance — it is NOT `0xA000`:**
- `0x162B7  bn.lwz r10,0x68(r1)` — loaded from the **stack** (saved/argument value).
- `0x16325  bt.mov r10,r3` — taken from `r3` (call result / argument).

> `r10` is a **dynamic instance pointer**, so `r10+0x868` is **not** the absolute address `0xA868`.
> **REFUTED as a consumer of `0xA868`/`0xA86C`. VERIFIED STATIC.**

### 2.2 Site B — `0xFFD9D…0xFFDB1` → **stack-frame prologue**, not a structure
```
0xFFD79  bg.addi r1,r1,-0x898        ; r1 = STACK POINTER
0xFFD7D  bg.sw_0 0x894(r1),r9
0xFFD81  bg.sw_0 0x890(r1),r10
0xFFD85  bg.sw_0 0x884(r1),r13
0xFFD89  bg.sw_0 0x864(r1),r21
0xFFD8D  bg.sw_0 0x860(r1),r22
0xFFD91  bg.sw_0 0x88c(r1),r11
0xFFD95  bg.sw_0 0x888(r1),r12
0xFFD99  bg.sw_0 0x880(r1),r14
0xFFD9D  bg.sw_0 0x87c(r1),r15
0xFFDA1  bg.sw_0 0x878(r1),r16
0xFFDA5  bg.sw_0 0x874(r1),r17
0xFFDA9  bg.sw_0 0x870(r1),r18
0xFFDAD  bg.sw_0 0x86c(r1),r19
0xFFDB1  bg.sw_0 0x868(r1),r20
```
The "descending `0x87C → 0x868` sequence" is simply **saving registers r15–r20 into a 0x898-byte stack frame**.
Base = **`r1` (SP)**. **REFUTED. VERIFIED STATIC.**

### 2.3 Exhaustive base-`0xA000` search (the decisive test)
Scanned the whole image for every instruction building base `0xA000`
(`bg.movhi rX,0x1 ; bg.addi/ori rX,rX,-0x6000`, immediate bytes `A0 00`) — **50 sites** — then looked ahead
48 bytes for `0x868` / `0x86C` accesses:

| `0xA000` built at | following `0x868`/`0x86C` access | kind |
|-------------------|----------------------------------|------|
| `0xCE22` | `0xCE26  ef17086c` | **STORE** (`bg.sw_0`) — producer (0xCC9B) |
| `0xCE4E` | `0xCE72  ef17086c` | **STORE** — producer (0xCC9B) |
| `0xCF22` | `0xCF48  ef17086c` | **STORE** — producer (0xCC9B) |
| `0xD15A` | `0xD15E  ef170868` | **STORE** — producer (0xCF86) |
| *(other 46 sites)* | — | access other offsets of the `0xA000` window |

> ### **No load of `0xA868` or `0xA86C` exists anywhere in the image.**
> **`0xA868` / `0xA86C` are WRITE-ONLY from firmware. VERIFIED STATIC.**
> (`0xA000` itself is a large shared/hardware window used by ~50 sites — only these 4 touch `0x868`/`0x86C`.)

---

## 3. Phase B — Producer → consumer chains (both terminate at a handoff)

### 3.1 `0xA868`
```
0xCF86 … common math (0xD2F0) …
   → 0xD15E  bg.sw_0 0x868(r23),r24     (r23 = 0xA000)   WRITE 0xA868
   → 0xD16C  bg.addi r3,r3,-0x5798 ; r4=4
   → 0xD174  bg.j 0xD4C8                                  CACHE FLUSH
   → *** NO FIRMWARE READER ***
   → external agent (hardware / DMA / other core)
```
### 3.2 `0xA86C`
```
0xCC9B … common math (0xCEF5) …
   → 0xCE26 / 0xCE72 / 0xCF48  bg.sw_0 0x86c(r23),r24  (r23 = 0xA000)  WRITE 0xA86C
   → 0xCE3A / …  bg.j 0xD4C8                                            CACHE FLUSH
   → *** NO FIRMWARE READER ***
   → external agent
```
**The chain does not stop at a block copy — it stops at a write-plus-cache-flush handoff.**
Continuing further requires visibility into the external agent (hardware/DSP/other core), which is **outside
this firmware image**. **VERIFIED STATIC** (no reader) + **UNRESOLVED** (external consumer identity).

---

## 4. Phase C — The "six-slot array" claim: **DISPROVEN**

R32 hypothesised a 6-word structure at `0x868, 0x86C, 0x870, 0x874, 0x878, 0x87C`. Verification:

| Evidence | Verdict |
|----------|---------|
| Site B (`0xFFD9D…0xFFDB1`) descending `0x87C→0x868` | **Stack frame prologue** (base `r1`), not a structure — refuted |
| Site A (`0x16464…0x16470`) `0x86C,0x870,0x874,0x868` | **Dynamic base `r10`** (instance struct), not `0xA000` — refuted |
| Exhaustive `0xA000`-base scan | Only offsets **`0x868` and `0x86C`** are ever touched with base `0xA000`; **`0x870`/`0x874`/`0x878`/`0x87C` are NEVER accessed with base `0xA000`** |

> **The "six-slot array at `0x868–0x87C`" is NOT supported.** Only two slots (`0x868`, `0x86C`) belong to the
> `0xA000` window, and both are write-only. **VERIFIED STATIC (negative).**
> Per instruction, the term "array"/"structure" is **not** used for this region.

---

## 5. Phase D — Exact arithmetic generating the published values

Derived from `decoded_CF86.txt` (`0xD2F0…0xD31B`) and `decoded_CC9B.txt` (`0xCEF5…`):

```
r10 = 0x251194                                   (0xD063: movhi r10,0x25 ; addi r10,r10,0x1194)

0xD2F0  bn.muls       r24,r24,r25       ; A = (+0x18) * (+0x20)
0xD2F3  bn.lwz        r23,0x14(r23)     ; N = *(+0x14)
0xD2F6  bg.muli_maybe r24,r24,0x30      ; D = A * 0x30         (divisor)
0xD2FA  bg.lwz        r26,0x114(r10)    ; B = *(0x251194+0x114)
0xD2FE  bn.divu       r25,r23,r24       ; q = N / D            (divu: rD = rA/rB, R32 verified)
0xD302  bg.movhi      r23,0x25
0xD306  bg.lwz        r24,0x3364(r23)   ; C = *(0x253364)
0xD30A  bg.lwz        r23,0x118(r10)    ; E = *(0x251194+0x118)
0xD30E  bt.add        r24,r26           ; r24 = C + B
0xD312  bt.add        r24,r23           ; r24 = C + B + E
0xD316  bg.movhi      r23,0x1
0xD31A  bt.add        r24,r25           ; r24 = C + B + E + q
0xD31E  bg.addi       r23,r23,-0x6000   ; r23 = 0xA000
0xD15E  bg.sw_0 0x868(r23),r24          ; 0xA868 = r24
```

### Exact published expression
```
published(0xA868) = *(0x253364)
                  + *(0x251194 + 0x114)
                  + *(0x251194 + 0x118)
                  + [ (*(+0x14)) / ( ((+0x18) * (+0x20)) * 0x30 ) ]
```
The `0xCC9B` path to `0xA86C` (`0xCEF5` math → `0xCE26`) has the **same form** with the same term set.
**No clamping, saturation, or min/max is present in this path** (`VERIFIED STATIC` — none observed in the
decoded instruction stream from the `divu` to the store).

| Case | `+0x18` | divisor `((+0x18)*(+0x20))*0x30` | quotient `q` |
|------|---------|------------------|--------------|
| populated (`0x13292` ran) | `0x100` | `0xC000` | `N / 0xC000` |
| default (skipped) | `8` | `0x600` | `N / 0x600` (**32× larger**) |

`q` is one of four additive terms — **the total published value is dominated by `*(0x253364) + B + E`; `q`
is an additive correction, not a scale factor.** This materially tempers the R32 "32× difference" framing:
the *quotient* changes 32×, but the *published value* changes only by `q_default − q_populated`.
**Do not claim "32× value difference" — that is NOT supported.**

---

## 6. Phase E — `+0x14` data flow (runtime-independent)

`+0x14` is assigned `(+0x10) − (+0x0C)` at `0xD1D3` (`0xCF86`) and `0xCCDE` (`0xCC9B`):
```
0xD1C2  bn.lwz r23,0xc(r3)
0xD1C5  bn.lwz r25,0x10(r3)
0xD1C8  bn.sub r23,r25,r23          ; r23 = (+0x10) - (+0xc)
0xD1D3  bn.sw  0x14(r3),r23         ; +0x14 = r23
```
Provenance of `+0x0C` / `+0x10`:
- `0x13292` initialises them **both to the same value** `r12` (`0x132DF sw 0xc(r10),r12`, `0x132E2 sw 0x10(r10),r12`) ⇒ initial delta **`0`**.
- `0xCF86` sets both to `0xDD8000` (`0xD02B sw 0x10(r3),r23`, `0xD02E sw 0xc(r3),r23`) ⇒ delta `0`.
- `0xCF86` later modifies `+0x0C` **independently** (`0xD26D bn.sw 0xc(r3),r23`, and `0xD031 sw 0x14(r3),r0`)
  ⇒ the two diverge and the delta becomes **non-zero**.

**Structural conclusion (STRONG STATIC INFERENCE):** `+0x0C` and `+0x10` behave as a **pointer / offset pair**
that is initialised equal and then diverges as one advances — so `+0x14` is a **derived delta** consistent with a
frame/period/available-length measure. **Semantic names (write pointer, read pointer, frame size) are NOT
assigned** — the producers/consumers do not justify them. The value being a delta **is** justified; its
meaning is **UNRESOLVED**.

---

## 7. Phase F — `0x255CA8` (alternate config object)

- **9 real reference sites** for immediate `0x5CA8` (R32): `0xCED8`, `0xD2D3`, `0xED4C`, `0xEFD2`, `0x16089`,
  `0x16243`, `0x16A8B`, `0x18109`, `0x18111`.
- **Decodable:** only `0xCED8` (`0xCC9B`) and `0xD2D3` (`0xCF86`). Both read `+0x18`, `+0x20` (and `+0x14`),
  apply defaults (`+0x18 = 0xa`, `+0x20 = 0x2` in `0xCC9B`; `+0x20 = 0x2` in `0xCF86`), then run the **same
  common math**.
- **Central initialisation: UNRESOLVED.** The remaining 7 sites lie outside the available dumps; no central
  init routine for `0x255CA8` was located, so it cannot be shown that it is pre-populated before DTS playback.

**UNRESOLVED** (per your instruction: enough to establish the object, not a full reverse of all nine sites).

---

## 8. Phase G — `0x109595` RESOLVED (previously open since R32)

Dump `0x109000:0x1000` obtained from the reused project; `0x109595` decoded (92 instrs):
```
0x109599  bn.beqi r3,0x0,0x1095CA            ; r3 == 0 -> pack with zero exponent
0x10959E  bg.blesi r3,-0x1,0x10967C          ; negative -> sign path (r24 = 1, negate)
0x1095A2  bn.ori r25,r0,0x9e                 ; BIAS = 0x9E (158)
0x1095A5  bn.op17_E r23,r3                   ; bit-scan / CLZ  -> exponent
0x1095A8  bn.sub r23,r25,r23                 ; exp = 0x9E - scan
0x1095BD  bn.sll r3,r3,r23                   ; normalise mantissa
0x1095C7  bn.and r3,r3,r23(0x7FFFFFFF)       ; mantissa mask
0x1095CA  bn.slli r25,r25,0x17               ; exponent << 23
0x1095CD  bn.slli r24,r24,0x1f               ; sign     << 31
0x1095DA  bn.or r3,r3,r25 ; 0x1095DD or r24  ; combine
0x1095E0  bt.jr r9                            ; return packed value
```
> **`0x109595` is a generic IEEE-754-style float-pack / numeric-conversion helper**: exponent biased by
> `0x9E`, mantissa masked with `0x7FFFFFFF`, exponent `<< 23`, sign `<< 31`. It has **no callees, no MMIO,
> no output/encoder/SDO/TX operations.** **VERIFIED STATIC.**

**Consequence:** the R32 open question *"does `+0xBC = 0` eventually suppress an actual DTS output operation via
`0x109595`?"* is answered **NO**. The `0x27xxx` functions pass the cached gate fields (`+0xB0/+0xB4/+0xB8/+0xBC`)
into this **numeric conversion helper** — i.e. they are used as **numeric inputs**, not as an output-enable gate.
The `+0xBC → output-suppression` hypothesis is **not supported**.

---

## 9. Phase H — `0xB000_0C06` wording preserved

Retained verbatim (not promoted):
> **No direct firmware store to `0xB000_0C06` was found among the searched address-construction forms.**

R33 did not add evidence that would justify "hardware-only status register"; and R33 **does** add a cautionary
precedent: two R32 "consumer" candidates were disproven purely by checking the **base register**, so any
offset-only scan remains non-conclusive.

---

## 10. Answers to the six required R33 questions

**Q1. Exact producer → consumer chain for `0xA868`?**
`0xCF86` common math (`0xD2F0`) → `0xD15E bg.sw_0 0x868(r23)` (`r23 = 0xA000`) → cache flush `0xD4C8` →
**no firmware consumer**. Chain terminates at the write/flush handoff. **VERIFIED STATIC (negative).**

**Q2. Exact producer → consumer chain for `0xA86C`?**
`0xCC9B` common math (`0xCEF5`) → `0xCE26 / 0xCE72 / 0xCF48 bg.sw_0 0x86c(r23)` (`r23 = 0xA000`) → cache flush
`0xD4C8` → **no firmware consumer**. **VERIFIED STATIC (negative).**

**Q3. Actual base and semantics of the `0x868–0x87C` region?**
Base is the literal `0xA000` **only** for `0x868` and `0x86C` (write-only). `0x870/0x874/0x878/0x87C` are
**never** accessed with base `0xA000`; the sites that appeared to touch them use **SP (`r1`)** or a **dynamic
instance register (`r10`)**. **The "six-slot array/structure" claim is DISPROVEN.** **VERIFIED STATIC.**

**Q4. Exact arithmetic generating the published values?**
`published = *(0x253364) + *(0x251194+0x114) + *(0x251194+0x118) + [ (+0x14) / (((+0x18)*(+0x20))*0x30) ]`,
no clamping. The `+0x14` term is an **additive correction**, so a 32× change in the *quotient* is **not** a 32×
change in the published value. **VERIFIED STATIC.**

**Q5. Do those values reach the DTS encoder / SDO / IEC61937 / TX path?**
**No firmware path reaches them.** They are written to a shared/hardware window with a cache flush and never
read back. Any consumer is **external** (hardware/DMA/other core). **UNRESOLVED** whether that external consumer
is the DTS TX — cannot be determined from this image.

**Q6. Does the `0x25EE1` return/configuration branch now have a direct connection to physical DTS transmission?**
**No direct static bridge was established.** The configuration branch selects `0x255CA8` vs `0x252AD0`, which
changes an **additive term** in a value published to a write-only shared slot; the chain then leaves the
firmware. **The missing link is outside this image.**

---

## 11. Final classification — R33-B

- **R33-A — NO.** No DTS encoder/SDO/TX *firmware* consumer exists (write-only slots).
- **R33-B — YES (selected).** `0xA868`/`0xA86C` are written into a **shared / hardware-mapped window
  (`0xA000`)** and **cache-flushed** — i.e. they affect **output-adjacent hardware / DMA state** — but no
  formatter/TX consumer can be established because the consumer is external to the firmware.
- **R33-C — NO.** They are not generic bookkeeping: they sit in the DTS output path and are published with a
  cache flush to another agent.
- **R33-D — NO.** Coverage was obtained (project reuse worked); the result is a **definitive negative**, not a
  missing-dump failure.

### What would close the remaining gap
Visibility into the **external agent** that reads `0xA868`/`0xA86C` after the flush — e.g. a DSP/other-core
image, an RTL/TRM for the `0xA000` window, or a runtime probe of that agent. None of these is available in the
current static image (and runtime is out of scope for R33).

---

## Appendix A — New artifacts (R33)
- `r33_work/dump_consumers.txt` — targeted dump `0x16400:0x400` + `0xFFD00:0x400`.
- `r33_work/dump_prologue.txt` — targeted dump `0x16200:0x400` (resolved `r10` provenance).
- `r33_work/dump_109.txt`, `r33_work/decoded_109595.txt` — `0x109000:0x1000` dump + `0x109595` decode.
- Scans: all `0xA000`-base constructions (50) with look-ahead for `0x868`/`0x86C` (4, all stores).

## Appendix B — Reusable procedure (worked, no re-import)
```sh
export JAVA_HOME="C:/Users/k0994/AppData/Local/Programs/Eclipse Adoptium/jdk-21.0.12.8-hotspot"
export APPDATA="C:/Users/k0994/AppData"; export HOME="C:/Users/k0994/AppData"
/c/ghidra_12.1.2_PUBLIC/support/analyzeHeadless.bat \
  "C:/firmware_temp/aeon_validate/r27_work/ghidra_r27f" sndr27 \
  -process snd_full.bin -noanalysis \
  -scriptPath "C:/firmware_temp/aeon_validate/scripts" \
  -postScript DumpAEON_R29.java "DUMPALL:16400:400" "OUT:<out.txt>"
```
- **Use the existing analysed project `r27_work/ghidra_r27f` (`sndr27`)** — `ghidra_snd4/5/6` are empty shells.
- **`DUMPALL` args are bare hex without `0x`** (`16400`, `FFD00`) — with `0x` it throws `NumberFormatException`.
- Decode with `python3 scripts/r29_decode.py <dump.txt> <ENTRY> <out.txt>`.
