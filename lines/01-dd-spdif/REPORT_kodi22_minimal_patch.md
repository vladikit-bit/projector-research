# Minimal Kodi 22 patch: IEC-carrier transport — source diff, наслідки, feasibility

Дата: 2026-08-29 · Джерело істини: `Maven85/kodi` гілка `22-android` (байт-ідентична upstream master для `AESinkAUDIOTRACK.cpp` і `AEPackIEC61937.cpp`; Maven-патчів у аудіо-sink стеку немає). Binary RE libkodi22.so — зупинено за вказівкою, не використано.

---

## 1. Точне місце зміни

`xbmc/cores/AudioEngine/Sinks/AESinkAUDIOTRACK.cpp`, метод `CAESinkAUDIOTRACK::Initialize()`, блок passthrough (рядки 331–359 у fetched-копії Maven 22-android):

```cpp
    if (m_info.m_wantsIECPassthrough)
    {
      m_format.m_dataFormat     = AE_FMT_S16LE;                      // ← carrier: Kodi-pack bursts як S16LE
      if (DTSHD || DTSHD_MA || TRUEHD)
        m_sink_sampleRate = 192000;                                  // ← не для AC3/DTS-core

      // new Android N format
      if (CJNIAudioFormat::ENCODING_IEC61937 != -1)
      {
        m_encoding = CJNIAudioFormat::ENCODING_IEC61937;             // ← ★ЄДИНА ЗМІНА
        // this will be sent tunneled, therefore the IEC path needs e.g.
        // 4 * m_format.m_streamInfo.m_sampleRate
        if (EAC3) m_sink_sampleRate = m_format.m_sampleRate;
      }

      // we are running on an old android version
      // that does neither know AC3, DTS or whatever
      // we will fallback to 16BIT passthrough
      if (m_encoding == -1)
      {
        m_format.m_channelLayout = AE_CH_LAYOUT_2_0;
        m_format.m_sampleRate     = m_sink_sampleRate;
        m_encoding = CJNIAudioFormat::ENCODING_PCM_16BIT;
        CLog::Log(LOGDEBUG, "Fallback to PCM passthrough mode - this might not work!");
      }
    }
```

## 2. Minimal diff (варіант 1 — один рядок)

```diff
--- a/xbmc/cores/AudioEngine/Sinks/AESinkAUDIOTRACK.cpp (Maven85/kodi 22-android)
+++ b/xbmc/cores/AudioEngine/Sinks/AESinkAUDIOTRACK.cpp
@@
       // new Android N format
       if (CJNIAudioFormat::ENCODING_IEC61937 != -1)
       {
-        m_encoding = CJNIAudioFormat::ENCODING_IEC61937;
+        m_encoding = CJNIAudioFormat::ENCODING_PCM_16BIT;
         // this will be sent tunneled, therefore the IEC path needs e.g.
         // 4 * m_format.m_streamInfo.m_sampleRate
         if (m_format.m_streamInfo.m_type == CAEStreamInfo::STREAM_TYPE_EAC3)
           m_sink_sampleRate = m_format.m_sampleRate;
       }
```

Другий `if (m_encoding == -1)` після цього стає мертвим (m_encoding = 2 ≠ −1) — його можна лишити або прибрати разом із першим `if` (варіант 2 — «чистий»):

```diff
-      // new Android N format
-      if (CJNIAudioFormat::ENCODING_IEC61937 != -1)
-      {
-        m_encoding = CJNIAudioFormat::ENCODING_IEC61937;
-        // this will be sent tunneled, therefore the IEC path needs e.g.
-        // 4 * m_format.m_streamInfo.m_sampleRate
-        if (m_format.m_streamInfo.m_type == CAEStreamInfo::STREAM_TYPE_EAC3)
-          m_sink_sampleRate = m_format.m_sampleRate;
-      }
-
-      // we are running on an old android version
-      // that does neither know AC3, DTS or whatever
-      // we will fallback to 16BIT passthrough
-      if (m_encoding == -1)
-      {
-        m_format.m_channelLayout = AE_CH_LAYOUT_2_0;
-        m_format.m_sampleRate     = m_sink_sampleRate;
-        m_encoding = CJNIAudioFormat::ENCODING_PCM_16BIT;
-        CLog::Log(LOGDEBUG, "Fallback to PCM passthrough mode - this might not work!");
-      }
+      // transport Kodi-packed IEC61937 bursts over an ordinary PCM16 carrier
+      m_encoding = CJNIAudioFormat::ENCODING_PCM_16BIT;
```

## 3. Source-верифікація наслідків (що змінюється / що ні)

### Бурсти НЕ змінюються
`CActiveAESink::OpenSink` (M_ActiveAESink.cpp:906–920): при `passthrough` і `NeedIECPacking()` (=`m_wantsIECPassthrough` обраного device) створюється `CAEBitstreamPacker`; `OutputSamples` (:1049–1067) викликає `m_packer->Pack(streamInfo, …)` → dispatch за stream type → `CAEPackIEC61937::PackAC3 / PackDTS_512…` → `packBuffer = m_packer->GetBuffer()`. Жоден з цих кроків не читає `m_encoding`. Бурсти ідентичні в A/B.

### Дані в `AudioTrack.write()` байт-ідентичні в A/B
`CAESinkAUDIOTRACK::AudioTrackWrite` (рядки 164–190):
- Variant A (IEC61937): копіює буфер у `m_shortbuf` → `write(short[], … WRITE_BLOCKING)`;
- Variant B (PCM_16BIT): копіює той самий буфер у `m_charbuf` → `write(byte[], … WRITE_BLOCKING)`.
Обидва шляхи — це memcpy тих самих байтів. Змінюється **тільки label AudioTrack**, який визначає, як framework класифікує потік (IEC61937-compressed → потрібен IEC-capable output; PCM16 → звичайний primary mixer path).

### AudioTrack параметри після патчу (AC3/DTS-core 48 kHz)
- encoding: `ENCODING_PCM_16BIT` (2)
- sampleRate: `m_sink_sampleRate` = `CAEBitstreamPacker::GetOutputRate` = **48000** для AC3/DTS_512/1024/2048/DTS-HD-core (rate = info.m_sampleRate, без множення)
- channelMask: `CHANNEL_OUT_STEREO` (GetOutputChannelMap → 2.0)
- dataFormat: S16LE → framework бачить звичайний PCM → **primary mixer path** (тотож шлях, який фізично дає звук на ресівері при вимкненому passthrough)

### Побічні ефекти патчу (повні)
- Рядок 389 `if (m_encoding == ENCODING_IEC61937)` (спец-випадок DTSHD/TrueHD 192 kHz) стає інертним. Для AC3/DTS-core — несуттєво.
- E-AC3 через IEC-carrier потребує 4× (192 kHz) carrier → AudioFlinger заресемплить 192k→48k → берсти зіпсовані. **Тому в kodi22 слід вимкнути `eac3passthrough` при IEC-тестах** (та DTS-HD/TrueHD — так само).
- Enumeration не змінюється (Patch торкається лише Initialize; Piers завжди перелічує IEC-сінк).

## 4. Changelog-контекст (upstream git history, AESinkAUDIOTRACK.cpp)

- 2019-11/2020-01: розділення RAW/IEC (f33122fbc5 "Allow RAW and IEC distinguishing", c499910cd2, d5ea38bd5b) і enum-gate m_hasIEC (e2776a3d91 "Fallback if IEC is not existent" — це enumeration-gate, **не** runtime fallback).
- ENCODING_IEC61937 обрано як "new Android N format" (tunneled; коментар у коді: carrier 4×) — платформи без IEC61937 профілю (наш MStar) отримують Variant A назавжди; legacy PCM16-galузь залишається тільки для SDK<23 (`m_encoding == -1`).
- Підтверджено: runtime fallback IEC61937-fail → PCM16 у коді відсутній ("will retry once" повторює encoding 13 ×2 → hard error — наш runtime лог збігається).

## 5. Feasibility мінімального тестового білда `net.kodinerds.maven.kodi22`

Можливо. Стандартний upstream Android-білд (docs/README.Android.md, master):
- Хости: Linux/WSL2 (або Docker). Залежності хоста: build-essential, default-jdk 17+, тощо.
- Android SDK (останній) + **NDK r28c**; Gradle 8+ (JDK 17+).
- Кроки: (1) `tools/depends` — крос-тулчейн + залежності для armeabi-v7a (таргет Kodi збірки Maven — arm 32-bit: встановлений lib/arm); (2) CMake build Kodi → `libkodi.so`; (3) Gradle збірка APK (-sign debug keystore).
- Час: depends ≈1–2 год, Kodi ≈2–5 год; диск 40–80 GB. Зміна = 1 файл/1 рядок (див. §2, варіант 2).
- Деплой: (а) легальний чистий — `adb uninstall net.kodinerds.maven.kodi22` (ВАЖЛИВО: зітре його налаштування/дані — зробити backup `guisettings.xml` та elementum-даних), встановити наш підписаний APK, повернути 4 аудіо-налаштування; (б) root-варіант без uninstall — замінити extracted `/data/app/…/lib/arm/libkodi.so` на зібраний (root; оригінал збережено; до першого reboot/PM-rescan). Для тест-сесії достатньо (б).
- Застереження: Kodinerds 22.0-BETA1 зібраний їхнім пайплайном; наш білд з Maven85 HEAD може відрізнятися іншими змінами між BETA1 і HEAD — для transport-тесту прийнятно, але фіксувати версію source (commit hash) на момент білда.

## 6. План тесту після білда (контроль AC3 → DTS)

1. Kodi C (patched): passthrough=true, passthroughdevice=AudioTrack (IEC), ac3passthrough=true, **eac3/dtshd/truehd passthrough=false**, dtspassthrough — для AC3-контролю будь-яке.
2. Native `test_ac3_51.mp4` → лог мусить показати `method: IEC (PT)` + `Trying to open: … encoding: 2` + успішне створення → flinger: **primary MIXER PCM16, AudioHAL codec type: NONE** (жодного offload/ADEC).
3. Фізично: ресівер = Dolby Digital? (Авто-детект берстів з audio-CS: залежить від ресівера.)
4. Якщо DD → DTS `test_dts_51.mp4` (dtspassthrough=true): очікуємо `method: IEC (PT)` + DTS-берсти як PCM16, **dmesg без `Codec:9`/`MI_AUDIO_Start(codecType=9)`** → ресівер = DTS → гіпотеза доведена.
5. Якщо ні (шум/PCM/тиша) — фіксувати де саме: біт-точність DSP PCM-шляху vs CS-біт vs ресівер; тоді рішення про подальші кроки (наприклад, перевірка іншого ресівера/вхідного CS-режиму).

## 7. Статуси

- Мінімальний diff сформульований точно (1 рядок семантики; варіант 2 — блок) — CONFIRMED по source
- Бурсти/дані AudioTrack.write байт-ідентичні в A/B — CONFIRMED по source (AudioTrackWrite branches)
- PCM16-carrier транспорт існує в Kodi 22 як SDK<23 legacy; активний шлях на Android 11 — Variant A — CONFIRMED
- Побічка: E-AC3/DTS-HD/TrueHD через PCM-carrier на 48k-промені неможливі (carrier 4×/…) — CONFIRMED arithmetic
- Feasibility білда — CONFIRMED (стандартний NDK-r28c пайплайн; без APK-binary hacking)
- Фізичний результат IEC-carrier — NOT YET TESTED (потрібен білд)
