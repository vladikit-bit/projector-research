# FINDING — 2026-10-01: the video settings surface, mapped (keys, values, what is locked)

Source: Android TV Settings (`com.android.tv.settings` — note `com.android.settings` is **not
installed** on this box; the launcher hijacks `android.settings.SETTINGS` to the projector's own
`com.bkdisplay.settings`) → `partnercustomizer.picture.PictureActivity` →
«Розширені налаштування відео». Driven with `input keyevent` + `screencap`; screenshots in
`C:\firmware_temp\shots\`.

## 1. Every one of these is a plain integer in `settings` — namespace **`global`**

```sh
settings get global picture_mode      # -> 7
settings put global picture_mode 3    # writable as root
```

86 `picture_*` / `tv_picture_*` keys exist; full dump: `C:\firmware_temp\runs\ns_global.txt`.
Nothing here is opaque — this is the handle for the whole video feature set.

## 2. Three different "picture mode" words exist — the menu confusion explained

| key | value | what the UI showed |
|---|---|---|
| `picture_mode` | 7 | OSD (projector menu): **«Індивідуальний»** (=Custom) |
| `picture_mode_dolby` | 7 | the Dolby-specific variant of the same field |
| `picture_format` | 7 | format/resolution selector |

Android TV Settings showed **«Стандартний»** for the same conceptual item. So the OSD and the
Android page are reading **different words** — that is why the OSD offers only «Індивідуальний»
while the Android page offers real presets. The option **lists** live in the OSD/TvSettingsPlus
resources (`picture_mode_entries` / `picture_mode_entry_values`, plus `picture_mode_dovi_entries`
/ `_values`, `picture_mode_hdr10_ref_entries`, `picture_mode_filmmaker_hdr/sdr_ref_entries`,
`picture_mode_fun_entries`, `picture_mode_aipq_ref_entries`) — extraction not finished.

## 3. Full inventory of «Розширені налаштування відео» (30 items, vendor names + key + value)

Ukrainian labels on this build are a **poor machine translation**; the vendor's own names come
from the TvSettingsPlus resource pool (`TvSettingsPlus.apk!resources.arsc`, extracted to
`C:\firmware_temp\runs\arsc_strings.txt`). Vendor name is authoritative.

| # | Vendor name | Ukrainian label | value | key | modes / what it does |
|---|---|---|---|---|---|
| 1 | Color Temperature | Температура кольорів | 2 | `picture_color_temperature=2` | colour temperature preset |
| 2 | Dolby Vision Notification | Сповіщення Dolby Vision | on | `sound_dolby_notification=1` | **notification only**, not DV itself |
| 3 | (video noise reduction) | Динамічне зменшення шуму | Автоматично | `tv_picture_advance_video_dnr=4` | **Вимкнено / Низька / Середня / Висока / Автоматично** |
| 4 | (mpeg noise reduction) | Зменшення шумів MPEG | Середня | `tv_picture_advance_video_mpeg_nr=2` | **Вимкнено / Низька / Середня / Висока** |
| 5 | Sharpness | Найвища чіткість | Вимкнено | `picture_sharpness=4` | **Вимкнено / Увімкнено** (2 states only) |
| 6 | Adaptive Luma Control | Адаптивне керування яскравістю | Середня | `tv_picture_video_adaptive_luma_control=2` | DV adaptive luma; list pending |
| 7 | Local Contrast Control | Керування локальним контрастом | Середня | `tv_picture_video_local_contrast_control=2` | DV local contrast; list pending |
| 8 | **Dynamic Color Booster** | Dynamic Color Booster | Off | (not in settings) | **Off / Low / Middle / High** |
| 9 | Director Mode | Режим режисера | — | — | **увімкнено / Автоперемикання** — bypasses picture processing, honours mastering metadata |
| 10 | Flesh Tone | Відтінок шкіри | Вимкнено | `tv_picture_video_flesh_tone=0` | skin-tone retention |
| 11 | Film Mode | Режим кіно DI | Вимкнено | `tv_picture_video_di_film_mode=0` | film ("DI") mode |
| 12 | Blue Stretch | Вирівнювання синього | off | `tv_picture_video_blue_stretch=0` | toggle |
| 13 | Gamma | Гама | Середня | `picture_gamma=2` | gamma curve |
| 14 | Game Mode | Режим гри | **greyed** | `tv_picture_video_game_mode=0` | low-latency game mode — **works from the projector OSD**, blocked only in the Android UI |
| 15 | ALLM | ALLM | off | `tv_picture_video_allm=0` | Auto Low Latency Mode (auto latency; `allmSupported=true` in display info) |
| 16 | PC Mode | Режим ПК | **greyed** | `tv_picture_video_pc_mode=0` | PC mode — **works from the projector OSD** |
| 17 | De-Counter | De-Counter | Вимкнено | `tv_picture_video_de_counter=0` | de-judder |
| 18 | AISR | AISR | Низький рівень | `tv_picture_video_aisr=1` | AI super resolution |
| 19 | MJC effect (`picture_advanced_vedio_mjc_effect_entries`) | MJC | Вимкнено | `tv_picture_advance_video_mjc_effect=0` | **Вимкнено / Низький / Середина / Високий**; also **Demo split: Усі / Праворуч / Ліворуч** and a Demo mode. This is MEMC; controllable only when "automatic playback optimization" is off |
| 20 | HDMI RGB Range | Діапазон RGB для HDMI | — | `tv_picture_video_hdmi_rgb_range=0` | full/limited RGB |
| 21 | Low Blue Light | Слабке бліките світло | Вимкнено | `tv_picture_video_low_bluelight=0` | blue-light / halo reduction |
| 22 | Color Space | Колірний простір | Auto | `tv_picture_video_color_space=0` | output colour space |
| 23 | Automatic playback optimization | Автоматична оптимізація відтворення | off | `tv_picture_video_automatic_playback_optimization=0` | not the same as AFR (user's point); it gates MJC control |
| 24 | **Dolby Vision PQ Calibration** | Калібрування Dolby Vision PQ | — | see §7 | **View Mode 0–9** + 11 calibration fields |
| 25 | Light Sensor | Датчик світла | off | `tv_picture_video_light_sense=0` | ambient light sensor |
| 26–28 | 3D Mode / 3D-to-2D / L/R Switch | — | **greyed** | `tv_picture_advance_video_3d_mode/_3d_to_2d/_3d_lr_switch` | **the genuinely locked group** — SBS/Top-Bottom frame unpacking; see §6 |
| 29 | Color Tuner | Налаштування кольорів | — | `tv_picture_color_tune_*` | submenu: hue/sat/brightness per colour |
| 30 | **11 Point White Balance Correction** | Корекція балансу білого за 11 параметрами | — | `picture_white_balance11_*` | 11-point white balance |

Base «Зображення» page: **Picture Mode** = Користувацький / Стандартний / Висока чіткість / Спорт / Фільм / **Гра** / Енергозбереження (7 presets — `Standard`, `Sharpness`, `Sport`, `Film Mode`, `Game Mode`, `Energy Saver`, `Custom`), Підсвічування (greyed, `picture_backlight=50`), Автоматична яскравість (Ввімкнено, `picture_als=1`), Яскравість/Контраст/Насиченість (50/50/50), Відтінок/Tint (-2), Різкість (4).

> Naming collision worth knowing: the **picture preset "Гра"** = Picture Mode → `Game Mode` (a picture
> look), which is a **different feature** from the **greyed "Режим гри"** = low-latency
> `tv_picture_video_game_mode`.

## 4. What the platform now claims

```
DisplayDeviceInfo{ "Вбудований екран", 1920x1080,
   supportedModes [{1, 1920x1080, 60.000004}, {2, 1920x1080, 50.0}],
   HdrCapabilities mSupportedHdrTypes=[1, 2, 3, 4], mMaxLuminance=500.0, allmSupported true }
```
1 = Dolby Vision, 2 = HDR10, 3 = HLG/HDR10+, 4 = HLG. Peak 500 nit. Only two display modes:
**1080p60 and 1080p50** — that is what limits "AFR"/all operating modes.

Also present: `nrdp_platform_capabilities={"hdrOutputType":"notApplicable",
"formatNotificationType":"suppressible", …}` — the Netflix device-profile blob still declares
HDR output **not applicable** (separate from the panel's HDR types).

## 5. The open question that matters (deferred at user's request)

`/vendor/etc/media_codecs.xml` contains **zero** `dvhe` entries (verified), yet Kodi reported a
decoder `amc-dvhe(S) (HW)`. Where that codec comes from, and whether it is really hardware, is
the single most useful thing left to establish here — because if DV is **software**-decoded,
the entire DV menu above is cosmetic and would explain the user's two symptoms (wrong colours on
Kodi start, terrible fps).

## 6. Locked items worth attacking

Game Mode, PC mode, 3D Mode, 3D-to-2D are greyed out in the Android UI, but **Game and PC mode
really do activate from the projector's own OSD** (user-confirmed) — the lock is an Android-UI
side condition only. Only the **3D group is genuinely locked**: `tv_picture_advance_video_3d_mode`,
`…_3d_to_2d`, `…_3d_lr_switch`. That group is SRS/Top-Bottom frame unpacking — this is an LCD
projector with no 3D panel support declared anywhere, and no 3D source is connected; the exact
gate is unverified (most likely a source-capability check rather than a missing feature).

---

## 9. FINAL COMPLETE INVENTORY — every item, its options and what it is for

One clean pass through «Розширені налаштування відео», every submenu entered. Ukrainian labels
are a bad machine translation; the vendor names below are from the TvSettingsPlus resource pool.
**Only two lines of `settings` differed before/after the pass (the bookkeeping
`running_package_name`/`pre_package_name`), so nothing was changed by the walk.**

### A. Dolby Vision picture pipeline (the DV processing chain)

| Vendor name | Modes / contents | Key | What it is for |
|---|---|---|---|
| **Dolby Vision Notification** | on/off | `sound_dolby_notification=1` | Shows the DV banner on DV content. Notification only — not DV itself |
| **Dolby Vision PQ Calibration** | **View Mode 0–9** (10 slots) → **End-user calibration**: Tmax, Tmin, screen Gamma, Rx, Ry, Gx, Gy, Bx, By, Wx, Wy — **all 0.0** | (no key in `global`) | The DV display-calibration payload: white points, gamma, and the Rx/Ry/Gx/Gy/Bx/By/Wx/Wy primaries. **All zero = the panel was never measured**, so the PQ engine has no display reference and its tone mapping is undefined — the likely cause of wrong colours at Kodi start |
| **Dynamic Color Booster** | **Off / Low / Middle / High** (Off) | — | Dolby Vision's local colour/vibrance boost |
| **Director Mode** | **Director Mode / Автоперемикання** (auto-switch) | — | Bypasses picture processing, honours the mastering metadata. *Recommended: Автоперемикання*, paired with DCB Off and NR/local-contrast/adaptive-luma Low or Off |
| **Adaptive Luma Control** | **Off / Low / Medium / High** (Medium) | `tv_picture_video_adaptive_luma_control=2` | DV adaptive luminance |
| **Local Contrast Control** | **Off / Low / Medium / High** (Medium) | `tv_picture_video_local_contrast_control=2` | DV local contrast |
| (Dynamic) **Noise Reduction** | **Off / Low / Medium / High / Auto** (Auto) | `tv_picture_advance_video_dnr=4` | frame-to-frame noise reduction |
| (MPEG) **Noise Reduction** | **Off / Low / Medium / High** (Medium) | `tv_picture_advance_video_mpeg_nr=2` | compression noise removal |
| **Sharpness** | **Off / On only** (Off) | `picture_sharpness=4` | edge enhancement — only two states |
| **Flesh Tone** | **Off / Low / Medium / High** (Off) | `tv_picture_video_flesh_tone=0` | skin-tone retention |
| **Film Mode** | **Off / SLOW_PIC / ACTION_PIC** (Off) | `tv_picture_video_di_film_mode=0` | "DI" film motion mode |

### B. Colour management

| Vendor name | Modes / contents | Key | What it is for |
|---|---|---|---|
| **Color Temperature** | submenu: **Standard colours / Custom** + **Red / Green / Blue boost** (0/0/0, range down to −50) | `picture_color_temperature=2`, `picture_{red,green,blue}_gain=0` | white balance trim |
| **Color Tuner** | submenu: **Enable + Tint / Saturation / Brightness / Offset / Gain** → **31 controls**: hue, saturation, brightness **per 7 components** (R, G, B, Y, Magenta, Cyan, Flesh tone) + gain/offset per RGB — all 50 | `tv_picture_color_tune_*` (31 keys, all 50) | a full grading matrix; the manual substitute for the missing DV calibration |
| **11 Point White Balance Correction** | **Enable, Gain 5%, R 50, G 50, B 50** — despite the name, the 11 temperature points themselves are **not** exposed | `picture_white_balance11_{enable=1,gain=5,red=50,green=50,blue=50}` | white-point correction |
| **Color Space** | **Auto / Off / On / sRGB-BT.709 / BT.2020 / Adobe RGB / DCI** (Auto) | `tv_picture_video_color_space=0` | output colour space. **BT.2020 is the one Dolby Vision expects**; Auto currently lets the platform decide |
| **Gamma** | **Dark / Medium / Light** (Medium) | `picture_gamma=2` | gamma curve |

### C. Motion / gaming

| Vendor name | Modes | Key | Notes |
|---|---|---|---|
| **MJC** (MEMC) | **Effect: Off / Low / Medium / High**; **Demo split: All / Right / Left**; plus a Demo mode | `tv_picture_advance_video_mjc_effect=0` | motion interpolation. Controllable only while "automatic playback optimization" is off |
| **De-Counter** | **Off / Low / Medium / High** (Off) | `tv_picture_video_de_counter=0` | de-judder |
| **AISR** | **Low / Medium / High** (Low) — no "Off"** | `tv_picture_video_aisr=1` | AI super-resolution |
| **ALLM** | toggle (off) | `tv_picture_video_allm=0` | auto low-latency mode; `allmSupported=true` |
| **Game Mode** | **greyed out in Android — works from the projector OSD** | `tv_picture_video_game_mode=0` | low-latency game mode |
| **PC Mode** | **greyed out in Android — works from the projector OSD** | `tv_picture_video_pc_mode=0` | PC mode |

### D. The genuinely locked group + other switches

| Vendor name | state | key |
|---|---|---|
| **3D Mode / 3D-to-2D** | **DISABLED** | `tv_picture_advance_video_3d_mode` / `_3d_to_2d` |
| **L/R Switch** | toggle (off), enabled | `tv_picture_advance_video_3d_lr_switch=0` |
| **HDMI RGB Range** | **DISABLED**, value Auto | `tv_picture_video_hdmi_rgb_range=0` |
| **Blue Stretch** | toggle | `tv_picture_video_blue_stretch=0` |
| **Low Blue Light** | toggle (Off) | `tv_picture_video_low_bluelight=0` |
| **Light Sensor** | toggle (off) | `tv_picture_video_light_sense=0` |
| **Automatic playback optimization** | toggle (off) — **not AFR**; it gates MJC control | `tv_picture_video_automatic_playback_optimization=0` |

### E. Picture Mode presets (base page)

Custom / **Standard** / **Sharpness** / **Sport** / **Film Mode** / **Game Mode** / **Energy
Saver** — 7 presets. The Android page shows all 7; the projector OSD shows only «Індивідуальний»
because `picture_mode=7` sits outside that 0–6 range and the OSD's own list is shorter/different.
Base page also: Backlight (greyed, 50), Auto brightness (on), Brightness/Contrast/Saturation 50,
Tint −2, Sharpness 4.

**Naming collision:** the preset **«Гра»** (Picture Mode → Game Mode, a picture look) is a
*different feature* from the greyed **«Режим гри»** (low-latency `tv_picture_video_game_mode`).

`adb shell uiautomator dump /sdcard/ui.xml` returns the whole screen as XML with `text`,
`resource-id`, **`enabled`** and `focused` per node — enough to read values, option lists and
greyed-out rows without a picture. Stored values come from `settings list global`.

### «Калібрування Dolby Vision PQ» — CORRECTION to the earlier claim
I wrote that its key was "absent from settings, never initialised". **Wrong.** Entering it shows:
* **Режим перегляду: 0** — a selector with **10 modes, 0–9** (the viewing-mode slots);
* **Калібрування кінцевого користувача** — 11 numeric fields, **all 0.0**:
  **Tmax, Tmin, Гама екрана, Rx, Ry, Gx, Gy, Bx, By, Wx, Wy**
  — the standard Dolby Vision display-calibration payload (white points, screen gamma, and the
  Rx/Ry/Gx/Gy/Bx/By/Wx/Wy primaries). All zero = **the panel was never measured**.
* **Нещодавно змінений час: 01.10.26** — the page's own timestamp.

**Why this matters:** with no calibration data, the Dolby Vision PQ engine has no display
reference and its tone mapping is undefined — the most likely cause of the user's "wrong colours
at Kodi start". Filling it needs measured data (colourimeter) or the vendor's panel data; without
that, the DV path cannot be made correct, only bypassed.

### Other sub-dialogs
* **Режим режисера** (Director Mode): options = **увімкнено** / **«Автоперемикання»** (auto-switch).
  Recommended: Автоперемикання for everyday use — DV picture processing is bypassed and the
  mastering metadata honoured, and the setting reverts by itself for non-DV content. Pair it with
  Dynamic Color Booster Off, Dynamic NR Low/Off, local contrast and adaptive luma Off/Low, and
  «Стандартні кольори» for colour temperature.
* **Dynamic Color Booster**: Off / Low / **Middle** / High (current Off).
* **MJC** (= MEMC, motion interpolation): **Ефект: Вимкнено / Низький / Середина / Високий**;
  **Розділення демо-режиму: Усі / Праворуч / Ліворуч** (demo split side); **Демо-режим**.
  Requires «Автоматична оптимізація відтворення» to be OFF before MJC becomes controllable.

### AFR is probably NOT this setting
`tv_picture_video_automatic_playback_optimization` (Automatic playback optimisation) is a
separate feature; auto-frame-rate switching normally lives elsewhere (user's point). Worth
hunting for a distinct AFR control in the OSD/engineer menu before assuming this is it.

## 8. Display modes — the EDID line, and why it may touch DTS too

Only `1920x1080@60` and `1920x1080@50` are offered. The EDID binary is the source of that list,
and `cusdata/common/EDID_BIN` + `edid_bin/` exist locally. Multiple EDID revisions (1.4 / 2.0 /
2.1) are present per the user. **EDID also carries CEA audio capability descriptors** — if the
advertised audio set omits DTS, that is a direct, host-visible reason DTS passthrough could be
refused downstream. This joins the video detour back to the DTS thread and is the single
highest-value static target on this side.