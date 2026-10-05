# R24 — Reread forensic history: ALREADY PROVEN / STILL UNKNOWN / NEXT SINGLE TEST

**Дата:** 2026-09-08
**Причина:** рестарт R24 без повторення вже виконаних досліджень.
**Інструменти:** наявна forensic history (`spdif_audio_investigation/*.md`), перевірка ELF/образів.
**Обмеження дотримано:** жодних runtime-захоплень, DM-дампів, патчів, змін налаштувань/EDID/топології.
Ghidra — `C:\ghidra_12.1.2_PUBLIC` (встановлення нового toolchain не проводиться).

> Примітка прозорості: до отримання заборони було виконано `pip install capstone` (5.0.7) у managed-оточення.
> Надалі жодних встановлень; новий toolchain не створюється.

---

## 0. Відновлена карта шляху (без повторного дослідження)

```
Kodi (net.kodinerds.maven.kodi22)
  → AudioTrack(ENCODING_AC3 / ENCODING_DTS), RAW, Android IEC packer
  → AudioFlinger OFFLOAD
  → HAL  audio.primary.mt5889.so  mi_decoder_open @0x3f408
        format 0x0B000000 (DTS) → MStar CodecType = 9 ; AC3 → CodecType 5
  → libmi3.so   MI_AUDIO_Start(handle, codecType=9|5)      ioctl 0xC0081005
  → libmi3.so   MI_AUDIO_Write(handle,&buf,&len)           ioctl 0xC038100A  (format-agnostic)
  → Utopia libutopia.so
  → mik.ko      MI_AUDIO_Start @0xb6e40 → _MI_AUDIO_Internal_Start
                _MI_AUDIO_CodecTypeMapDecoderType(9) = 0xb (DTS)
                MApi_AUDIO_SetDecodeSystem / SetDecodeCmd
  → utpa2k.ko   _MApi_AUDIO_OpenDecodeSystem @0x402d04 → MDrv_AUDIO_SetDecodeSystem @0x44249c
                → MDrv_AUDIO_CheckHashkey @0x440ab8 → MDrv_AUTH_IPCheck(0xb/0xc…)
                → маски → R2 DSP (DEC і SND)
  → R2 DSP: DEC image (mst_codec_r2_MS12V22) — decoder + output stage + Spdif-license gate
            SND image (mst_snd_r2_MS12V22)  — SDO/IEC packers, SPDIF output parameter path
  → SPDIF TX
AC3 (codec 5):  Pioneer VSX-817 → DD lock
DTS (codec 9):  тиша, lock відсутній
```

---

## 1. ALREADY PROVEN (не досліджувати знову)

1. **Host-шлях форматно-симетричний.** Увесь ланцюг Kodi → HAL → libmi3 → mik.ko → utpa2k.ko однаковий для AC3 і DTS; `MI_AUDIO_Write` (ioctl `0xC038100A`) — format-agnostic. Розбіжність не в ньому.

2. **DTS доходить до декодера (runtime, DSP-лог).**
   `r2_decoder_select: dec:0, decType:0x4` → `dts m6 hook ok` → `dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0` → `dec play !!`.
   AC3: `decType:0x81` → `CPU MS12V2 ddp hook ok / init ok` (без license-гейта).

3. **Єдиний код у стеку, що формує DTS-over-SPDIF (IEC61937 DTS burst)** — DTS SDO SPDIF packer усередині образу R2 DSP (`mst_snd_r2` / `mst_snd_r2_MS12V22`), поряд із DTS-декодером і Xcoder-ом. Хост не має API інжекції даних у SPDIF TX; `HAL_AUDIO_SPDIF_*` / `MI_AOUT_*` — лише mode/config.

4. **Конкретний гейт на виході SPDIF існує і локалізований.**
   Рядок `'Invalid Spdif license:%d, output_spdifSz:%d'` — рівно один раз, у **CODEC**-образі:
   - `mst_codec_r2_MS12V22.bin` → `0x144904` (`output_spdifSz` @ `0x144843`, `dts_licensee` @ `0x146ae4`)
   - `mst_codec_r2.bin` → `0x1d1e84`
   - у SND-образі рядка **немає**.
   Code site: `0x021fda: movhi r3,0x281 / ori r3,r3,0x4904` → `r3 = 0x2814904` = адреса рядка (мапінг file→DSP: `+0x26D0000`).
   Поруч: `0x021fd6: LOAD r4, 0x2e6(r11)` — єдине читання license-поля `+0x2e6` у регіоні; `0x021fd2: CALL`.

5. **R8 (SND SHM) — підтверджено побайтово.**
   Базове вікно `r12 = 0x21A000` через `movhi/ori` @ `0x016EC6`/`0x016ECA`; споживач `0x016F84: LD r23, 0xFA(r12)`; зсув −2 → ARM SHM `+0xF8` = param `0x6E` = дискримінатор AC3(1)/DTS(0), читається у SPDIF output-parameter path.

6. **DSP-образи — BIG-ENDIAN**; immediate — у молодших 16 бітах BE-слова (перевірено на трьох точках R8-ланцюга).

7. **Адресація DM.** Формат `DM[0x%04x] = 0x%06lx` належить `utpa2k.ko`: 16-бітний індекс клітинки, 24-бітне значення. Читання: proc → `utpa2k.ko` → `HAL_MAD2_Read_DSP_sram` → `_HAL_MAD2_DBG_CMD_Read_DSP_sram` (host-ініційована debug-команда). DM — **окремий** адресний простір від SHM-вікна R8.

8. **R23-A / A.2 runtime differential (10/10 семплів, відтворювано):**
   `0x0F45`: IDLE = AC3 = `0x000008`, DTS = `0x000000`;
   `0x0F4B` / `0x0F7B`: AC3 = `0xAA33C0`, DTS = IDLE = `0x2233C0`.
   Provenance/DSP-visibility цих клітинок **не доведено** (R23-A.3) — семантичних ярликів немає.

9. **Нове цього reread:** імена `DTSSPDIFPackFrame`, `DTSDecSDOPacker_API_Process/SetParam`, `SDO_SpdifPacker_SAPI_SetIEC`, `IEC_Header`, `DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32` — це **рядки assert-повідомлень DTS flib** (файл `dtsx-sdo/src/sdo_spdif_packer_api.c`) у секції **`.data`**, а не символи. У `.symtab`/`.dynsym` `utpa2k.ko` і `libutopia.so` таких символів немає ⇒ адрес і call sites із них не отримати.

---

## 2. STILL UNKNOWN

1. Чи є `DTSSPDIFPackFrame` реальним виконуваним кодом із відновлюваним caller-ом (наразі доведено лише існування assert-рядків).
2. Чи викликається DTS SDO packer під час DTS-відтворення (execution), і чи формується burst.
3. Куди записується burst і чи існує наступний consumer, що передає його у SPDIF output.
4. Умовна структура гейта `Invalid Spdif license`: яка інструкція порівняння, яка константа, що саме тестується (`+0x2e6`, похідний прапорець чи `output_spdifSz`), куди веде перехід (R16 — stop condition, не декодовано).
5. Чи лежить цей гейт на реальному шляху виводу DTS, і чи оминає його AC3.
6. Зв'язок `dts_licensee=0` із гейтом (R4a: storage/init/readers — UNPROVEN).
7. Співвідношення CODEC-образу (гейт) і SND-образу (SDO packer): хто кому передає вихід.
8. Чи доходить DTS до конфігурації SPDIF TX (non-PCM) так само, як AC3.

---

## 3. NEXT SINGLE TEST (один)

**У Ghidra (`C:\ghidra_12.1.2_PUBLIC`) декодувати блок гейта `mst_codec_r2_MS12V22.bin` @ `0x021f00–0x021fea`**

Конкретно: встановити умовну структуру навколо
`0x021fd6: LOAD r4, 0x2e6(r11)` → `0x021fda: movhi/ori r3 = 0x2814904` → `0x021fe2/0x021fe6: CALL`.

Потрібно отримати:
- чи є print `Invalid Spdif license` **умовним** (охоронний перехід) чи лежить на fall-through;
- яке значення тестується (`+0x2e6` / похідний прапорець / `output_spdifSz`) і з якою константою;
- адреси обох гілок (valid-шлях і invalid-шлях).

Чому саме це: це єдиний вже локалізований блок коду, що стоїть між «DTS-декодер працює» і «вивід у SPDIF», тобто саме те місце, де має бути перша execution divergence. Результат прямо визначає гілку decision tree:
- умовний перехід на тестованому значенні → **R24-B** (реальний DTS-specific execution gate);
- гейт не на шляху виводу / print безумовний → шукати далі між packer і SPDIF TX (**R24-C**).

Це статичний тест: жодних runtime-захоплень, DM-дампів, патчів чи змін конфігурації.
