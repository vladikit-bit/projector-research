# TD98 Pro / C50A — Raw/ARC Runtime Verification

**Date:** 2026-09-25  
**Device:** `192.168.0.183:5555` via the existing `aeon_validate/adbr.sh` wrapper  
**User action under test:** projector OSD changed to `RAW/ARC`; eARC reported enabled by the user.  
**Scope:** runtime verification only; no patch, firmware write, module reload, reboot, or persistent Android setting change was performed by this pass.

## 1. What the projector menu actually stored

The retained logcat shows the TV settings provider update:

```text
09-25 18:20:38.816  SSP-SystemProvider:
  update ... contentValues:[spdif_mode:2]
09-25 18:20:38.834  SSP-SystemProvider:
  cursor:[spdif_mode:2]
```

The same UI enumeration is logged in order:

```text
off
PCM/ARC
RAW/ARC
AUTO
```

Therefore the projector's `spdif_mode=2` corresponds to the displayed `RAW/ARC` selection, not to the Android HAL's `spdif mode` field.

The Android-side audio state did **not** change to Raw:

```text
dumpsys media.audio_flinger:
  spdif mode: ui(auto), current(auto)
  hdmi arc mode: ui(auto), current(auto)
```

The projector OSD value and the Android HAL desired/current mode are therefore separate state stores in this test.

## 2. eARC/ARC state

Read-only global settings and dumpsys showed:

```text
hdmi_arc_control_enabled=1
sound_earc=0
sound_spdif_type=4
Supported ARC capability: dd:0, ddp:0(atmos:0), dts:0, aac:0,
                         mat-truhd:0(Byte3:0), dtshd:0
```

The primary AudioFlinger output remained:

```text
Output devices: 0x2 (AUDIO_DEVICE_OUT_SPEAKER)
```

So the device did not expose an active Android HDMI-ARC audio route during the test, even though the projector-side OSD value was `RAW/ARC` and the global ARC control flag was `1`.

## 3. AC3 control playback

Test file: locally muxed `/storage/emulated/0/Movies/test_ac3_51.mp4` from the existing raw AC3 test asset.

Observed:

```text
MI_AUDIO_Start: Codec:5, hAudio:0x19000000, eRet:0x0
AudioFlinger offload format: AUDIO_FORMAT_AC3
Output devices: AUDIO_DEVICE_OUT_SPEAKER
Total writes: 4075
Normal mixer underruns: partial=0 empty=0
```

The AC3 stream reached the offload/HAL path without a start error. This is a control result only; it does not prove that the receiver received Dolby Digital.

A non-fatal diagnostic appeared immediately before the AC3 start:

```text
HAL_MAD_GetAudioInfo2: Unknown dolby type (255) adec_id(0)
```

`MI_AUDIO_Start` still returned `eRet:0`.

## 4. DTS playback

Test file: `/storage/emulated/0/Movies/test_dts_51.mp4`.

Observed:

```text
Kodi: Creating audio stream (codec: dts, channels: 6, sample rate: 48000,
       pass-through)
Kodi: RAW (PT) STREAM_TYPE_DTS_512
MI_AUDIO_Start: Codec:9, hAudio:0x19000000, eRet:0x0
AudioFlinger offload format: AUDIO_FORMAT_DTS
Total writes: 4120
Normal mixer underruns: partial=0 empty=0
Active offload track: 1
```

The DTS decoder/offload request was accepted at the Android/MI level. No `SetSpdifOutputMode=...`, `SetHdmiArcOutputMode=...`, nonPCM arming, or IEC61937/Spdif ISR evidence appeared in the captured logs.

After the stream ended, the HAL emitted the known teardown/polling diagnostics:

```text
mi_getCodecType: MI_AUDIO_GetAttr failed, ret=0x3
mi_getCodecType: MI_AUDIO_GetHandle failed, ret=9
Get codec type None!
```

These appeared after the DTS test and are not, by themselves, proof of a start failure; the start itself returned `eRet:0`.

## 5. Interpretation

This run establishes three separate facts:

1. The projector OSD value `spdif_mode=2` was written to the TV settings provider.
2. Android AudioFlinger accepted both AC3 and DTS as `DIRECT|COMPRESS_OFFLOAD` streams and the MI start calls returned success.
3. The Android/HAL state did not switch to Raw/ARC or an HDMI/SPDIF sink device; the primary output remained speaker, and no native nonPCM mode trace was observed.

Therefore the `RAW/ARC` menu selection, by itself, did not demonstrably propagate to `MApi_AUDIO_SYSTEM_Control`, the MI EDID mode gate, or the eARC route. The test does not prove that the projector hardware mode changed; it only proves that the OSD provider value changed and that the compressed streams reached the HAL.

The user's report that Dolby Digital also did not work in this state is consistent with the device-side observation: the stream reached the HAL, but the Android output route and eARC capability remained unestablished.

## 6. Artifacts and preservation

All raw captures are in:

```text
C:\firmware_temp\runtime_raw_arc_20260926\
```

Selected capture hashes are recorded in:

```text
38_capture_hashes.txt
```

This pass used `adb root` for read-only probing, pushed only the locally built AC3 control MP4, started Kodi twice for the authorized tests, and force-stopped Kodi afterward. The projector menu was left in the user-selected `RAW/ARC` state. No patch was generated or applied; no firmware, module, mixer, or persistent Android setting was changed by the test procedure.
