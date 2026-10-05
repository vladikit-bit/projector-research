# Source-Level Forensic Report: Kodi 22 (Piers/Maven) `AudioTrack (IEC) → Kodi IEC packer` transport

Дата: 2026-08-29 · Метод: статичний audit вихідників (Omega гілка, master/Piers, Maven85/kodi 22-android) + GitHub commit history + runtime-артефакти попередніх фаз. Патчів і змін на пристрої немає. Probe6 (PCM-carrier IEC) — готовий, НЕ запускався (за вказівкою).

Ключовий факт передусім: **`Maven_AESinkAUDIOTRACK.cpp` і `Maven_AEPackIEC61937.cpp/.h` байт-у-байт ідентичні upstream Kodi master (Piers)**. Maven85/kodi 22-android не містить жодного аудіо-sink патчу. Усі відмінності IEC-поведінки — це еволюція upstream Kodi 21.2(Omega) → 22(master).

---

## Q1. Який точний transport реалізує Kodi 22 `AudioTrack (IEC)`?

Ланцюг (Piers/master, підтверджено у файлах):

```text
passthrough stream (AE_FMT_RAW, streamInfo = AC3/DTS/…)
  ↓ CActiveAESink::OpenSink()                       [M_ActiveAESink.cpp:906-920]
      bool passthrough = (m_requestedFormat.m_dataFormat == AE_FMT_RAW);
      m_needIecPack = NeedIECPacking();              // = m_wantsIECPassthrough обраного device
      if (m_needIecPack) {
        m_packer = make_unique<CAEBitstreamPacker>();
        m_requestedFormat.m_sampleRate    = CAEBitstreamPacker::GetOutputRate(streamInfo);
        m_requestedFormat.m_channelLayout = CAEBitstreamPacker::GetOutputChannelMap(streamInfo);
      }
  ↓ CAESinkFactory::Create(device, m_sinkFormat)
  ↓ CAESinkAUDIOTRACK::Initialize():                 [Maven_AESinkAUDIOTRACK.cpp:325-361]
      if (m_info.m_wantsIECPassthrough) {
        m_format.m_dataFormat = AE_FMT_S16LE;        // carrier = S16LE
        if (ENCODING_IEC61937 != -1)
          m_encoding = ENCODING_IEC61937;            // ← Variant A (SDK ≥ 23; на Android 11 = завжди)
        if (m_encoding == -1) {
          m_encoding = ENCODING_PCM_16BIT;           // ← Variant B (лише SDK < 23)
          "Fallback to PCM passthrough mode - this might not work!"
        }
      }
  ↓ CActiveAESink::OutputSamples():                  [M_ActiveAESink.cpp:1049-1067]
      if (m_needIecPack) {
        m_packer->Reset();
        m_packer->Pack(streamInfo, buffer, frames);  // ← IEC61937 берсти СТВОРЮЮТЬСЯ ТУТ (Kodi-side)
      }
      packBuffer = m_packer->GetBuffer();
  ↓ m_sink->AddPackets(packBuffer) → CAESinkAUDIOTRACK::AddPackets → AudioTrack.write()
```

**Відповідь на питання №2 з ТЗ (що на межі Kodi→Android):** у Variant A (наш випадок, SDK 30) через `AudioTrack.write()` йде **burst-буфер, збудований CAEBitstreamPacker** ( Kodi-pack), а AudioTrack створюється з `ENCODING_IEC61937`. Тобто Kodi сам пакує, але вимагає від Android транспортувати ці дані як IEC61937-нормативний потік (platform support). У Variant B той самий буфер пишеться як звичайний PCM16.

Критично: **берсти генеруються в ActiveAESink незалежно від варіанту A/B** — дані ідентичні, відрізняється лише label кодування в AudioTrack.

## Q1a. Структура берстів (CAEPackIEC61937, Omega==Piers==Maven)

| Формат | Burst | Header (LE bytes) | Payload |
|---|---|---|---|
| AC3 | `1536*4 = 6144 B` | `72 F8 1F 4E`, type=`0x01|(bsmod<<8)`, length=bits | AC3 frame, слова byte-swapped, zero-pad |
| DTS 512 | `512*4 = 2048 B` | `72 F8 1F 4E`, type=`0x0B`, length=size<<3 | DTS BE frame → swapped, zero-pad |
| DTS 1024/2048 | 4096/8192 B | type 0x0C/0x0D | так само |
| DTS-HD | period*4 | type 0x11|(subtype<<8) | |

Носійна смуга: AC3 6144B/32ms = 1.536 Мбіт/с; DTS-core 2048B/10.67ms = 1.536 Мбіт/с — **рівно 48 kHz stereo 16-bit carrier (1×)**. E-AC3 потребує 4×, DTS-HD/TrueHD — 192 kHz (на цьому HAL неможливо: primary/out profiles лише 48 kHz).

`PackDTS_512(size=2012)`: 2012 ≤ 2048-8 → повноцінний валідний IEC61937 DTS type-1 burst. **Відповідь на №10: так, Kodi 22 генерує завершений IEC61937 DTS Core burst, придатний для фізичного TX без жодного звернення до DTS ADEC** (це PCM-дані для платформи).

## Q2. Чому ENCODING_IEC61937 не відкривається на цьому MStar Android

Точний ланцюг відхилення (CONFIRMED runtime + policy артефакти):

1. Kodi 22 IEC sink: `AudioTrack(STREAM_MUSIC, 48000, CHANNEL_OUT_STEREO, ENCODING_IEC61937, …)` (наш runtime: `Trying to open: samplerate: 48000, channelMask: 12, encoding: 13` ×3).
2. `AudioSystem::getOutputForAttr` → `AudioPolicyManager`: **жоден output profile не декларує `AUDIO_FORMAT_IEC61937`**:
   - `audio_policy_configuration.xml`: primary = PCM16 only; offload = AC3/E-AC3/JOC/DTS/DTS-HD/AAC*/AC4 (**IEC61937 відсутній**);
   - HAL `mi_common_raw_get`/`utils_isPassthroughSupported`: сім'я 0xd не мапиться (map: 0x4/0x5/0x6→bit7, 0x9/0xa→bit5, 0xb/0xc→bit9, 0x22→bit6);
   - runtime probe: для AC3 policy знаходить offload (minBuffer 4330), для IEC61937 повертає PCM-дефолт 17320 (валідації формату немає на цьому етапі).
3. `AudioTrack` creation → `AudioFlinger::createTrack` → для compressed формату без direct/offload output → **INVALID_OPERATION (-38)** → `STATE_UNINITIALIZED` → "Unable to create AudioTrack" ×3 (retry того самого encoding) → "no sink was returned".

Тобто відхилення — **на рівні AudioPolicy/HAL capability** (немає IEC61937-профілю), а не AudioFlinger-мікшера і не Kodi-пакувальника.

## Q3. Чи може Kodi 22 використовувати PCM-carrier замість tunneled?

**Так, transport-механіка вже повністю присутня в Kodi 22**: `CAEBitstreamPacker` будує S16LE берсти, і сінк пише їх через звичайний `AudioTrack.write()`. Єдине, що відрізняє Variant B від Variant A — значення `m_encoding` (PCM_16BIT vs IEC61937). Variant B у upstream активується лише коли константа `ENCODING_IEC61937` відсутня (SDK < 23): `if (m_encoding == -1)`. На Android 11 (SDK 30) константа є → завжди Variant A. **Fallback A→B при невдалому open відсутній** ("will retry once" повторює той самий encoding 13; далі — hard error; commit історія це підтверджує — fallback-коміт e2776a3d91 (2020-01-05) додав лише enum-гейт `m_hasIEC` в enumeration, не runtime fallback).

Отже **мінімальна Kodi-side зміна** (Maven fork, один рядок умови в `CAESinkAUDIOTRACK::Initialize`): для IEC-пристрою примусово `m_encoding = ENCODING_PCM_16BIT` (carrier 48 kHz stereo з `GetOutputRate`=48000 для AC3/DTS-core). Усе інше (пакування, формат S16LE, 2.0 layout, buffer) вже в коді.

## Q4. Чи може PCM-carrier IEC фізично пройти через цей SPDIF stack?

Невідомо доти, доки не виконано empirical тест. Технічні передумови ЗА так:
- смуга: AC3/DTS-core burst = рівно 1× 48 kHz stereo16 carrier (без rate-множення);
- HAL PCM-шлях 48 kHz stereo працює (native PCM baseline чутний на ресівері);
- Kodi передає берсти як звичайний PCM — **жодного `MI_AUDIO_Start(codecType=9)`**;
- ризики: (а) DSP-обробка PCM-шляху (volume/eq/resample) має бути біт-точною; (б) SPDIF TX channel-status = "audio" (не non-audio) — частина ресіверів не декодує non-PCM з audio-CS.

Probe6 (готовий, алгоритм = точний Kodi packer) дає прямий empirical тест обох ризиків.

## Q5. DTS Core через PCM-carrier без `MI_AUDIO_Start(codecType=9)`?

Якщо Q4 = так: **так**. Дані є звичайним PCM-потоком для всього Android/HAL/DSP-шляху; жоден ADEC не інстанціюється; DTS «декодує» лише ресівер з власним DTS-декодером. Це і є транспорт повз ліцензійний гейт — легітимний, бо проєктор не декодує DTS.

## Q6. MStar direct SPDIF/IEC injection?

Статично: `MI_AOUT_SetDigitalMode` (ioctl 0xC0501111, payload {handle, attr×3, char*}) — це **селектор режиму** (AUTO/BYPASS/PCM/TRANSCODE enum-строки), не data injection. `HAL_AUDIO_SPDIF_*` (utpa2k) — mode/SCMS/delay/channel-status конфігурація TX-стану, теж не data path. Єдиний data path у HAL — `MI_AUDIO_Write` → декодерний вхід. Прямих API запису в SPDIF TX buffer не знайдено (перевірено: libmi3 imports, kallsyms [mik]/[utpa2k], strings). Це узгоджується з тим, що TX фізично живиться з DSP output stage; але PCM-дані, подані через звичайний decode-вхід при вимкненому декодуванні (PCM render mode), доходять до TX як PCM — саме це і використовує PCM-carrier варіант.

## 8. Історія: коли і чому з'явився ENCODING_IEC61937

- До 2019: єдиний IEC-транспорт = Kodi packer → PCM16 carrier (works on any Android).
- 2019-11/2020-01 (commits d5ea38bd5b, c499910cd2, f33122fbc5 "Allow RAW and IEC distinguishing", e2776a3d91 "Fallback if IEC is not existent"): розділення RAW/IEC пристроїв; e2776a3d91 додав лише enumeration-gate `m_hasIEC` (НЕ runtime fallback).
- Причина переходу на ENCODING_IEC61937 (SDK≥23): carriers >48 kHz (E-AC3 4×, TrueHD/DTS-HD 192 kHz) неможливі як PCM-carrier через Android mixer обмеження; tunneled IEC61937 покладався на платформену підтримку (Shields/більшість TV їх мають).
- Наслідок для нашої платформи: Android 11 + MStar HAL без IEC61937 → Variant A мертвий, Variant B — dead code.

## 9. Тристороннє порівняння

| | Omega 21.2 | upstream master (Piers) | Maven85 22-android |
|---|---|---|---|
| AESinkAUDIOTRACK.cpp | 45704 B | 45913 B | **== master (0 diff)** |
| AEPackIEC61937.cpp | 7018 B | 7018 B | **== master** |
| IEC sink enumeration | лише якщо VerifySink(IEC61937) ok → на MStar не з'являється | завжди (константа існує) → з'являється | == master |
| IEC open | Variant A | Variant A | == master |
| Maven audio-патчі | — | — | **відсутні** |

## 10. Статуси (лише нове)

- Kodi 22 IEC transport = Kodi-pack bursts + AudioTrack(ENCODING_IEC61937); packing Kodi-side, transport вимагає platform IEC61937 — **CONFIRMED (source)**
- Variant B (PCM16 carrier) існує в коді, обмежений `SDK < 23`; runtime A→B fallback відсутній — **CONFIRMED (source)**
- Для AC3/DTS-core @48k PCM16-carrier bandwidth = рівно 1× carrier — **CONFIRMED (arithmetic + GetOutputRate)**
- MStar: IEC61937 відхилено на AudioPolicy рівні (немає профілю) + HAL format map — **CONFIRMED (policy dump + HAL disasm + probe)**
- `MI_AOUT_SetDigitalMode` = mode selector, не injection — **CONFIRMED (libmi3 disasm)**
- PCM-carrier transport физично працює через MStar DSP/SPDIF і ресівер — **NOT YET TESTED (Probe6 готовий)**
- DTS Core через PCM-carrier без decType=11 — **NOT YET TESTED (залежить від попереднього)**
