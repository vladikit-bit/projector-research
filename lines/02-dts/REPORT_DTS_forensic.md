# Forensic Report: AC3 працює на SPDIF, DTS — ні. Доказовий ланцюг.

Дата: 2026-08-29 · Метод: статичний аналіз (capstone + ELF symbols/PLT + strings) за збереженими артефактами + runtime-спостереження попередньої сесії. Жодних змін на пристрої не внесено.

---

## 1. Точний AC3 path (CONFIRMED)

```text
AudioTrack(ENCODING_AC3, DIRECT|OFFLOAD)
→ AudioFlinger offload thread (format 0x09000000)
→ adev_open_output_stream: GRM acquireHwResource, parser "ac3" (utils_get_audio_parser_name)
→ mi_decoder_open(config):
    format 0x09000000 → codecType = 5        (мапінг: AC3/E-AC3/JOC→5, TrueHD→6, PCM→7, DTS/DTS-HD→9)
    MI_AUDIO_Open(attr{name?, 3, 0})         → ioctl 0xC04C1003 → handle 0x19000000 (екземпляр "AudioHAL")
    MI_AOUT_ConnectInput(aout, handle, &f)   → підключення декодера до AOUT
    MI_AUDIO_SetSpeed(handle,{1000,8,...});  MI_AUDIO_SetAttr(0x606,1)
    MI_AUDIO_Start(handle, &codecType(5))    → ioctl 0xC0081005, payload {handle, 5}
    MI_AUDIO_SetAttr(0x204 / 0x205=1000 / 0x607)
→ utils_common_out_write → Parser_Write → Parser_Do_Parse → (framing)
→ mi_common_raw_write → MI_AUDIO_Write(handle,&buf,&len) → ioctl 0xC038100A
→ compressed AC3-фрейми йдуть у DSP INPUT БЕЗ ЗМІН (parser не декодує)
→ DSP декодує DD + SPDIF(AUTO) відмічає non-PCM → фізичний "Dolby Digital" (підтверджено користувачем)
```

## 2. Точний DTS path (CONFIRMED до того ж місця)

```text
AudioTrack(ENCODING_DTS, DIRECT|OFFLOAD)
→ offload thread (format 0x0b000000)
→ GRM acquire, parser "dts"
→ mi_decoder_open: format 0x0b000000 → codecType = 9
    MI_AUDIO_Open → ТОТ САМИЙ handle 0x19000000
    MI_AOUT_ConnectInput → так само
    MI_AUDIO_Start(handle, &codecType(9)) → ioctl 0xC0081005 payload {handle, 9}
    MI_AUDIO_SetAttr(0x204/0x205/0x607) → так само
→ Parser_Write/Do_Parse → dts-фрейми по 2012 байт (спостережено runtime: "Out f(5805) frame size[2012]")
→ MI_AUDIO_Write → compressed DTS у DSP INPUT БЕЗ ЗМІН
→ ДАЛІ РОЗХОДЖЕННЯ: codec-type атрибут handle'а читається як NONE (замість AC3, як для DD)
→ SPDIF(AUTO) не переходить у non-PCM → ресівер не бачить DTS
```

## 3. Перша доведена точка розходження (STRONG EVIDENCE)

**Userspace ідентичний до байта** — той самий код, різне лише значення codecType (5 vs 9), і kernel приймає обидва START-ioctl без помилки (жодного "failed" у логах; фінальний лог "format 9 got decoder handle 0x19000000" друкується після успішного Start). Розходження проявляється **у kernel/DSP-стані**: зворотне читання codec type екземпляра "AudioHAL" дає `AC3` для DD і `NONE` для DTS (runtime-спостереження попередньої сесії: `dumpsys` "AudioHAL audio codec type" = AC3 при DD, NONE при DTS; kernel друкує `_MI_AUDIO_GetCodecType: Get codec type None!`).

## 4. Перевірка гіпотез

| Гіпотеза | Вердикт |
|---|---|
| "dts_parser декодує DTS → PCM" | **СПРОСТОВАНА** (userspace): libaudioparser — тільки sync/framing (символи preparse/probe/read; 2012-байтні DTS-фрейми виходять з парсера як є; жодних PCM-перетворень у 64KB бібліотеці немає) |
| "DTSDecSDOPacker / HAL_AUDIO_SPDIF_* — потрібний шлях" | **СПРОСТОВАНА для Android offload**: audio.primary імпортує тільки libmi3; libmi3 імпортує з libutopia ЛИШЕ CMA/системні MApi (0 × HAL_AUDIO_SPDIF_*, 0 × MApi_AUDIO_SPDIF_*). Ці функції — для native/DTV/OMX-стеку. До того ж AC3 працює БЕЗ будь-якого userspace-пакера |
| "HAL не передає codecType=9 в kernel" | **СПРОСТОВАНА**: START-ioctl 0xC0081005 несе {handle, codecType} дослівно для обох |
| "DTS декодується в PCM у DSP (DD- licensed, DTS-ні)" | **UNKNOWN** — немає прямого доказу; і декодування, і відмова старта дають однаковий userspace-слід |
| "Kernel/DSP build не має DTS-декодера/мапінгу для type 9 (продуктова конфігурація без DTS-ліцензії)" | **INFERENCE (сильна)** — єдиний несуперечливий варіант; підтримується нульовими GetCaps-бітами ( OEM-таблиця, а не реальні можливості: AC3 працює попри bit5=0) |

## 5. Root cause ( formulation )

> Kernel/DSP рівень приймає START з codecType=9, але не реєструє/не запускає DTS-декомпресійний обробник для екземпляра "AudioHAL": атрибут codec type залишається NONE, тому AUTO-логіка SPDIF ніколи не перемикає TX у non-PCM для DTS. DD (type 5) у цій продуктовій збірці присутній, DTS (type 9) — відсутній. Це конфігурація рівня kernel-драйвера ([mik]/[dtv_driver]/DSP), а не userspace.

Точна kernel-гілка (мапінг 9→"не підтримується" vs DSP-відмова) — UNKNOWN без дизасемблювання ядра; локально kernel-бінарника немає (E:\ з екстрактами недоступний).

## 6. Відповідальні бінарники

- `audio.primary.mt5889.so` — коректний, змін не потребує (мапінг 9 передається).
- `libmi3.so` — коректний транспорт (ioctl-таблиця нижче).
- `libaudioparser.so` — коректний (тільки framing).
- `libutopia.so` — НЕ у цьому шляху (native/DTV path).
- **Kernel [mik] + [dtv_driver] + DSP firmware** — САМЕ ТУТ: обробка START{codecType} → реєстрація декодера → codec-type атрибут → SPDIF AUTO flagging.

ioctl-таблиця MI_AUDIO (з libmi3.so): Open 0xC04C1003 · Start 0xC0081005 {handle,codecType} · Write 0xC038100A · GetAttr 0xC0181013 · SetAttr 0xC0201014 · GetCaps 0xC0141002 · SetCodecParams 0xC08C100C (HAL не використовує).

## 7. Кандидати на фікс (ПРОПОЗИЦІЇ, не застосовано)

1. **Kernel-рівень (правильний):** у обробнику START/SetDecType додати type 9 (DTS) у таблицю підтримуваних декодерів "AudioHAL" екземпляра так само, як 5. Потребує дизасемблювання [mik]/[dtv_driver] (через /proc/kcore з root-adb або kernel-образ з ree_payload.bin) → точна функція _MTAUDDEC_SetDecType/mik START-handler.
2. **Перевірка спрощеного сценарію (без патчу):** перемкнути SPDIF mode AUTO→BYPASS у TV-settings на час DTS-відтворення. Якщо DTS з'явиться — DSP отримує дані і не вистачає лише AUTO-flagging (тоді можливий маленький kernel-хук: примусовий non-PCM для offload-стрімів з codecType 9). Якщо тиша — дані не доїжджають до SPDIF TX (старт декодера проваленося) → тільки варіант 1.
3. **НЕ робити:** виклики DTSDecSDOPacker/HAL_AUDIO_SPDIF_* з HAL — не цей шлях; підмена codecType 9→5 — неправильний декодер.

## 8. Runtime-тести, що остаточно підтвердять (на майбутнє)

1. `dmesg -w` під час DTS offload-відтворення: рядки `_MI_AUDIO_GetCodecType`, `SetDecType`, декодер-провали (і те саме для AC3 як baseline).
2. `dumpsys media.audio_flinger` (HAL-секція) **під час** відтворення: "AudioHAL audio codec type" — AC3 vs NONE (вже частково зібрано).
3. Експеримент BYPASS (п.7.2) — розділяє "AUTO-flagging" vs "data path".
4. `grep -E "SetDecType|DecType|AUDDD" /proc/kallsyms` + `/proc/kcore` дизасембл START-handler — точна kernel-гілка.
5. Тест DTS 44.1 kHz (наш тест-файл 48k): виключити специфіку ≤32k-правила (adev->0x3c98==3 && sr<=0x7d00 → DTS rejected у capability check).

## Рівні впевненості

- Userspace-шляхи AC3/DTS, ioctl-таблиця, відсутність PCM-конверсії, невикористання libutopia: **CONFIRMED**
- Розходження в kernel/DSP-стані (codec type AC3 vs NONE): **STRONG EVIDENCE** (live-спостереження попередньої сесії)
- Root cause = відсутній DTS-мапінг у kernel/DSP продуктовій конфігурації: **INFERENCE (сильна)**
- Чи декодує DSP DTS у PCM: **UNKNOWN** (несуттєво для фіксу; користувач не чує нічого на ресівері)
