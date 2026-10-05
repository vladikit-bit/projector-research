# Engineering Report — C50A Projector (MediaTek MT5889/MStar) Digital Audio Path & SPDIF Passthrough Investigation

Date: 2026-08-26 · Method: read-only ADB inspection (device 192.168.0.183, adbd=root) + one reversible observation (Kodi launched, then force-stopped). No firmware, settings, mixer, policy, or Kodi configuration was modified.

---

## A. Actual audio architecture (with real component names)

```
Kodi 21.2 (org.xbmc.kodi)
  ↓ AudioTrack, PCM float (AE_FMT_FLOAT, "PCM-STREAM")        [RAW/IEC61937 never opened — see §C]
  ↓ libaudioclient → AudioFlinger (audioserver pid 344, Android 11)
  ↓ ONE Mixer output thread "AudioOut_D", flags PRIMARY, 48 kHz stereo PCM16
  ↓ AudioPolicyManager (configurable engine, /vendor/etc/audio_policy_configuration.xml)
  ↓ HIDL android.hardware.audio@6.0 (binderized) → mstar.hardware.audio.service (pid 322)
  ↓ /vendor/lib/hw/audio.primary.mt5889.so  (strings: "mi_common_pcm_open … device[speaker], path[HW DMA Reader1]")
  ↓ libmi3.so (MI_AOUT_* / MI_PCM_* APIs) → libutopia.so (Utopia2K middleware)
  ↓ kernel ioctl → built-in modules [mik], [utpa2k], [dtv_driver]; DT node /alsa { compatible="Mstar-alsa" }
  ↓ MStar MAD audio DSP (card0 "MStar-MAD-No00"), input via HW/SW/R2 DMA Reader channels
  ↓ DSP output matrix (registers 0x112D50–0x112D54: dac0-3 / i2s sd0 / [11:8]spdif / [7:4]arc)
  → internal DACs (speaker) · SPDIF TX · HDMI/ARC TX   (parallel taps of ONE DSP pipeline)
```

Key services: `audioserver`, `vendor.audio-hal` (= mstar.hardware.audio.service), `tv-mtkmtalservice-hal-1-0` (MTAL), `tv-mtkrm-hal-1-0` (resource manager), `cpu-audio`.

## B. PCM path (UI sounds / normal playback) — CONFIRMED live

Live dumpsys while YouTube played, and again while Kodi played:

- Single output thread `AudioOut_D` (MIXER), 48 kHz, 2ch, PCM16, HAL frame count 864, `AUDIO_OUTPUT_FLAG_PRIMARY`.
- Active tracks: YouTube (float 48k stereo, 0 underruns) and later **Kodi pid 8896 (float 44.1 kHz stereo, resampled by AudioFlinger to the thread's 48 kHz)** — both on the *same* thread.
- Patch: `mix handle 13 → AUDIO_DEVICE_OUT_SPEAKER`. Only speaker is a connected/available output device; SPDIF/HDMI/ARC are declared but **not connected** in the available-devices list.
- HAL log: `mi_common_pcm_open: handle 13, pcm_handle 0x81000000, device[speaker], path[HW DMA Reader1], format pcm, chn 2, sr 48000, dma input chn 2, blocking-mode 0` → PCM written into the DSP through DMA-reader channel 1.
- All kernel ALSA playback subdevices (`card0 pcm0p/1p/2p/3p/7p`) were **closed** during playback → the main path does not use ALSA PCM at all; ALSA on this platform serves auxiliary paths (tinymix exposes only DMIC/mic controls).

System-path timing health: thread timestamp stats n=1,131,515 disc=42, localSR 47999.3 (drift 1.25e-8), corrected jitter σ≈0.009 ms; mixer underrun counters partial=0 empty=0; zero underrun/xrun lines in dmesg/logcat.

## C. Passthrough path — declared but unreachable from Kodi

Declared capability (audio_policy_configuration.xml + dumpsys):
- `offload output`: flags `DIRECT|COMPRESS_OFFLOAD`, formats **AC3, E_AC3, E_AC3_JOC, DTS, DTS_HD**, AAC family, AC4; supported devices include Speaker, HDMI Out, HDMI ARC, **Spdif**. curOpenCount=0 (never opened).
- `hw_av_sync output`: `DIRECT|HW_AV_SYNC`, PCM + all compressed formats. Never opened.
- HAL implements real bitstream machinery: `utils_isPassthroughSupported()`, AC3/EAC3 header parsers (`__have_ac3_header`), digital-output controls `Set{Spdif,HdmiTx,HdmiArc}OutputMode = AUTO/BYPASS/PCM/TRANSCODE`, `Set…OutputType = AC3/AC3P/DTS/NONE`; Utopia contains DTS/SDO SPDIF packers (`DTSDecSDOPacker_API_*`, IEC-header insertion), `HAL_AUDIO_SPDIF_{Pcm,Bypass,Transcode,Auto}Mode`, `HAL_AUDIO_SPDIF_Tx_SetNonPCM`.

Why Kodi never uses it:
1. `dumpsys audio` log: boot-time `setForceUse(FOR_ENCODED_SURROUND, FORCE_NONE)`; no digital sink is connected, and the HAL reports `Supported ARC capability: dd:0 ddp:0 dts:0 aac:0 …` all zero, plus `HDMI audio codec type: XPCM`.
2. Kodi therefore enumerates its only device with `m_dataFormats: …,AE_FMT_RAW` but **`m_streamTypes : No passthrough capabilities`** (kodi.log, reproduced live).
3. Result for AC3 6ch content (codec id 86019): `Creating audio stream … no pass-through` → `CDVDAudioCodecFFmpeg` decodes to PCM → `CAESinkAUDIOTRACK::Initializing … format: AE_FMT_FLOAT … stream-type: PCM-STREAM`. Kodi's `audiooutput.passthrough=true` / `ac3passthrough=true` settings have no effect; `libaudiospdif.so` (the framework IEC61937 packer) is loaded but idle.

So passthrough on this box today would be "decoded PCM masquerading as passthrough" — i.e., it does not exist as a bitstream route for Kodi. Experiments B/C (observe AC3/DTS bitstream routing) were **not executable read-only**: Kodi cannot open a RAW track at all, and switching the projector's digital-output mode is a persistent-settings change that was forbidden.

## D. Why UI/menu sounds disappear when "SPDIF passthrough" is enabled

Evidence-based chain (not the generic "passthrough doesn't carry PCM" hand-wave):

1. Kodi's GUI sounds and its decoded-to-PCM media are ordinary float AudioTracks on the primary MIXER thread → HW DMA Reader1 → DSP main mixer (§B).
2. The projector-level SPDIF mode (HAL/Utopia: `SetSpdifOutputMode=BYPASS`, `HAL_AUDIO_SPDIF_BypassMode`, `Tx_SetNonPCM`) switches the SPDIF TX stage to non-PCM operation.
3. The driver explicitly mutes PCM content toward SPDIF in that state: kernel symbols `_MI_AOUT_SetMuteInBypassMode`, `_bIsSpdifMuteInBypass` ([mik]), `_MTAUDDEC_SetSPDIFMute` ([dtv_driver]). This is the concrete mechanism — PCM is not merely "unsupported" on a bypassing SPDIF; the stack deliberately mutes it.
4. Since Kodi never produces a real bitstream (§C), after switching the projector's digital output to BYPASS *everything Kodi plays* — UI clicks included — is PCM that gets muted on the digital output. On an amplifier fed from the optical port the result is complete silence, including menu sounds.
5. Additionally, if a genuine passthrough sink were ever negotiated, Kodi's audio engine renders GUI sounds only through its PCM sink; with output switched to a bitstream device the GUI-sound path is dropped on the Kodi side too. That second mechanism currently cannot engage because the sink never advertises passthrough (§C).

Trigger classification: the mute-in-bypass behavior is CONFIRMED at symbol level; the end-to-end reproduction (flip projector SPDIF mode → observe silence) was not performed because it requires changing persistent settings. Classification: STRONG EVIDENCE.

## E. Stuttering — classification and evidence

Measured facts:
- System path is clean and clock-stable: 42 discontinuities in 1.13M timestamps, sample-rate drift ≈1e-8, jitter σ≈0.009 ms, zero mixer-thread underruns, zero XRUNs in kernel log, no format-negotiation failures, no route flapping (one patch, stable since boot except app foreground changes).
- **Kodi's own track accumulated 244 underruns within ~1 minute of menu navigation** (live dumpsys, track id 106, client 8896), while YouTube tracks show 0. Kodi's AudioTrack buffer is only ~180 ms (log: "playing time of 180.7 ms"), it reopens its sink on every sample-rate transition (44.1↔48 kHz entries around every stream change), and `CDVDVideo::AddPacketsRenderer - timeout adding data to renderer` / `stream stalled` / `ActiveAE - large audio sync error: -11663 ms` appear in kodi.log.

Conclusion: **cause of any audible digital-output stutter = UNKNOWN** (no electrical measurement of the SPDIF/HDMI line was possible read-only). However, the evidence that exists points at Kodi-side buffering/sink-churn, not at HDMI/SPDIF hardware contention: the shared-resource architecture below is real, but the shared system path shows excellent stability, and the only starvation signal measured sits inside Kodi's client buffers.

Missing evidence to close: optical-line capture (external), long-run offload-path trace with a real bitstream player (needs settings change), DSP DMA-reader occupancy counters under dual-stream load.

Shared-resource facts (relevant to §F): one AudioFlinger thread + one HAL stream serve all PCM; the DSP exposes a small fixed pool of DMA reader inputs (HW×2, SW×3, R2×2) with literal contention errors (`Err! HW DMA Reader 1 was already used !` in libutopia.so); speaker/headphone/SPDIF/ARC are parallel taps of the same DSP output matrix with a common delay/latency model.

## F. HDMI/SPDIF relationship (explicit answers)

- Separate PCM devices? **No.** Both are sink-device attributes of the same HAL streams; every mixPort routes to both (XML routes), and the ALSA PCMs aren't even used for main playback.
- Shared PCM device / shared HAL output? **Yes** — one HAL output stream (handle 13) per profile; device selection happens inside the HAL/DSP, not by opening different streams.
- Shared DMA? **Yes at the DSP boundary** — content enters via shared DMA-reader channels (primary uses HW DMA Reader1; pool is small and contended).
- Shared I2S? SPDIF is a distinct TX block (`AU_SPDIF_TX_CS_INT0/1` interrupts, `MApi_AUDIO_I2S_SetMode` governs the I2S pins separately), but both take their data from the same DSP output matrix (reg 0x112D52 i2s sd0, 0x112D54 [11:8]spdif/[7:4]arc).
- Shared clock? **Yes — a single audio DSP clock domain** feeds DACs, I2S, SPDIF, ARC (common delay model, single resampler; measured drift ≈1e-8). No separate SPDIF PLL appears anywhere in symbols/strings.
- Mutually exclusive? At Android level, no (multiple profiles exist and up to 2 simultaneous offload actives are allowed). At DSP level, PCM-vs-bypass on the digital outputs is mutually exclusive *per output*: bypass mutes PCM on that output (§D).

Note: this model has no HDMI *output* currently connected (available-devices list holds only Speaker; ARC caps all zero) — "HDMI Out"/ARC exist in config; the physically exercised digital port is SPDIF.

## G. Exact relevant files / libraries

On device:
- Policy: `/vendor/etc/audio_policy_configuration.xml` (authoritative), `/vendor/etc/audio_policy.conf` (legacy), `/vendor/etc/audio_policy_engine_*.xml`, `/vendor/etc/audio_effects.xml`. **No mixer_paths*.xml exists** (mixer unused for playback).
- HAL: `/vendor/lib/hw/audio.primary.mt5889.so`, `/vendor/lib/hw/android.hardware.audio@6.0-impl.so`, service `/vendor/bin/hw/mstar.hardware.audio.service`.
- Middleware: `/vendor/lib/libmi3.so`, `/vendor/lib/libutopia.so` (19 MB, contains all SPDIF/DSP logic), `/vendor/lib/libalsautils.so`, `/vendor/lib/libaudioparser.so`.
- Framework: `/system/lib/libaudiospdif.so` (idle), AudioFlinger/AudioPolicy in audioserver.
- Kernel: built-ins `[mik]`, `[utpa2k]`, `[dtv_driver]` (see `_MTAUDDEC_*`, `MApi_AUDIO_SPDIF_*`, `_MI_AOUT_*` symbols); DT node `/sys/firmware/devicetree/base/alsa` (compatible "Mstar-alsa"); interrupts `AUDMA_V2_INTR`, `AU_SPDIF_TX_CS_INT0/1`, `HDMI_NON_PCM_MODE_INT_OUT`, `SPDIF_IN_NON_PCM_INT_OUT`.
- Settings: `persist.vendor.audio.{spdif,hdmi_arc,hdmi_tx}.{mode,type}` props exist (BYPASS/DTS defaults) but effective values reported by HAL are `ui(auto)/current(auto), type not-specified` — the operative store is the ATV TV-settings path via mtktvapi/mtkdmservice, not these props.
- Kodi: `/sdcard/Android/data/org.xbmc.kodi/files/.kodi/userdata/guisettings.xml`, `…/temp/kodi.log`.

Local evidence copies: `C:\firmware_temp\spdif_audio_investigation\` (props_all.txt, dumpsys_audio_flinger.txt, dumpsys_audio_policy.txt, dumpsys_audio_service.txt, policy_confs.txt, proc_asound.txt, tinymix_dump.txt, kallsyms_*.txt, dmesg_full.txt, logcat_full.txt, kodi_guisettings.xml, kodi_log.txt, kodi_log_live.txt, exp_kodi_*.txt, libs/*.so + *.strings).

## H. Confidence summary

| Conclusion | Status |
|---|---|
| Full PCM chain Kodi→AudioTrack→AudioFlinger→mt5889 HAL→MI_AOUT→DSP DMA Reader1→speaker | **CONFIRMED** (live dumpsys + HAL logs) |
| SPDIF is not a separate PCM device; shared HAL stream/DSP path | **CONFIRMED** |
| Platform has real bitstream capability (offload DIRECT AC3/EAC3/DTS/DTS-HD + HAL/Utopia packers) but it is never opened | **CONFIRMED** (declared + curOpenCount=0) |
| Kodi cannot passthrough because Android reports no passthrough capabilities (FOR_ENCODED_SURROUND=NONE, no digital sink, ARC caps=0) | **STRONG EVIDENCE** (exact framework decision path is INFERENCE) |
| UI-sound silence under SPDIF bypass = deliberate PCM mute in bypass mode | **STRONG EVIDENCE** (`_bIsSpdifMuteInBypass` et al.); end-to-end repro INFERENCE (settings change forbidden) |
| Single shared audio clock domain for DAC/I2S/SPDIF/ARC | **STRONG EVIDENCE** |
| Shared clock causes observed stutter | **UNKNOWN** — contradicted by measured system-path stability; measured starvation is inside Kodi's buffers (244 underruns/min on Kodi track) |
