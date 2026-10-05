# R4 — DTS Post-Parser Transmit-Path Differential Trace

**Date:** 2026-09-05
**Device:** Thundeal TD98 Pro / C50A — MediaTek MT5889, Android 11, kernel 4.19.116
**Search boundary:** `DTS parser / Parser_Write → mi_common_raw_write → MI_AUDIO / MI_PCM / DSP transport → MAD R2 / DTS processing → DTS SDO / IEC61937 packing → SPDIF / HDMI digital TX → physical output`
**Verdict:** **R4-B (qualified)** — DTS is **not observed to enter** the non-PCM / IEC61937 SPDIF transmit path. The qualification matters: the SPDIF output-mode selection is **not instrumented** in the captured logs, so this is *"no evidence of reaching"*, **not** *"positive proof of not reaching"*.

---

## 0. Constraint compliance

| Constraint | Status |
|---|---|
| No binary patching | **Honored.** All analysis static; no module/library written to device. |
| No Binder-transaction probing | **Honored.** |
| No SettingsProvider / `sound_spdif_type` probing | **Honored.** |
| No EDID / ARC / HDMI topology / routing / property change | **Honored.** EDID binaries were only *listed*; nothing written. |
| No `dumpsys` inside the playback-only capture windows | **Honored.** R4c windows are playback-only. |
| State words `0x440 / 0x444 / 0x4d0 / 0x4d8 / 0x582` not patched | **Honored.** Discussed read-only (§5). |

---

## 1. Objective 1 — the `mi_common_raw_write` call chain

`mi_common_raw_write` — HAL `audio.primary.mt5889.so` @ **0x031164** (Thumb-2, 0x2b0 bytes).

Callees (resolved via PLT):

| PLT | Target | Notes |
|---|---|---|
| `0x3d4e0` | `utils_isParserSupported` | HAL-internal @ **0x28342** — **format-dependent** (see §2) |
| `0x3df70` | `MI_AUDIO_GetAttr` | args `(handle, 0x104, &buf, &size)` |
| `0x3e240` | `MI_AUDIO_Write` | the actual compressed-frame handoff |
| `0x3e1a0` | `utils_common_out_open` | lazy output open |
| `0x3d8a0` | `utils_write_data_dump` | debug dump (the R3 "noise" source) |
| `0x3db60` | `systemTime` | timestamps |

**Finding:** the body of `mi_common_raw_write` contains **no AC3-vs-DTS branch**. It is format-agnostic; the only format-sensitive decision it makes is delegated to `utils_isParserSupported`.

Downstream, `MI_AUDIO_Write` in `libmi3.so` (**0x62afd**, 0x190 bytes) is likewise **format-agnostic** — it builds an AOUT config struct and issues ioctl **`0xC038100A`** (`movw r1,#0x100a; movt r1,#0xc038`), branching only on debug log level.

> **Conclusion (obj. 1):** the compressed-frame transport from parser output to the kernel ioctl is **identical code for AC3 and DTS**. No divergence exists here.

---

## 2. The one format-dependent gate before the kernel — and DTS *passes* it

`utils_isParserSupported` (HAL @ **0x28342**, Thumb) reads the format's high byte from `mstar_stream_out` and returns 0/1:

```
ldrb.w r0, [r0, #0x183]        ; format high byte
lsls   r1, r0, #0x18
movs   r0, #1                  ; default = supported
cmp.w  r1, #0xa000000 / bge  ; >= E-AC3
cmp.w  r1, #0x6000000 / bge
cmp.w  r1, #0x4000000 / beq  -> ret 1
cmp.w  r1, #0x5000000 / bne  -> ret 0
...
cmp.w  r1, #0xb000000          ; DTS
it ne ; movne r0, #0           ; equal -> r0 stays 1
cmp.w  r1, #0x9000000 / beq  -> ret 1   ; AC3
cmp.w  r1, #0xc000000 / beq  -> ret 1   ; DTS-HD
cmp.w  r1, #0x22000000 / beq -> ret 1
movs   r0, #0 ; bx lr                   ; otherwise unsupported
```

Reconstructed supported set: **0x04, 0x05, 0x06, 0x09 (AC3), 0x0A (E-AC3), 0x0B (DTS), 0x0C (DTS-HD), 0x22**.

| Format | Constant | Result |
|---|---|---|
| AC3 | `0x09000000` | **1 (supported)** |
| E-AC3 | `0x0A000000` | 1 |
| **DTS** | **`0x0B000000`** | **1 (supported)** |
| DTS-HD | `0x0C000000` | 1 |
| IEC61937 | `0x0D000000` | **0 (not supported)** |

> **This closes the gate question:** `utils_isParserSupported` is format-dependent, but **DTS explicitly passes it**. It is *not* the DTS blocker. (Notably `AUDIO_FORMAT_IEC61937` is *not* parser-supported — the stack feeds AC3/DTS raw formats, not IEC61937, to the parser.)

This is the **only** format-dependent branch found anywhere between the parser output and the kernel ioctl.

---

## 3. Objective 2 — SDO / IEC61937 / non-PCM references and their callers

### 3.1 Where the code actually lives

**Important ISA correction (invalidates any earlier Thumb-based reading of this module):**
`utpa2k.ko` is **ARM (A32)**, not Thumb. Verified: symbol values are even, and ARM-mode disassembly of `HAL_AUDIO_SPDIF_SetMode` @0x448a10 yields a valid prologue (`push {r4,r5,r6,r7,fp,lr}`, `movw`/`movt`), while Thumb-mode yields garbage (0 valid branches per 16 KB vs 385 in ARM mode). `utpa2k.ko` is also **not stripped** for audio.

### 3.2 Kernel-side (`utpa2k.ko`, ARM) — the real SDO / non-PCM layer

| Address | Symbol |
|---|---|
| `0x44428c` | `HAL_AUDIO_SPDIF_Tx_SetNonPCM` |
| `0x444010` | `HAL_AUDIO_DTSELoadCode` |
| `0x444398` / `0x4444b4` | `HAL_AUDIO_HDMI_ARC_SetNonPCM` / `…eARC_SetNonPCM` |
| `0x4448fc` | `HAL_AUDIO_SPDIF_Set_OmxOutputPcmMode` |
| `0x444914` / `0x444f40` / `0x445504` / `0x445dbc` | `HAL_AUDIO_SPDIF_AutoMode` / `BypassMode` / `TranscodeMode` / `PcmMode` |
| `0x44660c` / `0x44674c` / `0x4467ec` | `HAL_AUDIO_DigitalTx_SPDIFConfig` / `ARCConfig` / `eARCConfig` |
| `0x44678c` / `0x445edc` | `HAL_AUDIO_SPDIF_ApplySetting` / `HAL_AUDIO_DigitalTx_ApplySetting` |
| `0x448a10` | `HAL_AUDIO_SPDIF_SetMode` |
| `0x44a2c4` | `HAL_AUDIO_SPDIF_SetOutputType` |
| `0x422aa0` | `MDrv_AUDIO_Get_DTS_License` |
| `0x464888` / `0x464958` | `HAL_MAD_SetDTSCommonCtrl` / `HAL_MAD_GetDtsInfo` |

DTS SDO/packer strings present in `utpa2k.ko` `.data`: `DTSX_CORE2_API_SDO_Packer`, `Mstar_DTS_Hdmi_Packer error`, `D2A_DTSXENC`, `spdif ISR`.
→ **The DTS SDO packer and the SPDIF non-PCM setters live in the kernel module, not in the HAL or libmi3.**

### 3.3 Caller analysis — a null result that is itself the finding

A full `bl`/`blx` scan of `utpa2k.ko` `.text` (6.4 MB, **136 192** branch instructions, ARM mode) against all 17 SPDIF/non-PCM/DTS targets returned:

```
NO direct callers found
```

Every one of `HAL_AUDIO_SPDIF_Tx_SetNonPCM`, `…SetMode`, `…ApplySetting`, `HAL_AUDIO_DTSELoadCode`, `MDrv_AUDIO_Get_DTS_License`, `HAL_MAD_GetDtsInfo`, … has **zero direct callers**.

**Interpretation:** these are reached exclusively through **Utopia function-pointer dispatch tables** (`spt_*` / module ioctl tables), not direct calls.

> **Consequence:** static call-graph analysis **cannot** establish whether a DTS playback stream reaches `DTSX_CORE2_API_SDO_Packer` or `HAL_AUDIO_SPDIF_Tx_SetNonPCM`. This is a hard limit of static xref here, not an oversight. It is the single most important methodological result of R4's static half.

### 3.4 The SPDIF-mode *decision* strings are dead in the shipped `libutopia.so`

These decision strings exist in `libutopia.so` `.rodata`:

| String | VA |
|---|---|
| `SPDIF out codec prev(%d), new(%d)` | `0x1656ca` |
| `Hash Key Check DTSX Fail, no DTSX license!!` | `0x165546` |
| `eDigitalOutfMode  = %x, eNonPcmPath = %x` | `0x1a7700` |
| `R2NonPcmSetting = %x` | `0x1a7738` |
| `Fail to switch SPDIF mode` | `0x1ad63e` |
| `HAL SPDIF set as Non-PCM` | `0x17d9cc` |
| `SPDIF mode set to %s` | `0x1b8d53` |
| `Tx_NonPCM`, `Invalid SPDIF Path`, `Hash-key Support DTSX.` | `0x1a74c8`, `0x16b49b`, `0x1acf12` |

Each was searched for by **all three** reference mechanisms, over all sections:

| Mechanism | Result |
|---|---|
| PC-relative `ldr rx,[pc,#imm]` literal pool in `.text` | **not found** |
| `movw`/`movt` immediate pairs (`.text` is far from `.rodata`, so this is the required idiom) | **not found** |
| Relocated pointer slot in `.data` / `.data.rel.ro` / `.got` | **not found** |

> **All 12 are unreferenced.** The SPDIF-mode decision code that owns these strings is **not linked into the shipped `libutopia.so`**. Any analysis that treats these strings as evidence of a live userspace decision path is unsound. (Methodology note: an initial pass reported false negatives due to `re.match(r'#(\d+)')` matching the leading `0` of hex immediates; corrected to hex-first.)

---

## 4. Objective 3 — AC3 vs DTS differential call-chain table

| Stage | Symbol / site | AC3 | DTS | Divergence? |
|---|---|---|---|---|
| Format constant | — | `0x09000000` | `0x0B000000` | by design |
| Parser support gate | `utils_isParserSupported` 0x28342 | **1** | **1** | **No** |
| Parser emits frames | `Parser_Write` | 873 calls | 724 calls | No (both live) |
| HAL write entry | `mi_common_raw_write` 0x031164 | entered | entered | **No** (format-agnostic) |
| Codec-ID mapping | `mi_decoder_open` 0x02f408 | `movs r1,#5` | `movs r1,#9` | **Yes — value only** |
| Decoder handle | `mi_decoder_open` | `0x19000000` | `0x19000000` | No |
| Kernel open | `MI_AUDIO_Open` | `AudioHAL`, `eRet:0x0` | `AudioHAL`, `eRet:0x0` | No |
| AOUT connect | `_MI_AOUT_NotifyConnectInput` | `hAout:0x17000000 hInput:0x19000000` | identical | No |
| AOUT profile | `…NotifyConnectInput:9078` | `eInput:0, eDrvOut:4, Profile[5,5], AqCsp[NONE,NONE]` | **identical** | **No** |
| Kernel start | `MI_AUDIO_Start` | `Codec:5, eRet:0x0` | `Codec:9, eRet:0x0` | **Yes — value only** |
| PCM read-out | `MI_PCM_Open HW DMA Reader1` | `0x81000000`, Ch8, `eAudioPath:0x7` | **identical** | No |
| Kernel ioctl | `MI_AUDIO_Write` → ioctl `0xC038100A` | format-agnostic | format-agnostic | No |
| `libmi3` write | `MI_AUDIO_Write` 0x62afd | no codec branch | no codec branch | No |
| SPDIF non-PCM set | `HAL_AUDIO_SPDIF_Tx_SetNonPCM` 0x44428c | **not in logs** | **not in logs** | unknown |
| DTS SDO packer | `DTSX_CORE2_API_SDO_Packer` | n/a | **not in logs** | unknown |

> **The only confirmed difference anywhere in the instrumented path is the codec-ID value (5 vs 9).** Every structural, handle, profile, DMA and return-code observable is byte-identical.

---

## 5. Objective 4 — format-dependent branches and the state words

- **Confirmed format-dependent branches:** exactly two —
  1. `mi_decoder_open` @0x02f408 (format → vendor codec ID 5 / 9; succeeds for both),
  2. `utils_isParserSupported` @0x28342 (§2; **DTS passes**).
- **`mi_common_raw_write` and `MI_AUDIO_Write`:** format-agnostic (§1).
- **State words** `0x440` (missing-bits mask), `0x444` (available-bits mask), `0x4d0` (Dolby profile / DSP image), `0x4d8` (DTS licence level: 0=none, 1=core, 2=HD, 3=DTS:X), `0x582` (DTS:X flag): produced by `MDrv_AUDIO_CheckHashkey` (`utpa2k.ko` @0x423494). **Not patched in R4**, per constraint.
- Critically, `REPORT_full_license_profile.md` found that `0x4d8` is **not read** by `SetDecodeSystem` / `SetSystem2` / `SetDecodeCmd` — i.e. the computed DTS-licence level is largely *not consulted* on the decode/transmit path.
- **R4 adds a decisive runtime fact here — see §6.3.**

---

## 6. Objective 5 — runtime differential (R4c, playback-only windows)

Artifacts: `runtime_phase4/R4c_20260905_010855_{ac3,dts}_playback.{logcat,dmesg}`
Method: Kodi launched via `am start -a android.intent.action.VIEW -d file://… -t video/mp4 -n net.kodinerds.maven.kodi22/.Splash`, SETTLE=22 s, **no `dumpsys` in window**.

### 6.1 Both formats establish passthrough correctly

DTS: `CAEStreamParser::SyncDTS - dts stream detected (6 channels, 48000Hz, 16bit BE, period: 512, syncword: 0x7ffe8001, framesize 2012)` → `Creating audio stream (codec: dts, channels: 6, sample rate: 48000, pass-through)` → `CAESinkAUDIOTRACK::Initializing … format: AE_FMT_RAW (AE) method: RAW (PT) stream-type: STREAM_TYPE_DTS_512`.
AC3: same shape with AC3 stream type. Both reach `enter PLAY` (AC3 ×1, DTS ×2).

### 6.2 Kernel traces are identical

A normalized diff (timestamps, PIDs, TIDs stripped) across the two `dmesg` files returns only PID/SELinux/MMA-allocation noise. The audio-relevant lines are **identical**, including the AOUT profile line — which is **not** codec-specific:

```
_MI_AOUT_NotifyConnectInput:9078] eInput:0, eDrvOut:4, Profile[pre,cur]:[5,5], AqCspName[pre,cur]:[NONE,NONE]
```

(Only 2 occurrences of `Profile[…]` exist across *all* captures, both `[5,5]` — so `Profile` is a fixed AOUT profile for this output path, **not** a per-codec value. Earlier suspicion that DTS was being registered downstream as profile 5 is therefore **not** supported.)

**Neither window contains any `SPDIF`, `SetNonPCM`, `SDO`, `DTSX`, `Transcode`, `burst` or channel-status line.** The SDO / IEC61937 / non-PCM layer is silent for **both** formats in the 22 s window.

### 6.3 ⚠ The captures are NOT stock baseline — and that is the most valuable result

On-device `/vendor/lib/modules/utpa2k.ko` md5 = **`4c5e6fbb10e24abc3b8d2b0490507983`**, which `REPORT_forensic_synthesis.md` §4.2 identifies as the **Exp B module** — i.e. the *fully DTS-licensed* emulation (`0x4d0=4` MS12V22 image, `0x4d8=3` DTS:X level, `0x582=1`, DTS AUTH missing-bits `0x8/0x80/0x20000` cleared).

> **Therefore: the R4c traces were captured with the DTS licence/AUTH gate already fully bypassed — and DTS still fails (AC3 passes).**
>
> This **falsifies the DTS-licence hypothesis as the blocker** and corroborates `REPORT_forensic_synthesis.md` §"Exp B deployed; AC3 PASS / DTS FAIL". Any future hypothesis must survive this fact.

---

## 7. Objective 6 — classification

Against the R4 answer-key:

- **R4-A** (DTS reaches the same SDO/IEC61937 path) — **not supportable**: neither format shows any SDO/IEC61937 activity.
- **R4-D** (no software divergence) — **rejected**: a codec-ID divergence (5 vs 9) demonstrably exists and is by design.
- **R4-C** (reaches packer, different failure branch) — **not supportable**: no evidence DTS reaches the packer at all.
- **R4-B** (never reaches the packer) — **best supported**.

### Verdict: R4-B (qualified)

Restated in the required conservative form:

- **Last confirmed common point:** `_MI_AOUT_NotifyConnectInput` (identical `eInput:0`, `eDrvOut:4`, `Profile[5,5]`, `AqCspName[NONE,NONE]`) alongside `MI_AUDIO_Start` returning `eRet:0x0` and `MI_PCM_Open HW DMA Reader1` (`0x81000000`, Ch8, `eAudioPath:0x7`) — **identical for AC3 and DTS**.
- **First confirmed divergence:** **none in the instrumented transmit path.** The only confirmed difference is the codec-ID *value* (AC3 `5`, DTS `9`) handed to `MI_AUDIO_Start`.
- **Qualification:** R4-B here means *"not observed to reach"*, **not** *"proven not to reach"*. The SPDIF output-mode selection is not instrumented, and `HAL_AUDIO_SPDIF_Tx_SetNonPCM` is function-pointer-dispatched (§3.3), so static reachability is also undecidable.
- **Next target:** the SPDIF/HDMI **output-mode selector** — see §8–§9.

---

## 8. Strongest hypothesis

The divergence is **not in the compressed-frame transport** (proven format-agnostic through the ioctl, §1) and **not in the licence/AUTH gate** (falsified by Exp B, §6.3). It is in the **SPDIF output-MODE selection**, which is **EDID-capability-gated**.

Per `REPORT_forensic_deepdive_static.md` (binary-verified, consumption side), `mik.ko::_MI_AOUT_SetHdmiAutoMode` (**0x989cc**) tests an EDID-derived capability bitmask keyed by **CEA-861 Short Audio Descriptor format code used as bit position**:

| Bit | CEA format code | Codec |
|---|---|---|
| `0x4` | `0x02` | AC3 |
| `0x80` | `0x07` | **DTS core** |
| `0x400` | `0x0A` | E-AC3 |
| `0x800` | `0x0B` | DTS-HD |
| `0x1000` | `0x0C` | MLP / TrueHD |

Selection: `r6==3 → r4=3`; `r6==1 → r4=2`; **`r6==0 → r4=0`**; then `MApi_AUDIO_SPDIF_SetMode(r4)`.

**Mechanism of the AC3-works / DTS-fails asymmetry:** when the matching compressed-audio bit is **absent**, the selector takes `r6=0 → r4=0` = **SPDIF PCM mode**. If the sink advertises AC3 (`0x4`) but **not** DTS (`0x80`) — an extremely common EDID — then:

- **AC3** → bit present → compressed SPDIF mode → IEC61937 passthrough works;
- **DTS** → bit `0x80` absent → `r6=0` → **SPDIF forced to PCM** → the DTS bitstream is never emitted as IEC61937, and the DTS path collapses to PCM (silence/undecodable).

This is an **output-mode gate, not a data mute** — the R2 DTS decoder may well run, but its output is reformatted to PCM at the SPDIF TX. That is exactly the "decoder alive, passthrough dead" signature observed across R1–R4.

**Corroboration:** Exp B (§6.3) removed the entire kernel DTS licence gate and DTS still failed; `mik.ko` on device is unmodified, so **this EDID gate is still live**.

Confidence: **binary mechanism = verified (prior phase); that it is the decisive runtime blocker on *this* sink = strong inference, not yet runtime-confirmed** — the actual bitmask value in effect was not observable (§9, caveat).

---

## 9. Next experiment (for authorization — not performed in R4)

**Goal:** turn §8 from strong inference into proof, **without patching and without altering EDID**.

1. **Read-only sink-capability dump.** Determine the EDID-derived audio bitmask actually in effect and verify whether AC3 bit `0x4` is set while DTS bit `0x80` is clear on the current sink. Candidate routes (all non-mutating): read the connected sink's EDID SADs; instrument `mik.ko::_MI_AOUT_SetHdmiAutoMode` (0x989cc) / `MApi_AUDIO_SPDIF_SetMode` with **ftrace/kprobe** to log the bitmask and the selected `r4` mode number.
   - *Blocked this session:* `mik!aout` sysfs exposes only `dev`/`uevent` — no runtime capability state; the `/vendor/cusdata/bsp/common/EDID_BIN/*` files are the projector's **own** EDID (presented to sources), **not** the audio sink's. So a live read needs ftrace/kprobe rather than sysfs.
2. **Differential mode capture.** With instrumentation live, replay AC3 and DTS and record the `MApi_AUDIO_SPDIF_SetMode` argument. Predicted: AC3 → `2`/`3` (compressed); DTS → `0` (PCM). A confirmed `0` for DTS closes the investigation.
3. **Only if confirmed** — mitigation would be forcing the compressed SPDIF sub-mode for DTS (`r6=1`) in `mik.ko`. **This is patching and was out of scope for R4**; it requires explicit authorization in a later phase.

**Not recommended (already falsified or already done):** further DTS-licence / AUTH / `0x4d8` work (falsified by Exp B, §6.3); further `mi_getCodecType` work (R3, closed); Binder or `sound_spdif_type` probing (excluded).

---

## 10. Caveats

1. **Device is not at stock baseline.** `utpa2k.ko` = Exp B (`4c5e6fbb`), with upstream `libmi3.so` v3 and a patched `mik.ko` MonitorTask per R2. All R4 runtime traces therefore describe the **patched** stack. Comparisons against a truly stock baseline were out of scope.
2. **ISA correction:** `utpa2k.ko` is **ARM (A32)**. Any earlier Thumb-mode disassembly of this module is invalid.
3. **`libutopia.so` SPDIF-decision strings are dead** (§3.4) — do not treat them as evidence of a live userspace path.
4. **`HAL_AUDIO_SPDIF_*` reachability is statically undecidable** (function-pointer dispatch, §3.3); only runtime instrumentation can resolve it.
5. `Profile[5,5]` is **not** codec-specific — an earlier working hypothesis that DTS was mis-registered downstream as AC3 is **not supported**.

---

## 11. Artifacts

| Path | Content |
|---|---|
| `runtime_phase4/R4c_20260905_010855_{ac3,dts}_playback.{logcat,dmesg}` | R4c playback-only windows (primary evidence) |
| `tools/r4_dis_hal.py` | HAL Thumb disassembler w/ PLT + literal-string annotation |
| `tools/r4_ko_syms.py`, `r4_ko_spdi2.py` | `utpa2k.ko` SPDIF/DTS symbol map |
| `tools/r4_ko_callers2.py` | ARM-mode `bl`/`blx` caller scan (136 192 branches) |
| `tools/r4_movw.py` | `movw`/`movt` string-reference scanner (hex-first) |
| `tools/r4_litref.py`, `r4_dataptr.py`, `r4_relref.py` | alternate reference-mechanism probes (all negative) |
| `tools/r4_normdiff.py` | timestamp/PID-normalized AC3↔DTS differ |
| `tools/r4_mode_check.py` | ARM-vs-Thumb ISA determination |
