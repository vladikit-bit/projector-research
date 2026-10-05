# Фаза 2: BYPASS-експеримент + kernel-рівень (DTS → NONE)

Дата: 2026-08-29 · Без патчів. Змін конфігурації немає (режим «Цифровой выход» був і залишився «Пропуск»).

---

## 1. Вердикт BYPASS-експерименту (CONFIRMED)

Екранні докази: `screen2.png` → `screen4.png` (Настройки → Настройки устройства → **Цифровой выход**):

| Опція | Стан |
|---|---|
| Авто | — |
| **Пропуск (= BYPASS)** | **ВИБРАНО (радіо-кнопка активна)** |
| PCM | — |
| Dolby Digital Plus | — |
| Dolby Digital | — |

Режим BYPASS вже був активний **до** експерименту, при цьому:
- AC3 → ресівер показує Dolby Digital (підтверджено користувачем раніше);
- DTS → на ресівері нічого.

**Висновок: перемикач SPDIF AUTO/BYPASS НЕ є блокером.** Проблема не в AUTO-детекції формату: у BYPASS DTS так само мовчить, бо **DTS-потік взагалі не доїжджає до SPDIF TX — DSP-декодер для DTS не стартує**.

Додатково встановлено механіку Android-сторони: `AudioSystem.setParameters("spdif_mode=bypass"/"spdif_type=dts")` оновлює тільки дзеркало HAL (`ui(bypass)/ui(dts)` в `dumpsys`), а реальний apply живе в TV-Settings → vendor TV-service → kernel; єдина функція apply у HAL (`utils_ApplyDigitalOutputSetting`) — dead code (0 xref-ів у всій файловій системі).

## 2. Kernel-рівень: що відбувається після MI_AUDIO_Start (CONFIRMED з логів)

Свіжий dmesg під час probe-тестів (AC3/DD і DTS підряд):

```
MI_AUDIO_Open:  Name:AudioHAL, StreamType:3, RenderMode:0 → AdecId:0, handle 0x19000000
MI_AUDIO_Start: Codec:5  hAudio:0x19000000, eRet:0x0      ← DD
MI_AUDIO_Start: Codec:9  hAudio:0x19000000, eRet:0x0      ← DTS
```
Вікна Start(5) і Start(9) **структурно ідентичні** (однакові MApi-виклики: SetDecodeCmd×8, SetCommAudioInfo×5, SetAudioParam2×3; жодних помилок для 9). Розходження — у результатах DSP-запитів під час відтворення:

```
<MI3_ERR>_MI_AUDIO_GetCodecType[2185]: Fail to get DTS codec type!
   MApi_AUDIO_GetAudioInfo2(eAdecId:0, Audio_infoType_Decoder_Type) failed!
<MI3_ERR>_MI_AUDIO_GetCodecType[2172]: Get codec type None!
```

Ланцюг підтверджено статично:
- `mik.ko`: друк рядка належить інлайну `_MI_AUDIO_GetCodecType` (всередині `_MI_AUDIO_EsBufMonTask` / GetAttr-обробника); запит іде `MApi_AUDIO_GetAudioInfo2(handle, infoType)` → `UtopiaIoctl(cmd 0xCF)` → `HAL_MAD_GetAudioInfo2` (utpa2k) → DSP;
- для AC3 той самий запит повертає тип (dumpsys: `AudioHAL audio codec type: AC3`), для DTS — fail → NONE;
- отже **DSP ADEC0 не має активного DTS-декодера** (немає чого читати), і non-PCM-прапорець для SPDIF ніхто не ставить.

`audio_parser` у userspace при цьому виправно_postачає цілі DTS-фрейми (2012 B) в `MI_AUDIO_Write` — дані входять, але ніхто їх не декодує і не віддає на SPDIF TX.

## 3. Витягнуто для наступної фази (готово до Ghidra)

| Файл | Розмір | Runtime base | Ключові символи |
|---|---|---|---|
| `kmods/mik.ko` | 8.0 MB | 0xE23CF1F0 | MI_AUDIO_Open/Start/Close, `_MI_AUDIO_GetCodecType` (інлайн), `mi_audio_SetCodecType` (пише `_aeCurAudioDecoderType[]`) |
| `kmods/utpa2k.ko` | 25.4 MB | 0xE383FB00 | `MApi_AUDIO_GetAudioInfo2` (0x3fd0b0), `_MApi…` (0x3ed960), **`HAL_MAD_GetAudioInfo2` (0x45e34c, 5904 B)** |
| `kmods/dtv_driver.ko` | 3.4 MB | 0xE262F580 | `MTGADEC_MTAUD_SetDecType` (0x8c980), `MTGADEC_MTAUD_GetDecType` (0x8c8ec), `_GlueDecTypeTransfer` |

kallsyms відкрито (`kptr_restrict=0`, символи читаються). `/proc/kcore` відсутній — не потрібен, бо аудіо-стек повністю в модулях.

## 4. Наступний крок (пропозиція, без виконання)

1. Ghidra (новий проєкт) → `utpa2k.ko`: гілка `Audio_infoType_Decoder_Type` в `HAL_MAD_GetAudioInfo2` → що саме читається з DSP і за якої умови повертається fail для DTS.
2. `mik.ko MI_AUDIO_Start`: 8× `MApi_AUDIO_SetDecodeCmd` — знайти, який із них несе DecType, і як MI codecType (5/9) конвертується в MStar `AU_DEC_TYPE` (таблиця/перемикач; шукати функцию з рядком `.L__FUNCTION__._MI_AUDIO_CodecTypeMapDecoderType`, що живе в безіменній області 0xa33e0–0xa4124 mik.ko).
3. `dtv_driver.ko`: `MTGADEC_MTAUD_SetDecType/GetDecType` + `_GlueDecTypeTransfer` — фінальний мапінг у DSP-enum; перевірити, чи DTS-значення присутнє.
4. Перевірити розділ `armfw` (DSP firmware) на наявність DTS-декодера (рядки "DTS"/"dca" у образі) — якщо декодера фізично немає, ніякий kernel-фік не допоможе, і єдиний шлях — bypass-обхід декодера (якщо DSP вміє RAW-канал у SPDIF TX).

## 5. Статуси

- BYPASS не розв'язує DTS; проблема в неактивному DTS-декодері DSP — **CONFIRMED**
- userspace (Kodi/HAL/парсер) коректні — **CONFIRMED**
- kernel передає codecType 9 без помилок — **CONFIRMED**
- точне місце відмови (utpa2k HAL_MAD гілка / dsp mapping / відсутність декодера у firmware) — **UNKNOWN, наступна фаза**
