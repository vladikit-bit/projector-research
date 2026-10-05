# Runtime — the private API call is reached, and it does not move the HAL

Date: 2026-09-26
Device: `192.168.0.183:5555`, MT5889, Android 11
Supersedes the open items 1–3 of `forensic_space_bunny_gate_live_test_20260926.md`
Evidence: `forensic_space_bunny_native_call_live_evidence_20260926.txt`

No patch, no APK modification, no module/vendor change.

---

## 1. The "hidden" control is not hidden

The Sound screen row titled **«Цифровий вихід»** is the `sound_spdif_type` preference
after all. In `SoundFragment.createPreferences()` that branch uses title
`2131755796` = `device_sound_digital_output`, entries `0x7f0300e2`, values `0x7f0300e4`
— the same generic table — and config key `g_audio__spdif`.

Its dialog is reachable, and the five items are:

```
Автоматично (Auto)  Bypass  PCM  Dolby Digital Plus  Dolby Digital
```

Selecting **Автоматично** wrote `g_audio__spdif = 7` (verified by the gate reading it
back within the same second). The `sound_digital_output` →
`g_fusion_sound__digital_autio_output` binding discovered earlier is a separate
preference; this one is the one the gate reads.

So the "impossible" precondition was reachable from the UI after all.

---

## 2. The gate armed and the native call executed — both branches

With `g_audio__spdif = 7`, flipping `Settings.Global.running_package_name` produced:

**Netflix starts** (`isNetflix = true`):
```
agent.tvmenusettingsservice: isNetFlix = com.netflix.ninja        isPreNetflix = com.bkdisplay.screensaver
TV_MtkTvConfig: Enter getConfigValue, inputGroup=-1, cfgId=g_audio__spdif
TVSettingConfig: getConfigValueInt configId:g_audio__spdif  value:7
TV_MtkTvAVModeBase: setAudioInfoValue type=0 val=5
TVNativeWrapper(Emu): handle MTK target case here
MtkTvAvMode_jni: Java_com_mediatek_twoworlds_tv_TVNative_setAudioInfoValue_1native
```

**Netflix stops** (`isNetflix = false`):
```
agent.tvmenusettingsservice: isNetFlix = com.google.android.youtube.tv  isPreNetflix = com.netflix.ninja
TVSettingConfig: getConfigValueInt configId:g_audio__spdif  value:7
TV_MtkTvAVModeBase: setAudioInfoValue type=0 val=7
MtkTvAvMode_jni: Java_com_mediatek_twoworlds_tv_TVNative_setAudioInfoValue_1native
```

No exception, no crash. **The MediaTek private API
`a_mtktvapi_audio_info_set_value` is live and accepts both 5 and 7.** This closes the
question left open by the parent report: the native call is not stubbed, not
permission-blocked, and not swallowed by the emulator gate.

---

## 3. But it changes nothing observable in the audio HAL

After the format was commanded (val=5, then val=7), DTS 5.1/1536 kb/s was played
through Kodi. `dumpsys media.audio_flinger` during playback:

```
- hdmi arc mode:  ui(auto),          current(auto)
- hdmi arc type:  ui(not-specified), current(not-specified)
- spdif mode:     ui(auto),          current(auto)
- spdif type:     ui(not-specified), current(not-specified)
- spdif delay:    0
Supported ARC capability: dd:0, ddp:0(atmos:0), dts:0, aac:0, mat-truhd:0, dtshd:0
Output devices: 0x2 (AUDIO_DEVICE_OUT_SPEAKER)
```

Then the projector's own OSD mode was cycled 2 → 1 → 0 → 2 (RAW → PCM → off → RAW)
with the HAL read after each step. **Identical output every time.** Neither the native
format command nor the projector menu moves `spdif_mode` or `spdif_type`.

Caveat, stated plainly: the audio route is `AUDIO_DEVICE_OUT_SPEAKER` throughout, so
the digital path may never have connected, and these fields are emitted at
`spdif connect`. This is a negative under the conditions tested, not proof that the
command is discarded — but the command *preceded* playback and the post-connect state
was still `auto`.

---

## 4. `g_audio__spdif` is not stable

The value was observed as `4` (session start), then `2`, `6`, `1`, and `7` after the UI
change — and it read back as `1` again at the end, minutes after the UI had set and
the gate had confirmed `7`. Repeated writes to the `action_value` row did not track
either:

| OSD row written | `g_audio__spdif` read after |
|---|---|
| 7 | 2 |
| 4 | 2 |
| 3 | 6 |
| 1 | 2 |
| 5 | 2 |
| 2 | 1 |
| 0 | 0 |
| (UI → Auto) | 7, then back to 1 |

There is at least one other writer. Until it is identified, `g_audio__spdif` cannot be
treated as a stable control surface, and no conclusion should be drawn from any single
reading of it.

---

## 5. Where this leaves the DTS question

The app-side chain is now **fully proven end to end**:

```
projector/Android UI  ->  g_audio__spdif  ->  TVMenuSettingsService.updatePackageChanged
                      ->  setSpdifFmt(v)   ->  setAudioInfoValue(0, v)
                      ->  TVNative JNI     ->  a_mtktvapi_audio_info_set_value   [LIVE, ACCEPTED]
```

What is **not** proven is that anything downstream of it acts on the value. The HAL's
two axes stay at `auto` / `not-specified`, and `Supported ARC capability` still reports
`dts:0`. That is consistent with the static model already built: the physical mode is
decided by the EDID-derived `_MI_AOUT_SetHdmiAutoMode` path in `libutopia.so` /
`utpa2k.ko`, not by the app-layer format request.

So the earlier layered assessment stands, now with the app layer *closed* rather than
merely located:

| layer | status |
|---|---|
| 1 — app command reaches the driver API | **PROVEN live, both branches, values 5 and 7** |
| 2 — HAL mode/type axes move | **not observed** (frozen at auto / not-specified) |
| 3 — EDID gate | open; `dts:0` in ARC capability; the `mik.ko` candidate sits here |
| 4 — decoder / Auth (`+0x4D8`, `0x582`, SHM38) | unproven |
| 5 — physical IEC61937/SDO transport, receiver lock | unproven |

The most valuable next experiment is no longer on the Java side. It is to get an actual
digital route active (so `spdif connect` fires and the axes are re-resolved) and then
repeat the format command against a live connection.

---

## 6. Device state left behind

| what | final state |
|---|---|
| `action_value.spdif_mode` | `2` (RAW/ARC) |
| `g_audio__spdif` | `1` (not the 7 that was set via the UI — see §4) |
| Sound screen «Цифровий вихід» | shows «Автоматично» (was «Пропустити») |
| `Settings.Global.running_package_name` | `com.bkdisplay.screensaver` (as found) |
| `Settings.Global.pre_package_name` | `null` (as found) |
| Kodi | force-stopped; the MP4 pushed to `/data/local/tmp` was removed |

**The one user-visible change to flag:** the «Цифровий вихід» row in Android Sound is
now «Автоматично» instead of «Пропустити». Restore it by reopening
`com.android.tv.settings/.device.sound.SoundActivity` and selecting «Пропустити».
