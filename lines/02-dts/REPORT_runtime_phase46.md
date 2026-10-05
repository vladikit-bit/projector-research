# REPORT — R46 — Full-image owner/consumer trace for `0xB000_086C` / `0xB000_0870`

**Subject device:** Thundeal TD98 Pro / MStar MT5889 (Android 11) — AEON (R2) audio DSP  
**Image under analysis:** `snd_full.bin` (SND R2 DSP firmware, `0x1C1330` bytes, base `0x0`, max code addr `0x1C132F`)  
**Method:** static-only. Full-image linear disassembly + backward abstract-interpretation register resolver (AEON decoder from `scripts/aeon_decode.py`, STATIC-verified against Ghidra slaspec). **No patch. No device access. No runtime MMIO. No license/AUTH/EDID reopening.** (Per R46 hard constraints.)  
**Prior phases:** R44 (misclassified these as MMIO; corrected in R45), R45 (established real MMIO writers `0xCD76`/`0xD0A1`, SHM base `0xA000` vs MMIO base `0xB000` distinction).

> **📌 Read `R46_FINAL_SUMMARY.md` first.** This phase report is the full chronological record (§0–§20 + appendices) and contains statements later corrected; the consolidated summary states the current authoritative position and is the better handover document.

---

## ⚠️ SUPERSEDED-CLAIMS REGISTER (added after R46b/R46c; do not read §0–§20 without it)

This report's sections are the *chronological* record. The following statements in them were later **corrected or withdrawn** by `REPORT_runtime_phase46b.md` (§21–§25) and `REPORT_runtime_phase46c.md` (§26–§40). **The later reports win.**

| Stale statement here (section)                                                                                                    | Status                                                                                                                                                                        | Corrected in     |
| --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| §0/§2/§12.2/Appendix C: `0x86C`/`0x870` are **"write-only"**, the DSP never reads them back                                       | **DISPROVED** — each has exactly **one** reader (`0xE674`/`0xE68D`/`0xE6A6`), all inside a telemetry dump. Correct wording: *no functional reader; one telemetry reader each* | R46b §21.3/§21.4 |
| §0/§16/§19: the timer operand is **`SPR(r5,16)`**                                                                                 | **WRONG** — it is `bg.mfspr1 rN,0x2808`; and that instruction is **not a timer read at all** — it is a **multiply-high transfer**                                             | R46c §18.3/§36   |
| §12.2/§16.4/§19.5: `0x86C`/`0x870`/`0x868` are **"sample-rate-derived SPDIF/IEC61937 timing registers"**, "strongly corroborated" | **WITHDRAWN** — `0x868` is counter-driven (0..67)×48, not rate-derived; and the `0x86C`/`0x870` input is a **ring-buffer byte offset**, not a time                            | R46c §27.3, §40  |
| §16.1/§16.2/§16.4/§19.1/§19.2: the `×0xA57EB503` multiply is **"dead code"**                                                      | **WRONG** — its **high word is consumed** by the following `mfspr1 …,0x2808`; the arithmetic is exact (`0xA57EB503/2³² = 64/99`)                                              | R46c §36.4/§36.5 |
| §16.2: `0x870 = (0xDD8000 / divisor) + Q`                                                                                         | **INCOMPLETE** — the division numerator is **zeroed** (`bt.movi r23,0x0` @`0xD034`), so `0x870 = Q` on the only path reaching the store                                       | R46c §27.2       |
| §16.1: `0x86C = Q`                                                                                                                | **INCOMPLETE** — there is a path-dependent additive term `T`                                                                                                                  | R46c §27.1       |
| §19.6/§28: `SPR 0x2808` is a **timer / monotonic counter / time base**                                                            | **CORRECTED** — it is the multiply-result transfer; the tick timer is `SPR 0x5000/0x5001`                                                                                     | R46c §36.4       |
| §13: `func 0xB290` is called by `0xC226` **only**                                                                                 | **WRONG** — ≥6 call sites (Python-walk undercount)                                                                                                                            | R46c §20.3       |
| §23.3 (R46b): "the actual byte-push / IEC61937 framing is done by **hardware**"                                                   | **DOWNGRADED** to STRONG INFERENCE — the values are either consumed by hardware or unused; the evidence cannot distinguish                                                    | R46c §32         |
| §30/§37.3/§38.4 (R46c): `0x86C`/`0x870` are **"interval-derived timing"**                                                         | **WITHDRAWN** — the input is a ring-buffer pointer delta in **bytes**                                                                                                         | R46c §40         |

**Net effect:** the *register inventory* (§1–§3) and the *writer sets* stand (and are Ghidra-verified in R46b §21); the *semantic* claims about these registers being "timing" do **not**. Current classification: **C — UNRESOLVED** (R46c §30/§36.6/§40).

---

**Companion artifacts:** `r46_work/r46_scan.py`, `r46_work/scan_out.txt`, `r46_work/r46_neighborhood.py`, `r46_work/neighborhood_out.txt`, `r46_work/r46_ownclues.py`.

---

## §0 — VERDICT (decision class R46-B)

`0xB000_086C` and `0xB000_0870` are **write-only, DTS-path-exclusive MMIO timing registers** in the SPDIF/HDMI-ARC output block (`0xB000_0800–0x08FF`). Across the **entire 1.84 MB firmware image**, each is written **exactly once**, both stores are on the DTS path (`0xCC9B` → `0xCD76` for `0x86C`; `0xCF86` → `0xD0A1` for `0x870`), and **neither is ever read back** anywhere in the image. Their values are **deterministically computed from sample-rate/timer/input state** (16-bit masked quotients). The firmware embeds the **DTS-X SDO packer + IEC61937 SPDIF framing** library, which strongly supports the interpretation that these are DTS-SDO burst/pause/sample-rate timing registers — but that mapping remains **STRONG INFERENCE**, not PROVEN (no register-name DB, no datasheet, no static edge tying a named SDO-packer parameter directly into these addresses).

→ **R46-B** for the *register* question (write-only, DTS-path-exclusive MMIO timing registers), **but see the §17 correction below for the root-cause question.**

**Root-cause status (corrected by §17):** the investigation is complete at the static level, but the earlier conclusion *"gate `0x25EE1` FAIL ⇒ no physical SPDIF TX"* is **NOT PROVEN and is hereby withdrawn**. What is established:

- the timing registers `0x86C`/`0x870` are written on the DTS path; their *value* correctness is unresolved (**H1**);
- the DTS-path gate `0x25EE1` is entered only on the DTS side (AC3 does not traverse it) and on PASS it calls `0x13292` (encoding-VERIFIED);
- the gate's two controlling inputs (`0xB000_0C06` bit7, `0xB000_0015`) have **no writer among decodable instructions** (§15, caveat §17.0);
- **however** `0x13292` has **12 call sites from 9 functions**, including `0xB290` which is **independent of the gate** (§17.4), and `0x13292`'s body shows **no provable MMIO/DMA/FIFO/trigger access** (§17.3). Its downstream helpers `0x10786E` (1919 call sites) and `0x115693` are **generic utilities**, not a DTS transmitter arm (§17.5).

⇒ **R46-FINAL-C**: `0x13292` is related to DTS state/configuration, but its relationship to physical TX is **UNRESOLVED**, and it is **not gate-exclusive**. **H1 (timing value) and H2 (gate) are co-equal**, not ranked; both may co-occur. **No patch produced.**

**§18 update (Ghidra ground truth, authoritative slaspec):** `0x13292` is now decoded — it is a **shared integer arithmetic / structure-store helper** with **no MMIO/DMA/FIFO/trigger**, so **Variant A and Variant B are refuted**; the gate **does return** (`0x25F81 bt.jr r9`); and the supposed `r31` threshold **does not exist** (`0x25F14` is `bn.sflesi r23,-0x1`, an immediate). Net effect: the chain `gate PASS → 0x13292 → 0x10786E → 0x115693` is a **computation/utility chain**, and gives **no evidence** that the gate controls physical TX. The gate is best described as a **DTS state/status evaluation routine** that returns a status in `r3`.

---

## §1 — Full-image scan (the object of R46 §1)

A linear disassembler walked **every 4-byte-aligned word** of `snd_full.bin` (`591,620` decoded instruction starts). For each store/load whose base is a register, a backward abstract interpreter (`movhi`/`ori`/`addi`/`muli` immediate-construction chains only) resolved the **effective address with verified base register** — never by low-offset alone. Hits resolving to the two targets:

| PC       | Mnemonic | Operands     | Effective MMIO | Base provenance                                      | Kind  |
| -------- | -------- | ------------ | -------------- | ---------------------------------------------------- | ----- |
| `0xCD76` | `bn.sw`  | `0(r26),r25` | `0xB000_086C`  | `r26 = movhi 0xb000 ; ori r26,0x86c` → `0xB000_086C` | STORE |
| `0xD0A1` | `bn.sw`  | `0(r26),r24` | `0xB000_0870`  | `r26 = movhi 0xb000 ; ori r26,0x870` → `0xB000_0870` | STORE |

**Total target hits across the whole image: 2.** Both are STORES. **Zero LOADs.** Both bases are built from the MMIO base `0xB000_0000` (`movhi 0xb000`), unambiguously distinct from the SHM base `0xA000`.

A raw big-endian literal scan for the bytes `B0 00 08 6C` / `B0 00 08 70` returned **0 hits** — i.e. no register-map data table or rodata entry embeds these addresses.

---

## §2 — Register-reference table (R46 §2)

**REAL STORE — MMIO targets (the only two in the whole image):**

| PC       | Func           | Op                 | Effective     | Base reg (proven)   | Src reg          | Class      | Confidence |
| -------- | -------------- | ------------------ | ------------- | ------------------- | ---------------- | ---------- | ---------- |
| `0xCD76` | `0xCC9B` (DTS) | `bn.sw 0(r26),r25` | `0xB000_086C` | `r26 = 0xB000_086C` | `r25` (quotient) | REAL STORE | VERIFIED   |
| `0xD0A1` | `0xCF86` (DTS) | `bn.sw 0(r26),r24` | `0xB000_0870` | `r26 = 0xB000_0870` | `r24` (quotient) | REAL STORE | VERIFIED   |

**REAL LOAD — MMIO targets:** **none** (0 in entire image).

**ADDRESS-CONSTRUCTION-ONLY:** none separate from the two stores above (the addresses are built inline immediately before each store; no independent "compute-into-register-then-use-as-index" pattern for these two offsets was found).

**Cross-reference — SHM equivalents (different region, base `0xA000`):** 7 hits, including READS, proving these are DSP-internal shared memory, NOT the hardware registers:

- `0xCE26`, `0xCE72`, `0xCF48` → `0xA86C` (STORE, DTS) — `bg.sw r24,2156(r23)`, `r23=0xA000`
- `0xD15E` → `0xA868` (STORE, DTS) — `bg.sw r24,2152(r23)`, `r23=0xA000`
- `0xB3DD` → `0xA870` (STORE) — `bg.sw r23,2160(r11)`, `r11=0xA000`
- `0xCE96` → `0xA86C` (**LOAD**) — `bn.lwz r25,0(r3)`, `r3=0xA86C`
- `0xD262` → `0xA868` (**LOAD**) — `bn.lwz r27,0(r3)`, `r3=0xA868`

> The SHM `+0x868`/`+0x86C` fields are **read back** by the DSP (`0xCE96`, `0xD262`) and cache-flush-published to the host (per R45). The MMIO `0xB000_086C`/`0xB000_0870` are **never read back**. This asymmetry is the cleanest static signal that the SHM fields are software/shared state while the MMIO registers are hardware-facing, write-only timing.

---

## §3 — Writers (R46 §3): the two known writers are the ONLY writers

**Yes — the two known writers (`0xCD76`, `0xD0A1`) are the only writers in the entire image.** This is itself strong evidence of DTS-path exclusivity.

**Value provenance (restated from R45, with corrected wording):**

- `0xCD76` (`0xB000_086C`) value in `r25` = `((sample_global[0x25194+0x14] + timer[0x2808]) >> 11) − sign`, then `& 0xFFFF` (16-bit masked). The value is **deterministically computed from sample-rate and timer state**; its correctness-for-hardware is **not established**.
- `0xD0A1` (`0xB000_0870`) value in `r24` = a 16-bit-masked quotient of an input field divided by a timer-derived term plus a sample-rate quotient. Also **deterministically computed from sample-rate/timer/input state**; correctness-for-hardware **not established**.

No new writers were decoded beyond `0xCD76`/`0xD0A1`.

---

## §4 — Readers / downstream hardware action (R46 §4)

**No reader of `0xB000_086C` or `0xB000_0870` exists anywhere in the image** (full linear scan, 0 LOADs; the `bg.opcode_2E` MMIO-touch ops adjacent to the writers — `0xCD91`/`0xCD9E` near `0xCD76`, `0xD0C3`/`0xD0C7` near `0xD0A1` — read `0x814`/`0x818`/`0x81C`-class status registers, **not** `0x86C`/`0x870`).

Therefore the requested "read → transform → branch/arithmetic → hardware action" trace **cannot be constructed statically**: the DSP writes the value and never inspects it again. The consumption is by the **hardware SPDIF/IEC output block**, which the DSP does not re-read. We can enumerate the *candidate* hardware consumers (SDO mode, IEC61937 burst length, pause interval, sample-rate divider, TX enable, FIFO timing, HDMI-SPDIF routing) but **cannot statically prove which one** without a datasheet or a runtime MMIO capture (both out of scope/blocked). The write-only, 16-bit-quotient, sample-rate-derived shape is consistent with a **burst-period / pause-interval / sample-rate timing register** — but that is inference, not proof.

---

## §5 — `0xB000_0800–0x08FF` neighborhood map, grouped, AC3-vs-DTS (R46 §5)

Full-image base-verified references in the block: **198** (STORE+LOAD). Summary by offset (ref count; STORE/LOAD; who touches it):

| Offset          | Refs   | Character                  | Notes                                                                                                                          |
| --------------- | ------ | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `0x800`–`0x840` | ~40    | mostly LOAD                | status/ISR-polling reads                                                                                                       |
| `0x814`         | 22     | LOAD                       | heavily-read status (likely fs/`nonpcm_sel`/`state`/`owner` per ISR strings)                                                   |
| `0x844`–`0x854` | ~50    | STORE+LOAD                 | **shared control/status** — written by many functions (`OTHER/unknown: 180` refs) **and by the DTS helper `0xCC9B` (18 refs)** |
| `0x85C`         | **42** | STORE+LOAD                 | **main shared SPDIF control/status register** — dominant shared control point, used by AC3 **and** DTS paths                   |
| `0x860`–`0x864` | few    | LOAD/STORE                 | shared                                                                                                                         |
| `0x868`         | 2      | LOAD + **STORE `0x178B9`** | MMIO `0xB000_0868` **is written by a non-DTS function (`0x178B9`, `OTHER`)** — i.e. `0x868` is shared/touched outside DTS      |
| **`0x86C`**     | **1**  | **STORE only (`0xCD76`)**  | **DTS-exclusive** — no other writer, no reader                                                                                 |
| **`0x870`**     | **1**  | **STORE only (`0xD0A1`)**  | **DTS-exclusive** — no other writer, no reader                                                                                 |
| `0x874`–`0x878` | few    | STORE/LOAD                 | shared                                                                                                                         |

**AC3-vs-DTS comparison:** the shared control registers (`0x844`–`0x85C`) are exercised by both paths. The two target registers sit **immediately after** the shared control block and are the only `0x08xx` registers written **exclusively on the DTS path**. The sibling `0x868` is written by a non-DTS function, so the DTS-specific divergence is precisely `0x86C`/`0x870`, not `0x868`. This rules out a "shared timing register misprogrammed" theory for `0x86C`/`0x870` — they are simply **not touched by AC3 at all**.

---

## §6 — Upgrade attempt: R45's "SDO/IEC61937 timing" hypothesis (R46 §6)

R45 rated `0x86C`/`0x870 = SDO/IEC61937 timing registers` as **STRONG INFERENCE**. R46 tries to push it to PROVEN / STRONG INFERENCE / UNRESOLVED.

**New corroborating evidence (R46 §8 strings):**

- The firmware embeds the **DTS-X SDO packer**: `DTSX_CORE2_API_SDO_Packer`, `DTSDecSDOPacker_API_Process`, `DTSDecSDOPacker_API_StartFrame`, `DTSDecSDOPacker_API_GetFrameSize`, `DTSDecSDOPacker_API_SetParam`.
- The SDO packer is parameterized by **`DTS_PARAM_SDO_PACKER_SPDIF_SAMPLE_RATE_I32`** and **`DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32`** with **`DTS_SDO_SPDIF_OUT`** — i.e. the firmware **does IEC61937 framing for SPDIF** and the SDO packer consumes a SPDIF sample rate. This is direct evidence that IEC61937/SPDIF burst framing exists in the DTS path.
- `dtsX spdif output reset`, `spdif_npcm_delay_flag`, `dts m6 enc: reset spdif`, `<dtsX>`, `D2A_DTSXENC` confirm a DTS-X encoder → SPDIF output path distinct from AC3 (`ms11 ddenc`, `MS12V2 ddpe`).

**Assessment:**

- The presence of the SDO packer + IEC61937 SPDIF framing in the same image that writes `0x86C`/`0x870` only on the DTS path **strengthens** the timing-register inference substantially.
- **Still not PROVEN:** no static instruction edge connects a named SDO-packer parameter (e.g. the SPDIF sample-rate) to a store into `0x86C`/`0x870`; the DSP writes these inline from arithmetic, and there is no register-name DB or datasheet. We also cannot prove these registers are *consumed by the SDO packer specifically* vs. by the generic SPDIF TX DMA/timer block.
- **Result: STRONG INFERENCE (well-corroborated), not PROVEN.** What would be needed for PROOF: (a) the MStar AEON audio MMIO register map naming `0x86C`/`0x870`, or (b) a runtime MMIO capture showing the hardware reading them on DTS TX and the value correlating with burst/pause timing. Both are out of scope/blocked.

---

## §7 — Wording correction (R46 §7)

Per R46 §7, this report does **not** claim the computed values are "correct-by-construction." The correct statement: the values written to `0xB000_086C`/`0xB000_0870` are **deterministically computed from sample-rate/timer/input state** (16-bit masked). Their correctness for the hardware transport is **not established** by static analysis.

---

## §8 — Ownership clues from the workspace (R46 §8)

**Searched:** binary ASCII strings (≥4 printable), Ghidra dumps, decoded text, reports.

- **Register-name DB:** **none found.** No symbol table, no `utpa2k` register-name structure, no rodata mapping `0x86C`/`0x870` to a name. The raw literal scan found 0 byte patterns `B0 00 08 6C`/`B0 00 08 70`.
- **Vendor headers / MStar/MTK audio sources:** not present in the workspace; only the SND DSP binary + Ghidra dumps. No header defining the `0xB000_08xx` block by name.
- **String evidence (firmware-internal, from `snd_full.bin`):** the DTS-X SDO packer + IEC61937 SPDIF framing library and DTS-X SPDIF output path (see §6). These are the **only** symbolic ownership hints and they point to the **DTS SDO packer / IEC61937 SPDIF output** subsystem.
- Per R46 §8 discipline, **no symbolic register name is asserted** beyond what the strings support. I refer to these only as "DTS-SDO/IEC61937 SPDIF timing registers (inferred)" — never as a vendor-defined name.

---

## §9 — Q1–Q6

- **Q1 — Written elsewhere in the image?** No. Exactly two stores in the entire `0x1C1330`-byte image: `0xCD76` (`0x86C`), `0xD0A1` (`0x870`). Both DTS path.
- **Q2 — Ever read back?** No. 0 loads of either register anywhere. (Contrast SHM `+0x86C`/`+0x868`, which ARE read back at `0xCE96`/`0xD262`.)
- **Q3 — Value followed into a hardware op?** Not provable statically. Write-only MMIO; hardware consumes. Candidate consumers: SDO mode / IEC61937 burst length / pause interval / sample-rate divider / TX-enable / FIFO timing — none statically proven.
- **Q4 — Ownership identified?** DTS-path-exclusive (VERIFIED). Subsystem ownership = **DTS SDO packer / IEC61937 SPDIF output** = **STRONG INFERENCE** (corroborated by embedded SDO-packer strings), not PROVEN.
- **Q5 — Strongest AC3-vs-DTS divergence?** AC3 **never writes** `0xB000_086C`/`0xB000_0870`; DTS writes each once with sample-rate/timer-derived 16-bit quotients. The shared control block (`0x844`–`0x85C`, esp. `0x85C` ×42) is used by both. Sibling `0x868` is shared (written by non-DTS `0x178B9`). So the divergence is exactly the two DTS-only timing registers.
- **Q6 — Enough evidence to design an experiment?** Yes — a controlled runtime experiment is the natural next step: on DTS passthrough, capture `0xB000_086C`/`0xB000_0870` (and compare to AC3, which leaves them at reset/defaults); verify the written value matches the expected IEC61937 burst-period/pause for the DTS sample rate; test whether forcing the AC3-default value on DTS unblocks TX. **However, runtime MMIO access is blocked by the standing STOP (device panic on `/dev/malloc`, DM mirror doesn't reflect MMIO, no safe peek facility).** The experiment is documented for when device access is permitted; it is **not executed** here.

---

## §10 — Addendum: execution-ordered differential + downstream-consumer quest (continuation of R46 §5)

R46 §5 mapped the `0x0800–0x08FF` block **by offset**. This addendum re-presents the DTS and AC3 helper accesses **in execution (PC) order** (base-verified per-instruction) and runs the downstream-consumer quest. Source: `r46_work/r46_seq.py` over `r29_work/decoded_all6.txt`.

**DTS helper `0xCC9B` (excerpt around `0x86C`):**

```
0xCCA9  bn.lwz r24,0(r24)  -> 0xB000_0814  LOAD   (status)
0xCCB6  bn.lwz r24,0(r23)  -> 0xB000_080C  LOAD   (status)
0xCCBC  bn.lwz r23,0(r23)  -> 0xB000_080C  LOAD   (status)
0xCCE5  bn.lwz r24,0(r24)  -> 0xB000_0854  LOAD   (shared SPDIF CTRL)
0xCD76  bn.sw  0(r26),r25  -> 0xB000_086C  STORE  <<< TARGET (DTS timing)
0xCD7D  bn.lwz r25,0(r25)  -> 0xB000_0814  LOAD   (status)
0xCD8E  bn.lwz r26,0(r24)  -> 0xB000_0818  LOAD   (status)
```

**DTS helper `0xCF86` (excerpt around `0x870`):**

```
0xCF96  bn.lwz r23,0(r23)  -> 0xB000_0814  LOAD   (status)
0xD002  bn.lwz r25,0(r23)  -> 0xB000_0854  LOAD   (shared SPDIF CTRL)
0xD00F  bn.sw  0(r23),r25  -> 0xB000_0854  STORE  (shared SPDIF CTRL)
0xD016  bn.sw  0(r24),r0   -> 0xB000_0850  STORE  (shared enable/clear)
0xD0A1  bn.sw  0(r26),r24  -> 0xB000_0870  STORE  <<< TARGET (DTS timing)
0xD0A8  bn.lwz r26,0(r27)  -> 0xB000_0814  LOAD   (status)
0xD0BD  bn.lwz r24,0(r24)  -> 0xB000_081C  LOAD   (status)
0xD0C0  bn.lwz r23,0(r28)  -> 0xB000_0818  LOAD   (status)
```

**AC3 helpers (`0xCB50`, `0xCAB4`, `0xC49D`, `0xC5C5`):** every `0x08xx` access is either a status **LOAD** (`0x80C`/`0x810`/`0x814`/`0x818`/`0x81C`) or a shared-control **STORE** (`0x850`/`0x84C`/`0x854`). **None writes `0x86C` or `0x870`.** (Consistent with §1: the whole image has 0 writers of those two registers outside the DTS path.)

**Downstream-consumer quest result:** immediately *after* each TARGET timing store, the following accesses are **status LOADs** (`0x814`/`0x818`/`0x81C`) — there is **no local apply/enable/trigger store** adjacent to either `0x86C` or `0x870`. The DSP writes the timing quotient and then returns to polling hardware status; it never reads the timing register back. This means:

1. The timing registers are **configuration inputs** to the SPDIF/IEC output block, consumed by **hardware** (not by a later DSP store in the same helper).
2. There is **no discrete "commit" register** written right after them in the DTS helper — the consumer is the output block itself, read continuously or on the next frame boundary.
3. This **reaffirms R46-B** (downstream consumer = hardware output block, not statically resolvable to a named sub-block). It rules out a trivial "write-timing-then-write-enable" local pattern; the enable/apply path is the shared `0x854`/`0x850` control (used by both AC3 and DTS) plus the shared finalizer leaf `0x115693`.

**Execution-order AC3-vs-DTS differential (sharpened):**

- AC3: `read status (0x80C/0x810/0x814/0x818/0x81C)` → `write shared control (0x850/0x84C/0x854)`. No `0x86x` timing write.
- DTS: `read status` → `write shared control (0x850/0x854)` → **additionally write DTS timing (`0x86C` in `0xCC9B`, `0x870` in `0xCF86`)**.
- The sibling `0xB000_0868` is written by a **non-DTS** function (`0x178B9`, §5/§7 note) — so the only strictly DTS-exclusive timing registers are `0x86C`/`0x870`.

**Ownership conclusion (unchanged):** `0x86C`/`0x870` belong to the **DTS SPDIF/IEC61937 output register bank** (same bank as `0x850`/`0x854`, immediately adjacent, written in the same output-config sequence), strongly inferred = DTS-SDO burst/pause/sample-rate timing, **STRONG INFERENCE not PROVEN**. R46-B stands.

## §11 — Addendum 2: the `0xB000_0868` writer is NOT the AC3 path (caller analysis)

R46 §5/§7 noted a MMIO store to `0xB000_0868` at `0x178B9` in a *non-DTS* function. A natural hypothesis was "AC3 supplies timing via `0x868` while DTS uses `0x86C`/`0x870`." This addendum tests that hypothesis and **refutes it**.

**Decode of `0x178B9` (region `0x17000–0x18000`):**

```
0x178AE  bg.movhi r23,0xB000
0x178B5  bg.ori   r23,r23,0x868      ; r23 = 0xB000_0868
0x17898  bn.movhi_2 r27,0xFF
0x178AA  bg.ori   r27,r27,0xFFFF     ; r27 = 0x00FFFFFF (24-bit mask)
0x17868  bg.lwz   r13,2928(r10)      ; r13 = config struct field @r10+2928
0x17874  bg.muli  r13,r13,48         ; r13 = field * 48
0x178B2  bn.and   r24,r13,r27        ; r24 = (field*48) & 0x00FFFFFF
0x178B9  bn.sw    0(r23),r24         ; STORE 0xB000_0868 = (field*48)&0xFFFFFF
```

Value = a **16/24-bit timing/size quotient** (`field × 48`, masked) — same shape as `0x86C`/`0x870`. The `×48` is consistent with sample-rate scaling (48 kHz base). The same function also writes 6 channels of (shift-left-8) data into a buffer at `r24` (0x17913–0x17926) — a channel/coefficient-table setup.

**Full-image caller scan (`r46_work/r46_callers.py`):** every call/jump target into `0x17000–0x18000` has its callers **entirely within `0x17xxx–0x19xxx`**. No caller is in the AC3 helpers (`0xCB50`/`0xCAB4`/`0xC49D`/`0xC5C5`), the DTS output-config helpers (`0xCC9B`/`0xCF86`), the gate (`0x25EE1`), the finalizer (`0x115693`), or the DMA (`0x13292`). The function entry (~`0x177A6`/`0x177B5`) is reached only from peer functions in `0x18xxx`.

**Conclusion:**

- `0xB000_0868` is written by a routine in the **`0x17xxx–0x19xxx` cluster**, which is the **DTS-X SDO packer library region** (per the embedded strings of §8: `DTSX_CORE2_API_SDO_Packer`, `DTSDecSDOPacker_API_*`, `dtsX_core2`, `D2A_DTSXENC`). It is **not** reached from the AC3 output path.
- Therefore the corrected register-ownership map is:
  - `0x86C` → DTS output-config helper `0xCC9B` (DTS-exclusive).
  - `0x870` → DTS output-config helper `0xCF86` (DTS-exclusive).
  - `0x868` → DTS-X SDO packer cluster (`0x178B9`), DTS-path only (not AC3).
  - **AC3 writes NONE of `0x868`/`0x86C`/`0x870`** — it relies on hardware defaults for all three.
- The original "AC3 uses `0x868`" hypothesis is **refuted**; `0x868` is a *third* DTS-specific timing register (sibling to `0x86C`/`0x870`), written from the SDO-packer library rather than the output-config helpers.
- This **strengthens** the DTS-vs-AC3 divergence: DTS explicitly programs three SPDIF/IEC61937 timing registers (`0x868` via SDO packer, `0x86C`/`0x870` via output-config); AC3 programs none and depends on silicon defaults. If any of the three DTS values is wrong for the actual transport, DTS mis-frames while AC3 (default) works — consistent with the observed failure.

**Caveat (honest):** the SDO-packer cluster's reachability *from the DTS decoder stage* (not the output-config helpers) is inferred from the string evidence + self-contained caller set, not from a traced edge into the decoder funnel; that edge is out of the examined helper set. The statement "AC3 does not reach it" is VERIFIED (no AC3-helper/gate/finalizer/DMA caller). The statement "it is the DTS SDO packer" is STRONG INFERENCE.

## §12 — Capstone synthesis: static causal model, root-cause ranking, decisive experiment

This addendum consolidates R46 + §10 + §11 into a single coherent model. Source tooling: `r46_work/r46_liveness.py` (caller check proving `0xCC9B`/`0xCF86` are live).

### §12.1 VERIFIED static facts

1. `0xB000_086C` is written exactly once in the image: `0xCD76` (helper `0xCC9B`); `0xB000_0870` exactly once: `0xD0A1` (helper `0xCF86`). Zero reads of either, anywhere.
2. `0xB000_0868` is written once: `0x178B9`, inside the `0x17xxx–0x19xxx` SDO-packer cluster.
3. `0xCC9B`/`0xCF86` are **live** (5/6 callers), including the SDO-packer cluster, the DTS dispatch region (`0xB368`/`0xB37B`), and the gate-adjacent region (`0x265FD`/`0x266E0`). They are NOT dead code.
4. **No AC3 helper, gate `0x25EE1`, finalizer `0x115693`, or DMA `0x13292` calls any of `0xCC9B`/`0xCF86`/`0x178B9`.** AC3 reaches none of the three timing writers.
5. The three registers are write-only; the DSP never reads them back. The SHM siblings (`+0x868`/`+0x86C`) are read back and cache-flush-published (different region, base `0xA000`).
6. All three values are 16/24-bit quotients **deterministically computed from sample-rate/timer/input state** (`0x86C`: `(sample_global[0x25194+0x14] + timer[0x2808])>>11 − sign`; `0x870`: div of input field by timer-derived + sample quotient; `0x868`: `config_field × 48 & 0xFFFFFF`). Correctness-for-hardware is **not established**.

### §12.2 Root-cause hypotheses (ranked)

- **H1 (STRONGEST): DTS writes incorrect / transport-mismatched timing to `0x868`/`0x86C`/`0x870`.** DTS overrides the silicon defaults that AC3 relies on; if any quotient is wrong for the actual SPDIF/IEC61937 burst (sample rate, pause interval, burst length), DTS mis-frames and the sink rejects it while AC3 (default) works. Consistent with all VERIFIED facts. **Cannot be confirmed statically** — value correctness needs a datasheet or runtime capture.
- **H2 (weaker): DTS writes the timing but the gate `0x25EE1` blocks the enable (`0x10786E` finalizer) so TX never arms.** Runtime evidence from prior phases (R16/R17/R22) saw the gate inputs block in observed states. If true, the timing registers are written but TX is never enabled → DTS silent. This is a *gate* failure, not a timing-value failure; the two are not mutually exclusive.
- **H3 (weakest): AC3's "default" path is actually configured elsewhere (e.g. a different register/init) that DTS skips.** The full-image scan found no AC3 writer of `0x868`/`0x86C`/`0x870`, so AC3 genuinely uses hardware defaults for these; H3 is essentially ruled out for these three registers.

> **⚠ Superseded in part by §17 (read this ranking as historical).** §17 shows the H2 mechanism is **not** established (`0x13292` is not a proven TX arm and is not gate-exclusive), and §17.3 shows the H1 formulas are concrete but unvalidated. The **final** stance is therefore **H1 and H2 co-equal, neither ranked**, with H3 still ruled out. See §17.8 / §17.10.

### §12.3 Decisive experiment (documented; runtime access BLOCKED)

To resolve H1 vs H2 and confirm/correct the timing values:

1. On **real Kodi DTS passthrough** (the only path that engages the gate — probe7 does not), capture `DM[]`-equivalent / safe MMIO readback of `0xB000_0868`/`0xB000_086C`/`0xB000_0870` and the shared `0x854`/`0x850` just after the DTS output helpers run.
2. Repeat for **AC3 passthrough** (which leaves these at reset/defaults).
3. Compare the captured DTS quotients against the IEC61937 burst-period / pause-interval formula for the DTS sample rate (48k/44.1k). A mismatch = H1 confirmed; capture the correct quotient and the register it must land in.
4. Separately, capture the gate inputs (`byte@0xB000_0C06` bit7, `0xB000_0015`, `0x2512C4`) during DTS to confirm H2.

- **Blocked:** `/dev/malloc` MMIO read panics the device (R43 STOP); `DM[]` does not reflect HW MMIO writes (R8.2); no safe readback path exists. The experiment is specified for when device access is permitted; it is **not executed**.

### §12.4 Final verdict

**R46-B** stands, now with higher confidence that the DTS-vs-AC3 divergence at `0x868`/`0x86C`/`0x870` is **real, live, and reached on the DTS path** (not dead code, not AC3-reachable). The registers are DTS SPDIF/IEC61937 output timing configuration (STRONG INFERENCE). The only unresolved link is **whether the written values are correct for the transport** — that is a runtime/datasheet question, outside static scope. **No patch.**

## §13 — Addendum 3: gate-vs-timing ordering (H1 vs H2 discriminator)

This addendum settles the open structural question from §12.2: does the gate `0x25EE1` gate the timing write (so a gate failure would also starve `0x86C`/`0x870`), or are the two independent? That ordering is the decisive structural fact for H1 (timing written, value wrong) vs H2 (gate blocks TX-arm only). Tooling: `r46_work/r46_gatectx.py` (alignment-correct gate-region decode pulled from the full-image index `S.idx`), `r46_work/r46_order.py` (whole-image caller map), `r46_work/r46_order3.py` (region-bounded call-target scan). `r46_work/r46_order2.py` was an intermediate that used a fragile function-exit heuristic and returned empty call-sets for large functions — its `calls_of()` results are discarded; only its `callers_of()` (whole-image) results are used.

### §13.1 What the gate actually is (VERIFIED)

- `0x25EE1` is a function entry (`bn.addi r1,r1,-32` prologue). Its body `0x25EE1..0x25F91` is a **pure predicate** (alignment-correct decode from `S.idx`):
  - `0x25EF2 bg.ori r24,r23,0xC06` → `0x25EF6 bn.lbz r12,0(r24)` : reads `byte@0xB000_0C06` (DTS license/state flag; bit7 isolated at `0x25F0D bn.srli r11,r12,7`).
  - `0x25EF9 bn.ori r23,r23,0x15` → `0x25EFC bn.lbz r23,0(r23)` : reads `byte@0xB000_0015` (status).
  - `0x25F06 bg.addi r13,r13,4500` (r13 = `0x250000+0x1194 = 0x251194`) → `0x25F10 bg.lwz r24,304(r13)` : reads `word@0x2512C4` (state word).
  - Branch outcome at `0x25F26 bg.beq r23,r11 -> 0x25F36` and `0x25F2A bn.bnei r11,0 -> 0x25F91`.
- The gate body contains **no store to `0x86C`/`0x870`, no call to `0xCC9B`/`0xCF86`/`0x265FD`/`0x266E0`, no call to the finalizer `0x10786E` or DMA `0x13292`** (VERIFIED via the region-bounded scan `0x25EE1..0x25F91` — the only matches in that window for the `want` set are empty). The finalizer/DMA calls sit in the *surrounding* functions (`0x25C1C`, `0x25FC3`, `0x26270`), not in the predicate.
- Predicate inputs are hardware/license **STATE** (`0xB000_0C06`, `0xB000_0015`, `0x2512C4`) — **never** the timing registers. The gate decides DTS permission/active-state, not timing programming.

### §13.2 The timing writers are NOT downstream of the gate (VERIFIED)

- `0xCC9B`/`0xCF86` have 5/6 callers spanning the DTS path (whole-image caller map, `r46_order.py`):
  - `0xB290` (DTS dispatch leaf) → `0xCC9B` @ `0xB368`, `0xCF86` @ `0xB37B` — VERIFIED by region scan `0xB290..0xB900` (`0xCC9B<-0xB368`, `0xCF86<-0xB37B`).
  - `0x1EC54`, `0x1F575` (mid-level DTS).
  - `0x26270` → via `0x265FD`→`0xCC9B`, `0x266E0`→`0xCF86`. VERIFIED: `0x265FD`/`0x266E0` have **zero** direct callers; they are reached by fall-through/branch *inside* `func 0x26270`.
  - `0x19490`/`0x1954C`/`0x1976E` (the `0x17xxx–0x19xxx` SDO-packer cluster).
- ~~`func 0xB290` is called by `func 0xC226` only~~ → **CORRECTED by §20.3:** `0xB290` has **at least 6 call sites** (3 in `0xC226`, plus `0xC30D`, `0xC46F`, `0xC48B`); the "`0xC226` only" reading was a Python-walk undercount. `func 0xB290` still does **NOT** call the gate `0x25EE1`, and the gate body contains no `0xB290` target. The gate `0x25EE1` is called from **4 sites** — **no caller/callee edge exists between the gate and `0xB290` in either direction** (this, not caller exclusivity, is what the independence argument rests on).
- Within the `0x25xxx` DTS state-machine cluster, the timing writers appear **only** inside `func 0x26270` (`0x265FD`/`0x266E0`), reached *after* `func 0x25FC3` (DMA `0x13292` @ `0x261C2`/`0x2624B`) and after the gate `0x25EE1`. The gate body never transfers control to `0x26270`'s timing writes.

### §13.3 Consequence for H1 vs H2

- **Ruled out (structural):** the H2 sub-case "the gate prevents the timing registers from ever being written." They ARE written on the DTS path (`0xB290`→`0xCC9B`/`0xCF86`; `0x26270`→timing) **independently of** the gate's go/no-go. The gate predicate is orthogonal to `0x86C`/`0x870`.
- **Therefore the failure is NOT "timing never programmed."** If DTS transmits nothing, the registers hold a deterministically-computed value — the only open question is whether that value is correct (H1) or whether TX is independently disarmed (H2).
- **H1 (timing value wrong) — now structurally favored as the "registers-written-but-output-fails" explanation.** The values are deterministically computed from sample-rate/timer/input (`§12.1.6`); a DTS-vs-AC3 formula/constant discrepancy is a *data* difference, invisible to static control-flow — exactly the shape of "written, but wrong for the transport."
- **H2 (gate blocks TX-arm) — still LIVE but now a *separate* blocker, not the controller of the timing write.** If the runtime gate predicate (bit7 of `0xB000_0C06`, etc.) is false for DTS, the finalizer/DMA calls in `func 0x25C1C`/`0x25FC3` may be skipped, leaving TX disarmed even though `0x86C`/`0x870` were written. Distinct failure mode (also yields "DTS completely silent"), and **not statically resolvable** because the predicate is runtime hardware/license state.
- **Both remain UNRESOLVED at the static level (R46-D):** the gate predicate values are runtime; timing-value correctness needs a datasheet or runtime capture. No new evidence closes the gap.

### §13.4 Updated root-cause ranking

- **H1** (STRONG INFERENCE, structurally favored): timing is written on the DTS path; if output fails it is the *value* (or a second, parallel gate/transport control), not "never written."
- **H2** (STRONG INFERENCE, separate blocker): gate may independently disarm TX; standing, but now understood as orthogonal to the timing write, not its controller.
- **H3** (ruled out, unchanged).
- Verdict **R46-B** confirmed; this addendum removes one ambiguous H2 sub-case and tightens the model. **No patch.**

> **⚠ Superseded in part by §17.** This ranking predates §17. Because the gate→TX link is **unproven** (§17.3/§17.4/§17.5), H1 can no longer be preferred over H2 on the grounds that "TX is blocked anyway" — and equally H2 can no longer be preferred on the grounds that "the gate is a proven TX gate". **Final: H1 = H2, co-equal** (§17.10).

## §14 — Addendum 4: the gate gates a call to `0x13292` — **(see §17 CORRECTION: the "TX/DMA arm" interpretation is NOT proven)**

§13 proved the gate does NOT gate the timing *write*. This addendum establishes the complementary fact: on PASS the gate **does** perform a call to `0x13292` that the non-PASS path does not. Tooling: `r46_work/r46_gatecallers.py` (alignment-correct decode of gate body `0x25EE1..0x26000` + the four caller sites).

> **⚠ Naming correction (applies to §11–§16):** in earlier sections `0x13292` is called "the DMA" / "TX-arm" by inherited label. **§17 shows that label is unsupported** — no DMA/FIFO/trigger/MMIO behaviour is provable in `0x13292`, and it is not gate-exclusive. Where §11–§16 say "DMA `0x13292`", read it as "`0x13292` (unresolved semantics)".

### §14.1 The gate is a DTS-path state machine that calls `0x13292` on PASS (VERIFIED)

Full alignment-correct decode of `0x25EE1..0x25FC3`:

- **Predicate:** `r11 = (byte@0xB000_0C06 >> 7)`  [DTS-license/state bit7, `0x25F0D bn.srli r11,r12,7`]  
  `AND (signed byte@0xB000_0015 <= -1)`  [`0x25F14 bn.sflesi r23,-0x1` — **immediate**, per §18.3; the earlier `r31` reading was a decoder artifact. `0x25F1B bn.cmov_0 r11,r11,r0` zeroes `r11` if the test fails]  
  `AND (word@0x2512C4 != 0)`  [`0x25F10 bg.lwz r24,304(r13)` with r13=`0x251194`; `0x25F1E bn.sfeqi r24,r0`; `0x25F21 bn.cmov_0 r11,r11,r0` zeroes `r11` if `==0`].  
  `r11` is stored to SHM `184(r10)` at `0x25F5A`.
- **Branch:** `0x25F2A bn.bnei r11,0 -> 0x25F91`. So `r11 != 0` (DTS allowed) takes the PASS path.
- **PASS path `0x25F91..0x25FBB`:** sets up `r3/r4/r5/r7` constants, then `0x25FBB bg.jal 0x13292`, returns via `0x25FBF bg.j 0x25F2D` (loop).
- **FAIL path (`r11 == 0`):** falls through `0x25F2F..0x25F91` **without** that call.
- The body is a **loop** (`0x25F68 → 0x25F7E → 0x25F83 → 0x25F8D → 0x25F68`) that re-checks and, on the allowed condition, performs the call.
- **CONCLUSION (VERIFIED):** the gate `0x25EE1` is a **DTS-path state machine** that calls `0x13292` **only when the DTS-license/state predicate passes**. The gating *logic* is certain; only the *runtime* predicate value is unknown.
- **⚠ CORRECTION (§17):** `0x13292` was labelled "the SPDIF/IEC output DMA" by inference only. §17.3 shows its body has **no provable MMIO/DMA-descriptor/FIFO/trigger access**, and §17.4 shows it has **12 call sites from 9 functions** including the gate-independent `0xB290`. It must **not** be called "DMA", and "gate FAIL ⇒ no TX" does **not** follow. See §17.

### §14.2 The gate is DTS-path (AC3 does not reach it) (VERIFIED)

- **Gate caller set (ONE fresh whole-image recomputation; use this list consistently — §8 of the R46-FINAL request):**
  | call site | enclosing function entry                          |
  | --------- | ------------------------------------------------- |
  | `0xCECC`  | `0xC91A`                                          |
  | `0xD2C8`  | *(unresolved — no prologue found in scan window)* |
  | `0xEC82`  | `0xE9E5`                                          |
  | `0xEF5E`  | `0xED6E`                                          |
  **Exactly 4 call sites**, all `bg.jal`. **None** are AC3 helpers (`0xCB50`/`0xCAB4`/`0xC49D`/`0xC5C5`). NOTE: older text mixed call sites with function entries (`0xD2C8` is a **call site**, not a resolved function entry) — the table above is the corrected, single authoritative list.
- Callers do DTS timing work: e.g. `0xC91A` computes a `×48` quotient right after the gate (`0xCEF5 bn.muls r24,r24,48` ; `0xCF03 bn.divu r25,r23,r24`) — same shape as the `0x86C`/`0x870` writers (`§12.1.6`).
- ⇒ AC3 transmits via a **different** (not gated by this predicate) path; the gate is the **DTS-specific** TX gate. This cleanly explains "AC3 works, DTS silent."

### §14.3 Revised root-cause ranking

- **H2 (RE-GRADED by §17 — STRONG INFERENCE, mechanism NOT proven):** the gate `0x25EE1` gates the call to `0x13292` on the DTS-license/state predicate (bit7 of `0xB000_0C06`, `byte@0xB000_0015` threshold, `word@0x2512C4`). **But** the step "this call = TX arm" is **UNRESOLVED** (§17.3), `0x13292` is **not gate-exclusive** (§17.4), and its downstream helpers are **generic** (§17.5). So "gate FAIL ⇒ DTS silent" is **NOT established** — it remains a candidate, not the leading explanation by default.
- **H1 (STRONG INFERENCE, restored to co-equal by §17):** timing values `0x86C`/`0x870`/`0x868` may be wrong. Since the gate→TX link is unproven, H1 can no longer be demoted on the grounds that "TX is blocked anyway". **H1 and H2 are co-equal and may co-occur.**
- **H3 (ruled out).**
- Decisive experiment (still blocked) must capture **both**: (a) the gate predicate inputs during DTS (`bit7@0xB000_0C06`, `0xB000_0015`, `0x2512C4`) to confirm H2; (b) `0x86C`/`0x870` values to confirm H1.

### §14.4 Verdict

**R46-B** for the register question, sharpened: the DTS-vs-AC3 divergence is localized to **two DTS-specific things** — (1) timing registers `0x86C`/`0x870` (written, value-questionable, §13), and (2) the gate `0x25EE1`, which is DTS-path-only (AC3 does not traverse it, §14.2) and whose predicate inputs have no firmware writer (§15). **What is NOT established** is that the gate controls physical TX: see **§17**, which re-classifies this as **R46-FINAL-C**. **No patch.**

## §15 — Addendum 5: the gate's predicate inputs have no firmware writer — **(see §17 CORRECTION: this alone does not prove "no TX")**

This addendum establishes the provenance of the gate's three predicate inputs. Tooling: `r46_work/r46_predwriters.py` (full-image writer trace of the predicate inputs + the `0xB000_0C00–0xCFF` status block and the `0x251000–0x251FFF` state region).

**⚠ Scope of this result (see §17.0):** "no writer" here means **no writer among decodable instructions** (79.7% of the image decodes). It is strong evidence for "hardware/license-owned", but not an absolute proof. **And it does not by itself prove that a gate FAIL suppresses physical TX** — that inference additionally requires `0x13292` to be a TX arm, which §17 shows is unresolved.

### §15.1 Who writes the predicate inputs? (VERIFIED)

- `word@0x2512C4` (must be ≠ 0): written **exactly once** in the image — `0x172FD bg.sw r23,304(r26)` — inside the **DTS-X SDO PACKER** cluster (`0x17xxx`). So the DTS SDO packer supplies this term when it runs.
- `byte@0xB000_0C06` (bit7 = DTS-license/state): **no writer found** among decodable instructions (§17.0 caveat). Read-only hardware status (read at gate `0x25EF6`, `func 0x25FC3` `0x25FDE`, `0x26064`, `0x262EC`).
- `byte@0xB000_0015` (status; tested `<= -1` as a signed byte — **not** against `r31`, see §18.3): **no writer found** among decodable instructions (§17.0 caveat). Read-only hardware status (read at gate `0x25EFC`). The `0xB000_0C00–0xCFF` block survey finds only `0xB000_0C24`/`0xC28` written — never `0x15`.
- Readers of `0xB000_0C06` / `0x2512C4` are confined to the gate cluster (`0x25Exx–0x262xx`) — corroborating they are the gate's DTS-state check.

### §15.2 Consequence (decisive for H2)

- The gate's pass/fail for DTS depends on **two hardware/license-status terms** (`0xB000_0C06` bit7, `0xB000_0015`) for which **no firmware writer was found** (caveat §17.0), plus the third term (`0x2512C4`) supplied by the DTS SDO packer.
- **On `r31` — SUPERSEDED by §18.3 (Ghidra ground truth): there is no `r31` operand.** `0x25F14` is `bn.sflesi r23,-0x1`, an **immediate** compare (slaspec `:bg.sflesi i32_rD, i32_simm5_16`); the earlier `r31` reading was a decoder artifact. The second predicate term is simply **"`byte@0xB000_0015` is negative (≥ 0x80)"**. No threshold register exists, so the entire `r31` provenance question is moot.
- ⇒ The gate behaves as a **hardware/license gate** in the sense that firmware was not observed setting its controlling inputs. **But (§17) this does not establish that a FAIL suppresses physical TX**, because the gated callee `0x13292` is neither proven to be a TX arm nor gate-exclusive.
- **RE-GRADED:** H2 is **not** the definitive root cause. H1 (timing value) is **not** demoted. Per §17.10 the two are **co-equal**.

### §15.3 How this would bear on DTS (conditional, per §17)

- **If** the gate's PASS path is in fact the physical TX arm (unproven, §17), then the hardware/license state `0xB000_0C06` bit7 and `byte@0xB000_0015` must be asserted for DTS to transmit, and the fix would be a **license/enable/config action**, not a timing-register change.
- Because that premise is unproven, this is a **conditional** statement, not a conclusion. No patch either way (R46 no-patch discipline; R43 device-access stop).
- Decisive experiment (still blocked) must now cover **three** things: (a) `0xB000_0C06` bit7 and `byte@0xB000_0015` (H2); (b) `0x86C`/`0x870` values (H1); and (c) **what `0x13292` actually does** — the missing link identified in §17.

### §15.4 Verdict (superseded — see §17)

This section's original text declared H2 the definitive root cause. **That is withdrawn.** The surviving, well-supported statements are: the gate is **DTS-path-only** (AC3 does not traverse it, §14.2); its two controlling inputs have **no observed firmware writer** (§15.1, caveat §17.0); and it **returns a status in `r3`** that every caller branches on (§17.6). What does **not** follow is "gate FAIL ⇒ no physical DTS transmission". Final classification is in **§17.10: R46-FINAL-C**.

## §16 — Addendum 6: exact timing-value formulas at the three DTS writers (H1 evidence base) — **validated & corrected by §19**

§12.1.6 only sketched the timing computations. This addendum decodes them in full (alignment-correct, from `S.idx`) so the H1 evidence base is concrete and a future analyst / datasheet can check the values. Tooling: `r46_work/r46_timingvals.py`.

### §16.1 `0x86C` — helper `0xCC9B`, store at `0xCD76` (value in `r25`)

- `0xCD41 bn.lwz r23,20(r10)` with `r10=0x251194` → `r23 = word@0x2511A8` (DTS state word).
- `0xCD4C bn.muls r25,r23,0xA57EB503` then `0xCD53 bn.add r25,r23,r26` (`r26 = SPR(r5,16)`, a timer/clock SPR) → **the `×0xA57EB503` result is overwritten** (dead code; see §16.4).
- `0xCD56 bn.srai r25,r25,11` (÷2048) ; `0xCD5C bn.sub r25,r25,(r23>>31)` (sign-correct rounding).
- `0xCD6F bn.and r25,r25,0x00FFFFFF` (24-bit mask) ; `0xCD76 bn.sw 0(r26),r25` → **`0x86C = ((word@0x2511A8 + SPR(r5,16)) >> 11) − sign, & 0x00FFFFFF`.**

### §16.2 `0x870` — helper `0xCF86`, store at `0xD0A1` (value in `r24`)

- Divisor term: `0xD060 bn.divs r23,0xDD8000,r24` where `r24 = ((SPR(r5,16)>>6) − sign) << 2`, possibly selected by `word@52(r3)≠0` (`0xD057`/`0xD05A`). → `r23 = 0xDD8000 / timer_divisor`.
- Base term (same as `0x86C`): `0xD076` recomputes `state×0xA57EB503` (dead) then `0xD07D bn.add r26,r25,timer` → `0xD086 = (word@0x2511A8 + SPR(r5,16))>>11 − sign`.
- `0xD08C bn.add r24,r23,r26` → sum ; `0xD09E bn.and r24,0x00FFFFFF` ; `0xD0A1 bn.sw 0(r26),r24` → **`0x870 = (0xDD8000 / timer_divisor) + ((word@0x2511A8 + SPR(r5,16)) >> 11) − sign, & 0x00FFFFFF`.**

### §16.3 `0x868` — `0x178B9` (DTS-X SDO packer), store at `0x178B9` (value in `r24`)

- `0x17868 bg.lwz r13,2928(r10)` → a DTS state field ; `0x17874 bg.muli r13,r13,48` ; `0x178B2 bn.and r24,r13,0x00FFFFFF` ; `0x178B9 bn.sw 0(r23),r24` → **`0x868 = (word@(r10+2928) × 48) & 0x00FFFFFF`.** (Matches §11/§12.1.6.)

### §16.4 H1 assessment (from the formulas)

- `0x86C` and `0x870` are **coupled**: both built on the **same base quantity** `(word@0x2511A8 + SPR(r5,16)) >> 11 − sign`; `0x870` adds a constant/divisor term (`0xDD8000 / timer_divisor`) that scales with the timer → sample-rate awareness.
- `0x868` is an independent `×48` field (sample/channel scaling).
- All three are **derived from a DTS state word + a timer SPR + fixed constants** → the *shape* is exactly what SPDIF/IEC61937 burst/pause timing requires. This **reinforces** (does not prove) the "DTS-SDO/IEC61937 timing registers" classification (STRONG INFERENCE → well-corroborated).
- **No static evidence of a wrong constant**: the only curiosity, the `0xA57EB503` multiply at `0xCD4C`/`0xD076`, is **dead code in BOTH** `0x86C` and `0x870` (overwritten by the `state+timer` add) — symmetric between the two registers, so it is *not* a DTS-vs-AC3-specific regression. It is compiler/leftover dead code, not a bug.
- **Value correctness remains UNRESOLVED** (R46-D): confirming the written quotients match the IEC61937 burst-period / pause-interval for the DTS sample rate needs the register datasheet or a runtime capture vs AC3's hardware default (the §15.3 experiment). Even if perfect, the gate (§15) still blocks TX unless the hardware/license state is asserted — so H1 is secondary to H2.

### §16.5 Note

These formulas are now the precise static characterization of what DTS writes to `0x86C`/`0x870`/`0x868`. They are the reference the decisive experiment must compare against AC3's reset defaults and the IEC61937 timing spec. **No patch.**

## §17 — Addendum 7 (R46-FINAL): what `0x13292` actually is, and whether the gate really cuts off TX

This addendum closes the last structural gap: it tests the claim implicit in §14/§15 — *"gate PASS calls `0x13292` = DMA ⇒ gate FAIL means no SPDIF TX"* — by establishing `0x13292`'s real semantics instead of inheriting the label "DMA". Tooling: `r46_work/r46_txchain2.py`, `r46_txchain3.py`, `r46_work/r46_walk.py`, `r46_work/r46_density.py`, `r46_work/r46_probe2.py`. No patch, no device.

### §17.0 Method caveat established first (decoder coverage)

Measured over the whole image: **591,620** instruction starts, **471,745 DEFINED (79.7%)**, **119,875 UNDEFINED (20.3%)**. Consequence: every "no X found" result below means **"no X found among decodable instructions"**, not an absolute proof of absence. This caveat also retro-applies to §15's zero-writer result.

### §17.1 The call target is real (encoding-verified)

`0x25FBB bg.jal` = bytes `E7 FD A5 AE`; op = `0x39`; `simm1_25` = **−77097**; `0x25FBB − 77097 = 0x13292`. The target is computed correctly from the raw bytes → **the gate really does call `0x13292`** (VERIFIED at encoding level).  
**But** `decode_at(0x13292)` is **UNDEFINED** (bytes `8B 26 40 …`; op `0x22`, not implemented). So the *first instruction of the callee* is not decodable by our decoder.

### §17.2 Body extent (bounded by control flow, not by guesswork)

All internal jumps stay inside `0x132C9..0x13691` (`0x1338F→0x132F6`, `0x133B4→0x132D3`, `0x133C3→0x132C9`, `0x134D0→0x134E8`, `0x1357B→0x135A5`, `0x135FE→0x13572`, `0x13654→0x134AE`, `0x1365F→0x13426`, `0x13679→0x13691`) ⇒ **body ≈ `0x13292..0x13691` (~0x400 bytes)**.

### §17.3 Hardware footprint of `0x13292` — NONE provable

- Base-resolved MMIO loads/stores in `0xB000_xxxx`: **ZERO**.
- The only hardware-ish residue is **7 undecoded special ops**: `bg.op2E r13` (`0x13309`), `bg.op2B r26` (`0x13479`, `0x1361C`), `bg.op2F r0` (`0x13587`, `0x135B4`), `bg.op2A r26` (`0x1358B`, `0x135B8`). The authoritative slaspec defines opcodes `0x28–0x2F` only as **empty placeholder constructors** (`bg.opcode_28`…`bg.opcode_2F`), with `0x2E` modelled as a trap-like construct — so **no MMIO semantics exist for them**, and the earlier habit of calling `op2E` "the MMIO-touch op" is **refuted** (see §18.6). §18.1 (Ghidra) shows the surrounding body is ordinary arithmetic.
- ⇒ **NO DMA descriptor / address / length programming is provable. NO SPDIF FIFO / interrupt / trigger access is provable.**
- Calls out: `0x10786E` (`0x13338`, `0x133A7`), `0x1078A9` (`0x134D9`, `0x134F2`, `0x1350E`), `0x10C5B9` (`0x13376`, `0x13680`).
- **Conclusion: `0x13292` must NOT be called "DMA".** (At the time of §17 its semantics were UNRESOLVED because our decoder could not read its first instruction; **§18.1 resolves it via Ghidra ground truth: it is a shared integer arithmetic / structure-store helper** — still no hardware.)

### §17.4 `0x13292` is NOT gate-exclusive (this is the decisive finding)

Whole-image caller scan: **12 call sites from 9 distinct functions** —

| call site                    | enclosing function   |
| ---------------------------- | -------------------- |
| `0xB554`                     | `0xB290`             |
| `0xC083`, `0xC0B0`, `0xC0DB` | `0xBE3A`             |
| `0x14261`                    | `0x14061`            |
| `0x145C6`                    | `0x14527`            |
| `0x160A1`                    | `0x15F53`            |
| `0x1625B`                    | `0x160EB`            |
| `0x16AA3`                    | `0x1680C`            |
| `0x25FBB`                    | `0x25EE1` (the gate) |
| `0x261C2`, `0x2624B`         | `0x25FC3`            |

`0xB290` is the **DTS dispatch function that writes the timing registers** (§13) and is called **only** from `0xC226` (3 call sites, all in `0xC226`; `0xC226` itself has **zero** callers = a task/root entry). The gate's callers are `0xC91A`/`0xE9E5`/`0xED6E`/`0xD2C8` — **no overlap**.  
⇒ **"gate FAIL ⇒ `0x13292` never runs" is FALSE.** Whatever `0x13292` does, it is also invoked on the DTS path from `0xB290` independently of the gate predicate.

### §17.5 Downstream: `0x10786E` and `0x115693`

- `0x10786E`: **1919 call sites from ~100 distinct functions** ⇒ a **generic utility**, not a DTS-specific TX arm. Body: `r3 = 0x228E77`, `r4 = 0x400`, `jal 0x107860`, `jal 0x115693`, epilogue `addi r1,r1,32`.
- `0x115693`: **2 callers** (`0x10786E`, `0x107379`). VERIFIED writes **MMIO `0xB000_003E`** (`0x1157A9`, `0x1157CF`) and **`0xB000_0020`** (`0x1157B1`, `0x1157D7`) via `bn.sh_1`; also `mfspr/mtspr` on **SPR-17 (SR mask ⇒ critical section)** and `sbz 0 → 0x9000_0001`.
- So the literal chain `0x25EE1 → 0x13292 → 0x10786E → 0x115693 → MMIO(0x0020/0x003E)` **does exist**. But its last two links are **generic, firmware-wide utilities**, not a DTS-specific transmitter arm.

### §17.6 Gate return semantics (corrected)

- ~~No `jr`/return decoded within `0x4000` bytes~~ → **RESOLVED by §18.2 (Ghidra): the gate returns.** The missing instruction is `0x25F81 bt.jr r9`, immediately after the epilogue `0x25F7E bn.addi r1,r1,0x20`. Our decoder missed it because the `bt.*` 16-bit class is unimplemented. **The gate is a normal function that returns.**
- The gate contains **loops** (`0x25F8D→0x25F68`, `0x25FBF→0x25F2D`) ⇒ it is a **state machine**, not a single-shot predicate. Combined with §18.2 (it returns) and §18.4 (`0x13292` = arithmetic helper), its profile is a **DTS state/status evaluation routine** that updates SHM state fields and returns a status.
- **Callers do consume a return value**: at all four call sites the instruction immediately after the call tests `r3` — `0xCED0 bg.bnei r3,1158`, `0xD2CC bn.bnei r3,0`, `0xEC86 bg.beqi r3,1554`, `0xEF62 bn.beqi r3,0`. The gate writes `r3` at `0x25F8A` (`cmovsi_2`) and on PASS at `0x25F91` (`sw r0,176(r3)`). ⇒ **The gate returns a status in `r3`; it is not a terminal sink.**

### §17.7 Caller context (all four call sites)

- **Argument is identical everywhere**: `r3 = 0x4D17C0` (`movhi r3,0x4D` + `addi r3,r3,6080`) at `0xCEC4`, `0xD2C0`, `0xEC78`, `0xEF56` — one shared state structure.
- **Before the gate**: `0xCECC` site updates SHM/`0xA86C`/`0xA874`/`0xA878` fields (`0xCE96`–`0xCEAB`); `0xEC82`/`0xEF5E` sites first call the generic `0x10786E`, then the gate.
- **After the gate**: each caller branches on `r3`, then continues with the **same DTS quotient computation** (`0x255CC0`/`0x255CC8`/`0x255CBC` fields, `×48` at `0xCEFB`/`0xD2F6`; `0x252AE8`/`0x252AF0` for the `0xE9E5`/`0xED6E` sites). These post-gate operations are **not** gated by the gate result.
- No TX/DMA action was found anywhere that is *uniquely* reachable only on gate PASS.

### §17.8 Evidence re-grading (strict separation)

**VERIFIED**

1. The gate reads the three predicate inputs (`byte@0xB000_0C06`, `byte@0xB000_0015`, `word@0x2512C4`).
2. `0x25FBB bg.jal` targets `0x13292` (encoding-verified from raw bytes).
3. That call is reached only via the PASS branch (`0x25F2A bnei r11,0 → 0x25F91`); the non-PASS path does not reach it in that iteration.
4. `0xB000_0C06` and `0xB000_0015` have **no writer among decodable instructions** (caveat §17.0).
5. `0x2512C4` has the identified DTS-side writer (`0x172FD`).
6. `0x13292` has **12 call sites from 9 functions**, including `0xB290` (gate-independent).
7. `0x13292`'s body contains **no resolvable MMIO access**; only undecoded `op2A/2B/2E/2F`.
8. `0x10786E` is generic (1919 sites); `0x115693` VERIFIED writes MMIO `0xB000_0020`/`0x003E`.
9. The gate returns a status in `r3` that all four callers branch on.

**NOT YET PROVEN (must not be upgraded)**

1. `0x13292` is the actual SPDIF DMA transfer.
2. `0x13292` directly arms/triggers TX.
3. `0x10786E`/`0x115693` in this path constitute a TX arm (they are generic).
4. Gate FAIL therefore guarantees zero physical SPDIF activity. **(This was overstated in §14/§15 and is hereby withdrawn.)**
5. The runtime value of the predicate is FAIL.
6. The meaning of `op2A/2B/2E/2F`.

### §17.9 Variant selection

- **Variant A** (`0x13292` → actual DMA/TX arm): **not supported** — no MMIO, no descriptor, no trigger provable.
- **Variant B** (`0x13292` → config prep → `0x10786E` → actual TX arm): the *shape* matches, but the "actual TX arm" is a **generic** helper with 1919 callers, so it cannot be the DTS-specific transmitter arm. **Not supported as stated.**
- **Variant C** (`0x13292` → DTS state/configuration, relation to physical TX unresolved): **SELECTED.**
- **Variant D** (cannot reconstruct): rejected — substantial structure *was* reconstructed reliably (predicate, PASS path, call graph, return semantics, finalizer MMIO). The failure is specifically at the "TX arm" link, not in the whole reconstruction.

### §17.10 R46-FINAL classification

**R46-FINAL-C.** `0x13292` is related to DTS state/configuration, but **its relationship to physical TX remains unresolved**, and it is **not gate-exclusive**. The DTS-specific facts that survive are: the predicate inputs are firmware-read-only hardware/license state (§15, caveat §17.0), and the gate is DTS-path-only while AC3 does not traverse it (§14.2). What does **not** survive is the inference *"therefore DTS physically cannot transmit."* H1 (timing value) is restored to **co-equal** status with H2; the two can co-occur.

### §17.11 Robustness check on the decisive claim (added after peer review of §17)

The refutation of gate-exclusivity rests entirely on the 12 call sites of `0x13292`. If the linear walk had decoded *data* as code, some could be spurious. Each site was therefore re-derived **independently from the raw 4 bytes** (`bg.jal`: op `0x39`, `target = addr + sign_extend25((w>>1) & 0x1FFFFFF)`), and its surrounding decode density measured. Tooling: `r46_work/r46_verify_sites.py`.

| site      | raw bytes  | derived target | kind     | density ±0x40 (def/tot) |
| --------- | ---------- | -------------- | -------- | ----------------------- |
| `0xB554`  | `E400FA7C` | `0x13292` ✅    | `bg.jal` | 31/37 — 84%             |
| `0xC083`  | `E400E41E` | `0x13292` ✅    | `bg.jal` | 29/38 — 76%             |
| `0xC0B0`  | `E400E3C4` | `0x13292` ✅    | `bg.jal` | 28/39 — 72%             |
| `0xC0DB`  | `E400E36E` | `0x13292` ✅    | `bg.jal` | 30/39 — 77%             |
| `0x14261` | `E7FFE062` | `0x13292` ✅    | `bg.jal` | 30/40 — 75%             |
| `0x145C6` | `E7FFD998` | `0x13292` ✅    | `bg.jal` | 31/41 — 76%             |
| `0x160A1` | `E7FFA3E2` | `0x13292` ✅    | `bg.jal` | 27/40 — 68%             |
| `0x1625B` | `E7FFA06E` | `0x13292` ✅    | `bg.jal` | 29/37 — 78%             |
| `0x16AA3` | `E7FF8FDE` | `0x13292` ✅    | `bg.jal` | 29/38 — 76%             |
| `0x25FBB` | `E7FDA5AE` | `0x13292` ✅    | `bg.jal` | 27/42 — 64%             |
| `0x261C2` | `E7FDA1A0` | `0x13292` ✅    | `bg.jal` | 28/42 — 67%             |
| `0x2624B` | `E7FDA08E` | `0x13292` ✅    | `bg.jal` | 27/41 — 66%             |

**12 / 12 VERIFIED.** Control: **87%** of `0x80`-blocks image-wide have ≥60% density, so these sites sit in **typical-to-dense code**, not in data. A 22-site sample of the `0x10786E` callers was checked the same way: **22 / 22 verified**, densities 42–92% — the "generic utility" characterisation of `0x10786E` (§17.5) likewise holds up.  
⇒ The §17.4 finding (**`0x13292` is not gate-exclusive**) is **confirmed at encoding level** and is not an artifact of the linear decode.

## §18 — Addendum 8: Ghidra ground-truth cross-check (authoritative slaspec) — closes the decoder gap

§17.0 established that our Python decoder covers only 79.7% of the image, and that `0x13292`'s first instruction was undecodable. This addendum removes that limitation by disassembling the same image with the **authoritative AEON slaspec** in Ghidra headless (`aeon_ghidra_public/aeon/data/languages/aeon.slaspec`; driver `scripts/DumpAEON_R29.java`, `SWEEP:` mode; output `r46_work/ghidra_txchain.txt`). Ground truth resolves three things the Python decoder got wrong.

### §18.1 `0x13292` is real code — a generic arithmetic/state helper (VERIFIED, Ghidra)

The entry decodes cleanly: `0x13292 bt.mov r25,r6`; `0x13297 bn.addi r1,r1,-0x20` (prologue). The body is **pure integer arithmetic plus structure stores**:

- **Remainder idiom:** `0x132C9 bn.divu r27,r11,r26` → `0x132CC bn.muls r26,r27,r26` → `0x132CF bg.bne r11,r26,0x13393` (i.e. `r11 mod r26`).
- **Second modulo with ×48 scaling:** `0x132F6 bg.muli_maybe r13,r13,0x30` (×0x30 = ×48) → `0x132FD bn.divs r23,r13,r24` → `0x13300 bn.muls r24,r23,r24` → `0x13306 bn.sub r13,r13,r24`.
- **Structure stores** to `r10` at offsets `0x0,0x4,0x8,0xc,0x10,0x14,0x18,0x1c,0x20,0x24,0x28,0x2c`, plus `0x132AB bn.sw 0x18(r3),r25`.
- Sets `r3 = 0x1283A7` (`0x1332E bg.movhi r3,0x13` + `0x13332 bg.addi r3,r3,-0x7c59`) and calls `0x10786E` (`0x13338`).
- **NO MMIO load/store. NO DMA descriptor/address/length. NO FIFO/interrupt/trigger.**

⇒ **`0x13292` is a shared arithmetic / state-computation helper.** Together with §17.4 (12 call sites from 9 functions) this **definitively rules out Variant A and Variant B** — the gate's PASS-path call is **not** a TX/DMA arm.

### §18.2 The gate DOES return — §17.6 caveat resolved (VERIFIED)

The Python decoder reported "no `jr` within 0x4000 bytes". Ghidra supplies the missing instruction:

```
0x25F7E  bn.addi r1,r1,0x20      ← epilogue (mirrors the -0x20 prologue)
0x25F81  bt.jr r9                ← RETURN
```

⇒ **The gate returns normally.** §17.6's "may be present-but-undecoded" is now settled in the affirmative. Our decoder missed `bt.jr` because the `bt.*` 16-bit class is unimplemented.

### §18.3 The "`r31` threshold" does not exist — it was a decoder artifact (CORRECTION)

Ghidra renders `0x25F14` as **`bn.sflesi r23,-0x1`** — an **immediate** compare (slaspec: `:bg.sflesi i32_rD, i32_simm5_16`), **not** a register compare against `r31`. Our Python decoder mis-rendered the 5-bit immediate as `r31`.

⇒ **The entire `r31` discussion (§15.2, §17.6, Appendix A `r46_r31.py`) is moot — there is no `r31` operand.** Corrected predicate:  
`r11 = (bit7 of byte@0xB000_0C06) AND (signed byte@0xB000_0015 <= −1) AND (word@0x2512C4 == 0 ⇒ clear r11)`  
i.e. the second term is simply **"`byte@0xB000_0015` is negative (≥ 0x80)"**. No threshold register is involved.

### §18.4 `0x10786E` and `0x115693` (VERIFIED, Ghidra)

- **`0x10786E`** — a thin generic wrapper: prologue; `r10 = 0x230000`; `r3 = 0x228E77`, `r4 = 0x400`, `r6 = r1+0xc`; `bg.jal 0x107860` (`0x107890`); re-set `r3`; `bg.jal 0x115693` (`0x107898`); epilogue `0x1078A0`; `bt.jr r9` `0x1078A3`. Consistent with §17.5's 1919-call-site "generic utility".
- **`0x115693`** — prologue; `bt.mov r13,r3`; `bg.jal 0x10C8EA`; `bg.mfspr r12,r0,0x11` (read SR); `bn.and r23,r12,-7`; `bg.mtspr r0,r23,0x11` (write SR) ⇒ **interrupt-mask critical section**; loads globals `0x2292B4`/`0x2292B8`; fixed-point arithmetic with `0x8000`/`0x7fff`; `bn.sbz 0x0(r24),r0` with `r24 = 0x9000_0001` ⇒ **byte store 0 → `0x9000_0001`**; then `bg.jal 0x1078A9`. Profile = **clock/timer/scheduler helper**, not an audio transmitter. (The earlier base-verified MMIO writes `0xB000_0020`/`0x003E` at `0x1157B1`/`0x1157D7` lie beyond this dump window and stand; they do not change the characterisation.)

### §18.5 Effect on the verdict

- **Variant A** (`0x13292` → actual DMA/TX arm): **refuted** (§18.1 + §17.4).
- **Variant B** (`0x13292` → config prep → `0x10786E` = actual TX arm): **refuted** — both are generic utilities.
- **R46-FINAL-C stands**, refined: `0x13292` is a **shared arithmetic/state helper** used on the DTS path among others.
- Net: the chain `gate PASS → 0x13292 → 0x10786E → 0x115693` is a **computation/utility chain** and provides **no evidence** that gate PASS/FAIL controls physical SPDIF TX. The only DTS-specific facts remain: the gate is DTS-path-only (§14.2) and its predicate inputs have no observed firmware writer (§15). The gate returns a status in `r3` that callers branch on (§17.6) ⇒ it is a **state/status evaluation function**, not a transmitter.

### §18.6 Residual (unchanged) and one new note

Unchanged: semantics of `op2A/2B/2E/2F`; runtime values of `0xB000_0C06` bit7 / `0xB000_0015`; correctness of the `0x86C`/`0x870` quotients.  
New note on `op2E`: the slaspec models op `0x2E` (decoder 5/6/7) as a trap-like construct (`EPCR0 = inst_start; ESR0 = SR; goto 0xe00`), but Ghidra still emits only the placeholder name `bg.opcode_2E`, and in the `0x13292` body it sits inside an integer rounding sequence (`bn.sub` → `bg.opcode_2E` → `bn.sub r13,r0,r13` → `bn.srli …0x1f`) where a trap is implausible. **See §19.6: `op2E` appears 5× in straight-line code as a value-producing op whose result is immediately tested — so the slaspec's trap model is unreliable and `op2E` is a real unmodelled DSP instruction.** The earlier heuristic "`op2E` = MMIO-touch op" is **definitively refuted**.

## §19 — Addendum 9: Ghidra re-decode of the timing writers — §16 validated, corrected, and materially strengthened

§16's formulas were produced by the same Python decoder that rendered `bn.sflesi`'s immediate as `r31` (§18.3), so they required independent validation. Ghidra ground truth (`SWEEP:CC40:180`, `SWEEP:CF40:180`, `SWEEP:17860:B0`, `SWEEP:B340:60`; output `r46_work/ghidra_timing.txt`) confirms the **structure** of all three formulas, corrects the **timer source**, and adds one materially important discovery.

### §19.1 `0x86C` writer (`0xCD76`, helper `0xCC9B`) — CONFIRMED, timer source corrected

```
0xCD39 bg.movhi r10,0x25 ; 0xCD3D bg.addi r10,r10,0x1194   → r10 = 0x251194
0xCD41 bn.lwz r23,0x14(r10)        → r23 = word@0x2511A8
0xCD44/48 r25 = 0xA57EB503 ; 0xCD4C bn.muls r25,r23,r25   (dead)
0xCD4F bg.mfspr1 r26,0x2808        → r26 = SPR 0x2808      ← CORRECTED
0xCD53 bn.add r25,r23,r26          (overwrites the mul → dead code confirmed)
0xCD56 bn.srai r25,r25,0xb         → >> 11   ✓
0xCD59 bn.srai r26,r23,0x1f ; 0xCD5C bn.sub r25,r25,r26   → sign correction ✓
0xCD64 r26 = 0x00FFFFFF ; 0xCD6F bn.and r25,r25,r26       → & 0x00FFFFFF ✓
0xCD68 bg.movhi r24,0xb000 ; 0xCD72 bg.ori r26,r24,0x86c  → 0xB000_086C
0xCD76 bn.sw 0x0(r26),r25          → STORE ✓
```

**Correction:** the timer operand is **`bg.mfspr1 rN,0x2808`** — i.e. **SPR `0x2808`** — not "`SPR(r5,16)`" as §16 wrote. Structure otherwise confirmed: `0x86C = ((word@0x2511A8 + SPR(0x2808)) >> 11) − sign, & 0x00FFFFFF`.

### §19.2 `0x870` writer (`0xD0A1`, helper `0xCF86`) — CONFIRMED **+ sample-rate dispatch discovered**

```
0xCF88 bg.movhi r23,0xb000 ; 0xCF92 bg.ori r23,r23,0x814
0xCF96 bn.lwz r23,0x0(r23)         → read status 0xB000_0814
0xCF99/9C r24 = 0x00FFFFFF ; 0xCFA0 bn.and r23,r23,r24
0xCFA3 bn.lwz r26,0x34(r3) ; 0xCFA6 bg.opcode_2E r24 ; 0xCFAA bt.movi r27,0x1
0xCFAC bg.beq r24,r26,0xD23C
0xCFB0-0xCFF1  DISPATCH ON SAMPLE RATE:
     0x15888 (= 88200) → code 0xe ; 0x1F400 (= 128000) → code 0xd
     0x2B110 (= 176400) → code 0xc ; 0x2EE00 (= 192000) → code 3
     (plus 1 / 2 / 0xe defaults)
0xCFF3 bn.andi r23,r23,0xf ; 0xCFF6 bg.beq r23,r25,0xD19F
0xCFFA/CFE r23 = 0xB000_0854 ; 0xD002 bn.lwz r25,0(r23)
0xD008 r26 = 0x005FFFFF ; 0xD00F bn.sw 0(r23),r25   → WRITE 0xB000_0854
0xD012 bg.ori r24,r24,0x850 ; 0xD016 bn.sw 0(r24),r0 → WRITE 0xB000_0850 = 0
0xD019 bn.lwz r24,0x30(r3) ; 0xD01C/21 r23 = 0xDD8000
0xD03D/40 r27 = 0x10624DD3 ; 0xD044 bn.muls r27,r24,r27
0xD047 bg.mfspr1 r28,0x2808        → SPR 0x2808          ← CORRECTED
0xD04E bn.srai r25,r28,0x6 ; 0xD051 bn.sub r24,r25,r24   → (SPR>>6) − sign
0xD054 bn.slli r25,r24,0x2 ; 0xD05A bn.cmov_0 (select) ; 0xD05D bn.slli r24,r24,0x2
0xD060 bn.divs r23,r23,r24         → r23 = 0xDD8000 / divisor   ✓
0xD063/67 r10 = 0x251194 ; 0xD06B bn.lwz r25,0x14(r10)   → word@0x2511A8 (same base)
0xD06E/72 r26 = 0xA57EB503 ; 0xD076 bn.muls (dead)
0xD079 bg.mfspr1 r27,0x2808 ; 0xD07D bn.add r26,r25,r27  → base term ✓
0xD080 bn.srai r26,r26,0xb ; 0xD086 bn.sub r26,r26,r27   → >>11 − sign ✓
0xD08C bn.add r24,r23,r26 ; 0xD093 bg.movhi r23,0xb000
0xD09A bg.ori r26,r23,0x870 ; 0xD09E bn.and r24,r24,r28
0xD0A1 bn.sw 0x0(r26),r24          → STORE 0xB000_0870 ✓
```

**Corrections:** (a) timer SPR is **`0x2808`**, not `r5,16`; (b) the divisor is `((SPR>>6) − sign)`, then `<< 2`, a conditional select, then **`<< 2` again** (§16 recorded a single `<< 2`).  
**NEW — material:** the helper **dispatches on the sample rate read from `0xB000_0814`**, comparing it against the literal constants **88200 / 128000 / 176400 / 192000** and selecting a rate code. It also writes the shared control registers **`0xB000_0854`** and **`0xB000_0850`**. This is **direct code-level evidence** that the `0x86C`/`0x870` block is **sample-rate-derived SPDIF/IEC61937 timing**, upgrading §12's classification from STRONG INFERENCE to **strongly corroborated**.

### §19.3 `0x868` writer (`0x178B9`) — CONFIRMED, one extra term

```
0x17868 bg.lwz r13,0xb70(r10)      → word@(r10+0xb70)   ✓ (§16's "r10+2928" = 0xb70 ✓)
0x17874 bg.muli_maybe r13,r13,0x30 → × 0x30 = ×48       ✓
0x17880 bt.add r13,r25             → + r25              ← §16 omitted this term
0x178B2 bn.and r24,r13,r27         → & 0x00FFFFFF       ✓
0x178B5 bg.ori r23,r23,0x868 ; 0x178B9 bn.sw 0x0(r23),r24 → STORE ✓
```

**Correction:** `0x868 = ((word@(r10+0xb70) × 48) + r25) & 0x00FFFFFF`.

### §19.4 The `0xB290` dispatch — timing calls are CONDITIONAL (call sites confirmed)

```
0xB344 bg.movhi r23,0xb000 ; 0xB348 bg.ori r23,r23,0x814
0xB34C bn.lwz r11,0x0(r23)         → read 0xB000_0814
0xB356 bn.and r11,r11,r23          → & 0x00FFFFFF
0xB359 bg.opcode_2E r23 ; 0xB35D bn.beqi r23,0x1,0xB36C   → SKIP 0x86C write if r23==1
0xB368 bg.jal 0xCC9B               → 0x86C writer        ✓ (§13 confirmed)
0xB36C bg.opcode_2E r11 ; 0xB370 bn.beqi r11,0x1,0xB37F  → SKIP 0x870 write if r11==1
0xB37B bg.jal 0xCF86               → 0x870 writer        ✓ (§13 confirmed)
```

⇒ §13's call-site addresses are **CONFIRMED by Ghidra**, and the timing writes are **conditional** on state derived from `0xB000_0814` (via the unmodelled `bg.opcode_2E`), not unconditional.

### §19.5 Effect on the hypotheses

- The `0x86C`/`0x870`/`0x868` classification as **sample-rate-derived SPDIF/IEC61937 timing** is now **strongly corroborated** by literal sample-rate constants in the code (88200/128000/176400/192000) — the strongest evidence obtained in R46 for the registers' role.
- **H1 (timing-value) is therefore better-founded in *semantics* than before**, but its *value correctness* remains UNRESOLVED (needs the register datasheet or runtime capture). H1 and H2 remain **co-equal** (§17.10); nothing here changes that.
- §16's `0xA57EB503` "dead multiply" finding is **CONFIRMED** in both writers (Ghidra shows the `bn.muls` immediately overwritten by `bn.add`).

### §19.6 `op2E` — the slaspec's trap model is unreliable

`bg.opcode_2E rX` appears **five times in straight-line code** (`0x13309`, `0xCFA6`, `0xB359`, `0xB36C`, and `0xCFA6`-adjacent), always as a **value-producing op on a register** whose result is immediately tested (`beqi rX,1`) or used arithmetically. That usage is incompatible with the slaspec's `EPCR0=inst_start; ESR0=SR; goto 0xe00` trap model. ⇒ **`op2E` is a real, unmodelled AEON DSP instruction**; the slaspec's semantics for it must not be relied on.

## §20 — Addendum 10: Ghidra re-decode of the AC3 path — differential CONFIRMED, shared quotient found, caller-count CORRECTED

The AC3-vs-DTS differential is the foundation of R44–R46, and it too was established with the Python decoder that §18 showed to be unreliable. Ghidra ground truth (`SWEEP:CB40:120`, `CAA0:90`, `C480:70`, `C5A0:70`, `CB00:40`, plus `C380:180`, `C2F0:140`; output `r46_work/ghidra_ac3.txt`, `ghidra_c380.txt`, `ghidra_c2f0.txt`) confirms the differential, adds a new shared-quotient finding, and corrects a caller count.

### §20.1 The core differential is CONFIRMED (VERIFIED, Ghidra)

MMIO touched by the four AC3 helpers:

| helper          | registers touched                              |
| --------------- | ---------------------------------------------- |
| `0xCB50`        | `0x854` (R/W), `0x5888`-family rate compares   |
| `0xCAB4`        | `0x854` (R/W), `0x850` (write 0)               |
| `0xC49D`        | `0x814` (read), `0x80C` (read), `0x854` (read) |
| `0xC5C5`        | `0x814` (read)                                 |
| `0xC5A8` region | `0x84C` (write 0)                              |

**AC3 never constructs `0x86C`, `0x870` or `0x868`.** DTS writes all three (§19). ⇒ The R44/R45/R46 differential **stands on ground truth**. All five AC3 helpers end with `bt.jr r9` (ordinary returning functions).

### §20.2 NEW — AC3 and DTS compute the *same* timing quotient and both dispatch on sample rate

`0xCB50` (AC3) contains:

```
0xCB50 bg.movhi r23,0x25 ; 0xCB56 bg.lwz r5,0x11a8(r23)   → r5 = word@0x2511A8   ← same state word as DTS
0xCB75/77 r23 = 0x15888 ; 0xCB7B bn.sfeq r4,r23            → 88200
0xCB89 r23 = 0xAC44    ; 0xCB8D bn.sfeq r4,r23            → 44100
0xCB97 r26 = 0xFA00    ; 0xCB9B bn.sfeq r4,r26            → 64000
0xCBA5 bg.sfeqi r4,0x7d00                                 → 32000
0xCBB4/BB8 r23 = 0xA57EB503 ; 0xCBBC bn.muls (dead)
0xCBBF bg.mfspr1 r24,0x2808                               → SPR 0x2808   ← same timer SPR as DTS
0xCBC3 bn.add r6,r5,r24 ; 0xCBC9 bn.srai r6,r6,0xb        → >> 11
0xCBD2 bn.sub r6,r6,r23                                   → sign
```

So AC3 computes **`((word@0x2511A8 + SPR(0x2808)) >> 11) − sign`** — **byte-for-byte the same formula** as the DTS `0x86C` writer (§19.1) — and dispatches on the sample rates **88200 / 44100 / 64000 / 32000**; the DTS `0x870` writer dispatches on **88200 / 128000 / 176400 / 192000** (§19.2).

⇒ Both paths derive the *same* sample-rate/timer quotient; the DTS-vs-AC3 difference is **which register receives it** (`0x86C`/`0x870`/`0x868` vs `0x854`/`0x850`/`0x84C`). This is a **new, concrete data point for H1**: the arithmetic is shared and therefore likely well-formed, shifting H1's residual risk away from "wrong formula" toward "wrong destination register / rate handling".

### §20.3 CORRECTION — `0xB290` has more callers than the report claimed

Ghidra shows `bg.jal 0xB290` at **`0xC30D`**, **`0xC46F`** and **`0xC48B`**, inside coherent code (loops at `0xC303`/`0xC33C`/`0xC442`, conditional branches selecting the call) spanning roughly `0xC2F0–0xC499` — **in addition to** the three sites in `0xC226` reported in §13/§17.

My Python caller scan (§17 pass 2/3, corrected parser) reported only the `0xC226` sites. ⇒ **§13's statement "`func 0xB290` is called by `func 0xC226` only" is WRONG** — it is an artifact of the Python linear walk, which drifts out of phase and therefore **misses call sites**.

- **Corrected:** `0xB290` has **at least 6 call sites** (3 in `0xC226` + `0xC30D`, `0xC46F`, `0xC48B`).
- **The core §13 conclusion survives**: it rested on (a) the gate body containing no call to the timing writers, and (b) `0xB290` not calling the gate while the gate does not call `0xB290` — both still hold. Only the *exclusivity detail* was wrong.
- **§17.4's counts** (12 call sites / 9 functions for `0x13292`) must likewise be read as **lower bounds**.

### §20.4 Methodology lesson (third decoder-induced defect in R46)

The Python linear walk (i) leaves 20.3% undecoded and (ii) **drifts out of phase**, so it **undercounts call sites**. Every "X is called only by Y" / "N call sites" claim from that walk is a **lower bound** and must be re-checked with Ghidra before being used as a load-bearing argument. This is the third such defect found in R46, after the `r31` immediate (§18.3) and the `op2E` heuristic (§19.6).

### §20.5 Net effect on the verdict

- **Differential confirmed** ⇒ **R46-B** (register question) unchanged.
- **New support for H1's arithmetic**: AC3 and DTS compute the same quotient with the same timer SPR ⇒ the formula is likely correct; the DTS-specific variable is the **destination register / rate dispatch**.
- **§13's independence conclusion stands**, but its stated justification required the §20.3 correction.
- **H1 and H2 remain co-equal** (§17.10); nothing here changes the gate finding.

## Appendices

**A. Tooling (reusable)**

- `r46_work/r46_gatectx.py` — dumps alignment-correct instructions (from full-image index `S.idx`) for the gate entry `0x25EE1` and the timing-writer callers `0x265FD`/`0x266E0`, settling gate-vs-timing ordering.
- `r46_work/r46_order.py` — whole-image caller map (target→caller-functions) for the gate, timing writers, finalizer, and DMA; used to prove no caller/callee edge between the timing path (`0xB290`) and the gate (`0x25EE1`).
- `r46_work/r46_order3.py` — region-bounded call-target scan (no fragile function-exit heuristic); VERIFIED the gate body `0x25EE1..0x25F91` calls none of the timing writers/finalizer/DMA, and that `0xB290` writes both timing registers and calls DMA+finalizer in one DTS frame function.
- `r46_work/r46_gatecallers.py` — alignment-correct decode of the full gate body `0x25EE1..0x26000` (proves the gate calls `0x13292` on the DTS-license/state predicate `r11`, and only on PASS) plus the four gate-caller sites (proves AC3 helpers do not call the gate). **Does not** prove `0x13292` is a DMA/TX arm — see §17.
- `r46_work/r46_predwriters.py` — full-image writer trace of the gate's predicate inputs (`0xB000_0C06`, `0xB000_0015`, `0x2512C4`) + the `0xB000_0C00–0xCFF` status block and `0x251000–0x251FFF` state region. Proves the two hardware/license inputs are firmware-read-only → the gate is firmware-independent.
- `r46_work/r46_r31.py` — **MOOT (§18.3).** It investigated an `r31` threshold at `0x25F14`; Ghidra ground truth shows that instruction is `bn.sflesi r23,-0x1`, an **immediate** compare. There is no `r31` operand, so the script's question no longer exists. Kept only as a record of the (decoder-induced) false lead.
- `r46_work/r46_opcodes.py` — **authoritative-ISA cross-check.** Reads the Ghidra slaspec (`aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc`) to establish that opcodes `0x28–0x2F` are placeholder constructors, and histograms opcode/decoder-field pairs over the image. Refuted the "`op2E` = MMIO touch" heuristic.
- `r46_work/r46_verify_sites.py` — re-derives every claimed call site independently from the raw bytes and reports decode density (§17.11).
- `scripts/DumpAEON_R29.java` (existing) — Ghidra headless dumper; run with `SWEEP:START:LEN` for ground-truth disassembly of a region. Outputs for this phase: `r46_work/ghidra_txchain.txt` (§18), `r46_work/ghidra_timing.txt` (§19), `r46_work/ghidra_ac3.txt` + `ghidra_c380.txt` + `ghidra_c2f0.txt` (§20). Recipe in the `aeon-snd-static-trace` skill §1.
- `r46_work/r46_txchain2.py` — **R46-FINAL pass 2 (corrected parsers).** Fixes two real bugs in earlier tooling: `parse_tgt` must ignore the `(r9=0x…)` display suffix on `jal` operands (previous versions silently reported `0x-1` and returned **empty** caller sets), and `find_func_entry` must require a **negative** `addi r1,r1,-N` (it previously matched `+N` epilogues). Produces the authoritative caller sets, the `0x13292` body extent, and its hardware footprint.
- `r46_work/r46_txchain3.py` — **R46-FINAL pass 3.** Caller-context windows for the 4 gate call sites, the independence check (`0xB290` vs gate), and the identification of the real finalizer MMIO targets `0xB000_0020`/`0x003E` inside `0x115693`.
- `r46_work/r46_walk.py` — independent decode walk from an arbitrary entry (does not rely on the global linear walk's phase), used to test whether a call target is a real instruction boundary.
- `r46_work/r46_density.py` — **methodological control**: measures decoder coverage (79.7% defined / 20.3% undefined) per region. Every "no X found" claim in R46 must be read against this number.
- `r46_work/r46_probe2.py` — raw `decode_at` probe at every address in a window (used to verify the `jal` target encoding by hand).
- `r46_work/r46_timingvals.py` — full alignment-correct decode of the three DTS timing-writer helpers (`0xCC9B`, `0xCF86`, `0x178B9`) to extract the exact value formulas for `0x86C`/`0x870`/`0x868` (§16).
- `r46_work/r46_scan.py` — full-image linear disassembler + backward register resolver; flags effective addresses, base-verified. Run: `python3 r46_scan.py` (also `probe <addr>...` mode).
- `r46_work/r46_neighborhood.py` — maps any MMIO range (here `0xB000_0800–0x08FF`), base-verified, grouped by offset.
- `r46_work/r46_ownclues.py` — binary string + workspace grep for ownership hints.
- Reuses `scripts/aeon_decode.py` (STATIC-verified AEON decoder).

**B. Key constants re-confirmed (R45 lesson, applied)**

- SHM base `0xA000` = `movhi 0x1 ; addi r,−0x6000` → offset region `0xA868`/`0xA86C`/`0xA870` (DSP-internal, read-back, cache-flush-published).
- MMIO base `0xB000_0000` = `movhi 0xb000 ; ori 0x8XX` → `0xB000_08XX` (hardware output block, write-only for `0x86C`/`0x870`).
- Same offset, different region — classified by **base register + cache-flush-arg corroboration**, never by offset alone.

**C. Verdict label (FINAL):** **R46-B** for the register question (write-only, DTS-path-exclusive MMIO timing registers `0x86C`/`0x870`) **+ R46-FINAL-C** for the root-cause question:

> `0x13292` is related to DTS state/configuration, **but its relationship to physical TX remains unresolved**, and it is **not gate-exclusive** (12 call sites / 9 functions, incl. the gate-independent `0xB290`). Its body has **no MMIO / DMA-descriptor / FIFO / trigger access** (§18.1, Ghidra ground truth). Its downstream helpers `0x10786E` (1919 call sites) and `0x115693` are **generic utilities**, not a DTS transmitter arm.
>
> **Variant A and Variant B are refuted** (§18.5): `0x13292` is a shared integer **arithmetic/structure-store helper**, and `0x10786E` is a generic wrapper. The chain `gate PASS → 0x13292 → 0x10786E → 0x115693` is a **computation/utility chain** and gives **no evidence** that gate PASS/FAIL controls physical SPDIF TX.

Consequences for the hypotheses: **H1 (timing value) and H2 (gate) are co-equal**, not ranked. The earlier claim *"gate FAIL ⇒ no physical SPDIF activity"* is **withdrawn as unproven** (§17.8). Surviving DTS-specific facts: the gate is DTS-path-only (AC3 does not traverse it); its predicate inputs have no observed firmware writer; it returns a status in `r3` consumed by all 4 callers.

**No patch.** Remaining unknowns: (i) semantics of `op2A/2B/2E/2F` and of `0x13292`; (ii) runtime values of `0xB000_0C06` bit7 / `0xB000_0015`; (iii) correctness of the `0x86C`/`0x870` timing quotients. All require a datasheet or a runtime capture — both blocked (R43 device-access stop).






















































