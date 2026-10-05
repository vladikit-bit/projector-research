# Журнал змін на пристрої TD98 — для відкату

Усі зміни, які я зробив 2026-10-03/04. Кожна — з командою відкату.

## 1. Відкочено

| що | стан | відкат |
|---|---|---|
| `settings put global tv_picture_video_end_user_calibration_*` (11 ключів) | видалено → `null` | вже відкочено |

Чорний екран після цього запису. Причина не пояснена.

## 2. Активні зміни

### 2.1 `/vendor/cusdata/bsp/common/HDR/HDR.ini`

- **Через флешку**, `Factory PQ Update` з типом `Ini`.
- Що змінено: `dv_apo_content_type_0..4` → `NR = 0` (було 2, 0, 0, 1, 2).
- md5 на пристрої: `699db018904a35fb47421f0cc36ca9cc`
- Бекап оригіналу на хості: `usb_pq/HDR_STOCK.ini`, md5 `375fdb160982a5e0eb0063c781295d9f`
- Тримається **незмінним** на всю серію вимірювань, щоб не плутав результати.
- Відкат: покласти на флешку `HDR_STOCK.ini` і прогнати `PQ Update` ще раз.

### 2.2 `tv_picture_color_tune_saturation_*` (7 ключів)

- **Через `settings put global`**, без флешки й без перезавантаження.
- Значення: `red, green, blue, yellow, cvan, megenta, flesh_tone` = **60**, було 50.
- Відкат: `for c in red green blue yellow cvan megenta flesh_tone; do settings put global tv_picture_color_tune_saturation_$c 50; done`

## 3. Не змінювалося (залишено як було)

| що | значення |
|---|---|
| `tv_picture_color_tune_enable` | 1 (був 1) |
| `tv_picture_color_tune_hue_*` | 50 (7 кольорів) |
| `tv_picture_color_tune_brightness_*` | 50 (7 кольорів) |
| `tv_picture_color_tune_gain_*` | red/green/blue = 50 |
| `tv_picture_color_tune_offset_*` | red/green/blue = 50 |
| `tv_picture_video_end_user_calibration_*` | null |
| `picture_mode`, `picture_list_hdr` | не чіпав |
| `utpa2k.ko` | dv3, md5 `d28b1080122d804e111e3a799e575e8c` |

## 4. Вимірювальний baseline (той самий кадр, 01:41)

| стан | насиченість | макс Y |
|---|---|---|
| DV увімкнено | 15.2% | 148 |
| DV вимкнено | 20.9% | 159 |

Дефіцит DV: **−27% насиченості**.