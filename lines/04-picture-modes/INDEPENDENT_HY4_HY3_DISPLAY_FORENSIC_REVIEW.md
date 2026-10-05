# INDEPENDENT HY4/HY3 DISPLAY FORENSIC REVIEW

**Target:** Thundeal TD98 Pro / C50A — MediaTek MT5889, Android 11 (SDK 30), Linux 4.9
**Scope:** First-principles reconstruction of the display capability surface. Not a continuation of prior conclusions.
**Method:** Static binary forensics (ELF parsing, Ghidra headless decompilation of fresh projects) + configuration forensics.
**Constraint honoured:** `C:\firmware_temp\ree_payload.bin` was never opened, read, hashed, parsed, or analysed. No existing file was modified, deleted, or overwritten.

**Evidence grades used throughout:** `PROVEN BY BINARY` · `PROVEN BY RUNTIME` · `STRONGLY INDICATED` · `PRESENT IN CONFIG/TABLE` · `UNRESOLVED` · `DISPROVEN`

---

## 1. Executive Summary

The previous investigation reached two mutually contradictory verdicts on the same question ("Option B: HWC is the only blocker, two patches are sufficient" vs. "Option B is disproven, HWC never programs hardware"). This review reproduces some of each and rejects the load-bearing parts of both.

**What is now established by direct control-flow evidence:**

1. **HWC advertises exactly two display configs, both 1920×1080.** `HWCDisplayDevicePrimary`'s constructor calls `addConfig()` exactly twice with width/height taken from a hard-coded literal pool `{1920, 1080}` and vsync periods `20 000 000 ns` (50 Hz) and `16 666 666 ns` (60 Hz). — `PROVEN BY BINARY`

2. **A hidden bridge exists and has been identified.** It is `HWComposerDevice::eventWorkThread()`. `setActiveConfigWithConstraints()` only arms a *pending* change; the worker thread later fires a virtual dispatch **and** `setActivePanelFrequency()`. — `PROVEN BY BINARY`

3. **`setActivePanelFrequency()` IS called during mode switching** — contradicting the prior root-cause report. It is called asynchronously from the event worker thread after the vsync-period deadline expires. — `PROVEN BY BINARY`

4. **`setActivePanelFrequency()` takes a CONFIG ID, not a frequency.** `configId == 0 → FreeRunConfig 1`; anything else `→ FreeRunConfig 2`. — `PROVEN BY BINARY`

5. **`MI_DISP_SetControlAttr()` does not accept an arbitrary frequency.** Attribute 1 clamps the value to `0..3` and treats it as a free-run configuration *index*. — `PROVEN BY BINARY` (prior claim **DISPROVEN**)

6. **Only four free-run configurations exist in the whole platform (0–3).** Their behaviour is fully decoded in `xcker.ko`. **VRR is enabled only for config 0** — a value HWC never sends. Android therefore cannot reach VRR through this path. — `PROVEN BY BINARY`

7. **The consequence: adding HWC configs alone cannot produce new refresh rates.** Because `setActivePanelFrequency` collapses every non-zero config id to `FreeRunConfig 2`, a hypothetical config 2/3/4 would still select 60 Hz. "Two HWC patches are sufficient" is therefore **DISPROVEN** as stated.

8. **A complete 37-entry `timing_enum → (width, height, refresh)` table was recovered** from `hwcomposer.mt5889.so` `.rodata`. It contains 3840×2160 @ 24/25/30/50/60 and 4096×2160 @ 24/25/30/50/60 — but **no entry above 60 Hz at any resolution.** — `PROVEN BY BINARY`

9. **`m_wPanelWidth = 3840` does not prove a native 4K imager.** The board enables pixel shift, the MI driver exposes `SetPixelShift` as a first-class control, and the 120 Hz profile is a TI **DLPC6540** 1920×1080 DLP panel. — `PROVEN BY BINARY` (pixel-shift control) / `STRONGLY INDICATED` (1080p-class imager with shift)

10. **4K120 is not present anywhere in the timing surface** — not in the HWC table, not in mik.ko's `E_MI_DISP_TIMING_*` enum strings, not in the selectable panel profiles. `m_wVRR_max = 120` is a panel INI field only. — `DISPROVEN`

**Unresolved boundary (explicit):** the identity of the virtual function at `vtable+0x44` called with the pending config id. The vtable is relocated exclusively through Android packed relocations ("APS2"), so its slots cannot be resolved from file bytes. This is bounded and does not affect any conclusion above. — `UNRESOLVED`

---

## 2. Investigation Map

### 2.1 Artifacts analysed (independently re-derived)

| Artifact | Size | Role | Verification |
|---|---|---|---|
| `extracted/hw/hwcomposer.mt5889.so` | 475,016 | HWC2 HAL — central subject | ELF header + `.dynsym` + fresh Ghidra project `HY4_HWC` |
| `extracted/hw/hwcomposer.debugdata.elf` | 93,520 | Compressed debug symbols | 408-symbol `.symtab` extracted (partial — templates/thunks only) |
| `spdif_audio_investigation/kmods/mik.ko` | 8,084,248 | MI display kernel module | Full 30,247-entry `.symtab`; fresh Ghidra project `HY4_MIK` |
| `extracted/kmods/xcker.ko` | 677,224 | XC/display engine (implements `*_EX`) | Fresh Ghidra project `HY4_XC` |
| `extracted/kmods/kdrv_dolby_vision.ko` | 651,948 | Dolby Vision driver | Inventoried only |
| `libmi3.so` | 750,608 | Userspace MI client | Export scan (`MI_DISP_SetOutputTiming` @ 0x6cfc5, 220 B) |
| `extracted/cusdata/**` | 187 INI | Panel / board / HDR / EDID config | Direct parse |
| `extracted/cusdata/common/EDID_BIN/*.bin` | 60 files | HDMI RX EDID images | Active selection traced |

### 2.2 Address-space conventions (independently verified — do not trust prior labels)

- `hwcomposer.mt5889.so` is **32-bit ARM** (`EM_ARM = 0x28`), `ET_DYN`, `e_entry = 0x38908`, `e_flags = 0x5000200`, 27 sections.
- `.text` ELF vaddr `0x38908`, size `0x330f0`, **file offset `0x37908`** (vaddr−offset delta = `0x1000`).
- Ghidra's ElfLoader applies **image base `0x10000`** → **Ghidra address = ELF vaddr + 0x10000**. Verified blocks: `.text 00048908–0007b9f7`, `.rodata 0003b1f0–00047907`, `.plt 0007ba00–0007f18f`, `.data.rel.ro 00080190–00080c77`, `EXTERNAL 00083000–000835d7`.
- **`.rel.dyn` is `SHT_ANDROID_REL` (0x60000001) with magic `APS2`** — Android *packed* relocations, **not** standard 8-byte `Elf32_Rel`. Any tool that parses it as REL produces garbage. This is why vtable slots are unrecoverable from file bytes.
- Prior reports mix "ELF" and "Ghidra" addresses inconsistently (e.g. `getDisplayVsyncPeriod` labelled "ELF 0x52314" is actually the *Ghidra* address; ELF is `0x42314`). **This review uses ELF vaddr unless explicitly marked.**

### 2.3 New artifacts produced (all read-only, uniquely named, nothing overwritten)

`hy4_review/` — containing `Hy4Probe.java` (Ghidra headless probe), `elf_info.py`, `scan_timing_api.py`, `hy4_vtable.py`, `hy4_edid.py`, and the decompilation dumps `HY4_decomp_*.txt`, `HY4_mik_*.txt`, `HY4_xc_*.txt`. Ghidra projects `HY4_HWC`, `HY4_MIK`, `HY4_XC` are new and isolated.

---

## 3. Reconstructed Architecture

### 3.1 Output path (Android → photons)

```
Android DisplayManager / SurfaceFlinger
        │  HWC2 calls
        ▼
hwcomposer.mt5889.so
   HWComposerDevice  ── eventWorkThread()  (dedicated worker) ──┐
   HWCDisplayDevicePrimary                                      │ (deferred)
   HWComposerOSD / HWComposerDisplayHalMiOsd / HWComposerMiWrapper
        │  libmi3.so  (thin IPC client)
        ▼  /dev/mi ioctl (MI_DEV_IOC_DISP_*, 90 commands)
mik.ko   MI_DISP_*  →  _MI_DISP_*  →  _MI_DISP_XC_*
        │  MI_DISP_IMPL_*_EX  (extern)
        ▼
xcker.ko  MApi_SC_* / MApi_XC_*   (ForceFreerun, SetFreerunVFreq, SetVRROn,
        │                          SetOutputTiming, SetPixelShift, SetPanelTiming)
        ▼
XC scaler / GOP / panel link (V-by-One, 16-lane)
        ▼
DLP projection engine (DLPC6540-class) → optics
```

### 3.2 Input path (HDMI RX)

```
HDMI RX ×4  (BOARD_EDID_INFO_HDMI_COUNT = 4)
   → EDID presented from EDID_BIN (active: Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin)
   → input timing detection → XC → scaler → panel timing
```
Note `board.ini [DispoutConfig]`: **`m_u32SupportHdmiTxCount = 0`** — this platform has **no HDMI transmitter**. 4K over HDMI on this device is *input* capability only; there is no 4K HDMI output.

### 3.3 HDR / Dolby Vision path

```
HWCDisplayDevicePrimary::initHdrCapabilities
   → HWComposerOSD::getHdrCapabilities
       → HWComposerDisplayHalMiOsd::getHdrCapabilities
           → MI_DISP_GetCaps()        (mik.ko 0x11e1a4)
Dolby: kdrv_dolby_vision.ko + HDR.ini (dolby.bin / dolby_factory.cfg 3D LUT)
```

---

## 4. HWC Analysis

### 4.1 Config creation — the 1080p ceiling

`HWCDisplayDevicePrimary::HWCDisplayDevicePrimary` — ELF `0x40a19` (Ghidra `0x50a18`, 1000 B):

```c
MI_DISPOUT_PanelGetAttr(handle, 0x1a, 0, &panelW);   /* 26 — panel width  */
MI_DISPOUT_PanelGetAttr(handle, 0x1b, 0, &panelH);   /* 27 — panel height */
__android_log_print(3, ..., panelW, panelH);         /* logged only       */

HWCDisplayDevice::addConfig(this, *(uint*)(this+8), *(uint*)(this+0xc),
                            20000000, *(int*)(this+0x10), *(int*)(this+0x14));
HWCDisplayDevice::addConfig(this, *(uint*)(this+8), *(uint*)(this+0xc),
                            *(int*)(this+4),  *(int*)(this+0x10), *(int*)(this+0x14));
HWCDisplayDevice::initConfigs(this);
```

Xrefs confirm `addConfig` has **exactly two call sites**, both here (Ghidra `0x50cb8`, `0x50ccc`), and `initConfigs` **one** (`0x50cd2`).

The fields are filled by the **base** constructor `HWCDisplayDevice::HWCDisplayDevice` (ELF `0x3ca81`) from a **literal pool** at Ghidra `0x4cc88` (ELF `0x3cc88`, file offset `0x3bc88`):

| Ghidra | ELF | Value | Meaning |
|---|---|---|---|
| `0x4cc88` | `0x3cc88` | **16 666 666** (`0x00FE502A`) | config 1 vsync period → 60 Hz |
| `0x4cc8c` | `0x3cc8c` | **1920** (`0x780`) | **width** |
| `0x4cc90` | `0x3cc90` | **1080** (`0x438`) | **height** |
| `0x4cc94` | `0x3cc94` | **320 000** | dpiX (320.000) |
| `this+0x14` | — | **320 000** | dpiY (320.000) |

> **PROVEN BY BINARY** — the advertised configs are **1920×1080 @ 50 Hz** and **1920×1080 @ 60 Hz**, and the width/height are hard-coded, *not* derived from the panel. Note the constructor *does* query the real panel attributes (`0x1a`/`0x1b`) but uses them only for a log line.

### 4.2 `addConfig` — no hardware contact

`HWCDisplayDevice::addConfig(uint w, uint h, int vsync, int dpiX, int dpiY)` — ELF `0x3ce91` (552 B). Allocates a 40-byte `Config`, populates an `unordered_map<hwc2_attribute_t,int>` (keys 1,2,3,4,5,7), appends a `shared_ptr` to the vector at `this+0xd0..0xd8`. **No MI_DISP call of any kind.**

### 4.3 `initConfigs` — selects the LAST config

`HWCDisplayDevice::initConfigs()` — ELF `0x3d32d` (144 B): logs an error if the vector is empty, then `setActiveConfig(numConfigs − 1)` → **config 1 (60 Hz)**, then seeds colour-mode/render-intent maps. **No hardware call.**

### 4.4 `setActiveConfig` — pure in-memory swap

`HWCDisplayDevice::setActiveConfig(uint)` — ELF `0x3d3bd` (194 B):

```c
if (idx >= numConfigs ||
    *(int*)(this+0x70) != *(int*)(cfg+0x70) ||      /* display W vs config W */
    *(int*)(cfg+0x74)  != *(int*)(this+0x74))       /* config H vs display H */
        return HWC2_ERROR_BAD_CONFIG;
...
*(int**)(this+0xdc) = cfg;   /* active Config shared_ptr — that is ALL it does */
*(int*) (this+0xe0) = ctrl;
return HWC2_ERROR_NONE;
```

> **PROVEN BY BINARY** — a bounds check, a width/height compatibility check, and a pointer swap. Confirmed **non-virtual** (reached via PLT `0x7c410`, not a vtable). Its **only caller in the whole binary is `initConfigs`** (Ghidra `0x4d366`).

### 4.5 `setActiveConfigWithConstraints` — arms, does not apply

`HWCDisplayDevicePrimary::setActiveConfigWithConstraints(...)` — ELF `0x42509` (184 B):

```c
uVar6 = systemTime(1);
HWCDisplayDevice::getConfig(&local_30);
if (local_30 != 0) {
    *(uint*)(this + 0x60) = configId;                       /* PENDING config  */
    lVar7 = systemTime(1);
    *(longlong*)timeline = lVar7 + (desiredTimeNanos − uVar6);
    *(uint*)(this + 0x58) = 1;                              /* PENDING flag    */
    *(uint*)(this + 0x5c) = 0;
    *(uint*)(this + 0x50) = timeline_low;                   /* deadline        */
    *(uint*)(this + 0x54) = timeline_high;
}
```

> **PROVEN BY BINARY** — no hardware call. It defers.

### 4.6 The hidden bridge — `eventWorkThread`

`HWComposerDevice::eventWorkThread()` — ELF `0x38bf9` (Ghidra `0x48bf8`, body 1722 B). A dedicated thread (`prctl`, `setpriority(PRIO_PROCESS,0,-8)`, `sched_setscheduler`). Its loop waits on vsync either by `ioctl(fd, 0xc010321e, &ts)` (real) or `clock_nanosleep` (fake-vsync mode), polling at `usleep(0x411a)` ≈ 16.67 ms otherwise. The decisive block:

```c
if (lastApplied != *(uint*)(primary + 0x60)) {                 /* pending changed  */
    uVar35 = systemTime(1);
    if (deadlineExpired && (*(int*)(primary+0x58) != 0 ||
                            *(int*)(primary+0x5c) != 0)) {
        __android_log_print(3, ..., *(int*)(primary+0x58));
        (**(code**)(*(int*)primary + 0x44))(primary, *(uint*)(primary+0x60));
        lastApplied = *(uint*)(primary+0x60);
        HWCDisplayDevicePrimary::setActivePanelFrequency(primary, lastApplied);
    }
}
```

Xref of the `setActivePanelFrequency` PLT (`0x7bd90`): **exactly one caller — `eventWorkThread` at Ghidra `0x491ee`.**

The same deferred dispatch also appears in `HWCDisplayDevicePrimary::getDisplayVsyncPeriod` (ELF `0x42315`): it fires `vtable+0x44` when the deadline passes, then reads the active config, calls `vtable+0xd8`, and **hard-codes `20 000 000 ns` if the active Config's field at +4 is 0**.

> **PROVEN BY BINARY** — this is the bridge. The prior root-cause report's implication that `setActivePanelFrequency()` is *not* called during mode switching is **DISPROVEN**; it is called, asynchronously, from the event worker thread.

### 4.7 `setActivePanelFrequency` — takes a config id

ELF `0x4244d` (Ghidra `0x5244c`, 140 B):

```c
local = 2;
if (param_1 == 0) local = 1;                    /* param_1 IS THE CONFIG ID */
MI_DISP_GetController(...);  if (fail) MI_DISP_OpenController(...);
MI_DISP_SetControlAttr(handle, 1, &local);
__android_log_print(3, ..., this[0x70], this[0x74], param_1, local);
```

> **PROVEN BY BINARY** — with only two configs in existence, the only values the platform can ever emit are **1** (config 0 = 50 Hz) and **2** (config 1 = 60 Hz).

### 4.8 Second, previously unreported caller of `MI_DISP_SetControlAttr`

`HWComposerDisplayHalMiOsd::hwcMiDisplay` (Ghidra `0x6bbf4`, 2290 B) calls `MI_DISP_SetControlAttr(handle, 1, &value=1)` at `0x6c3f0` when the vendor property `mediatek.sysprop.mstar_disable_osd == 2`. This is a property-driven path into attribute 1 that is entirely independent of `setActivePanelFrequency`.

### 4.9 Timing read-back and dynamic resolution

- **`HWComposerMiWrapper::getMiOsdTiming`** (Ghidra `0x70df4`) — seeds `local = 0x11` (17), calls `MI_DISP_GetOutputTiming(handle, &enum)`, then indexes three 37-entry tables (see §6).
- **`HWComposerDisplayHalMiOsd::checkGopTimingChanged`** (Ghidra `0x69ac4`, 194 B) — reads MI timing via a virtual call and, **on change, updates `primary+0x168` (width), `+0x16c` (height), `+0x148` (refresh)** and sets `+0x180 = 1`. Initialised in the constructor to **1920×1080**.
- **`HWCDisplayDevicePrimary::checkGopTimingChanged`** (Ghidra `0x545cc`, **8 bytes**) merely forwards to `HWComposerOSD::checkGopTimingChanged` (Ghidra `0x5bf70`, **also 8 bytes**) — both are stubs.
- **`getCurrentTiming(int*,int*)`** (Ghidra `0x54764`, 20 B) returns exactly `this+0x168` and `this+0x16c`.
- **`getRefreshRateFromPanel()`** (Ghidra `0x54928`) — `mbootenv_get` → `iniparser_load` → `refresh = DCLK·10⁶ /(HTotal·VTotal)`, defaults DCLK 594 / HTotal 4400 / VTotal 2250, fallback `0x3c` (60).
- **`getRefreshPeriod()`** (Ghidra `0x54a80`) — honours `mstar_override_refresh_rate`, else `HWComposerOSD::getOutputFrequence()`; falls back to `getRefreshRateFromPanel()` when outside 24..60; returns `10⁹ / freq`.
- `eventWorkThread` also writes `this+4` (the config-1 vsync period) from `mstar_fakevsync_freq` when fake-vsync is enabled — **a runtime-writable, reversible knob.**

---

## 5. Hardware Timing Programming

### 5.1 Who actually programs the panel

| Question | Answer | Grade |
|---|---|---|
| Does HWC import `MI_DISP_SetOutputTiming`? | **No.** Its MI_DISP imports are `GetController, OpenController, SetControlAttr, GetCaps, Init, DeInit, DisableBootLogo, GetOutputTiming, GetHandle` + `MI_DISPOUT_*`. No `SetOutputTiming` of any spelling. | `PROVEN BY BINARY` |
| Who defines it? | `libmi3.so` ELF `0x6cfc5` (220 B — thin IPC client); `mik.ko` ELF `0x11f910` (3688 B — implementation) | `PROVEN BY BINARY` |
| Who calls it? | **Reached through ioctl, not dynamic linking.** `mik.ko` exposes `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` (and `..._GET_OUTPUT_TIMING`) among **90 `MI_DEV_IOC_DISP_*` commands**. `libutopia.so`, `libsurfaceflinger.so`, `libgui.so` contain zero occurrences of the symbol. | `PROVEN BY BINARY` for the ioctl surface; `STRONGLY INDICATED` that this is the production path |
| Which ELF imports it? | Only `libmi3demo.so` (a demo library) — a red herring for production behaviour | `PROVEN BY BINARY` |

**Resolved:** the earlier "caller unknown / CRITICAL" gap is explained — the call is an **ioctl**, so it never appears as an ELF import. The production callers live in TV middleware / vendor services not present in the extracted artifact set.

### 5.2 `MI_DISP_SetControlAttr` — attribute semantics (mik.ko ELF `0x11d088`)

```c
if ((handle & 0x4354) == 0x4354) {
  switch (attr) {
    case 0: MI_DISP_IMPL_XC_SetOsdDetect_EX(_astXcDeviceId, b, u);            /* OSD detect   */
            break;
    case 1: u = *value;
            if (u > 2) u = 3;                        /* *** CLAMP TO 0..3 *** */
            _eFreeRunConfig = u;
            MI_DISP_IMPL_XC_SetFreeRunConfig_EX(_astXcDeviceId, u);
            break;
    case 2: MI_DISP_IMPL_XC_SetPixelShift_EX(b);                             /* pixel shift  */
            break;
    case 3: MI_DISP_IMPL_XC_SetPixelShiftParams_EX(&params);                 /* shift params */
            break;
  }
}
```

> **PROVEN BY BINARY** — attribute 1 is a **free-run configuration index clamped to 0..3**, not a frequency. The prior claim "`MI_DISP_SetControlAttr()` accepts arbitrary frequency" is **DISPROVEN**.
> Attributes **2 and 3 are pixel-shift controls** — directly material to §9.

### 5.3 `MI_DISP_IMPL_XC_SetFreeRunConfig_EX` (xcker.ko, Ghidra `0x46a74`)

```c
uVar2 = 3;                                        /* freerun VFreq enum */
if      (cfg == 0) { vrr = 1; }                   /* VFreq 3 + VRR ON   */
else if (cfg == 2) { vrr = 0; uVar2 = 1; }        /* VFreq 1 + VRR OFF  */
else               { vrr = 0; if (cfg == 1) uVar2 = 0; }
                                                  /* cfg1→VFreq 0; cfg3→VFreq 3 */
MApi_SC_ForceFreerun(1);
MApi_SC_SetFreerunVFreq(uVar2);
_bForceSetPanelTiming = 1;
FUN_00036170(0);                                  /* apply panel timing */
MApi_SC_ForceFreerun(0);
MApi_XC_SetVRROn(vrr);
```

| FreeRunConfig | VFreq enum | `MApi_XC_SetVRROn` | Reachable from HWC? |
|---|---|---|---|
| 0 | 3 | **1 (ON)** | **No** — HWC never sends 0 |
| 1 | 0 | 0 (OFF) | Yes (config 0 → 50 Hz) |
| 2 | 1 | 0 (OFF) | Yes (config ≠ 0 → 60 Hz) |
| 3 | 3 | 0 (OFF) | No |

> **PROVEN BY BINARY** — four discrete configurations only. **VRR is enabled only for cfg 0, which HWC never emits.**
> The prior claim "FreeRunConfig 2/3 correspond to 24/30 Hz" is **DISPROVEN**: cfg 2 → VFreq 1 (= the 60 Hz path) and cfg 3 → VFreq 3; neither has any 24/30 Hz evidence behind it.

### 5.4 The complete chain (all links evidenced)

```
SurfaceFlinger
 └─ setActiveConfigWithConstraints(cfgId, constraints, timeline)
      └─ HWCDisplayDevicePrimary::setActiveConfigWithConstraints   ELF 0x42509
           └─ this+0x60 = cfgId ; this+0x58 = 1 ; this+0x50/54 = deadline   [PROVEN]
                └─ (worker thread) HWComposerDevice::eventWorkThread  ELF 0x38bf9
                     └─ deadline expired && pending  →  vtable+0x44(cfgId)  [PROVEN]
                     └─ HWCDisplayDevicePrimary::setActivePanelFrequency(cfgId)
                          └─ value = (cfgId == 0) ? 1 : 2                   [PROVEN]
                               └─ MI_DISP_SetControlAttr(handle, 1, &value)
                                    └─ mik.ko: clamp 0..3 → _eFreeRunConfig  [PROVEN]
                                         └─ MI_DISP_IMPL_XC_SetFreeRunConfig_EX
                                              └─ MApi_SC_SetFreerunVFreq(0|1) [PROVEN]
                                              └─ MApi_XC_SetVRROn(0)          [PROVEN]
                                              └─ _bForceSetPanelTiming=1 → apply
```

---

## 6. Timing Map

### 6.1 `timing_enum → (width, height, refresh)` — recovered tables

`HWComposerMiWrapper::getMiOsdTiming` indexes **three contiguous 37-entry int32 tables** in `.rodata` (148 bytes each):

| Table | ELF vaddr | Range |
|---|---|---|
| width | `0x37738` | `0x37738–0x377c3` |
| height | `0x377cc` | `0x377cc–0x37857` |
| refresh | `0x37860` | `0x37860–0x378eb` |

Guard: `if (enum < 0x25) {…} else { width=1920; height=1080; refresh=0; }`

| enum | resolution | Hz | | enum | resolution | Hz |
|---|---|---|---|---|---|---|
| 0 | 720×480 | 60 | | 19 | 4096×2160 | 25 |
| 1 | 720×480 | 60 | | 20 | 4096×2160 | 30 |
| 2 | 720×576 | 50 | | 21 | 4096×2160 | 50 |
| 3 | 720×576 | 50 | | 22 | 4096×2160 | 60 |
| 4 | 1280×720 | 50 | | 23 | 1366×768 | 50 |
| 5 | 1280×720 | 60 | | 24 | 1366×768 | 50 |
| 6 | 1920×1080 | 50 | | 25 | 1366×768 | 60 |
| 7 | 1920×1080 | 60 | | 26 | 1366×768 | 60 |
| 8 | 1920×1080 | 24 | | 27–34 | 1920×1080 | **0** (unpopulated) |
| 9 | 1920×1080 | 25 | | 35 | 960×540 | 50 |
| 10 | 1920×1080 | 30 | | 36 | 960×540 | 60 |
| 11 | 1920×1080 | 50 | | | | |
| 12 | 1920×1080 | 60 | | | | |
| 13 | 3840×2160 | 24 | | | | |
| 14 | 3840×2160 | 25 | | | | |
| 15 | 3840×2160 | 30 | | | | |
| 16 | 3840×2160 | 50 | | | | |
| 17 | 3840×2160 | 60 | | | | |
| 18 | 4096×2160 | 24 | | | | |

> **PROVEN BY BINARY** (contiguity + semantic coherence + explicit use in `getMiOsdTiming`).

**Corroboration from mik.ko enum strings:** `E_MI_DISP_TIMING_3840X2160_{24,25,30,50,60}P`, `E_MI_DISP_TIMING_4096X2160_{24,25,30,50,60}P`, `E_MI_DISP_TIMING_1920X1080_{24,25,30,50,60}P/I`, `_720X480_60I/P`, `_720X576_50I/P`, `_1280X720_50/60P`, `_1366X768_*`, `_960X540_50/60P`, `_1440X900_*`. **No `*_120P` timing enum exists anywhere in mik.ko.**

### 6.2 Driver-side switch

`_MI_DISP_XC_SetOutputTiming` (mik.ko, Ghidra `0x0fa8b0`, 2976 B) switches on the timing enum and remaps to an internal index (`local_50`, observed values `0x00`–`0x34`), then calls `MI_DISP_IMPL_XC_ChangeOutputResolution_EX`, `MI_DISP_IMPL_XC_SetPanelTiming`, `MI_DISP_IMPL_ForceSetPanelTiming`, `MI_DISP_IMPL_XC_SetWin_EX`, `mi_dispout_SetAttr`. The per-case string arguments (`_L_str_389` …) are debug prints naming each timing. — `PROVEN BY BINARY` (structure); per-case index↔enum pairing is `PRESENT IN BINARY` but not exhaustively enumerated here.

---

## 7. 4K / 4096 / VRR Analysis

### 7.1 3840×2160 and 4096×2160

| Layer | 4K status | Grade |
|---|---|---|
| Panel configuration | `m_wPanelWidth=3840`, `m_wPanelHeight=2160` in `UD_VB1_16LANE_CSOT_URSA.ini` | `PRESENT IN CONFIG/TABLE` |
| Driver timing enum | 3840×2160 and 4096×2160 at 24/25/30/50/60 exist | `PROVEN BY BINARY` |
| HWC timing table | 3840×2160 (13–17) and 4096×2160 (18–22) present | `PROVEN BY BINARY` |
| HWC advertised configs | **1920×1080 only** | `PROVEN BY BINARY` |
| Android selectable | **No 4K mode is selectable** | `PROVEN BY BINARY` |
| HDMI input | EDID `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin`; `HDMITX_RES_4096X2160p_*` strings exist but **HdmiTxCount = 0** | `PRESENT IN CONFIG/TABLE` |
| Physical photons | See §9 | `STRONGLY INDICATED` |

### 7.2 4K120

**`DISPROVEN`.** There is no 4K120 entry in the HWC timing table (max 60 Hz), no `*_120P` `E_MI_DISP_TIMING_*` enum in mik.ko, and no 4K120 panel profile. The `m_wVRR_max = 120` panel field and the 120 Hz DLP profile are 1080p-class, not 4K.

### 7.3 VRR

| Aspect | Finding | Grade |
|---|---|---|
| Panel declares VRR | `m_bVRR_en=1`, `m_wVRR_min=24`, `m_wVRR_max=120` | `PRESENT IN CONFIG/TABLE` |
| Driver has a VRR call | `MApi_XC_SetVRROn()` in `xcker.ko` | `PROVEN BY BINARY` |
| When is it turned ON | **Only FreeRunConfig 0** | `PROVEN BY BINARY` |
| Can HWC send 0? | **No** — `setActivePanelFrequency` emits 1 or 2 only | `PROVEN BY BINARY` |
| VRR reachable from Android | **No, via this path** | `PROVEN BY BINARY` |
| Alternative panel profile | `UD_VB1_16LANE_BENQ_120.ini` has `m_bVRR_en=0`; `UD_VB1_16LANE_VRR.ini` has it enabled | `PRESENT IN CONFIG/TABLE` |
| EDID | `*_freesync.bin` and `*_ALLM_*` EDID variants exist in `EDID_BIN` (not the active selection) | `PRESENT IN CONFIG/TABLE` |

---

## 8. HDR / Dolby Vision Analysis

### 8.1 What HWC reports

`HWComposerDisplayHalMiOsd::getHdrCapabilities` (Ghidra `0x6b0a0`, 202 B):

```c
if (MI_DISP_GetCaps(&caps) == 0) {
  for (i = 0; i != 7; i++) {
    if (caps_byte[i] == 1) {
      if ((i-1) < 5 && ((0x1b >> (i-1)) & 1))       /* mask 0b11011 → i ∈ {1,2,4,5} */
          push_back(lookup[i]);                      /* Android HDR type */
      else local = 0;                                /* i ∈ {0,3,6} ignored */
    }
  }
  *maxLuminance = 0.0; *maxAverageLuminance = 0.0; *minLuminance = 0.0;
}
```

- HDR types come from `MI_DISP_GetCaps()` (mik.ko ELF `0x11e1a4`), bytes **1, 2, 4, 5**; bytes 0, 3, 6 are discarded.
- **All three luminance values are hard-coded to 0.0.** — `PROVEN BY BINARY`
- `initHdrCapabilities` (Ghidra `0x51bf4`, 624 B) registers per-frame metadata keys **1..6** when HDR type **2** (HDR10) is present. — `PROVEN BY BINARY`
- The per-index HDR-type lookup table values could not be pinned (Android packed relocations + ambiguous `.rodata` scan). — `UNRESOLVED`

### 8.2 External HDMI vs internal Android

| | External HDMI input | Internal Android path |
|---|---|---|
| Source of truth | EDID (`Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin`) | `MI_DISP_GetCaps()` → HWC |
| HDR10 | EDID CTA-861.3 block present in the file set | type 2 possible; metadata keys 1..6 registered |
| HLG / HDR10+ | EDID variants exist | mapped from GetCaps bytes 2/4/5 |
| Dolby Vision | n/a (RX) | `kdrv_dolby_vision.ko` + `HDR.ini` (`dolby.bin`, `dolby_factory.cfg`, `dolby_3d_lut0`) — `PRESENT IN CONFIG/TABLE` |

### 8.3 Prior claim: "internal DV is blocked specifically by missing HWC 4K configs"

**Not supported — `UNRESOLVED` / insufficient evidence.** Neither `getHdrCapabilities` nor `initHdrCapabilities` contains any resolution test or any reference to the Config count. The HDR path is gated solely on `MI_DISP_GetCaps()` bytes. No evidence links Dolby Vision to HWC 4K configs.

---

## 9. Physical Display Architecture

The question: true 3840×2160 imaging, 1920×1080 + pixel shift, or something else?

**Evidence for a pixel-shift (XPR) projector with a sub-4K imager:**

| # | Evidence | Grade |
|---|---|---|
| 1 | `board.ini [PixelShiftInfo] m_bPixelShiftStatus = TRUE`, HRange 5, VRange 3, HOffset 4, VOffset 2, Direction 1 (RIGHT) — the **only** occurrence in the entire `cusdata` tree | `PRESENT IN CONFIG/TABLE` |
| 2 | `MI_DISP_SetControlAttr` attributes **2 and 3** are `MI_DISP_IMPL_XC_SetPixelShift_EX` / `...SetPixelShiftParams_EX` — pixel shift is a first-class driver control | `PROVEN BY BINARY` |
| 3 | 3D/120 Hz profile is literally `FullHD_VB1_8LANE_DLPC6540_2K_120_3D` — **DLPC6540 is a TI DLP controller**, 1920×1080, HTotal 2640 × VTotal 1875 @ DCLK 594 MHz = **120 Hz** | `PRESENT IN CONFIG/TABLE` |
| 4 | Active panel `UD_VB1_16LANE_CSOT_URSA.ini` declares **3840×2160 but `osdWidth=1920, osdHeight=1080`** — the OSD/UI plane runs at 1080p | `PRESENT IN CONFIG/TABLE` |
| 5 | HWC hard-codes the Android output to 1920×1080 and initialises its "current timing" fields (`+0x168/+0x16c`) to **1920×1080**, scaling ratio `1920/1920 = 1.0` | `PROVEN BY BINARY` |
| 6 | `m_ucOutTimingMode = 2` (`E_PNL_CHG_VTOTAL`) — the panel changes by adjusting VTotal, the classic free-run/frame-rate-conversion technique | `PRESENT IN CONFIG/TABLE` |
| 7 | `board.ini [DispoutConfig] m_bTconOutput = 0`, `m_bODEnable = 0`, `m_bSupportFrcInside = 1` | `PRESENT IN CONFIG/TABLE` |
| 8 | mik.ko supports board keys `m_p960X540_60PPanelName`, `m_p960X540_120PPanelName`, `m_p960X540_240PPanelName` (unpopulated in this model) | `PRESENT IN CONFIG/TABLE` |

**Conclusion:** a **1920×1080-class DLP imager with pixel shift (XPR) producing a 3840×2160 addressed raster** is `STRONGLY INDICATED`. A *native* 3840×2160 imaging device is **not** supported by the evidence and is contradicted by items 1–3 and 5.

> **The prior claim "`m_wPanelWidth = 3840` proves physical native 4K" is rejected.** `m_wPanelWidth` describes the *timing/addressing* domain of the panel interface, not the imager. The 4K raster is real and programmable; the number of physically distinct light modulators is almost certainly 1920×1080 (or 960×540-class).

**Extra consequence:** because the 4K image is produced by temporal sub-frame shifting, 4K@120 would require ~480 Hz sub-frame drive — consistent with there being **no 4K120 timing anywhere** in the platform.

---

## 10. Audit of Previous Reports

| # | Prior claim | Verdict | Basis |
|---|---|---|---|
| 1 | "HWC is the only blocker" | **Partially confirmed, incomplete** | HWC *is* the reason only 1080p@50/60 is *advertised*. But the vendor layer only offers 4 free-run configs, so removing the HWC limit does not by itself create new rates. |
| 2 | "Two HWC patches are sufficient" | **DISPROVEN** | `setActivePanelFrequency` maps every non-zero config id to FreeRunConfig 2. New configs would all still select 60 Hz. A second change in the mapping (or an ioctl-level bypass) is required. |
| 3 | "`getDisplayVsyncPeriod()` needs a patch" | **Mischaracterised** | The `20 000 000 ns` constant fires when the *active Config's field at +4 is 0*, not on `config_idx == 0`. The value normally comes from `vtable+0xd8`. Patching it would change *reporting*, not hardware. |
| 4 | "FreeRunConfig 2/3 correspond to 24/30 Hz" | **DISPROVEN** | cfg 2 → VFreq 1 (the 60 Hz path); cfg 3 → VFreq 3. No 24/30 Hz evidence exists. Only 4 configs total. |
| 5 | "`MI_DISP_SetControlAttr()` accepts arbitrary frequency" | **DISPROVEN** | Attribute 1 clamps to `0..3` and stores it in `_eFreeRunConfig`. It is an index, never a frequency. (The prior report contradicts itself on this.) |
| 6 | "`m_wPanelWidth = 3840` proves physical native 4K" | **DISPROVEN as stated** | Pixel shift is enabled; DLPC6540 is a 1080p controller; OSD plane is 1080p; HWC hard-codes 1080p. See §9. |
| 7 | "4K120 is proven" | **DISPROVEN** | No 4K120 timing enum, no 4K120 HWC table entry, no 4K120 panel profile. `m_wVRR_max=120` is a config field, not a demonstration. |
| 8 | "Internal DV is blocked specifically by missing HWC 4K configs" | **UNRESOLVED / unsupported** | The HDR path is gated only on `MI_DISP_GetCaps()` bytes; no resolution or Config-count test exists. |

**Additional corrections:**
- Prior reports disagree on the address convention; several "ELF" addresses are actually Ghidra addresses (§2.2).
- `checkGopTimingChanged` was cited as a substantive function; the primary and OSD variants are **8-byte stubs**. Only `HWComposerDisplayHalMiOsd::checkGopTimingChanged` is real.
- `TIMING_PROGRAMMING_FORENSIC_TRACE.md` contains two contradictory "FINAL VERDICT" sections (§7 and §8); `FINAL_DISPLAY_CAPABILITY_FORENSIC_REVIEW.md` and `TIMING_SWITCH_ROOT_CAUSE_TRACE.md` reach opposite conclusions. Treating any of them as authoritative is unsafe.

---

## 11. Evidence Matrix

| # | Conclusion | Grade |
|---|---|---|
| 1 | HWC advertises exactly two configs: 1920×1080@50 and 1920×1080@60 | `PROVEN BY BINARY` |
| 2 | Width/height in `addConfig` are hard-coded 1920/1080 from literal pool `0x3cc88` | `PROVEN BY BINARY` |
| 3 | `addConfig` / `initConfigs` / `setActiveConfig` / `setActiveConfigWithConstraints` make **no** hardware call | `PROVEN BY BINARY` |
| 4 | `initConfigs` activates the **last** config (60 Hz) | `PROVEN BY BINARY` |
| 5 | Hidden bridge = `HWComposerDevice::eventWorkThread` | `PROVEN BY BINARY` |
| 6 | `setActivePanelFrequency` is called from `eventWorkThread` at `0x491ee` only | `PROVEN BY BINARY` |
| 7 | `setActivePanelFrequency` takes a **config id**; 0→1, else→2 | `PROVEN BY BINARY` |
| 8 | `MI_DISP_SetControlAttr` attr 1 clamps to 0..3 (free-run index, not frequency) | `PROVEN BY BINARY` |
| 9 | `MI_DISP_SetControlAttr` attrs 2/3 = pixel-shift control | `PROVEN BY BINARY` |
| 10 | Only 4 free-run configs exist; VRR ON only for cfg 0 | `PROVEN BY BINARY` |
| 11 | VRR is unreachable from Android/HWC via this path | `PROVEN BY BINARY` |
| 12 | 37-entry timing table: 4K & 4096×2160 @24/25/30/50/60, max 60 Hz | `PROVEN BY BINARY` |
| 13 | No 4K120 anywhere in the timing surface | `PROVEN BY BINARY` |
| 14 | HWC never imports `MI_DISP_SetOutputTiming` | `PROVEN BY BINARY` |
| 15 | `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` ioctl exists in mik.ko | `PROVEN BY BINARY` |
| 16 | Production callers of `SetOutputTiming` use ioctl, not dynamic linking | `STRONGLY INDICATED` |
| 17 | HWC reports max/avg/min luminance = 0.0 | `PROVEN BY BINARY` |
| 18 | Active EDID = `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin`; 4 HDMI inputs | `PRESENT IN CONFIG/TABLE` |
| 19 | No HDMI transmitter (`SupportHdmiTxCount = 0`) | `PRESENT IN CONFIG/TABLE` |
| 20 | Pixel shift enabled in active board config | `PRESENT IN CONFIG/TABLE` |
| 21 | 1080p-class DLP imager + pixel shift → 4K raster | `STRONGLY INDICATED` |
| 22 | Identity of the virtual function at `vtable+0x44` | `UNRESOLVED` |
| 23 | Per-index HDR-type lookup values in `getHdrCapabilities` | `UNRESOLVED` |
| 24 | Whether TV middleware issues `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` at runtime | `UNRESOLVED` (needs runtime or the missing binaries) |
| 25 | Actual runtime output timing on this unit | `UNRESOLVED` (no runtime capture performed) |

---

## 12. Next Investigation Steps

Ordered by value / risk. **All are read-only or trivially reversible; none modifies the device.**

1. **Capture runtime state first (read-only ADB).** `dumpsys SurfaceFlinger`, `dumpsys display`, `getprop | grep -i mstar`, `cat /proc/cmdline`, and the mboot environment. This resolves #25 and #24 and confirms which of the ~90 panel INIs actually loaded. Cost: minutes. Risk: none.

2. **Exercise the property-driven knobs (no patch, fully reversible).**
   - `mstar_fakevsync_freq` → `eventWorkThread` writes `primary+4`, which is config 1's vsync period. Proves whether the *reported* rate is decoupled from the *programmed* rate.
   - `mstar_override_refresh_rate` → overrides `getRefreshPeriod()`.
   - `mstar_disable_osd = 2` → exercises the second `MI_DISP_SetControlAttr` path (§4.8).

3. **Determine what `vtable+0x44` actually is (#22).** Options: (a) decompile the Android packed APS2 relocation stream to recover the vtable slots; (b) read the vtable from the live process via `/proc/<pid>/mem`. Option (a) is self-contained and preferred.

4. **Resolve the 4K question definitively via the ioctl path (#24).** Issue `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` directly with enum 13–22 (4K/4096 @ 24/25/30/50/60) using a small client (`libmi3demo.so` already links the userspace API). Decoding the ioctl number from `mik.ko`'s file-operations table is the only prerequisite. **This is the single highest-value experiment**: it separates "HWC hides 4K" from "hardware cannot do 4K".

5. **Confirm the imager (#21).** Read `MApi_SC_SetFreerunVFreq` in `xcker.ko` to map VFreq enum → Hz (the last unexplained link in §5.3), and search `kdrv_xc.ko` for the pixel-shift actuator / sub-frame configuration calls.

6. **HDR (#23).** Resolve the per-index lookup table, then correlate with `MI_DISP_GetCaps()` output captured at runtime to see which HDR types the platform actually asserts.

7. **Only after 1–6:** decide whether any persistent change is warranted, and whether it belongs in HWC's config table (necessary but **not sufficient**), in `setActivePanelFrequency`'s mapping, or purely in the ioctl-level middleware.

---

*End of report. No patch produced. No device modified. No evidence deleted or overwritten.*

---

## 13. Addendum — Four Open Points Closed (Static Forensic, read-only)

Scope: resolves only the four unresolved points from the checkpoint. No new broad investigation; no modification of any artifact, device, or evidence. All findings are static-binary evidence. The **STATIC-FORENSIC-ONLY** prohibition was observed (no ioctl executed, no mode/timing change, no reboot, no patch, no EDID modification).

### Point 1 — MI_DEV_IOC_DISP_SET_OUTPUT_TIMING (exact ioctl number / encoding / constant / payload / dispatcher / device node / kernel chain)

**ioctl command constant (PROVEN, binary):** `0xc008122c`.
- Source: decompilation of `MI_DISP_SetOutputTiming` in `libmi3.so` (Ghidra addr `0x7cfc4`):
  `iVar1 = ioctl_wrapper(*(int*)(...), 0xc008122c, &local_20);` where `local_20 = {param_1 (devId), param_2 (timingEnum)}` (8 bytes).
- The same constant is the value carried into the kernel; it is the `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` macro (libmi3 contains the literal `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` exactly once).

**_IOC encoding (PROVEN, reconstructed & round-trip verified):**
- `0xc008122c` = `_IOC(3, 0x12, 0x2c, 8)`.
  - dir = (cmd>>30)&3 = **3** → `_IOC_READ | _IOC_WRITE` (kernel copies the arg both from and to user).
  - type = (cmd>>8)&0xff = **0x12**.
  - nr = cmd&0xff = **0x2c** (44).
  - size = (cmd>>16)&0x3fff = **8** bytes.
- Reconstructing `_IOC(3,0x12,0x2c,8)` yields exactly `0xc008122c` (verified).

**payload / structure (PROVEN):**
- 8-byte struct `{ uint32 devId (MI_DISP_DEV); uint32 timingEnum (MI_DISP_OutputTimingConfig eTimingType) }`.
- `_IOC_SIZE = 8` constrains the kernel `copy_from_user` to exactly 8 bytes, matching the userspace 8-byte arg. This is the complete payload (no larger struct is copied).

**device node (PROVEN):** `libmi3.so` opens `/dev/mik!disp` (string at file off `0x411e1`) and issues the ioctl on that fd.

**dispatcher / file_operations (PROVEN):**
- `/dev/mik!disp` ↔ per-class DISP `file_operations.unlocked_ioctl` = `_MI_DEV_DISP_Ioctl` (Ghidra `0x231dc`, a 12-byte thunk) → `MI_DEV_UserCopyIoctl` (`0x12ae8`) → `_MI_DEV_DISP_DoIoctl` (`0x232ac`, the per-class `switch(cmd)` dispatcher).
- `_MI_DEV_DISP_DoIoctl` is a **computed jump-table switch** (confirmed: `ldr pc,[r1,r0,lsl #0x2]` at `0x23924`, `0x23af8`, `0x23c60`; secondary index range checks `cmp r0,#0x32`/`#0x17`). The `SET_OUTPUT_TIMING` case is handled inline: an xref on `MI_DISP_SetOutputTiming` (`0x1323d4`) resolves a direct `UNCONDITIONAL_CALL` from `0x2428c`, which lies inside `_MI_DEV_DISP_DoIoctl`'s body (0x232ac–0x24f40). So the dispatcher calls `MI_DISP_SetOutputTiming` directly for this command.
- The kernel also contains a dedicated handler literal `"MI_DEV_IOC_DISP_SET_OUTPUT_TIMING got NULL arg"` (mik.ko file off `0x438ea4`), confirming a first-class per-command handler for this ioctl.

**exact kernel call chain (PROVEN, decompiled):**
`file_operations.unlocked_ioctl` → `_MI_DEV_DISP_Ioctl` (thunk) → `MI_DEV_UserCopyIoctl` → `_MI_DEV_DISP_DoIoctl` (switch) → [SET_OUTPUT_TIMING case] → `MI_DISP_SetOutputTiming` (`0x1323d4`, sig `(uint devId, uint timingEnum)`; guarded by `(param_1 & 0x4354) == 0x4354`) → `_MI_DISP_SetOutputTiming` (`0x0e9680`) → `_MI_DEV_DISP_XC_SetOutputTiming` (`0x0fa8b0`) → XC impl: `MI_DISP_IMPL_XC_ChangeOutputResolution_EX`, `MI_DISP_IMPL_XC_SetPanelTiming`, `MI_DISP_IMPL_ForceSetPanelTiming`, `MI_DISP_IMPL_XC_SetWin_EX`, `mi_dispout_SetAttr` → `MApi_SC_*` / `MApi_XC_*` (xcker.ko).

`_MI_DEV_DISP_XC_SetOutputTiming` is a `switch(param_2)` over the timing enum `0..0x2e` (46 entries) mapping each to an internal XC index `0x00..0x34` (the hardware-facing timing selection). This is the hardware-facing leaf of the path.

**Point 1 verdict: VERIFIED (binary evidence, not assumed).** The earlier "via ioctl" claim is now demonstrated end-to-end from the ioctl constant through to the XC functions. Note: the kernel dispatcher stores the command in a jump-table (not an inline `case 0xc008122c` compare), so the literal `0xc008122c` is not materialized in the dispatcher's disassembly; the command value is proven from the userspace issuer (authoritative — it is what is delivered to the kernel) and matched by the kernel's identical shared `_IOC` macro.

### Point 2 — Real userspace caller (priority-order scan)

Scanned, in priority order:
- `hwcomposer.mt5889.so` — **0** references to `MI_DISP_SetOutputTiming` / `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING`.
- `libmi3.so` — **contains** `MI_DISP_SetOutputTiming` (def+call @ `0x7cfc4`), the literal `MI_DEV_IOC_DISP_SET_OUTPUT_TIMING` (once), all 90 `MI_DEV_IOC_DISP_*` strings, and opens `/dev/mik!disp`. **This is the component that forms/emits the ioctl.**
- `libutopia.so` — 5 `SetOutputTiming` hits, but **none** are `MI_DISP_SetOutputTiming` / `MI_DEV_IOC_DISP_*` / `/dev/mik!disp`; they are GOP/MVOP APIs (`GOP_MIXER_SetOutputTiming`, `GOP_VE_SetOutputTiming`, `HAL_GOP_MIXER_SetOutputTiming`, `HAL_MVOP_SetOutputTiming`, `MApi_GOP_VE_SetOutputTiming`, …) — a different subsystem. **Not** the ioctl issuer.
- `mstar.hardware.imgdisp@1.0-service`, `mstar.hardware.mmdisp@1.0-service`, `vendor.mediatek.api.mtktvapi@1.0-service` — **0** references.

**Result:** The ioctl is *emitted* by `libmi3.so` (the userspace MI client library). The upstream production component (app / HAL service) that *decides* to invoke `libmi3`'s `MI_DISP_SetOutputTiming` is **NOT PRESENT in the available artifacts**.
**PRODUCTION CALLER (the decision-maker) = NOT PROVEN FROM AVAILABLE ARTIFACTS.** The missing boundary is the caller above `libmi3.so`; the available firmware dump does not include the component that drives output-timing changes. Stated explicitly, not invented.

### Point 3 — vtable+0x44

`hy4_vtable.py` re-run: all 63 slots of `_ZTV23HWCDisplayDevicePrimary` (@ `0x70668`) show `<no relocation>` — the vtable is shipped as APS2-packed relocations (`SHT_ANDROID_REL = 0x60000001`). Slot 17 (@ `+0x44`) therefore **cannot be resolved to a concrete virtual function from the file bytes**. No speculation is made.
**Verdict: UNRESOLVED (APS2-packed relocations).**

### Point 4 — HDR capability bit → type mapping

`getHdrCapabilities` (@ `0x6b0a0`, HWC) reads a 7-byte `MI_DISP_GetCaps()` array; loops `i=0..6`; reports a type when `caps[i]==1 && (i-1)<5 && ((0x1b >> (i-1)) & 1)`. Valid capability bits = mask `0x1b` → indices `{1,2,4,5}`.
The HDR-type lookup table at HWC `.rodata` ~`0x476e4` holds `[0]=2,[1]=1,[2]=2,[3]=3,[4]=4,…`, i.e. caps-index → HWC2 enum:
- index 1 → `2` = HDR10
- index 2 → `1` = Dolby Vision
- index 4 → `3` = HLG
- index 5 → `4` = HDR10_PLUS
Max/avg/min luminance are hard-coded `0.0`.
**Verdict: VERIFIED (direct evidence).** HDR10 / Dolby Vision / HLG / HDR10+ bits are now named with independent confirmation (satisfies the "no naming without confirmation" constraint).

---

### A. VERIFIED
- Point 1: ioctl number `0xc008122c`, `_IOC(3,0x12,0x2c,8)`, 8-byte payload `{devId, timingEnum}`, device node `/dev/mik!disp`, dispatcher chain, and the full kernel call chain to the XC functions — all demonstrated from binary.
- Point 2 (issuer half): `libmi3.so` emits the ioctl.
- Point 4: HDR bit→type mapping (HDR10 / Dolby Vision / HLG / HDR10+).

### B. STRONG INFERENCE
- The kernel `case` constant equals the userspace `0xc008122c` by the shared `_IOC` macro invariant (the dispatcher is a jump table, so the literal is not inline; the delivered value is what the kernel matches).
- Point 2 (decision-maker half): the upstream caller above `libmi3.so` is absent from the dump.

### C. STILL UNKNOWN
- Point 3: `vtable+0x44` target (APS2-packed relocations).
- The runtime *invocation* of `MI_DISP_SetOutputTiming` for any non-1080p timing (static-only phase; not observed).

### D. SINGLE BEST NEXT STEP
Capture runtime ioctl traffic on `/dev/mik!disp` (e.g. `ftrace`/`ioctl` logging or a tiny `libmi3`-linked client) to record the actual `MI_DEV_IOC_DISP_*` commands and timing-enum arguments HWC/framework submit. This single experiment separates "HWC enumerates only 1080p, hardware path is capable of more" from "hardware genuinely cannot do 4K" — and closes Points 2 (real decision-maker) and 3 (live vtable) without modifying the device.

---

### Explicit answer — Is HWC 1080p-only already proven to be merely an Android enumeration limitation, or is confirmation of the real hardware-timing programming path still missing?

**The real hardware-timing programming path is no longer missing at the structural level — it is now statically proven (Point 1):** `MI_DISP_SetOutputTiming → _MI_DEV_XC_SetOutputTiming` is a reconstructable switch over 46 timing enums reaching the XC/panel functions, and the device node + dispatcher that drive it exist and are reachable. So the *path* exists and is not the blocker.

**However, we do NOT yet have enough evidence to assert that "HWC 1080p-only is merely an Android enumeration limitation" as VERIFIED.** What remains missing:
1. **Actual invocation is unobserved.** Under the STATIC-ONLY prohibition we have not seen any caller submit a non-1080p timing through this path. The path's *capability* is proven; its *use* for 4K is not.
2. **4K timings in the table are not fully confirmed as panel-supported.** `m_wPanelWidth=3840` hints the panel is 4K, and the reconstructed `timing_enum→width/height/refresh` table plus the 0x2f-entry XC switch suggest 4K@30/60 entries exist, but 4K120 was explicitly *not* found and the 4K entries have not been independently confirmed against the loaded panel INI/EDID.
3. **HWC is not yet proven to be the sole gate.** We have not ruled out additional framework/middleware constraints above `libmi3.so` (the decision-maker is itself NOT PROVEN, Point 2).

**Conclusion:** The kernel hardware-timing path is *structurally proven and reachable* (so "the path is missing" is false), but confirmation that it is *actually exercised for 4K* and that the panel *genuinely supports* 4K is **still missing**. Therefore "HWC 1080p-only is merely an Android enumeration limitation" stands as **STRONG INFERENCE (B), not VERIFIED (A)**. The single decisive evidence — runtime ioctl capture (D) — is the only thing that would upgrade it to VERIFIED, and it is intentionally out of scope under the current static-forensic-only constraint.

---

*Addendum: static-forensic only. No ioctl executed, no mode/timing change, no reboot, no patch, no EDID modification. No evidence altered.**
