# TD98 Pro / C50A — SPDIF & digital-output control surfaces: complete Java-layer trace

Date: 2026-09-26
Scope: MediaTek factory menu (`Factory.apk`), projector OSD (`NtechSettings.apk`),
Android TV Settings (`TvSettingsPlus.apk`), TV services (`TvServices.apk`), and the
twoworlds JNI layer, on device `192.168.0.183` (MT5889, Android 11).

**No patch has been applied. No device setting has been changed. No prior artifact was
modified.** All new artifacts live under
`C:\firmware_temp\factory_menu_analysis_20260926\`.

---

## 1. Bottom line

The SPDIF TX format command **exists, is live, and works** — but on this firmware it is
reachable from exactly one place in the whole system, and that place is gated behind a
**Netflix app start/stop transition**:

```
com.netflix.ninja starts or stops
  -> TVMenuSettingsService.updatePackageChanged()
       reads g_audio__spdif
       setSpdifFmt(5)  or  setSpdifFmt(7)      <-- hardcoded, user's choice NOT forwarded
         -> MtkTvAVMode.setAudioInfoValue(0 /*SPDIF_FMT*/, value)
              -> TVNativeWrapper.setAudioInfoValue_native
                   -> TVNative.setAudioInfoValue_native   (JNI, libcom_mediatek_twoworlds_tv_jni.so)
                        -> driver
```

`com.netflix.mediaclient` is the only Netflix package installed; the app the code
actually tests for, `com.netflix.ninja`, is **not present**. So this branch never runs,
which is exactly why `setAudioInfoValue` appears zero times in the AC3 and DTS playback
captures.

Consequence: **every user-facing "passthrough" control on this device is a dead end.**
The projector's SPDIF/ARC menu and the Android "Digital output = Bypass" row both write
TVConfig values that no code path forwards to the driver in a non-Netflix scenario.

---

## 2. The three user-facing SPDIF controls, and why none of them work

### 2.1 Projector OSD — `g_audio__spdif` (values 0..3)

```
AUDIO_nonStan-class OSD item "SPDIF/ARC"
  Lcom/bkdisplay/settings/model/action/a;      registers "ACTION_SOUND_SPDIF" -> a$132
  Lcom/bkdisplay/settings/model/action/a$132;  a(Map) -> index
  Lcom/bkdisplay/b/a/c;->m(int)Z
      ContentResolver.update("content://com.bkdisplay.system.provider/system/action_value",
                             "spdif_mode", index)
  -> SSP-SystemProvider -> MtkTvConfig.setConfigValue("g_audio__spdif", index)
```

Observed list order (logcat, `onUpdateData`): `off`, `PCM/ARC`, `RAW/ARC`, `AUTO`
→ indices `0, 1, 2, 3`.

Class members: `Lcom/bkdisplay/settings/model/action/a;`, `a$132;`,
`Lcom/bkdisplay/b/a/c;` (methods `K()I` reads `spdif_mode`, `m(I)Z` writes it).

### 2.2 Android TV Settings — `sound_spdif_type` (hidden)

`Lcom/android/tv/settings/device/sound/SoundFragment;->createPreferences()V` binds the
list preference key `sound_spdif_type` **directly to config `g_audio__spdif`**:

```
createListPreference(frag, "sound_spdif_type",
                     title  = 2131755796 (device_sound_digital_output),
                     entries= 0x7f0300e2, values = 0x7f0300e4,
                     configId = "g_audio__spdif")
```

Two label tables ship, chosen by `setupSPDIF_TypePref(ListPreference)`:

```java
int opt = TVSettingConfig.getMax("g_audio__spdif") - TVSettingConfig.getMin("g_audio__spdif");
MtkLog.d("SoundFragment", "spdifTypeOption is " + opt);
if (opt == 5) {                     // 6-valued range => MS12B table
    pref.setEntries(R.array.sound_spdif_type_entries_ms12b);        // 0x7f0300e3
    pref.setEntryValues(R.array.sound_spdif_type_entry_values_ms12b);// 0x7f0300e5
}
```

| table | labels | values |
|---|---|---|
| generic | Auto, Bypass, PCM, Dolby Digital Plus, Dolby Digital | **7, 4, 2, 5, 1** |
| MS12B   | Off, Dolby Digital, PCM, DDP Passthrough, DDP | **0, 1, 3, 4, 5** |

**Value-space collision:** both the OSD and this list write `g_audio__spdif`, but the OSD
means 0=off/1=PCM/2=RAW/3=AUTO while the Android generic table means 1=DD/2=PCM/4=Bypass/
5=DD+/7=Auto. OSD index 1 ("PCM/ARC") and Android value 1 ("Dolby Digital") are different
settings writing the same byte.

**Verified on screen:** `sound_spdif_type` is listed in `/vendor/etc/tv-settings-configs.xml`
but does **not** appear in the rendered Sound screen — it is filtered out by the display
logic. Live `am start com.android.tv.settings/.device.sound.SoundActivity` + `uiautomator
dump` shows no SPDIF-type row.

### 2.3 Android TV Settings — `sound_digital_output` (visible, already "Bypass")

```
Lcom/android/tv/settings/partnercustomizer/tvsettingservice/TVSettingConfig;
    SOUND_DIGITAL_OUTPUT = "g_fusion_sound__digital_autio_output"   (vendor typo "autio")
array sound_digital_output_entry_values = [0, 4, 3, 5, 1]
```

Live screen capture (`factory_menu_analysis_20260926/runtime/`):

```
eARC              summary = Автоматично        (Auto)
Цифровий вихід     summary = Пропустити        (Bypass -> value 4)
Затримка цифрового виводу  summary = 0
Режим зведення каналів     summary = Об'ємний звук
Керування динамічним діапазоном DTS  (DTS DRC)
```

So "Digital output" is already set to **Bypass (4)** and the user still gets no DD and no
DTS. That is consistent with §3: the value is stored and never forwarded.

### 2.4 The MediaTek factory menu has no SPDIF control

`Factory.apk` (`mediatek.factorymenu.ui`) bundles the full `com.mediatek.twoworlds`
library, which declares the SPDIF constants:

```
Lcom/mediatek/twoworlds/tv/MtkTvAVModeBase;
    AUDIO_INFO_SET_TYPE_SPDIF_FMT            = 0
    AUDIO_INFO_SET_TYPE_HP_DETECT_NFY       = 1
    AUDIO_INFO_SET_TYPE_ANDROID_APP_MUTE_AUDIO   = 2
    AUDIO_INFO_SET_TYPE_ANDROID_APP_UNMUTE_AUDIO = 3

Lcom/mediatek/twoworlds/tv/common/MtkTvConfigTypeBase;
    CFG_AUD_SPDIF          = "g_audio__spdif"
    CFG_AUD_SPDIF_DELAY    = "g_audio__spdif_delay"
    SPDIF_MODE_FMT_OFF                 = 0
    SPDIF_MODE_FMT_RAW                 = 1
    SPDIF_MODE_FMT_PCM16               = 2
    SPDIF_MODE_FMT_PCM24               = 3
    SPDIF_MODE_FMT_DDP_PASS_THROUGH    = 4
    SPDIF_MODE_FMT_DDP                 = 5
    SPDIF_MODE_FMT_AUTO2               = 6
```

A field-reference scan over **both** `classes.dex` and `classes2.dex` (scanner validated
with a passing control on `mSoundMode` and on the `CFG_AUD_SOUND_MODE` field) finds
**zero** references to `CFG_AUD_SPDIF`, `SPDIF_MODE_FMT_*`, or
`AUDIO_INFO_SET_TYPE_SPDIF_FMT`. The constants ship unused.

The screen the user opened, `AUDIO_nonStan`
(`res/layout/audio_nonstan.xml`), writes only:

```
g_audio__dolby_audio_processing    (read; gates the whole panel's enabled state)
g_audio__sound_mode                (written by onKeyDown)
g_misc__high_deviation_mode        (written by onKeyDown)
```

and drives factory-audio tables through
`MtkTvFApiAudio.setPrescaleByAudioSource / setNrThreshold /
setSpeakerDelayDefaultOffset / saveAudioIni`. Its `mOutputType` field
(`MtkTvFApiAudioTypes$EnumAudOutputType`) is used **only** as an index into
`getPrescaleByAudioSource` / `getSpeakerDelayDefaultOffset` — it is a factory EQ/latency
selector, not a route selector. No `MApi_AUDIO_SYSTEM_Control`, no `SetSpdifOutputMode`.

**Answer to the question asked: no, the factory menu does not contain a hidden SPDIF
control.** `NonStandardAdjustViewHolder` is a thin wrapper that opens `AUDIO_nonStan`.

---

## 3. The single consumer of `g_audio__spdif` in the entire system

`Lcom/mediatek/tv/agent/TVMenuSettingsService;->updatePackageChanged()V` is the only
method in the whole device that both reads `g_audio__spdif` and pushes a value to the
driver:

```java
String running   = Settings.Global.getString(cr, "running_package_name");
String pre       = Settings.Global.getString(cr, "pre_package_name");
boolean isNetflix    = "com.netflix.ninja".equals(running);
boolean wasNetflix   = "com.netflix.ninja".equals(pre);
...
Log.d("agent.tvmenusettingsservice", "isNetFlix = " + running + "   isPreNetflix = " + pre);
if (isNetflix != wasNetflix) {                       // <-- Netflix start/stop only
    int v = mSettingConfig.getConfigValueInt("g_audio__spdif");
    if (v == 7) {                                   // "Auto"
        setSpdifFmt(isNetflix ? 5 : 7);
    } else {
        setSpdifFmt(7);
    }
    return;
}
```

and the helper it calls is a one-liner in both APKs that carry it:

```java
// Lcom/mediatek/tv/agent/TVMenuSettingsService;      (TvServices.apk)
// Lcom/android/tv/settings/partnercustomizer/tvsettingservice/util/PreferenceConfigUtils;  (TvSettingsPlus.apk)
void setSpdifFmt(int value) {
    MtkTvAVMode.getInstance().setAudioInfoValue(0 /* AUDIO_INFO_SET_TYPE_SPDIF_FMT */, value);
}
```

Verified on device:

```
settings get global running_package_name -> com.bkdisplay.screensaver
settings get global pre_package_name     -> com.android.tv.settings
pm list packages | grep netflix          -> package:com.netflix.mediaclient
```

`com.netflix.ninja` is not installed, so `isNetflix` can never differ from `wasNetflix`
and the branch is unreachable.

Note also: even when it does fire, the value sent is the **hardcoded literal 5 or 7** —
the user's actual `g_audio__spdif` choice is only used as an `== 7` test, never
forwarded. And literal `7` is outside the shipped `SPDIF_MODE_FMT_*` range (0..6,
`AUTO2 = 6`); whether the native accepts 7 as an alias is unresolved and is listed in §7.

---

## 4. The JNI is live on this device (the "Emu" log tag is misleading)

```
Lcom/mediatek/twoworlds/tv/TVNativeWrapper;->is_emulator()Z
    VendorProperties.mtk_inside().orElse(Boolean.FALSE)
    if (value == true)  { Log.d("TVNativeWrapper(Emu)", "handle MTK target case here");  return false; }
    else                { Log.d("TVNativeWrapper(Emu)", "handle Emulator case here called"); return true; }
```

Device state: `vendor.mtk.inside = 1` → `is_emulator()` returns **false** → the real
native is called. The `TVNativeWrapper(Emu)` tag is only a log tag; the "handle MTK target
case here" line is the **real** path.

```java
int setAudioInfoValue_native(int type, int val) {
    if (is_emulator()) { Log.d("TVNativeWrapper(Emu)", "setAudioInfoValue_native..."); return 0; }
    return TVNative.setAudioInfoValue_native(type, val);   // real JNI
}
```

JNI entry point (verified in the pulled binary):

```
/vendor/lib/libcom_mediatek_twoworlds_tv_jni.so   (32-bit ARM, stripped)
  0x49231  Java_com_mediatek_twoworlds_tv_TVNative_setAudioInfoValue_1native  (60 bytes, THUMB)
  0x4926d  Java_com_mediatek_twoworlds_tv_TVNative_getAudioInfoValue_1native  (136 bytes, THUMB)
  NEEDED: libmediatek_tv_basic.so, vendor.mediatek.api.mtktvapi@1.0-cwrapper.so, ...
```

Runtime confirmation from the existing AC3 and DTS captures: the only audio-info call
ever logged is

```
TV_MtkTvAVModeBase: getAudioInfoValue type=3      (== ANDROID_APP_UNMUTE_AUDIO)
```

`setAudioInfoValue` never appears, and type `0` (SPDIF_FMT) is never queried — exactly as
predicted by §3.

---

## 5. No native component on the device consumes any TVConfig audio key

Device-wide greps (method validated with a positive control):

| pattern | hits in native code |
|---|---|
| `g_audio__spdif` | **0** |
| `g_audio__` (any) | **0** |
| `g_fusion_sound` (any) | **0** |
| `spdif_mute` (control) | `/vendor/lib/libutopia.so`, `/vendor/lib/modules/utpa2k.ko` |
| `c_scc_aud_set_spdif_fmt` (control) | `/vendor/lib/libmediatek_tv_rpcwrapper.so` |

All TV API services and libraries contain **0** occurrences of `g_audio__` /
`g_fusion_sound`:

```
/vendor/bin/hw/vendor.mediatek.tv.mtktvfactory@1.0-service
/vendor/bin/hw/vendor.mediatek.api.mtktvapi@1.0-service
/vendor/lib/vendor.mediatek.api.mtktvapi@1.0{,-cwrapper}.so
/vendor/lib/vendor.mediatek.custom.mtktvapicustom@1.0{,-cwrapper,-impl}.so
/vendor/lib/vendor.mediatek.tv.mtktvfactory@1.0{,-imp}.so
```

`g_audio__*` / `g_fusion_sound__*` exist only inside APKs, i.e. TVConfig keys are a
Java-only namespace. `/vendor/lib/hw/audio.primary.mt5889.so` likewise contains **0**
TVConfig key strings.

Second, independent SPDIF-format API found and set aside:

```
/vendor/lib/libmediatek_tv_rpcwrapper.so
  0x556ad  c_scc_aud_set_spdif_fmt   (172 bytes, THUMB; marshals via rpc_del/rpc_add_ref_buff)
  0x579e5  c_scc_aud_get_spdif_fmt
  0x5aff5  c_scc_aud_set_spdif_level
  0x5b14d  c_scc_aud_set_spdif_copy_protect
```

This is the SCC (TV-sound-over-RPC) audio path; a full-library scan finds **no caller**
inside the library itself, and it is unrelated to the local SPDIF TX.

---

## 6. Where the SPDIF state that *is* live actually comes from

`audio.primary.mt5889.so` is the only module on the device containing the string
`spdif mode`. It keeps **two orthogonal axes**:

```
spdif_mode   ∈ {AUTO, BYPASS, PCM, TRANSCODE}        (log strings SetSpdifOutputMode=*)
spdif_type   ∈ {AC3, DTS, NONE}                       (log strings SetSpdifOutputType=*)
hdmi_arc_mode∈ {AUTO, BYPASS, PCM, TRANSCODE}         (SetHdmiArcOutputMode=*)
hdmi_arc_type∈ {AC3, AC3P, DTS, NONE}                (SetHdmiArcOutputType=*)
plus: SetHdmiTxOutputMode=BYPASS,  set_ARC_format,  is_passthrough_active
```

and reports them as:

```
" - spdif mode: ui(%s), current(%s)"
" - spdif type: ui(%s), current(%s)"
"%s: spdif mode    = ui(%s), current(%s); spdif type    = ui(%s), current(%s); digital setting = Mode(%s), Type(%s)"
"%s: spdif connect" / "%s: spdif disconnect"
```

Our captures only ever showed the short mode line with both axes `auto`
(`spdif mode: ui(auto), current(auto)`, `hdmi arc mode: ui(auto), current(auto)`); the
`spdif type` axis was never separately driven from the app layer. This is the *same*
mode dispatch already mapped statically in earlier work
(`MApi_AUDIO_SYSTEM_Control` → `MDrv_AUDIO_SPDIF_SetMode`, Auto/Bypass/Transcode) —
so the HAL is a second, independent writer of the physical state, and it is the one that
is actually in control.

---

## 7. Open items (not claimed as resolved)

1. Whether the native `setAudioInfoValue(SPDIF_FMT, ...)` accepts the literal `7` that
   `updatePackageChanged` sends, given the shipped `SPDIF_MODE_FMT_*` constants stop at
   `AUTO2 = 6`.
2. What `SPDIF_MODE_FMT_RAW (1)` and `DDP_PASS_THROUGH (4)` actually do at the Utopia
   layer, versus the HAL's own `BYPASS`/`DTS` axes. The two enumerations are not
   reconciled.
3. Whether the MS12B or generic `sound_spdif_type` table is the one selected on this
   device — `setupSPDIF_TypePref` needs `getMax - getMin == 5` from the live TVConfig DB,
   and the shipped `res/raw/factory.db` is truncated (header claims 4347 pages, file has
   ~43), so the min/max could not be read statically. The row is hidden either way.
4. `res/raw/factory.db` and `res/raw/user_setting.db` remain unrecovered (SQLite reports
   "database disk image is malformed"; header page-count mismatch).
5. Nothing here has been exercised against a live DTS stream. Every runtime statement
   above is from already-captured logs or from a read-only screen launch.

---

## 8. Practical implication (no action taken)

The gap is now precisely located and it is **small**: the native SPDIF format command is
live, the value space is known (`SPDIF_MODE_FMT_*` 0..6, `RAW = 1`,
`DDP_PASS_THROUGH = 4`), and the single missing thing is a caller that forwards the
user's chosen value instead of a Netflix-gated hardcoded literal.

Candidate patch points, in increasing order of invasiveness — **all require explicit
approval and none has been touched**:

1. `Lcom/mediatek/tv/agent/TVMenuSettingsService;->updatePackageChanged()` in
   `TvServices.apk` — remove the Netflix gate and forward `g_audio__spdif` directly.
2. Hook the OSD's `spdif_mode` write (`Lcom/bkdisplay/b/a/c;->m(I)Z` in
   `NtechSettings.apk`) to call `setSpdifFmt(mappedValue)`.
3. Leave Android alone and fix the HAL/`libutopia` auto-mode negotiation instead, per
   the earlier `_MI_AOUT_SetHdmiAutoMode` / `_u32CurEdidSupportList` analysis.

Options 1 and 2 both need a value translation, because the OSD's 0..3 space does not match
`SPDIF_MODE_FMT_*` 0..6 or the Android generic 1/2/4/5/7 space.

---

## 9. Artifacts produced by this investigation (all new)

Working directory: `C:\firmware_temp\factory_menu_analysis_20260926\`

```
apk/Factory.apk                                 71a99fba7146ec2f0ec74b490d35f51c23bdf7033c1e5ce19f5d9037b6a74c01
osd/NtechSettings.apk                          674c5351ca44fe9301f61449f86394bb06140a03e5c686e8674e0ca9daeb7013
osd/TvSettingsPlus.apk                         2ad6cd689feacbd1aa4cc143c8349e4f39867d6bdab957cdad4b1926c4633aea
osd/TvServices.apk                             e315c5d7fee71785ba29fcd4fc89fb6b8e641537f232c603419d3a2407ee95e5
osd/tv-settings-configs.xml                    1ff92cf003910c8101d03e1dc1c4c5a7123e229d701530d7071b24b277faea96
native/libcom_mediatek_twoworlds_tv_jni.so     90c356b8023a6d3c6d100fbecf44245d99d0b769b6df34ded4e3aa7142688bdf
native/libmediatek_tv_basic.so                 381a77f2e2000f43e9f33cc2f0c826e8d6dba52e4ec0270155972703f94a4f4b
native/libmediatek_tv_rpcwrapper.so            bf3096257b20389fd26265f1b4c955d26838e2cd52eef37d5d9fde3ca6384454
native/audio.primary.mt5889.so                 f801f2434f451839b369a8f2f618e8cc19e10227f73708f9dd74ff7b9c7b9543
```

Scripts (`scripts/`): `dex_strings.py`, `dex_fieldvals.py`, `dex_fielduse.py`,
`dex_refs.py`, `dexcallers.py`, `dexdis.py`, `dumpclass.py`, `resarrays3.py`,
`armdis2.py`.

Runtime captures: `runtime/sound_activity_logcat.txt`, `runtime/sound2_logcat.txt`
(both from a read-only `am start` of `SoundActivity`, closed again with
`am force-stop`; no settings were modified).
