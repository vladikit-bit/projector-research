# MT5889/C50A: найпростіший практичний шлях DTS bitstream → SPDIF TX (transport-only)

Дата: 2026-08-30. Статус: **тільки розслідування**, пристрій не модифікувався.
Базові факти взяті з попередніх фаз (REPORT_DTS_final_evidence_chain.md та ін.).
Нові докази цього етапу отримані статичним розбиранням `kmods/utpa2k.ko`,
`kmods/mik.ko` (з .symtab і релокаціями) і сорсів Kodi Omega.

**Уточнення адрес**: адреси в старих Ghidra-декомпіляціях utpa2k.ko зсунуті на
+0x1D624 віддійсних символьних адрес. Нижче всюди **реальні** адреси з .symtab
(напр. `MDrv_AUDIO_CheckHashkey` = 0x423494, а не 0x440ab8).

---

## 0. ГОЛОВНИЙ ВИСНОВОК / DO THIS FIRST

**Патчити `MDrv_AUDIO_CheckHashkey()` в utpa2k.ko — 7 слів (28 байтів) ARM-коду,
які змушують DTS-стан виглядати ліцензованим. Все інше вже готове.**

Виявлено три факти, які разом роблять рішення майже безкоштовним:

1. **Єдине джерело DTS-заборони — host-side.** `MDrv_AUTH_IPCheck` (utpa2k.ko
   @0x1390c, 0xd8 байт) — це не криптографія, а **зчитування біта з готової
   бітової маски** `gIpAuthVars` (заповнюється AUTH-підсистемою). `MDrv_AUDIO_CheckHashkey`
   (0x423494) просто розкладає ці біти в `g_AudioVars2`. Жодних перевірок у шляху
   старту декодера (`MDrv_AUDIO_OpenDecodeSystem` 0x405cc8 — обгортка 0x1c байта,
   `SetDecodeCmd`-ланцюг, `mi_hwcaps_GetAudioCaps` mik.ko 0x2b907c — чип-капабиліті)
   **немає**. Маска доставляється в R2 через SE-IDMA (`HAL_AUDSP_CheckSeIdmaReady`),
   тобто R2 **довіряє** тому, що записав хост.
2. **Kodi 21.2 (оригінальний білд) вже вміє слати RAW DTS.** У
   `AESinkAUDIOTRACK.cpp` є другий сінк **"AudioTrack (RAW)"**
   (`m_wantsIECPassthrough=false` → `m_encoding = CJNIAudioFormat::ENCODING_DTS`,
   сирі DTS-кадри без IEC-заголовків). На цьому пристрої перелічений **саме він**
   ("Enumerated AUDIOTRACK devices: only AudioTrack (RAW)"). Потрібно лише вибрати
   його в налаштуваннях Kodi.
3. **audio policy вже декларує AUDIO_FORMAT_DTS/DTS_HD** (offload + hw_av_sync
   профілі у vendor XML; наш апатчений XML з профілем IEC61937 їх зберіг).
   HAL-мапа 0x0B000000 → CodecType 9 існує в ROM-коді HAL; бракує лише
   capability-бітів DTS-сім'ї у `MI_AUDIO_GetCaps` — і **патч libmi3 v3
   (біти 24/25/26) вже приготований раніше** (`libs/libmi3_patched_v2.so`,
   md5 b5425941).

Мінімальний план (все оборотне, root достатній):
1) замінити `libmi3.so` на приготований v3 (два раніше приготовані байти уже
   перевірені методикою); 2) підмінити `utpa2k.ko` на патчений (7 слів,
   специфікація в §4); 3) ребут; 4) у Kodi вибрати вихід "AudioTrack (RAW)";
   5) програти DTS-файл. Очікуваний ланцюжок: RAW DTS → ENCODING_DTS → HAL
   format 0x0B000000 → CodecType 9 → R2 DTS decode program стартує (маска
   "ліцензовано") → синхронізація на DTS core → SPDIF non-PCM bypass →
   **незмінений DTS payload → AVR**.

Єдине, що не доведено статично і перевіряється одним тестом: чи R2 DTS-програма
після отримання дозвільної маски реально синхронізується і вмикає bypass (Mode 2).
Аналогія з AC3 (працює на цьому ж пристрої тим самим механізмом) і наявність
DTS-галужень у SPDIF-логіці vendor'а роблять це сильно виведеним (strongly
inferred), а не доведеним.

---

## 1. Карта license-шлюзів (Android DTS request → SPDIF)

Класифікація: QUERY / CONFIG / GATE / DSP GATE / DATA PATH / OUTPUT PATH.

| # | Вузол | Адреса (utpa2k.ko, реальна) | Клас | DTS-block? | Доказ |
|---|---|---|---|---|---|
| 1 | HAL `mi_decoder_open`: 0x0B000000 → CodecType 9 | audio.primary 0x3f408 | CONFIG | ні (мапа є в ROM) | k_mi_decoder_open.c:85-95 |
| 2 | HAL gate1 `flags & 0x400` (IEC958_OPTIMAL) | audio.primary 0x21de8 | CONFIG | ні для Kodi RAW-треку (прапорів 0x400 нема) | REPORT_hal_gate1_experiment |
| 3 | HAL gate2 `MI_AUDIO_GetCaps` family-біти | libmi3 (patch @VA 0x6263e) | GATE | **так, у стоку** | фаза 2; v2 для AC3, v3 додає DTS-біти 24/25/26 |
| 4 | HAL gate3 `utils_isPassthroughSupported` | audio.primary | GATE | так у стоку (сім'я DTS відсутня) | REPORT_paths_matrix §1 |
| 5 | audio policy профілі | audio_policy_configuration.xml | CONFIG | ні — DTS/DTS_HD вже декларовані (offload, hw_av_sync) | XML рядки 81/84/123/126 |
| 6 | `MI_AUDIO_Start(9)` → mik.ko `_MI_AUDIO_Internal_Start` | mik 0xb6e40 | DATA PATH start | ні | k_mik_decomp.c; `mi_hwcaps_GetAudioCaps` (0x2b907c) = чип-капабиліті, без ліцензії |
| 7 | `_MI_AUDIO_CodecTypeMapDecoderType(9)=0xb` | mik 0xb5ea4 | CONFIG | ні | k_mapfun_decomp.c |
| 8 | `MApi_AUDIO_SetDecodeSystem` → `MDrv_AUDIO_SetDecodeSystem` | **0x424e78** | GATE (композитор) | — | ARM-розбирання: CheckHashkey → push масок → pFuncPtr_Setsystem → SPDIF_SetMode |
| 9 | **`MDrv_AUDIO_CheckHashkey`** | **0x423494** | **GATE — центральний** | **так** | 20× `MDrv_AUTH_IPCheck` → g_AudioVars2[0x440]/[0x444]/[0x43d]/[0x43e]/[0x4d0..] |
| 10 | `MDrv_AUTH_IPCheck` | **0x1390c** | GATE (джерело) | так | читає біт `id` з `gIpAuthVars`-маски (ldrb+lsr+and), повертає 1=ліцензовано |
| 11 | **`MDrv_AUDIO_ApplyHashkey`** | **0x424b90** | DSP GATE transfer | так | (0, mask[0x440]) і (1, mask[0x444]) → func @0x424bd8: GetDSPalive → mutex → `AbsWriteMaskReg(0x112AC0)` → **SE-IDMA** (CheckSeIdmaReady) → R2 |
| 12 | `HAL_AUDIO_SetSystem2`, DTS-гілка | 0x44d0b0, читання 0x43d @0x44d2c0 | DSP GATE (конфіг байта) | так | `AbsWriteByte(r6, 0x43d==1 ? 5 : 0xd)` у DTS-кейсі (r6 = 0x112E98-база) |
| 13 | `HAL_AUDIO_SPDIF_SetMode` | 0x448a10 | OUTPUT PATH config | ні | dispatch: 0/1→PcmMode, 2→**BypassMode**, 3→TranscodeMode, Auto; викликається з SetDecodeSystem |
| 14 | `HAL_AUDIO_SPDIF_AutoMode` | 0x444914, читання 0x43e @0x444c98 | OUTPUT PATH | ні для прямого bypass (це DTV-автологіка) | при 0x43e!=1 пропускає DTS-гілку (HAL_MAD_GetDTSInfo → g_u32bDTSCD) |
| 15 | `MDrv_AUDIO_Get_DTS_License` | 0x422aa0 | QUERY | **ні** | IPCheck(0xf,0x3a,0x12,7) AND регістр 0x112cf0[7]; викликається лише з Get_Decoder_Support/Get_License (капабиліті-звіти) |
| 16 | strap-регістр 0x112cf0 bit7 | — | QUERY | ні | у всьому .text немає ЖОДНОГО запису в 0x112cf0; читають тільки Get_*_License і debug-дамп |
| 17 | R2-внутрішня перевірка маски | mst_snd_r2 | DSP GATE | **невідомо детально (AEON ISA)** | маска доставлена SE-IDMA; R2 довіряє вхідній масці — сильно виведено |
| 18 | `MDrv_AUDIO_SetAudioParam2(param=3/0x12)` | 0x424d3c | DSP GATE re-push | так | перераховує CheckHashkey+маски на кожен такий параметр |
| 19 | `MDrv_AUDIO_Debug_Cmd_Read` | 0x41xxxx (кілька) | QUERY (верифікація) | — | дампить 0x440/0x444/0x43d/0x43e — готовий інструмент перевірки після патчу |

Ключові поля `g_AudioVars2`: **[0x440] = маска "IP відсутні"** (bit0 = DTS
відсутній: ставиться при fail IPCheck(0xb), ** стирається** при pass IPCheck(0xc));
**[0x444] = маска "IP доступні"** (init 0x00ffffff, біти стираються при fail);
**[0x43d]/[0x43e] = DTS-рівень 1/0** (споживачі: SetSystem2 #12, AutoMode #14,
GetAudioInfo2, TranscodeMode+0x6c0); [0x4d0] = MS12/Dolby рівень (вибір
базові/MS12V22 R2-образи в `HAL_AUDSP_DspLoadCode` @0x46f5a8 — **не чіпати**,
щоб не переключити R2-програму).

## 2. Mode 1 vs Mode 2 (декодування vs bypass)

Статично встановлено:
- `HAL_AUDIO_SPDIF_SetMode` має явний **BypassMode** (0x444f40) і **AutoMode**
  (0x444914 з окремою DTS-гілкою + `HAL_MAD_GetDTSInfo`/`g_u32bDTSCD`) — vendor
  реалізував DTS-специфічну non-PCM логіку виходу.
- R2-образ `mst_snd_r2` містить **SDO SPDIF packer**, який приймає готовий DTS
  кадр (`dtsParseFrame`) і формує SPDIF-вихід, з параметром
  `DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32` (raw DTS для фізичного SPDIF
  vs IEC-обгортка) — тобто **Mode 2 машинерія присутня і призначена саме для
  passthrough**.
- Емпірія AC3 на цьому пристрої: decode program стартує → компресований AC3
  виходить на SPDIF (DD на AVR). Це той самий bypass-механізм output matrix.

Висновок: старт DTS decode program'и потрібен як **вмикач** входу/синхронізації/
SPDIF-машинерії (Mode 2), декодування до PCM не обов'язкове — але це
strongly inferred, фінально підтверджується одним runtime-тестом (§7).
Якщо R2 раптом вимагатиме повного декодування — Mode 2 через SDO packer усе
одно видає компресований кадр у spdifFrmBuf (пакер не декодує), тож payload
залишиться незміненим.

## 3. Розв'язки A–H: ланцюжки і статуси

### A. Патч license gate + штатний vendor bypass  → **РЕКОМЕНДОВАНО**
```
DTS-файл → Kodi (RAW DTS, ENCODING_DTS)
  → AudioFlinger (offload port; DTS профіль вже в policy)
  → HAL adev_open_output_stream (gate2/gate3 ← libmi3 v3 біти)
  → mi_decoder_open: 0x0B000000 → CodecType 9
  → MI_AUDIO_Start(9) → mik.ko → MApi_AUDIO_SetDecodeSystem
  → MDrv_AUDIO_SetDecodeSystem(0x424e78)
      → MDrv_AUDIO_CheckHashkey(0x423494) [ПАТЧ: DTS=ліцензовано]
      → ApplyHashkey(0x424b90) → SE-IDMA → R2 маска "DTS дозволено"
      → pFuncPtr_Setsystem → HAL_AUDIO_SetSystem2 → байт 5 (замість 0xd)
      → HAL_AUDIO_SPDIF_SetMode → BypassMode
  → MI_AUDIO_Write: сирі DTS кадри → ES буфер (libaudioparser dts_parser
    коректно рахує frame size для 0x7FFE8001)
  → R2 DTS decode program: sync → bypass/pack → SPDIF TX → AVR
```
Статус: усі ланки proven, крім останньої (R2 sync→bypass) — strongly inferred.

### B. Доставка кадру в SDO packer зовні (D1/D2 питання)
Хост↔R2 канали: `HAL_DEC_R2_Set/Get_SHM_PARAM`, `MApi_AUDIO_SetAudioParam2`,
`SetCommAudioInfo` (командні id), SE-IDMA (тільки маски). Рядок `D2A_DTSXENC`
(у R2-образі) — статусне повідомлення R2→хост (encType/spkrMask/packType),
не інжекція. Викликів file-player/Xcoder з хоста нема (0 xref-ів поза образом).
**Статус: D2 (packer приймає кадри лише від внутрішніх decoder/Xcoder) —
architectural restriction, обходження вимагало б реверсу AEON-коду. Клас C/D
переможені варіантом A, який досягає того ж результату штатним шляхом.**

### C. Прямий IEC61937 з Android повз декодер
Мертво з двох причин (обидві proven): 1) HAL/мікшер не має шляху даних у SPDIF
TX окрім ES-буфера декодера; 2) IEC-обгорнуті дані ламають parser framing /
sync декодера (експерименти фаз 3–4). BREAK POINT: `MI_AUDIO_Write → ES → ADEC`.

### D. Kernel/Utopia інжекція
`HAL_AUDIO_SPDIF_*`/`HAL_DEC_R2_*` — config/param API; жодної функції
"байти → SPDIF FIFO" у хост-частині не існує (consumer-map + symbol scan).
Низькорівневий записач — лише R2-програма. BREAK POINT той самий.

### E/F/G. Пакер напряму (host-controlled R2 / заміна буфера)
Вимагає AEON-реверсу і шину команд, якої нема в хост-обгортках →
невиправдано дорожче за A при тому ж результаті (A видає той самий
DTS-over-SPDIF штатним шляхом).

### H. Софт-фолбек (Kodi AC3 transcode) — **НУЛЬ патчів**
`ActiveAE.cpp:1655-1677`: при `passthrough && ac3passthrough && ac3transcode`
Kodi **декодує DTS софтом → кодує AC3 (ffmpeg) → IEC AC3** — через вже робочий
AC3-шлях. Тільки налаштування Kodi (Guі: увімкнути "Transcode to Dolby
Digital"/AC3 transcode + AC3 passthrough). Ціна: перекодування (якість
1536kbps DTS → 640kbps AC3), CPU/затримка — некритично для transport-використання.

## 4. Специфікація патчу A (утpa2k.ko, 7 слів у `MDrv_AUDIO_CheckHashkey`)

Сайт 1 — змусити pass IPCheck(0xb) (DTS core):
```
0x423514: 0a00000c (beq fail) → e1a00000 (nop)     ; fail-гілка maskA|=1 ніколи не береться
```
Сайт 2 — змусити pass IPCheck(0xc) (DTS-HD, стирає bit0 «DTS missing» у 0x440):
```
0x423594: 0a000011 (beq fail) → e1a00000 (nop)     ; завжди pass-шлях:
                                                   ; maskA &= ~1 ; 0x581 |= 1
```
Сайти 3–7 — примусово 0x43e=1 і 0x43d=1 (заміна блоку фінальних сторів;
жертва — тільки debug-друк summary, який стрибаємо):
```
0x42487c: e5c0743e (strb r7,[r0,#0x43e]) → e3a07001 (mov r7,#1)
0x424880: e5c0a43d (strb sl,[r0,#0x43d]) → e5c0743e (strb r7,[r0,#0x43e])
0x424884: 0a000007 (beq log)             → e3a0a001 (mov sl,#1)
0x424888: e59004c8 (ldr r0,[r0,#0x4c8])  → e5c0a43d (strb sl,[r0,#0x43d])
0x42488c: e3500003 (cmp r0,#3)           → ea000005 (b 0x4248a8)
```
Ефект: maskA bit0=0 (R2 отримує «DTS дозволено» через SE-IDMA), 0x43d=0x43e=1
(SetSystem2 пише байт 5, AutoMode/TranscodeMode включають DTS-гілки), Dolby/MS12
стан і [0x4d0] не змінюються (R2-образ залишається базовим). CheckHashkey
продовжує ранній вихід при 0x503=1 — патч діє з першого SetDecodeSystem після
завантаження модуля.

Ризики/контингенції:
- `HAL_AUDIO_CheckHashkeyDone(maskA)` (консистентність маски, panic при -22):
  стерти біти — валідна «ліцензована» комбінація; якщо все ж panic — додати
  відповідні біти в [0x444] (окремий микропатч, адреси є у розбиранні).
- Якщо R2 має власний strap-вестибіль (0x112cf0-подібний) — тест покаже
  «декодер стартує, але sync нема»; тоді ескалація на R2-рівень (дорогий шлях,
  ймовірність низька, бо маска доставляється саме для цього).
- Перевірка стану після патчу: `MDrv_AUDIO_Debug_Cmd_Read` дампить
  0x440/0x444/0x43d/0x43e; logcat: `MI_AUDIO_Start` друкує codecType,
  SetDecodeSystem/SPDIF_SetMode мають error-принти.

## 5. Порівняння scope (модифікації)

| Scope | Компоненти | Складність | RE-робота | Ребут | Root достатній | Персистентність | Стабільність | Confidence |
|---|---|---|---|---|---|---|---|---|
| **A: utpa2k.ko (7 слів) + libmi3 v3 + налаштування Kodi** | 2 файли vendor | **мінімальна** | 0 (все вже зроблено) | так | так | bind-mount/заміна файлу | висока (штатний шлях) | ~0.85 |
| B/C/D/E/F/G (SDO packer / інжекція / R2) | R2 firmware або новий TX | висока–дуже висока | AEON ISA реверс | так | так | firmware заміна | невідома | ≤0.3 |
| H: Kodi AC3 transcode | 0 (налаштування) | нульова | 0 | ні | так (для налаштувань) | guisettings.xml | висока | ~0.9 |

## 6. Фінальний рейтинг

| Rank | Рішення | Компоненти | DTS ADEC? | License bypass? | R2? | Android? | Kernel? | Складність | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| 1 | A: license-патч utpa2k.ko + vendor bypass + Kodi RAW sink | utpa2k.ko (28 B) + libmi3.so (готовий) | стартує (bypass, не декодує в PCM) | так (host-only, 7 слів) | ні | ні (лише вибір пристрою в налаштуваннях) | ні | мінімальна | 0.85 |
| 2 | H: Kodi settings → DTS decode + AC3 transcode → AC3 SPDIF | 0 | декодує софтом (fallback) | ні | ні | ні | ні | нульова | 0.90 |
| 3 | B/G: зовнішній кадр у R2 SDO packer | R2 firmware + шина команд | ні | так + R2 RE | так | ні | ні | дуже висока | 0.20–0.30 |

(Рішення C/D/E відпадають: той самий break point, що A, але дорожчі; новий
SPDIF-передавач з нуля — гірший за 1 і 2 у всіх метриках.)

## 7. Рішення

### 1) Найлегше — варіант A
- **Що патчити:** `utpa2k.ko` → `MDrv_AUDIO_CheckHashkey` (7 слів, §4);
  `libmi3.so` → приготований v3 (біти DTS/DTS-HD/IEC сімей).
- **Дані:** сирі DTS core кадри (Kodi "AudioTrack (RAW)", STREAM_TYPE_DTS_*).
- **Android:** без змін коду; **HAL:** без змін (gate1 не задіяний для RAW-треків,
  gate2/3 закриває libmi3 v3); **kernel:** заміна модуля utpa2k.ko (це і є патч);
  **R2:** без змін; **license gate:** нейтралізований на host-рівні;
  **root:** достатній (заміна/bind-mount /vendor/lib/modules + ребут);
  **ребут:** так (CheckHashkey кешується прапором 0x503);
  **надійність:** штатний vendor passthrough-механізм, той самий, що працює для
  AC3 → очікується стабільний; AVR декодує, projector = transport.
- **Перша перевірка після патчу:** AC3 passthrough не зламався → DTS-файл →
  `dumpsys`/logcat (`MI_AUDIO_Start` codec=9, відсутність error від
  SetDecodeSystem) → DD/DTS-індикатор на AVR.

### 2) Друге — варіант H (нуль патчів)
- Kodi (будь-який з трьох білдів): passthrough=on, AC3 passthrough=on,
  **AC3 transcode=on**, DTS passthrough=off. DTS декодується ffmpeg'ом Kodi,
  перекодовується в AC3, йде через вже робочий AC3→SPDIF→DD ланцюжок.
- Компроміс: втрата якості (re-encode 640 kbps), CPU; перевага: жодних змін
  пристрою, працює негайно.

### 3) Останній резерв — B/G
- Реверс AEON R2 (mst_snd_r2), пошук команди прийому зовнішнього кадру в SDO
  packer або патч R2-образу. Лише якщо A раптово блокований R2-внутрішнім
  strap-вестибілем, а якість H неприйнятна.

---

## 8. Порядок дій (коли буде дозволено змінювати пристрій)

1. Зберегти оригінали: `utpa2k.ko`, `libmi3.so` (їх md5).
2. Застосувати патч §4 до utpa2k.ko (офлайн, бінарний), підмінити модуль
   (bind-mount методика з фаз 2–3), libmi3 → v3, ребут.
3. Налаштування Kodi → вихід "AudioTrack (RAW)" (якщо не вибраний).
4. Тест 1: AC3-файл → DD на AVR (регресія відсутня?).
5. Тест 2: DTS-файл → індикатор DTS на AVR; у logcat перевірити
   `MI_AUDIO_Start ... 9`, відсутність `CheckHashkey`/`SetDecodeSystem` error.
6. Якщо T2 мовчить, а декодер стартував: зняти `MDrv_AUDIO_Debug_Cmd_Read`
   дамп масок → ескалація за §4-контингенціями.
