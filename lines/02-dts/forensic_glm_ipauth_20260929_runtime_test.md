# Спроба прочитати живий AudioVars2 / +0x4F8 (2026-09-29)

## Два реальні баги у власних інструментах (зафіксовано, бо повторюються)

1. **Адреси вище 0x7FFFFFFF** ламають `pread` — потрібен `-D_FILE_OFFSET_BITS=64` **і**
   знаковий тип `long` у сигнатурі. Через це `av2`/`peek` мовчки повертали **0 замість
   помилки**, і я двічі витлумачив це як «ланцюг мертвий». `memread` зібраний правильно,
   тому він читав — саме це навело на хибний висновок.
2. **ASLR** щоразу рухає базу `libapiAUDIO.so`, тому зафіксовані адреси безперервні.

## Що вдалося

```
libapiAUDIO.so r-xp base          0xB549F000 (cpu_audio, pid 490) / 0xAE6EA000 (midaemon, 193)
GOT+0xC0C  ->  gvars = 0xB5D4AB1C / 0xAEF95B1C      (стабільно, отже слот правильний)
*gvars      ->  0x00000000                            (під час AC3-відтворення!)
g_DSPMadBaseBufferAdr (base+0x8ABC00) = 24 нульових байтів
```

## Що заблоковано

`*gvars == 0` — і це **не** «аудіо-система не ініціалізована»: відтворення AC3 ішло, а
значення все одно нульове. Отже або GOT+0xC0C — не `g_AudioVars2`, або структура
init'иться не через цей покажчик.

**Найімовірніше:** покажчик `0x...5B1C` стабільний у різних процесах, тобто слот валідний, але
він указує на **інший** глобал, який просто нульовий. Аудіо-стан, за гіпотезою му́зя,
має бути **у kernel** (спільна пам'ять), а userspace-глобал — це клієнтське дзеркало, яке
не ініціалізується.

**Отже конкретне питання до му́зя** (його релокаційні дані точніші за мої здогадки):
точний GOT-слот і символ для `g_AudioVars2` **у build'і, який реально мапиться в cpu_audio**,
і — чи взагалі `g_AudioVars2` у libapiAUDIO — це kernel-структура, до якої є лише бекенд, чи
це окремий userspace-дзеркало.

Без цієї відповіді зріз `+0x4F8` під AC3 проти DTS зробити не вдасться.

## Важливе: метод вимірювання НЕ зламаний

`memread` працює надійно (з `-D_FILE_OFFSET_BITS=64`), npcm-датчик працює, Kodi-RPC
працює. Зламана лише **резолвція адреси MAD-стану**, і це питання одного символу, а не
принципова стіна.


## Muse's answer: chain confirmed, mirror unpopulated — read impossible, write possible

1. GOT link-VA `0xCDEF8` → `R_ARM_GLOB_DAT → g_AudioVars2` ✓ (my slot was right)
2. `g_AudioVars2` is defined **three** times: `libapiAUDIO`@0x8ABB1C, `libutopia`@0x1278654,
   `utpa2k`@0xEC7B8 (kernel). No `DF_SYMBOLIC` → **pre-emption**: first loaded wins.
3. **The 30-second check I could have done myself:**
   my measured `gvars = 0xB5D4AB1C`
   `bias_apiAUDIO (0xB549F000) + 0x8ABB1C = 0xB5D4AB1C`   ← **exact match**

   **→ libapiAUDIO's own copy won. My chain was correct all along.**

4. `*gvars == 0` therefore means: **the userspace mirror is simply never populated in this
   system.** The audio state exists only in the kernel instance (`utpa2k` @0xEC7B8), and
   `/proc/PID/mem` cannot reach kernel memory. So the `+0x4F8` **read is not achievable from
   userspace.** (The 24-byte `g_DSPMadBaseBufferAdr` holds the kernel pointer `0xFE4780AD`, and
   the base is 0x90 further in — also unreachable.)

### But the write path is open

Muse's report #2 already established the writers of `+0x4F8`, and both are userspace-reachable:

```
MDrv_AUDIO_SYSTEM_Control        0x412138  ← викликається з _MApi_AUDIO_SYSTEM_Control (MApi-команда)
kernel HAL_MAD_SpecifyDigitalOutputCodec 0x4662BC
```

`MDrv_AUDIO_SYSTEM_Control` takes the codec in `r5` and writes it into the **kernel's**
AudioVars2+0x4F8 (gate `r5 != 7`). The SPDIF TX then emits that codec.

**So we do not need to read `+0x4F8` — we need to WRITE the DTS codec value through the
driver's own entry point.** The remaining unknown is only the enum value for DTS (the gate
shows 7 is excluded, so the enum is small), and that can be taken from the codec enum in the
driver or inferred from what the system itself writes during AC3 passthrough.

