# R2 — SPDIF Output Mode vs. Downstream DTS SDO: Controlled Experiment

**Date:** 2026-09-04 (UTC)
**Device:** Thundeal TD98 Pro / C50A — MTK MT5889, Android 11, kernel 4.19.116
**Baseline:** patched, AC3-working / DTS-failing experimental baseline (per R1): libmi3.so v3 (`4c199a5c…`), utpa2k.ko Exp B (`4c5e6fbb…`), mik.ko patched MonitorTask (`647b08db…`).
**Constraints honored:** no binary patched; no EDID altered; no ARC established; no topology change; SPDIF/DD setup unchanged; single reversible configuration-only experiment; restored at end.

---

## EXECUTIVE RESULT — Decision C

> **The SPDIF output mode/type cannot be forced via any available reversible, configuration-only lever.**
> Therefore, from this experiment, the original split question ("output-mode/type OR downstream DTS SDO?") resolves to: **the output-mode setting is not the reachable blocker** — it is permanently stuck in AUTO and is *never* explicitly set to DTS/BYPASS on this firmware. The DTS-specific failure must be sought in the **downstream DTS SDO / IEC‑61937 transmit packer** path (next phase), consistent with R1's finding that the AC3/DTS divergence occurs *below* `MI_AUDIO_Start` at the transmit stage.

---

## A. NORMAL STATE (Objective 4 — read-only documentation)

`dumpsys media.audio_flinger` (HAL dump), captured before any change and confirmed at end of session:

```
 - hdmi arc mode: ui(auto), current(auto)
 - hdmi arc type: ui(not-specified), current(not-specified)
 - hdmi tx mode:  ui(auto), current(auto)
 - hdmi tx type:  ui(not-specified), current(not-specified)
 - spdif mode:    ui(auto), current(auto)
 - spdif type:    ui(not-specified), current(not-specified)
 - spdif delay:   0
Supported ARC capability: dd:0, ddp:0(atmos:0), dts:0, aac:0, mat-truhd:0(Byte3:0), dtshd:0,
```

**This is the runtime state at rest, during working AC3 playback, and during failing DTS playback** (R1 confirmed identical AC3/DTS). All three digital outputs are **permanently AUTO with no explicit type**. The HAL dump is the safest read-only mechanism (confirmed); `adev_get_parameters` is the in-HAL reader, reachable only via the binder `getParameters` transaction (which does not route on this fork — see C.2).

---

## B. RUNTIME MECHANISM SUPPLYING THE KEYS (Objectives 1 & 3)

Reverse-engineered `audio.primary.mt5889.so` (verified STOCK, hash `8c11c348…`).

**Entry points (resolved from `HMI` → `hw_module_methods_t.open` → `adev_open` vtable):**

| vtable slot | function | VA |
|---|---|---|
| `set_parameters` | `adev_set_parameters` | **0x21BF0** |
| `get_parameters` | `adev_get_parameters` | **0x2072C** |
| `dump` | `adev_dump` | 0x24DE8 |
| `open_output_stream` | … | 0x22D4C |
| `create_audio_patch` | … | 0x2070E |

**Key parsing (delta-target xref analysis):**

| Key string | used in `get_parameters` | used in `set_parameters` | consumed by |
|---|---|---|---|
| `spdif_type` | 0x20F86 | 0x2239C, 0x22404 | `utils_ApplyDigitalOutputSetting` |
| `hdmi_tx_mode` | 0x20D12, 0x20E92 | 0x22184, 0x221EC | `utils_ApplyDigitalOutputSetting` |
| `hdmi_tx_type` | 0x20E92 | — | (read) |
| `sound_type` | — | 0x229E8 | `utils_ApplyDigitalOutputSetting` |
| **`spdif_mode`** | — | — | **DEAD — no xref anywhere** |

**Critical correction to R1's premise:** the live key is **`spdif_type`**, NOT `spdif_mode`. `spdif_mode` is an unused string in the binary. The Settings/glue side uses `sound_spdif_type`.

**Value encoding:** `spdif_type` is an **`AUDIO_FORMAT` constant** (verified in `utils_ApplyDigitalOutputSetting` @0x29038: it compares the parsed value's byte[3] against `0x0b` = DTS).

| value | meaning |
|---|---|
| `0x09000000` | AC3 |
| `0x0A000000` | E‑AC3 |
| `0x0B000000` (= 184549376) | **DTS** |
| `0x0C000000` | DTS‑HD |

So "force DTS" = `spdif_type=184549376`.

**Caller chain (Objective 3):** (framework) `AudioManager.setParameters` → `AudioSystem::setParameters` → `AudioFlinger::setParameters` → **`adev_set_parameters`** (parses the k=v pairs) → **`utils_ApplyDigitalOutputSetting`** (0x28684; emits `SetSpdifOutputMode=*` / `SetSpdifOutputType=*` logs and applies the mode) → `MI_AOUT_SetDigitalMode` → `_MI_AOUT_SetHdmiAutoMode` → `MApi_AUDIO_SPDIF_SetMode` → `HAL_AUDIO_SPDIF_SetMode` (in `libmi3.ko` / `utpa2k.ko`).

---

## C. THE ONE CONTROLLED EXPERIMENT (Objectives 5 & 6)

**Objective 5 — attempt to force the strongest explicit nonPCM passthrough:** call `adev_set_parameters("spdif_type=0x0B000000")` = DTS/BYPASS, the native DTS combination.

Tests performed (all reversible, no binary patched):

1. `service call media.audio_flinger 18 i32 0 s16 "spdif_type=184549376"` → `Result: Parcel(00000000)` (status 0)
2. `service call media.audio_flinger 18 i32 0 s16 "spdif_type=DTS"` → 0
3. `service call media.audio_flinger 18 i32 0 s16 "spdif_delay=10"` → 0
4. `service call media.audio_flinger 19 i32 0 s16 "spdif_type"` (getParameters) → `Parcel(NULL)`
5. `settings put system sound_spdif_type 2` → no reaction
6. Probe `media.audio_flinger` txns 14–24 → all 0/NULL/errno, **no HAL log, no state change**
7. Probe `media.audio_policy` txns 28–34 → all 0/NULL/errno, **no HAL log, no state change**

**Objective 6 — replay DTS under forced mode:** not reachable (mode could not be forced → see below).

**Result of every attempt:** `dumpsys media.audio_flinger` unchanged (`spdif type: ui(not-specified), current(not-specified)`); **no `SetSpdifOutputType`/adev_set_parameters log** in logcat. The configuration write is accepted by the binder but never reaches `adev_set_parameters`.

### Root cause — why the mode cannot be forced (Decision C)

1. **The invocation path is absent.** The only code that sets `spdif_type`/`hdmi_tx_mode` is `adev_set_parameters`, driven *exclusively* by the vendor TV‑settings glue: `SettingsProvider` `sound_spdif_type` → `AudioManager.setParameters` → `AudioFlinger::setParameters` → `adev_set_parameters`. That glue is **non‑functional on this projector** — proven two ways:
   - Changing `sound_spdif_type` (settings put) produces **zero HAL reaction**.
   - R1 already proved `adev_set_parameters` is **never called** during AC3 **or** DTS playback (HAL stays AUTO throughout).
2. **The standard binder `setParameters`/`getParameters` transactions do not route to the HAL on this fork.** The descriptor `android.media.IAudioFlinger` is **absent from every audio library** (`libaudioflinger`, `libaudioclient`, `libaudiohal@*`, `libaudiopolicyservice`, `libaudiomanager`); txns 14–24 return success/null without ever producing an `adev_set_parameters` log, and `getParameters` returns NULL.
3. **No sysfs / module‑parameter / property path exists** for `spdif_type` (R1: no `vendor.audio.spdif` string in any deployed binary; `mik`/`utpa2k` expose no digital‑mode module parameter; `/sys/kernel/mik/MI_*` returns `-EACCES`).

**Conclusion:** the HAL *capability* to force DTS/BYPASS exists (`utils_ApplyDigitalOutputSetting` + `MApi_AUDIO_SPDIF_SetMode`), but it is unreachable through any available non‑patch lever on this firmware. The output mode is therefore stuck in AUTO for both AC3 and DTS — which is exactly why R1 saw "output mode never reacts to DTS."

---

## D. INCUBATING LEAD — DSP/DTS path (next phase direction)

During every settings touch, logcat emitted, directly from the HAL (`audio_hw_primary`):

```
E audio_hw_primary: mi_getCodecType: MI_AUDIO_GetAttr failed, ret=0x3
E audio_hw_primary: mi_getCodecType: MI_AUDIO_GetHandle failed!!! ret = 9
```

`MI_AUDIO_GetHandle` returning `9` (= `-ENODEV`/no codec handle) while the HAL queries the codec type is a strong, fresh signal that **the DSP codec/SDO handle acquisition for the DTS path is failing** — i.e., the failure is inside the transmit/packer stage rather than the (unreachable) output-mode selector. This points squarely at the **DTS SDO / IEC‑61937 packer** and the AUTH/DSP license words (`0x440/0x444/0x4d0/0x4d8/0x582`) as the next investigation target (R1's remaining DTS‑specific, EDID‑independent candidates).

---

## E. ARTIFACTS (all new, none overwritten)

`runtime_phase2/`:
- `R2_levers_scan.txt` — module params / `cmd`/service survey
- `R2_baseline_dump.txt` — baseline HAL dump + settings list
- `R2_dump_before.txt`, `R2_dump_after_setDTS.txt`, `R2_final_state.txt` — HAL dumps (auto/not-specified throughout)
- `R2_final_hashes.txt` — post-session module/lib hashes (libmi3.so still `4c199a5c…`, unchanged)

`tools/`:
- `r2_xrefs.py`, `r2_hal_vtable.py`, `r2_vtable_resolve.py`, `r2_funcs.py` (exidx-based function map + annotated disasm), `r2_delta_xrefs.py` (PC-delta target resolution — the key technique for this compiler), `r2_tbl_xrefs.py`, `r2_kv.py`, `r2_plt.py`, `r2_aftrans*.py`
- `r2_adev_open.asm`, `r2_set_parameters.asm`, `r2_get_parameters.asm`, `r2_kv.asm`
- `libs_host/libaudioflinger.so` (pulled for txn-code analysis)

---

## F. DECISION & STOP

**Decision C.** Output mode/type is not the reachable blocker; it is permanently AUTO and cannot be forced without patching/re-wiring (excluded by constraints). The DTS failure is therefore to be pursued in the **downstream DTS SDO/IEC‑61937 packer** and AUTH/DSP license path.

**Device state at stop:** restored. HAL dump = AUTO/not-specified; `sound_spdif_type` = 4 (restored); no binary modified; module/lib hashes unchanged from R1 baseline.

No patch deployed. No EDID replaced. No ARC established. Per directive, **stopping after this single controlled experiment.**
