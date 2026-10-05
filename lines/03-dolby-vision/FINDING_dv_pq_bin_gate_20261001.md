# FINDING — Dolby Vision PQ: калібрування заблоковано вмістом `dolby.bin`

**Дата:** 2026-10-01
**Платформа:** Thundeal TD98 Pro / C50A, MediaTek MT5889, Android 11
**Статус:** ланцюг розкрито до кінця; причина доведена статично + на екрані

---

## 1. Висновок одразу

Тонмапінг Dolby Vision на цьому проєкторі **не калібрується ні через меню, ні через
конфіги, ні через код**. Причина — зміст PQ-бінарника: завантажений `dolby.bin`
не містить клієнтського IP-режиму (customer IP mode), який PQ-рушій шукає.

**Найпряміший доказ — сам драйвер відповідає на запит:**

```
# echo IsSupport_GroupIPBIN > /proc/utopia_mdb/pq
[  117.447980] Support [0]
```

`Support [0]` = GroupIP-бінарник не підтримується. Це не припущення, а відповідь
штатного діагностичного інтерфейсу самого MediaTek PQ.

Справжнє «полагодити DV» вимагає **клієнтського PQ-бінарника** (`PQ_BIN_CUSTOMER_MAIN`),
який готує Dolby/ChipOne для конкретної панелі. На пристрої його, за непрямими
ознаками, немає; `/vendor` читається лише.

> **ВИПРАВЛЕННЯ (2026-10-01, пізніше).** Версія, наведена нижче в §5.2 («розбіжність
> версій 2019 проти 2023»), — **хибна**. `PQ Standard Version` — це версія
> завантаженого STD-пакета, а не вимога рушія. Див.
> `FINDING_video_g_namespace_and_3d_20261001.md` §5.

### 1.1 Другий, незалежний дефект — вендорний конфіг не має сховища

Паралельно знайдено, що шар налаштувань MediaTek (`g_*` ключі) на цій збірці
**відсутній повністю**:

| Де шукали | Результат |
|---|---|
| `settings list global / system / secure` | ключів `g_*` — **0** |
| `/data` (grep `g_video__`, `vid_3d_item`) | не знайдено |
| `/vendor/tvconfig`, `/vendor/cusdata` | `vid_3d` не знайдено |
| `/data/data/com.mediatek.tv.service/shared_prefs/mediatek_pref.xml` | 2 записи, не конфіг |
| `/vendor/persist` (куди мав іти `LINUX_PERSIST_PATH`) | **каталогу не існує** |
| `/dev/block/by-name/*persist*` | розділу persist **немає** |

Ключі `g_video__vid_3d_item`, `g_video__vid_3d_mode`, `g_fusion_picture__pq_wb_cor_*`
живуть лише у коді `TvServices.apk` (рядки 14316–14327 і 12932–12936) — фізичного
сховища для них на пристрої немає.

**Наслідок:** `isSupport3D` читає `g_video__vid_3d_item` → не знаходить → 3D сірий.
`EndUserCalibrationFragment` через `MtkTvConfig.setFDolbyUserData*` пише туди ж →
запис зникає. `tv_picture_video_color_space` після перезавантаження повертається
у `Auto`, бо на старті його переписує шар, який читає з порожнього сховища.

Це **єдиний спільний дефект** за калібруванням PQ, кольоровим простором і сірим 3D.

---

## 2. Виправлення попереднього твердження

Раніше я писав, що `dolby.bin` — це «заглушка з байтів 0x01». **Це було
неправильно.** Виміряно:

```
/vendor/tvconfig/config/HDR_BIN/dolby.bin   14235 байт
різних значень байтів: 256
ентропія по блоках: 1.92 … 6.64 біта
```

Файл справжній і структурований: 565 байт поспіль 0x01 (заголовок), далі 0x02,
потім regions з ентропією до 6.64. Це скомпільований Dolby PQ-рушій, просто
**не той** варіант, який має клієнтський IP.

`dolby_factory.cfg` (2767 байт) при цьому **повний і правильний** — містить
повні VSVDB v2 дані панелі:

```
Tmax = 281          Tmin = 0.23      TEOTF = POWER    Tgamma = 2.2
TPrimaries = 0.6291 0.3425 0.3100 0.6106 0.1508 0.0496 0.3124 0.3319
[PictureMode 0] Dark  (Tmax 281, GlobalDimming 1)
[PictureMode 1] Bright(Tmax 140, GlobalDimming 0)
[PictureMode 2] Vivid (Tmax 210, GlobalDimming 0)
```

Тобто панель описана правильно — код просто не читає це як customer IP.

---

## 3. Екран калібрування: чому «Зберегти» не працює

### 3.1 Механіка вводу (через що я спочатку помилявся)

`EndUserCalibrationFragment` показує 11 полів. Натискання ENTER на рядку
відкриває **модальне вікно `EndUserCalibrationEditDialog` з екранною цифровою
клавіатурою**. Значення потрапляє в список лише після натискання кнопки ✓
(координати ≈ 1573,1024) у цьому вікні. ENTER усередині вікна не підтверджує,
а **дописує цифру** — відтворено: `281 → 2811`, `140 → 1408`.

### 3.2 «Зберегти» — порожня дія

Після коректного заповнення всіх 11 полів і натискання «Зберегти» у logcat
з'являється рівно один рядок:

```
EndUserCalibrationFragment: prefKey = tv_picture_video_end_user_calibration_save
```

Жодного запису. Ключ `tv_picture_video_end_user_calibration_tmax` (формат
ключів видно з цього ж логу) не з'являється ані в `global`, ані в `system`,
ані в `secure`.

### 3.3 Чому не допомогло записати ключі напряму

Я записав усі 13 ключів (`..._tmax`, `..._tmin`, `..._gamma`, `..._rx/ry/gx/gy/bx/by/wx/wy`,
`..._save`, `..._view_mode`) через `settings put global` і відкрив екран знову —
усі поля знову показали 0.0.

**Висновок:** фрагмент читає значення **не** з Android settings, а з вендорного
шару через JNI `MtkTvConfig.getFDolbyUserDataMax/Min/Gamma/Primaries`, і ці
геттери повертають 0. «Зберегти» мав викликати `setFDolbyUserData*` — і не
викликає. Обидві половини цього шляху — no-op на цій збірці.

Тимчасові ключі, які я створив для перевірки, видалені (`settings delete`).

### 3.4 Де живе реалізація JNI

```
/vendor/lib/libcom_mediatek_twoworlds_tv_jni.so   (681260 байт, ARM)
  -> TVNativeCustom / MtkTvConfig
  -> містить символи setFDolbyUserDataMax/Min/Gamma/Primaries
  -> має точку входу setIsPersistent
```

Файл не має жодного шляху-конфігу для цих даних (усі 147 path-рядків у
`.rodata` перебрано — лише `/mnt/vendor/tmp/*`, `/data/vendor/3rd_rw/localtime`,
`/proc/*`). Тобто запис іде через сервіс MTK, а не у файл.

---

## 4. Справжній гейт — у коді PQ (PROVEN BY BINARY)

Ланцюг із живої розвідки:

```
_MI_DISP_SetHdrPanelNits[10055]        (mik.ko, 0xe43c8)
  -> MDrv_PQ_Set_CustomerIp_Parameter   (utpa2k.ko)
     -> MDrv_PQ_Set_CustomerIp_Parameter_U2  (utpa2k.ko 0xcb5cc, 500 B)
        [PQ] PQ_DBG_ERROR [...,4962]: Customer IP is not set as customer mode
  <MI3_ERR>_MI_DISP_SetHdrPanelNits[10055]: Set E_PQ_XC_IP_TMO Customer Data fail.
```

### 4.1 `MDrv_PQ_Set_CustomerIp_Parameter_U2` @ utpa2k 0xcb5cc

```asm
0xcb67c  movw r0, #0            ; u32IPModeAddress
0xcb680  mov  r1, #0x5c         ; 92 = розмір запису
0xcb68c  mla  r0, r7, r1, r0    ; r0 = &u32IPModeAddress[modeIdx]
0xcb690  ldr  r0, [r0, sb, lsl #2]
0xcb694  bl   MDrv_PQBin_VR_Read2Byte      ; <-- читає з VRAM PQ-бінарника
0xcb698  movw r1, #0            ; u8IPModeAddressMask
0xcb69c  mov  r2, #0x17
0xcb6a8  mla  r1, r7, r2, r1    ; r1 = &u8IPModeAddressMask[modeIdx]
0xcb6b0  ldrb r2, [r1, sb]      ; r2 = mask[modeIdx][tableIdx]
0xcb6b8  tst  r2, r0
0xcb6bc  beq  0xcb72c           ; <== ГЕЙТ: mask & ipMode == 0 -> "not customer mode"
```

`u32IPModeAddress` і `u8IPModeAddressMask` — таблиці, **які приходять з PQ-бінарника**
через `MDrv_PQBin_VR_Read2Byte`. Тобто «customer mode» не зберігається ніде у
конфігурації: воно визначається наявністю запису у `dolby.bin`.

### 4.2 `MDrv_PQBin_GetGRule_GroupIPNum` @ utpa2k 0x8dd54

```asm
0x8dd70  cmp  r3, #0xa          ; індекс групи < 10
0x8dd74  bhs  error
0x8dd7c  movw r1, #0x255        ; дозволені індекси: 0,2,4,6,8
0x8dd84  tst  r2, r1, lsr r0
0x8dd8c  movw r0, #0            ; таблиця покажчиків (локальна, у .text)
0x8dd94  ldr  r0, [r0, r3, lsl #2]
0x8ddb0  ldrb r0, [r0]          ; ознака активності групи
0x8ddb4  cmp  r0, #1
0x8ddb8  bne  0x8de38           ; -> printk "=PQ_BIN_NOT_ENABLE!!!"
```

Кожна група PQ-IP мусить мати байт активності == 1. У нашому бінарнику він 0.

---

## 5. Чому це не виправити на пристрої

| Шлях | Статус |
|---|---|
| Меню «Калібрування Dolby Vision PQ» | no-op, доведено логом (§3.2) |
| `settings put global` напряму | фрагмент їх не читає (§3.3) |
| Правка `dolby_factory.cfg` | дані вже правильні; проблема не в них |
| Правка `hdr.ini` / `PQConfig.ini` | customer-mode не береться з конфігів (§4.1) |
| Заміна `dolby.bin` | потрібен клієнтський PQ-бінарник від Dolby/ChipOne; на пристрої немає |
| Запис у `/vendor` | `/vendor` = `/dev/block/dm-1`, `ro`; `touch` → Permission denied |
| Патч коду, щоб обійти гейт | можливо, але PQ-рушій усе одно не має IP-даних — результат не має сенсу |

---

## 6. Про малий патч — чому «просто зняти гейт» небезпечно

Ідея «пропатчити умову й змусити функцію пройти» перевірена статично і
**відхилена**: гейт у `MDrv_PQ_Set_CustomerIp_Parameter_U2` захищає не одну умову.

```asm
0xcb67c  r0 = &u32IPModeAddress[modeIdx]     ; теж з PQ-бінарника
0xcb694  bl  MDrv_PQBin_VR_Read2Byte
0xcb6b8  tst r2, r0
0xcb6bc  beq <помилка>
0xcb6cc  r0 = &u32IPParameterAddress[modeIdx] ; І ЦЕ теж з PQ-бінарника
0xcb6dc  ldr r0, [r0, sb, lsl #2]
```

Якщо зняти умову, наступна інструкція бере **адресу параметра з того ж бінарника**.
Якщо групи немає — там нуль, і запис піде в scaler-регістр за адресою 0.
Це гірше за поточну graceful-помилку: можливе зависання панелі або пошкодження
режимів PQ.

Коректний патч мав би **підкласти дані**, а не зняти перевірку, тобто фактично
замінити вміст `dolby.bin` — тобто це вже не «малий патч», а deliverable від Dolby.

## 6.1 Небезпечна команда — `LoadBinTest` крашить ядро

```
# echo LoadBinTest MAIN On > /proc/utopia_mdb/pq
# echo LoadBinTest MAIN_TMO On > /proc/utopia_mdb/pq
→ adb: device offline, пристрій перезавантажився
```

Примусове завантаження PQ-бінарника в обхід шляху ініціалізації **обриває ядро**.
Не повторювати. (Перевірено один раз — пристрій піднявся сам, дані цілі.)

## 6.2 Що реальто застосовано

```
settings put global tv_picture_video_color_space 4     # BT.2020
```

**Не переживає перезавантаження** (перевірено: set → reboot → читається 0).
Для контрасту `tv_picture_video_3d_to_2d=1`, записаний напряму, **пережив** ребут —
тобто `Settings.Global` як сховище працює, але `color_space` переписується на
старті шаром, що читає з порожнього вендорного конфігу (§1.1).

Можлива гіпотеза (не перевірена): під час відтворення DV-контенту `color_space`
може не скидатися, бо тоді вендорний шар бачить реальний сигнал.

## 7. Інструмент, який зроблено

`C:\firmware_temp\patch_baseline\kan.py` — анотований дизасемблер ARM/Thumb для
`utpa2k.ko` / `mik.ko` (обидва не strip-нуті). Показує ім'я символу поруч із
`movw/movt`-парами, PC-відносними завантаженнями та цілями `bl`/`b`, розбираючи
`R_ARM_ABS32` / `R_ARM_CALL` / `R_ARM_JUMP24` без застосування релокацій.

```bash
python kan.py <addr_hex> <size> <label>        # utpa2k.ko
KO=mik_stock.ko python kan.py <addr> <size> <label>
```

Приклад, який дав відповідь:

```
$ KO=mik_stock.ko python kan.py e43c8 448 _MI_DISP_SetHdrPanelNits
  0xe442c: bl  #0xe442c   ; ->MDrv_PQ_Set_CustomerIp_Parameter
  0xe4474: bl  #0xe4474   ; ->MApi_XC_HDR_Control
```

---

## 8. Що це означає практично

Dolby Vision **декодується і показується** (банер DV з'являється, кодеки
`OMX.MS.DOLBY_VISION.DVHE.*` присутні, `picture_mode_dolby` працює). Але PQ-рушій
не отримує клієнтських даних панелі, тому **`_MI_DISP_SetHdrPanelNits` не
застосовує пані-нітси** і тонмапінг DV працює за запасним шляхом. Практичний
наслідок — DV-картинка буде «неправильною» за яскравістю/контрастом, і це
**неможливо виправити налаштуваннями на цьому пристрої**.

Якщо потрібна правильна DV-картинка, єдиний реальний шлях — отримання
клієнтського `dolby.bin` (Dolby Vision certification від виробника) або
пристрою з тією самою прошивкою, де він є.

---

## 9. Пов'язані документи

- `FINDING_dv_no_pq_tuning_20261001.md` — попередній (містить помилкове твердження
  про заглушку `dolby.bin`, тут виправлено)
- `FINDING_dolby_vision_hw_20261001.md` — апаратна присутність DV
- `spdif_audio_investigation/base_dmesg.txt` — рядки 309–311, жива розвідка
