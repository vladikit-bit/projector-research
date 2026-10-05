# TD98 Pro / C50A — Non-Standard Adjust Runtime Check

**Date:** 2026-09-25  
**Device:** `192.168.0.183:5555`  
**Current user activity:** MediaTek factory `DesignMenu` → `AUDIO_nonStan`  
**Method:** read-only activity/window, TV-config logcat, settings, audio and AudioFlinger dumps. No setting write or patch was performed by the check.

## 1. Current Activity

The foreground task/activity is exactly:

```text
mediatek.factorymenu.ui/
  mediatek.tvsetting.factory.ui.designmenu.AUDIO_nonStan
```

The activity is hosted by:

```text
/system/system_ext/app/Factory/Factory.apk
```

This is the real `Non-Standard Adjust` screen, not a generic Android sound-effects page.

## 2. Values read on entry

On `AUDIO_nonStan` entry, the activity reads:

```text
g_audio__dolby_audio_processing
AUDIO_nonStan: currentDapEnableIndex = 0
g_audio__sound_mode
g_misc__high_deviation_mode
```

It does **not** read or write `g_audio__spdif` in the observed entry path.

## 3. Values written while navigating

The activity wrote the following TV configuration values:

```text
g_audio__sound_mode:
  4 -> 3 -> 2 -> 1

g_misc__high_deviation_mode:
  1 -> 2 -> 3 -> 2 -> 1 -> 0
```

All writes returned `0` from `TV_MtkTvConfig`.

The activity also generated the factory-menu broadcast:

```text
com.ntech.intent.action.VIRTUAL_EVENT_MODE_IR
```

## 4. Runtime audio state

After the Non-Standard Adjust navigation:

```text
DAP: mcu 0
downmix_mode: 0
spdif mode: ui(auto), current(auto)
hdmi arc mode: ui(auto), current(auto)
Output devices: AUDIO_DEVICE_OUT_SPEAKER
sound_earc=0
hdmi_arc_control_enabled=1
Supported ARC capability: dd:0, ddp:0, dts:0, aac:0, dtshd:0
```

No `SetSpdifOutputMode=`, `SetHdmiArcOutputMode=`, `MDrv_AUDIO_SPDIF_SetMode`, nonPCM arming, or SPDIF ISR trace was emitted.

## 5. Interpretation

`Non-Standard Adjust` is a factory Dolby/audio-processing screen. Its observed controls are:

- Dolby audio processing / DAP enable index;
- Dolby sound mode;
- high-deviation mode.

It is not the projector SPDIF mode control. The earlier `spdif_mode=2` (`RAW/ARC`) belongs to the separate projector OSD/TV provider path, while this factory activity operates on `g_audio__sound_mode` and `g_misc__high_deviation_mode`.

Therefore this screen is not, on the evidence captured so far, the missing DTS passthrough unlock.

## 6. Artifacts

Raw capture directory:

```text
C:\firmware_temp\runtime_nonstandard_adjust_20260926\
```

Selected evidence:

```text
01_activity.txt
02_window.txt
07_logcat_full.txt
```

No patch, firmware write, module reload, reboot, or persistent setting write was performed by this check.
