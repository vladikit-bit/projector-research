# HAL gate-1 bypass experiment: результат і наступна точка відмови

Дата: 2026-08-29 · Об'єкт: `/vendor/lib/hw/audio.primary.mt5889.so` (md5 оригіналу `8c11c348…`, backup: `/data/local/tmp/apm_orig.so` + локальна копія). Обхід виконано через bind-mount, **повністю відкочено**, md5 оригіналу верифіковано.

---

## 1. Точний патч першого гейту

Файл: `audio.primary.mt5889.so` (ARM32 Thumb), функція `adev_open_output_stream`.

Ghidra-адресний простір: entry `0x32d4c`, гейт на `0x32de6/0x32de8` (Ghidra image base = 0x10000).
ELF-файл: **file offset `0x21de8`** (VA `.text` `0x22de8`), 2 байти:

```text
до:   7a d4        bmi  0x22ee0            ; if ((flags<<21) < 0) → reject
після: 00 bf        nop                      ; fall-through → normal open path
```

Контекст (ELF VA `.text`, Thumb):

```text
0x22de0  blx   <__android_log_print>     ; "…fmt/chn/sr %d/%d/%d, flag 0x%08x"
0x22de6  lsls  r0, r5, #21              ; flags & 0x400 → N-flag
0x22de8  bmi   0x22ee0                  ; ★ ЦЕЙ ГЕЙТ (2 байти)
0x22dea  movs  r0, #1                   ; нормальний шлях (r0 одразу перезаписується)
```

Патч = заміна умовної гілки BMI на NOP: виконання завжди йде нормальним шляхом; жоден інший байт не змінено (файл +2→0 байт дельти, розмір незмінний 265604 B).

Прапорець, що перевіряється: біт 10 = `AUDIO_OUTPUT_FLAG_IEC958_OPTIMAL` (0x400) — його додає framework для `ENCODING_IEC61937` (0x411 = DIRECT|OFFLOAD|IEC958_OPTIMAL).

## 2. Runtime-результат (Kodi C = net.kodinerds.maven.kodi22, native AC3)

До патчу (baseline цієї фази):
```text
adev_open_output_stream: flag 0x411 → "unsupport flags 0x411" → -EINVAL
```
Після точкового патчу (з dmesg/logcat):
```text
AudioFlinger: openOutput(0x0d000000, flags 0x411)
audio_hw_primary: adev_open_output_stream: flag 0x411        ← гейт ПРОЙДЕНО (без "unsupport flags")
audio_hw_primary: open output stream with offload …           ← offload-гілка виконана
audio_hw_primary: out=0x…, isAvDelaySet=1                     ← стрім відкрився на MI-рівні
audio_hw_primary: adev_open_output_stream: unsupport format (0x0d000000)   ← ★ НАСТУПНА ВІДМОВА
```
Kodi: `Unable to create AudioTrack` → `no sink was returned` (AudioTrack не створився; відтворення не почалось).

## 3. Наступна точка відмови — ідентифікована точно

`utils_common_out_get(stream, PARAM_9)` → `mi_common_raw_get` **case 9** → switch по `format >> 24`:
```text
0x4/0x5/0x6 → caps bit 7      0x9/0xa → bit 5      0xb/0xc → bit 9      0x22 → bit 6
0x0d (IEC61937) → default → caps bit 0
```
→ `MI_AUDIO_GetCaps()` (libmi3, наш патч OR-ить **0x2E0 = bits 5,6,7,9**) → **bit 0 = 0** → перевірка fail → `unsupport format (0x0d000000)` → open → -EINVAL.

Тобто другий гейт — це **формат-фамілі мапа в HAL** + відповідний біт capability, який наш існуючий libmi3-патч не виставляє.

## 4. Rollback — підтверджено

- bind-mount знятий (залишок неактивний, контент ідентичний оригіналу); на випадок займання — чиститься reboot.
- `/vendor/lib/hw/audio.primary.mt5889.so` md5 = `8c11c348…` (оригінал), HAL running.
- Оригінал збережено: `/data/local/tmp/apm_orig.so` + локальні копії. Patched-варіант: `libs/apm_gate_patched.so`.

## 5. Що це означає для шляху IEC61937

Повний ланцюг відмов (після policy-фікса):
```text
✓ AudioPolicy: IEC61937 профіль (reversible policy-фійкс, залишено)
✓ AudioFlinger: openOutput доходить до HAL
✗ HAL gate 1: flags & IEC958_OPTIMAL → БУВ ОБМОЖЕНИЙ (цім експериментом)
✗ HAL gate 2: format-family caps (0x0d → caps bit 0) → НЕМОЖЛИВИЙ БЕЗ ДРУГОГО ПАТЧА
```
Мінімальний варіант gate 2: розширити наш **вже існуючий reversible libmi3-патч** `0x2E0 → 0x2E1` (один нібл!) — тоді `0x0d → bit 0` пройде. АЛЕ: після цього open впаде/зійде далі на `utils_get_audio_parser_name(0x0d000000)` (мапінг парсера не містить 0xd — default) та/або `mi_decoder_open` (мапінг 0xd відсутній) — тобто потрібен аналіз, що станеться після; і навіть при успішному open дані IEC61937 потраплять в декодерний вхід, який очікує raw-кодек — подвійне фреймування. Це вже зона, де реалізація MStar принципово не передбачала IEC61937-транспорт через ADEC.

## 6. Стан системи після експерименту

- HAL: оригінал (md5 verified), running ✓
- Policy: IEC61937 профіль залишено (зворотно: `cp .pre_iec → audio_policy_configuration.xml` + restart audioserver) ✓
- libmi3: наш робочий патч (DD baseline) без змін ✓
- Kodi A 21.2: без змін ✓
- Kodi C: `passthroughdevice=IEC` відновлено, eac3/dtshd/truehd=false, dtspassthrough=true, ac3transcode=true ✓
- Прибрані артефакти: `/data/local/tmp/apm_{orig,patched}.so` залишено як backup; bind-remnant після reboot зникне сам.
