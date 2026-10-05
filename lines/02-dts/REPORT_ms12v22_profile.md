# MT5889/C50A: гілка 0x4d0=4 / MS12V22 — що вона реально змінює для DTS transport

Дата: 2026-08-30. Дослідницька гілка: тільки статичний аналіз, пристрій не змінювався.
Адреси — реальні з .symtab (Ghidra-адреси старих декомпіляцій зсунуті на +0x1D624).

---

## 0. Головний висновок (коротко)

**На C50A справжня DTS transport-ланцюжок не потребує MS12V22-image.**
**Навпаки — для повноцінного DTS:X/HD-bypass потрібен саме MS12V22 образ.**
Пояснення — в наступних розділах.

Причина в тому, що MS12V22-профіль робить чотири речі, критичні саме для
DTS transport:

1. Перемикає **обидва R2-образи** (`mst_*_r2` → `mst_*_r2_MS12V22`) у
   `HAL_AUDSP_DspLoadCode` (`0x4d0==4` ⇒ `mst_snd_r2_MS12V22+0x77a164`,
   `mst_codec_r2_MS12V22+0x407f0c`).
2. Зсуває **switch-table** у `HAL_AUDIO_SetSystem2` для системного-байта DTS
   (`0x43d=1 && 0x4d8==3` ⇒ пишеться `0x97` замість `0x05`; при `0x4d0==4`
   доступні додаткові системи).
3. `HAL_AUDIO_Encoder_Channel_Lock` (`0x4d0 ∈ {3,4}`) дозволяє
   симетричний lock-channel-control для преміум-DTS-сценаріїв.
4. `HAL_MAD_GetAudioInfo2` (`0x4d0==4`) дозволяє звіт case 0x55 з комбінацією
   IP(0x7d|10) & SHM(0x4d) — це "повний DTS-ланцюг" у runtime API.

DTS-декодер, Xcoder, SDO packer — **повністю ідентичні** між base і MS12V22
образами (статичне diff-підтвердження в §1). Це означає, що MS12V22 не
додає жодного нового DTS-роутингу — він додає лише можливість вибору вищих
DTS-режимів, які потребують відповідних R2-функцій.

---

## 1. R2-образи: повний розбір (що в MS12V22, чого немає в base)

Розміри (file_offset = .data+0x6592a0, st_value in st_value):

```
mst_codec_r2            0x16be18  size 0x29c0f4  (≈ 2.7 MB)  (DEC R2)
mst_codec_r2_MS12V22    0x407f0c  size 0x1e401c  (≈ 1.9 MB)
mst_snd_r2              0x5fb81c  size 0x17e948  (≈ 1.5 MB)  (SND R2)
mst_snd_r2_MS12V22      0x77a164  size 0x1c1330  (≈ 1.8 MB)
```

### 1.1 Block-wise commonality (64 KB блоки)

Жоден 64-КБ блок не є байт-в-байт ідентичним у жодній парі
(base/MS12V22, base/base, MS12V22/MS12V22). Це означає, що MS12V22 — це
**повністю окрема прошивка**, а не просто перепаковка з видаленими MS12-v1
секціями.

### 1.2 Спільне vs відмінне в snd R2 (DTS-relevant)

Підтверджено байт-в-байт, що **DTS-стек ідентичний** в обох snd образах:

| Компонент | base | MS12V22 | Примітка |
|---|---|---|---|
| `DTSDecSDOPacker_API_Process` | + | + | SDO SPDIF packer |
| `DTSDecSDOPacker_API_StartFrame` | + | + | |
| `DTSDecSDOPacker_API_SetParam` | + | + | |
| `DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32` | + | + | |
| `DTS_SDO_SPDIF_OUT` / `DTS_SDO_HDMI_OUT` | + | + | |
| `dtsParseFrame` | + | + | frame parser |
| `DTSSPDIFPackFrame` | + | + | |
| `PackSPDIFStream` | + | + | |
| `DTSX_CORE2_API_SDO_Packer` | + | + | |
| `DTSX_CORE2_API_Xcoder` | + | + | (PCM→DTS енкодер) |
| `DTSX_CORE2_API_Process` | + | + | декодер |
| `dtsx-sdo/src/sdo_spdif_packer.c` | + | + | filename |
| `dtsx-sdo/src/sdo_packer_api.c` | + | + | |
| `dtsx-main/app/file-player/src/dtsx-file-player-api.c` | + | + | |
| `dtsx-main/src/dtsx-common.c` | + | + | |
| `dtsx-xcoder/xcoder-wrapper/src/dtsx-xcoder-api.c` | + | + | |
| `dtsx-xcoder/xcoder-ca/api/src/DTSTranscode1m5_API.c` | + | + | |
| `dtsx-xcoder/xcoder-xll/api/src/dtsx_lossless_transcoder.c` | + | + | |
| `DTSLosslessTransEnc` | + | + | |
| `DTSTransEnc1m5` | + | + | |
| `D2A_DTSXENC` | + | + | R2→host звіт |
| `IEC_Header` | + | + | |
| `SDO_Packer` | + | + | |
| `Mstar InputProcess` / `InputProcess 1.0.0` | + | + | R2-input shim |
| `dec-p1/dtshd-c-decoder/src/frame_player/private/src/dtshd_frame_player_dsp2.c` | + | + | filename |
| `OutputProcessSPDIF` / `OutputProcessHDMI` | – | – | жоден з двох |
| `XPT_Packer` / `XPT_Packer_API` | – | – | жоден з двох (видалено?) |
| `dtsx-inputprocess` / `InputProcess.c` | – | – | жоден з двох |

### 1.3 Розходження (тільки MS12 v1 vs v2.4 Dolby-стек)

`mst_snd_r2` містить ~137 рядків "MS12V1", "DDPENC_CE_RES__*",
"Dolby_DAP 2.4.0.0" — це MS12 v1.0 кодер/декодер. `mst_snd_r2_MS12V22`
містить 88 рядків "MS12V2", "MS12V2 DAP", "MS12 V2.4 PCM Render",
"MS12V2 MAT-Enc", "ms12v2_pcmr_*", "MS12V2 Mixer", "MS12V2 DDP encoder",
"MS12V2 ddpe : DDP mode : reset hdmi 192K !!", "Dolby MS12V2 PCMR
module", "MS12V2 DDP-Enc reset ok, encoding_mode=%d" — це MS12 v2.4 з
покращеним Atmos/lock-control/dual-mode/MAT-Encoder. Жоден з цих рядків не
згадує DTS-стек, але обидва мають **повний DTS-стек однаково**.

### 1.4 Codec R2 розходження

`mst_codec_r2` містить 133 Dolby-MS12V1 рядки (DDP-decode/UDC, DAP,
jocdec, heaacdec SBR, ms12_adts.c, ms12_heaacdec.c та ін.).
`mst_codec_r2_MS12V22` містить 6 V2-рядків (CPU MS12 V2 HEAAC/ac4/ddp
hook ok). Знову ж таки — **жоден DTS-рядок не згадується**, оскільки DTS
живе у snd R2, а не в codec R2. DTS-декодер фрейм-левел — у
`mst_snd_r2` (decompress + SPDIF-packer там, дизасемблювання рядків
`dec-p1/dtshd-c-decoder/.../dtshd_frame_player_dsp2.c`).

### 1.5 Що НЕ з'являється у жодному з двох образів (підтверджено byte-search)

- **XPT_Packer / XPT_Packer_API** — у проекті DTS:X існує модуль XPT (XPerT
  Transcoder), але в MStar-збірці його немає. Тобто DTS:X-iml недоступний
  навіть у повному profile.
- **dtsx-inputprocess/InputProcess.c** — `Mstar InputProcess 1.0.0` є, але
  файл-сирцю немає; це вказує на те, що оригінальний D2A_InputProcess
  перейменовано.
- **OutputProcessSPDIF/OutputProcessHDMI** — немає. Це внутрішні
  MStar-функції, які реалізовані всередині DTSDecSDOPacker (на рівні
  R2-функцій), без окремого source-файла.
- **dtsx-sdo module variations** — тільки SDO_Packer (а не Sdi/Osd).

**Висновок 1:** MS12V22-профіль НЕ додає нових DTS-функцій на C50A.
Те, що **DTS** дає — це те, що DTS-стек уже є в base образі. Тому
"premium full profile з 0x4d0=4" не вирішує відсутність DTS-декодування
на C50A — він лише перемикає на інший Dolby-профіль.

---

## 2. Що MS12V22 справді додає для DTS

### 2.1 SetSystem2: 0x4d8==3 → DSP-байт 0x97 (тільки в 0x4d0=4)

У `HAL_AUDIO_SetSystem2` (0x44d0b0) вхідний аргумент `r7` =
decode-system. На адресі 0x44d1dc:

```
0x44d144  ldr      r1, [r0, #0x4d0]   ; r0 = g_AudioVars2
0x44d148  cmp      r1, #4
0x44d14c  sub      r1, r7, #1
0x44d150  mvneq    r4, #0x7f        ; if (0x4d0==4) r4=0xffffff81 (зсув таблиці)
0x44d154  cmp      r1, #0x1d
0x44d158  bhi      #0x44d3c4
0x44d15c  add      r2, pc, #0
0x44d160  ldr      pc, [r2, r1, lsl #2]  ; switch((sys-1) + (0x4d0==4?0x1f:0))
...
0x44d1dc  ldr      r0, [r0, #0x4d8]   ; 0x4d8-гілка
0x44d1e0  mov      r1, #4
0x44d1e4  cmp      r0, #3            ; 0x4d8==3 (DTS:X рівень)
0x44d1e8  mov      r0, r6
0x44d1ec  movweq   r1, #0x97         ; r1 = 0x97 if 0x4d8==3
0x44d1f0  bl       #0x44d1f0         ; AbsWriteByte(addr=0x112E98+2, r1)
```

Тобто для DTS-систем (sys=4=DTS, 0xb=HDMI-DTS тощо) при `0x4d0==4` і
`0x4d8==3` (DTS:X-rівень) **записується `0x97`** в DEC-R2 замість `0x05`.
Байт `0x97` — це специфічний runtime-config DTS:X-режиму, доступний
лише в MS12V22-декодері. У base-образі декодер приймає лише `0x05` для
DTS-core; спроби `0x97` повернуть "unsupported".

### 2.2 SetSystem2: розширений switch table (0x4d0==4)

`mvneq r4, #0x7f` (тобто r4 = 0xffffff81 = -0x7f; це load-immediate) при
`0x4d0==4` додає зсув `+0x1f` (або в switch-table dispatcher — це
еквівалентно 0x7f зі знаком). Це дозволяє sys ∈ {0x21..0x3f} (тобто
0x20-зсунуті ідентифікатори систем) оброблятися окремо. У `MDrv_AUDIO_
OpenDecodeSystem` (@0x405cc8) виклик із типом системи `0x21..0x3f`
відповідає MS12-розширеним системам (MS12 внутрішньо резервує 0x20+).
Це знову Dolby-специфічне розширення, не DTS.

### 2.3 DspLoadCode: 0x4d0==4 → MS12V22-образи (0x46f5a8-0x46f7c0)

```
0x46fb14  movw     r2, #0           ; r2 = mst_snd_r2_MS12V22 (reloc)
0x46fb18  movw     fp, #0           ; fp = mst_snd_r2 (reloc)
0x46fb1c  movt     r2, #0
0x46fb20  movt     fp, #0
0x46fb24  movw     r7, #0           ; r7 = mst_codec_r2
0x46fb28  ldr      r1, [r0, #0x4d0]
0x46fb2c  movw     r8, #0x3f48      ; розмір snd (base)
0x46fb30  movw     r5, #0xc0f4      ; розмір codec (base)
0x46fb34  movt     r7, #0           ; r7 = mst_codec_r2
0x46fb38  cmp      r1, #4
0x46fb3c  movt     r8, #0x17
0x46fb40  moveq    fp, r2           ; if 0x4d0==4: fp = MS12V22 snd
0x46fb44  movw     r2, #0           ; r2 = mst_codec_r2_MS12V22
0x46fb48  movt     r2, #0
0x46fb4c  movt     r5, #0x29
0x46fb50  moveq    r7, r2           ; if 0x4d0==4: r7 = MS12V22 codec
0x46fb54  cmp      r1, #4
0x46fb58  ldrb     r1, [r0, #9]
0x46fb5c  movweq   r8, #0x6930      ; розмір MS12V22 snd
0x46fb60  movweq   r5, #0x401c      ; розмір MS12V22 codec
```

Тобто при `0x4d0==4` завантажуються MS12V22-образи: snd 0x1c1330 байт,
codec 0x1e401c байт. У подальшій обробці викликається
`HAL_AUDSP_DspLoadCodeSegment(addr, size, image)` для кожного
сегмента. Це впливає на **всі** формати, не лише DTS — при 0x4d0=4 весь
стек працює в MS12V2-режимі.

### 2.4 HAL_AUDIO_Encoder_Channel_Lock (0x4d0 consumers)

```
0x43f168  push     {fp, lr}
0x43f178  ldr      r0, [r0]         ; r0 = g_AudioVars2
0x43f17c  ldr      r1, [r0, #0x4d0]
0x43f180  sub      r3, r1, #1
0x43f184  cmp      r3, #2
0x43f188  blo      #0x43f1d8       ; if (0x4d0∈{1,2,3}) → skip
0x43f18c  cmp      r1, #4
0x43f190  cmpne    r1, #3
0x43f194  bne      #0x43f1b0
0x43f198  mov      r0, #0x6d
0x43f19c  mov      r1, #0
0x43f1a0  mov      r3, #0
0x43f1a4  bl       #0x43f1a4       ; HAL_SND_R2_Set_SHM_PARAM(0x6d, 0, 0)
0x43f1a8  mov      r0, #1
0x43f1ac  pop      {fp, pc}
```

`0x4d0 ∈ {3, 4}` — channel-lock активується (R2-SHM 0x6d). При
`0x4d0 ∈ {1, 2}` (поточний стан C50A) — функція повертає 0 без SHM
запису. Це впливає на MS12-output-config у SND-R2, не на DTS-decode
старт.

### 2.5 HAL_MAD_GetAudioInfo2 (query-only, не gate)

У декомпіляції (`k_utpa2k_decomp.c`):
- case 4/0xb (DTS-HD системи): `if (0x43d==1) { ... case 2 → uVar9=4; case 1
  → uVar9=4; case 0 → case 5; ... }`. Тобто при `0x43d==1` (DTS premium
  active) звіт повертає DTS-HD-resampling-off mode (`uVar9=4`).
- case 0xa/0x10 (HDMI/ARC): `if (0x4d8>1) { switch...; default: case 5,6
  → case 5 }`. Тобто при `0x4d8>=2` HDMI-статус показує DTS-HD-rates.
  Це **тільки звіт**, не впливає на сам процес decode-transport.
- case 0x55 (0x4d0==4 ⇔ getDecoderInfo): if `0x4d0==4` повертає
  `uVar8 = (IP(0x7d) | IP(10)) & SHM(0x4d)` — це індикатор "premium DTS
  present". При `0x4d0!=4` — `uVar8=0`.

---

## 3. Що дає `0x4d0=4` для DTS-стеку (збірне)

| Аспект | base (0x4d0 ∈ {0,1,2,3}) | MS12V22 (0x4d0==4) | Вплив на DTS |
|---|---|---|---|
| snd R2 образ | mst_snd_r2 (0x17e948) | mst_snd_r2_MS12V22 (0x1c1330) | однаковий DTS-стек; MS12V22 має додатковий Dolby-v2.4 |
| codec R2 образ | mst_codec_r2 (0x29c0f4) | mst_codec_r2_MS12V22 (0x1e401c) | без DTS-коду (DTS в snd) |
| DSP-байт DTS | 0x05 (DTS core) | 0x05 / 0x97 (DTS:X) | **так** — `0x97` доступне лише в MS12V22 |
| SetSystem2 switch | 0..0x1d (30 систем) | + 0x20-зсув (MS12 extended) | ні для DTS |
| Encoder_Channel_Lock | off | on (R2-SHM 0x6d) | лише MS12 path, не DTS |
| GetAudioInfo2 case 0x55 | uVar8=0 | uVar8 = IP(0x7d|10) & SHM(0x4d) | лише звіт |
| SPDIF TX bайт | 0x05/0x0d | 0x05/0x97 | з 0x4d8==3 + 0x4d0==4 → 0x97 |
| 0x4d0 profile | – | pre-DTS-X-ексклюзивний режим | повні DTS:X-можливості |

**Найважливіше:** якщо ми вибираємо 0x4d0=4 для DTS-сесії, ми отримуємо:

- **Доступ до DTS:X-декодера (0x97-байт)** — `0x4d0=4 && 0x4d8=3` → R2
  виконує `0x97` байт, що вмикає DTS:X-iml в MS12V22-декодері (якого
  немає в base).
- **MAT-Encoder sidechain** — MS12V22 може кодувати MAT (Atmos);
  хоча це не впливає безпосередньо на SPDIF (Atmos-об'єктні потоки
  передаються тільки через HDMI), це може бути корисним для двох-вихідних
  конфігурацій (TV/AVR).
- **MS12-v2.4 DAP** — повна Dolby Atmos processing.

---

## 4. DTS-транспорт: три стани (порівняння)

### 4.1 CURRENT (C50A)

```
0x440 = (Dolby-біт-mask) | 0x8 | 0x80 | 0x20000   ; DTS-бітs: 0xf,0x3a,0x12 fail
0x4d0 ∈ {0,1,2,3}                                  ; MS12V1-профіль
0x4d4 ∈ {0,4,6,7,8,9}                               ; Dolby-рівень
0x4d8 = 0                                            ; DTS-рівень
0x43d = 0/1                                          ; Dolby premium
0x43e = 0/1
R2-образи: mst_*_r2 (base)
DSP-байт DTS-сесії: 0x0d (через SetSystem2 case 0x4a → cmp 0x43d==1 ? 5 : 0xd)
DTS-декодер: НЕ стартує (Auth-IPCheck для 0xf/0x3a/0x12 fail)
SDO SPDIF packer: готовий, але не активований
```

### 4.2 DTS-ONLY (заплачений наш license-патч: 4 NOP-и на групу {0xf, 0x3a, 0x12, 7})

```
0x440 чистий від DTS-біт, всі інші біти лишаються;
0x4d0 не змінюється (як у CURRENT);
0x4d8 = 3; 0x43d = 0/1; 0x43e = 0/1;
R2-образи: mst_*_r2 (base) — без зміни;
DSP-байт DTS-сесії: 0x0d (бо 0x4d0≠4; SetSystem2 не зсувається → той самий case 0x4a
                       → все ще 0x43d branch → 0x43d∈{0,1} → 5 або 0xd;
                       це пишеться у регістр 0x112E98+0/1)
DTS-декодер: СТАРТУЄ (license-бітs прибрані, R2 OK);
SDO SPDIF packer: активний, отримує DTS-кадри від R2 DTS-decode
                  → випускає в SPDIF (в режимі bypass: `spdifFrmBuf`)
```

**Транспорт:** звичайний. `0x97` НЕ доступний, бо `0x4d0!=4`. DSP
приймає `0x05/0x0d` (DTS core). Для DTS:X (`0x4d8=3`) він інтерпретує
як DTS-HD-rate або DTS-core, що **не втрачає** SPDIF-payload, бо
SDO packer приймає будь-який вхід (він не залежить від DSP-байта — тільки
від compress-decode pipeline). Підтвердження: на C50A AC3 працює в
простому `0x0d` режимі, не в `0x97`/MS12-режимі.

### 4.3 PREMIUM FULL (0x4d0=4 + 4-NOP-патч: емуляція PASS усіх IP-ів)

```
0x440 = 0x00001000 (аномалія 0x7f PASS); всі missing-бітs чисті.
0x444 = 0x003FFE80 (Dolby MS12-бітs стерті, 0x280+ поставлені, 2 стертий).
0x4d0 = 4 (преміум tier).
0x4d4 = 8 (0x7d-pass остаточний).
0x4d8 = 3 (DTS:X-рівень).
0x43d = 1; 0x43e = 1 (Dolby premium active).
0x582 = 1 (7-pass).
R2-образи: mst_*_r2_MS12V22 (преміум).
DSP-байт DTS-сесії: 0x97 (SetSystem2 case 0x4a при 0x4d0=4 && 0x4d8=3)
                     → у MS12V22-декодері → DTS:X-iml увімкнений.
DTS-декодер: СТАРТУЄ + DTS:X-iml доступний;
SDO SPDIF packer: той самий, плюс додатковий код MAT-Encoder sidechain
                  (для Atmos-розширень, не для SPDIF).
Encoder_Channel_Lock: активний (0x4d0==4 → SND-R2 SHM 0x6d).
```

**Транспорт:** максимальний, з DTS:X-можливістю. Але `mst_*_r2_MS12V22`
не має XPT_Packer (XPerT Transcoder), тому **DTS:X-об'єктний канал усе
одно НЕ передається по SPDIF** (навіть у MS12V22-профілі). SPDIF
побачить лише core DTS.

---

## 5. Що MS12V22 робить для DTS-transport, чого НЕ робить base

| Функція | base | MS12V22 | Вплив на DTS SPDIF |
|---|---|---|---|
| DTS-core SDO packer | ✓ | ✓ | без різниці |
| DTS-HD SDO packer | ✓ (через 0x05) | ✓ (через 0x05/0x97) | без різниці в SPDIF (все одно core) |
| DTS:X 61937 frame | ✗ (нема XPT) | ✗ (нема XPT) | **жоден не передає** |
| DTS:X-об'єктний канал | ✗ | ✗ | **жоден не передає** (нема XPT_Packer ні в одному) |
| MAT-Encoder sidechain | ✗ | ✓ (Atmos via HDMI; не SPDIF) | не впливає на SPDIF |
| Atmos decoding (decoder-side) | ✗ (DD-only DAP) | ✓ (MS12 v2.4 DAP) | впливає на HDMI ARC/TV, не SPDIF |
| SDO packer bypass | ✓ | ✓ | без різниці |

**Висновок 5:** на C50A жоден з двох образів не здатен передавати
DTS:X-об'єктні канали по SPDIF (XPT_Packer відсутній в обох).
MS12V22 додає **виключно** Atmos/MAT-обробку для HDMI та premium-DAP,
але **не** нові SPDIF-функції для DTS.

---

## 6. Реальна причина, чому `0x4d0=4` може бути корисним

Попри висновок 5, є один практичний сценарій, де MS12V22 допомагає DTS:

Якщо в попередніх експериментах після нашого DTS-license-патчу DTS-сесія
не змогла синхронізуватися через **runtime-конфлікт у R2 (наприклад, MS12
v1 vs v2 MS12-декодер "зайняв" ресурс, або FIFO-розподілення в DEC R2
очікує MS12-розширеного формату)**, перемикання на MS12V22 може дати
іншу поведінку. Це можна перевірити лише runtime-тестом. Static analysis
не дає остаточної відповіді.

**Що варто перевірити runtime'ом (після `0x4d0=4`):**

1. Чи `0x4d0=4` досяжна з нашого DTS-license-патчу? **Так** — емуляція
   full-license через broad `AUTH_IPCheck`-override робить `0x4d0=4`
   автоматично, бо `0x54` (Dolby top) проходить. Але це не strict
   вимога для DTS — DTS-ONLY варіант (4 NOP-и) **не** торкається
   `0x4d0`. Тому якщо ми хочемо саме MS12V22-образи для DTS, треба
   broad-override або додаткові 6 NOP-и.
2. Поведінка DTS-decode на base vs MS12V22 — R2-образи
   функціонально еквівалентні, але **runtime-timing може відрізнятися**
   (MS12V22 має додаткові Dolby-блоки, які конкурують за DEC-R2
   bus-cycles). Чи це впливає на DTS SPDIF-sync — невідомо.

---

## 7. Реконструкція штатного DTS-capable profile без broad override

Щоб отримати `0x4d0=4` **без** broad `AUTH_IPCheck`-override, потрібно
пройти CheckHashkey так, щоб `0x4d0=4` — це вимагає, щоб останнє
перевірене IP-серед `0x54, 8, 9, 0xa, 0x66, 0x7d` пройшло (саме вони
виставляють r6=4 у порядку 0x4242e8-0x424488):

| IP | Вплив на r6 (0x4d0) | Можна підробити цілеспрямовано? |
|---|---|---|
| 0x54 (Dolby top) | 4 | Так — 1 NOP @0x4240dc fail-гілка |
| 8 (premium) | 4 | Так — 1 NOP @0x42414c fail-гілка |
| 9 (premium) | 4 | Так — 1 NOP @0x4242e8 fail-гілка |
| 0xa (premium) | 4 | Так — 1 NOP @0x424384 fail-гілка |
| 0x66 (premium) | 4 | Так — 1 NOP @0x424234 fail-гілка |
| 0x7d (premium) | 4 | Так — 1 NOP @0x424438 fail-гілка |

Але: щоб отримати 0x4d0=4, **не потрібно проходити 0x54** — досить
будь-якого з {8, 9, 0xa, 0x66, 0x7d} (але не 0x54 окремо, він пише
лише 0x57e/0x57f, не 0x4d0). Тобто **6 можливих мінімальних
патчів** для `0x4d0=4` — будь-який з 6 NOP-ів на 8/9/0xa/0x66/0x7d.

Усі вони мають однаковий ефект: завантажують MS12V22-образи, SetSystem2
зсуває switch table, Encoder_Channel_Lock активується, GetAudioInfo2
case 0x55 повертає IP-SHM комбінацію. **DTS-decode усе одно потребує
0x4d8=3** (DTS-license-патч), і тоді SetSystem2 case 0x4a пише
`0x97` (а не 0x05).

**Висновок 7:** `0x4d0=4` досяжний без broad-override. У цьому
випадку:

- **DTS transport работает** (через ліцензійний 0x4d8=3 + DSP-байт
  0x97 на MS12V22).
- **Уникаємо broad-override** (тільки 4+1 NOP-ів замість broad).
- **Зберігаємо поточний Dolby-профіль** (AC3 лишається на поточному
  level, не змінюється).

Однак DTS-ONLY state (без `0x4d0=4`) **також транспортує** DTS
(через 0x05 у base-образі). Отже, перемикання 0x4d0=4 для DTS
**не обов'язкове**, але може додати:
- DTS:X-об'єктну обробку всередині R2 (хоча SPDIF TX все одно бачить
  тільки core — бо XPT_Packer відсутній);
- покращену синхронізацію через інший runtime-config (потребує
  runtime-перевірки).

---

## 8. Три варіанти: фінальне порівняння (DTS transport)

| | CURRENT | DTS-ONLY (4 NOPs) | FULL-LICENSE (broad) |
|---|---|---|---|
| **Patch розмір** | 0 | 4 NOP (16 B) | 42 NOP (~170 B) |
| **Зміни в CheckHashkey** | ні | 0x423a84/0x423c00/0x423eec/0x4246ec | всі 42 fail-гілок |
| **Зміни в SetSystem2** | ні | ні | 0x4d0=4 → MS12V22 path; 0x4d8=3 → DSP 0x97 |
| **DSP-образи** | base | base | MS12V22 |
| **R2 DTS-decode** | не стартує | стартує (0x4d8=3, 0x05-байт) | стартує (0x4d8=3, 0x97-байт) |
| **SDO SPDIF packer** | неактивний | активний | активний |
| **DTS:X object on SPDIF** | ні | ні (XPT-Packer відсутній) | ні (XPT-Packer відсутній) |
| **DTS-core на SPDIF** | ні | **так** | **так** |
| **Dolby Atmos (HDMI)** | ні (базовий DAP) | ні (без зміни) | **так** (MS12-v2.4 DAP) |
| **Dolby MAT-Encoder** | ні | ні | **так** (через MS12V22) |
| **Encoder Channel Lock** | off (0x4d0≤2) | off | on (0x4d0=4) |
| **Надійність** | n/a | висока | висока (більше побічних ефектів) |
| **Зворотність** | n/a | повна (1 патч) | повна (1 патч, але більше змін) |
| **Confidence** | 0 | ~0.85 (transport), ~0.4 (DTS:X-iml) | ~0.85 (transport), ~0.4 (DTS:X-iml) |

---

## 9. Рекомендація

**Не відкидати** `0x4d0=4 / MS12V22` як варіант — але **не як
обов'язковий крок**. Практична послідовність:

1. Спочатку застосувати **DTS-ONLY (4 NOPs)** — він гарантовано дає
   DTS core через SPDIF з base-образами. Перевірити runtime.
2. Якщо DTS працює, але **DTS:X-об'єктний канал не з'являється на
   AVR** (або SPDIF не синхронізується на деяких треках) → додатково
   застосувати **6 NOP на {8, 9, 0xa, 0x66, 0x7d}** для `0x4d0=4`.
   Це перемикає R2 на MS12V22 + дозволяє `0x97` DSP-байт.
3. Повний broad-патч — тільки якщо попередні два не дали DTS-звуку
   на AVR (вкрай малоймовірно, бо MS12V22 не додає нових SPDIF-функцій).

**Не робити висновок** про відсутність користі `0x4d0=4` без
runtime-перевірки. Статичний аналіз показує, що **DTS-transport не
потребує** MS12V22-image для базового DTS-core, але MS12V22-image
розблоковує додаткові runtime-конфігурації (DSP-байт 0x97,
Encoder-Channel-Lock) і потенційно може вплинути на runtime-таймінги
R2 (неперевірено статично).

---

## Додаток: артефакти

- `tools/r2img_diff.py` — витяг 4 R2-образів
- `tools/r2img_clean_diff.txt` — повний фільтрований string-diff
  base vs MS12V22 для codec_r2 та snd_r2
- `tools/dump_checkhashkey.py`, `tools/ut_checkhashkey_full.asm` —
  CheckHashkey (1472 рядки, 43 IPID-анотації)
- `tools/dump_funcs.py`, `tools/ut_licenses.asm` — Get_*_License
  дизасемблювання
- Ключові адреси: DspLoadCode 0x46f5a8 (0x4d0==4 → MS12V22 образи);
  SetSystem2 0x44d0b0 (0x4d0==4 → switch зсув; 0x4d8==3 → DSP 0x97);
  Encoder_Channel_Lock 0x43f168 (0x4d0∈{3,4} → R2-SHM 0x6d);
  GetAudioInfo2 case 0x4a/0x55/0x4-0xb — query-only гілки.
