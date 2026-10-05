# Лінія 01 — DD/SPDIF-пастрування ✅ (успішна)

**Статус: завершена успіхом.** Dolby Digital / E-AC3 пасстрування по оптиці працює.

## Суть
- Тракт: Kodi → AudioFlinger → `audio.primary.mt5889.so` → `libmi3` → `libutopia` → mik/utpa2k/dtv_driver → MAD DSP → SPDIF-тап на вихідній матриці DSP (реги 0x112D50–54).
- Два гейти, які блокували пастру:
  1. `MI_AUDIO_GetCaps` повертав 0 → Kodi не бачив RAW-можливостей. Фікс: 20-байтовий патч libmi3 (OR caps-бітів; v1 0x2E0 → v2 0x2E1 → v3 0x070002E1).
  2. Kodi `passthroughdevice` не персистився → ручне `AUDIOTRACK:AudioTrack (RAW)` у guisettings.
- Kodi IEC61937-транспорт неможливий архітектурно (HAL/policy без IEC-профілю, "-38" від AudioPolicy) — тому тільки offload-RAW через HAL.
- Приховане кермо SPDIF — звичайний рядок «Цифровий вихід» (`g_audio__spdif`; значення: off=0, RAW/ARC=1, PCM/ARC=2, AUTO=6), досяжне, але саме по собі формат не змінює.

## Ключові файли цієї папки
- `REPORT_SPDIF_audio_path.md` — карта тракту (фундамент).
- `REPORT_PASSTHROUGH_FIX.md` — фаза 0: сам фікс.
- `REPORT_paths_matrix.md` — матриця транспортів + фолбек-транскод.
- `REPORT_hal_gate1_experiment.md`, `REPORT_kodi22_minimal_patch.md`, `REPORT_source_audit_kodi22_iec.md`, `REPORT_hal_iec_patch_prepared.md` — IEC61937-лінія (відхилена як архітектурно хибна).
- `REPORT_final_verdict.md` — вердиктна таблиця епохи.
- `forensic_space_bunny_spdif_*` (09-25/26) — карта кермувалів, живі тести приватного API, Netflix-гейт.

## Практичний вихід
Дивись репозиторій [projector-passthrough-dd-restore](https://github.com/vladikit-bit/projector-passthrough-dd-restore) — гайд відновлення після OTA + скрипт + бібліотеки.

## Перехрестя
- `dtspassthrough` bug/fallback → DTS-лінія (02).
- EDID-біт 0x80 → гейт в `_MI_AOUT_SetHdmiAutoMode` (02, T2).
- «HAL нерухомий» зі SPDIF-керма → підтверджено host_exonerated (02).
