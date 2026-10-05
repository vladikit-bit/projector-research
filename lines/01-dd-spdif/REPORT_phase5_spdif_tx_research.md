# Phase 5: Чи існує compressed → SPDIF TX path без ADEC DTS decoder?

Дата: 2026-08-30 · Метод: source-level аналіз наявних артефактів (utpa2k.ko, mik.ko, dtv_driver.ko, libutopia.so, audio.primary.mt5889.so disassembly) + runtime логи трьох підтверджених станів. Нічого не змінено на пристрої.

---

## Стартовий стан (після cleanup)

```text
audio.primary.mt5889.so: md5 8c11c348… = ОРИГІНАЛ ✓
libmi3.so:               md5 d71a330f… = v2 патч (caps bit0) — залишено
policy:                  IEC61937 профіль — залишено
HAL:                     running ✓
```

---

## 1. Повний реєстр SPDIF/IEC/non-PCM функцій (CONFIRMED)

### utpa2k.ko (kernel, Utopia2K — native/DTV audio stack)

| Функція | Розмір | Призначення |
|---|---|---|
| `HAL_AUDIO_SPDIF_Tx_SetNonPCM` | 268 | TX → non-PCM режим |
| `HAL_AUDIO_SPDIF_SetMode` | 592 | TX mode: AUTO/BYPASS/PCM/TRANSCODE |
| `HAL_AUDIO_SPDIF_AutoMode` | 1580 | AUTO: TX перемикається за детекцією контенту |
| `HAL_AUDIO_SPDIF_BypassMode` | 1476 | BYPASS: TX пропускає compressed |
| `HAL_AUDIO_SPDIF_TranscodeMode` | 2232 | TRANSCODE: decode→re-encode→TX |
| `HAL_AUDIO_SPDIF_PcmMode` | 288 | PCM: TX = звичайний PCM |
| `HAL_AUDIO_SPDIF_SetSCMS` | 300 | Serial Copy Management System |
| `HAL_AUDIO_SPDIF_ChannelStatus_CTRL` | 832 | Channel status біти (audio/non-audio) |
| `MApi_AUDIO_SPDIF_SetMode` | 264 | wrapper → HAL |
| `MApi_AUDIO_SPDIF_HWEN` | 264 | Hardware enable |
| `MDrv_AUDIO_SPDIF_SetMode` | 168 | driver → HAL |
| `MDrv_AUDIO_SPDIF_ByPassChannel` | 84 | bypass channel routing |
| `HAL_MAD_Monitor_DDPlus_SPDIF_Rate` | 296 | моніторинг DD+ SPDIF rate |

### mik.ko (kernel, MI3 layer — Android HAL interface)

| Функція | Розмір | Призначення |
|---|---|---|
| `MI_AOUT_SetDigitalMode` | 600 | Digital output mode (ioctl 0xC0501111) |
| `MI_AOUT_GetDigitalMode` | 432 | Читання поточного режиму |
| `_MI_AOUT_SetDigitalChannelStatus` | 144 | Channel status біти |
| `MI_AOUT_SetMute` | 664 | Mute control |

### dtv_driver.ko (kernel, MTAUDDEC — native decoder driver)

| Функція | Розмір | Призначення |
|---|---|---|
| `_MiIecConfig` | 96 | IEC конфігурація |
| `MTGADEC_MTAUD_SetIecConfig` | 168 | IEC config від декодера |
| `MTGADEC_MTAUD_SetSPDIFEnable` | 84 | SPDIF enable/disable |
| `MTGADEC_MTAUD_SetIecChannel` | 8 | IEC канал |
| `MTGADEC_MTAUD_SetIecCopyright` | 64 | Copyright біт |

## 2. Хто викликає SPDIF TX функції — caller chain

### У робочому Android AC3 → DD шляху:

**ЖОДЕН з цих викликів НЕ з'являється в logcat/dmesg.**

Це означає: SPDIF TX non-PCM стан встановлюється **всередині DSP firmware** автоматично при запуску AC3 decode-програми (`MI_AUDIO_Start(Codec:5)`). Kernel/userspace не надсилає окремої команди "увімкни SPDIF non-PCM" — DSP firmware сам робить це при ініціалізації decode-програми з типом compressed codec.

### Хто викликає у native/DTV шляху:

`MApi_AUDIO_SPDIF_SetMode` та `HAL_AUDIO_SPDIF_AutoMode/BypassMode` викликаються з native/DTV audio стеку (MSrv_MM_Player, MSrv_DTV_Player тощо) при зміні TV settings або при виявленні формату. Ці виклики **не доступні з Android AudioTrack/HAL** — вони йдуть через окремий MApi/MMF interface, який Android audio framework не використовує.

### Висновок по caller chain:

- Android HAL → `MI_AUDIO_Start(codecType)` → DSP firmware → **SPDIF mode встановлюється автоматично** (firmware-internal)
- Native DTV → `MApi_AUDIO_SPDIF_SetMode(mode)` → явне керування → доступно тільки з native стеку
- **Немає мосту між цими двома системами** на рівні Android audio framework

## 3. `MI_AUDIO_Start(Codec:5)` — достатня умова для SPDIF non-PCM?

**Ні.** Доказ:

| Тест | `MI_AUDIO_Start(Codec:5)` | `AudioHAL codec type` | `mi_common_raw_write` | Ресівер |
|---|---|---|---|---|
| Native AC3 (working) | Codec:5, eRet=0x0 ✓ | AC3 | без timeout ✓ | **DD** |
| DTS→transcode→AC3 | Codec:5, eRet=0x0 ✓ | AC3 | без timeout ✓ | **DD** |
| Kodi IEC61937 (AC3 в IEC framing) | Codec:5, eRet=0x0 ✓ | AC3 | без timeout ✓ | **тиша** |
| Kodi IEC61937 (DTS в IEC framing) | Codec:5, eRet=0x0 ✓ | NONE | timeout ⚠ | **тиша** |

`Codec:5` — необхідна, але недостатня умова. Потрібно щоб **DSP decode program успішно синхронізувався та декодував вхід**. AC3 elementary frames → декодується → bypass вихід = компресований вхід → SPDIF non-PCM → DD. IEC61937 framing → DSP parser не знаходить AC3 sync (бачить Pa/Pb/Pc/Pd замість 0x0B77) → decode fail → bypass вихід = пусто → тиша.

## 4. Чи `mi_common_raw_write` — тільки ADEC input?

**Так (CONFIRMED).** З декомпіляції `mi_common_raw_write` (ghidra_hal_funcs2.c):

```c
mi_common_raw_write(stream, data, size):
    → utils_common_out_open(stream)          // ініціалізація ADEC входу
    → loop:
        MI_AUDIO_GetAttr(handle, 0x104, 0, &avail)   // ← ADEC input buffer avail
        MI_AUDIO_Write(handle, &buf, &written)       // ← ADEC input buffer write
```

`MI_AUDIO_Write` = ioctl 0xC038100A = запис у **ADEC input DMA buffer**. Немає жодного альтернативного path (SPDIF TX buffer, bypass DMA, digital-out write) у `mi_common_raw_write` або в `utils_common_out_write`.

`mi_common_raw_write: timeout` = ADEC input FIFO заповнений і DSP не споживає (декодер не працює). Відсутність timeout = DSP споживає вхід (в FIFO), але це **не гарантує** проходження до SPDIF TX (декодер може не синхронізуватись на не-AC3 даних).

## 5. Формат даних: elementary AC3 vs IEC61937 burst

| | Working AC3 (elementary) | Kodi IEC (framed) |
|---|---|---|
| Синхронізація DSP | AC3 sync 0x0B77 кожні 1536B | IEC61937 Pa/Pb/Pc/Pd замість AC3 sync |
| DSP AC3 decoder | lock → decode → bypass ✓ | **не знайде 0x0B77** → fail → silence |
| Bypass-тап вихід | компресований AC3 → SPDIF | **пусто** |
| Ресівер | Dolby Digital | тиша |

IEC61937 burst структура: `[Pa(2B) Pb(2B) Pc(2B) Pd(2B)] [AC3 frame 1536B] [zero padding to 6144B]`. DSP AC3 decoder шукає `0x0B77` (AC3 sync) — IEC header `0x72F8 0x4E1F` не є AC3 sync → decoder не синхронізується → немає output.

## 6. Чи існує окремий IEC61937 burst writer у vendor stack?

**Так, але недоступний з Android audio framework:**

- `CAEPackIEC61937` (Kodi side) — формує берсти в userspace, передає через AudioTrack
- `libutopia.so` — містить `DTSDecSDOPacker_API_*`, `PackSPDIFStream`, `SDO_SpdifPacker_*` — але ці функції викликаються **тільки** з native/DTV audio шляху (MApi/MMF), не з Android audio HAL
- `HAL_AUDIO_SPDIF_*` (utpa2k.ko) — керують TX mode/CS/mute, але не приймають data buffers
- Немає жодного `SPDIF_TX_Write()` або еквівалента, який приймав би довільні байти напряму в SPDIF TX DMA

## 7. Висновок

### NOT POSSIBLE

**Усі compressed SPDIF paths у MStar архитектурі залежать від ADEC decode program.** SPDIF TX фізично живиться з DSP output matrix, який перемикається в non-PCM тільки коли decode program успішно декодує компресований вхід. Немає окремого raw/IEC61937 TX writer, доступного з Android audio framework.

### Підтвердження

| Факт | Статус | Джерело |
|---|---|---|
| `MI_AUDIO_Start(Codec:5)` → DSP AC3 program → SPDIF non-PCM → DD | **CONFIRMED** | runtime: working AC3 → DD |
| `MI_AUDIO_Start(Codec:5)` + IEC61937 data → decode fail → silence | **CONFIRMED** | runtime: IEC test → тиша |
| `mi_common_raw_write` = тільки ADEC input writer | **CONFIRMED** | decompile |
| `HAL_AUDIO_SPDIF_*` = mode/CS/mute only, не data injection | **CONFIRMED** | symbol analysis |
| IEC61937 framing блокує DSP AC3 sync → silence | **CONFIRMED** | runtime + format analysis |
| `DTSDecSDOPacker` / native packer — недоступний з Android HAL | **CONFIRMED** | import analysis |
| Фізичний SPDIF TX для DTS без DTS ADEC | **NOT POSSIBLE** | повний ланцюг |

### Єдині робочі шляхи для DTS на ресівері

1. **DTS → AC3 transcode** → native AC3 passthrough → DD (WORKING, CONFIRMED)
2. **DTS → PCM decode** → SPDIF PCM → ресівер у PCM mode (стерео)
