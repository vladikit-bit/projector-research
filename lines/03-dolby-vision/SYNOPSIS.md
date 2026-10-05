# Лінія 03 — Dolby Vision ⏳ (калібрування триває)

**Статус: корінь знайдено; колірне калібрування — в процесі, не доведене.**

## Підсумок
- **Залізо є**: 6 OMX-компонент DV (DVHE.DTR/.STN/.ST + AVC-enhanced + DVAV.SE, по 2 concurrent), vendor MStar engine (`FINDING_dolby_vision_hw`).
- **Metadata-конвеєр працює** — DV декодується і тонмапінг рухається.
- **Корінь зламаного кольору**: завантажений PQ-бінарник (`dolby.bin`) **не містить customer IP mode** → `IsSupport_GroupIPBIN → Support [0]` — калібрування блоковане на рівні сертифікованого бінарника; меню/конфіги/код не допоможуть (`FINDING_dv_pq_bin_gate`; підтверджено драйвером: `kdrv_dolby_vision.ko` не читає жодного файла — все через ioctl, `REPORT_dv_static_20261004`).
- **Рефакторинг причин**: «DynamicANWBuffer Kodi» спростовано — зламано з будь-яким плеєром; reboot лише перемикає симптоми («молоко» на старті / оранжево-фіолетовий на повторному вході). Робочий обхід: грати HDR10 base layer, ігноруючи DV-шар.
- **Колориметрія** (`DV_COLOUR_CALIBRATION.md`): зелений ×11 (0.431 vs 0.039), синій ×6, чорна підлога 17–25 замість 0, дефіцит насиченості −27%. План: NLA.ini Brightness[0] 88→0, Contrast[0] 30→0 поетапно.
- **Найсильніший слід «молока»**: PWM floor 143/479 = **29.9%** (u16MinPWMvalue 0x8F), а не 12.5% (`FACTORY_MENU_CHEATSHEET`, див. лінію 04).
- **Поточний стан пристрою** — `ROLLBACK.md`: активні HDR.ini `dv_apo_content_type_0..4 NR=0` (сток: `usb_pq/HDR_STOCK.ini`), 7 saturation-ключів; utpa2k.ko = dv3.

## Файли
`FINDING_dolby_vision_hw_20261001`, `FINDING_dv_no_pq_tuning_20261001` (містить виправлену помилку), `FINDING_dv_pq_bin_gate_20261001`, `FINDING_dv_3d_static_20261001`, `REPORT_dv_static_20261004`, `DV_COLOUR_CALIBRATION.md`, `ROLLBACK.md`.

## Перехрестя
- 3D-під-DV → лінія 05 (`FINDING_dv_3d_static` у 05 також).
- Два шари налаштувань → лінія 04 (`FINDING_video_g_namespace_and_3d`).
- Бекап `backup_20261003\` (kdrv_dolby_vision.ko, dolby.bin, dolby_factory.cfg) — локально в firmware_temp.
