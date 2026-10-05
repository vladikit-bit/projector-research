# TD98 Pro / C50A — DAP Settings Runtime Check

**Date:** 2026-09-25  
**Device:** `192.168.0.183:5555`  
**Action under test:** user navigated/changed Android/projector `DAP Settings`.  
**Method:** read-only `settings`, `dumpsys audio`, `dumpsys media.audio_flinger`, `dumpsys media.audio_policy`, `logcat`, and `dmesg` capture. No settings were written by the test procedure.

## 1. Observed DAP changes

The TV configuration provider logged repeated writes:

```text
18:35:42.840  g_audio__dolby_sound_mode, value=9
18:35:42.883  g_audio__dolby_sound_mode, value=10
18:35:42.968  g_audio__dolby_sound_mode, value=2
18:35:43.752  g_audio__dolby_sound_mode, value=10
18:35:44.275  g_audio__dolby_sound_mode, value=9
18:35:44.799  g_audio__dolby_sound_mode, value=10
18:35:45.091  g_audio__dolby_sound_mode, value=2
18:35:45.353  g_audio__dolby_sound_mode, value=3
18:35:45.731  g_audio__dolby_sound_mode, value=4
18:35:46.225  g_audio__dolby_sound_mode, value=5
18:35:46.711  g_audio__dolby_sound_mode, value=8
18:35:47.263  g_audio__dolby_sound_mode, value=9
18:35:47.873  g_audio__dolby_sound_mode, value=8
```

The final Android global state is:

```text
sound_advanced_sound_mode=8
sound_advanced_dolby_ap=0
sound_advanced_dolby_atmos=0
sound_advanced_dialogue_enhancer=0
sound_downmixmode=1
sound_earc=0
hdmi_arc_control_enabled=1
```

The DAP UI therefore wrote a real TV configuration key, ending at `dolby_sound_mode=8`.

## 2. Runtime effect

AudioFlinger still reports:

```text
DAP: mcu 0
downmix_mode: 0
spdif mode: ui(auto), current(auto)
hdmi arc mode: ui(auto), current(auto)
Output devices: AUDIO_DEVICE_OUT_SPEAKER
Supported ARC capability: dd:0, ddp:0, dts:0, aac:0, dtshd:0
```

No `SetSpdifOutputMode=`, `SetHdmiArcOutputMode=`, `MDrv_AUDIO_SPDIF_SetMode`, nonPCM arming, or SPDIF ISR evidence appeared in the DAP capture.

## 3. Interpretation

`DAP Settings` is a Dolby audio post-processing/Dolby-sound-mode configuration path. Its writes are visible in the TV configuration provider, but the captured state shows no propagation into:

- the MI SPDIF mode writer;
- the HDMI-TX/eARC route;
- the sink EDID capability list;
- the physical SPDIF/ARC output state.

The final `dolby_sound_mode=8` is also the same value present in the earlier baseline dump, so this particular DAP navigation did not leave a distinct mode change relative to the prior state.

This menu is not the missing DTS unlock in the available evidence. It may affect Dolby/DAP processing, but it is separate from the projector `spdif_mode=2` (`RAW/ARC`) control and from the native `MApi_AUDIO_SYSTEM_Control` path.

## 4. Artifacts

Raw captures and hashes:

```text
C:\firmware_temp\runtime_dap_settings_20260926\
C:\firmware_temp\runtime_dap_settings_20260926\09_capture_hashes.txt
```

No patch, firmware write, module reload, reboot, or persistent setting write was performed by this DAP check.
