# Фаза 4: Матриця шляхів passthrough — IEC (Kodi/Android), native offload, AC3-transcode

Дата: 2026-08-29 · Встановлена версія: **Kodi 21.2 Omega** (versionCode 2102000) — НЕ Piers/Maven; усі висновки звірені з вихідниками гілки Omega.

---

## 0. Нова конфігурація (робоча, задокументована)

| Параметр | Значення | Чому |
|---|---|---|
| `audiooutput.passthrough` | true | |
| `audiooutput.passthroughdevice` | AUDIOTRACK:AudioTrack (RAW) | |
| `audiooutput.ac3passthrough` | true | |
| **`audiooutput.dtspassthrough`** | **false (змінено в цій фазі)** | true перехоплював DTS у мертвий RAW-шлях |
| **`audiooutput.ac3transcode`** | **true** | робочий транскод DTS→AC3 |
| `audiooutput.eac3passthrough` | true | DD+ ліцензований ("Hash-key Support DD+") |
| `audiooutput.dtshdpassthrough` | false | DTS-HD → core-fallback→(raw вимкнено)→transcode |

Бекап оригіналу: `/data/local/tmp/guisettings.xml.bak`.

## 1. Шлях A — Kodi IEC packer ("AudioTrack (IEC) → Kodi IEC packer (recommended)")

Статус: **НЕДОСТУПНИЙ на цьому пристрої — пристрій не перелічується взагалі.**

Механіка (AESinkAUDIOTRACK.cpp, Omega):
- `EnumerateDevicesEx` створює "AudioTrack (IEC)" лише якщо `VerifySinkConfiguration(48000, STEREO, ENCODING_IEC61937)` успішний і набір streamTypes непорожний; інакше `m_hasIEC=false` і сінк не потрапляє у список.
- `VerifySinkConfiguration` будує legacy `AudioTrack(..., ENCODING_IEC61937, ...)`. Probe на пристрої: `getMinBufferSize=17320 (>0)`, але ctor → `STATE_UNINITIALIZED` (framework -38). Причина: **платформа не має формату IEC61937 (0x0d000000) ані в audio_policy (offload mixPort: AC3/E-AC3/JOC/DTS/DTS-HD/AAC*/AC4), ані в HAL** (`adev_open_output_stream: unsupport format (0x0d000000)`; у формат-мапі `mi_common_raw_get`/`utils_isPassthroughSupported` сім'ї 0xd немає).
- Runtime: `Enumerated AUDIOTRACK devices:` → лише `AudioTrack (RAW)`; у логах епохи до фікса: `encoding: 13 (IEC61937) success: false`.

Якби сінк існував: для IEC-режиму на SDK≥23 Kodi запитує AudioTrack **ENCODING_IEC61937 (tunneled)** — тобто все одно потрібна платформена підтримка IEC61937; старий шлях — Kodi-пакувальник (`CAEBitstreamPacker`: AC3 → Pc=0x01, burst 1536 B подвійною частотою PCM16 stereo) через звичайний **PCM** мікшер — біт-точність не гарантується (Float-мікшер/ресемпл/DSP-обробка/SPDIF TX channel-status = audio), тому RAW-шлях і є рекомендованим.

**Чому IEC fail'ить навіть для AC3:** не через AC3 і не через пакувальник — через відсутність IEC61937-формату на рівні HAL/policy. Це незалежно від нашого GetCaps-фікса (сім'я 0xd не входить у жодну перевіряну групу).

## 2. Шлях B — DTS Core як IEC/RAW БЕЗ decType=11

**Відповідь: НІ, на цьому пристрої легітимного шляху DTS-бітстрім-у-обхід-DSP-декодера не існує.**

Докази:
- Єдиний шлях compressed-даних у SPDIF TX — offload RAW: `HAL parser → MI_AUDIO_Write → ADEC-вхід DSP`; SPDIF TX отримує вихід декодера (bypass-режим декодера). Без `MI_AUDIO_Start(decType=11)` декодер не стартує (hashkey-ліцензія, фаза 3) → на TX нічого не надходить.
- HAL приймає DTS-open лише завдяки нашому GetCaps-фіксу, але далі все одно вимагає декодер (`mi_decoder_open` обов'язковий для raw-стрімів). "MStar HAL явно відкидає DTS незалежно від ліцензії" — НЕ знайдено: відмова лише на етапі старту декодера (ліцензія).
- PCM-варіант (Kodi IEC-пакувальник у PCM): по-перше недоступний (шлях A); по-друге — навіть технічно: IEC-берсти через Float-мікшер AudioFlinger + DSP-обробку (ресемпл/Volume) втрачають біт-точність, а SPDIF TX у PCM-режимі виставляє channel-status "audio" — ресівер не декодує non-audio з audio-каналу.
- Висновок: TX-шлях сам по собі ліцензії не вимагає — вимагає **старт DTS-декодера**, який ліцензіється.

## 3. Шлях C — AC3-transcode: аномалія знайдена і виправлена (CONFIRMED runtime)

Ланцюг умови (ActiveAE `ApplySettingsToFormat`): транскод задіюється ТІЛЬКИ коли вхідний стрім не пройшов direct-passthrough (`SupportsRaw(DTS)==false`) і `passthrough && ac3passthrough && ac3transcode`.

**Корінь аномалії:** `audiooutput.dtspassthrough=true` → `SupportsRaw(DTS_512)=true` → Kodi обирає RAW DTS → HAL open OK (наш фікс) → DSP декодер не стартує (ліцензія) → **тиша, і транскод ніколи не викликається** (транскод — fallback, а не конкурент RAW).

Тест A/B на пристрої (test_dts_51.mp4, DTS core 5.1 48 kHz):

| Прогін | dtspassthrough | Лог | Вихід |
|---|---|---|---|
| RUN1 | **true** | `Creating audio stream ... pass-through`; sink `STREAM_TYPE_DTS_512 RAW` | `AudioHAL codec type: NONE` → **тиша** |
| RUN2 | **false** | `no pass-through` → **`CAEEncoderFFmpeg::Initialize - AC3 encoder ready`** → sink `AE_FMT_RAW STREAM_TYPE_AC3` | flinger format 0x09000000 ACTIVE, **`AudioHAL codec type: AC3`** → **DD 5.1 на ресівері** |

Повторено двічі, останній прогін залишено відтворюватись. Ланцюг: FFmpeg декодує DTS **софтом на CPU** (жодного DTS-декодера DSP) → CAEEncoderFFmpeg кодує AC3 → RAW AC3 → **ліцензований** DD-шлях → SPDIF.

## 4. Фінальна матриця

| Шлях | AC3 | DTS | Використовує DTS-декодер DSP? | Потребує DTS-ліцензії? | Фізичний SPDIF? |
|---|---|---|---|---|---|
| Android IEC packer (ENCODING_IEC61937 tunneled) | ✗ (формату IEC61937 немає на платформі) | ✗ | ні | ні | ✗ недосяжно |
| Kodi IEC packer ("AudioTrack (IEC)") | ✗ (сінк не перелічується: IEC61937 verify fail) | ✗ | ні | ні | ✗ недосяжно (а якби був — PCM-шлях, біт-точність/CS під питанням) |
| Native MI_AUDIO offload (RAW) | **✓** (decType 3, ліцензований) | **✗** (decType 11 → hashkey fail) | так | **так** | ✓ для AC3 |
| **Kodi AC3 transcode (поточний робочий)** | — (це і є AC3) | **✓ DTS/DTS-HD→AC3 5.1** | **ні** (FFmpeg софт-декод + софт AC3-enc) | **ні** | **✓ підтверджено runtime** |

## 5. Рекомендації

1. **Головна**: залишити `dtspassthrough=false` + `ac3transcode=true` — DTS/DTS-HD контент грає як Dolby Digital 5.1 на ресівері (перевірено; ресівер покаже Dolby Digital).
2. Опційно: `eac3passthrough=true` (DD+ ліцензований — "Hash-key Support DD+") для E-AC3 контенту без транскоду.
3. `dtshdpassthrough`, `truehdpassthrough` — лишити false (все одно транскодується в AC3).
4. Відновити оригінал: `cp /data/local/tmp/guisettings.xml.bak .../guisettings.xml` (повертає dtspassthrough=true — поверне тишу для DTS!).

## 6. Статуси

- Шлях A недоступний через відсутність IEC61937 у HAL/policy — **CONFIRMED** (probe + policy XML + HAL disasm)
- Шлях B неможливий без decType=11 (TX живиться з декодера; іншого маршруту немає) — **CONFIRMED** (HAL flow) / відсутність окремої відмови DTS у HAL поза ліцензією — CONFIRMED
- Шлях C працює; аномалія = dtspassthrough=true перехоплював потік — **CONFIRMED** (A/B runtime)
- Kodi на пристрої 21.2 Omega, не Piers — CONFIRMED (package dumpsys)
