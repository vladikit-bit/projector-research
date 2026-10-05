# R44 — AC3-vs-DTS Hardware-Output Differential (pure static analysis)

**Prior:** R29–R43. Per the STOP instruction, the following branches are **CLOSED** and NOT revisited:
`/dev/malloc` mmap, MIU probes, `A868/A86C` runtime observation, `DspMadBase`, `DM[]` reads, SHM-reader search.
**This phase:** static tracing of the hardware-facing output registers touched by the AC3 vs DTS helpers, to
locate the **first AC3-vs-DTS divergence in hardware-facing state**. No runtime, no patching, no device change.

**Evidence base:** `r29_work/decoded_all6.txt` (contains AC3 helpers `0xCB50`, `0xCAB4` and DTS helpers
`0xCC9B`, `0xCF86`, plus `0xC49D`, `0xC5C5`). Register addresses are reconstructed **only** where the base register
is verifiably established as `movhi rX,0xb000` followed by `ori rX,rX,0x8XX`. Low-offset-only matches are **rejected**.

---

## 1. AC3 output trace

```
input → AC3 helper(s)
        0xCB50, 0xCAB4 (also 0xC49D, 0xC5C5 in the same AC3 tool set)
        ↓
hardware register constructions (base verified movhi 0xb000 + ori 0x8XX)
        ↓
common finalization (0x10786E → 0x115693)
```

### AC3 hardware-register constructions (base-verified)

| Helper | PC | Instruction | Register |
|--------|----|-------------|----------|
| `0xCB50` | `0xCB67` / `0xCB6B` | `movhi r23,0xb000` → `ori r23,r23,0x854` | **`0xB000_0854`** |
| `0xCB50` | `0xCBF0` / `0xCBF7` | `movhi r23,0xb000` → `ori r23,r23,0x854` | **`0xB000_0854`** |
| `0xCB50` | `0xCC08` / `0xCC0C` | `movhi r24,0xb000` → `ori r23,r24,0x854` | **`0xB000_0854`** |
| `0xCB50` | `0xCC20` | `ori r24,r24,0x84c` (r24=0xb000) | **`0xB000_084C`** |
| `0xCAB4` | `0xCAC6` / `0xCACA` | `movhi r24,0xb000` → `ori r23,r24,0x854` | **`0xB000_0854`** |
| `0xCAB4` | `0xCADE` | `ori r24,r24,0x850` | **`0xB000_0850`** |
| `0xCAB4` | `0xCAFF` / `0xCB03` | `movhi r24,0xb000` → `ori r24,r24,0x854` | **`0xB000_0854`** |
| `0xC49D` | `0xC49D`→`0xC4A1` | `movhi r23,0xb000` → `ori r24,r23,0x814` | **`0xB000_0814`** |
| `0xC49D` | `0xC4AE` | `ori r23,r23,0x80c` | **`0xB000_080C`** |
| `0xC49D` | `0xC4D8`→`0xC4DC` | `movhi r23,0xb000` → `ori r23,r23,0x854` | **`0xB000_0854`** |
| `0xC49D` | `0xC4F0` | `ori r24,r23,0x854` | **`0xB000_0854`** |
| `0xC49D` | `0xC504` | `ori r23,r23,0x84c` | **`0xB000_084C`** |
| `0xC5C5` | `0xC5C5`→`0xC5C9` | `movhi r23,0xb000` → `ori r23,r23,0x814` | **`0xB000_0814`** |
| `0xC5C5` | `0xC631`→`0xC635` | `movhi r24,0xb000` → `ori r23,r24,0x854` | **`0xB000_0854`** |
| `0xC5C5` | `0xC649` | `ori r24,r24,0x850` | **`0xB000_0850`** |
| `0xC5C5` | `0xC696`→`0xC69A` | `movhi r24,0xb000` → `ori r24,r24,0x810` | **`0xB000_0810`** |
| `0xC5C5` | `0xC6C4`→`0xC6C8` | `movhi r23,0xb000` → `ori r23,r23,0x854` | **`0xB000_0854`** |

**AC3 register set (base-verified):** `0x854`, `0x850`, `0x84C`, `0x814`, `0x80C`, `0x810`.
**AC3 does NOT construct:** `0x86C`, `0x868`, `0x870`, `0x818`, `0x81C` — none of these appear in the AC3 helpers.

---

## 2. DTS output trace

```
input → DTS helper(s)
        0xCC9B, 0xCF86
        ↓
DTS-private gate/state (0x25EE1) + output prep (0x13292)  [established R32/R37]
        ↓
hardware register constructions (base verified)
        ↓
common finalization (0x10786E → 0x115693)
```

### DTS hardware-register constructions / stores (base-verified)

| Helper | PC | Instruction | Register |
|--------|----|-------------|----------|
| `0xCC9B` | `0xCC9D`→`0xCCA5` | `movhi r23,0xb000` → `ori r24,r23,0x814` | **`0xB000_0814`** |
| `0xCC9B` | `0xCCB2` | `ori r23,r23,0x80c` | **`0xB000_080C`** |
| `0xCC9B` | `0xCCDA`→`0xCCE1` | `movhi r24,0xb000` → `ori r24,r24,0x854` | **`0xB000_0854`** |
| `0xCC9B` | `0xCD04` | `ori r24,r23,0x854` | **`0xB000_0854`** |
| `0xCC9B` | `0xCD18` | `ori r23,r23,0x84c` | **`0xB000_084C`** |
| `0xCC9B` | `0xCD68`→`0xCD72` | `movhi r24,0xb000` → `ori r26,r24,0x86c` | **`0xB000_086C`** ← DTS-SPECIFIC |
| `0xCC9B` | `0xCD79` | `ori r25,r24,0x814` | `0xB000_0814` |
| `0xCC9B` | `0xCD87` | `ori r24,r24,0x818` | **`0xB000_0818`** |
| `0xCC9B` | `0xCE26` | `bg.sw_0 0x86c(r23),r24` | **STORE → 0x86C** ← DTS-SPECIFIC STORE |
| `0xCC9B` | `0xCE72` | `bg.sw_0 0x86c(r23),r24` | **STORE → 0x86C** ← DTS-SPECIFIC STORE |
| `0xCC9B` | `0xCF48` | `bg.sw_0 0x86c(r23),r24` | **STORE → 0x86C** ← DTS-SPECIFIC STORE |
| `0xCF86` | `0xCF88`→`0xCF92` | `movhi r23,0xb000` → `ori r23,r23,0x814` | `0xB000_0814` |
| `0xCF86` | `0xCFFA`→`0xCFFE` | `movhi r24,0xb000` → `ori r23,r24,0x854` | `0xB000_0854` |
| `0xCF86` | `0xD012` | `ori r24,r24,0x850` | `0xB000_0850` |
| `0xCF86` | `0xD093`→`0xD09A` | `movhi r23,0xb000` → `ori r26,r23,0x870` | **`0xB000_0870`** ← DTS-SPECIFIC |
| `0xCF86` | `0xD0A4` | `ori r27,r23,0x814` | `0xB000_0814` |
| `0xCF86` | `0xD0B2` | `ori r24,r23,0x81c` | **`0xB000_081C`** |
| `0xCF86` | `0xD0B9` | `ori r28,r23,0x818` | `0xB000_0818` |
| `0xCF86` | `0xD15E` | `bg.sw_0 0x868(r23),r24` | **STORE → 0x868** ← DTS-SPECIFIC STORE |
| `0xCF86` | `0xD1A3`→`0xD1A7` | `movhi r25,0xb000` → `ori r25,r25,0x810` | `0xB000_0810` |
| `0xCF86` | `0xD1CF`→`0xD1D6` | `movhi r25,0xb000` → `ori r25,r25,0x854` | `0xB000_0854` |

**DTS register set (base-verified):** `0x854`, `0x850`, `0x84C`, `0x814`, `0x80C`, `0x810`, `0x818`, `0x81C`,
**plus `0x86C` (built + stored ×3) and `0x868` / `0x870` (built/stored in `0xCF86`).**

---

## 3. Register differential table

| Register | AC3 (`0xCB50/0xCAB4/0xC49D/0xC5C5`) | DTS (`0xCC9B/0xCF86`) | Common? |
|----------|------|-------|---------|
| `0xB000_0854` | **YES** (`0xCB6B,0xCBF7,0xCC0C,0xCACA,0xCB03,0xC4DC,0xC4F0,0xC635,0xC6C8`) | **YES** (`0xCCE1,0xCD04,0xCFFE,0xD1D6`) | **COMMON** |
| `0xB000_0850` | **YES** (`0xCADE,0xC649`) | **YES** (`0xD012`) | **COMMON** |
| `0xB000_084C` | **YES** (`0xCC20,0xC504`) | **YES** (`0xCD18`) | **COMMON** |
| `0xB000_0814` | **YES** (`0xC4A1,0xC5C9`) | **YES** (`0xCCA5,0xCD79,0xCF92,0xD0A4`) | **COMMON** |
| `0xB000_080C` | **YES** (`0xC4AE`) | **YES** (`0xCCB2`) | **COMMON** |
| `0xB000_0810` | **YES** (`0xC69A`) | **YES** (`0xD1A7`) | **COMMON** |
| `0xB000_0818` | — | **YES** (`0xCD87,0xD0B9`) | DTS-only (in this set) |
| `0xB000_081C` | — | **YES** (`0xD0B2`) | DTS-only |
| **`0xB000_086C`** | **NO** | **YES** — built `0xCD72`, **stored 3×** (`0xCE26,0xCE72,0xCF48`) | **DTS-SPECIFIC** |
| **`0xB000_0868`** | **NO** | **YES** — **stored** `0xD15E` | **DTS-SPECIFIC** |
| **`0xB000_0870`** | **NO** | **YES** — built `0xD09A` | **DTS-SPECIFIC** |

**The important result is not "DTS writes more registers" — it is that the DTS helpers additionally program the
`0x86C / 0x868 / 0x870` block, while the AC3 helpers do not construct those addresses at all.**

---

## 4. First divergence

```
FIRST VERIFIED AC3/DTS HARDWARE DIVERGENCE:
  DTS helper 0xCC9B programs 0xB000_086C (address built at 0xCD72 via movhi r24,0xb000 + ori r26,r24,0x86c),
  storing to it three times (0xCE26, 0xCE72, 0xCF48: bg.sw_0 0x86c(r23),r24).
  DTS helper 0xCF86 additionally builds 0xB000_0870 (0xD09A: ori r26,r23,0x870) and stores 0xB000_0868 (0xD15E:
  bg.sw_0 0x868(r23),r24).
  Neither 0x86C, 0x868 nor 0x870 is constructed by any AC3 helper (0xCB50, 0xCAB4, 0xC49D, 0xC5C5).
  Both codecs share the 0x854 / 0x850 / 0x84C / 0x814 / 0x80C / 0x810 block.
```

This is a **hardware-facing configuration difference**: DTS programs an extra register block that AC3 does not.

---

## 5. SDO / IEC61937 relation

**NOT PROVEN.** I found **no static call/branch evidence** linking the `0x86C/0x868/0x870` writes to the
`DTSX_CORE2_API_SDO_Packer` / `DTSDecSDOPacker_API_Process` / `Mstar_DTS_Hdmi_Packer` functions. The relationship
between the DTS packer output and these registers is therefore **UNRESOLVED** — no dispatch table, function
pointer, or shared-structure link was established in this pass. I do **not** claim that `0x86C/0x868/0x870`
"control SDO / non-PCM / TX" — the addresses alone do not prove that, per the R44 rule.

---

## 6. Evidence classification

### VERIFIED
- Base-verified (`movhi rX,0xb000` + `ori …0x8XX`) register constructions for AC3 and DTS helpers, with exact PCs (tables §1–§3).
- `0x854`, `0x850`, `0x84C`, `0x814`, `0x80C`, `0x810` are constructed by **both** AC3 and DTS helpers → **common block**.
- `0x86C` is built (`0xCD72`) and stored **3×** (`0xCE26`, `0xCE72`, `0xCF48`) in DTS `0xCC9B`.
- `0x868` is stored (`0xD15E`) in DTS `0xCF86`; `0x870` is built (`0xD09A`) in DTS `0xCF86`.
- **No AC3 helper (`0xCB50`, `0xCAB4`, `0xC49D`, `0xC5C5`) constructs `0x86C` / `0x868` / `0x870`.**

### STRONG INFERENCE
- The DTS output path performs **additional hardware-output configuration** (the `0x86C/0x868/0x870` block) that the
  working AC3 path does not. This is the first hardware-facing divergence between the two codec paths.
- Because AC3 (known-working) never programs this block, the divergence is a **credible next place to investigate**
  as the DTS blocker — but it is **not yet proven to be causal**.

### UNRESOLVED
- **Exact values** written to `0x86C / 0x868 / 0x870` (value-register provenance was not fully traced; the store
  value registers were not resolved to constants in this pass).
- Whether `0x86C/0x868/0x870` gate **SDO / non-PCM / IEC61937 / TX** behaviour (not proven; no function linked).
- Whether the SDO/IEC61937 packer functions relate to these writes (no static link found).
- Whether other (unexamined) AC3 functions write this block (only the four AC3 helpers above were traced).
- Whether the divergence is **causal for the DTS failure** (needs a controlled experiment, not more static work).

---

## 7. Patch decision

**Do NOT create a patch.** No fix is justified by the current evidence: the divergence is identified, but its
value semantics and causal role are unproven.

### Future minimal experiment (if approved, and NOT a firmware patch)
The only variable worth testing is the **DTS-specific hardware register block**. A minimal experiment would:
1. Observe/compare the `0x86C / 0x868 / 0x870` register contents during **working AC3** vs **failing DTS**
   (this requires a **safe** read path — note the previously used `/dev/malloc` mmap is FORBIDDEN; a
   kernel-sanctioned readback, or a safe register-peek facility, would be required).
2. Determine whether DTS leaves `0x86C/0x868/0x870` unprogrammed or programs them with a different value.
3. Only if a concrete, plausible deficiency is confirmed, consider a targeted, reversible change to the DTS
   programming of that block — still **not** a speculative patch.

**Critical constraint:** this experiment must **not** use `/dev/malloc` mmap, MIU probes, `DM[]`, or any
unsafe memory access. If no safe readback facility exists, the experiment **cannot proceed** and the honest
conclusion remains: *the AC3/DTS hardware divergence is identified but its causal role is unresolved.*

---

## Summary

> At the hardware-facing output stage, DTS **first diverges** from the working AC3 path by additionally
> programming the **`0xB000_086C` (0xCC9B) and `0xB000_0868` / `0xB000_0870` (0xCF86)** block, which AC3 never
> touches. Both codecs share the common `0x854 / 0x850 / 0x84C / 0x814 / 0x80C / 0x810` configuration.
> The **semantics and causal role** of the DTS-specific block are **UNRESOLVED** — this is the next credible
> place to look for the DTS blocker, but it is **not yet proven** to be the cause.
