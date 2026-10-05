# ПОВНИЙ FORENSIC AUDIT — хронологія та evidence ledger (R3→R24)

**Дата аудиту:** 2026-09-08 · **Режим:** лише workspace, без device-дій
**Джерела:** 61 REPORT*.md, r6/r8/r9/r11/r15/r16_out, r18–r22_out, r23a_out, runtime_phase1–5

---

## 1. ХРОНОЛОГІЯ ЕКСПЕРИМЕНТІВ

| Experiment / Phase | Date/time | Device state | Kodi state | Actual harness | Input format | HAL codec | DSP codec | What measured | Result |
|---|---|---|---|---|---|---|---|---|---|
| Phase 0 (AC3 fix) | 2026-08-26 | stock | Kodi 21 `org.xbmc.kodi` | Kodi | AC3 | `0x09000000` | codec 5 | caps patch `MI_AUDIO_GetCaps` | AC3 passthrough WORKS |
| Phase 1 (DTS forensic) | 08-29 | stock | Kodi | Kodi | DTS | `0x0b000000` | codec 9 | static chain | vendor ids 5/9; Kodi IEC paths dead |
| **T2 breakpoint** | 08-30 | Solution A (utpa2k patched + libmi3 v3) | **DTS PT ON** | **Kodi** | **real DTS PT** `STREAM_TYPE_DTS_512` | `0xb000000` | codec 9 `eRet:0` | runtime dmesg/dumpsys | **EDID→SPDIF mode gate found** (`_MI_AOUT_SetHdmiAutoMode`) |
| Exp A | 08-30/31 → 09-02/03 | utpa2k IPID `0x7d` NOP → `0x4d0=4` | — | Kodi | DTS | — | MS12V22 images | runtime | AC3 OK, **DTS fail** |
| Exp B (AUTH bypass) | ~09-04 deployed | `0x4d8=3`, `0x582=1` | — | Kodi | DTS | — | — | runtime | **DTS still fail** → ARM-licence hypothesis falsified |
| **R5 / Phase E** | 09-06 | mik.ko R5 patch `03fc2c0d…` | **transcode ON** (`dtspassthrough=false`) | **Kodi** | **DTS→AC3 transcode** | `0x09000000` (**AC3!**) | codec 5 | runtime logcat | gate NOT traversed → "falsified" = **UNTESTED, premature** |
| **R5b** | 09-06 | R5 patch | **user disabled transcoding** | **Kodi** | **real DTS PT** | `0x0b000000` | codec 9 | runtime | DTS reaches kernel as codec 9; **AVR DTS lock = NO** |
| cmd04 | 09-06 | Exp-B + dec cmd `0x97→0x04` | — | — | DTS | — | — | deployment | AVR lock pending/unrecorded |
| **R9 probe7** | 09-07 | utpa2k `f44ad0a4…`, mik `03fc2c0d…` | n/a | **Probe7** `AudioTrack ENCODING_DTS` | **real DTS raw elementary** | `0xb000000 AUDIO_FORMAT_DTS` | codec 9 | A/B windows | **DTS silence; AC3 → DD lock** |
| **R10** | 09-07 | same | n/a | **Probe7** | real DTS | — | dts_licensee=0 | static + DSP log | DSP-side license gate; 5 hypotheses eliminated |
| **R16** | 09-07 19:56 | — | n/a | **Probe7** `app_process Probe7 … ac3_51.raw` | **raw AC3 / raw DTS** | — | DEC R2 logs | DEC R2 log | `decType change 81→4`, `dts m6 init ok` |
| R17 | 09-07/08 | — | — | SND R2 + DM `0x0900–0x0C00` | — | — | — | logs | companion to R16/R20 |
| R18 / R19 | 09-07 | — | — | static DEC image | — | — | — | static | `0x81→0x4` NOT string-traceable; rodata `VA = file + 0x26CD000` |
| R20 / R21 | 09-08 (early) | — | — | static SND | — | — | — | static | `0xe30000` = **generic fill**, not DTS config |
| **R22** | **09-08 01:40–02:16** | — | **DTS PT working** | **real Kodi DTS** + probe7 | **DTS-real ×2, AC3-real** | `0x0b000000` | codec 9 | DM temporal | burst windows **IDENTICAL**; IEC active 100% both; **IEC packer located** |
| **R23-A / A.2** | **09-08 14:13–15:04** | post-reboot known-good | **UNKNOWN** | Kodi JSON-RPC | **UNKNOWN** | UNKNOWN | UNKNOWN | DM `0x0F40–0x0F50` | reproducible differential; **transport state NOT established** |
| R23-A.3 | 09-08 15:27+ | — | — | static | — | — | — | addressing | DM provenance UNPROVEN |
| R24 | 09-08 15:50+ | — | — | static DEC | — | — | — | gate sites | `+0x2E6` read at 5 sites; `+0x2E4` writer UNPROVEN |

### Ключові висновки з хронології

1. **Real-DTS harnesses (підтверджені):** T2 (Kodi PT), R5b (Kodi PT), **R9/R10/R16 (Probe7 raw elementary)**, **R22 (real Kodi DTS ×2)**.
2. **AC3-transcode harness:** R5/Phase E — DTS ніколи не дійшов як DTS; тому його "falsification" EDID-гейта **недійсна**.
3. **R23-A: формат НЕ встановлено.** `guisettings.xml` (host mtime 13:59 та 14:09, тобто за ~4 хв до R23-A) показує `dtspassthrough=false`, але **не доведено, що цей файл належить саме тому екземпляру Kodi, яким керували** (`net.kodinerds.maven.kodi22`); крім того, R22 того ж дня о 01:40 **успішно робив real-Kodi DTS passthrough**. Отже R23-A = **UNKNOWN**, а не "AC3 transcode" і не "real DTS".

---

## 2. EVIDENCE LEDGER (вибіркові ключові твердження)

| CLAIM | SOURCE FILE | EXACT EVIDENCE | DATE | EXPERIMENT | STATUS |
|---|---|---|---|---|---|
| AC3 passthrough працює → AVR DD lock | REPORT_phaseE_results §5.1 | `codec: ac3 … pass-through`, `RAW (PT) STREAM_TYPE_AC3`; E1+E4 | 09-06 | R5/Phase E | VERIFIED RUNTIME |
| Real DTS доходить у DSP як `AUDIO_FORMAT_DTS` | REPORT_r9 §4 | OFFLOAD thread, HAL format `0xb000000`, 67.9 MB | 09-07 | R9 probe7 | VERIFIED RUNTIME |
| DTS-декодер запускається | REPORT_r10 | `dts m6 hook ok` → `init ok, dts_licensee=0` → `dec play !!` | 09-07 | R10/R16 | VERIFIED RUNTIME (артефакт лише на пристрої) |
| DTS → тиша, AC3 → DD lock (A/B) | REPORT_r9 §5 + R10 §1 | WINDOW-D silent, WINDOW-A DD | 09-07 | R9/R10 | VERIFIED RUNTIME |
| EDID→SPDIF mode gate існує | REPORT_T2 §2 | `_MI_AOUT_SetHdmiAutoMode`, `tst r0,#0x80` → `SPDIF_SetMode(0)` | 08-30 | T2 | VERIFIED STATIC + RUNTIME |
| EDID gate НЕ є блокувальним | independent audit §2 Phase 5 | R5b: forced compressed + real DTS → AVR lock NO | 09-06 | R5b | VERIFIED RUNTIME |
| EDID gate "falsified" (Phase E) — помилкове | REPORT_phaseE §8 vs audit §2 | у Phase E був транскод → гейт не пройдений | 09-06 | R5/Phase E | **REFUTED (як falsification)** |
| ARM AUTH/license — не джерело | R10 §2 | `alllic_diag` (key1/2=0xffffff) і `dtslic_pass` → licensees лишились 0 | 09-07 | R10 | VERIFIED RUNTIME |
| SND не читає OTP | R10 §3 | `'OTP fail : 0x1, 0x1000, 0xff00'` @SND `0x1287c3` | 09-07 | R10 | VERIFIED STATIC |
| Гейт `'Invalid Spdif license…'` у DEC-образі | R10 §3, R24 | рядок 1× @`0x144904`; сайт `0x021FDA`; DM `0x1c011000` | 09-07/08 | R10/R24 | VERIFIED STATIC |
| Гейт виконується під DTS | — | **рядок відсутній у всіх захоплених логах** | — | — | **UNRESOLVED** |
| `+0x2E6` має license-семантику | — | лише proximity до print | — | — | HYPOTHESIS |
| `0x02F64C` читає SHM `+0x2E4` | R24 | база `r12=0x21A000` доведена для **SND**, сайт у **CODEC** | 09-08 | R24 | HYPOTHESIS |
| DM `0x0Fxx` diferenціал відтворюваний | R23-A/A.2 | 10/10 семплів, `0x0F45`/`0x0F4B`/`0x0F7B` | 09-08 | R23-A | VERIFIED RUNTIME (формат НЕвідомий) |
| DM `0x0Fxx` причинний | — | provenance не доведена; формат транспорту невідомий | — | — | **UNRESOLVED** |
| DM burst-config вікна ідентичні DTS vs AC3 | R22_belowDM §1 | `0x4ee0–0x4f30`, `0x0900–0x0910` — identical (`0x5b`) | 09-08 | R22 | VERIFIED RUNTIME |
| IEC61937 packer локалізовано в SND | R22_belowDM §3 | `"Packer"` `0x18232c`, `Pa=0xF872`, `Pb=0x4E1F`, **data-type table `0x13f181`: DTS=`0x0B`, AC3=`0x01`** | 09-08 | R22 | VERIFIED STATIC |
| `0xe30000` = DTS config | R11 (старе) | спростовано: це IEC-inactive transient | 09-08 | R22 | **REFUTED** |
| DTS SDO-імена = код | R24 reread | assert-рядки в `.data`, відсутні в symtab | 09-08 | R24 | **REFUTED** (як entry points) |
| `SpecifyDigitalOutputCodec*` мертвий код | audit §3 | нуль caller-ів | 09-06 | audit | VERIFIED STATIC |

---

## 3. CLOSED-гіпотези: WHY / TEST / RESULT / CHI САМЕ ЗАКРИВАЄ

| Гіпотеза | WHY WE THOUGHT THIS | TEST | RESULT | Чи справді закриває? |
|---|---|---|---|---|
| **AUTH/license (ARM)** | `MDrv_AUTH_IPCheck`, "Hash Key Check DTS Fail" | Exp-B AUTH bypass; `dtslic_pass`; `alllic_diag` | licensees лишились 0 | **ТАК для ARM-рівня.** Не закриває DSP-рівень (OTP недоступний DSP) |
| **EDID gate** | монітор ставить `SPDIF_SetMode(0)` без DTS-біта | R5 patch + R5b | гейт реальний, але forced compressed → lock все одно NO | **ТАК як єдиної причини.** Механізм реальний, але недостатній |
| **MI_AUDIO_GetHandle** | caps = 0 | libmi3 caps patch `0x070002E1` | AC3+ codec 9 проходять | ТАК |
| **MI_AUDIO transport** | формат-залежність | ioctl `0xC038100A` | format-agnostic | ТАК |
| **SPDIF mode/config** | `HAL_AUDIO_SPDIF_*` | relocation-аудит | AC3/DTS-симетрично; DTS-гілки немає | ТАК |
| **DM `0x0Fxx`** | diferenціал 10/10 | R23-A/A.2 + A.3 | provenance не доведена; формат транспорту невідомий | **НІ — лише deprioritized** (diferenціал лишається фактом) |
| **SHM `+0x2E4`** | DSP читає `+0x2E6` | R24 | writer не знайдено; міжобразне припущення | **НІ — UNRESOLVED** |
| **`Invalid Spdif license`** | гейт у output path; тиша не шум | R10, R16, R24 | гейт локалізовано, але виконання не доведено | **НІ — головний живий кандидат** |
| **DTS SDO/IEC61937** | SDO packer формує DTS-over-SPDIF | R22 (packer + data-type table) | packer локалізовано; чи виконується/що пише — невідомо | **НІ — другий живий кандидат** |
| **SPDIF TX (фізика)** | — | AC3 працює | апарат справний | ТАК |

---

## 4. CONTRADICTIONS

1. **"EDID gate FALSIFIED" (Phase E)** vs **"gate real but insufficient" (R5b)** — Phase E мав транскод, тому гейт не був пройдений. Коректне формулювання: *untested in Phase E; insufficient per R5b*.
2. **`Pb`: користувач вказав `DTS Pb=0x4E0B / AC3 Pb=0x4E01`;** R22 знайшов сталий `Pb=0x4E1F` і окрему **таблицю типів даних** (`DTS=0x0B`, `AC3=0x01`). Тобто `0x0B`/`0x01` — це **Pc (data-type)**, а не Pb.
3. **rodata mapping:** R10/R19 — `VA = file + 0x26CD000` (DEC); R23-A.3 обчислив `+0x26D0000` для рядка гейта. Розбіжність `0x3000` не з'ясована.
4. **R11 `0xe30000`** подавався як DTS-конфіг; R22 довів, що це IEC-inactive transient.
5. **База `r12=0x21A000`** доведена для SND, але застосовувалась до CODEC-сайту `0x02F64C`.
6. **"license gate не є причиною"** (формулювання користувача) узгоджується зі спростуванням **ARM**-гейта, але **DSP-side** гейт лишається провідним кандидатом — термінологічна колізія.
7. **R23-A** vs **R22**: R22 о 01:40 мав real-Kodi DTS PT; R23-A о 14:13 — формат невідомий. Не можна переносити висновки між ними.
