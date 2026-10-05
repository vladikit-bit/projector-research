# FINDING — відео: хто реально керує налаштуваннями, і чому 3D сірий

**Дата:** 2026-10-01 · **Платформа:** TD98 Pro / C50A, MT5889, Android 11
**Статус:** ланцюг «UI → вендор» розкрито; DV-причина уточнена до версії PQ-пакета
**З інструментів:** `runs/dexdis.py` (декодер DEX), `runs/findm.py`, `patch_baseline/kan.py`

---

## 1. Головне

На цьому пристрої існують **два незалежні шари налаштувань**, і Android-меню
write-ить лише в один із них. Вендорний шар (`g_*`) воно **тільки читає**.

| Шар | Ключі | Хто пише | Стан |
|---|---|---|---|
| Android | `tv_picture_*` у `Settings.Global` | TV settings APK | ✅ працює |
| Вендорний | `g_video__*`, `g_fusion_picture__*` | лише вендорні застосунки | ⚠️ порожній |

Наслідок один на три речі: калібрування PQ «Зберегти» — no-op; `Color Space`
повертається у Auto на кожному старті; **3D Mode сірий**.

---

## 2. Доказ (живий експеримент)

```
# settings put global g_video__vid_3d_item 1     (і g_video__vid_3d_mode/to_2d/lr_switch)
# перезапустити com.android.tv.settings
```

Скріншот після цього: **«Датчик світла» увімкнено** (його ключ
`tv_picture_video_light_sense=1` прочитано), а **«3D Mode» і «3D-to-2D»
лишилися сірими**.

Висновок: `g_*` у `Settings.Global` застосунок **не читає**.

---

## 3. Статика: `isSupport3D` — нативний виклик, а не ключ

`AdvanceVideoFragment` (classes7.dex, `com.android.tv.settings.partnercustomizer.picture`)
має чотири методи 3D: `init3DStatus` (0x128ABC), `init3DimModeData` (0x1276F0),
**`isSupport3D` (0x12776C)**, `update3D` (0x128DF0).

```java
// isSupport3D(), 43 інструкції
MtkTvAVModeBase av = new MtkTvAVModeBase();
sb.append("isSupDeType=").append(av.getVideoInfoValue(8));
MtkLog.d("AdvanceVideoFragment", sb.toString());
if (av.getVideoInfoValue(<x>) != 0) { v1 = 1; goto ret; }
v0 = 0;
ret: return v0;
```

**Гейт = `MtkTvAVModeBase.getVideoInfoValue(8)`** — нативний запит у MediaTek TV
stack. Ніякого ключа.

### 3.1 Як `g_*` взагалі читається

```
TVSettingConfig.getConfigValueInt(configId)          // classes7.dex 0x140258
  └─ MtkTvConfigBase.getConfigValue(configId)        // JNI -> /vendor/lib/libcom_mediatek_twoworlds_tv_jni.so
```

`update3D` читає через нього `g_video__picture_mode`.

### 3.2 Як Android-меню пише

```
PreferenceConfigUtils.putSettingValueInt(resolver, key, value)   // classes7.dex 0x1422C8
  └─ Settings.Global.putInt(...)     <- contentResolver, key = tv_picture_*
```

**Жодного виклику `MtkTvConfig.setConfigValue` у шляху збереження немає.**
Тому калібрування PQ і не пишеться нічого — `EndUserCalibrationFragment`
рахує ключ (`tv_picture_video_end_user_calibration_tmax`) і йде далі.

---

## 4. Хто має ключ від вендорного шару — фабричне меню

У системі є два фабричні застосунки:

| Пакет | APK | Роль |
|---|---|---|
| `mediatek.factorymenu.ui` | `/system/system_ext/app/Factory/Factory.apk` | **MediaTek factory menu** |
| `com.bk.factory` | `/vendor/app/NtechFactory/NtechFactory.apk` | вендорне factory menu |

У `Factory.apk` повний 3D-конфіг:

```
CFG_3D_MODE_OFF / SIDE_SIDE / TOP_AND_BTM / LINE_INTERLEAVE / DOT_ALT
CFG_3D_MODE_FRM_SEQ / 2D_TO_3D / AUTO / REALD / SENSIO
CFG_3D_MODE_CHK_BOARD            <-- ймовірно те саме, що type=8
CFG_3D_TO_2D_LEFT / RIGHT
CFG_VIDEO_VID_3D_{MODE, TO_2D, LR_SWITCH, FPR, FLD_DEPTH, OSD_DEPTH, NR, DISTANCE, IMG_SFTY, PROTRUDEN, NAV, NAV_AUTO}
CFG_SCC_3D_TO_2D_OFF
```

`CFG_3D_MODE_CHK_BOARD` («check board» = перевірка плати) збігається з
`getVideoInfoValue(8)`. Тобто **3D треба вмикати саме через фабричне меню**,
а не через Android UI.

### 4.1 Живі екрани фабричного меню (перевірено запуском)

```
FactoryMenuActivity          -> ADC Adjust / White Balance / OverScan / Other Options / Info
  Info                        -> PQ Standard Version = 22529-202302      (!)
DesignMenuActivity           -> Factory Menu / Picture Mode / Non_linear /
                                Non-standard options / SSC Adjust / PEQ /
                                Panel Info / Other Options / Test Pattern /
                                Advanced Effect / Execute Shell /
                                Remove factory partition
FactoryDolbyActivity         -> Banner=On, Compression=Line, CompressionFactor=100
FactoryPqUpdateActivity      -> PQ Update Type=Bin, PQ Update="Please Click!"
```

---

## 5. DV: що реально встановлено (і що я помилився)

### 5.1 СПРОСТОВАНО — попереднє «розбіжність версій» було хибним

Я писав, що рушій PQ очікує стандарт 22529-202302, а пакет на пристрої
2019 року. **Це неправильно.** У ресурсах `Factory.apk` поруч лежать чотири
написи:

```
PQ Customer Version
PQ HSY Version
PQ Standard Version
PQ TMO Version
```

Це чотири **різні PQ-бінарники** (вони ж є в рядках `utpa2k.ko`:
`PQ_BIN_CUSTOMER_MAIN`, `PQ_BIN_HSY`, `PQ_BIN_STD_MAIN`, `PQ_BIN_TMO`).
Отже «PQ Standard Version» — це версія *завантаженого STD-пакета*, а не вимога
рушія. Дата `dolby_factory.cfg` (2019-05-14) — це дата створення самого
конфігу, а не версія формату PQ.

### 5.2 Підтверджено двома незалежними способами

**(а) UI.** Сторінка Info фабричного меню показує **рівно один** рядок про PQ:

```
PQ Standard Version   22529-202302
```

Три інші написи (Customer / HSY / TMO) **не виводяться**.

**(б) Файлова система — це сильніше.** Драйвер шукає ці імена (рядки в
`utpa2k.ko`): `Main.bin`, `Sub.bin`, `HSY.bin`, `Main_TMO.bin`, `Main_Ex.bin`,
`Sub_Ex.bin`, `Main_Text.bin`, `Sub_Text.bin`, `Main_Ex_Text.bin`,
`Sub_Ex_Text.bin`, `Main_TMO_Text.bin`, `UFSC.bin`, `UFSC_Text.bin`,
`Main_CF.bin`, `Sub_CF.bin`, `Bandwidth_RegTable.bin`.

Що реально лежить у `/vendor/cusdata/bsp/chip/pq/`:

```
Bandwidth_RegTable.bin   868        Main_TMO.bin         13060
HSY.bin                 914696     Main_TMO_Text.bin     3860
M7642_Bandwidth_RegTable.bin 844    Main_Text.bin        43220
Main.bin                903300     Sub.bin              245396
Main_Ex.bin             460436     Sub_Ex.bin             6468
Main_Ex_Text.bin         26116     Sub_Ex_Text.bin        5028
                                     Sub_Text.bin          17252
                                     UFSC.bin               3236
                                     UFSC_Text.bin          1556
```

**Клієнтського PQ-бінарника немає.** Ні `Main_CF.bin`/`Sub_CF.bin`
(`CF` — customer), ані будь-якого файлу в каталозі рівня плати:

```
/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/
    model/  model_1/  module/  voc/        <- каталогу pq/ немає, .bin — жодного
```

Перевірено на **обох** каталогах плани (спершу був пропущений другий варіант):

```
/vendor/cusdata/bsp/board/
    BD_MT5889_H2V1-B4-S/         model/ model_1/ module/ voc/
    BD_MT5889_H2V1-B4-S_EWS/     model/ model_1/ module/ voc/
find /vendor/cusdata/bsp/board -type d -name pq   ->  нічого
find /vendor/cusdata/bsp/board -name "*.bin"      ->  нічого
```

Тобто шару **`/vendor/cusdata/bsp/board/<board>/pq/` з клієнтським тонмапінгом
для цієї панелі на цій прошивці просто не існує**. Це не помилка завантаження і
не конфіг — це рішення збірки виробника.

*(Також розкрито, що `/vendor/tvconfig/config/aipq/*.bin` з GUID-іменами — це
не PQ-бінарники, а моделі розпізнавання сцени для AI; `aipq.ini` керує саме ними.)*

### 5.3 Що з цього випливає

| Питання | Відповідь |
|---|---|
| Чи можна це виправити налаштуваннями | **Ні** |
| Чи потрібне повне оновлення прошивки | **Ні** — достатньо додати каталог `board/<board>/pq/` з клієнтським бінарником |
| Що саме потрібно | `Main_CF.bin` / `Sub_CF.bin` (або еквівалент) + `dolby_factory.cfg` |
| Чи приймає платформа такий файл | **Так** — `FactoryPqUpdateActivity`, `PQ Update Type = Bin` |
| Звідки взяти | інший пристрій **з тією ж платою** `BD_MT5889_H2V1-B4-S`, у збірці якого шар `board/…/pq/` присутній |

Важливе уточнення щодо донорського пристрою: справа **не** в тому, телевізор це
чи проєктор, і не в тому, чи той самий SoC. Потрібна **та сама плата** і
збірка, де постачальник цю клієнтську PQ-шар поклав. Телевізори на інших
платах (напр. TCL C825/C728 на MT9615) не підійдуть — у них інший board ID і
інші дані панелі.

---

## 6. Бонус: збірка — `serdebug`

Factory Info: `mt5889_k32-u / serdebug 11`, board `BD_MT5889_H2V1-B4-S`.
Це userdebug-збірка — саме тому доступні root через adb і фабричне меню.

---

## 7. Що зроблено і повернуто

Під час перевірки змінено і **повернуто**: `g_video__vid_3d_*` (видалено),
`tv_picture_video_light_sense` (1 → 0), `screen_off_timeout` (лишився
збільшеним — повернути до 600000). `tv_picture_video_color_space` і так
повертається у 0 після ребута.

## 7.1 Інструмент

`runs/dexdis.py` — коректний декодер DEX-коду (таблиця ширин інструкцій,
розбір `class_data`, посилання на рядки/поля/методи). `runs/findm.py` —
самоперевірний пошук методу за індексом (обходить розбіжність обходу класів).

---

## 6. НОВИЙ ВІДКРИТТЯ (2026-10-01, після тестів GRule) — напрям змінився

### 6.1 Результати `PQGruleTest` (штатний дебаг-інтерфейс драйвера)

`echo DebugLevel=ALL > /proc/utopia_mdb/pq`, далі
`echo PQGruleTest=<type> window=0 leLevelIndex=0 > /proc/utopia_mdb/pq`:

| type | тип GRule | результат у dmesg |
|---|---|---|
| 0 | SWDRModeGRule | тиша |
| **1** | **TMOModeGRule** | **`MDrv_PQ_LoadTMOModeGRuleTable_U2,21899: u16PQ_TMOMode_Idx == PQ_INVALID_QMAP_INDEX`** |
| 2 | UCDModeGRule | `[_MDrv_PQ_GetTMOGRuleTableInfo,3875]: [PQ] GroupNum: 8, IPNum: 1` + `PQ table type: DumF series` |
| 3 | EnableHDRMode | тиша |
| 4 | EnableGAMEMode | тиша |
| **5** | **PictureMode** | **`MDrv_HSYBin_GetGRuleParse,5446: GRule found!`** + `HSY Rule found! Label:[2080811 400000f]` |
| 6 | GAMUTMAPPING | тиша |
| 7 | AVCMode | тиша |

**TMO — це тонмапінг.** Його індекс невалидний, тоді як UCD- і HSY-таблиці
працюють. Тобто рушій працездатний, проблема не в ньому і не в відсутності
таблиць — **немає привʼязки режиму**.

### 6.2 Гіпотеза, яку можна перевірити

`dolby_factory.cfg` визначає рівно три секції режимів:

```ini
[PictureMode 0] PictureModeName = Dark    (Tmax 281, GlobalDimming 1)
[PictureMode 1] PictureModeName = Bright  (Tmax 140)
[PictureMode 2] PictureModeName = Vivid   (Tmax 210)
```

Вендорне **Picture Mode зараз = `User`** (сторінка Picture Mode у factory menu).
Жодної секції `User` у конфігу немає — звідки й `PQ_INVALID_QMAP_INDEX`.

**Якщо привʼязати PQ до індексу 0/1/2, тонмапінг може запрацювати** — і тоді
DV Calibration (`Set E_PQ_XC_IP_TMO Customer Data fail`) теж має зникнути.

### 6.3 Що не вдалося

Змінити Picture Mode зі сторінки factory menu не вийшло: ані ENTER (23), ані
LEFT/RIGHT на рядку «Picture Mode» значення не змінюють — лишається `User`.
`settings put global picture_mode N` PQ не бачить (жодного рядку в dmesg) —
що узгоджується з §3.2: Android-шар не доходить до вендорного.

**Наступний крок:** знайти, звідки PQ бере індекс TMO-режиму
(`MDrv_PQ_LoadTMOModeGRuleTable_U2`, `u16PQ_TMOMode_Idx`), і зрозуміти,
як його задати 0/1/2. Це вже конфігураційний шлях, а не патч.

---

## 7. Уточнення після перевірки (2026-10-01, пізніше) — два хибні читання відкликано

### 7.1 Хибне: «Picture Mode не змінюється»

Значення **змінюється** клавішами LEFT/RIGHT на рядку «Picture Mode»
у factory menu. Цикл режимів (6 штук):

```
Standard → Natural → Movie → Game → Energy Saving → AI PQ → (на початок)
```

Значення читається з UI-дерева: node з `bounds="[266,66]"` містить текст режиму.
(Раніше я начебто не побачив зміни, бо дивився на застарілий кадр.)

### 7.2 Хибне: «TMO привʼязується в Energy Saving / AI PQ»

`PQGruleTest` — **одноразова** команда: вона друкує лише коли стан реально
змінюється. Повторний виклик без зміни стану не друкує нічого. Мій «перебір
режимів» просто не міняв стан, тому тишу я прочитав як успіх.

Контрольний A/B (режим змінюється перед кожним тестів) — стабільно:

```
Movie → Not support this type = 0 | u16PQ_TMOMode_Idx == PQ_INVALID_QMAP_INDEX
Movie → Window[0] enSWDRGRuleLevelIndex is 0 | Not support this type = 0 | INVALID_QMAP
(повторюється щоразу)
```

**Висновок:** TMO-індекс невалідний **стабільно**, у всіх режимах, які вдалося
перевірити. Жодного режиму, де він стає валідним, не виявлено.

### 7.3 Також хибне: «рівні 1 і 2 валідні»

`leLevelIndex=1/2` — це параметр *тестової* команди, тобто прохання драйверу
оцінити інший рівень. Він нічого не змінює в робочому `u16PQ_TMOMode_Idx`.

Справжній факт: коли драйвер оцінює рівень 1 або 2, він **знаходить** групу —
`[PQ] GroupNum: 7, IPNum: 3`, `PQ table type: DumF series`. Тобто **TMO-таблиця
в бінарнику є**. Але під час роботи рушій просить рівень **0**, а на нього
запису немає.

Це добра новина і вона змінює картину: справа **не** у відсутності даних, а в
тому, що вибирається рівень 0. `dolby_factory.cfg` має лише
`ReferenceDarkPicModeIndex = 0` та `DoViBrightPicModeIndex = 1`, секції
Dark/Bright/Vivid.

**Що лишається невідомим:** який саме селектор живить `u16PQ_TMOMode_Idx`, і
як його перевести на 1 або 2 під час роботи. `picture_mode` (0/1/2),
`picture_mode_dolby` (0/1/2) — не перемикають його (нуль рядків у dmesg).

---

## 8. ЗВІДКИ, ЩО ЖИВИТЬ `u16PQ_TMOMode_Idx` (статично, udpa2k.ko)

### 8.1 Ланцюг викликів

```
MDrv_PQ_General_GruleTable_U2  (0xd54dc)  -> MDrv_PQ_SetQmapGRuleLevelIndex
MDrv_PQ_LoadTMOModeGRuleTable_U2 (0xb77dc) -> MHal_PQ_GetTMOModeLevelIndex
```

`MDrv_PQ_SetQmapGRuleLevelIndex` @0x93ABC (40 Б) — тривіальний сеттер:

```asm
0x93abc  cmp  r1, #1
0x93ac0  bxhi lr
0x93ac4  rsb  r1, r2, r2, lsl #4       ; devId*15
0x93ac8  movw r2, #0 / movt -> _gu16QmapGRuleLevel
0x93ad4  add  r1, r2, r1, lsl #1
0x93ad8  add  r1, r1, r3, lsl #1
0x93adc  strh r0, [r1]                 ; _gu16QmapGRuleLevel[devId][k]
```

Глобал: **`_gu16QmapGRuleLevel`**.

### 8.2 `MHal_PQ_GetTMOModeLevelIndex` @0xE27B4 (632 Б)

`switch` на другому аргументі (14 випадків, 1..14). У гілках читаються:

```asm
movw r0, #0 / movt -> _stMultiMedia_Info
ldr  r0, [r0, r5, lsl #2]     ; _stMultiMedia_Info[devId]
cmp  r0, #2                    ; -> HDMI
cmp  r0, #3                    ; -> ADC
cmp  r0, #4                    ; -> MM
...
movw r0, #0 / movt -> _enInputSourceType
ldr  r0, [r0, r5, lsl #2]     ; _enInputSourceType[devId]
sub  r0, r0, #9
cmp  r0, #3                    ; типи 9..12
movhs r6, #2                   ; >= 12 -> 2, инакше 5
```

Значення `_stMultiMedia_Info` зі рядка самого модуля
`Main Input Source: %x (0:DTV, 1:ATV, 2:HDMI, 3:ADC, 4:MM)`:
**2 = HDMI, 3 = ADC, 4 = MM**.

### 8.3 Висновок

**Рівень TMO-тонмапінгу визначається джерелом сигналу, а не режимом
картинки.** Гілка HDMI / ADC / MM + тип джерела 9..12.

Наша перевірка показала, що коли драйвер оцінює рівень 1 або 2, група
**знаходиться** (`GroupNum: 7, IPNum: 3`, `PQ table type: DumF series`).
Під час роботи рушій просить рівень, який дає `PQ_INVALID_QMAP_INDEX`.

**Перевірюваний висновок:** проблема привʼязана до **внутрішнього джерела MM**
(лаунчер Android), де тип джерела не потрапляє у вікно 9..12, яке потрібне
шляху TMO. Dolby Vision приходить по HDMI, тобто **на зовнішньому HDMI-вході
рівень може розв'язатися** — і тоді зникнуть і `PQ_INVALID_QMAP_INDEX`,
і `Set E_PQ_XC_IP_TMO Customer Data fail`.

**Це перевіряється одним тестом:** увімкнути HDMI-вхід і запустити DV-контент
із зовнішнім джерелом, подивитися dmesg. Пристрої не змінював.

Також у модулі є прямі підказдки про ту саму логіку:
`[PQ] ... Input source type: %d, Non HDMI/MM source should not ...` і
`[PQ] ... input source DV std., should not open game mode`.

---

## 9. Механізм активації DV і умови перемикання

### 9.1 Відео-ліцензія — мертвий код (перевірено)

| Символ | Адреса | Посилань у модулі |
|---|---|---|
| `MDrv_SYS_GetDolbyKeyCustomer` | 0x25AF0 | **0** |
| `MDrv_SYS_QueryDolbyHashInfo` | 0x25AEC | **0** |
| `MDrv_SYS_GetDolbyKeyCustomer.u8gDolbyKeyCustomer` | .bss, 24 Б | **0** |

`MDrv_SYS_GetDolbyKeyCustomer` просто копіює 16 байт із `u8gDolbyKeyCustomer`
(+4…+0x13) у буфер виклику. Саме цей масив **ніколи не заповнюється** — жодного
запису. Рядок `Dolby efuse status : FALSE` приходить зі статус-дампа, а не з гейта.

**Висновок:** DV на цій платформі **не чекає на efuse/хеш**, на відміну від
аудіо-ліцензії (див. DTS-лінію). Цю гіпотезу можна зняти.

### 9.2 Як активується Dolby Vision

`mdrv_HDMI_ParseDolbyInfoFrame` @0x16A26C (552 Б) — обходить список HDMI
InfoFrame (`[r1+0x78]` = кількість, крок 0x20 = 32 Б на IF), копіює кожен і
розбирає заголовок за першими 3 байтами, порівнюючи з `0xD046` та `0xC03`.
Далі ставить:

```c
mdrv_HDMI_ParseDolbyInfoFrame.u8PreDoviFlag        // 1 Б, .bss 0xCAB44
mdrv_HDMI_ParseDolbyInfoFrame.u8PreLowLantencyFlag // 1 Б, .bss 0xCAB45
```

Рядки, які цей шлях друкує:

```
dv vsif HDMI_DOLBY_HDR_DETECT_STANDARD     <- звичайний DV
dv vsif HDMI_DOLBY_HDR_LOW_LATENCY        <- DV з низькою затримкою
Dolby Vision XC SHM Structure MCU mode
Dolby Vision XC SHM Structure Hardwire mode
[PQ] ... input source DV std., should not open game mode
```

Стрім, окремо:
`%s single dolby vision` / `%s dual dolby vision` (mvop),
`Dolby Vision Profile:0x%x level:%d SingleLayer=%d`.

### 9.2а Це ТІЛЬКИ ОДИН шлях — виправлення

Твердження «DV активується виключно з HDMI» було **надто категоричним**.
Користувач експериментально показав, що DV-тригер спрацьовує **у Kodi**,
тобто на внутрішньому джерелі.

Другий шлях — через DMS: **`_MDrv_DMS_Video_Flip_StDolbyHDRInfo`**
@0x2E937C (448 Б). На кожному відео-фліпі DMS дістає Dolby HDR-інформацію
зі структури статусу і кладе її в приватний ресурс скалера:

```asm
2e9408  bl   UtopiaResourceGetPrivate      ; g_pDMSRes[0x18]
2e940c  ldr  r0, [r5, #0x259]             ; поле Dolby HDR у статусі
2e9410  str  r0, [r4, #0x11E]             ; -> ресурс скалера
2e9414  tst  r0, #0x1C                    ; біти 2..4
2e9418  bne  0x2e942c
2e941c  ldrb r0, [res, #0xE3]
2e9424  cmp  r0, #1
2e9428  bne  0x2e951c                    ; інакше — не прокидати DV
```

Тобто **умова активації DV у DMS**: `status[0x259] & 0x1C != 0` **або**
`resource[0xE3] == 1`. Саме це і спрацьовує на внутрішньомуsource — Dolby
Vision з динамічних метаданих декодера (`hevc_dv_layer_update`,
`Dolby Vision without metadata in DV_MD_shm`, `Dolby Vision XC SHM Structure`).

**Два незалежні шляхи активації DV:**

| # | Шлях | Коли |
|---|---|---|
| 1 | `mdrv_HDMI_ParseDolbyInfoFrame` | HDMI-джерело з Dolby Vision VSIF |
| 2 | `_MDrv_DMS_Video_Flip_StDolbyHDRInfo` | внутрішній декодер, на кожному фліпі |

Обидва живі на цій платформі; HDMI-ліцензія (efuse/хеш) на шляху 2 **не** потрібна.

### 9.3 Проєктор має 4 HDMI-входи

`/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/board.ini`:

```ini
BOARD_INPUT_23_ENABLE_PORT = 1;   #HDMI 1
BOARD_INPUT_24_ENABLE_PORT = 1;   #HDMI 2
BOARD_INPUT_25_ENABLE_PORT = 1;   #HDMI 3
BOARD_INPUT_26_ENABLE_PORT = 1;   #HDMI 4
BOARD_EDID_INFO_HDMI_COUNT = 4;
```

(HDMI **TX/вивід** відсутній — `m_u32SupportHdmiTxCount = 0`, але це не заважає.)

### 9.4 Розв'язка з §8

Рівень TMO-тонмапінгу береться з `_stMultiMedia_Info[devId]`
(2=HDMI, 3=ADC, 4=MM). Отже внутрішнє джерело (MM) і HDMI йдуть
**різними гілками**. Раніше всі наші тести йшли на MM (лаунчер/Kodi),
де DV-шлях узагалі не активується.

**Вирішальний тест:** ввімкнути в HDMI 1 джерело з Dolby Vision-контентом і
зняти dmesg. Очікування за §8: на HDMI гілка TMO має дати валідний рівень,
бо група присутня (`GroupNum: 7, IPNum: 3`, `DumF series`) для рівнів 1 і 2.

---

## 10. ФІНАЛ: DV псує колір — доведено A/B-виміром (2026-10-01)

### 10.1 Метод

Один і той самий файл (`ONE_PIECE.2023E01E01.ROMANCE.DOWN.2160p.NF.WEB-DL.DV.HDR-DVT.mkv`),
один і той самий кадр (логотип «N», 00:00:02), один і той самий пристрій.
Знімання через `screencap` — тобто **до світлового тракту**, екран проектора на вимірювання не впливає.

Еталонний колір Netflix-логотипа відомий точно: `#E50914` = R229 G9 B20.

### 10.2 Результат

| Стан | Виміряно | R | G | B |
|---|---|---|---|---|
| Еталон | `#E50914` | 229 | 9 | 20 |
| **DV-обробка увімкнена** | `#BD6206` | **189** | **98** | 6 |
| **DV знято (SDR)** | `#E83200` | **232** | 50 | 0 |

Червоний канал у DV-режимі втрачає 18%, зелений завищений **у 11 разів**.
Простий підсилювач каналу такого не дає — це **перехрестне змішування**,
тобто матриця / часткове застосування метаданих DV.

### 10.3 Підтвердження з logcat

```
Kodi:  Testing codec: OMX.MS.DOLBY_VISION.DVHE.STN/ST/DVAV.SE/...   (усі DV-варіанти)
       Using codec: OMX.MS.HEVC.Decoder                             <- після зняття DV
MSTAR: AcquireVdecHw planar 0, dv 0, secure 0, over_fhd 1
       AcquireVdecHw DolbyVision 0, DRM 0, UHD 1                    <- dv 0 / DolbyVision 0
```

### 10.4 Замкнений ланцюг

```
DV Profile 8 single-layer (DVHE.ST), RPU присутній
  → PQ-рушій не виконує перетворення кольору
  → бо клієнтська IP-група відсутня (PQ_BIN_NOT_ENABLE,
      IsSupport_GroupIPBIN = Support [0])
  → метадані DV застосовуються частково / неправильно
  → зсув відтінку + знебарвлення + періодичні збої відео
```

**Висновок:** це не «поганий тонмапінг», а **зламана обробка DV**.
`Customer IP` — корінь; видимий дефект — руйнування кольору й збої відтворення.

### 10.5 Практичний обхід (перевірено)

Kodi → Налаштування → Програвач → Відео → «Дозволені формати динамічних
метаданих HDR» → **зняти Dolby Vision**. Потім зупинити і перезапустити
відтворення (налаштування діє на потік, який відкривається після нього).

Результат: звичайний `OMX.MS.HEVC.Decoder`, правильний червоний логотип,
картинка коректна з того самого 4K-файлу. Втрачається HDR-діапазон і
тонмапінг, але картинка правильна замість зруйнованої.

### 10.6 Урок по методу

Правильну картину дало не статичне розбирання, а **вимірювання одного
кадру в двох режимах**. Чотири попередні «доведені» версії були хибними
через те, що я виводив висновки з коду, не з картинки.
