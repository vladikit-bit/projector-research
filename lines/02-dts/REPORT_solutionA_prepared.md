# Solution A — offline preparation report (variant A: license-gate patch + libmi3 v3 + Kodi RAW)

Дата: 2026-08-30. Пристрій НЕ змінювався. Усі артефакти створені офлайн у workspace.

```text
PATCH STATUS: PREPARED & VERIFIED (offline; не встановлено на пристрій)

ORIGINAL utpa2k:
path:    C:\firmware_temp\spdif_audio_investigation\kmods\utpa2k.ko
size:    25381336 (0x18349D8)
MD5:     2fc6e9fc46b6402d73aa6f84f9599d46
SHA256:  8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da

PATCHED utpa2k:
path:    C:\firmware_temp\spdif_audio_investigation\libs\utpa2k_dts_patched.ko
size:    25381336 (ідентичний)
MD5:     f3ce08513be3b6276a7f2cf0e7103d14
SHA256:  4199bc60e06a296ba9318491e360c605f10c85f3fe818c53719ad8b22c3cb6c1
pather:  tools\patch_utpa2k_dts.py (ідемпотентний, з assert'ами на original bytes)
diff vs original: 22 bytes (7 слів) у діапазоні file 0x430b68..0x431ee3,
                  всі всередині MDrv_AUDIO_CheckHashkey [0x423494..0x424B90]
```

## PATCH VALIDATION

**address checks (усі всередині MDrv_AUDIO_CheckHashkey):** OK
```
0x423514 (file 0x430b68)  ✓ in-function
0x423594 (file 0x430be8)  ✓ in-function
0x42487c (file 0x431ed0)  ✓ in-function
0x424880 (file 0x431ed4)  ✓ in-function
0x424884 (file 0x431ed8)  ✓ in-function
0x424888 (file 0x431edc)  ✓ in-function
0x42488c (file 0x431ee0)  ✓ in-function
```

**original byte checks:** OK — всі 7 слів точно відповідають expected перед записом
(assert у патчері):
```
0x423514: 0a00000c ✓   0x423594: 0a000011 ✓   0x42487c: e5c0743e ✓
0x424880: e5c0a43d ✓   0x424884: 0a000007 ✓   0x424888: e59004c8 ✓
0x42488c: e3500003 ✓
```

**ARM decode (patched words):** OK
```
e1a00000  mov  r0, r0        (ARM NOP)
e3a07001  mov  r7, #1
e5c0743e  strb r7, [r0, #0x43e]
e3a0a001  mov  sl, #1
e5c0a43d  strb sl, [r0, #0x43d]
ea000005  b    0x4248a8
```

**control flow:** OK
- `b 0x4248a8`: 0x42488c+8+5*4 = 0x4248a8 → `bl HAL_AUDIO_GET_INIT_FLAG (0x4411ac)`
  → прапор 0x503 → епілог. Шлях GET_INIT_FLAG повністю збережений.
- Orphan-блок (колишній debug-summary log 0x424890–0x4248a4) — мертвий:
  0 реальних branch-таргетів у нього (скан гілок + скан R_ARM_CALL/JUMP24
  релокацій усього .text: 0 warnings). Налаштування debug-рівня >3 втратить
  лише друк summary-рядка CheckHashkey (косметика).
- Branch-target scan: жодна гілка (ні в функції, ні міжсекційна через релокації)
  не цілиться в жодну з 7 патчених адрес.

**register/lifetime safety:** OK
- `r0` у момент обох `strb` = pointer на g_AudioVars2: завантажений `ldr r0,[r4]`
  на 0x424868 і не перезаписується до 0x424888 (str-и/mov-и r0 не чіпають).
- `r7`/`sl` після блоку не потрібні: єдиний дальший шлях 0x4248a8→0x4248bc
  використовує r0/r1/r4; обидва регістри callee-saved і відновлюються епілогом
  `pop {r4,r5,r6,r7,r8,sb,sl,pc}`. Пропущений log-блок перевантажував їх сам.
- **MS12/0x4d0 недоторканий:** `str r6,[r0,#0x4d0]` (0x424870), `str r5,[r0,#0x4d4]`,
  `str r8,[r0,#0x4d8]` — в оригіналі; r6/r5/r8 патч не змінює (ruхає тільки r7/sl).
  Інвентаризація всіх 44 `MDrv_AUTH_IPCheck` викликів у функції: примусово
  «pass» тільки id=0xb (0x423508) та id=0xc (0x423584); усі інші 42
  (Dolby 0x50/0x73/0x53/0x52/0x75/0x51/0x74, MS12-suite 0xe/9/8/0xa/0x7d/0x7f/0x66/0x54,
  DTS:X-запити Get_DTS_License 0xf/0x3a/0x12/07 тощо) — семантика оригінальна.

**Семантика після патчу (підтверджено пост-патч дизасемблюванням,
tools/ut_checkhashkey_postpatch.asm):**
```
IPCheck(0xb) → forced PASS   (beq fail @0x423514 → nop; maskA |= 1 ніколи)
IPCheck(0xc) → forced PASS   (beq fail @0x423594 → nop; pass-шлях: maskA &= ~1, 0x581 |= 1)
maskA bit0   → cleared       (bic r0,r2,#1 @0x423598 → str @0x42359c)
0x581 bit0   → set           (orr/strb @0x4235a8..0x4235ac)
0x43e        → forced 1      (mov r7,#1 @0x42487c → strb r7,[r0,#0x43e] @0x424880)
0x43d        → forced 1      (mov sl,#1 @0x424884 → strb sl,[r0,#0x43d] @0x424888)
0x4d0/0x4d4/0x4d8 (MS12)    → untouched
HAL_AUDIO_GET_INIT_FLAG path → preserved (b 0x4248a8 → bl → strbeq 0x503)
```
Споживачі, які від цього залежать: ApplyHashkey → SE-IDMA маски в R2 (0x440 bit0=0
= DTS дозволено), HAL_AUDIO_SetSystem2 DTS-кейс пише байт 5 замість 0xd (0x43d==1),
SPDIF AutoMode/TranscodeMode включають DTS-гілки (0x43e==1).

## LIBMI3 V3

**ВАЖЛИВЕ ВИЯВЛЕННЯ:** файл `libs/libmi3_patched_v2.so` на диску мав md5
`d71a330f...` — це **v2**-контент (OR 0x2E1, той самий, що стоїть на пристрої);
заявлений раніше «v3»-артефакт (b5425941) **не зберігся**. v3 реконструйований
з оригіналу і верифікований.

```text
LIBMI3 V3:
path:               C:\firmware_temp\spdif_audio_investigation\libs\libmi3_dts_v3.so
built from:         libs\libmi3.so (MD5 c2b5c57dba4f89228da72a29c71f0c06)
MD5:                4c199a5c739e61741ba731534fbb3bf4
SHA256:             edac3dd78cc46fe9bc4f85d08ec28419b0589202b6b4823b63706a29a12987d3
size:               750608 (ідентичний оригіналу)
diff vs original:   24 bytes @ file 0x6163e..0x61655 (va 0x6263e..0x62655)
builder:            tools\build_libmi3_v3.py
DTS capability bits: OR-константа 0x070002E1 =
                     біти 24/25/26 (сім'ї DTS / DTS-HD / IEC61937) +
                     біти 0/5/6/7/9 (як у runtime-протестованому v2 для AC3)
identity verified:  YES (дизасемблювання нижче; біти 24/25/26 у константі movt)
```

Патч-послідовність (va 0x6263e, MI_AUDIO_GetCaps attr-блок):
```
0x6263e: ldr  r1, [r5, #-4]      ; caps-значення
0x62642: movw r2, #0x2e1
0x62646: movt r2, #0x700         ; ← єдине семантичне доповнення проти v2
0x6264a: orrs r1, r2             ; caps |= 0x070002E1
0x6264c: str  r1, [r5, #-4]
0x62650: movs r4, #0             ; повертає success
0x62652: b.w  0x62658            ; join (stack-guard/return) — target ідентичний v2
```
Зауваження: v2 (runtime-протестований на пристрої для AC3) відрізняється лише
відсутністю `movt` і b.w на 4 байти раніше; v3 поглинає 2 мертві байти
(0x62652–0x62653, хвіст перезаписаного blx — у v2 вони вже були за b.w).
Гілок у регіон [0x6263e..0x62656) ззовні немає (скан усього .text: 0).

## UNMODIFIED COMPONENTS (умова §4 дотримана)

- R2 firmware (`mst_snd_r2*`, `mst_codec_r2*` всередині utpa2k.ko) — байти не чіпались
  (патч лише в .text MDrv_AUDIO_CheckHashkey).
- `audio.primary.mt5889.so` — оригінал (8c11c348), не патчився.
- Kodi (усі 3 білди) — не чіпались; на етапі runtime лише вибір пристрою
  "AudioTrack (RAW)" у налаштуваннях.
- audio policy XML — не чіпався (AUDIO_FORMAT_DTS/DTS_HD вже є у vendor XML).
- `MDrv_AUTH_IPCheck` та всі інші функції utpa2k.ko — байти оригінальні (22-byte
  diff тому доказ).

## RUNTIME READY: YES (offline artifacts complete)

Мінімальний runtime-set коли буде дозволено встановлення:
1. `utpa2k.ko` → `libs/utpa2k_dts_patched.ko` (знайти шлях модуля через
   `/proc/modules` або /vendor/lib/modules; зберегти оригінал; заміна+ребут).
2. `libmi3.so` → `libs/libmi3_dts_v3.so` (замість поточного v2 d71a330f).
3. Kodi → налаштування аудіовиходу "AudioTrack (RAW)".
4. Тест-порядок: AC3-регресія → DTS-файл → logcat (`MI_AUDIO_Start ... 9`,
   відсутність error SetDecodeSystem) → індикатор DTS на AVR.

## BLOCKERS / RISKS

1. **R2-внутрішня перевірка (головний невідомий).** Патч гарантує лише host-стан
   (маски SE-IDMA + SetSystem2 байт 5 + AutoMode). Якщо R2 DTS-програма має
   власний strap-вестібіль поза доставленою маскою — decode стартує, але sync/
   bypass не увімкнеться. Вірогідність низька (маска доставляється саме для
   цього; AC3 працює тим самим механізмом), але це strongly inferred, не proven.
   Діагностика: MDrv_AUDIO_Debug_Cmd_Read дамп 0x440/0x444/0x43d/0x43e після
   старту; logcat SPIDIF mode принти.
2. **`HAL_AUDIO_CheckHashkeyDone(maskA)` консистентність.** Функція панікує при
   несподіваній масці (-22). Стертий bit0 = валідна «ліцензована» комбінація,
   ризик низький; контингенція — додатковий мікропатч бітів [0x444].
3. **Перевірка підпису kernel-модулів.** vermagic: `4.19.116+ SMP preempt
   mod_unload modversions ARMv7`; патч не чіпає vermagic/символи/CRC (modversions
   інтактні). Якщо ядро має CONFIG_MODULE_SIG_FORCE — патчений .ko не
   завантажиться; тоді шлях — патч у пам'яті або інший механізм завантаження
   (перевіряється на етапі runtime одним dmesg-рядком).
4. **CheckHashkey кешується** прапором 0x503: патч ефективний лише якщо модуль
   замінений ДО першого SetDecodeSystem (тобто ребут після заміни — обов'язковий).
5. libmi3 v3 реконструйований (не той файл, що готувався раніше) — але побудований
   з runtime-протестованого дизайну v2 і повністю верифікований дизасемблюванням.
