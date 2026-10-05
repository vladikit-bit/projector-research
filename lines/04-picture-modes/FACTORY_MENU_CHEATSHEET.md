# Шпора: інженерні меню TD98 Pro / C50A

**На пристрої два незалежні меню від різних фірм. Не плутати їх.**

| Меню | Застосунок | Хто зробив | Про що |
|---|---|---|---|
| **Factory Menu** (工厂老化模式) | `com.bk.factory` / `NtechFactory` | **Thundeal**, виробник пристрою | приймальне: геометрія, калібрування, прошивання, скидання |
| **Design Menu** | `mediatek.factorymenu.ui` | **MediaTek**, виробник чипа | PQ, Picture Mode, Dolby Vision, калібрування DV |

Підтверджено наживо 2026-10-03 через `dumpsys window`:
`mCurrentFocus=Window{... com.bk.factory/com.bk.factory.ui.FactoryActivity}`

---

# ⚠️ Не натискати ніколи

| Пункт | Переклад | Чому небезпечно |
|---|---|---|
| `一键校准` | Калібрування одним натисканням | **перезаписує заводську калібровку панелі** |
| `清除出厂数据` | Очистити заводські дані | **скидає все** |
| `恢复出厂数据` | Відновити заводські дані | **скидає все** |
| `Remove factory partition` | Видалити factory-розділ | **руйнівний, без відкату** |
| `.ui.VideoActivity` (окремий екран) | 老化模式 — тест старіння | **запускає стрес-відео на весь екран одразу** |

Останній пункт мене вже вбив у лапки двічі в різні дні — саме тому дивлюся
за `VideoActivity` насамперед у переліку, а не з точки входу.

---

# Factory Menu — `com.bk.factory`, пункт за пунктом

### Пункти

| Китайською | Переклад | Що це | Безпечно |
|---|---|---|---|
| 工厂老化模式 | Режим старіння/приймання | заголовок меню | так |
| 一键校准 | **Калібрування одним натисканням** | перезаписує заводську калібру панелі | **НІ** |
| 水平基准校正 | Коригування по горизонталі | keystone X, геометрія | так |
| 吊装基准校正 | Коригування підвісного монтажу | стельовий монтаж | так |
| 摄像头图片 | Знімок камерою | зображення з камери для звірки геометрії | так |
| 参数修正 | Коригування параметрів | | так |
| 投射比修正 | Коригування Throw Ratio | відстань проєктування | так |
| 亮度 | Яскравість | рівень 0–100 | **так, можна піднімати** |
| 显参 | **Параметри панелі** | вивід параметрів панелі | так, тільки читання |
| 主页切换 | Зміна домашнього екрана | який лаунчер відкриває HOME | так |
| u盘升级 | **Прошивання з USB** | оновлення прошивки | обережно |
| 清除出厂数据 | **Очистити заводські дані** | | **НІ** |
| 恢复出厂数据 | **Відновити заводські дані** | | **НІ** |
| 单色 | Одноколірна заливка | тест-кольорова пляма | так |
| 图片 | Зображення | тест-картинка | так |
| **Global Dimming** | **Глобальне затемнення** | див. окремо нижче | так, але не перевірено |
| 工厂调参 | **Заводське налаштування** | підменю | так, досліджуємо |
| 日志 | Логи | | так |
| 未知应用安装 | Встановлення невідомих застосунків | | так |
| 马达老化 | Старіння мотора | тест вентилятора | так |
| 风扇转速 | Оберти вентилятора | | так |
| 检测 | Перевірка: wifi / bluetooth / video | | так |

### Статусні рядки

| Рядок на екрані | Переклад | Що це |
|---|---|---|
| `eth : 74:25:84:D4:B8:2C 未连接` | eth — **не підключено** | MAC Ethernet-порту |
| `MAC Wifi: 3C:3B:AD:9A:64:11` | MAC Wi-Fi | адреса Wi-Fi |
| `wifi模块: AICSemi:AIC` | Модуль Wi-Fi: AICSemi / AIC | виробник чіпа |
| `8800D80(36225,42652)` | **WiFi-чіп AIC8800D80**, версії 36225 / 42652 | **НЕ** build-відбиток |
| `ADB: 开` | ADB: **увімкнено** | ось чому є root |
| `日志: 关` | Логи: вимкнено | |
| `未知应用安装: 关` | Встановлення невідомих: вимкнено | |

> **Виправлено раніше:** `8800D80(36225,42652)` — це не ідентифікатор прошивки і не
> чіпсет. Це **WiFi-чіп AIC8800D80**; підтверджено наявністю `aic8800_fdrv.ko`,
> `aic_load_fw.ko`, `aic_btusb.ko` у `/vendor/lib/modules/`.

Напівпрозорий текст на тлі (`SOUND_EHD_V1…`, назва файлу) — **службові оверлеї,
не пункти меню.** Не натискати.

---

# Global Dimming — що це і чому важливе

Це **не перемикач яскравості**, а окрема можливість Dolby Vision PQ.

На телевізорах вона керує підсвічуванням: рушій аналізує сцену й **затемнює підсвічування
у темних сценах**, щоб чорне виглядало чорнішим за рахунок контрасту, а не яскравості.

На проекторі «підсвічування» — це світлове джерело DLP. Тобто Global Dimming
регулює потужність лампи за сценою.

**Чому це кандидат на причину «молока».** Молоко — це піднятий чорний. Глобальне
затемнення мало б чорний **опускати**. Якщо воно активне без калібрувальних даних
панелі, воно може робити навпаки.

`dolby_factory.cfg` ставить `GlobalDimming = 1` для секції `Dark` і `0` для
`Bright` / `Vivid` — тобто це справді залежить від режиму.

**Що виміряно:** мінімум яскравості **17** (Bright) і **25** (Vivid) замість 0.

**Не перевірено.** Перемкнення — це зміна налаштування, питати спершу.

---

# Design Menu — `mediatek.factorymenu.ui`

Індекс = кількість натискань DOWN. **DPAD_CENTER (`23`) відкриває**, ENTER (`66`) — ні.

| # | Пункт | Стан |
|---|---|---|
| 1 | Factory Menu | перехід далі |
| 2 | **Picture Mode** | Dovi Dark / Dovi Bright; усі поля в нейтралі |
| 3 | Non_linear | не досліджено |
| 4 | **Non-standard options** | **не досліджено, кандидат на налаштування декодування** |
| 5 | SSC Adjust | |
| 6 | PEQ | |
| 7 | **Panel Info** | **відкривається ПОРОЖНЬОЮ** |
| 8 | Other Options | 14 пунктів, останній — `Dolby` |
| 9 | **Test Pattern** | вбудований, кращий за згенерований |
| 10 | Advanced Effect | лише DAP Setting |
| 11 | Execute Shell | **потрібен окремий дозвіл** |
| 12 | Remove factory partition | **руйнівний** |

### Panel Info — порожня

```
Panel Info
  Type      (порожньо)
  Version   (порожньо)
  Width     (порожньо)
  Height    (порожньо)
```

**Проєктор не знає, яка в нього панель.** Це корінь колірного ланцюга DV:
панель не описана → немає панельної гамми → немає калібрування → PQ-рушій нічого
не застосовує.

### Picture Mode — сторінка DV

```
Picture Mode      Dovi Dark      (або Dovi Bright)
Brightness        50
Contrast          50
Saturation        50
Hue               0
Sharpness         12
Backlight         50
```

Усі значення нейтральні, **однакові для Dovi Bright і Dovi Dark**, і список
**не прокручується** ні клавішами, ні свайпом. Тобто режими DV розрізняються
полями, яких **на цій сторінці немає**, і це **не** сторінка калібрування.

`picture_mode`: **6 = Dovi Dark**, 13 = Vivid. (Раніше я помилково шукав Dark
під номером 7 і через це двічі вирішив, що він «не застосовується».)

### ⛔ Калібрування Dolby Vision PQ — не натискати

Це найочевидніша наступна кнопка, і саме її не можна. Вона **пише реальний стан
панелі**. Діалог `EndUserCalibrationEditDialog` має власні пастки: ENTER усередині
**дописує цифру** замість підтвердження, а BACK **викидає все введене**.

Додати до списку небезпечних разом із `一键校准`, `清除出厂数据`, `恢复出厂数据`.

---

## Навігаційна небезпека

`uiautomator dump` на цих сторінках **не повертає атрибутів `focused` і `selected`** —
підсвічування живе лише в малюнку, не в дереві доступності. Перевірено: `focused="true"`
не збігається ніде.

**Наслідок:** сліпий `DPAD_CENTER` по списку з 12 пунктів реально небезпечний —
пункт 12 видаляє factory-розділ.

**Правило: користувач їде, я читаю.** Він керує пультом, я беру `uiautomator dump`
і `screencap`. Якщо все ж доводиться навігувати — **знімати скрінкап і дивитися
на підсвічену позицію перед кожним OK.**

Відходити BACK надто далеко теж погано: двічі я випадково випав у Factory Menu,
а одного разу — взагалі в Kodi, тричі турбуючи сесію.

---

## Головне з цього етапу

1. **Панель не описана** — `Panel Info` порожня. Корінь усього.
2. **Правильний колір на екрані можливий.** Панель меню відображається синім
   `#3280A0` правильно, в ту саму мить, коли літера N у DV-відео помаранчева
   `#8F3E02`. Отже дефект **вище** за дисплейний тракт, у декодуванні DV.
3. **DQ-контур живий** після патча `dv3`, помилки PQ зменшилися, але не зникли —
   `gbPQBinEnable` хтось скидає після ініціалізації.
4. **Обидва меню працюють.** `dv3` повернув інженерне меню, не зламавши налаштування Android.


---

# ОНОВЛЕННЯ 2026-10-03 (пізніше): три меню, джерела, DV-профілі, таблиця W/B

## Меню насправді ТРИ

| Меню | Застосунок | Хто | Що |
|---|---|---|---|
| 工厂老化模式 | `com.bk.factory` | Thundeal | приймальне, `一键校准` небезпечно |
| **Design Menu** | `mediatek.factorymenu.ui` / `DesignMenuActivity` | MediaTek | Picture Mode (Dovi Dark/Bright), Panel Info, Test Pattern |
| **Factory Menu** | `mediatek.tvsetting.factory.ui.factorymenu.FactoryMenuActivity` | MediaTek | **ADC Adjust, White Balance, OverScan, Other Options, Info** |

Design Menu → «Factory Menu» відкриває саме третій пункт. Шлях:
`am start -n mediatek.factorymenu.ui/mediatek.tvsetting.factory.ui.factorymenu.FactoryMenuActivity`

## Джерела масштабувача — 10, циклічні

```
HDMI 1   HDMI 2   HDMI 3   HDMI 4
ATV      DTV      Component   VGA   AV   VLC
```

**`VLC` — канал контенту Android TV.** Перелік джерел — це можливості драйвера,
не фізичні порти (фізично два HDMI + навушники + SPDIF + LAN).

⚠️ **Навігація в W/B Adjust: DOWN не рухає рядки**, рухає лише Source (LEFT/RIGHT).
Щоб дістатись до інших рядків, користувач керує пультом.

## ADC Adjust (Source = VGA)

```
R/G/B Gain  1509 / 1509 / 1509     (однакові — балансу немає)
R/G/B Offset 0 / 0 / 0
Phase 0
Auto Tune (Please waiting! ...)      <- активна операція, НЕ натискати
```

## Таблиця W/B по Color Temperature (Source = ATV, Picture Mode = Standard)

| CT | R Gain | G Gain | B Gain | R Off | G Off | B Off |
|---|---|---|---|---|---|---|
| Standard | 1023 | 1023 | 1023 | 1023 | 1023 | 1023 |
| **Warm** | 1025 | 1025 | **776** | 1025 | 1025 | 1026 |
| User | 1023 | 1023 | 1023 | 1023 | 1023 | 1023 |
| **Cool** | **774** | 1023 | 1023 | 1023 | 1023 | 1023 |

**Warm придушує синій на 24%, Cool — червоний на 24%.** Кожна комбінація
`Picture Mode × Color Temperature × Source` має власний набір значень — це і є
таблиця поканального калібрування, якої не було в файлах.

## Picture Mode пресети — білий баланс ідентичний у всіх шести

`Standard, Natural, Movie, Game, Energy Saving, AI PQ` — у всіх
`R/G/B Gain = 1023`, `R/G/B Offset = 1023`, `CT = Standard`.
**Пресети відрізняються не балансом білого, а гамою/тон-мапінгом.**

## Adjust Video vs Adjust UI — різні сторінки з різною нейтраллю

| | Adjust **Video** | Adjust **UI** |
|---|---|---|
| нейтраль | **1023** | **1024** |
| поля | R/G/B Gain, R/G/B Offset | R/G/B Gain, Contrast, Brightness, Hue, Saturation |
| додатково | — | **Reset**, **Pattern** (вбудований тест-патерн) |

`Adjust Video/UI` перемикається LEFT/RIGHT, коли фокус на цьому рядку.
**Увага: RIGHT/LEFT пробою змінюють значення, якщо фокус на слайдері, а не на назві.**

## DV-профілі — три, підтягуються під час відтворення

| профіль | де | значення |
|---|---|---|
| **Dolby Vision Vivid** | W/B Adjust (під час DV) | R/G/B Gain і Offset усе **1023** — нейтраль |
| **Dovi Dark** | Design Menu → Picture Mode | Brightness 50, Contrast 50, Saturation 50, Hue 0, Sharpness 12, Backlight 50 |
| **Dovi Bright** | Design Menu → Picture Mode | ті самі нейтральні |

**Усі три DV-профілі нейтральні.** Користувач нічого не крутив, а колір неправильний.

## Висновок по рівнях

| рівень | інструмент | чи діє на DV-колір |
|---|---|---|
| декодування DV | PQ-калібрування (11 полів) | **так — але порожнє (0.0)** |
| профіль DV | Dovi Dark/Bright, Dolby Vision Vivid | усе нейтральне |
| масштабувач (SDR) | W/B Adjust, Color Temperature, ADC | **ні** — DV вище |
| панель | Panel Info (порожня), panel_gamma.bin | не описана |

**SDR — еталон.** Логотип NETFLIX у SDR: `#F6000D` при еталоні `#E50914` (збіг
G=9, B=7, R=17). **Панель показує колір правильно.** Дефект — у декодуванні DV.

**Вбудований тест-патерн:** Design Menu → Test Pattern, і рядок `Pattern` на
сторінці Adjust UI.

**Записи замірів:** `runs/dvcal/factory/wb_by_ct.json`, `wb_by_preset.json`,
`runs/dvcal/dvprofiles/wb_by_dv_profile.json`, скрінкапи в `runs/dvcal/`.


---

# ДОБОВИТИЙ ВІД 2026-10-03 — інвентаризацію завершено

## MediaTek Factory Menu — повний склад

| пункт | вміст |
|---|---|
| ADC Adjust | R/G/B Gain **1509/1509/1509**, R/G/B Offset 0/0/0, Phase 0, **Auto Tune** (не натискати) |
| White Balance | див. таблицю W/B нижче |
| OverScan | Source, Right/Left/Top/Bottom — усі **50** |
| **Other Options** | `Mute Color (Blue)` Off · `Test Pattern For Panel` · `UART to HDMI` On |
| **Info** | SW 0001 · Name mt5889 · Build `mt5889_k32-**userdebug** 11 RP1A.200720.011 eng.wangzh.20250910.192054 dev-keys` · Board BD_MT5889_H2V1-B4-S · **Panel (ПОРОЖНЄ)** · Date Wed Sep 10 19:22:00 CST 2025 · PQ Standard Version **22529-202302** (= Dolby PQ v3.023) |

`Panel` порожній тут **і** в Design Menu → Panel Info. Панель не описана двічі.

## Design Menu → Test Pattern — 10 режимів

```
MVOP mode · ADC mode · XC IP mode · XC OP LINEBUFF mode · XC OP HVSP mode
XC VOP mode · XC MOD mode · **XC WHITE BALANCE mode** · VIDEO MUTE COLOR mode
```

**XC WHITE BALANCE** відкриває параметризований шаблон:
```
XC_WHITE_BALANCE enable   Range: 0 ~ 1
White Balance Radio       Range: 0 ~ 100
CANCLE / OK
```
⚠️ У діалозі **ENTER дописує цифру**, а **BACK стирає все введене** — не вводити без потреби.

## ⭐ Test Pattern For Panel — ГАЛЕРЕЯ СУЦІЛЬНИХ КОЛЬОРІВ

Factory Menu → Other Options → `Test Pattern For Panel`. Навігація
`Prev / Next`, масштаб `UP / DOWN` (100%).

| # | колір | виміряно через screencap |
|---|---|---|
| 1 | Червоний | `#FE0000` |
| 2 | Зелений | `#00FF01` |
| 3 | Синій | `#0000FE` |
| 4 | Жовтий | `#FFFF01` |
| 5 | Пурпуровий | `#FF00FF` |
| 6 | Бірюзовий | `#01FFFF` |
| 7 | Білий | `#FEFEFE` |
| 8 | Сірий | `#7D7D7D` |
| 9 | Сірий | `#7C7C7C` |

**Це готовий інструмент калібрування** — первинні, вторинні, білий і сірі з
відомими значеннями, плюс окремий регулятор балансу. Див. `DV_COLOUR_CALIBRATION.md`.

## Design Menu → Advanced Effect → DAP Setting — аудіо-DSP

```
DAP Setting Enable  OFF
DAP Mode           Standard
Gain · SurroundVirtualizer · Mi · CalibrationBoost · DapLeveLer · Modeler
Ieq · De · VolMaxBoost · Geq · Optimizer · Bass · Reg · Regulator   Paramter
```

**15 груп параметрів Dolby Audio Processing, увімкнення OFF.** Це аудіо-DSP, не
колір, але блок Dolby Audio існує і вимкнений.

## Design Menu → Other Options → Dolby

```
Banner              On
Compression         Line | RF          <- два значення
CompressionFactor   100
```

**Перевірено числами: `Line` і `RF` дають однаковий N байт-у-байт**
(`#903E02`, різниця R/G/B = 0.0). Стиснення DV **не є** причиною помаранчевої літери.

## Останнє з інвентаризації

Не відкрито: `Execute Shell` (потрібен окремий дозвіл) і
`Remove factory partition` (руйнівний). Не перебирав `Audio Input/Output Source`
на сторінці Volume (`Non_linear → Curve Type = Volume`).

---

# `Factory PQ Update` — як це працює насправді (розібрано з APK, 2026-10-03)

Джерело: `/system/system_ext/app/Factory/Factory.apk`, розбраний через
`baksmali.jar` (є в `/c/Android/crb_340/Binaries/apktool/`).

```
PQ Update Type   Bin | Ini | All      <- три значення, RIGHT/LINE
PQ Update        Please Click!        <- відкриває браузер файлів, НЕ кнопка Execute
```

## Три типи — `All` НЕ авто

| тип | що робить |
|---|---|
| `Bin` | `_updatePqBinFiles()` — копіює 18 `.bin` |
| `Ini` | `_updatePqIniFiles()` — копіює 7 `.ini` |
| `All` | **обидва** разом (операція OR), не авто-визначення |

## Списки файлів (точні)

**Ini** — рівно **три** файли (решта імен у рядках належать іншим методам):

| файл | куди пише (`dataIndex_1.ini`) |
|---|---|
| `ColorMatrix.ini` | `/vendor/cusdata/bsp/common/ColorMatrix/ColorMatrix.ini` |
| `NLA.ini` | `/vendor/tvconfig/config/PQ_general/NLA.ini` |
| `hdr.ini` | **`/vendor/cusdata/bsp/common/HDR/HDR.ini`** (6724 B) — не той 165-байтний, що в `PQ_general` |

Відсутній файл **не зупиняє** решту — просто пропускається.

### 🔴 `hdr.ini` — це і є двигун DV-тонмапінгу

```ini
[dolby_level]
dolby_3d_lut0        = "/vendor/tvconfig/config/HDR_BIN/dolby.bin"
dolby_factory_config = "/vendor/tvconfig/config/HDR_BIN/dolby_factory.cfg"
dolby_user_config    = "/data/vendor/3rd_rw/dolby_vision_iq/dolby_user.cfg"   # <- каталогу немає!

[tmo_curve]
PanelMaxLum = 3405     # 340.5 ніт
PanelGamma  = 22       # 2.2
input_PQ_10bits / output_TMO_nits   # LUT на 256 точок

[dv_apo_content_type_0..4]   FRC / NR / Sharpness за типом DV-контенту
```

**Отже я помилявся, сказавши, що `Ini` не діє на DV.** Він не заповнює 11 полів
калібрування — це так — але він **редагує тонмапінг DV**.

**Найкращий важіль із знайдених:** `dolby_user_config` вказує на
`/data/vendor/3rd_rw/dolby_vision_iq/dolby_user.cfg`, а **`/data/vendor/3rd_rw/`
існує і порожній**. Каталогу `dolby_vision_iq/` немає — це третій відсутній DV-артефакт
після `Main_CF.bin` і нульових 11 полів, але він у **`/data`**, тож редагується
**без флешки, без remount `/vendor` і без перезавантаження**.

### Флешка: потрібен FAT32, exFAT не працює

`/proc/filesystems` не містить `exfat`, немає `/system/bin/mount.exfat`, немає модуля.
Місця вистачить із запасом (3 файли ≈ 12 КБ).

> ⚠️ **`F:` переформатовувати НЕ можна** — там теки `tclsparse`, `v474`, `v098`, `v083`
> (прошивки TCL). Потрібна інша флешка, або спочатку скопіювати вміст на хост.

### `ColorMatrix.ini` існує — я помилявся

`/vendor/cusdata/bsp/common/ColorMatrix/ColorMatrix.ini`, 4096 B. Я шукав лише в
`/vendor/tvconfig` і оголосив відсутнім. Це третій випадок за сесію, коли `find`
був надто вузьким.

**Bin** — `Main*.bin`, `Sub*.bin`, `UFSC*.bin`, `HSY.bin`, `Bandwidth_RegTable.bin`
(18 файлів). Це заводські калібрувальні блоби, яких у нас немає і які неможливо
синтезувати. **Гілка `Bin` мертва остаточно.**

## ⚠️ Дві причини, чому нічого не відбувалося

**1. Вибір папки — окремий крок від самої установки.**
`onActivityResult` вимагає `requestCode == 3` і `resultCode == -1`, після чого сам
викликає `updatePqIniFiles()`. Отже: натиснули `PQ Update` → відкрився браузер →
обрали папку → **копіювання відбувається одразу при поверненні**.
Друге натиснення `PQ Update` просто відкриє браузер ще раз — це не кнопка Execute.

**2. після `am start` екран не має фокусу, тому не працює ЖОДНА клавіша.**
`onKeyDown` починається з `getCurrentFocus()`; якщо він null — id стає `-1` і всі
гілки мертві. Працюють тільки LEFT/RIGHT/CENTER, і кожна гілка додатково перевіряє
id сфокусованого елемента (`0x7f060121` — рядок **PQ Update Type**).

> **Наслідок: цей екран треба відкривати з батьківського меню, де вікно отримує
> фокус.** Через `am start` ним керувати не можна. Це пульт користувача, не мій.

Також `PQ Update Type` скидається на `Bin` при кожному новому екземплярі activity.

## Що вже підготовлено

`/storage/emulated/0/A_PQ/` — 6 наявних `.ini` (усі, крім `ColorMatrix.ini`,
якого немає ніде на пристрої). Назва `A_PQ` спеціально така, щоб папка sorting-ом
йшла **третьою** у браузері: два DOWN від `..` і OK — без прокручування.

## Корисний інструмент: `/data/mtk_fapi_debug`

Логи MediaTek factory-API (`MtkTvFApiDisplayBase`) **мовчать**, поки не створити цей
файл — і тоді видно кожен ini-шлях, кожне копіювання і геометрію панелі.

```bash
adb shell touch /data/mtk_fapi_debug
adb shell am force-stop mediatek.factorymenu.ui
adb shell am start -n mediatek.factorymenu.ui/mediatek.tvsetting.factory.ui.designmenu.FactoryPqUpdateActivity
adb logcat -d --pid=$(adb shell pidof mediatek.factorymenu.ui) -v brief
```

## 🔴 АКТИВНИЙ панельний INI — не той, що вважали

```
/vendor/cusdata/config/dataIndex/dataIndex_1.ini
  [panel] m_pPanelName = /vendor/cusdata/bsp/common/panel/CAFH016A_V3_60P_LVDS_JEIDA_SYNC_SPI.ini
```

У каталозі панелей **99** файлів; активний — `CAFH016A_V3…` (2026-09-16), решта
2025-09-10. Правило: **`dataIndex_1.ini` — єдине джерело правди**, не «найновіший»
і не «найбільший».

| поле | активний CAFH016A | PWM_INVERSE (був у записі) |
|---|---|---|
| `m_wPanelWidth/Height` | **1920 × 1080** | відсутнє |
| `m_bPanelPDP10BIT` | `1` | відсутнє |
| `m_ePanelLinkType` | `1` (LINK_LVDS) | `10` |
| `u16MaxPWMvalue` | **`0x1DF`** (479) | `0xFFFF` |
| `u16MinPWMvalue` | **`0x8F`** (143) | `0x2000` |
| `bPolPWM` | `0` = NON_INVERSE | (у назві INVERSE) |

**Підлога підсвітки = 143/479 = 29.9%, а не 12.5%.** `u32DutyPWM = 0x137` (311, 65%)
— типове значення. Це значно сильніший кандидат на «молоко», ніж 12.5%.

`[CUSTOMER_PQ]` (`CUSTOMER_PQ_BIN/Main_Color.bin`), `m_PanelBitNums = 2` (10 біт) і
криві `contrast 0x10 / brightness 0x74 / saturation 0x01` у обох файлах ** збігаються** —
ці висновки лишаються в силі.
