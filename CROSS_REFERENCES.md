# Перехресні посилання, суперечності та ланцюги спростувань

Все тут — зведене з первинних звітів. **Разом задокументовано 40+ перевірених гіпотез: 23 ланцюги спростувань у таблиці нижче + 20+ підтверджених висновків у SYNOPSIS.md кожної лінії (lines/01..05). Хронологія — TIMELINE.md.**

 (повні дигести — в SYNOPSIS кожної лінії).

## 1. Головні ланцюги «гіпотеза → спростування»

| # | Гіпотеза (файл) | Спростування (файл) | Чим |
|---|---|---|---|
| 1 | EDID SAD-бітмапа — диференціатор AC3/DTS (REPORT_static_final, T2) | REPORT_runtime_phase1 (R1) | AC3 працює при нульовому EDID-caps; гейт реальний, але не причина тиші |
| 2 | ARM-ліцензія (CheckHashkey/IPCheck) — блокер DTS | REPORT_runtime_phase4 (R4) §6.3; REPORT_r10; REPORT_tier0_r2log_runtime | Exp-B задеплоєний → тиша лишилась; діаг-ядра: licensee=0 в DEC-DSP |
| 3 | 0xb/0xc = DTS IPID; 0x43d/0x43e = DTS-прапори | REPORT_full_license_profile | Це Dolby AC3/DD+ і Dolby-premium; DTS = {0xf,0x3a,0x12,7} |
| 4 | «Гейт 0x25EE1 FAIL ⇒ немає TX» | REPORT_runtime_phase32; REPORT_runtime_phase46 | Return-path вирішує; MMIO пишеться до гейту; 0x13292 — загальна арифметика (12 сайтів) |
| 5 | Pc=0x0001 (AC3-константа) — корінь DTS-тиші | r57_npcm/FINDING_dts_pc_REFUTED_by_experiment | Живий тест: Pc=0x0B на дроті — TX все одно 0 Б |
| 6 | L2 capability-гейт — останній блокер | r57_npcm/FINDING_gate_L2_pre_experiment §9 | Патч задеплоєний — БЕЗ ЕФЕКТУ |
| 7 | «libmi3 v3 зламав DTS, який працював» | r57_npcm/FINDING_last_blocker_libmi3_regression | Ретракція: раніше був транскод, не DTS |
| 8 | ipauth-бітмапа/провізіювання — вузьке місце | forensic_glm_e1_ipauth_live_experiment_20260929 §23 | FF-бітмап rc=0 за 6.4с до аудіо → DTS заблокований; SE перераховує 5→4 |
| 9 | Прошивка видає вердикт ліцензії | forensic_glm_ipauth_20260929_muse_report3 | 0 записів у 0xB000001C..1F у 7 образах; вердикт пише SE |
| 10 | +0x4F8 = ліцензія/DTS-слово | forensic_glm_ipauth_20260929_muse_report2; FINDING_host_exonerated_20260930 | Це TX-кодек-слово; хост озброюється однаково для AC3/DTS |
| 11 | Патч-офсети неможливо змапити | REPORT_dts_20261004 §1 | address == file offset; провал був зі старим лістингом + 4Б-вирівнювання |
| 12 | HWC — єдиний блокер 4K (C1+гейт = достатньо) | INDEPENDENT_HY4_HY3_DISPLAY_FORENSIC_REVIEW | setActivePanelFrequency мапить усе≠0 → 60Гц; потрібен ioctl 0xc008122c |
| 13 | MI_DISP_SetOutputTiming не викликається HWC / ActivePanelFrequency не кликається | INDEPENDENT_HY4_HY3 (eventWorkThread Є) | Мости існують; питання в ioctl-wiring |
| 14 | 4K120 / 1080p120 підтримується | INDEPENDENT_HY4_HY3 (таблиця 37 таймінгів) | Максимум 60 Гц будь-де |
| 15 | EDID лімітує внутрішні режими | REPORT_EDID_DISPLAY_MODES; FINDING_edid_4k_and_internal_modes | EDID = RX-адвертайзинг; 4K-профілі активні |
| 16 | «1080p бо EDID 1.4 у конфігу» | FINDING_edid_version_runtime_20261001 | Рантайм-глобал `tv_input_hdmi_edid_version=0` перекриває конфіг 2.1 |
| 17 | Kodi DynamicANWBuffer ламає DV | FINDING_dv_3d_static_20261001 | Зламано з будь-яким плеєром |
| 18 | dolby.bin = заглушка / тюнінгу просто нема | FINDING_dv_pq_bin_gate_20261001; REPORT_dv_static_20261004 | Бінарник без customer IP mode; драйвер не читає файли взагалі |
| 19 | Панельні INI керують списком режимів HWC | FINDING_modes_consolidated_20261001 | Літерали @0x3BC88 |
| 20 | «0x103F/0x105C = живий ринг» | FORENSIC_dts_final_state_20260730 (ретракція) | Таймінг- counters, не ринг |
| 21 | 205 self-loop BL = заводський виріз AUTH | dec_work/forensic_longcat_investigation_20260927 | Пре-релокаційний стан; при insmod заповнюється |
| 22 | «Прорив: біт 7 @0x112CF0» | REPORT_session_retrospective_20260929 | Оголошено двічі, спострілено двічі (методологічна заборона) |
| 23 | /vendor audio_bin/*.bin керує DSP | REPORT_phaseC_corpus_synthesis_20260922 (LOADER TRUTH) | DspLoadCode копіює вбудовані блоби utpa2k.ko → /vendor патчі ІНЕРТНІ |

## 2. Спільні/дубльовані файли між лініями
- `FINDING_dv_3d_static_20261001.md` — 03 (DV) і 05 (3D).
- `FINDING_video_g_namespace_and_3d_20261001.md` — 05, і його §10 — база для 03/04.
- `REPORT_static_final.md`, `REPORT_runtime_phase1.md`, `FINDING_host_tx_chain/host_exonerated` — 02, стосуються 01.
- `EDID_FORENSIC_ANALYSIS.md` — 05 (3D-профілі EDID) і 04.
- `ROLLBACK.md` — 03 (стан пристрою), але стосується всіх ліній.
- `VIDEO_SETTINGS_CHEATSHEET.md` / `FACTORY_MENU_CHEATSHEET.md` — 04, DV/«молоко»-лід у 03.

## 3. Лінії ↔ інші репозиторії
| Лінія/тема | Де |
|---|---|
| Відновлення AC3-фіксу | `projector-passthrough-dd-restore` |
| Патч-білдери, свипи, Skill R2-log | `projector-diagnostic-tools` |
| TCL-донорські дифи (V083/V098/V474) | `tcl-t615t-firmware-analysis` |
| Усі .md як є | `projector-research-base` |
| Сесійні архіви Union Alpha | `projector-research-base/Бекап розслідування/...` (+ локально GLM_archive) |

## 4. Нерозв'язані суперечності (зафіксовані у звітах, не зняті)
- Р26-B «невірно сформований DTS-burst» проти 09-30 «дві runtime-умови»: сумісні (P-A-burst'и були 0.4с/2.18с), але причинний механізм між ними не з'єднаний байт-в-байт.
- «IPAUTH-лінію треба перевідкрити» (FINDING_host_tx_chain) проти «хост виправдано втретє» (FINDING_host_exonerated, той самий день) — напруга не знята жодним пізнішим файлом.
- Android-версія: звіти дослідження кажуть Android 11 / kernel 4.19.116; окремі сирові getprop-знімки в platform-tools містять 4.4.4-fingerprint (імовірно від іншого пристрою/заводу) — орієнтуйтесь на звіти.
