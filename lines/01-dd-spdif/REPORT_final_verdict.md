# SPDIF TX Path Investigation - Final Technical Report

## Executive Summary

**DTS/AC3 IEC61937 → SPDIF passthrough without licensed DTS decoder: NOT POSSIBLE on this MStar architecture.**

After exhaustive investigation spanning kernel modules, HAL libraries, userspace libraries, and runtime verification, the conclusion is definitive:

**Compressed DTS/AC3 bitstream cannot reach SPDIF TX without an active licensed DTS decoder program running on the MStar DSP.**

---

## Key Findings

### 1. Architecture Confirmation
- **SPDIF TX is not a standalone data pipe** - it's a tap on the DSP decoder output matrix
- SPDIF non-PCM mode is **exclusively activated** by a successfully running decode program (AC3/DTS)
- No register, ioctl, or API can enable SPDIF non-PCM mode independently of a running decode program

### 2. Kodi IEC61937 Path Analysis
```
Kodi 22 (Piers/Maven) IEC packer → AudioTrack(ENCODING_IEC61937)
    → AudioFlinger → AudioPolicy → HAL (audio.primary.mt5889.so)
    → libmi3.so (MI_AUDIO_*) → Utopia (libutopia.so) → MStar DSP
```

**Finding**: Kodi IEC61937 path reaches DSP and starts AC3 decoder (Codec:5), but:
- DSP AC3 decoder expects **raw AC3 frames** (0x0B77 sync word)
- Kodi sends **IEC61937 frames** (Pa/Pb/Pc/Pd headers + AC3 payload)
- AC3 decoder **fails to synchronize** on IEC61937 framing → no audio output

### Three Paths Investigated

| Path | Mechanism | Result |
|---|---|---|
| **Native AC3 RAW** (Android IEC packer) | Android IEC packer → HAL → DSP AC3 decoder → SPDIF | ✅ **WORKS** - Dolby Digital confirmed |
| **Kodi IEC61937** | Kodi packs IEC61937 → AudioTrack(ENCODING_IEC61937) → HAL → DSP | **Silence** - AC3 decoder fails to sync on IEC61937 framing |
| **DTS Native** | Blocked by DTS license (MDrv_AUDIO_GetDTSLicense fails) | N/A |
| **DTS→AC3 Transcode (Kodi)** | Kodi software DTS decode → AC3 encode → AC3 passthrough | ✅ **WORKS** - Dolby Digital output confirmed |
| **DTS IEC61937** | Same as AC3 IEC - decoder fails to sync on IEC framing | Silence |
| **PCM-carrier IEC** | IEC61937 bursts in PCM16 carrier → MStar PCM path | Silence/Noise - receiver sees PCM, not IEC61937 |

## Root Cause: Architecture

```
AudioTrack → AudioFlinger → AudioPolicy → HAL (audio.primary.mt5889.so)
    → libmi3.so (MI_AUDIO_*) → Utopia (libutopia.so) → MStar DSP Firmware
                                        ↓
                              DSP Core
                                      ↓
                              ADEC0 (Audio Decoder 0)
                                      ↓
                              Decode Program Running?
                              ├── YES (AC3/DTS decode successful)
                              │       → DSP output matrix routes compressed stream to SPDIF TX
                              │       → SPDIF TX: non-PCM mode, IEC61937 framing, correct CS
                              └── NO (decode fail / no decoder)
                                      → Silence / PCM output
```

**Critical Finding**: SPDIF TX non-PCM mode is **NOT a standalone hardware path**. It's a **tap on the decoder output matrix** that only activates when a decode program successfully synchronizes and decodes.

### What Was Proven

| Path | Status | Evidence |
|---|---|---|
| Native AC3 RAW → DD | ✅ CONFIRMED | Baseline verified post-reboot |
| DTS Native | ❌ | License blocked at `MDrv_AUDIO_GetDTSLicense` |
| Kodi IEC61937 AC3 | ❌ | AC3 decoder fails to sync on IEC61937 framing |
| DTS IEC61937 | ❌ | Same as AC3 - decoder fails on IEC61937 framing |
| DTS→AC3 transcode | ✅ WORKS | DTS→FFmpeg decode→AC3 encode→AC3 passthrough → DD |
| PCM-carrier IEC (Probe6) | ❌ | PCM path doesn't pass IEC61937 to receiver |

### Key Technical Findings

1. **No compressed → SPDIF path exists without active decoder**
   - `MI_AUDIO_Start(CodecType=5/9)` is the ONLY trigger for SPDIF non-PCM mode
   - `HAL_AUDIO_SPDIF_*` functions are mode/configuration only, NOT data injection
   - `MI_AOUT_SetDigitalMode` = mode selector, NOT data injection
   - No `SPDIF_TX_Write()` or equivalent exists in any library

2. **DTSDecSDOPacker** functions exist in libutopia but:
   - Called ONLY from native DTV file player path
   - Pack **decoded PCM** into IEC61937 for HDMI/SPDIF output
   - `DTS_SDO_SPDIF_OUT` = HDMI/SPDIF output of **decoded PCM**, not compressed pass-through
   - No caller path from Android AudioTrack/HAL to these functions

3. **DTS Decoder License** blocks native DTS at `MDrv_AUDIO_GetDTSLicense`
   - `MDrv_AUDIO_GetDTSLicense` returns failure
   - DTS decoder never initializes → `MI_AUDIO_Start(codecType=9)` fails

## Final Classification

| Path | Status | Verdict |
|---|---|---|
| Native AC3 RAW → DD | ✅ **WORKING** | Verified post-reboot |
| Native DTS RAW | ❌ | License blocked |
| Kodi IEC61937 AC3 | ❌ Silence | IEC framing breaks AC3 decoder sync |
| Kodi IEC61937 DTS | ❌ | Same as above |
| DTS → AC3 Transcode | ✅ **WORKING** | Software decode → AC3 encode → passthrough |
| PCM-carrier IEC | ❌ | MStar PCM path doesn't preserve IEC framing |
| Direct SPDIF TX injection | ❌ **IMPOSSIBLE** | No API, no DMA path, no register |

---

## FINAL VERDICT

> **COMPRESSED DTS/AC3 BITSTREAM → SPDIF TX WITHOUT LICENSED DTS DECODER = IMPOSSIBLE ON THIS ARCHITECTURE**

The MStar DSP architecture **fundamentally requires** an active, licensed decode program to route compressed audio to SPDIF TX. There is no hardware bypass, no register write, no DMA path, no register write that can inject IEC61937 frames directly into SPDIF TX.

**Only viable path for DTS content on this device:**
```
DTS Content → Kodi Software Decode (FFmpeg) → AC3 Encode → AC3 Passthrough → Dolby Digital on Receiver
```

This is **already working** and requires no further investigation.

---

## Final Artifacts

| Report | Path |
|---|---|
| HAL Gate 1 Experiment | `REPORT_hal_gate1_experiment.md` |
| Gate 2 / Format Mapping | `REPORT_hal_gate1_experiment.md` |
| libmi3 Capability Patch | `REPORT_kodi22_minimal_patch.md` |
| HAL IEC Patch v3 | `REPORT_hal_iec_patch_prepared.md` |
| Phase 5 SPDIF TX Architecture | `REPORT_phase5_spdif_tx_research.md` |
| **Final Verdict** | **`REPORT_final_verdict.md`** (this file) |

---

## Final Statement

> **No compressed DTS/AC3 bitstream can reach SPDIF TX on this MStar platform without a licensed, running DTS/AC3 decoder program on the DSP. The architecture fundamentally couples SPDIF non-PCM output to successful decoder synchronization.**

**The only working DTS path on this device: DTS → AC3 transcode (software) → AC3 passthrough → Dolby Digital.** This is already working and requires no further action.