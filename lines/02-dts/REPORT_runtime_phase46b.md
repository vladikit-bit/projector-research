# REPORT — R46b — Ghidra-only writer verification + AC3/DTS post-quotient differential

**Subject:** Thundeal TD98 Pro / MStar MT5889 (Android 11) — AEON (R2) audio DSP
**Image:** `snd_full.bin` (1,839,920 bytes, base 0)
**Method:** **Ghidra only** for all disassembly/decoding in this report. The Python decoder is used **nowhere** for any conclusion below.
**Constraints honoured:** no patch, no device access, no runtime experiment, no EDID/AUTH re-opening, no re-run of the `0x13292`/gate investigation.
**Versioning:** this is a **new** report. `REPORT_runtime_phase46.md` (§0–§20) and `R46_FINAL_SUMMARY.md` are preserved unchanged as prior evidence; where they conflict with this report, **this report supersedes them**.

> **⚠️ THIS REPORT IS ITSELF SUPERSEDED IN PART by `REPORT_runtime_phase46c.md` (§26–§40).** Specifically: §22's "the DTS values are the quotient / AC3 discards it" stands, but the *semantic* label "timing quotient" does not; §23.3's "the byte-push / IEC61937 framing is done by hardware" is **downgraded to STRONG INFERENCE** (R46c §32); and the divisor in §22.2 is not `SPR(0x2808)`-derived but derived from `rate/1000` with the multiply-high transfer (R46c §36). §21 (the Ghidra-only writer verification) is **unaffected and remains authoritative**.

---

## §21 — Ghidra-only writer verification

### §21.1 Method (no Python decoder involved)

Two independent, decoder-free mechanisms were used and then cross-checked against each other:

1. **`scripts/FindMMIO.java`** — a new Ghidra postscript that walks the **entire image using Ghidra's own disassembly** (the authoritative `aeon.slaspec`), maintaining a register-constant map across the linear stream (`movhi` / `ori` / `addi` / `muli` / `andi`; any other instruction invalidates the destination register). It reports every **store and load** whose effective address resolves into `0xB000_0000–0xB000_FFFF`.
   - Coverage: **587,030 instructions disassembled**; **106 MMIO stores** and **281 MMIO loads** reported.
   - Output: `r46_work/ghidra_writers.txt`.
2. **`r46_work/r46_bytescan.py`** — a **pure big-endian word-pattern scan** (no decoder at all), at **every byte offset** (the code is not 4-byte aligned), for:
   - (a) immediate-bearing ops (`ori 0x32`, `addi 0x3F`, `andi 0x31`, `xori 0x36`, `addci 0x37`) whose low halfword equals the target offset — i.e. every possible **constant-construction** of the address;
   - (b) direct base+offset forms (`sw/lwz/sh 0x3B`, `sbz 0x3E`).

All addressing found is **computed** (`bg.movhi rX,0xb000` + `bg.ori/addi rX,…`) — **no direct absolute addressing** of these registers exists anywhere in the image.

### §21.2 Writer / reader results (Ghidra)

| Target | Writers (Ghidra) | How the address is built | Readers (Ghidra) |
|---|---|---|---|
| `0xB000_0868` | **1** — `0x178B9 bn.sw 0x0(r23),r24` | `0x178B5 bg.ori r23,r23,0x868` (r23 from `bg.movhi r23,0xb000` @`0x178AE`) | **1** — `0xE674` |
| `0xB000_086C` | **1** — `0xCD76 bn.sw 0x0(r26),r25` | `0xCD72 bg.ori r26,r24,0x86c` (r24 from `bg.movhi r24,0xb000` @`0xCD68`) | **1** — `0xE68D` |
| `0xB000_0870` | **1** — `0xD0A1 bn.sw 0x0(r26),r24` | `0xD09A bg.ori r26,r23,0x870` (r23 from `bg.movhi r23,0xb000` @`0xD093`) | **1** — `0xE6A6` |
| `0xB000_0C06` | **0** | — | **4** — `0x25EF6`, `0x25FDE`, `0x26064`, `0x262EC` |
| `0xB000_0015` | **0** | — | **4** — `0x25EFC`, `0x2606A`, `0x1E6C2`, `0x1F0D2` |

**Cross-check (independent byte scan).** Construction candidates found by pattern matching:
- `0x868`: `0x0E670` (reader base), `0x178B5` (**writer base**), `0x16391` (excluded).
- `0x86C`: `0x0CD72` (**writer base**), `0x0E689` (reader base), `0x78642` (excluded).
- `0x870`: `0xD09A` (**writer base**), `0xE6A2` (reader base).

The two extra candidates were disassembled and **excluded**: `0x16391 bg.addi r24,r10,0x868` and `0x78642 bg.addi r28,r11,0x86c` are **data-structure offsets added to unrelated base registers** (`r10`/`r11`), part of long `addi` series (`0x3e8,0x508,0x628,0x748,0x868,0x988` and `0x840,0x1080,0x1bd8,0x344,0x86c,0x10d8,…`), each followed by ordinary `bg.sw_0` stores into a private structure. They do **not** construct a `0xB000_xxxx` address.

⇒ **Two independent methods agree: there are exactly three writers in total, one per register, and they are the three already known.**

### §21.3 The readers are a DIAGNOSTIC DUMP, not a functional consumer

`0xE60C–0xE702` is a **straight-line unrolled register dump**: it reads the MMIO block **consecutively at +4** —

```
0x858 → 0x85C → 0x860 → 0x864 → 0x868 → 0x86C → 0x870 → 0x874 → 0x878 → 0x87C
```

Each value is masked with `0x00FFFFFF` (`bn.movhi_2 r23,0xff` + `bg.ori r23,r23,0xffff` + `bn.and`), a message pointer is loaded (`bg.addi r3,r10,0x7091`), and the result is passed to **`bg.jal 0x10786E`** — the generic wrapper / logging helper identified in §17.5 and §18.4 (1919 call sites).

⇒ **No functional firmware consumer reads `0x868` / `0x86C` / `0x870`.** The only reads in the entire image are this telemetry dump. The functional consumer must therefore be **hardware**.

### §21.4 Reconciliation of previous R46 claims

| Previous claim | New status |
|---|---|
| `0x868` has exactly one writer (`0x178B9`) | **VERIFIED BY GHIDRA** |
| `0x86C` has exactly one writer (`0xCD76`) | **VERIFIED BY GHIDRA** |
| `0x870` has exactly one writer (`0xD0A1`) | **VERIFIED BY GHIDRA** |
| "`0x86C`/`0x870` are **write-only, never read back** anywhere" | **DISPROVED as stated** — each has exactly **one** reader (`0xE674` / `0xE68D` / `0xE6A6`). **Restated:** no *functional* reader; exactly one *telemetry* reader in the `0xE6xx` dump. |
| "`0xB000_0C06` has zero writers" | **VERIFIED BY GHIDRA** (0 writers, 4 readers) |
| "`0xB000_0015` has zero writers" | **VERIFIED BY GHIDRA** (0 writers, 4 readers) |
| "`0x0C06` **bit 7** is the predicate term" | **VERIFIED BY GHIDRA** — `0x25EF6 bn.lbz r12,0(r24)` then `0x25F0D bn.srli r11,r12,0x7`; all four readers are inside `0x25Exx–0x262xx` |
| "`0x0015` tested as **signed-negative** (`<= -1`, immediate)" | **VERIFIED BY GHIDRA** — `0x25F14 bn.sflesi r23,-0x1` |
| "readers of `0x0015` are confined to the gate cluster" | **DISPROVED** — `0x0015` is also read at `0x1E6C2` and `0x1F0D2`, where a **different bit** is used: `bn.andi r23,r23,0x10` (**bit 4**). So `0x0015` is a multi-bit status byte consumed in ≥3 places. |
| "exactly one writer each **in the whole image**" (R46 §1) | **VERIFIED BY GHIDRA** — closes risk **R3** |
| "readers of `0x2512C4` confined to the gate cluster" | **UNRESOLVED** — `0x2512C4` is a data address outside the MMIO window; this sweep does not cover it |

### §21.5 Answer to the section-3 question

> Are `0x868/0x86C/0x870` genuinely DTS-specific output-state registers, and do we know their complete statically provable writer set?

**Yes — with one wording refinement.** The complete statically provable writer set is:

- `0x868` ← exactly `0x178B9` (DTS-X SDO-packer cluster)
- `0x86C` ← exactly `0xCD76` (DTS output-config helper `0xCC9B`)
- `0x870` ← exactly `0xD0A1` (DTS output-config helper `0xCF86`)

and **AC3 writes none of them** (the AC3 helpers' entire MMIO set is `0x854`, `0x850`, `0x84C`, `0x814`, `0x80C`). The refinement: they are **not literally write-only** — each has exactly one *telemetry* reader (`0xE674` / `0xE68D` / `0xE6A6`), which does not consume them functionally.

---

## §22 — AC3-vs-DTS post-quotient differential

The common computation (`word@0x2511A8 + SPR(0x2808)` → `>>11` → sign correction) is **already established and is not re-investigated here**. This section traces only what happens **after** it.

### §22.1 Where AC3 stops using the quotient (VERIFIED, `0xCB50`)

```
0xCBC3  bn.add r6,r5,r24        r6 = state + SPR
0xCBC9  bn.srai r6,r6,0xb       r6 >>= 11
0xCBD2  bn.sub r6,r6,r23        r6 = THE QUOTIENT          ← computed
0xCBD5  bg.ori r26,r26,0xfff
0xCBD9  bg.movhi r3,0x12
0xCBDF  bn.ori r23,r23,0x44     r23 = 0xA0044 (flags)
0xCBE2  bn.and r11,r11,r26      r11 = current 0x854 & 0xFFF
0xCBE5  bg.addi r3,r3,0x6862    r3 = 0x126862 (message pointer)
0xCBE9  bn.or r11,r11,r23
0xCBEC  bg.jal 0x10786E         → logging call
0xCBF4  bn.or r10,r11,r10       r10 = flags | RATE CODE
0xCBFB  bn.sw 0x0(r23),r10      → WRITE 0xB000_0854
0xCC06  bt.jr r9                return
```

**The quotient (`r6`) is never stored to any MMIO register on the AC3 path.** The only MMIO write in this helper is `0xB000_0854`, and its value is a **sample-rate code** — `0x1000` (88200), `0x4000` (44100), `0x2000` (64000), `0x5000` (32000), `0x3000` (default) — OR'd with the flag constant `0xA0044` and the low 12 bits of the register's previous value.

⇒ **AC3 configures the output block by *rate code only*, and leaves `0x868`/`0x86C`/`0x870` at their hardware reset defaults.** (Whether `r6` is additionally passed to the logging call at `0xCBEC` does not change this: it has **no state destination**.)

### §22.2 Where DTS uses the quotient (VERIFIED)

| register | store | value written |
|---|---|---|
| `0x86C` | `0xCD76` (helper `0xCC9B`) | `((word@0x2511A8 + SPR(0x2808)) >> 11) − sign` & `0x00FFFFFF` — **the quotient itself** |
| `0x870` | `0xD0A1` (helper `0xCF86`) | `(0xDD8000 / divisor) + <same quotient>` & `0x00FFFFFF`, where `divisor` derives from `SPR(0x2808)` and the helper **dispatches on the sample rate** (`0xB000_0814` vs 88200/128000/176400/192000) |
| `0x868` | `0x178B9` (SDO-packer cluster) | `((word@(r10+0xb70) × 0x30) + r25)` & `0x00FFFFFF` |

The DTS helpers additionally write the shared control registers `0x854` (masked preserve) and `0x850` (0).

### §22.3 The consumer chain (Ghidra-verified edges only)

```
DTS timing write:  0xB290 (conditional, §19.4) ──► 0xCC9B ──► 0x86C
                                              └──► 0xCF86 ──► 0x870   (+ 0x854, 0x850)
DTS SDO packer:    0x178B9 ─────────────────────────────────► 0x868
readers of 0x868/0x86C/0x870:  ONLY the 0xE6xx telemetry dump (§21.3)
⇒ functional consumer = HARDWARE (SPDIF / IEC61937 output block)
```

### §22.4 What the registers control

Statically determinable:
- They are **timing parameters** (sample-rate- and timer-derived quotients), not enable/disable bits: nothing in the firmware tests them, and no control-flow depends on them.
- They are **not** framing bits the firmware acts on — the firmware never reads them for a decision.
- Because the only firmware reads are telemetry, the values must be **consumed by the output hardware block**; the firmware cannot observe their effect.

Not statically determinable: whether the hardware block treats `0x86C`/`0x870` as **burst/pause period** vs. e.g. **FIFO thresholds** — the firmware gives no functional reader from which to infer it.

### §22.5 The divergence, stated precisely

| | AC3 | DTS |
|---|---|---|
| computes `(word@0x2511A8 + SPR(0x2808))>>11 − sign` | **yes** | **yes** |
| dispatches on sample rate | yes (88200/44100/64000/32000) | yes (88200/128000/176400/192000) |
| **writes the quotient to MMIO** | **no — quotient discarded** | **yes → `0x86C`, and `0x870` (+`0x868` via the SDO packer)** |
| writes `0x854` | yes — **rate code** + flags | yes — masked preserve |
| writes `0x850` / `0x84C` | yes | `0x850` yes |

⇒ **The single structurally-supported asymmetry in the output-timing path is: DTS overrides the hardware output-timing registers with computed values; AC3 does not, leaving the hardware defaults in place.** If the DTS-computed values are not correct for the actual transport, DTS fails while AC3 (defaults) works. That is exactly **H1**, and it is now the only asymmetry of this kind that the static evidence supports.

**H2 (the gate) remains a separate, unproven mechanism** (§17) — this analysis neither confirms nor refutes it, but it removes the need to invoke it: H1 alone is now structurally sufficient to explain the symptom.

### §22.6 Residual (not closable statically)

1. Numeric correctness of the three written values (needs the register datasheet or a runtime capture).
2. Which hardware behaviour `0x86C`/`0x870`/`0x868` actually govern (burst/pause period vs. FIFO/watermark) — no firmware reader exists to infer it.
3. Full semantics of `0x854`'s flag bits (`0xA0044`) and of the rate-code field's low 12 bits.
4. Semantics of the unmodelled DSP ops `op2A`/`op2B`/`op2E`/`op2F` (they appear in this path).

---

## §23 — MMIO output-block map, and where the transmit interface actually is

Derived **entirely from the existing Ghidra inventory** (`r46_work/ghidra_writers.txt`: 106 stores / 281 loads over `0xB000_0000–0xB000_FFFF`). **No new pass was run** — this is data mining of §21's output.

### §23.1 Complete register map with roles (Ghidra inventory)

| Role | Registers (stores/loads) |
|---|---|
| **Read-only status** (loads only) | `0x800`(0/1) `0x804`(0/3) `0x808`(0/3) `0x80C`(0/7) `0x810`(0/7) **`0x814`(0/24)** `0x818`(0/6) `0x81C`(0/5) `0x820`(0/1) `0x824`(0/3) `0x828`(0/3) `0x82C`(0/5) `0x830`(0/4) **`0x834`(0/12)** `0x838`(0/5) **`0x83C`(0/12)** `0x840`(0/1) `0x860`(0/1) `0x87C`(0/1) `0xC00`–`0xC0C` `0xC2C` `0x0FFC` `0x4104` `0x4108` |
| **Status block** `0x0000–0x001F` (loads only) | `0x0008`(0/15) **`0x000A`(0/27)** `0x0013`(0/14) **`0x0015`(0/4)** `0x0016`(0/15) + singles |
| **Read-modify-write control** (paired load→store, bit-field updates) | **`0x85C`(20/22 — busiest)** `0x854`(11/18) `0x844`(3/1) `0x848`(4/1) `0x84C`(3/2) `0x850`(5/2) |
| **Write-once parameters** (exactly 1 store; no functional read) | **`0x868` `0x86C` `0x870`** (DTS) + `0x858` `0x864` `0x874` `0x878` |
| **16-bit command/value pair** (all stores are `bn.sh_1`) | **`0x0020`(11 stores) / `0x003E`(9 stores)** + `0x0023` `0x0024` `0x0026` `0x002A` `0x002E` `0x0032` `0x0033` `0x0034` `0x003A` `0x003C` |
| **Config block** | `0xC20`(7/8) `0xC24`(1/1) `0xC28`(1/1) |

### §23.2 Where the transmit interface is

- **`0xB000_085C` is the busiest register and is used almost exclusively as read-modify-write.** Ten paired `bn.lwz`→`bn.sw` sites operate on it: `0xF7DA`/`0xF7E4`, `0xF883`/`0xF88E`, `0xFA6B`/`0xFA71`, `0x16C19`/`0x16C23`, `0x16C26`/`0x16C30`, `0x16CBD`/`0x16CC5`, `0x16CC8`/`0x16CD8`, `0x176F2`/`0x176F8`, `0x18075`/`0x1807D`, `0x19223`/`0x1922B`. ⇒ a **control register whose bits are individually set/cleared** — the main shared output/SPDIF control (consistent with the long-standing "`0x85C` = SPDIF CTRL" note). Same pattern for `0x854`/`0x850`/`0x844`/`0x848`/`0x84C`.
- **`0xB000_0020` / `0xB000_003E` form a 16-bit command/value pair** — every store to them is `bn.sh_1` (16-bit), from a small set of functions: `0xF1A7` (`0xF390`), `0xF693` (`0xF7B9`, `0xF8B1`, `0xF919`), `0xF946` (`0xFB5C`), `0x1149CB` (`0x114EE6`), and `0x115693` (`0x115471`, `0x1154F4`, `0x115577`, `0x1157A9`, `0x1157B1`, `0x1157CF`, `0x1157D7`). ⇒ an **audio-IP command/parameter interface**, not a data path.
- The DTS registers `0x868`/`0x86C`/`0x870` sit **outside both** mechanisms: write-once parameters, no functional reader (§21.3).

### §23.3 Answer to "how does this connect to the SPDIF/SDO transmit machinery?"

Statically supported:
- The firmware's **transmit-side interface** consists of **(a)** the shared control register **`0x85C`** (bit-field read-modify-write) and **(b)** the **16-bit command pair `0x0020`/`0x003E`**.
- The DTS-specific timing registers are **pure hardware parameters**: written once per output-config pass, never read for a decision, never used to gate anything.
- ⇒ The actual byte-push / IEC61937 framing is done by **hardware**; the firmware only *configures* it. There is **no firmware-side consumer** of the timing values, which is consistent with §22 and completes the consumer question.

**Residual (honest):** no single register could be isolated as a "transmit start" trigger distinguishable from the `0x85C` control bits and the `0x0020`/`0x003E` command pair — both are used by many callers, and separating "configure" from "start" requires the register datasheet. This is a genuine ISA/datasheet limitation, not an analysis gap.

---

## §24 — Status of the R46 risk register after this report

| Risk | Statement | Status now |
|---|---|---|
| **R1** | `0xB000_0C06` / `0xB000_0015` have zero writers | **CLOSED — VERIFIED BY GHIDRA** (0 writers each) |
| **R2** | caller counts are lower bounds | **OPEN** — not re-verified in this pass (Ghidra confirmed `0xB290` ≥6 in §20.3; other counts untouched) |
| **R3** | `0x86C`/`0x870` have exactly one writer each | **CLOSED — VERIFIED BY GHIDRA** |
| **R4** | `0x868` written by exactly one non-DTS function | **CLOSED — VERIFIED BY GHIDRA** (one writer, `0x178B9`) |

**New risk introduced by this report:** the "write-only / never read back" characterisation was **wrong as literally stated**; the corrected statement is "no functional reader, one telemetry reader each". Any downstream text relying on the literal form must be updated.

---

## §25 — Tooling added

| Tool | Purpose |
|---|---|
| `scripts/FindMMIO.java` | **Ghidra-only full-image MMIO store/load sweep** with register-constant tracking. Run: `-postScript FindMMIO.java "OUT:<path>"`. Output `r46_work/ghidra_writers.txt`. |
| `r46_work/r46_bytescan.py` | Decoder-free big-endian word-pattern scan (every byte offset) for address-construction candidates; the independent completeness cross-check. |
| `r46_work/ghidra_candidates.txt` | Ghidra disassembly of the cross-check candidates + the `0xE6xx` dump + the out-of-cluster `0x0015` readers. |

**No patch. No device access. No runtime experiment.**
