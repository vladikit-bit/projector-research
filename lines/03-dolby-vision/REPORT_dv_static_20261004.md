# DV — статичний аналіз двигуна, 2026-04 ніч

Запит: «продовж дослідження DV, але статику, а не калібрування».

---

## 1. Що є двигуном

`/vendor/lib/modules/kdrv_dolby_vision.ko` — **652 КБ, 961 символ** (не strip-нутий).
Це справжній Dolby Vision рушій. Разом із `dtv_driver.ko` (3.45 МБ, читає PQ ini) і
`utpa2k.ko` (25 МБ, PQ/аудіо).

## 2. ⭐ Головний висновок: рушій НІЧОГО не читає з файлів

```bash
strings -n 4 kdrv_dolby_vision.ko | grep -E '^[A-Za-z_]\w*:\w*$'
# -> лише "z:uA" (шум)
```

**Жодного ключа виду `секція:ключ`.** Порівняйте: у `dtv_driver.ko` їх ~160.

Отже `dolby.bin`, `dolby_factory.cfg` і будь-який «профіль» **не читає сам драйвер**.
Усе приходить через **ioctl з користувацького простору**.

**Наслідок — центральний.** Ланцюг «Dolby Vision → конфіг» виглядає так:

```
користувацький застосунок читає dolby_factory.cfg  ->  чого не робить ніхто
   (перевірено: grep -rla "dolby_factory" /vendor /system/lib64 -> лише HDR.ini)
   X
   (розрив)
   Y
kdrv_dolby_vision.ko отримує лише те, що надіслав userspace
```

Якщо користувацька сторона не надсилає параметрів дисплея — рушій працює на
значеннях за замовчуванням.

## 3. Які параметри рушій узагалі приймає

Повна поверхня налаштувань (з символів):

```
DoVi_GetSettings_CA          DoVi_GetSettings_R2Y / Y2R
DoVi_GetSettings_CC          DoVi_GetSettings_LB
DoVi_GetSettings_CSC         DoVi_GetSettings_TM      (tone mapping)
DoVi_GetSettings_Composer    DoVi_GetSettings_Composer_Dm
DoVi_GetSettings_DR          DoVi_GetSettings_Gamma / Degamma
```

**Це і є колірний конвеєр Dolby:** CSC — colour space conversion, R2Y/Y2R — переходи
між просторами, TM — тонмапінг, CA/CC — адаптація та корекція.

Реалізація видно в символах:
```
CommitMdsCsc / CommitMdsCsc_v31p      ← "Mds" = master display, керує CSC
DM2xAdaptor / DM2xAdaptor_v31p        ← Display Management 2, адаптер
g_au16DoViGammaPqLut
g_au32DoViDegammaPqLut / PqLut2 / HlgLut / SdrLut / PulsarLut
V8PrimartTbl                          ← таблиця primaries
GetPrimaries / GetPrimariesPtr
```

**`DM2xAdaptor`** — це місце, де Dolby Vision стискає колірний обсяг перед
висновком на панель. Саме тут логічно шукати наші втрачені 27% насиченості.

## 4. Що рушій читає з контенту (і чого там немає)

```c
DoVi_GetL11MetadataCB  u8ContentType=%u, u8WhitePoint=%u, u8WPValid=%u, ...
max_display_mastering_luminance = %d nits
min_display_mastering_luminance = %d nits
```

L11 — статичні метадані Dolby Vision. **Біла точка і максимальна яскравість
майстрингу — це характеристики ДЖЕРЕЛА, а не панелі.**

Тобто рушій знає, який контент прийшов, і **не має** даних про те, в яку панель
це йде. Профілю дисплея в ньому немає ніде, крім того, що може надіслати userspace.

Це узгоджується з виміром: DV зменшує насиченість на 27%, начебто обережно
втискаючи контент у невідомий гамут.

## 5. `DoVi_InitializeAndLoadTargetBin` і `commit_target_config`

```
DoVi_InitializeAndLoadTargetBin   "%s: ... fail, errorcode = %d"
commit_target_config / commit_target_config_dm3
```

Існує механізм завантаження «target»-конфігурації і коміт
(`dm3` = Dolby Vision Mode 3). **Але оскільки рушій не читає жодного файлу,
цей target приходить з userspace**, а не з `dolby.bin` напряму.

**Питання, яке треба вирішити далі:** хто в userspace надсилає `commit_target_config`,
і звідки він бере дані. Якщо відповідь — «ніхто», то рушій працює на дефолтах, і це
готове пояснення дефекту.

## 6. 🔴 Помилка, яку я зробив у цьому ж проході

Знайшов у dmesg живий інтерфейс:
`dolby_dmx`, `dolby_highcut`, `dolby_lowboost`, `dolby_drc`, `dolby_tb11`
і **вирішив, що `dmx` = Display Management X**, тобто відео-контрол.

Розшифрував рядок у модулі:
```
dolby dmx = %d  (0:LtRt, 1:LoRo, 2:Auto, 3:Arib)
```
**`dmx` — це downmix Dolby Atmos.** Разом з `high cut` / `low boost` / `drc` / `tb11`
це **аудіо**-параметри в `/proc/utopia_mdb/audio`. До кольору не мають жодного
стосунку.

**Я витлумачив вибірково:** побачив у списку знайоме слово й причепив до нього
бажану гіпотезу, не перевіривши розшифровку.

**Правило:** не називай параметр «тим, що схоже на назву», доки не розшифровано
його власним рядком-розшифровкою в модулі.

## 7. Побічний, але цінний знахідок

```
Dolby Vision DMAddr:0x%x CompAddr:0x%x DMEnable:%d CompEnable:%d
```

Модуль **знає адреси в DSP**, де лежать динамічні метадані та композитор.
Це вікно в пам'ять DSP — те, чого я earlier заявляв «немає».

---

## Що далі (статика)

1. **Знайти в userspace, хто надсилає `commit_target_config` і `DoVi_GetSettings_*`.**
   Найімовірніше — `libutopia.so` або MTK TV-бібліотека; ioctl-коди доведеться
   витягнути з диспетчера `kdrv_dolby_vision.ko`.
2. Якщо надсилає лише метадані контенту, а не параметрів дисплея — це і є доказ,
   що калібрувати нічого: рушій не знає про панель.
3. **Розібрати `DM2xAdaptor_v31p`** — чи є там явний коефіцієнт насиченості
   або gamut-compression, який можна побачити в коді.

Пункт 3 — найпряміший: якщо в коді видно, який саме параметр визначає стиснення,
ми знатимемо, що подавати, навіть без userspace.
---

# Продовження статики — ланцюг DV замкнено

## ✅ КОРЕКЦІЯ: секцію `[dolby_level]` ТАК читають

Я писав: «секція `[dolby_level]` не читається ніким — `dolby_level:dolby_3d_lut0`
відсутня у списку ключів `dtv_driver.ko`». **Це було узагальнення з одного модуля.**

Перевірка по всьому дереву знайшла читача:

```bash
grep -rla "dolby_3d_lut" /vendor/lib/modules  ->  mik.ko
```

```c
// mik.ko, ключ із форматуванням:
dolby_level:dolby_3d_lut%d

_MI_SYS_CfgLoadDolbyHdr3dLut   // завантаження LUT з ini
_MI_SYS_GetDolbyHdr3dLut
_MI_SYS_CfgSetDolbyHdr3dLut
_MI_DISP_HdrDolby3dLut
_MI_DISP_UpdateDolbyIqPictureQuality
_MI_DISP_SetDolbyApoEnable
mi_video_GetDolbyVisionInfo
```

**Повний ланцюг:**

```
HDR.ini  [dolby_level] dolby_3d_lut0
   -> mik.ko: _MI_SYS_CfgLoadDolbyHdr3dLut
   -> ioctl -> userspace
   -> kdrv_dolby_vision.ko: DoVi_InitializeAndLoadTargetBin  (файлів не читає!)
   -> DoVi_3DLUT_parsing_reorder_noHeader
   -> DV pipeline
```

`DoVi_InitializeAndLoadTargetBin` містить **жодного файлового виклику** —
лише `printk`, `memset`, `memcpy`, `init_cp`. Отже назва «LoadTargetBin» означає
«прийняти буфер», а не «відкрити файл».

## Структура конфігурації DV (з коду)

`DoVi_3DLUT_parsing_reorder_noHeader` обробляє **8 підструктур** через одну
підпрограму, зі зсувами в структурі:

```
0x1140   0x20A0   0x3000   0x3D80   0x4CE0   0x5A60   0x67E0
```

Глобали, які читає `DoVi_InitializeAndLoadTargetBin`:
`_u8NumViewModes`, `_ui_menu_params`, `_pq_config`, `_stDmContext`,
`_pstDmConfig`, `_bIsInitalized`, `run_mode`, `dm_ctx_buf`.

**`_u8NumViewModes`** — це той самий «View Mode 0–9», що в меню калібрування Android.
Отже target bin може містити N режимів перегляду з власним калібруванням.

## dolby.bin — формат

```
розмір 14 235, md5 5e2e0da96e9e16452dfae26db858f3d9
RLE: 0x01 x 565 | 0x02 x 193 | 0x03 x 116 | 0x04 x 85 | 0x05 x 69 | 0x06 x 57 ...
хвіст: 0xFF x 1876, 0xFF x 1578, 0xFF x 1578
```
спадний розподіл + насичені значення — таблиця квантування, не код.

---

# 🔴 ЧЕТВЕРТИЙ ВИПАДОК ОДНОЇ ПОМИЛКИ ЗА ВЕЧІР

| # | що | де помилився |
|---|---|---|
| 1 | 15.3% → 20.2% «Color Tuner працює» | різні кадри |
| 2 | «10544/10544» без застереження | вибірка лише 4-байтових |
| 3 | `dolby_dmx` = «Display Management» | назва без розшифровки |
| 4 | «`[dolby_level]` ніхто не читає» | узагальнив з одного модуля |

Спільне: **вузька вибірка або схожий напис → сильне твердження.**

## Охорона, яку впроваджую

1. «Перевірено N/N» супроводжується **вибіркою**: що саме входило.
2. «Ніхто не читає X» вимагає пошуку **по всьому дереву, усі модулі** —
   а не за ключем в одному з них. Практично: `grep -rla` по `/vendor/lib/modules`
   **і** `/vendor/lib`, `/vendor/bin`, `/system/lib*` разом, з переліком того,
   що перевірено.
3. Назву функції/параметра не використовувати як доказ змісту — тільки після
   дизасемблювання або рядка-розшифровки в самому модулі.
4. Висновок «X мертвий» вимагає **двох незалежних перевірок** (структурної
   і динамічної), інакше формулювати як «не знайдено, де саме».
