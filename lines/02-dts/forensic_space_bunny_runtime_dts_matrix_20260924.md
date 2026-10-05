# SPACE BUNNY — RUNTIME DTS / OUTPUT MODE MATRIX

**Date:** 2026-09-24  
**Device:** TD98 Pro / C50A, `192.168.0.183:5555`  
**ADB wrapper:** `aeon_validate/adbr.sh`, local server port `5038`  
**Scope:** Kodi playback and reversible Kodi audio-setting observations only  
**Firmware:** no patch, no module replacement, no flash, no reboot  
**TCL/T615:** not used

## Executive result

The runtime test did **not** reproduce a DTS decoder failure. It did establish that the output mode changes the path substantially:

| Condition | AC3 | DTS |
|---|---|---|
| Passthrough enabled, RAW Android IEC device | Kodi detected AC3, RAW PT, hardware decoder `format 5` | Kodi detected DTS, RAW PT, hardware decoder `format 9` |
| Passthrough disabled | Kodi/FFmpeg decoded AC3 to PCM, speaker output | Kodi/FFmpeg decoded DTS (`dca`) to PCM, speaker output |
| Passthrough device set to Kodi IEC packer | Repeated `CAEStreamInfo::GetDuration - invalid stream type` | Same repeated error class |

The test did **not** exercise a connected physical SPDIF/HDMI device. Android reported the available declared digital devices, but the connected media device remained `Speaker`. Therefore this run proves DTS input/decoder behavior and mode sensitivity, not final physical SPDIF delivery.

## 1. Device and test media inventory

Read-only device inventory succeeded through the existing wrapper:

```text
uid=2000(shell) ... groups=...adb...,sdcard_rw...
```

Available test media:

```text
/data/local/tmp/test_ac3_51.ac3       1,920,000 bytes
/data/local/tmp/test_dts_51.dts       7,545,000 bytes
/storage/emulated/0/Movies/test_ac3_51.mp4
/storage/emulated/0/Movies/test_dts_51.mp4
```

Kodi was running and JSON-RPC became available on `127.0.0.1:9090` after startup settled. The initial 9090 probe was refused while Kodi was restarting; subsequent RPC calls succeeded.

## 2. Initial Kodi output state

Before changing anything:

```text
audiooutput.config = 2
audiooutput.passthrough = true
audiooutput.ac3passthrough = true
audiooutput.dtspassthrough = true
audiooutput.audiodevice =
  AUDIOTRACK:AudioTrack (IEC)|Kodi IEC packer (recommended)
audiooutput.passthroughdevice =
  AUDIOTRACK:AudioTrack (RAW)|Android IEC packer
audiooutput.channels = 1  (2.0)
```

Kodi reported no active player at baseline. `dumpsys audio` showed STREAM_MUSIC on `speaker`; HDMI/ARC volumes existed, but the active device list was speaker.

## 3. Baseline: passthrough enabled, RAW Android IEC

### AC3

Raw file:

```text
/data/local/tmp/test_ac3_51.ac3
```

Kodi opened it on internal audio player 0. The log showed:

```text
CAEStreamParser::TrySyncAC3 - AC3 stream detected (6 channels, 48000Hz)
CAESinkAUDIOTRACK::Initializing ...
format: AE_FMT_RAW (AE)
method: RAW (PT)
stream-type: STREAM_TYPE_AC3
mi_decoder_open: format 5
mi_common_raw_open ... device[speaker]
```

The media session was active with no playback error.

### DTS

Raw file:

```text
/data/local/tmp/test_dts_51.dts
```

Kodi opened it on internal audio player 0. The log showed:

```text
CAEStreamParser::SyncDTS - dts stream detected
(6 channels, 48000Hz, 16bit BE, period 512, syncword 0x7ffe8001)
CAESinkAUDIOTRACK::Initializing ...
format: AE_FMT_RAW (AE)
method: RAW (PT)
stream-type: STREAM_TYPE_DTS_512
mi_decoder_open: format 9
mi_common_raw_open ... device[speaker]
```

The media session was active with no playback error. The HAL opened the hardware raw decoder path; this is the same broad class of path used by the current Kodi RAW passthrough setting.

## 4. Passthrough disabled: decoded PCM control

`audiooutput.passthrough` was temporarily changed from `true` to `false`, then the two raw files were played again.

### AC3 with passthrough off

```text
CDVDAudioCodecFFmpeg::Open() Successful opened audio decoder ac3
CAESinkAUDIOTRACK::Initializing ...
format: AE_FMT_FLOAT (AE)
method: PCM
stream-type: PCM-STREAM
mi_common_pcm_open ... device[speaker]
```

### DTS with passthrough off

```text
CDVDAudioCodecFFmpeg::Open() Successful opened audio decoder dca
CAESinkAUDIOTRACK::Initializing ...
format: AE_FMT_FLOAT (AE)
method: PCM
stream-type: PCM-STREAM
mi_common_pcm_open ... device[speaker]
```

Both streams played actively without a media-session error. The setting was restored to `true` after the test.

## 5. Kodi IEC-packer device condition

`audiooutput.passthroughdevice` was temporarily changed to:

```text
AUDIOTRACK:AudioTrack (IEC)|Kodi IEC packer (recommended)
```

With the raw `.ac3` and `.dts` files, Kodi emitted repeated:

```text
CAEStreamInfo::GetDuration - invalid stream type
```

This setting was not a valid controlled passthrough mode for these raw test files. It is not evidence that the hardware DTS path failed; it shows that the Kodi IEC-packer route does not accept the raw file input in this setup.

The original setting was restored:

```text
AUDIOTRACK:AudioTrack (RAW)|Android IEC packer
```

## 6. Physical output limitation

`dumpsys media.audio_policy` showed the primary output module advertises:

```text
Speaker
HDMI Out
HDMI ARC
Spdif
```

but the connected-device state used by the active STREAM_MUSIC route was:

```text
Devices: speaker
```

The HAL log also showed HDMI-ARC muted while speaker was unmuted. Therefore the successful DTS test above proves:

```text
DTS input detection -> RAW PT AudioTrack -> hardware decoder handle 9
```

on the currently selected speaker route. It does not prove:

```text
DTS IEC61937 burst -> physical SPDIF/HDMI -> receiver lock
```

## 7. What this changes in the static model

The runtime result supports a mode-dependent model:

```text
Kodi passthrough=true + RAW Android IEC
    -> raw DTS reaches hardware decoder/offload path

Kodi passthrough=false
    -> Kodi/FFmpeg dca decoder -> PCM speaker path

physical SPDIF/ARC route
    -> not exercised in this run
```

It also means the DTS problem, if it is still present on the projector/receiver, is not explained by basic DTS parsing or the initial hardware decoder opening. The remaining suspects are narrowed to:

- physical digital route selection / HDMI-ARC-SPDIF state;
- non-PCM channel-status configuration after RAW opening;
- SND/output-side packer behavior;
- the `AV+0x1CC`/SPDIF mode/source branch identified statically;
- differences between Kodi’s route and the native MStar route.

## 8. Final state and preservation

After testing:

```text
active players = none
audiooutput.passthrough = true
audiooutput.passthroughdevice = AUDIOTRACK:AudioTrack (RAW)|Android IEC packer
audiooutput.ac3passthrough = true
audiooutput.dtspassthrough = true
```

Temporary ADB forwarding was removed. No firmware/module/binary file was changed. No root escalation, reboot, flash, or persistent system audio property change was performed.

The next useful experiment would be to route the same RAW DTS test to the actual SPDIF/ARC hardware path and capture the SND/output-side state. That requires a physical digital route or the existing native/probe tooling; this report does not claim that route was tested here.
