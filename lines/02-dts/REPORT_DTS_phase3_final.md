# Фаза 3: Повний доказовий ланцюг до DSP — DTS заблоковано ліцензійним hashkey/AUTH, не багом

Дата: 2026-08-29 · Патчів немає. Runtime: dmesg-зніми, UI-навігація, знім розділу armfw (read-only).

---

## 1. Вердикт BYPASS-експерименту (фаза 2, підтверджено)
«Цифровой выход = Пропуск (BYPASS)» вже активний (screen4.png). DD проходить, DTS — ні. **Режим SPDIF не є блокером.**

## 2. Повний kernel-ланцюг для DTS (усі ланки CONFIRMED дизасемблюванням/декомпіляцією)

```text
mik.ko:
  MI_AUDIO_Start(handle,{codecType=9})
  → FUN_000b5ea4 = _MI_AUDIO_CodecTypeMapDecoderType:
        5(DD) → 3      9(DTS) → 11 (0xb)      22(TrueHD) → 0x1b   [таблиця повна, 23 формати]
  → MApi_AUDIO_SetDecodeSystem(ADEC0, {decType=11, ...})
        utpa2k _MApi_AUDIO_SetDecodeSystem: гейки (adec!=5/-1; type!=0xff/0; g_AudioVars2[adec*0x28+0xfa]) → PASS
        → HAL_AUDIO_SetDecodeSystem: switch(decType): **0xb У СПИСКУ ПІДТРИМАНИХ** → HAL_AUDIO_SetSystem2
        → HAL_DEC_R2_Set_SHM_PARAM(0x54, adec, decType=11)   ← DSP ОТРИМАВ "DTS"
dtv_driver.ko:
  MTGADEC_MTAUD_SetDecType → _GlueDecFmtTransfer: таблиця identity (11→0xb) — без фільтрації
```

Жодна ланка Android/kernel glue НЕ відфільтровує DTS. Команда коректно досягає DSP.

## 3. Де саме ламається: запит стану декодера (infoType 0x30)

`_MI_AUDIO_GetCodecType` (інлайн у mik, 0xb3be8–0xb4378):
- для codecType ∈ {4,5} (DD) і {9,10,11} (DTS), і 0x17 — **той самий запит** `MApi_AUDIO_GetAudioInfo2(adec, 0x30)`;
- utpa2k `HAL_MAD_GetAudioInfo2` case 0x30: читає `g_AudioVars2[0x524/0x528]` (decode system per ADEC) → для 11 гілка `case 0xb` читає **DTS-специфічний SHM(0x36)** під гейтом **`g_AudioVars2[0x43d]==1`**; SHM(0x36)∉{0,1,2} або прапорець≠1 → ERROR → MApi ret≠0;
- mik: ret≠0 → **"Fail to get DTS codec type!"** → повертає NONE; для DD запит успішний (DSP у стані DD-декоду, SHM валідний).

Runtime-підтвердження (фаза 2): друк "Fail to get DTS codec type!" під час DTS-відтворення; NONE у dumpsys.

## 4. ПРИЧИНА: ліцензійний hashkey-гейт (CONFIRMED статично, підтримано runtime)

`utpa2k.ko MDrv_AUDIO_CheckHashkey()` (~50× `MDrv_AUTH_IPCheck(id)` — апаратна AUTH-перевірка по OTP/сертифікатах):
- результати → бітмаски `g_AudioVars2[0x440]/[0x444]` і per-codec прапорці:
  - **`[0x43d] = uVar10`** ← прапорець DTS-гілки (IPCheck(0x50)/IPCheck(0x73) блок);
  - `[0x43e] = uVar7` (другий кодек-прапорець).
- Дослівні рядки utpa2k:
  - **"Hash Key Check DTS Fail, no DTS license!!"**
  - "Hash Key Check DTSX Fail, no DTSX license!!"
  - "Hash-key Support DD+." / "Hash-key Support Dolby DDCO" / "Support Dolby MS11/MS12..."
  - **"UnSupportType: %x (0:support, 1:no hashkey, 2:OTP NG, 3:format unsupport)"**
- Перевірки виконуються при ініціалізації DEC DSP (викликається і з `MDrv_AUDIO_SetDecodeSystem`); результат кешується. Без reboot-у перевірити друк наживо неможливо (ring прогорів, DSP не перезавантажується).

**Висновок: DTS-декодер у DSP не стартує, бо hashkey-перевірка ліцензії DTS на цьому пристрої не проходить (OTP/сертифікатне пропіціонування). DD/DD+ пропіціоновані — тому AC3/E-AC3 працюють повністю.**

## 5. Статус тверджень

| Твердження | Рівень |
|---|---|
| Kernel/glue ланцюг для DTS коректний до команди DSP (мапінг 9→11, SetDecodeSystem приймає) | **CONFIRMED** |
| Відмова: `GetAudioInfo2(0x30)` для DTS → fail → NONE | **CONFIRMED** (runtime + disasm) |
| Гейти fail-шляху: `g_AudioVars2[0x43d]` (AUTH-прапорець) та DSP SHM(0x36) | **CONFIRMED** (disasm) |
| 0x43d встановлюється виключно `MDrv_AUDIO_CheckHashkey` з `MDrv_AUTH_IPCheck` (клієнтської емуляції немає) | **CONFIRMED** (єдиний writer у модулі) |
| Причина відмови AUTH = відсутня DTS-ліцензія на пристрої | **STRONG EVIDENCE** (дослівні рядки "no DTS license!!"; фізичний чит 0x43d потребує kernel-пам'яті) |
| DSP firmware містить DTS-декодер (блокований ліцензією, а не відсутній) | **INFERENCE** (middleware повністю підтримує DTS; armfw — компресований контейнер, розпакування не вдалось) |

## 6. Простір рішень

1. **Легітимний DTS bitstream**: тільки пропіціонування ліцензії (OEM/SoC vendor; партиції tvservice/tvcertificate/eeprom + OTP AUTH). Програмний патч нереалістичний: AUTH виконується secure-світом (HAL лінкує libteec → TEE).
2. **Практичний обходь БЕЗ патчів — транскод**: Kodi має вбудований режим **"Transcode to AC3"** (`audiooutput.ac3transcode=true`): DTS/DTS-HD → AC3 5.1, і цей бітстрім пристрій **вже вміє** пропускати на SPDIF (DD працює). Ресівер покаже Dolby Digital і відтворить 5.1. Рекомендований практичний розв'язок.
3. Що НЕ спрацює: зміна SPDIF режиму (перевірено), kernel-патчі mik/utpa2k/dtv_driver (ланцюг і так коректний), userspace IEC-пакер (не використовується цим стеком).

## 7. Артефакти фази

- `kmods/{mik,utpa2k,dtv_driver}.ko` + runtime-бази (mik 0xE23CF1F0, utpa2k 0xE383FB00, dtv 0xE262F580)
- Декомпіляції: `k_mik_decomp.c` (MI_AUDIO_Start), `k_mapfun_decomp.c` (таблиця мапінгу), `k_dtv_decomp.c`/`k_dtv_fmt.c` (glue), `k_utpa2k_decomp.c` (Set/GetDecodeSystem + HAL_MAD_GetAudioInfo2 case 0x30), `k_hashkey.c` (CheckHashkey), `k_GetCodecType_inline.asm`
- `armfw.bin` (ATDF-контейнер, 1 MB; компресований — пряме розпакування не виконано)
- Скріншот налаштування: `screen4.png` (Цифровой выход = Пропуск)
