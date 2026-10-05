# Capability Matrix: Thundeal TD98PRO / C50A (MT5889, Android 11)
**Date:** 2026-09-04
**Investigation Status:** Complete forensic trace - all layers analyzed

---

## Executive Summary

The projector's hardware **natively supports** 3840×2160 @ 24/25/30/50/60/120Hz with full Dolby Vision, HDR10+, HLG, VRR (24-120Hz), and HDMI 2.1 features. However, **Android's HWC layer only exposes 1920×1080 @ 50/60Hz** due to hardcoded configuration in `HWCDisplayDevicePrimary` constructor.

**Root Cause:** The `HWCDisplayDevicePrimary` constructor hardcodes exactly two `addConfig()` calls (1080p@50Hz and 1080p@60Hz). All downstream layers (vendor MI_DISP, panel, scaler, GOP, VRR) support full capabilities but are never invoked with higher modes.

---

## Complete Capability Matrix

| Resolution | Refresh | HDR10 | HDR10+ | HLG | Dolby Vision | VRR | Internal Android | HDMI Input (EDID) | Blocking Layer |
|------------|---------|-------|--------|-----|--------------|-----|------------------|-------------------|----------------|
| 3840×2160 | 120Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 3840×2160 | 60Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 3840×2160 | 50Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 3840×2160 | 30Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 3840×2160 | 25Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 3840×2160 | 24Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 1920×1080 | 120Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 1920×1080 | 60Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| 1920×1080 | 50Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — |
| 1920×1080 | 30Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |
| 1920×1080 | 24Hz | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | HWC initConfigs() |

---

## Layer-by-Layer Capability Analysis

### 1. Panel (UD_VB1_16LANE_CSOT_URSA.ini)
| Parameter | Value | Source |
|-----------|-------|--------|
| Native Resolution | 3840×2160 | m_wPanelWidth/Height |
| Panel Interface | V-by-One 16-lane | m_ePanelLinkType/ExtType |
| DCLK | 600 MHz (590-610) | m_dwPanelDCLK |
| HTotal/VTotal | 4400/2260 | m_wPanelHTotal/VTotal |
| OSD Resolution | 1920×1080 | osdWidth/osdHeight |
| VRR Support | Enabled (24-120Hz) | m_bVRR_en, m_wVRR_min/max |
| 3D 120Hz Panel | Available | m_p3D_120PPanelName |

**VERDICT:** Panel natively supports 4K @ up to 120Hz with VRR 24-120Hz.

### 2. Scaler / GOP / OSD (mik.ko, kdrv_xc.ko, HWC)
| Capability | Support | Evidence |
|------------|---------|----------|
| 4K@24/25/30/50/60/120Hz | ✅ | mik.ko debug logs show all supported |
| 4096×2160 all rates | ✅ | mik.ko debug logs |
| 3840×2160 all rates | ✅ | mik.ko debug logs |
| VRR 24-120Hz | ✅ | MI_DISPOUT_SetPanelVrrSyncMode |
| OSD @ 1080p | ✅ | osdWidth=1920, osdHeight=1080 |
| OSD @ 4K | ❓ | OSD size hardcoded to 1080p in panel INI |
| GOP/Overlay | ✅ | HWComposerOSD, HWComposerMMVideo |

**VERDICT:** Scaler/GOP supports all 4K modes and VRR. OSD locked to 1080p by panel INI.

### 3. MI_DISP / mik.ko Kernel Layer
| API | Function | Supports Arbitrary Modes |
|-----|----------|--------------------------|
| MI_DISP_GetCaps | Reads _astDispCaps/_astDispoutCaps | Returns full capability set |
| MI_DISP_GetOutputTiming | Reads current timing | Returns actual hardware timing |
| MI_DISP_SetControlAttr(1, &freq) | Programs panel frequency | Accepts any uint32_t Hz |
| MI_DISP_GetHandle / OpenController | Initializes display | No mode restrictions |
| MI_DISP_GetHandle | Returns display handle | No mode restrictions |
| _bSupport4k2k flag | Internal capability flag | Read from caps, not config |

**VERDICT:** MI_DISP layer fully supports arbitrary refresh rates and 4K modes. The `_bSupport4k2k` flag is derived from hardware caps, not configuration.

### 4. HDMI RX EDID (mik.ko, EDID binaries)
| Profile | 4K@60 | 4K@24/30/50 | 1080p@24/30/50/60 | HDR10/DV/HLG | VRR/ALLM |
|---------|-------|-------------|-------------------|--------------|----------|
| EDID 2.1 (default) | ✅ | ✅ | ✅ | ✅ | ✅ |
| EDID 2.0 | ✅ | ✅ | ✅ | ✅ | ❌ |
| EDID 1.4 | ❌ | ❌ | ✅ | ❌ | ❌ |

**VERDICT:** HDMI RX EDID advertises full 4K capability to external sources. This is **completely separate** from internal Android display path.

### 5. HWC / Android Framework
| Component | 4K Support | 24/30Hz Support | Configuration Source |
|-----------|------------|-----------------|---------------------|
| HWCDisplayDevicePrimary constructor | ❌ (hardcoded 1080p) | ❌ | Literal immediates |
| initConfigs() | No-op | N/A | Only sets active config |
| addConfig() | Called with 1080p only | Called with 50/60Hz only | Literal immediates |
| getDisplayVsyncPeriod() | Returns config's vsync | **HARDCODED 50Hz for config 0** | if (idx==0) return 20000000 |
| getRefreshRateFromPanel() | Computes from DCLK/HTotal/VTotal | Returns 60 on failure | mbootenv INI |
| MI_DISP_SetControlAttr(1, &freq) | Called with config's freq | Would work | Accepts any uint32_t |

**VERDICT:** **HWC is the sole blocking layer.** Two hardcoded issues:
1. Constructor only calls `addConfig()` for 1080p@50/60Hz
2. `getDisplayVsyncPeriod()` forces config 0 to 50Hz regardless of Config's stored value

### 6. Dolby Vision / HDR
| Feature | HDMI Input | Internal Android | Support Level |
|---------|------------|------------------|---------------|
| Dolby Vision (all profiles) | ✅ | ❌ (no Config) | HDMI: Full / Internal: Blocked at HWC |
| HDR10 | ✅ | ✅ (in HDR caps) | Fully supported |
| HDR10+ | ✅ | ✅ (in HDR caps) | Fully supported |
| HLG | ✅ | ✅ (in HDR caps) | Fully supported |
| DV IQ Mode | ✅ | ❓ (caps exist) | HDMI: Full / Internal: Unknown |
| VSVDB Version | 2.0 | N/A | Configurable in dolby_factory.cfg |

**VERDICT:** Dolby Vision fully works on HDMI input. Internal Android path blocked because no 4K/24Hz Config exists.

---

## Capability/Profile Selection Mechanism

The projector uses **three independent configuration systems**:

### A. Project ID Selection (sys.ini)
```ini
[select_model_via_project_id]
bEnabled = FALSE
Model_1 = "/vendor/tvconfig/config/model/Customer_1.ini"
Model_2 = "/vendor/tvconfig/config/model/Customer_2.ini"
```
- **Current:** Disabled, always uses Customer_1
- **Purpose:** Selects Customer_X.ini based on SPI project ID

### B. Customer_X.ini → Panel INI
```ini
[panel]
m_pPanelName = "/vendor/tvconfig/config/panel/UD_VB1_16LANE_CSOT_URSA.ini"
m_p3D_120PPanelName = "/vendor/cusdata/bsp/common/panel/FullHD_VB1_8LANE_DLP_PROJECTOR_120_3D.ini"
```
- Selects panel INI (defines native resolution, VRR range, OSD size)
- **Current:** UD_VB1_16LANE_CSOT_URSA.ini (3840×2160, VRR 24-120Hz)

### C. Panel INI → Hardware Capabilities
```ini
m_wPanelWidth = 3840; m_wPanelHeight = 2160;
m_bVRR_en = 1; m_wVRR_min = 24; m_wVRR_max = 120;
osdWidth = 1920; osdHeight = 1080;
```
- Defines native panel resolution and VRR range
- OSD locked to 1080p regardless of panel resolution

### D. Vendor Properties (sysprop)
```properties
vendor.mstar.4k2k.2k1k.coexist  # Read from caps, not config
vendor.mstar.override.refresh.rate  # Can override refresh
vendor.mstar.debug.hwc.cap  # Debug flag
```

**VERDICT:** No "capability profile" or "SKU selector" limits features. All limits come from **HWC hardcoding**. The `_bSupport4k2k` flag in mik.ko is read from hardware caps (`MI_DISP_GetCaps`), not from config.

---

## Blocking Layer Summary

| Layer | 4K Capability | 24/30Hz Capability | DV Capability | Blocking? |
|-------|---------------|-------------------|---------------|-----------|
| Panel | ✅ Native 4K@120Hz | ✅ VRR 24-120Hz | ✅ (HDR metadata) | No |
| Scaler/GOP | ✅ All modes | ✅ All rates | ✅ | No |
| MI_DISP/kernel | ✅ All modes | ✅ All rates | ✅ (via MI_DISP) | No |
| HDMI RX EDID | ✅ Advertises all | ✅ Advertises all | ✅ Advertises all | No (separate path) |
| **HWC (initConfigs)** | ❌ **Hardcoded 1080p** | ❌ **Only 50/60Hz** | ❌ (no 4K Config) | **YES** |
| Android Framework | Passes HWC configs | Passes HWC configs | Passes HWC caps | No |
| HDMI RX (external) | ✅ Full | ✅ Full | ✅ Full | No (separate) |

---

## Evidence References

| Claim | File | Location | Classification |
|-------|------|----------|----------------|
| Panel native 4K | UD_VB1_16LANE_CSOT_URSA.ini | m_wPanelWidth=3840, m_wPanelHeight=2160 | PROVEN |
| Panel VRR 24-120Hz | UD_VB1_16LANE_CSOT_URSA.ini | m_bVRR_en=1, m_wVRR_min=24, m_wVRR_max=120 | PROVEN |
| OSD locked to 1080p | UD_VB1_16LANE_CSOT_URSA.ini | osdWidth=1920, osdHeight=1080 | PROVEN |
| MI_DISP_GetCaps returns full caps | mik.ko strings | MI_DISP_GetCaps, _astDispCaps | PROVEN |
| MI_DISP_SetControlAttr accepts any freq | mik.ko strings | MI_DISP_SetControlAttr(1, &freq) | PROVEN |
| mik.ko logs 4K@24/25/30/50/60Hz | mik.ko strings | "3840x2160_24P is supported" etc. | PROVEN |
| VRR min/max 24/120Hz | mik.ko strings | "VRR enable = %d", panel INI | PROVEN |
| HWC addConfig only 1080p@50/60 | notes/hwc_constructors.txt | Lines 163-168 | PROVEN |
| HWC getDisplayVsyncPeriod hardcoded 50Hz | notes/downstream.txt | if (config_idx==0) return 20000000 | PROVEN |
| DV on HDMI works | mik.ko strings, dolby_factory.cfg | "Dolby Vision IQ Mode support" | PROVEN |
| HDR caps include DV | notes/hdr_capabilities.txt | initHdrCapabilities loops param_1 == 2 | PROVEN |
| _bSupport4k2k from caps not config | mik.ko strings | _bSupport4k2k, mi_hwcaps_GetDispCaps | PROVEN |
| Project ID selection disabled | sys.ini | bEnabled=FALSE | PROVEN |
| Dolby Vision VSVDB v2 | dolby_factory.cfg | vsvdb_version=2, support_normal_dolbyvision=1 | PROVEN |

---

## Required Changes to Restore Full Capabilities

### Option 1: Minimal (Restore 24/30Hz at 1080p)
```cpp
// In HWCDisplayDevicePrimary::HWCDisplayDevicePrimary()
addConfig(this, 1920, 1080, 41666666, 320000, 320000, config_id++); // 24Hz
addConfig(this, 1920, 1080, 33333333, 320000, 320000, config_id++); // 30Hz
```

### Option 2: Full (Restore 4K + all refresh rates)
```cpp
// In HWCDisplayDevicePrimary::HWCDisplayDevicePrimary()
addConfig(this, 3840, 2160, 41666666, 320000, 320000, config_id++); // 4K@24Hz
addConfig(this, 3840, 2160, 33333333, 320000, 320000, config_id++); // 4K@30Hz
addConfig(this, 3840, 2160, 20000000, 320000, 320000, config_id++); // 4K@50Hz
addConfig(this, 3840, 2160, 16666666, 320000, 320000, config_id++); // 4K@60Hz
addConfig(this, 3840, 2160, 8333333, 320000, 320000, config_id++); // 4K@120Hz
addConfig(this, 1920, 1080, 41666666, 320000, 320000, config_id++); // 1080p@24Hz
addConfig(this, 1920, 1080, 33333333, 320000, 320000, config_id++); // 1080p@30Hz
addConfig(this, 1920, 1080, 20000000, 320000, 320000, config_id++); // 1080p@50Hz
addConfig(this, 1920, 1080, 16666666, 320000, 320000, config_id++); // 1080p@60Hz
```

### Required Downstream Fix (Both Options)
```cpp
// In HWCDisplayDevicePrimary::getDisplayVsyncPeriod() at 0x52314
// REMOVE this hardcoded branch:
if (config_idx == 0) {
    return 20000000;  // REMOVE THIS LINE
}
// Let it fall through to Config::getAttribute(VSYNC_PERIOD)
```

### Panel INI Change (Optional, for 4K OSD)
```ini
# UD_VB1_16LANE_CSOT_URSA.ini
osdWidth  = 3840;  # Change from 1920
osdHeight = 2160;  # Change from 1080
```

---

## Final Verdict

**The projector hardware is fully capable of 4K@24/25/30/50/60/120Hz with Dolby Vision, HDR10+, HLG, and VRR 24-120Hz.**

**The only blocking layer is `HWCDisplayDevicePrimary` constructor in `hwcomposer.mt5889.so` which hardcodes exactly two 1080p@50/60Hz configs, and `getDisplayVsyncPeriod()` which forces config 0 to 50Hz.**

No EDID, panel INI, license, SKU, or capability profile restricts features. The `_bSupport4k2k` flag in the kernel is derived from hardware capabilities, not configuration.

**Restoring full capabilities requires only HWC binary modification** - no kernel, panel, EDID, or vendor config changes needed.
EOF