# T2 breakpoint: чому DTS не виходить на SPDIF після успішного MI_AUDIO_Start(9)

Дата: 2026-08-30. Пристрій: у стані Solution A (utpa2k патчений + libmi3 v3). Змін під час
цього етапу НЕ вносилось. Метод: runtime dmesg/dumpsys/logcat (read-only) + статика mik.ko.

## 1. Верифікований runtime-ланцюжок (все до входу в DSP — працює)

| Крок | Доказ | Статус |
|---|---|---|
| Kodi AudioTrack (RAW), `STREAM_TYPE_DTS_512`, `AE_FMT_RAW` | logcat `CAESinkAUDIOTRACK::Initializing ... method: RAW (PT) stream-type: STREAM_TYPE_DTS_512` | OK |
| AudioFlinger offload активний | `dumpsys media.audio_flinger`: `HAL format: 0xb000000 (AUDIO_FORMAT_DTS)`, `Standby: no`, flags `DIRECT\|COMPRESS_OFFLOAD` | OK |
| MI_AUDIO_Open/Start | dmesg `[MI_AUDIO_Open]AdecId:0,*phAudio:0x19000000,eRet:0x0`, `[MI_AUDIO_Start]Codec:9,eRet:0x0` | OK — **license gate пройдено патчем** |
| HAL-парсер розпізнав DTS-кадри | dmesg `<MI3_INFO>reallocate memory: _u32DefaultWriteBufferSize:0 -> stWriteParams.u32BufSize:2012` (2012 B = DTS core 48 kHz 512-семпл кадр, 1509 kbps) | OK — DTS ES пишеться у DSP |
| Помилок старту/underrun | logcat/dmesg чисто | OK |

Висновок: license/capability/startup-фаза (все, що ми патчили) **працює**. Розрив — далі.

## 2. BREAKPOINT: `_MI_AOUT_SetHdmiAutoMode` (mik.ko, безіменна static @0x989cc)

MI-шар має власний монітор виходів: `_MI_AOUT_MonitorTask` → підзадача
`_MI_AOUT_HdmiInfoMonitor`:

```
_MI_AOUT_HdmiInfoMonitor (mik.ko):
    читає HDMI TX EDID: find_symbol("_MApi_HDMITx_GetDataBlockLengthFromEDID")
                        / "_MApi_HDMITx_GetRxAudioFormatFromEDID"
    → будує бітову маску форматів сінка (_u32CurEdidSupportList)
    mi_audio_GetCodecType(0, &codecType)      // тип ДЕКОДЕРА що грає
    if codecType != cached:
        _eCurHdmiAudioType = codecType
        _MI_AOUT_SetHdmiAutoMode(edidFmtBitmap /*r8*/, codecType /*r1*/)
```

`_MI_AOUT_SetHdmiAutoMode(r0=EDID-бітмапа, r1=codecType)` (tools/mik_SetHdmiAutoMode.asm):
jump-table за codecType (індекс r1-4, іди 4..0x17):

| codecType id | обробник | сенс |
|---|---|---|
| 0x09, 0x0a, 0x0b, 0x17 | 0x98a38 | **DTS-сім'я** (викликає `MApi_AUDIO_SetDTSCommonCtrl(0x10,x)`, `_eCurHdmiAudioType=9`) |
| 0x05 | 0x98b3c | AC3/EAC3/TrueHD (`MApi_AUDIO_SetAC3PInfo`) |
| 0x04, 0x07 | 0x98a88 | DD+ варіант |
| 0x06, 0x0c–0x16 | 0x98b8c | інше → PCM |

DTS-кейс (0x98a38) — рішення по **бітах EDID-бітмапи**:
```
tst r0, #0x800  → є DTS-HD (CEA-861 format code 11) → r6=3, r4=3 → TRANSCODE
tst r0, #0x80   → є DTS   (CEA-861 format code  7) → r6=1, r4=2 → BYPASS
ні біта                                          → r6=0, r4=0 → PCM (+SetDTSCommonCtrl(0x10,0))
```
Хвіст (спільний):
```
0x098dd4: bl  MApi_AUDIO_HDMI_TX_SetMode (r4)
0x098ddc: bl  MApi_AUDIO_SPDIF_SetMode   (r4)   ← режим SPDIF перезаписується тут
0x098de0: _stAoutSndParam[0x7c/0x80/0x6c/0x70] = r6/r5
```

Біти точно відповідають CEA-861 EDID Audio Data Block кодам: AC-3=2 (0x4),
E-AC-3=10 (0x400), MLP/TrueHD=12 (0x1000), **DTS=7 (0x80)**, DTS-HD=11 (0x800).
Це узгоджується з тим, що AC3-перемикання в цьому ж коді йде по біту 0x4.

**Висновок: SPDIF-режим на Android-шляху не константа [0xc], а динамічно
перераховується монітором за EDID HDMI TX.** У проектора EDID (внутрішній/virtual
HDMI TX) декларує AC-3, але не декларує DTS → для DTS-сесії монітор ставить
`SPDIF_SetMode(0 = PCM)` (і `HDMI_TX_SetMode(0)`), і тому bypass ніколи не
активується, попри те, що DTS decode program запущений і ES надходить.

Це також пояснює, чому license-патч дав очікуваний ефект (Codec:9 стартує —
раніше неможливо), але звуку DTS на AVR нема: не-PCM вимикається ПІСЛЯ старту
декодера на рівні AOUT-політики MI-шару.

## 3. Побічні спостереження

- Монітор реагує тільки на ЗМІНУ codecType (кеш `u32CurAudioCodecType`) — отже
  перемикання відбувається на момент початку/завершення DTS-треку.
- `MDrv_AUDIO_SPDIF_SetMode` (utpa2k 0x40587c) пише [0xc] і викликає
  HAL_AUDIO_SPDIF_SetMode; HTS-монітор utpa2k (`HAL_MAD_Monitor_DDPlus_SPDIF_Rate`)
  лише перепризначає той самий [0xc] — рішення прийнято вище, в MI-шарі.
- Решта 5 викликів MApi_AUDIO_SPDIF_SetMode в mik.ko: 4 у `MI_AOUT_Open`
  (boot-конфіг) і 1 у `MI_AOUT_SetAttr`-сусідній безіменній функції (шлях
  MI_AOUT_SetDigitalMode від HAL) — теж у результаті перекриваються монітором.
- T3-значення (0x440 bit0=0, 0x43d=1, 0x43e=1) детерміновано встановлюються
  нашим патчем на кожному SetDecodeSystem; їх апаратний зчит (devmem) відкладено
  на твій дозвіл — на ланцюжок breakpoint вони не впливають.

## 4. Варіанти фікса (НЕ застосовуються зараз)

1. **Мікропатч mik.ko `_MI_AOUT_SetHdmiAutoMode`, DTS-кейс** — примусово
   bypass для DTS незалежно від EDID: у блоці 0x98a54 (`mov r5,#1; mov r6,#0`
   при відсутності EDID-бітів) змінити `mov r6,#0` на `mov r6,#1` (2 байти:
   0x2600 → 0x2601) → r6==1 → r4=2 (BYPASS) → SPDIF_SetMode(2). Аналогічна
   точка є у гілці 0x800/0x80 — але достатньо однієї. Побічне: HDMI TX теж
   отримає bypass (r4 один на обидва виходи).
2. Додати DTS у внутрішній EDID HDMI TX — залежить від драйвера/конфіга,
   крутість вища, гнучкість нижча.
3. Вимкнути HDMI-info монітор (`_bHdmiInfoEnable`/`_bHdmiTxMonitorEnable`) —
   широковий Kong: вплине на всі виходи й усі кодеки.

## 5. Стан

- Solution A частково підтверджена runtime: license/startup пройдено.
- Повний шлях блокує EDID-driven auto-mode в MI AOUT (mik.ko), точка вказана.
- Пристрій у патченому стані; нічого нового не встановлено; всі зміни оборотні
  (оригінали: /data/local/tmp/backup_solutionA + локально runtime_backup/).
