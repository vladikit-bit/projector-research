# DTS → SPDIF TX: фінальний evidence chain (статичний аналіз, без змін на пристрої)

Дата: 2026-08-30
Метод: тільки статичний аналіз (`libs/libutopia.so`, `kmods/utpa2k.ko`, `kmods/mik.ko`,
`kmods/dtv_driver.ko`, `libs/libmi3.so`, `libs/audio.primary.mt5889.so`) + вже наявні
Ghidra-декомпіляції (`k_mi_decoder_open.c`, `k_mik_decomp.c`, `k_mapfun_decomp.c`,
`k_ut_setdecsys.c`, `k_hashkey.c`). Пристрій не чіпався.

---

## Вердикт (коротко)

**Реального software path для готового (готово-стиснутого) DTS bitstream → SPDIF TX
без запуску DTS ADEC (decode program) і без проходження license gate у цьому vendor
stack НЕ ІСНУЄ.** Це не «ми не знайшли» — це структурна властивість стеку, доведена
нижче двома незалежними статичними фактами:

1. **Єдиний** код у всьому стеку, який взагалі формує DTS-over-SPDIF вихід
   (IEC61937/DTS burst), — це DTS SDO SPDIF packer, і він живе **всередині R2 DSP
   firmware-образу** (`mst_snd_r2` / `mst_snd_r2_MS12V22`) разом із DTS декодером та
   Xcoder-ом. Його вхід (`pFrameBuf`) — це кадри, згенеровані **тим самим DTS
   програмним блоком** (вихід декодера або вихід PCM→DTS енкодера). Входу
   «зовнішній готовий DTS бітстрім» у нього не існує.
2. Хост (CPU/Android) не має жодного API інжекції даних у SPDIF TX. Всі
   `HAL_AUDIO_SPDIF_*` / `MI_AOUT_*` — тільки mode/config. Єдина дана-доріжка з
   хоста в DSP — `MI_AUDIO_Write` → ES-буфер **декодерної сесії**. Декодерна сесія
   DTS стартує лише через `MI_AUDIO_Start(CodecType=9)` → `MApi_AUDIO_SetDecodeSystem`
   → `MDrv_AUDIO_CheckHashkey` → `MDrv_AUTH_IPCheck` (DTS = fail на цьому пристрої)
   → license-маски в R2 → DTS decode program не стартує.

---

## Ланцюжок A: єдина доріжка, якою стиснутий бітстрім взагалі потрапляє в DSP (Android HAL)

```
[INPUT: готовий DTS bitstream, Kodi/HAL]
  ↓ (після gate1/gate2/gate3 — вже пропатчено/досліджено раніше)
audio.primary.mt5889.so  mi_decoder_open @0x3f408
    format 0x0B000000 (DTS) / 0x0C000000 (DTS-HD)  →  MStar CodecType = 9
    (k_mi_decoder_open.c:85-95: `uVar3 = 9` для 0xb000000 та 0xc000000)
  ↓
MI_AUDIO_Start(handle, &codecType=9)                     (k_mi_decoder_open.c:132)
  ↓ ioctl 0xC0081005 (libmi3.so)
mik.ko  MI_AUDIO_Start @0xb6e40 → _MI_AUDIO_Internal_Start
    _MI_AUDIO_CodecTypeMapDecoderType(9) = 0xb (DTS)     (k_mapfun_decomp.c case 9 → 0xb)
    MApi_AUDIO_SetDecodeSystem / MApi_AUDIO_SetDecodeCmd
    (mik.ko R_ARM_CALL relocs: 0x66700, 0x66790 → MApi_AUDIO_SetDecodeSystem;
     22× MApi_AUDIO_SetDecodeCmd)                        (k_mik_decomp.c:110-129,154)
  ↓
utpa2k.ko  _MApi_AUDIO_OpenDecodeSystem @0x402d04
  → MDrv_AUDIO_SetDecodeSystem @0x44249c
      1) MDrv_AUDIO_CheckHashkey()  @0x440ab8           (k_ut_setdecsys.c:168)
           20× MDrv_AUTH_IPCheck(id): id=0xb (DTS), 0xc (DTS-HD) …
           → capability-маски g_AudioVars2[0x440]/[0x444],
             DTS-флаг g_AudioVars2[0x43d], MS12-рівень [0x4d0]
           (k_hashkey.c:33-63; підсумкові записи :759-763)
      2) маски передаються в R2 DSP: FUN_004421fc(0, mask[0x440]) і (1, mask[0x444])
         — це guarded register-write: всередині функції
         audio_regCheck_startAddress/endAddress/bitMask/backTrace,
         тобто запис capability-масок у регістри/SHM **обох** R2 (DEC і SND)
         (розбирання 0x4421fc, ARM; символи audio_regCheck_*)
      3) HAL_AUDIO_SPDIF_SetMode(g_AudioVars2[0xc], g_AudioVars2[0x1cc])
                                                        (k_ut_setdecsys.c:179-180)
  ↓ (якщо license OK — DTS decode program стартує; на C50A — fail)
[SPDIF non-PCM сесія відкривається ЛИШЕ всередині decode program]
  ↓
SPDIF TX  ←  DSP output matrix (R2), НЕ з хоста
```

**Точка обриву ланцюжка A:** `MDrv_AUDIO_SetDecodeSystem` → `MDrv_AUDIO_CheckHashkey`
→ `MDrv_AUTH_IPCheck(0xb/0xc)` = fail → DTS-флаг/маски не встановлені → R2 DTS decode
program не запускається → non-PCM SPDIF не активується. Підтвердження з іншого боку:
`HAL_MAD_GetAudioInfo2` (utpa2k, k_utpa2k_decomp.c:761) для decode system 4/0xb
(DTS) явно гілкується за `g_AudioVars2[0x43d] == 1`.

## Ланцюжок B: «інший шлях» — DTS SDO SPDIF packer (перевірка гіпотези bypass-транспорту)

DTS SDK знайдено в `.data` обох бінарників — це **DSP firmware-образи**, не хост-код:

| Образ | utpa2k.ko (символ) | Розмір | Вміст |
|---|---|---|---|
| `mst_codec_r2` | .data @0x16be18 | 0x29c0f4 | DEC R2 (кодек-програма) |
| `mst_codec_r2_MS12V22` | .data @0x407f0c | 0x1e401c | DEC R2, варіант MS12 |
| `mst_snd_r2` | .data @0x5fb81c | 0x17e948 | **SND R2 — повний DTS:X SDK** |
| `mst_snd_r2_MS12V22` | .data @0x77a164 | 0x1c1330 | SND R2, варіант MS12 |

Той самий DTS-кластер є і в `libs/libutopia.so` (.data ~0xee1000/0x108e800) —
копії тих самих образів у userspace-бібліотеці. В `armfw.bin`, `mik.ko`,
`dtv_driver.ko`, `libmi3.so`, HAL — **немає**.

Доказ, що це firmware, а не хост-код:
- усі рядки DTS SDK (`DTSDecSDOPacker_API_Process`, `dtsx-sdo/src/sdo_spdif_packer.c`,
  `DTSTransEnc1m5`, `D2A_DTSXENC`, `Mstar InputProcess`) лежать **тільки всередині**
  діапазонів `mst_snd_r2*` (перевірка tools/ скриптом: 0 збігів у `.text`/`.rodata`);
- єдині посилання хоста на образи — 6 релокацій, і всі в
  `HAL_AUDSP_DspLoadCode` @0x46f5a8: вибір варіанту за `g_AudioVars2[0x4d0]==4`
  (MS12-ліцензія → MS12V22-образи, інакше базові) + завантаження сегментів
  (`HAL_AUDSP_DspLoadCodeSegment`/`DspVerifySegmentCode`, init SHM, EnableR2).

Дані-флоу всередині DTS SDK (відновлено з DTS_VERIFY assert-рядків — це оригінальні
виклики з сорців DTS, збережені в образі):

```
dtsx-main/app/file-player/src/dtsx-file-player-api.c:
    DTSEnc_Wave_Read(...)  /  dtsSetInputBuffPtrs(... pcmBuf ...)   ← вхід = PCM
    DTSX_CORE2_API_Xcoder(pCore2Instance, pcmBuf, &pFrameBuf, ...)  ← PCM → DTS кадри (ЕНКОДЕР)
        (dtsx-xcoder: DTSTransEnc1m5_API_Encode_Frame / DTSLosslessTransEnc_API_Encode_Frame;
         pInit->nBitsPerSample 16/20/24, nFS 44.1/48/88.2/96k — це енкодер, не парсер)
    DTSX_CORE2_API_SDO_Packer(pCore2Instance, pFrameBuf, ...)       ← пакування кадрів
    DTSDecSDOPacker_API_Process(pSDOPacker, output_buf, nuFrameSize,
                                p_file_player->spdifFrmBuf, …, DTS_SDO_SPDIF_OUT)

dtsx-main/src/dtsx-common.c (декодерний варіант):
    DTSDecSDOPacker_API_Process(pstCommon->SDOPacker,
                                pFrameBuf + k*fsize[0], fsize[k], …, DTS_SDO_SPDIF_OUT)

dtsx-sdo/src/sdo_spdif_packer_api.c (сам пакер):
    dtsParseFrame(pBitStream, &StreamInfo, …)     ← вхід пакера = ЦІЛИЙ DTS-кадр (парсинг, не декод)
    DTSSPDIFPackFrame(p_spdif_packer, nBytesToPack, nPCMformat, nStride)
    DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32  ← опційний IEC61937-заголовок
dtsx-sdo/src/sdo_spdif_packer.c:
    PackSPDIFStream(…)  →  spdifFrmBuf
```

**Класифікація (за классифікацією A/B/C з постановки задачі):**
- PackSPDIFStream/DTSSPDIFPackFrame — це **клас B-сумісний форматер** (стиснений DTS
  кадр → IEC/SPDIF без декодування), НО:
- його вхід `pFrameBuf` у 100% викликів генерується **усередині того самого DTS
  програмного блоку**: або вихід декодера (`DTSX_CORE2_API_Process`), або вихід
  PCM→DTS **енкодера** (`DTSX_CORE2_API_Xcoder` → `DTSTransEnc1m5`). Тобто це
  фактично **варіант A** (PCM→IEC після decode/re-encode), а не варіант B
  (зовнішній готовий бітстрім → IEC).
- Зовнішнього входу «готовий DTS бітстрім → пакер» не існує: єдина дана-доріжка в R2
  програму — ES-буфер декодерної сесії (ланцюжок A).
- Пакер виконується **на R2** всередині DTS-програми, яка стартує лише через ланцюжок A
  з ліцензійними масками, записаними в обидва R2 при кожному `SetDecodeSystem`.

## Хост-API перевірені на «інжекцію» — всі config-only

| API | Адреса (utpa2k.ko) | Результат аналізу |
|---|---|---|
| `HAL_AUDIO_SPDIF_TranscodeMode` | 0x445504 (0x8b8 байт) | Логіка вибору типу виходу: `Get_CodeTypeByDecodeID`, `HAL_MAD_GetDTSInfo`, `HAL_DEC_R2_Get_SHM_INFO`, таймери; пише g_AudioVars2[0x52c]/[0x530]. Даних не приймає. |
| `HAL_AUDIO_ConfigureAlwaysTranscoder` | 0x44465c (0x3c байта) | `g_AudioVars2[0x52c]=a; [0x530]=b` — два конфіг-слова. Даних немає. |
| `HAL_AUDIO_DTSELoadCode` | 0x444010 | Заглушка: тільки `printk`, завантаження немає. |
| `HAL_AUDIO_SPDIF_BypassMode/AutoMode/PcmMode/SetMode` | 0x444f40/0x444914/0x445dbc/0x448a10 | mode/config (фази 3–5). |
| `MI_AOUT_SetDigitalMode` (libmi3/mik) | — | selector режиму, не інжекція (фаза 5). |

`MDrv_AUDIO_Get_DTS_License` @0x422aa0 (для повноти): повертає 0 (успіх) тільки якщо
`MDrv_AUTH_IPCheck(0xf|0x3a|0x12|7)` всі OK **І** біт 7 регістра 0x112cf0
(`HAL_AUDIO_AbsReadReg`) == 1. Викликається з `MDrv_AUDIO_Get_Decoder_Support`
(капабиліті-звіт) і `MDrv_AUDIO_Get_License` — тобто ніхто не будує з нього
data path.

## Досяжність з Android HAL

- `audio.primary.mt5889.so` DT_NEEDED: libmi3, libaudioparser, libtinyalsa, … —
  **libutopia відсутня**. Імпорти HAL: тільки `MI_AUDIO_*` / `MI_AOUT_*`.
- `libmi3.so` імпорти: `MApi_AUDIO_SYSTEM_Control`, `MApi_AUTH_Process`, CMA, DMX —
  без жодного DTS/MApi-AUDIO-декодного API.
- `mik.ko` → `MApi_AUDIO_OpenDecodeSystem/SetDecodeSystem/SetDecodeCmd` (R_ARM_CALL
  relocs) → `utpa2k.ko` (MApi реалізація + R2 образи).
- Жоден хост-компонент не лінкується і не викликає DTS SDK — образ R2 взагалі не
  має хост-видимих точок входу, крім завантаження через `HAL_AUDSP_DspLoadCode`.

## Відповідь на питання постановки

> Чи існує в цьому vendor stack реальний software path для готового DTS bitstream →
> SPDIF TX без запуску DTS ADEC/license gate?

**Ні.** Статичний доказ місця обриву:
1. Форматер DTS→SPDIF (SDO packer) існує тільки всередині R2-образу DTS-програми
   (`mst_snd_r2*`) і приймає кадри лише від декодера/Xcoder того ж блоку —
   зовнішнього входу для готового бітстріму немає (assert-рядки + 0 xref-ів з хоста).
2. Єдиний шлях доставки даних у DSP — ES-буфер декодерної сесії; DTS-сесія стартує
   тільки через `MDrv_AUDIO_SetDecodeSystem` → `MDrv_AUDIO_CheckHashkey`
   (utpa2k.ko 0x44249c → 0x440ab8) з `MDrv_AUTH_IPCheck(0xb/0xc)`, що на C50A
   провалюється; результат (маски) записується в обидва R2 при кожному
   SetDecodeSystem.
3. Хост не має API запису даних у SPDIF TX (таблиця вище), а HAL не лінкує
   libutopia взагалі.

Отже, на цьому пристрої DTS-over-SPDIF можливий **тільки** як bypass-вихід
запущеного (ліцензованого) DTS decode program. Kodi-IEC61937-пакер не може
замінити це навіть при пропатчених HAL-gates, бо його burst-и потрапляють у
ES-вхід декодерної сесії, а не в SPDIF TX (узгоджується з експериментами фаз 3–4).

## Що залишається відкритим (не впливає на вердикт)

- Точний механізм, чому IEC61937-обгорнутий AC3 не синхронізується в AC3-декодері
  (payload містить валідний 0x0B77; перевірка стану декодера на R2 потребує
  зворотного інжинірингу R2-ISA-коду — поза межами цього етапу).
- Внутрішня логіка R2-програми (AEON ISA) по-справжньому не дизасемблювалась;
  все, що стосується R2, доведено через структуру образів, рядки, релокації та
  хост-обгортки.

## Артефакти перевірки

- `tools/sdo_xref.py` — розкладка рядків/кластерів DTS SDK у libutopia.so, xref-скан.
- Інлайн-скрипти цього сеансу: пошук DTS-рядків по всіх бінарниках; парсинг
  `dsp_info`/`mst_*_r2` символів utpa2k.ko; релокаційний скан посилань на образи;
  розбирання `HAL_AUDSP_DspLoadCode` (`tools/ut_DspLoadCode.asm`),
  `MDrv_AUDIO_Get_DTS_License` (`tools/ut_dtslicense.asm`),
  `MDrv_AUDIO_Get_Decoder_Support` (`tools/ut_decsup.asm`),
  `HAL_AUDIO_SPDIF_TranscodeMode` (`tools/ut_spdif_transcode.asm`),
  `HAL_AUDIO_ConfigureAlwaysTranscoder` (`tools/ut_alwaystrancode.asm`),
  `HAL_AUDIO_DTSELoadCode` (`tools/ut_dtseload.asm`), функції 0x4421fc.
