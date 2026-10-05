# Звіт Muse #2 (2026-09-29) — +0x4F8 виявився НЕ ліцензією, а кодеком для SPDIF TX

## Ключове

**Хто читає `AudioVars2+0x4F8`:** тільки `HAL_AUDIO_SPDIF_ApplySetting` (0x4467CC).
Слово йде в програмування передавача: `AbsWriteByte(0x112E96)`, `SPDIF_Tx_SetNonPCM`,
`SND_R2_SetSHM_PARAM`, `DigitalTx_ApplySetting`, `HDMI_ARC_SetNonPCM`, `DEC_R2_GetSHM_PARAM`.

**Хто пише:** `MDrv_AUDIO_SYSTEM_Control` (0x412138) і kernel `HAL_MAD_SpecifyDigitalOutputCodec`
(0x4662BC) — обидва доступні з userspace.

**У `SET_IPAUTH_GROUP` (0x424BD8) — НІЧОГО.** Жодного читання `+0x4F8` ані в ньому, ані в
викликачах, ані в `CheckHashkey`.

```
Дві ПАРАЛЕЛЬНІ РЕЙКИ:
 (а) ліцензія: IPCheck → CheckHashkey → AV+0x440 → SE → DSP-гейт 0xB000001E
 (б) кодек  : Specify/SYSTEM_Control → AV+0x4F8 → SPDIF TX (що саме емітити)
```

## Чому це переписує модель

Ми весь час писали в рейку (а) — прапорці. А DTS **декодується** (`format: DTS, PLAY`).
Якщо рейка (б) каже передавачу «еміти PCM», то SPDIF TX робить даунмікс — і Pioneer
не отримує бітstream, хоча декодер працює.

**Гіпотеза, яка пояснює все:** DTS-декодер працює, але `+0x4F8` не каже передавачу
пропускати DTS як non-PCM. Тому й немає passthrough — і тому жоден запис у рейку (а)
нічого не давав.

**Це дає конкретний тест:** прочитати `+0x4F8` під час AC3-PLAY (де передавач в non-PCM)
і під час DTS-PLAY. Якщо в AC3 там «кодек DD», а в DTS — «PCM» при працюючому декодері,
то ми бачимо причину безпосередньо, і залишається лише записати правильне значення.

## Бонус Muse

DSP-образ лежить **усередині `utpa2k_stock.ko`** — `mst_codec_r2_MS12V22` @ `.data+0x407F0C`,
побайтово ідентичний `dec_full.bin` (sha256 530bfa68…). `DspLoadCode` вантажить його звідти.
Тобто фірмвара декодера в нас **вже є локально** — окремий образ не потрібен для аналізу.

## Завдання 2 (виправлення му́зя)

Перша розшифровка PLT була частково невірною (12 Б замість 16 Б):
* `bl 0x8c7120` = **`MDrv_ADVSOUND_SubProcessEnable`** (не memcpy)
* `bl 0x8c7620` = **`_MApi_AUDIO_WritePreInitTable`**
* `bl 0x8c6310` = **`UtopiaOpen`**

## Завдання 3

V083-`utpa2k.ko` заблокований розміром (потрібен повний 5-ГБ super; F: 1.9 ГБ).
Muse пропонує хірургію зі звільненням ~4 ГБ: `super.bin` 827 М + `_decrypted.bin` 1.9 Г +
`update.img` 1.9 Г — усе regenerable.

