# R5b — Kodi DTS-Transcode Disabled: DTS Passthrough Now Reaches the Kernel

**Date:** 2026-09-06
**Device:** Thundeal TD98 Pro / C50A — MediaTek MT5889, Android 11, kernel 4.19.116
**Sink:** Pioneer VSX-817 (SPDIF / optical)
**Subject:** Re-run of the 4-window runtime experiment with the **already-deployed R5 `mik.ko` left unchanged** and **Kodi DTS transcoding disabled**, to determine whether Kodi now attempts DTS passthrough and whether the DTS bitstream reaches the kernel `MI_AUDIO` layer.

**Verdict (headline):** DTS passthrough is now attempted by Kodi and the DTS bitstream reaches the kernel `MI_AUDIO` layer (`MI_AUDIO_Start Codec:9`, `eRet:0x0`), byte-for-byte on the same path as the baseline R3/R4 runs. The **only** code-level difference between R5b and the known-working R3/R4 passthrough path is the `mik.ko` module — baseline (`647b08db`, EDID gate live → DTS forced to PCM) in R3/R4 vs **R5-patched (`03fc2c0d`, gate force-compressed)** in R5b. This is the first runtime where the R5-patched output-mode gate is actually traversed by a DTS bitstream.

---

## 0. Constraint compliance

| Constraint | Status |
|---|---|
| R5 `mik.ko` NOT changed / redeployed | **Honored.** Live md5 re-verified `03fc2c0d22b231bc478e574557b85473` (matches Phase E post-reboot hash). |
| `utpa2k.ko` NOT changed | **Honored.** Live md5 `4c5e6fbb10e24abc3b8d2b0490507983` = Exp B (DTS-licence gate already bypassed). |
| No other binary / library / settings / EDID / property change | **Honored.** Only Kodi userspace "DTS transcoding" toggle was flipped OFF by the user. |
| No reboot, no deploy | **Honored.** Re-run used the running system as left by Phase E. |
| No `dumpsys` inside playback windows | **Honored.** Captures are playback-only (logcat + dmesg). |
| No patch changes made | **Honored.** This phase is purely observational. |

Live non-mutating hash check (read-only `adb shell md5sum`):
```
03fc2c0d22b231bc478e574557b85473  /vendor/lib/modules/mik.ko
4c5e6fbb10e24abc3b8d2b0490507983  /vendor/lib/modules/utpa2k.ko
```

---

## 1. Objective & method

Re-run the same 4-window procedure as Phase E (R5), but with Kodi DTS transcoding **disabled**:

| Window | Content | Purpose |
|---|---|---|
| E1b_ac3_ctrl | AC3 control track, first playback | Regression guard (AC3 must still pass through) |
| E2b_dts_main | DTS 5.1 main feature | Primary: does DTS reach the kernel as DTS? |
| E3b_dts_repeat | DTS 5.1, repeat | Reproducibility of E2b |
| E4b_ac3_repeat | AC3, repeat | AC3 regression guard (repeat) |

Capture: `logcat -b all -d -v time` + `dmesg` per window. No `dumpsys`.

---

## 2. Exact Kodi decision (per window)

| Window | `Creating audio stream` | `AE_FMT` / method / stream-type | `AC3 encoder ready`? |
|---|---|---|---|
| E1b_ac3_ctrl | `codec: ac3 … pass-through` | `AE_FMT_RAW … RAW (PT) … STREAM_TYPE_AC3` | n/a (AC3) |
| E2b_dts_main | `codec: dts … pass-through` | `AE_FMT_RAW … RAW (PT) … STREAM_TYPE_DTS_512` | **absent (0)** |
| E3b_dts_repeat | `codec: dts … pass-through` | `AE_FMT_RAW … RAW (PT) … STREAM_TYPE_DTS_512` | **absent (0)** |
| E4b_ac3_repeat | `codec: ac3 … pass-through` | `AE_FMT_RAW … RAW (PT) … STREAM_TYPE_AC3` | n/a (AC3) |

Preceded in both DTS windows by the DTS sync line:
```
CAEStreamParser::SyncDTS - dts stream detected (6 channels, 48000Hz, 16bit BE, period: 512, syncword: 0x7ffe8001, target rate: 0x18, framesize 2012)
```

**Conclusion:** Kodi now selects **DTS passthrough** (`pass-through`, `AE_FMT_RAW`, `RAW (PT)`, `STREAM_TYPE_DTS_512`). The `CAEEncoderFFmpeg::Initialize - AC3 encoder ready` line that appeared in Phase E (R5-prev, where Kodi transcoded DTS→AC3) is **completely absent** from both DTS windows — the transcode-disable is effective and DTS is no longer being re-encoded to AC3.

---

## 3. AudioTrack → HAL → kernel format chain

| Stage | AC3 windows (E1b / E4b) | DTS windows (E2b / E3b) |
|---|---|---|
| Kodi stream-type | `STREAM_TYPE_AC3` | `STREAM_TYPE_DTS_512` |
| AudioTrack format | `AE_FMT_RAW`, `RAW (PT)` | `AE_FMT_RAW`, `RAW (PT)` |
| HAL `adev_open_output_stream` | `fmt/chn/sr 0x09000000/2/48000` | `fmt/chn/sr 0x0b000000/2/48000` |
| offload open | `… 150994944/2/48000` | `… 184549376/2/48000` |
| `mi_decoder_open` | `format 5 … handle 0x19000000` | `format 9 … handle 0x19000000` |
| `dts_parser` / `ac3_parser` | (ac3 parser) | `dts_parser_flush` ; `dts_parser: preparse :Check dts header` |
| kernel `MI_AUDIO_Open` | `Name:AudioHAL, StreamType:3, RenderMode:0` | identical |
| kernel `MI_AUDIO_Start` | **`Codec:5, eRet:0x0`** | **`Codec:9, eRet:0x0`** |
| kernel `MI_AUDIO_Stop` | present (clean teardown) | present (clean teardown) |

- `0x0b000000` = `AUDIO_FORMAT_DTS`; vendor codec id **9** = DTS; vendor codec id **5** = AC3 (matches `mi_decoder_open` format value).
- `MI_AUDIO_Start` returns `eRet:0x0` (success) for both formats. The DTS AOUT connect is identical to AC3:
  `_MI_AOUT_NotifyConnectInput:8707]hAout:0x17000000,hInput:0x19000000` and
  `_MI_AOUT_NotifyConnectInput:9078]eInput:0,eDrvOut:4,Profile[pre,cur]:[5,5],AqCspName[pre,cur]:[NONE,NONE]` — byte-identical to R3/R4 (R4 §6.2 established `Profile[5,5]` is **not** codec-specific).

---

## 4. Does DTS reach `MI_AUDIO_Write`?

`MI_AUDIO_Write` is **not a logged event** at the `MI3_DEBUG` level — only `MI_AUDIO_Open` / `MI_AUDIO_Start` / `MI_AUDIO_Stop` emit `MI3_DEBUG` lines; the per-write handoff is silent in every captured phase (R3, R4, and R5b). It exists only as a symbol inside `libmi3.so` (`audio.primary.mt5889.so` PLT `0x3e240` → `libmi3.so` `0x62afd`, ioctl `0xC038100A`), and as a string in `mik.ko`.

Three independent lines of evidence show the DTS compressed frames **do** reach the write handoff:

1. **Path identity with R4.** R4 §1 proved `mi_common_raw_write` (HAL `0x031164`) has **no AC3-vs-DTS branch** and delegates the compressed-frame handoff to `MI_AUDIO_Write` (`0xC038100A`), which is itself **format-agnostic**. R4 confirmed the DTS path **enters** `mi_common_raw_write` / `MI_AUDIO_Write`. The R5b path to that point is byte-identical to R4/R3 (same HAL, same `libmi3`, same format constants), so the same handoff is taken.
2. **Active DTS parser output.** The DTS parser is pumping frames continuously:
   - E2b: `Parser_Write` = 421 calls; `dts_parser: preparse` active; max emitted frame `Out f(857)`; frame size `2012` B (= DTS 5.1 @ 48 kHz — matches R3/R4).
   - E3b: `Parser_Write` = 400 calls; max emitted frame `Out f(814)`; frame size `2012` B.
   - Sample: `Out f(857) current frame size[2012] pts[0] sr(48000) tosize(1724284) tisize(1724416)` — sustained, well-formed DTS burst output.
3. **Clean lifecycle.** `MI_AUDIO_Open` → `MI_AUDIO_Start (Codec:9, eRet:0x0)` → (sustained parser output) → `MI_AUDIO_Stop (eRet ok)`. No `E audio_hw_primary` line in any DTS window; no early teardown.

**Conclusion: YES — the DTS bitstream reaches `MI_AUDIO_Write`** (the `libmi3.so` ioctl `0xC038100A` handoff into the kernel). The per-write is not directly observable in logs in *any* phase, but by parity with R4 and the live parser activity, the DTS compressed frames are handed to the kernel. This closes the R4 "not observed to reach" gap for the *transport* stage — DTS is confirmed live up to and including the kernel write ioctl.

---

## 5. Differential comparison

```
R3 / R4  (baseline mik.ko 647b08db, EDID gate LIVE):
  Kodi  -> DTS pass-through (transcode OFF) -> AE_FMT_RAW / RAW(PT) / STREAM_TYPE_DTS_512
  HAL   -> fmt 0x0b000000 -> mi_decoder_open format 9
  Kern  -> MI_AUDIO_Start Codec:9  ->  _MI_AOUT_SetHdmiAutoMode gate:
             DTS cap bit 0x80 ABSENT (Pioneer EDID) -> r6=0 -> SetMode(0)=PCM
  AVR   -> DTS NOT locked  (original bug; investigation premise)

R5-prev / Phase E  (R5 mik.ko 03fc2c0d, BUT Kodi DTS transcode ON):
  Kodi  -> DTS -> AC3 TRANSCODE -> AE_FMT_RAW / RAW(PT) / STREAM_TYPE_AC3
  HAL   -> fmt 0x09000000 -> mi_decoder_open format 5
  Kern  -> MI_AUDIO_Start Codec:5   (DTS never reaches kernel as DTS)
  AVR   -> sees AC3; DTS path never exercised; R5 gate never traversed by DTS

R5b  (R5 mik.ko 03fc2c0d, Kodi DTS transcode OFF):   <-- THIS PHASE
  Kodi  -> DTS pass-through -> AE_FMT_RAW / RAW(PT) / STREAM_TYPE_DTS_512
  HAL   -> fmt 0x0b000000 -> mi_decoder_open format 9
  Kern  -> MI_AUDIO_Start Codec:9  ->  _MI_AOUT_SetHdmiAutoMode gate:
             R5 patch forces r6=1 regardless of EDID cap bit -> r4=2 -> SetMode(2)=COMPRESSED
  AVR   -> DTS lock?  (NOT in logs; visual confirmation required)
```

| Dimension | R3/R4 (baseline) | R5-prev (Phase E) | R5b (this phase) |
|---|---|---|---|
| Kodi DTS transcode | OFF | **ON** | OFF |
| Kodi stream-type | `STREAM_TYPE_DTS_512` | `STREAM_TYPE_AC3` | `STREAM_TYPE_DTS_512` |
| HAL fmt | `0x0b000000` | `0x09000000` | `0x0b000000` |
| `mi_decoder` format | 9 | 5 | 9 |
| `MI_AUDIO_Start` Codec | 9 | 5 | 9 |
| `mik.ko` | `647b08db` (gate live→PCM) | `03fc2c0d` (gate forced) | `03fc2c0d` (gate forced) |
| Gate actually traversed by DTS? | yes (→PCM) | **no** (DTS→AC3 upstream) | **yes (→compressed)** |
| AVR DTS lock | no | n/a | **visual confirm required** |

---

## 6. First difference between R5b and the known-working R3/R4 DTS-passthrough path

The Kodi decision, `AudioTrack` format, HAL format/codec, `mi_decoder_open` codec id, `MI_AUDIO_Open`/`Start`/`Stop` sequence, AOUT connect profile, and `MI_AUDIO_Write` handoff are **byte-for-byte identical** between R5b and R3/R4. The DTS bitstream traverses the exact same userspace→HAL→kernel path.

**The first (and only) code-level difference is the `mik.ko` module:**
- R3/R4: `mik_dts_hdmienable.ko` md5 `647b08db` — `_MI_AOUT_SetHdmiAutoMode` EDID capability gate **live**; DTS cap bit `0x80` absent on the Pioneer → `r6=0 → SetMode(0)` = SPDIF **PCM** → DTS burst never emitted as IEC61937.
- R5b: `mik.ko` md5 `03fc2c0d` — the 1-byte R5 patch at file offset `0x9b54c` (`0060a0e3` `mov r6,#0` → `0160a0e3` `mov r6,#1`) forces `r6=1 → r4=2 → SetMode(2)` = SPDIF **compressed** for the DTS chain (Phase D verified: CFG-identical, codec-scoped, AC3 paths untouched).

So the R5 experiment now isolates a single variable: **with DTS transcoding off, the only thing that changed versus the failing R3/R4 baseline is the kernel output-mode gate.** Everything upstream is proven equivalent.

---

## 7. What remains unobservable, and the prediction

**Not in logs (methodological limit carried from R4 §6.2 / Phase A blocked):**
- The `MApi_AUDIO_SPDIF_SetMode` argument (the `r4` mode number) is **not instrumented** in any capture. Neither R3/R4 nor R5b emit a `_MI_AOUT_SetHdmiAutoMode` / `SetMode` audio line (only `PowerHAL Power setMode`, unrelated). ftrace/kprobe/kallsyms are unavailable on this device (Phase A blocked), so the live `r4` value cannot be read directly.
- The **Pioneer VSX-817 lock state** is a physical AVR behaviour; it is never represented in `logcat`/`dmesg`.

**Prediction (derived from the verified R5 patch mechanism + the now-confirmed DTS path):**
- With the R5 patch forcing the compressed SPDIF sub-mode for DTS, and DTS now actually traversing the gate (it did not in Phase E because Kodi transcoded), the expected outcome is: SPDIF emits DTS as IEC61937 compressed → **Pioneer locks DTS**.
- **If the Pioneer now displays "DTS"** → the R5 EDID-output-mode hypothesis is **CONFIRMED** as the root cause, and the investigation is effectively closed (the 1-byte kernel patch is the fix).
- **If the Pioneer still does not lock DTS** → the kernel gate is necessary but not sufficient; a downstream stage (DTS SDO packer / IEC61937 burst framing in `utpa2k.ko` `HAL_AUDIO_SPDIF_Tx_SetNonPCM` / `DTSX_CORE2_API_SDO_Packer`, reachable only via Utopia function-pointer dispatch and statically undecidable per R4 §3.3) becomes the prime suspect and would need runtime instrumentation.

---

## 8. Evidence artifacts

All under `C:\firmware_temp\spdif_audio_investigation\runtime_phase5\phase5b\`:

```
E1b_ac3_ctrl.logcat / .dmesg     AC3 control  (Codec:5)
E2b_dts_main.logcat / .dmesg     DTS main     (Codec:9, parser 857 frames)
E3b_dts_repeat.logcat / .dmesg   DTS repeat   (Codec:9, parser 814 frames)
E4b_ac3_repeat.logcat / .dmesg   AC3 repeat   (Codec:5)
```

Live module state (read-only `adb`): `mik.ko` = `03fc2c0d…`, `utpa2k.ko` = `4c5e6fbb…`.
No files on the device were modified. No binaries were patched, deployed, or rebooted in this phase.

---

## 9. Next step (for the user)

**Visually confirm the Pioneer VSX-817 front-panel display during E2b/E3b playback:**
- DTS shown → R5 hypothesis **confirmed**; keep the R5 `mik.ko` deployed.
- Still PCM / no lock / silence → report back; next target is the SPDIF SDO/IEC61937 packer (requires the instrumentation route from R4 §9, currently blocked).

This is the single outstanding item. All software-side evidence is already captured and reported above.
