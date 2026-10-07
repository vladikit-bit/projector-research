# Дослідження проектору Thundeal TD.98 Pro (MStar MT5889) — головний репозиторій

Форензик-реверс інженерія проектора **Thundeal TD.98 Pro / C50A** (MStar MT5889, Android 11, ODM Ntech), 2026-08-26 → 2026-10-04. Мета — вичавити максимум поза заводськими обмеженнями: аудіо-пастру, Dolby Vision, режими картинки, 3D.

**Звідси починайте:**
1. [TIMELINE.md](TIMELINE.md) — відновлена історія по датах: що міняли, які гіпотези, що знайшли, як перевіряли, що відкинули.
2. [STATUS.md](STATUS.md) — поточний стан кожної лінії і наступні кроки.
3. [CROSS_REFERENCES.md](CROSS_REFERENCES.md) — 23 ланцюги «гіпотеза → спростування», дублі, нерозв'язані суперечності.
4. [DEVICE.md](DEVICE.md) — паспорт пристрою.

## П'ять лінії (`lines/`)

| Папка | Лінія | Статус | Суть |
|---|---|---|---|
| `01-dd-spdif/` | DD/SPDIF-пастру | ✅ успіх | libmi3 caps-патч + Kodi RAW → AC3/EAC3 працює (SYNOPSIS) |
| `02-dts/` | DTS-пастру | ⏳ відкрита | декодер ОК, блок = 2 runtime-слова в DEC-DSP; 170 документів |
| `03-dolby-vision/` | Dolby Vision | ⏳ калібрування | залізо є; dolby.bin без customer IP mode; NLA.ini-калібрування |
| `04-picture-modes/` | режими картинки | ⏳ патч готовий | 2 захардкоджені 1080p50/60 у HWC; платформа вміє 4K@24-60; патч ~250 Б специфікований |
| `05-3d/` | 3D | ⏳ не почата | DLP 120Гц-панель в образі; Android-обв'язки немає |

Кожна папка: `SYNOPSIS.md` (стислий підсумок арки: підтверджене/спростоване/незавершене) + копії первинних звітів.

## Супутні репозиторії
- [projector-research-base](https://github.com/vladikit-bit/projector-research-base) — **всі 285 .md як є** (без сортування) + INDEX.
- [projector-diagnostic-tools](https://github.com/vladikit-bit/projector-diagnostic-tools) — скрипти/інструменти з документацією.
- [projector-passthrough-dd-restore](https://github.com/vladikit-bit/projector-passthrough-dd-restore) — відновлення AC3-фіксу після OTA (гайд + скрипт + libmi3).
- [tcl-t615t-firmware-analysis](https://github.com/vladikit-bit/tcl-t615t-firmware-analysis) — донорські TCL-прошивки (дифи для DTS-лінії).

## Методологічна примітка
Первинні документи — логи AI-сесій (код-назви GLM/forensic_space_bunny/MUSE/LongCat/Union Alpha), збережені як є; свідомо залишено помилки та їхні подальші виправлення — це частина доказового ланцюга. Консолідована, вивірена картина — тут. Сесійні архіви повних траєкторій: `projector-research-base/Бекап розслідування/` (7 FULL_UNION + 13 субагентських траєкторій, snapshot 2026-09-18).
