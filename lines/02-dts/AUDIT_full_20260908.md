# ПОЛНИЙ НЕЗАЛЕЖНИЙ АУДИТ — DTS/SPDIF forensic investigation

**Дата:** 2026-09-08 · **Обсяг:** 61 REPORT*.md + r*_out + runtime_phase* + static notes
**Режим:** тільки читання. Жодних patch / settings / EDID / topology / нових runtime-експериментів.

---

## 1. EVIDENCE LEDGER

### 1.1 VERIFIED BY STATIC (бінарний аналіз)

| Твердження | Джерело |
|---|---|
| Повний ARM-ланцюг: `mi_decoder_open@0x3f408` (0x0B000000→CodecType 9) → `MI_AUDIO_Start` ioctl `0xC0081005` → `MI_AUDIO_Write` ioctl `0xC038100A` (format-agnostic) → `mik.ko@0xb6e40` → `utpa2k _MApi_AUDIO_OpenDecodeSystem@0x402d04` → `MDrv_AUDIO_SetDecodeSystem@0x44249c` → `MDrv_AUDIO_CheckHashkey@0x423494` → `MDrv_AUTH_IPCheck@0x1390c` | REPORT_DTS_final_evidence_chain, deepdive_static |
| `Get_DTS_License` = IPCheck(0xf,0x3a,0x12,0x07) AND RIU `0x112CF0` bit7 | R10 §3 |
| Імена DTS SDO (`DTSSPDIFPackFrame`, `SDO_SpdifPacker_SAPI_SetIEC`, `IEC_Header`, …) — **assert-рядки в `.data`**, не символи; відсутні в `.symtab`/`.dynsym` | R24 reread (перевірено) |
| Рядок `'Invalid Spdif license:%d, output_spdifSz:%d'` — рівно 1 раз, у DEC-образі `mst_codec_r2_MS12V22.bin` @`0x144904`; блок формує DM-вказівник **`0x1c011000`** | R10 §3, R24 gate_sites |
| Поле `+0x2E6` читається у 5 місцях DEC-образу | R24 gate_sites (нове) |
| DEC-образ = той, що виконується (`md5 4b7e9509…` ≡ `/vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin`) | R10 §3 |
| SND DSP **не може читати OTP**: `'OTP fail : 0x1, 0x1000, 0xff00'` @SND `0x1287c3` | R10 §3 |
| R8: `r12 = 0x21A000`; `LD r23,0xFA(r12)` @`0x016F84` ↔ ARM SHM `+0xF8` = param `0x6E` (AC3/DTS-дискримінатор), у SPDIF output-parameter path | R8 (перевірено побайтово в R23-A.3) |
| DSP-образи **big-endian**; imm16 — молодші 16 бітів BE-слова; потік змішаний 16/32-bit; op `0x3B`=LOAD, `0x39`=CALL, `movhi`=`0x30`, `ori`=`0x3F` | R23-A.3, dec_state_transition |
| DM: формат `DM[0x%04x] = 0x%06lx` у `utpa2k.ko`; читання — host debug-шлях `HAL_MAD2_Read_DSP_sram` → `_HAL_MAD2_DBG_CMD_Read_DSP_sram` | R23-A.3 |
| Єдиний формувач DTS-over-SPDIF — SDO packer у SND R2-образі; хост не має API інжекції у SPDIF TX | REPORT_DTS_final_evidence_chain |

### 1.2 VERIFIED BY RUNTIME (спостереження на пристрої)

| Твердження | Джерело |
|---|---|
| **AC3**: `codec: ac3 … pass-through`, `RAW (PT) STREAM_TYPE_AC3`, AVR **DD lock** (E1 + E4, відтворено) | Phase E §5.1 |
| **DTS через Kodi** (`dtspassthrough=false`): `codec: dts … no pass-through` → `CAEEncoderFFmpeg AC3 encoder ready` → AVR ловить **DD, не DTS** | Phase E §5.2 |
| **DTS через probe7** (AudioTrack `ENCODING_DTS`, raw DTS): DSP `dts m6 hook ok` → `init ok, dts_licensee=0, lbr/xll/transcoder_licensee=0` → `dec play !!` → **тиша** (без lock, без шуму) | R9/R10 |
| Живі виключення: `alllic_diag` (key1/2=`0xffffff`) → licensees лишились 0; `dtslic_pass` (Get_DTS_License→0) → licensees лишились 0 | R10 §2 |
| R5/Phase E: патч `mik.ko` EDID→mode gate розгорнуто (md5 `03fc2c0d…`) → **DTS passthrough не відновлено**, AC3 без регресії | Phase E §8 |
| R23-A/A.2: відтворюваний DM-диференціал `0x0F45` / `0x0F4B` / `0x0F7B` (10/10 семплів) | R23-A, A.2 |

> Зауваження аудиту: `dts m6 …` / `dts_licensee=0` зафіксовані у DSP-логах на пристрої
> (`/data/AudioDECR2*`); копії цих логів у workspace **немає** — твердження спирається на
> текст звіту/пам'ять сесії, а не на артефакт у workspace.

### 1.3 STRONG INFERENCE

- **Тиша (не шум) = DSP output-stage license gate**: licensees=0 → придушення виводу. Підтримується тим, що (a) тиша, а не шум; (b) гейт розташований у output path; (c) усі ARM-джерела ліцензії виключені живом експериментом; (d) SND не читає OTP. **Не доведено**: умовний перехід, що живить гейт; сам print ніколи не спостерігався.

### 1.4 UNPROVEN / REJECTED

- Гілка гейта справді виконується під DTS — **не доведено** (print ніколи не бачили; перехід не декодовано).
- `+0x2E6` має license-семантику — лише proximity.
- `0x02F64C` читає SHM `+0x2E4` — **міжобразне припущення**: база `r12=0x21A000` доведена для **SND**-образу, а сайт знайдено в **CODEC**-образі.
- DM `0x0Fxx` як причина — provenance не доведена; крім того, DM-знімки зроблено під Kodi-конфігурацією, де DTS **транскодується в AC3**.
- Writer для SHM `+0x2E4` — не знайдено (ARM-trace незавершений).
- Виконання DTS SDO packer — не доведено (лише assert-рядки).

---

## 2. AC3 ↔ DTS EXECUTION MAP (лише доведене)

```
AC3 (проven повністю)
  Kodi → AudioTrack AC3 RAW (Android IEC packer)
  → HAL mi_decoder_open → MI_AUDIO_Start(5) → MI_AUDIO_Write
  → DSP: AC3 / MS12V2 decode ("ddp hook ok / init ok", без license-гейта)
  → output stage → SPDIF TX
  → AVR: DD LOCK                                            ✔

DTS (probe7 harness — єдиний, що реально доставляє DTS)
  AudioTrack(ENCODING_DTS, raw DTS)
  → MI_AUDIO_Start(9) → MI_AUDIO_Write
  → DSP: "dts m6 hook ok" / "init ok, dts_licensee=0" / "dec play !!"
  → ███ НЕ ДОВЕДЕНО ███
  → AVR: тиша, no lock                                      ✘

DTS (Kodi harness — поточний стан)
  Kodi dtspassthrough=false → "no pass-through" → AC3-транскод
  → DSP отримує AC3 → AVR ловить DD (не DTS)
```

**Останній безумовно доведений спільний етап:**
`MI_AUDIO_Write` → ioctl `0xC038100A` → ADEC ES-буфер (format-agnostic, доведено для обох).

**Перший етап, що лишається недоведеним:**
внутрішній **output stage DEC DSP після декодування** (емісія у SPDIF),区域 коду ~`0x021f00–0x021fea`, що формує DM-вказівник `0x1c011000`.

---

## 3. CANDIDATE CAUSES — статус

| Hypothesis | Evidence FOR | Evidence AGAINST | Status |
|---|---|---|---|
| DTS AUTH/license (ARM) | `MDrv_AUTH_IPCheck` існує; DTS-біти | 2 живі діагностики (`alllic_diag`, `dtslic_pass`) → licensees лишились 0 | **CLOSED** (ARM-джерело) |
| EDID gate | механізм реальний (Phase C) | Phase E: патч розгорнуто → DTS не відновлено; рішення приймається вище (Kodi/policy) | **CLOSED** |
| `MI_AUDIO_GetHandle` / transport | ioctl 0xC038100A | format-agnostic; обидва кодеки проходять | **CLOSED** |
| DM `0x0Fxx` | відтворюваний диференціал 10/10 | provenance/DSP-visibility не доведені; знімки під AC3-транскодом | **CLOSED** як причинна гілка |
| SHM `+0x2E4` / `Invalid Spdif license` | гейт локалізовано у output path; тиша, не шум; усі ARM-джерела виключені | перехід не декодовано; print ніколи не спостерігався; writer `+0x2E4` не доведено; семантика `+0x2E6` не доведена | **UNPROVEN — найближча до межі** |
| DTS SDO/IEC61937 | SDO packer існує (assert-рядки) | рядки ≠ код; немає адреси/caller/виконання | **UNPROVEN** |
| SPDIF mode/config | `HAL_AUDIO_SPDIF_*` налаштовуються | немає DTS-specific branch; mode/config лише | **CLOSED** |
| Physical SPDIF TX | AC3 працює фізично | — | **CLOSED** (апаратна справність доведена AC3) |

---

## 4. Перевірка R24 (окремо)

| Питання | Статус |
|---|---|
| Чи `0x021FDA` є conditional failure gate? | **НЕ ДОВЕДЕНО** — відома лише послідовність LOAD → матеріалізація рядка → CALL |
| Чи `+0x2E6` має license semantics? | **НІ** — лише proximity до print |
| Чи path виконується під DTS? | **НЕ ДОВЕДЕНО** — print не спостерігався в жодному логу |
| Runtime evidence виконання? | **НЕМАЄ** — `'Invalid Spdif license'` присутній лише в образах/статичних сканах |
| Чи `output_spdifSz` змінюється через цей path? | **НЕВІДОМО** |
| Доведений causal link до SDO/SPDIF suppression? | **НІ** |

## 5. Стан незавершеного ARM trace

`HAL_SND_R2_Set_SHM_PARAM @0x003A342C` (`libutopia.so`, справжній експортований символ).
Перший прогін з `-noanalysis` дав лінійний disassembly, що зайшов у switch/data region —
результат **недійсний**. Твердження «0x2E4 не знайдено» **не є** доказом відсутності writer.
За наказом користувача trace **зупинено**; відновлювати його слід лише якщо аудит визнає
високий information value (див. §7 — не визнано).

---

## 6. FINAL AUDIT (висновок)

**PROVEN**
- AC3: повний шлях від Kodi до DD-lock.
- ARM-ланцюг до `MI_AUDIO_Write` — спільний і format-agnostic.
- DTS доходить до DSP-декодера (probe7) і декодер запускається (`dec play !!`).
- Усі ARM-джерела ліцензії виключені живом експериментом.
- EDID/mode gate — не блокуючий шар.
- Гейт `'Invalid Spdif license'` локалізовано в DEC-образі у output path (DM `0x1c011000`).

**CLOSED / REFUTED**
- ARM AUTH/license · EDID gate · IDMA key masks · userspace libmi3 · image variant ·
  mdb CS-reg poking · `MI_AUDIO_GetHandle`/transport · SPDIF mode DTS-branch ·
  physical SPDIF TX (справний) · DM `0x0Fxx` як причинна гілка.

**STILL UNKNOWN**
- Чи виконується гейт під DTS (перехід не декодовано, print не спостерігався).
- Чи виконується DTS SDO packer і чи доходить burst до SPDIF TX.
- Writer SHM `+0x2E4` і семантика `+0x2E6`.

**CURRENT EXECUTION BOUNDARY**
> **DEC DSP output stage після декодування** —区域 ~`0x021f00–0x021fea`, DM-вказівник `0x1c011000`,
> що містить print `'Invalid Spdif license:%d, output_spdifSz:%d'`.
> AC3 цей етап проходить доведено; DTS — не доведено.

**ONE BEST NEXT TEST**
> Прочитати **вже наявні** DSP-логи з пристрою: `adb pull /data/AudioDECR2*` (і `/data/AudioSNDR2*`)
> з попереднього probe7-прогону (R9/R10) і перевірити:
> (a) чи був emit рядок `'Invalid Spdif license …'` під DTS;
> (b) значення `output_spdifSz`;
> (c) чи відсутній такий рядок в еквівалентному AC3-лозі (`decType 0x81`).
> Це лише читання вже зібраних файлів: жодного нового відтворення, патча чи зміни налаштувань.

**WHY THIS TEST IS MORE INFORMATIVE THAN THE PREVIOUS ONES**
> Кожен попередній кандидат (`0x0Fxx`, `+0x2E4`, SDO-рядки) не мав **або** provenance,
> **або** runtime-підтвердження. Цей тест б'є точно в єдиний механізм, який (i) розташований
> на доведеній межі виконання, (ii) узгоджується з «тиша, а не шум», (iii) не потребує
> декодування невідомого AEON-переходу: наявність надрукованого рядка довела б виконання гейта
> напряму. І він використовує вже існуючі дані — найдешевший з можливих.
