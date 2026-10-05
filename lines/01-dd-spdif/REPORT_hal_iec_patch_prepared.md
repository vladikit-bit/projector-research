# Підготовлений reversible HAL patch: IEC61937 → AC3-програма (gate 1 + формат-мапа)

Дата: 2026-08-30 · Об'єкт: `audio.primary.mt5889.so` (оригінал md5 `8c11c348…`).
**Статус: ПІДГОТОВЛЕНО, НЕ ВСТАНОВЛЕНО.** Файл: `libs/apm_iec_patched.so` (md5 `6ce18216…`).

---

## 1. Три точкові зміни (усі в `adev_open_output_stream` / `mi_decoder_open`)

### Gate 1 — flags IEC958_OPTIMAL
```text
file 0x21de8 (ELF 0x22de8, Ghidra 0x32de8):
до:   7a d4    bmi  0x22ee0       ; if (flags & 0x400) → "unsupport flags" → -EINVAL
після: 00 bf    nop                 ; завжди нормальний шлях
```
Контекст: `blx <log>; lsls r0,r5,#21; [nop]; movs r0,#1`.

### Формат-мапа — поріг AC3-сім'ї
```text
file 0x2e4e2 (ELF 0x2f4e2):
до:   b0 f1 30 6f   cmp.w r0, #0xb000000
після: b0 f1 60 6f   cmp.w r0, #0xe000000
```
→ 0x0d000000 тепер входить в AC3-сімейний блок (разом з 0xb/0xc — див. побічку §4).

### X-addend внутрішньої перевірки
```text
file 0x2e4ea (ELF 0x2f4ea):
до:   00 f1 76 42   add.w r2, r0, #0xf6000000   ; X(0x0d)=0x03000000 → fail
після: 00 f1 73 42   add.w r2, r0, #0xf3000000   ; X(0x0d)=0 ✓ ≤ 2 → pass
```
Після: `cmp r2,#2; blo 0x2f52e` → 0 < 2 → OK → тип 5.

## 2. Підсумкова таблиця мапінгу (після патчу)

| Android format | до | після |
|---|---|---|
| 0x09000000 AC3 | 5 (AC3-програма) | 5 ✓ без змін |
| 0x0a000000 E-AC3 | 5 | **5** ✓ без змін |
| 0x0b000000 DTS core | 9 (DTS-програма, license-blocked) | **5** (AC3-програма) |
| 0x0c000000 DTS-HD | 9 | **5** |
| 0x22000000 TrueHD | 6 | 6 ✓ |

`utils_get_audio_parser_name(0x0d000000)` → default-рядок; `utils_isParserSupported` — перевірити runtime (після патчу open проходив у гейт-1 експерименті навіть без парсера).

## 3. Очікуваний ланцюг після встановлення (не виконано)

```text
Kodi 22 IEC sink → AudioTrack(ENCODING_IEC61937, 48k, stereo)
→ AudioPolicy: IEC61937 профіль (policy-фійкс вже встановлений) ✓
→ AudioFlinger openOutput(flags 0x411)
→ adev_open_output_stream: gate 1 NOP ✓ → threshold 0xe ✓ → X(0x0d)=0 ✓
→ offload open OK (без ADEC-програми, дані як PCM-подібний вхід)
→ mi_common_raw_write → DSP вхід
→ DSP: decode-програма типу 5 (AC3) ЗАПУЩЕНА → bypass-тап входу → SPDIF TX non-PCM
→ дріт: IEC61937-AC3 берсти Kodi + non-audio CS → ресівер = Dolby Digital
→ для DTS: Kodi IEC61937 type 0x0B берсти → дріт → ресівер = DTS (без ADEC DTS)
```

## 4. Встановлення (запускається окремою командою, зараз НЕ виконано)

```bash
adb push libs/apm_iec_patched.so /data/local/tmp/apm_iec_patched.so
adb shell 'mount --bind /data/local/tmp/apm_iec_patched.so /vendor/lib/hw/audio.primary.mt5889.so
stop vendor.audio-hal; start vendor.audio-hal'
```
(перед біндом: зняти стекнутий залишок — `umount` доки не звільниться; або reboot)

## 5. Rollback

```bash
adb shell 'umount /vendor/lib/hw/audio.primary.mt5889.so   # зняти bind (може бути busy — спочатку stop vendor.audio-hal)
cat /data/local/tmp/apm_orig.so > /vendor/lib/hw/audio.primary.mt5889.so && sync
stop vendor.audio-hal; start vendor.audio-hal'
adb shell md5sum /vendor/lib/hw/audio.primary.mt5889.so    # = 8c11c348…
```

## 6. Ризики / побічні ефекти

- DTS/DTS-HD RAW (native offload) тепер мапляться в AC3-програму: bypass-тап виграє (DTS байти на TX), але speaker-декод AC3-програмою над DTS-даними — garbage на динаміках (на цьому пристрої DTS-декод і так був заблокований → було тишу).
- E-AC3 мапінг ламається (X(0xa)=0x9d… → error) — в kodi22 `eac3passthrough=false`, Kodi падає в decode→PCM. Прийнятно.
- На відомих форматах поза списком (garbage) — без змін (error).
- Policy IEC61937 профіль — вже встановлений раніше (reversible), залишається.

## 7. Статуси

- Патч побудований, байт-точно верифікований (assert оригінальних байтів при створенні) — CONFIRMED
- Мапінг-симуляція всіх 6 форматів — CONFIRMED (таблиця §2)
- Встановлення/тест — ЗАЧЕКАЄ КОМАНДИ
