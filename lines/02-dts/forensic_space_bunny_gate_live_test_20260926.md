# Runtime — the Netflix gate fires, but is doubly locked

Date: 2026-09-26
Device: `192.168.0.183:5555`, MT5889, Android 11
Corrects: §3 and §7 of `forensic_space_bunny_spdif_control_surfaces_20260926.md`
Evidence: `forensic_space_bunny_gate_live_test_evidence_20260926.txt`

No patch, no APK modification, no module/vendor change. The only device writes were
`Settings.Global` keys and one content-provider table, all itemised in §5.

---

## 1. Correction to the parent report

`updatePackageChanged()` does **not** have an `else` branch. The exact tail is:

```
00b2  [4] if-eq    v0, v2, +017h        // if (isNetflix == wasNetflix) skip everything
00b6  [4] iget-object  v1, mSettingConfig
00ba  [4] const-string v2, "g_audio__spdif"
00be  [6] invoke-virtual v1, v2, getConfigValueInt
00c4  [2] move-result v1
00c6  [2] const/4  v2, 7
00c8  [4] if-ne    v1, v2, +00ch        // if (v != 7) jump past BOTH setSpdifFmt calls
00cc  [4] if-eqz   v0, +007h            // if (!isNetflix) skip the "5"
00d0  [2] const/4  v0, 5
00d2  [6] invoke-virtual v6, v0, setSpdifFmt
00d8  [2] goto +4h
00da  [6] invoke-virtual v6, v2, setSpdifFmt
00e0  [2] return-void
```

Correct semantics:

```java
if (isNetflix != wasNetflix) {
    int v = getConfigValueInt("g_audio__spdif");
    if (v == 7) {                          // <-- precondition, no else
        setSpdifFmt(isNetflix ? 5 : 7);
    }
}
```

The parent report stated `else { setSpdifFmt(7); }`. That is wrong. **The gate is
doubly locked**: it needs a Netflix transition *and* `g_audio__spdif == 7`.

This is not academic — it is the reason the first attempt produced no native call.

---

## 2. The trigger is `running_package_name`, not `pre_package_name`

`pre_package_name` is only read, for the comparison. The change observer is:

```
Lcom/mediatek/tv/agent/TVMenuSettingsService$SettingObserver;->onChange
    Log.d("agent.tvmenusettingsservice", "KEY_RUNNING_PACKAGE_NAME:settingItemId = " + ...)
    -> access$1200 -> updatePackageChanged()
```

`pre_package_name` is *written* by
`MonitorActivityController.setPackageName(String)`.

The first attempt (flipping `pre_package_name`) produced **zero** log output. The
second attempt (flipping `running_package_name`) produced the expected log on both
transitions.

---

## 3. Gate verified live, both directions

```
09-26 16:00:30.763 1214 1579 D agent.tvmenusettingsservice: isNetFlix = com.netflix.ninja      isPreNetflix = null
09-26 16:00:36.138 1214 1579 D agent.tvmenusettingsservice: isNetFlix = com.bkdisplay.screensaver isPreNetflix = null
```

With the transition detected, the code proceeded to read the config:

```
D TV_MtkTvConfig: Enter getConfigValue, inputGroup=-1, cfgId=g_audio__spdif
D TVNativeWrapper(Emu): handle MTK target case here
D TV_MtkTvConfig: Leave getConfigValue
D TVSettingConfig: getConfigValueInt configId:g_audio__spdif  value:4
```

`TVNativeWrapper(Emu): handle MTK target case here` confirms `vendor.mtk.inside = 1`,
i.e. the real native path, read live.

Then **nothing**: no `TV_MtkTvAVModeBase: setAudioInfoValue`, no
`a_mtktvapi_audio_info_*` effect. Consistent with `v = 4 != 7` and the corrected
control flow in §1.

---

## 4. New finding: the OSD row and `g_audio__spdif` are decoupled stores

The projector's SPDIF menu writes into the `action_value` table
(`content://com.bkdisplay.system.provider/system/action_value`, key `spdif_mode`).
The Netflix gate reads the TVConfig key `g_audio__spdif`. These are **not the same
store** — with the OSD row at `2` (the `RAW/ARC` item), `g_audio__spdif` read `1`.

The provider is reachable from shell and accepts writes:

```
D SSP-SystemProvider: update uri:content://com.bkdisplay.system.provider/system/action_value, contentValues:[spdif_mode:5]
D SSP-SystemProvider: table:system/action_value, number:1
```

but the resulting `g_audio__spdif` values across repeated writes did **not** form a
single consistent mapping:

| OSD row written | `g_audio__spdif` read afterwards |
|---|---|
| 7 | 2 |
| 4 | 2 |
| 3 | 6 |
| 1 | 2 |
| 5 | 2 |
| 2 | 1 |
| 0 | 0 |

`0→0` and `1↔2` look like a real relation; the rest is inconsistent and may reflect
another writer racing the sync. **The mapping is NOT established.** Do not build on it.

The conclusion that survives: **the projector's SPDIF menu does not land the value the
Netflix gate reads**, so no projector-menu setting can satisfy `v == 7`. Only the
hidden Android `sound_spdif_type` list, whose `Auto` entry is `7`, can.

---

## 5. Everything changed on the device, itemised

| what | writes | final state |
|---|---|---|
| `Settings.Global.running_package_name` | flipped to `com.netflix.ninja` and back, ~6 times | `com.bkdisplay.screensaver` (as found at session start) |
| `Settings.Global.pre_package_name` | set to `com.netflix.ninja`, then deleted | `com.zeasn.whale.saas` — written by `MonitorActivityController.setPackageName` during normal use, not left by me |
| `action_value` table, key `spdif_mode` | 7, 4, 3, 1, 5, 4, 2, 0, 2 | `spdif_mode=2` (the `RAW/ARC` item) |
| `g_audio__spdif` (TVConfig) | not written directly; moved as a side effect | read `1`; it read `4` at the start of this test and I could not restore it |

Not touched: modules, `/vendor`, any APK, any setting other than the two
`Settings.Global` keys and the one provider row.

The OSD row was deliberately left at `2` so the projector stays in the `RAW/ARC` state
this investigation has been testing. The user should confirm the projector menu
visually shows `RAW/ARC`.

---

## 6. What is still open

1. **The native call is still unexercised.** Reaching it needs `g_audio__spdif == 7`,
   which no projector-menu setting can produce. The question "does the native accept
   the literal 7" remains unanswered.
2. The `action_value` → `g_audio__spdif` mapping is unexplained and possibly racy.
3. If the hidden Android list could be surfaced, choosing `Auto` (7) would arm the
   gate and let the pending no-patch test complete.
4. Everything in the parent report's §7 open-item list still stands, plus the fact that
   the `else`-branch error there was load-bearing for the diagnosis.
