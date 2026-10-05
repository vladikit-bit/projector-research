# R35 — Symbol-Anchored Search for the Reader of SND R2 SHM `+0x868` / `+0x86C`

**Prior:** R33-B (write-only in `snd_full.bin`) · R34-C (window = SND R2 shared memory; consumer not recovered)
**R34 baseline taken as established:** `g_virSndR2shm = MsOS_PA2KSEG1(HAL_AUDIO_GetDspMadBaseAddr() + 0x7000A000)`;
`0xCF86 → A868`, `0xCC9B → A86C`, then `0xD4C8` cache maintenance.
**This phase:** pure static. No patch, no device change, no runtime experiment, no EDID/ARC/topology/settings/AUTH change.
**Method:** **symbol-anchored** (ELF `.symtab` + relocation parsing), *not* literal byte search.
**Deliverable date:** 2026-09-09

---

## Headline classification

> ### R35 = **R35-D** — **No reader of SND R2 SHM `+0x868`/`+0x86C` was found in the complete currently
> ### available evidence set.**

Using the real symbol `g_virSndR2shm` as anchor, every host function that touches the SND R2 shared memory was
enumerated **exhaustively**, and **none** of them accesses offset `0x868` or `0x86C`. Whole-`.text` scans of the
kernel modules confirm this independently.

**Coverage actually achieved (all modules named in the R35 scope were examined):**
`utpa2k.ko` (unstripped, symbol-anchored) · **`mik.ko`** (unstripped) · `dtv_driver.ko` · `libutopia.so` ×2 ·
`libmi3*.so` · `audio.primary.mt5889.so` · `libaudioparser.so` · `mwgifker.ko` · `xcker.ko` — plus, from R34,
`mst_codec_r2*.bin` and `armfw.bin`. **Nothing in the workspace was left unsearched.**

---

# 1. VERIFIED

### V1 — The SHM symbols live in `utpa2k.ko`, not `libutopia.so`
| Binary | `g_virSndR2shm` | `g_virDecR2shm` | `.symtab` | Notes |
|--------|-----------------|-----------------|-----------|-------|
| `utpa2k.ko` (original, unpatched) | **present** | **present** | **yes (unstripped)** | the implementation |
| **`mik.ko`** (`spdif_audio_investigation/kmods/`) | **absent** | **absent** | **yes (unstripped)** | imports `MDrv_AUDIO_GetDspMadBaseAddr` / `MApi_AUDIO_GetDspMadBaseAddr` (NOTYPE, external); **no** `R2shm`/`SndR2` symbols |
| `ghidra_proj_new/libutopia.so` / `ghidra_proj_sdo/libutopia.so` | absent | absent | **no (stripped)** | only `HAL_AUDIO_GetDspMadBaseAddr` exported `@0x395DF4` |
| `dtv_driver.ko` | absent | absent | — | `DspMad=0`, `PA2KSEG1=0`, `SHM_PARAM=0` → not a candidate |

### V2 — Exact SHM symbol addresses (from `utpa2k.ko` `.symtab`)
```
g_virDecR2shm  .bss  0xEFBF4   size 4
g_virSndR2shm  .bss  0xEFBFC   size 4
```

### V3 — Every load site enumerated via relocation (R_ARM_MOVW_ABS_NC `0x2B` / R_ARM_MOVT_ABS `0x2C`)
19 relocation sites in `.rel.text` (10 SND, 9 DEC). Mapping each to its containing **FUNC** symbol:

**SND R2 SHM host users (all A32, `thumb=False`):**
| # | Function | func@vaddr | size | `g_virSndR2shm` load @ |
|---|----------|-----------|-----|------------------------|
| 1 | `HAL_AUR2_BackupShareMemory` | `0x457E04` | `0x250` | `0x457EA8` |
| 2 | `HAL_AUR2_RestoreShareMemory` | `0x458054` | `0x254` | `0x4580E4` |
| 3 | `HAL_SND_R2_init_SHM_param` | `0x4582A8` | `0x190` | `0x4582AC` |
| 4 | `HAL_SND_R2_Get_SHM_INFO` | `0x458E24` | `0x280` | `0x458E2C` |
| 5 | `HAL_SND_R2_Get_SIGNED_SHM_INFO` | `0x4590A4` | `0x128` | `0x4590AC` |
| 6 | `HAL_SND_R2_Set_SHM_COMMOM_PARAM` | `0x459A44` | `0x28C` | `0x459A4C` |
| 7 | `HAL_SND_R2_Get_SHM_COMMOM_PARAM` | `0x459CD0` | `0x12C` | `0x459CD4` |
| 8 | `HAL_SND_R2_Set_SHM_PARAM` | `0x459DFC` | `0x638` | `0x459E04` |
| 9 | `HAL_SND_R2_Get_SHM_PARAM` | `0x45B3D8` | `0x3EC` | `0x45B3E0` |
| 10 | `HAL_SND_R2_Get_SHM_INFO_64bit` | `0x45C284` | `0x100` | `0x45C288` |

DEC equivalents: `HAL_AUR2_Backup/RestoreShareMemory`, `HAL_DEC_R2_init_SHM_param`, `HAL_DEC_R2_SetCommInfo2`,
`HAL_DEC_R2_Get_SHM_INFO2`, `HAL_DEC_R2_Get_SHM_INFO`, `HAL_DEC_R2_Set_SHM_PARAM`, `HAL_DEC_R2_Get_SHM_PARAM`,
`HAL_DEC_R2_Get_SHM_INFO_64bit`.

### V4 — **None** of the 10 SND SHM functions accesses `+0x868` or `+0x86C`
Per-function scan for A32 `LDR`/`STR` with `imm12 ∈ {0x868, 0x86C}`:
```
HAL_AUR2_BackupShareMemory          hits=0
HAL_AUR2_RestoreShareMemory         hits=0
HAL_SND_R2_init_SHM_param           hits=0
HAL_SND_R2_Get_SHM_INFO             hits=0
HAL_SND_R2_Get_SIGNED_SHM_INFO      hits=0
HAL_SND_R2_Set_SHM_COMMOM_PARAM     hits=0
HAL_SND_R2_Get_SHM_COMMOM_PARAM     hits=0
HAL_SND_R2_Set_SHM_PARAM            hits=0
HAL_SND_R2_Get_SHM_PARAM            hits=0
HAL_SND_R2_Get_SHM_INFO_64bit       hits=0
```
Same result for all 9 DEC SHM functions (hits=0).

### V5 — Whole-`.text` scan of `utpa2k.ko` (`.text` size `0x6287DC`)
| imm12 | hits | detail |
|-------|------|--------|
| `0x868` | **0** | — |
| `0x86C` | **1** | `@0x3D6584`, word `0xE5D2386C` = `LDRB r3, [r2, #0x86c]`, in function **`mfeSetVopType`** (`@0x3D65D0`, size 588) |

`mfeSetVopType` is a **VOP (Video Output Processor) type** function, ~0x8A000 bytes away from the SHM functions
(`0x457E04+`), and its base is `r2`, which is **not** loaded from `g_virSndR2shm` (no `0x2B/0x2C` relocation for
that symbol anywhere near it). **Confirmed false positive — rejected per Phase B rules.**

### V6 — `0x7000A000` is built with `movw`/`movt`, not stored as a literal
Search for the LE literal `00 A0 00 70` in `libutopia.so`: **0 occurrences**. This is why a literal byte search
cannot work and the symbol/relocation anchor was required.

### V7 — All mapping paths are covered
Enumerated every relocation call site (`R_ARM_CALL 0x1C` / `R_ARM_JUMP24 0x1D`) to
`HAL_AUDIO_GetDspMadBaseAddr` / `MDrv_AUDIO_GetDspMadBaseAddr` / `_MDrv_AUDIO_GetDspMadBaseAddr`.
The complete set of callers that also load `g_virSndR2shm` is exactly the 10 functions in V3. Other callers
(`HAL_MAD_SetAudioParam2`, `HAL_MAD_GetAudioInfo2`, `HAL_MAD_GetCommInfo`, `HAL_MAD2_SetMemInfo`,
`HAL_AUDIO_COPY_Parameter`, `HAL_ADVSOUND_*`, `MDrv_AUDIO_Dump_*_Monitor`, `HAL_AUDSP_DspLoadCode`,
`HAL_SND_R2_Check_Meminfo`, `HAL_DEC_R2_Check_Meminfo`) do **not** load `g_virSndR2shm` — they use a different
base (MAD base), and per V5 none uses immediate `0x868`/`0x86C` anywhere in `.text`.

### V8 — `mik.ko` analysed: **not** the consumer (corrects an earlier "absent" assumption)
`mik.ko` **is present** at `spdif_audio_investigation/kmods/mik.ko` (8,084,248 bytes, ELF32 ARM, **unstripped**).
- Symbols matching `shm|Mad|Dsp|R2`: only `_eHdmiUseDspId`, `_bIsR2Audio`, `_bHwSupportDNR2VE` (local data) plus
  **undefined imports** `MDrv_AUDIO_SetDspBaseAddr`, `MDrv_AUDIO_GetDspMadBaseAddr`,
  `MApi_AUDIO_GetDspMadBaseAddr`. **No `g_virSndR2shm` / `R2shm` / `SndR2` symbol.**
- `.text` scan for A32 `LDR`/`STR` with `imm12 ∈ {0x868, 0x86C}`: **0 hits for both.**

⇒ `mik.ko` maps DSP memory via `PA2KSEG1` but never addresses `+0x868`/`+0x86C`. **VERIFIED — rejected as consumer.**

*(Also checked: `mwgifker.ko`, `xcker.ko` (video; `PA2KSEG1` only, no audio SHM), all `libmi3*.so`
(`PA2KSEG1=1`, no SHM symbols), `audio.primary.mt5889.so` and `libaudioparser.so` (no `DspMad`/`SHM_PARAM`).)*

---

# 2. STRONG INFERENCE

- **The host CPU is NOT the consumer.** The only host code with the SND R2 SHM pointer is the 10 HAL functions;
  none reads `+0x868`/`+0x86C`, and no other host code in the module encodes those offsets.
- **Combined with R33/R34:** the consumer is not the SND R2 firmware (write-only), not the codec R2
  (accesses MMIO `0xB000_0868/086C`, not the SND SHM), and not the host CPU. Therefore it lies **outside all
  images available in this workspace**.
- The SHM region *around* `0x868` is real host-visible memory (the HAL maps and uses the window for the
  `0x104–0x194` param block), so `+0x868` is most plausibly a field used by an agent not present here — but this
  is **not** asserted as a finding.

# 3. UNRESOLVED

- Identity and class of the consumer of `+0x868` / `+0x86C`.
- Whether those slots are read at all after publication.
- The SND R2 SHM layout/field definitions at offset `0x868`.
- Whether `0x868` and `0x86C` form a pair/descriptor or are independent.
- Any connection to DTS SDO / IEC61937 / packer / SPDIF-HDMI TX.
- **Not** established as DMA/MMIO/hardware — no evidence for any of those labels.

# 4. Reader table

No reader found. Table lists the verified SHM users and the rejected candidate:

| Address | Binary | Base | Effective address | Access | Width | Function | Confidence |
|---------|--------|------|-------------------|--------|-------|----------|------------|
| — | `utpa2k.ko` | `g_virSndR2shm` (`0xEFBFC`) | SHM+`0x868` | **none found** | — | — | — |
| — | `utpa2k.ko` | `g_virSndR2shm` (`0xEFBFC`) | SHM+`0x86C` | **none found** | — | — | — |
| `0x459E04` | `utpa2k.ko` | `g_virSndR2shm` | SHM base | load ptr | 4 | `HAL_SND_R2_Set_SHM_PARAM` | VERIFIED (offsets `0x104–0x194` only) |
| `0x45B3E0` | `utpa2k.ko` | `g_virSndR2shm` | SHM base | load ptr | 4 | `HAL_SND_R2_Get_SHM_PARAM` | VERIFIED (no `0x868/0x86C`) |
| `0x3D6584` | `utpa2k.ko` | `r2` (**not** SHM) | unknown | `LDRB` | 1 | **`mfeSetVopType`** | **REJECTED** — wrong base, unrelated video function |

# 5. Critical chain

```text
0xCF86  →  compute value  →  STORE [g_virSndR2shm + 0x868]  →  0xD4C8 flush+invalidate+sync
                                                              ↓
                                              [ FIRST VERIFIED READER ]   ← NOT FOUND
                                                              ↓  unknown
                                              [ next stage ]              ← unknown
                                                              ↓
                                              [ DTS TX ? ]                ← unknown
```
```text
0xCC9B  →  compute value  →  STORE [g_virSndR2shm + 0x86C]  →  0xD4C8 flush+invalidate+sync
                                                              ↓
                                              [ FIRST VERIFIED READER ]   ← NOT FOUND
                                                              ↓
                                              [ DTS TX ? ]                ← unknown
```
Unknown links are **left empty** — not filled with assumptions.

# 6. Consumer class

```text
UNKNOWN
```
Evidence: not HOST (V4/V5/V7 — no host access to those offsets); not the SND R2 DSP (write-only, R33);
not the codec R2 (R34 — MMIO `0xB000_0868/086C`, not SND SHM); **not proven HARDWARE/DMA** (no engine,
descriptor, or register evidence — therefore the term DMA is deliberately **not** used).

# 7. What artifact is still missing (specific)

1. ~~`mik.ko` — absent~~ → **RESOLVED in this pass**: `mik.ko` *is* present and was analysed (V8) — no SHM
   symbol, 0 hits for `0x868`/`0x86C`. **No module in the workspace is missing from the search.**
   What remains absent is any image/component *outside* this workspace that shares the SND R2 SHM.
2. **SND R2 SHM layout header / struct definition** covering offset `0x868` (vendor header, `SND_R2_SHM` /
   `SHM_PARAM` definitions). No `.h`/`.c`/`.cpp` with `868`/`86C`/`SND_R2_SHM` exists in the workspace (searched
   `.h .c .cpp .asm .md .txt` under `spdif_audio_investigation/` — only the HAL asm fragments matched, and they
   document the `0x104–0x194` block only).
3. **R2 linker map / symbol table for `snd_full.bin`** — would name `0xA868`/`0xA86C` and thus the intended
   consumer.
4. **Any additional DSP/core image that maps the same window** — `armfw.bin` was checked and shows no `0xA000`
   mapping (0 candidates).
5. **Hardware/TRM documentation** for the memory shared with the R2 (to establish whether an engine reads it).

---

## Appendix — Method (reusable, symbol-anchored)
1. Parse ELF32 section headers of the **unstripped** module (`utpa2k.ko`).
2. Read `.symtab`/`.strtab` → locate `g_virSndR2shm` (`.bss 0xEFBFC`) and `g_virDecR2shm` (`.bss 0xEFBF4`).
3. Scan all `.rel.*` sections for `r_info>>8` == those symbol indices → every code site that materialises the
   pointer (types `0x2B` `R_ARM_MOVW_ABS_NC`, `0x2C` `R_ARM_MOVT_ABS`).
4. Map each site to its enclosing `STT_FUNC` symbol → the complete host accessor layer.
5. Within each function's byte range, scan 4-byte words for A32 `LDR`/`STR` immediate:
   `(w & 0x0E000000) == 0x04000000 && (w & 0xFFF) ∈ {0x868, 0x86C}`; extract `Rn` (`(w>>16)&0xF`) to verify the
   base is the SHM pointer.
6. Cross-check with a whole-`.text` scan and attribute every hit to its owning function before accepting or
   rejecting it (this is what rejected `mfeSetVopType`).
