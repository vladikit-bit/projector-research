# MT5889/C50A: повна реконструкція license-стану аудіо і пошук "повного профілю"

Дата: 2026-08-30 (гілка research-only, пристрій не змінювався).
Метод: повне линейне розбирання `MDrv_AUDIO_CheckHashkey` (utpa2k.ko @0x423494,
0x16fc байт, ARM, 43 виклики `MDrv_AUTH_IPCheck`), всіх `MDrv_AUDIO_Get_*_License`,
`MDrv_AUDIO_Get_License`, `MDrv_AUDIO_Get_Decoder_Support`,
`HAL_AUDIO_DDELoadCode/DDPELoadCode`, `HAL_AUDSP_DspLoadCode`,
`MDrv_AUTH_IPCheck` @0x1390c, `utopia_proc_ioctl` @0x13a90, push-функції масок
@0x424bd8. Адреси — реальні з .symtab (Ghidra-адреси старих декомпіляцій зсунуті
на +0x1D624).

---

## 0. Головні висновки (коротко)

1. **Готового конфігураційного "повного профілю" у firmware НЕМАЄ.** Стан
   ліцензій — це **дані** (140-байтна бітова таблиця `gIpAuthVars`), яку
   userspace/secure AUTH-демон передає в ядро через `ioctl(0xC08C5506)`; код
   має рівно один шлях її розбору (CheckHashkey). Альтернативних таблиць,
   SKU-перемикачів або регіональних профілів у бінарнику не існує.
2. **Справжній "full-license" стан реконструйовано повністю** (таблиці в §5–§7):
   це не довільна комбінація, а детермінована функція бітів AUTH-таблиці.
3. **КРИТИЧНЕ перевідкриття щодо попереднього DTS-патчу:** ID `0xb`/`0xc` — це
   **Dolby AC3/DD+**, а НЕ DTS (доведено через `GetLicenseAc3`, який перевіряє
   0xb першим). Справжня DTS-група — `{0xf, 0x3a, 0x12, 7}` (та сама, що в
   `MDrv_AUDIO_Get_DTS_License`), і вона **ортогональна** до Dolby-профілю:
   не змінює 0x4d0, не перемикає DSP-образи, а лише виставляє рівень `0x4d8`
   (1=DTS core, 2=DTS-HD/LBR, 3=DTS:X) і прибирає "missing"-біти 0x8/0x80/
   0x20000 у масці 0x440, яка йде в R2 через SE-IDMA.
4. Наслідок: **мінімальний вірний DTS-патч — 4 NOP-и (по одному слову) на
   fail-гілках IPCheck(0xf/0x3a/0x12/7)** — точна емуляція справжнього
   "DTS-ліцензованого" стану без жодного побічного ефекту на Dolby-профіль.
   Попередній кандидат (примусові 0x43d/0x43e + NOP 0xb/0xc) частково
   зайвий/неточний: 0x43d/0x43e — індикатори **Dolby-premium**, а 0xb/0xc на
   C50A і так проходять (AC3 працює).
5. `0x112cf0[7]` — справжній hardware/secure strap-біт DTS: читається тільки
   query-API (`Get_*_License`) і жодним записом у хост-коді не керується;
   у decode-start ланцюжку не бере участі (перевірено сканом movw/movt по
   всьому .text — 0 записів).

---

## 1. Джерело стану: gIpAuthVars (першоджерело всіх ліцензій)

- `gIpAuthVars` (OBJECT @0x2b210, вказівник) заповнюється в
  `utopia_proc_ioctl` (0x13a90): під-команда **0xC08C5506** копіює з
  userspace **0x8c = 140 байт** (arm_copy_from_user → kmem_cache_alloc 0x8c →
  memcpy) у виділений буфер; вказівник зберігається в gIpAuthVars.
- `MDrv_AUTH_IPCheck(id)` @0x1390c: `byte9 = tbl[9]` (розмір бітового поля);
  перевірка діапазону; біт = `(tbl[10 + byte9 - 1 - id/8] >> (id&7)) & 1`.
  Тобто **ліцензія = один біт у таблиці, яку кладе AUTH-демон** (він вже
  перевірив eFuse/hashkey у secure-середовищі). У ядрі криптографії нема.
- Наслідок: "повний профіль" = таблиця ліцензованого пристрою. Її вміст
  повністю визначений полями нижче.

## 2. Матриця audio IP (ID → семантика)

Позначення: **CONFIRMED** — пряний зв'язок з іменованим кодеком;
**STRONGLY INFERRED** — кілька незалежних свідчень; **UNKNOWN** — вплив
зафіксовано, семантика ні.

| ID | Семантика | Conf. | Споживачі / доказ |
|----|-----------|-------|-------------------|
| 0xb | **Dolby AC3 (DD)** | CONFIRMED | `GetLicenseAc3` перевіряє 0xb ПЕРШИМ; C50A декодує AC3 |
| 0xc | **Dolby DD+ (E-AC3)** | CONFIRMED | `GetLicenseAc3` fallback (r4=1 через 0xc); біт0 0x440 стирається pass'ом 0xc |
| 0x50 | Dolby MS12-сімейство, primary | STRONGLY INFERRED | пара 0x50/0x73; Get_AC3/AAC OR-дерева; tier 2 |
| 0x73 | Dolby MS12-сімейство, fallback | STRONGLY INFERRED | пара; при 0x73-pass рівні нижчі (sl=0) |
| 0x52 / 0x75 | Dolby suite (tier 3b, 0x4d4=6) | STRONGLY INFERRED | пара N/N+0x23 |
| 0x51 / 0x74 | Dolby suite (tier 3a, 0x4d4=4) | STRONGLY INFERRED | пара |
| 0x54 | Dolby топ-suite (tier 4, 0x4d4=7) | STRONGLY INFERRED | Get_MAT fallback; 0x57e\|=1 |
| 8 | Dolby premium family (tier 4, 0x4d4=7) | STRONGLY INFERRED | член OR-дерев AC3/AAC/AC4/MAT |
| 9 | Dolby premium family (tier 4, 0x4d4=9) | STRONGLY INFERRED | член OR-дерев; sl=1 |
| 0xa | Dolby premium family (tier 4, 0x4d4=8) | STRONGLY INFERRED | член OR-дерев |
| 0x66 | Dolby premium family (tier 4, 0x4d4=9) | STRONGLY INFERRED | член OR-дерев |
| 0x7d | Dolby premium family (tier 4, 0x4d4=8) | STRONGLY INFERRED | останній у деревах; чистить 0x440 bit4 |
| 0xe | агрегат MS12-сімейства (8 перевірок) | UNKNOWN | fail-ланцюжок ставить 0x440\|=0x10 |
| **0xf** | **DTS core** | **CONFIRMED** | `GetLicenseDts` член №1; **0x4d8=1** |
| **0x3a** | **DTS family (Neural:X/Headphone:X?)** | STRONGLY INFERRED | `GetLicenseDts` член №2 |
| **0x12** | **DTS-HD / LBR** | STRONGLY INFERRED | `GetLicenseDts` член №3; **0x4d8=2** |
| **7** | **DTS:X (top)** | STRONGLY INFERRED | `GetLicenseDts` член №4; **0x4d8=3**, 0x582=1 |
| 0x46 | AAC main | CONFIRMED | `GetLicenseAac` перший |
| 0x79 | AC4 | CONFIRMED | `GetLicenseAc4` перший |
| 0x1e | WMA | CONFIRMED | `GetLicenseWma` + strap bit11 |
| 0x41 | DRA | CONFIRMED | `GetLicenseDra` + strap bit10 |
| 0x53 | ? (0x440 bit18) | UNKNOWN | 0x12-pass НЕ чистить цей біт (чистить 0x400=0x49-біт) |
| 0xd | ? (0x440 bit2) | UNKNOWN | — |
| 0x49 | ? (0x440 bit10) | UNKNOWN | 0x12-pass стирає цей missing-біт (зв'язок з 0x49?) |
| 0x3, 0x37, 0x45, 0x1c, 0x38, 0x42 | ? (біти 0x2000/0x4000/0x8000/0x10000/0x400000/0x800000) | UNKNOWN | single-check патерн |
| 0x7f | ? — **pass ставить** 0x440\|=0x1000 | UNKNOWN (аномалія: bit на PASS) | — |
| 0x7e/0x7c/6/0x40/5 | family-прапори в 0x444 (біти 2/3/4/6/5) | UNKNOWN | pass → bic відповідного біта |

 Strap-регістр 0x112cf0 (біти, з Get_*_License): bit2/3 → AC3-маска 0xf,
 bit7 → **DTS**, bit10 → DRA, bit11 → WMA, bit12 → AAC-family 0x10 / MAT 0x1000.
 Читається ТІЛЬКИ query-API; записів не існує (скан movw/movt по всьому .text
 та модулях: 0 результатів для запису).

## 3. Машина стану CheckHashkey (точна, з реальних адрес)

Ініціалізація (0x4234cc-0x4234e0): `0x57e=0; 0x440=0; 0x444=0x00ffffff;
0x582=0`. Далі 43 перевірки в фіксованому порядку; кожна fail-гілка ставить
свій "missing"-біт у **0x440**, pass-гілка преміум-сімейств — рівні.

**0x440 (maskA) — "missing-біти"** (ставляться на FAIL):
`0x1=0xb, 0x2=0xc, 0x4=0xd, 0x8=0xf(DTS), 0x10=0xe-ланцюг|8|9|0xa,
0x20=0x1e(WMA), 0x40=0x41(DRA), 0x80=0x3a(DTS), 0x100=0x46(AAC),
0x200=0x50/0x73(Dolby), 0x400=0x49 (стирається PASS'ом 0x12!),
0x2000=3, 0x4000=0x37, 0x8000=0x45, 0x10000=0x1c, 0x20000=0x12(DTS),
0x40000=0x53, 0x80000=0x52/0x75, 0x200000=0x51/0x74, 0x400000=0x38,
0x800000=0x42; 0x1000 ставиться на PASS 0x7f (аномалія)`.
Виключення: PASS 0xc стирає bit0 (DD+ включає AC3); PASS 0x12 стирає bit10.

**0x444 (maskB) — "доступні-біти"** (0x00ffffff; PASS сімейства СТИРАЄ біт):
0x79→bit0; 0x54/8→bit7; 0x66/9→bit9; 0xa/0x7d→(r2|0x280)&~2 (ставить 7,9,
стирає 1); 0x7e→bit2; 0x7c→bit3; 6→bit4; 0x40→bit6; 7→bit8; 5→bit5.

**Рівневі регістри (перезаписуються по мірі проходження):**
- **r6 → 0x4d0 (Dolby-профіль/DSP-образ):** 2 = 0x50|0x73; 3 = 0x52|0x75|0x51|0x74;
  4 = {0x54, 8, 9, 0xa, 0x66, 0x7d}. **Жоден DTS-id (0xf/0x3a/0x12/7) не пише
  r6.** Пізніший pass перезаписує раніший → пріоритет: DTS-незалежно,
  останній premium-сімейство у порядку коду виграє.
- **r5 → 0x4d4 (під-рівень преміум):** 4=0x51/0x74; 6=0x52/0x75; 7=0x54|8;
  9=0x66|9; 8=0xa|0x7d.
- **r8 → 0x4d8 (РІВЕНЬ DTS):** 0 = нема; **1 = 0xf (DTS core); 2 = 0x12;
  3 = 7 (DTS:X)**. Пізніший pass перезаписує.
- **r7 → 0x43e ("premium active"):** 1 на будь-якому premium-pass
  (0x50, 0x52/0x75, 0x51/0x74, 0x54, 8, 0x66, 9, 0xa, 0x7d); 0 на
  fallback-гілках 0x73/0x75/0x74 та на pass 8/0x66/0xa.
- **sl → 0x43d ("premium top active"):** 1 на 0x50, 0x52/0x75, 0x51/0x74,
  0x54, 9, 0x7d; 0 на 0x73/0x75/0x74-fallback, 8, 0x66, 0xa.
- Допоміжні: 0x580\|=1 (0x50/0x73), 0x581\|=1 (0xc/0x50/0x73),
  0x57f\|=1 (0x52/0x75/0x51/0x74/0x54/8/0x66/9/0xa/0x7d),
  0x57e\|=1 (0x54/8/0x66/9/0xa/0x7d), 0x582=1 (7).
- Фінал (0x424868-0x424880): `0x4d0=r6; 0x4d4=r5; 0x4d8=r8; 0x43e=r7; 0x43d=sl`;
  `HAL_AUDIO_CheckHashkeyDone(maskA)` — **заглка (mov r0,#0; bx lr)** — panic-
  шлях неактивний; прапор 0x503=1 кешує результат (перерахунок лише після
  DspReboot/перезавантаження).

**Вивід про семантику 0x43d/0x43e:** це **не** DTS-біти. Це "Dolby premium
(MS12)-рівень активний". Той факт, що `HAL_MAD_GetAudioInfo2` гілка
decode-system {4, 0xb} читає 0x43d, а {0xa, 0x10} — 0x4d8, означає лише, що
статус DTS-систем звітується "повним" лише за наявності преміум-ланцюга —
це статусна логіка, а не gate старту декодера (у ланцюгу старту —
SetDecodeSystem/SetSystem2/SetDecodeCmd — 0x4d8 не читається ніде).

## 4. Вибір DSP-образу (дерево)

`HAL_AUDSP_DspLoadCode` (0x46f5a8): `if (g_AudioVars2[0x4d0]==4) →
(mst_codec_r2_MS12V22, mst_snd_r2_MS12V22) else → (mst_codec_r2, mst_snd_r2)`.
- 0x4d0 ∈ {0,2,3} → **базові образи** (C50A зараз тут).
- 0x4d0 == 4 → **MS12V22-пара** (преміум: MS12 v2.2 обробка).
- `HAL_AUDIO_DDELoadCode`/`DDPELoadCode` (0x443e98/0x443f54) — вибір
  Dolby-декодерного блоку за 0x4d0 (cmp #4 / cmphs #4) — теж лише Dolby.
- DTS SDK (декодер+SDO packer+Xcoder) присутній у **базовому** mst_snd_r2 —
  тобто DTS-машинерія не залежить від варіанта образу.

**Важливий висновок:** перемикання 0x4d0 → 4 змінює R2-образи (побічний
ефект: MS12-обробка, інші бінарники). Справжній DTS-ліцензований пристрій
з базовим Dolby (0x4d0=2) продовжує працювати на базових образах — DTS
не вимагає tier 4.

## 5. CURRENT — C50A (реконструкція)

Відомо емпірично: AC3 локально декодується (0xb PASS), AAC працює (0x46
PASS), DTS не працює (0xf/0x3a/0x12 FAIL), strap bit7 = 0.

```
0x440: | 0x8 | 0x80 | 0x20000                      (DTS missing)
     + | 0x200000? | 0x80000? | 0x40000? | 0x400?  (залежно від Dolby-рівня)
     + | 0x10? (0xe-ланцюг, якщо нічого з 0x50/0x51/0x52/0x54/9/0x7d не пройшло)
     + | інші single-id біти (0xd, 3, 0x37, 0x45, 0x1c, 0x38, 0x42, 0x49, 0x7f-нема)
0x444: 0x00ffffff & ~біти сімейств, що пройшли
0x4d0: 0 | 2 | 3   (залежно від того, який Dolby-рівень ліцензовано)
0x4d4: відповідно 0/4/6/7/9/8
0x4d8: 0            (DTS нема)          0x43d/0x43e: 0|1 (Dolby-premium)
0x582: 0
```
Точні значення читаються runtime'ом без жодного патчу: увімкнути audio debug
рівень ≥3 → CheckHashkey друкує summary (0x4d0/0x43d/0x43e/0x440/0x444,
printk-блок 0x4248e4-0x424914), або виклик `MDrv_AUDIO_Debug_Cmd_Read`
(дампить ті самі поля). **Це перший крок майбутнього експерименту.**

## 6. DTS-ONLY (справжня емуляція, оновлена)

Емулювати PASS групи {0xf, 0x3a, 0x12, 7} — 4 singe-word NOP-и на fail-гілках:

```
0xf : 0x423a84  0a00000f? -> e1a00000 (nop)  ; fail → 0x440|=8 не виконується;
                                             ; pass → 0x4d8=1
0x3a: 0x423c00  -> e1a00000                  ; 0x440|=0x80 не виконується
0x12: 0x423eec  -> e1a00000                  ; 0x440|=0x20000 не виконується;
                                             ; 0x4d8=2; 0x440 &= ~0x400
7  : 0x4246ec  -> e1a00000                    ; 0x4d8=3; 0x582=1; 0x444&=~0x100
```
(мінімум — тільки 0xf → 0x4d8=1; повна сім'я — усі 4).

Результат = точний стан "DTS-ліцензованого" пристрою: `0x4d8=3, 0x582=1,
0x440: DTS-біти чисті, 0x444 bit8 чистий`, **0x4d0/0x43d/0x43e/образи — без
змін**. Порівняно з попереднім кандидатом (примусове 0x43d=0x43e=1 + NOP
0xb/0xc-гілок): попередній змінював Dolby-преміум-прапори (побічні ефекти в
SetSystem2/AutoMode/GetAudioInfo2) і патчував перевірки, які й так проходять.
Єдине, що варто зберегти зі старого плану — **fallback**: якщо runtime
покаже `0x43d==0` і `SetSystem2` пише DSP-байт 0x0d у DTS-гілці (це зʼясується
дампом/логом), тоді додатково примусово `0x43d=1` (1 слово у хвостовому блоці,
0x424880).

## 7. FULL-LICENSE (реконструкція стану "все ліцензовано")

При PASS усіх 43 перевірок (порядок коду фіксований, перезаписи
детерміновані):

```
0x440 = 0x00001000        (тільки аномальний PASS-біт 0x7f; усі missing-біти чисті)
0x444 = 0x003FFE80        (біти 0..6,8 стерті сімействами; 7,9 підняті 0xa/0x7d)
0x4d0 = 4                 (останнім пише 0x7d → MS12V22-образи!)
0x4d4 = 8                 (0x7d; було б 9, якби 0x7d не пройшов)
0x4d8 = 3                 (7 = DTS:X — повне DTS-сімейство)
0x43d = 1, 0x43e = 1      (0x7d)
0x57e=0x57f=0x580=0x581=0x582=1
strap 0x112cf0: bit2/3/7/10/11/12 = 1 (апаратно, від AUTH-демона)
```

Що відкриє full unlock (за матрицею + Get_*_License): локальне декодування
**DTS/DTS-HD/DTS:X** (0x4d8=3), **AC4** (0x79), **WMA** (0x1e), **DRA**
(0x41), **MPEG-H**, **Cook**, **TrueHD/MAT-декод** (0x54-family),
повний **MS12 v2.2** пост-процесинг (0x4d0=4 → MS12V22 образи).
Строкаті single-id (0x3/0xd/0x37/0x45/0x1c/0x38/0x42/0x49/0x53/0x7f) —
невідомі кодеки/фічі, ймовірно регіональні (DRA/Cook = Китай).

## 8. Відповідь на головне питання

> Чи містить firmware готовий вищий/повний профіль, який можна активувати?

**Ні, як конфігурації/перемикача — не існує.** Але **так, як точно визначеного
стану даних**: full-profile = конкретні значення 8 полів g_AudioVars2 + маска
0x440, які код обчислює з AUTH-таблиці детерміновано. Активувати його можна
двома шляхами: (а) підкласти повну AUTH-таблицю (140 байт через ioctl
0xC08C5506 — але її перезапише AUTH-демон при наступній перевірці) або
(б) емулювати pass потрібних IP-перевірок у CheckHashkey (стабільно).
Для DTS-transport **повний профіль не потрібен і небажаний** (він перемикає
R2-образи на MS12V22): достатньо DTS-підмножини (§6) — 4 слова, нуль
побічних ефектів на Dolby-профіль і образи.

## 9. Side effects повного розблокування (для повноти)

- 0x4d0=4 → R2-пари MS12V22: інший пост-процесинг (MS12), інші бінарники —
  змінює звук на вбудованих динаміках і поведінку Dolby-ланцюга.
- 0x4d8>0 → HDMI non-PCM type selection (`HAL_AUDIO_HDMI_SetNonpcm`) і
  GetAudioInfo2 {0xa,0x10} статуси починають враховувати DTS — для
  transport-only SPDIF це нейтрально.
- `HAL_AUDIO_SPDIF_AutoMode` (0x43e-гілка) активує DTS-автологіку —
  корисно для DTV-джерел, не заважає Android-шляху.
- Status: біти 0x3/0xd/0x37/0x45/0x1c/0x38/0x42/0x49/0x53/0x7f мають
  невідому семантику — forcing їх без потреби не рекомендовано.

## 10. Рекомендована реалізація (ранжовано)

1. **Runtime-рекогносцировка (без патчів):** зняти поточний стан
   (debug-принт CheckHashkey при dbg≥3 або MDrv_AUDIO_Debug_Cmd_Read) →
   точно знати 0x440/0x444/0x43d/0x43e/0x4d0/0x4d4/0x4d8.
2. **Targeted license-state emulation (новий основний кандидат):** 4 NOP-и
   §6 (0xf/0x3a/0x12/7) в utpa2k.ko; fallback +1 слово (0x43d=1) якщо
   SetSystem2-байт виявиться 0x0d. Далі — план транспорту з
   REPORT_DTS_transport_solution.md (libmi3 v3 + Kodi RAW sink).
3. **Genuine full profile (не рекомендовано для transport):** емуляція
   pass усіх 43 → 0x4d0=4 → MS12V22; виправдано лише якщо ціль — локальний
   DTS/MAT/AC4-декод, а не транспорт.
4. **AUTH_IPCheck broad override (last resort):** повернення 1 для всіх id —
   еквівалент (3), але мутує також Get_*_License query-результати; жодних
   переваг перед (3), більший blast radius.

## Додаток: артефакти

- `tools/dump_checkhashkey.py`, `tools/ut_checkhashkey_full.asm` (1472 рядки,
  43 IPID-анотації), `tools/dump_funcs.py`, `tools/ut_licenses.asm`,
  `tools/ut_ddload.asm`.
- Ключові адреси: CheckHashkey 0x423494; IPCheck 0x1390c; ApplyHashkey
  0x424b90; SE-IDMA push 0x424bd8 (маска байтами → 0x112A82..0x112A85,
  dsp-id 0x83/0x84, kick 0x1f); SetDecodeSystem 0x424e78; SetSystem2
  0x44d0b0 (0x43d-гілка @0x44d2c0, байт 5/0xd, рег-база 0x112E98);
  SPDIF_SetMode 0x448a10 (2→BypassMode); DspLoadCode 0x46f5a8
  (0x4d0==4 → MS12V22); GetLicense-диспетчер 0x426c1c (строкові
  'GetLicenseAac/Ac3/Ac4/MpegH/Wma/Dts/Dra/Cook/Mat/Truehd');
  AUTH-таблиця: ioctl 0xC08C5506, 140 байт, буфер 0x8c.
