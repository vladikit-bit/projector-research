# REPORT — R46d — `0xB000_0854`: AC3-vs-DTS bit differential (narrow static check)

**Subject:** Thundeal TD98 Pro / MStar MT5889 (Android 11) — AEON (R2) audio DSP
**Method:** **Ghidra ground truth only.** No new full-image survey. Sources: the existing MMIO inventory `r46_work/ghidra_writers.txt` (106 stores / 281 loads) plus targeted dumps of the known `0x854` consumers (`r46_work/ghidra_854.txt`), and the already-produced helper dumps (`ghidra_formulas.txt`, `ghidra_ac3.txt`).
**Constraints honoured:** no patch, no device access, no runtime experiment, no re-opening of CheckHashkey / AUTH / EDID / gate `0x25EE1` / `0x13292` / the `0x868-0x86C-0x870` arithmetic / Python-decoder issues.
**Versioning:** new report. R46c is otherwise CLOSED; this is a single targeted sanity check on `0x854`.

**Coverage caveat:** the MMIO inventory does **not** contain the `0xCC9B` PATH-B accesses (`0xCD08`/`0xCD15`) — its base register `r23` is clobbered earlier in that helper, so the tracker could not resolve it. Those two sites are taken from the direct dump (`ghidra_formulas.txt`) and are marked "†" below.

---

## D1 — Exact AC3 vs DTS bit diff for `0xB000_0854`

### D1.1 All twelve writers (Ghidra-verified masks)

| # | site | operation on `0x854` | bits cleared | bits set |
|---|---|---|---|---|
| 1 | `0xC559` | `old & 0xF7FFFF` | 19 | — |
| 2 | `0xC573` | `old & 0xF5FFFF` | 17, 19 | — |
| 3 | `0xC646` | `old & 0x5FFFFF` | 21, 23 | — |
| 4 | `0xC72C` | `old & 0x7FFFFF` | 23 | — |
| 5 | `0xC746` | `old & 0x5FFFFF` | 21, 23 | — |
| 6 | `0xC82D` | `(old & 0xEFF0FF) \| r24 \| r23` | 8,9,10,11, 20 | rate code → bits **8–11** |
| 7 | `0xCADB` | `old & 0xFFFF` | 16–31 | — |
| 8 | `0xCB1D` | `old & 0xFFFF` | 16–31 | — |
| 9 | **`0xCBFB` (AC3 `0xCB50`)** | `(old & 0xFFF) \| 0xA0044 \| code` | **12–23** | **2, 6, 17, 19** + rate code → bits **12–14** |
| 10 | `0xCC1D` | `old & 0xFFFF` | 16–31 | — |
| 11 | **`0xD00F` (DTS `0xCF86`)** | `old & 0x5FFFFF` | **21, 23** | — |
| 12 † | **`0xCD15` (DTS `0xCC9B` PATH B)** | `old & 0xF5FFFF` | **17, 19** | — |

Mask arithmetic (all verified): `0xF7FFFF = ~0x80000` (bit 19); `0xF5FFFF = ~0xA0000` (bits 17,19); `0x5FFFFF = ~0xA00000` (bits 21,23); `0x7FFFFF = ~0x800000` (bit 23); `0xEFF0FF = ~0x100F00` (bits 8–11, bit 20). `0xA0044` = bits 2, 6, 17, 19.

### D1.2 What AC3 leaves in `0x854`

`(old & 0xFFF) | 0xA0044 | code` fully determines bits 12–23:

```
bits  0–11  = preserved from the previous value
bits 12–14  = rate code  (0x1000=1, 0x2000=2, 0x3000=3, 0x4000=4, 0x5000=5  → a 3-bit value 1..5)
bits 15–16  = 0
bit  17     = 1        ← set
bit  18     = 0
bit  19     = 1        ← set
bits 20–23  = 0        ← bits 21,23 are cleared
```

### D1.3 What DTS leaves in `0x854`

- `0xCC9B` PATH B: `& 0xF5FFFF` → **bits 17 and 19 cleared**; nothing set.
- `0xCF86`: `& 0x5FFFFF` → **bits 21 and 23 cleared**; nothing set.

### D1.4 The exact difference (the answer to D1)

| bit(s) | after AC3 | after DTS | difference? |
|---|---|---|---|
| 2, 6 | **1** | preserved (not touched) | AC3 sets them; DTS leaves them |
| 8–11 | preserved | preserved | none |
| 12–14 | **rate code 1..5** | preserved (not touched) | AC3 writes the field; DTS does not |
| 15, 16, 18, 20 | 0 | preserved | AC3 forces 0 |
| **17, 19** | **1** | **0** (explicitly cleared) | **YES — the only bit pair AC3 sets and DTS actively clears** |
| 21, 23 | 0 | 0 | none (both clear) |

⇒ **The one functional difference is bits `17|19`.** (`0xA0044`'s other bits — 2 and 6 — are set by AC3 but never *cleared* by DTS, so they are not a reliable discriminator.)

---

## D2 — Relevant downstream consumers of `0x854`

Of the 18 readers, **exactly two are functional**; the rest are telemetry.

### D2.1 Predicate A — tests bits `17|19` (VERIFIED)
```
0xC4D8/DC  r23 = 0xB000_0854
0xC4E0 bn.lwz r23,0x0(r23)      → r23 = 0x854
0xC4E3 bt.movhi r26,0xa         → r26 = 0xA0000
0xC4E5 bn.and r23,r23,r26       → isolate bits 17|19
0xC4E8 bn.sfne r23,r26          → F = ((v & 0xA0000) != 0xA0000)
0xC4EB bn.cmovsi_2 r3,r0,0x1    → return boolean in r3
0xC4EE bt.jr r9                 → leaf, returns
```
**This is a getter that returns whether bits 17|19 are both set.** It is a leaf function; its callers act on the boolean.

### D2.2 Predicate B — tests bits `21|23` (VERIFIED)
```
0xC6C4/CAF70854  r23 = 0xB000_0854
0xC6CC bn.lwz r23,0x0(r23)      → r23 = 0x854
0xC6CF bt.movhi r26,0xa0        → r26 = 0xA00000
0xC6D2 bn.and r23,r23,r26       → isolate bits 21|23
0xC6D5 bn.sfne r23,r26
0xC6D8 bn.cmovsi_2 r3,r0,0x1
0xC6DB bt.jr r9                 → leaf, returns
```

**⇒ Each DTS clearing mask has a matching predicate:** `0xF5FFFF` (bits 17|19) ↔ Predicate A; `0x5FFFFF` (bits 21|23) ↔ Predicate B. That symmetry is the strongest structural signal in this check.

### D2.3 Telemetry readers (no control consequence)
`0xE404`, `0xE4E9`, `0xE5F7` all read `0x854`, mask it with `0x00FFFFFF`, and call `bg.jal 0x10786E` — the generic logging wrapper (R46b §21.3). `0xE5F7` is part of the same straight-line dump loop (`0xE60C–0xE702`) that sweeps `0x858…0x87C`. **These are diagnostics, not consumers.**

### D2.4 Other readers
`0xD1DA` reads `0x854` inside the `0xCF86` helper region but the immediately following branch (`0xD1DD bg.bnei r24,0x0,…`) tests `r24`, not the loaded value. No branch on the `0x854` value was found there.

### D2.5 Caveat on "DTS-specific"
The masks that clear these bits are **not exclusive to DTS**: `0xF5FFFF` is also used at `0xC573`, and `0x5FFFFF` at `0xC646` and `0xC746` — both in the shared `0xC4xx–0xC8xx` config function. So the *clearing* of bits 17|19 and 21|23 is a **shared behaviour** in which the DTS helpers participate; only the **setting** of bits 17|19 (via `0xA0044`) was observed exclusively on the AC3 path.

---

## D3 — Semantic classification of each changed field

| field | classification | basis (not proximity) |
|---|---|---|
| **bits 8–11** | **sample-rate code (4-bit)** — STRONG INFERENCE | `0xC82D` writes it from a rate dispatch comparing against **88200 / 128000 / 176400 / 192000**, producing values **1,2,3,4,5,12,13,14** — the *same value set* the DTS `0x870` writer produces and compares against `word@0xB000_0814 & 0xF` (R46c §19.2). Value-set match, not address proximity. |
| **bits 12–14** | **second rate/format code (3-bit, values 1..5)** — STRONG INFERENCE | written only by AC3's dispatch, whose cases are 88200/44100/64000/32000 + default → 1..5 |
| **bits 2, 6** | **UNKNOWN flags** | set only by AC3 (`0xA0044`); never cleared by DTS; no reader isolates them |
| **bits 17\|19** | **a 2-bit field with a real firmware consumer; semantics UNKNOWN** | set by AC3 (`0xA0044`), cleared by `0xC573` and DTS `0xCD15`, **read and tested by Predicate A**. Could be a mode/valid/enable pair — **cannot be proven to be non-PCM / SDO / output-enable from the firmware** |
| **bits 21\|23** | **a 2-bit field with a real firmware consumer; semantics UNKNOWN** | cleared by `0xC646`, `0xC746`, DTS `0xD00F`; **read and tested by Predicate B**. No setter observed |
| **bit 20** | **UNKNOWN** | cleared by `0xC82D` only |
| **bits 0–7, 15, 16, 18, 22, 24–31** | **UNKNOWN / preserved** | no isolated read or write observed beyond the above |

**No bit is classified as non-PCM, SDO, output-enable, mute, or decoder-selection.** Those labels would require the register datasheet; nothing in the firmware distinguishes them.

---

## D4 — Is `0x854` a credible first patch target?

### D4.1 Against the specific hypothesis ("DTS-specific output-enable / non-PCM / SDO bit") — **downgraded**
- The clearing masks used by the DTS helpers are **shared** with non-DTS routines (§D2.5), so the DTS helpers are not doing anything exclusive to DTS in `0x854`.
- No bit in `0x854` can be shown to mean non-PCM / SDO / enable / mute / decoder-select.
- ⇒ **The "DTS-specific output-enable bit" lead is NOT supported.** This is a **Case B** outcome for that specific hypothesis, with the qualifier below.

### D4.2 But `0x854` **is** a materially better experimental target than `0x868`/`0x86C`/`0x870`
Three concrete, verified reasons:
1. **It has functional consumers.** Predicates A and B read `0x854` and return booleans to callers. The three timing registers have **no functional reader at all** (R46b §21.3) — a change there cannot be observed in firmware behaviour.
2. **It has an exact, opposing AC3-vs-DTS difference**: AC3 sets bits `17|19` to 1; DTS clears them to 0. That is a single, precisely-specified 2-bit state difference — far easier to reason about than a 24-bit quotient.
3. **Its field layout is partly decoded** (rate code in bits 8–11, second code in 12–14), so a change can be attributed to a named field rather than to an opaque value.

### D4.3 Recommendation on patching
**Do not patch `0x854` yet.** Bits `17|19` are *functionally consumed*, which cuts both ways: changing them will change firmware behaviour in a way whose meaning is unknown, and the register is shared with the AC3 path — so a mistake there can break the **working** AC3 case. The correct next step is **observation**, not modification (see D5).

**Classification of the whole lead: Case C** — the *structure* (a 2-bit field set by AC3, cleared by DTS, read by a predicate) is exactly the shape of a mode/enable flag, but the **semantics cannot be reconstructed statically**, and the clearing is not DTS-exclusive.

---

## D5 — One recommended runtime experiment (read-only)

**Objective:** determine, at runtime, whether bits `17|19` of `0x854` actually differ between AC3 and DTS passthrough — i.e. whether the structural difference in D1.4 is realised.

**Procedure (read-only, using the already-proven safe mechanism):**
1. On real Kodi, play an **AC3** passthrough file; sample the DSP-SRAM mirror cell for `0x0854` — `echo read_dsp_sram_type=1 addr=0x0854 len=1 > /proc/utopia_mdb/audio`, then `dmesg | grep 'DM\['` (the R30 mechanism; `DM[0x0854]` = low 16 bits of the register image).
2. Repeat with a **DTS** passthrough file.
3. Compare the two captures, specifically bits 17 and 19 of the captured cell.

**Decision value:**
- **If the captured values differ in bits 17|19** → the structural difference is real at runtime, and `0x854` bits 17|19 becomes the **primary R47 lead**.
- **If both capture `0x000000`** → either the mirror does not expose this register (consistent with the earlier note that MMIO **writes** are not mirrored — see the `0x854`/`DM[0x0854] = 0` observation during AC3), **or** Predicates A/B are reading 0 at runtime (which would itself be a significant finding about the config path).
- **Either outcome is informative**, which is why this is the right first experiment.

**Explicit caution:** this experiment must be **read-only**. Do **not** attempt a write to `0x854` in R47 until the read-back question above is settled — writing a shared control register whose bit semantics are unknown risks breaking AC3, the one path that currently works.

---

## D6 — Summary

1. **Exact diff:** AC3 sets bits `2, 6, 17, 19` and a 3-bit rate code in bits `12–14`, and forces bits `20–23` to 0. DTS clears bits `17|19` (`0xCC9B`) and `21|23` (`0xCF86`), and sets nothing. **The only functional difference is bits `17|19`.**
2. **Consumers:** two leaf predicates (`0xC4E0`, `0xC6CC`) test `0x854 & 0xA0000` and `0x854 & 0xA00000` respectively and return booleans. All other readers are telemetry.
3. **Semantics:** bits 8–11 = rate code (STRONG, by value-set match); bits 12–14 = second rate code; bits 17|19 and 21|23 = **functionally-consumed 2-bit fields of UNKNOWN meaning**; everything else UNKNOWN.
4. **Credibility as a first target:** the *specific* "DTS-specific output-enable" hypothesis is **downgraded (Case B)**; but `0x854` is still a **better observation target** than the timing registers because its bits have real firmware consumers. **Do not patch it.**
5. **Recommended experiment:** read-only AC3-vs-DTS capture of `DM[0x0854]` (D5).

**No patch. No device modification. No new full-image survey. Static analysis stops here.**

---

## D7 — Artifacts

| Artifact | Content |
|---|---|
| `r46_work/ghidra_writers.txt` | (existing) MMIO inventory — the 29 `0x854` accesses and all masks |
| `r46_work/ghidra_854.txt` | (new) targeted dump of the `0x854` consumer chains: `0xC4C0–0xC780`, `0xC7A0–0xC840`, `0xD1C0–0xD200`, `0xE3F0–0xE620` |
| `r46_work/ghidra_formulas.txt` | (existing) the `0xCC9B` PATH-B body, incl. the missed `0xCD04`–`0xCD15` accesses |
| `r46_work/ghidra_ac3.txt` | (existing) the AC3 `0xCB50` writer incl. `0xCBFB` |
