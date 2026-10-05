# Forensic EDID Analysis Report

**Total EDID Profiles Analyzed:** 60

## 1. Direction and Ownership Classification

A critical forensic distinction established by our firmware reverse engineering:

- **DIRECTION A: HDMI RX EDID (Projector Input Sink Presentation)**
  - **Files:** `/vendor/cusdata/bsp/common/EDID_BIN/*.bin`
  - **Loader:** `mik.ko` -> `_MI_SYS_CfgLoadHdmiEdidInfo` -> `MI_EXTIN_SetAttr` -> `MI_DISP_IMPL_XC_HDMIRx_SetEDID`
  - **Configuration:** `/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/edid_cfg.ini`
  - **Ownership:** Presented by the projector's MT5889 HDMI RX hardware to ANY EXTERNAL HDMI SOURCE (e.g. LibreELEC box, PC, Blu-ray player, gaming console).
  - **Role:** Tells the external source what modes the projector can RECEIVE over HDMI.

- **DIRECTION B: Internal Display Panel Timing Configuration (Internal Engine)**
  - **Files:** `/vendor/cusdata/bsp/common/panel/UD_VB1_16LANE_CSOT_URSA.ini`
  - **Loader:** `kdrv_xc.ko` / `mstar_fbdev_mi.ko` / `hwcomposer.mt5889.so`
  - **Configuration:** `/vendor/tvconfig/config/sys.ini` -> `/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/model/Customer_1.ini`
  - **Ownership:** Hardcoded internal panel timings driving the internal optical engine/LCD via V-by-One / LVDS.
  - **Role:** Drives Android UI and internal media playback.

- **DIRECTION C: Downstream Sink EDID (HDMI TX / ARC)**
  - **Driver:** `_MApi_HDMITx_GetEDIDData`
  - **Role:** Used only when transmitting audio/video to an external AVR or soundbar.

## 2. In-Depth Analysis of Active HDMI RX EDID Profiles

### Profile: `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin`

- **Size:** 256 bytes
- **MD5:** `1352c0c360439adcf93946a7c800d2fb`
- **SHA256:** `5dde70d85a0eada4954d6654254294b49e5faa724b4ca0546adc4dfeae26f48d`
- **Manufacturer / Product:** `MST` (Product Code: 48)
- **EDID Version:** 1.3, Extensions: 1

#### Base Block Detailed Timings (DTDs):
- 3840x2160 @ 60.00Hz (PCLK=594.00MHz)
- 1920x1080 @ 60.00Hz (PCLK=148.50MHz)

#### CTA-861 Extension Capabilities:
- Basic Audio: True, YCbCr 4:4:4: True, YCbCr 4:2:2: True

**Supported CEA/CTA VIC Modes (Video Data Block):**

| VIC | Native | Resolution / Refresh Rate |
| --- | --- | --- |
| 5 | No | 1920x1080i @ 59.94/60Hz (16:9) |
| 4 | Yes (Preferred) | 1280x720p @ 59.94/60Hz (16:9) |
| 3 | No | 720x480p @ 59.94/60Hz (16:9) |
| 1 | No | 640x480p @ 59.94/60Hz (4:3) |
| 18 | No | 720x576p @ 50Hz (16:9) |
| 19 | No | 1280x720p @ 50Hz (16:9) |
| 20 | No | 1920x1080i @ 50Hz (16:9) |
| 7 | No | 720(1440)x480i @ 59.94/60Hz (16:9) |
| 16 | Yes (Preferred) | 1920x1080p @ 59.94/60Hz (16:9) |
| 31 | No | 1920x1080p @ 50Hz (16:9) |
| 32 | No | 1920x1080p @ 23.976/24Hz (16:9) |
| 34 | No | 1920x1080p @ 29.97/30Hz (16:9) |
| 93 | No | 3840x2160p @ 23.976/24Hz (16:9) |
| 95 | No | 3840x2160p @ 29.97/30Hz (16:9) |
| 96 | No | 3840x2160p @ 50Hz (16:9) |
| 97 | No | 3840x2160p @ 59.94/60Hz (16:9) |
| 98 | No | 4096x2160p @ 23.976/24Hz (256:135) |
| 100 | No | 4096x2160p @ 29.97/30Hz (256:135) |
| 101 | No | 4096x2160p @ 50Hz (256:135) |
| 102 | No | 4096x2160p @ 59.94/60Hz (256:135) |
| 94 | No | 3840x2160p @ 25Hz (16:9) |
| 99 | No | 4096x2160p @ 25Hz (256:135) |
| 2 | No | 720x480p @ 59.94/60Hz (4:3) |
| 6 | No | 720(1440)x480i @ 59.94/60Hz (4:3) |
| 33 | No | 1920x1080p @ 25Hz (16:9) |
| 63 | No | 1920x1080p @ 119.88/120Hz (16:9) |
| 64 | No | 1920x1080p @ 100Hz (16:9) |

**HDMI 1.4 Vendor-Specific Data Block (OUI 00-0c-03):**
- Physical Address: `1.0.0.0`
- Max TMDS Clock: 160 MHz

**HDMI Forum VSDB (HDMI 2.0/2.1, OUI c4-5d-d8):**
- Version: 1
- Max TMDS Rate: 600 MHz
- SCDC Present: True

**HDR Static Metadata (CTA-861.3):**
- Supported EOTFs: Traditional gamma - SDR, Traditional gamma - HDR, SMPTE ST 2084 (HDR10), Hybrid Log-Gamma (HLG)

**Audio Capabilities (Short Audio Descriptors):**

- **LPCM**: 2 channels, sample rates: [32, 44.1, 48, 88.2, 96, 176.4, 192] kHz (16bit/20bit/24bit)
- **AC-3 (Dolby Digital)**: 6 channels, sample rates: [32, 44.1, 48] kHz (max 640 kbps)
- **E-AC-3 (Dolby Digital Plus)**: 2 channels, sample rates: [44.1, 48] kHz (byte3=0x00)

**Extension Detailed Timing Descriptors:**
- 720x480 @ 59.94Hz (PCLK=27.00MHz)

---

### Profile: `Mstar_EDID1_v2.0_4K_2K_3D_HDR.bin`

- **Size:** 256 bytes
- **MD5:** `dd40b3bdc990348841fe3f03d49587c5`
- **SHA256:** `83b11ec3440f3f097f798c9d4c484620e58c4355ec4ee214f711e21239f815b2`
- **Manufacturer / Product:** `MST` (Product Code: 48)
- **EDID Version:** 1.3, Extensions: 1

#### Base Block Detailed Timings (DTDs):
- 3840x2160 @ 60.00Hz (PCLK=594.00MHz)
- 1920x1080 @ 60.00Hz (PCLK=148.50MHz)

#### CTA-861 Extension Capabilities:
- Basic Audio: True, YCbCr 4:4:4: True, YCbCr 4:2:2: True

**Supported CEA/CTA VIC Modes (Video Data Block):**

| VIC | Native | Resolution / Refresh Rate |
| --- | --- | --- |
| 5 | No | 1920x1080i @ 59.94/60Hz (16:9) |
| 4 | Yes (Preferred) | 1280x720p @ 59.94/60Hz (16:9) |
| 3 | No | 720x480p @ 59.94/60Hz (16:9) |
| 1 | No | 640x480p @ 59.94/60Hz (4:3) |
| 18 | No | 720x576p @ 50Hz (16:9) |
| 19 | No | 1280x720p @ 50Hz (16:9) |
| 20 | No | 1920x1080i @ 50Hz (16:9) |
| 22 | No | 720(1440)x576i @ 50Hz (16:9) |
| 7 | No | 720(1440)x480i @ 59.94/60Hz (16:9) |
| 16 | Yes (Preferred) | 1920x1080p @ 59.94/60Hz (16:9) |
| 31 | No | 1920x1080p @ 50Hz (16:9) |
| 32 | No | 1920x1080p @ 23.976/24Hz (16:9) |
| 34 | No | 1920x1080p @ 29.97/30Hz (16:9) |
| 93 | No | 3840x2160p @ 23.976/24Hz (16:9) |
| 95 | No | 3840x2160p @ 29.97/30Hz (16:9) |
| 96 | No | 3840x2160p @ 50Hz (16:9) |
| 97 | No | 3840x2160p @ 59.94/60Hz (16:9) |
| 98 | No | 4096x2160p @ 23.976/24Hz (256:135) |
| 100 | No | 4096x2160p @ 29.97/30Hz (256:135) |
| 101 | No | 4096x2160p @ 50Hz (256:135) |
| 102 | No | 4096x2160p @ 59.94/60Hz (256:135) |
| 94 | No | 3840x2160p @ 25Hz (16:9) |
| 99 | No | 4096x2160p @ 25Hz (256:135) |
| 2 | No | 720x480p @ 59.94/60Hz (4:3) |
| 6 | No | 720(1440)x480i @ 59.94/60Hz (4:3) |
| 17 | No | 720x576p @ 50Hz (4:3) |
| 21 | No | 720(1440)x576i @ 50Hz (4:3) |

**HDMI 1.4 Vendor-Specific Data Block (OUI 00-0c-03):**
- Physical Address: `1.0.0.0`
- Max TMDS Clock: 160 MHz

**HDMI Forum VSDB (HDMI 2.0/2.1, OUI c4-5d-d8):**
- Version: 1
- Max TMDS Rate: 600 MHz
- SCDC Present: True

**HDR Static Metadata (CTA-861.3):**
- Supported EOTFs: Traditional gamma - SDR, Traditional gamma - HDR, SMPTE ST 2084 (HDR10), Hybrid Log-Gamma (HLG)

**Audio Capabilities (Short Audio Descriptors):**

- **LPCM**: 2 channels, sample rates: [32, 44.1, 48, 88.2, 96, 176.4, 192] kHz (16bit/20bit/24bit)
- **AC-3 (Dolby Digital)**: 6 channels, sample rates: [32, 44.1, 48] kHz (max 640 kbps)
- **E-AC-3 (Dolby Digital Plus)**: 2 channels, sample rates: [44.1, 48] kHz (byte3=0x00)

**Extension Detailed Timing Descriptors:**
- 720x480 @ 59.94Hz (PCLK=27.00MHz)

---

### Profile: `Mstar_EDID1_v1.4_3D_Frame_SideHalf_Top.bin`

- **Size:** 256 bytes
- **MD5:** `3b4c1c31c9ae084b678a3a4c4af8394e`
- **SHA256:** `6e24e532fa6058ab1c0961ed7478cb8576f6d2fe1ea3f75d41e34abc0e35c484`
- **Manufacturer / Product:** `MST` (Product Code: 48)
- **EDID Version:** 1.3, Extensions: 1

#### Base Block Detailed Timings (DTDs):
- 1920x1080 @ 60.00Hz (PCLK=148.50MHz)
- 1360x768 @ 60.02Hz (PCLK=85.50MHz)

#### CTA-861 Extension Capabilities:
- Basic Audio: True, YCbCr 4:4:4: True, YCbCr 4:2:2: True

**Supported CEA/CTA VIC Modes (Video Data Block):**

| VIC | Native | Resolution / Refresh Rate |
| --- | --- | --- |
| 1 | No | 640x480p @ 59.94/60Hz (4:3) |
| 3 | No | 720x480p @ 59.94/60Hz (16:9) |
| 4 | No | 1280x720p @ 59.94/60Hz (16:9) |
| 5 | No | 1920x1080i @ 59.94/60Hz (16:9) |
| 7 | No | 720(1440)x480i @ 59.94/60Hz (16:9) |
| 16 | Yes (Preferred) | 1920x1080p @ 59.94/60Hz (16:9) |
| 18 | No | 720x576p @ 50Hz (16:9) |
| 19 | No | 1280x720p @ 50Hz (16:9) |
| 20 | No | 1920x1080i @ 50Hz (16:9) |
| 22 | No | 720(1440)x576i @ 50Hz (16:9) |
| 31 | Yes (Preferred) | 1920x1080p @ 50Hz (16:9) |
| 32 | No | 1920x1080p @ 23.976/24Hz (16:9) |
| 34 | No | 1920x1080p @ 29.97/30Hz (16:9) |
| 2 | No | 720x480p @ 59.94/60Hz (4:3) |
| 6 | No | 720(1440)x480i @ 59.94/60Hz (4:3) |
| 17 | No | 720x576p @ 50Hz (4:3) |
| 21 | No | 720(1440)x576i @ 50Hz (4:3) |

**HDMI 1.4 Vendor-Specific Data Block (OUI 00-0c-03):**
- Physical Address: `1.0.0.0`
- Max TMDS Clock: 160 MHz

**HDR Static Metadata (CTA-861.3):**
- Supported EOTFs: Traditional gamma - SDR, Traditional gamma - HDR, SMPTE ST 2084 (HDR10), Hybrid Log-Gamma (HLG)

**Audio Capabilities (Short Audio Descriptors):**

- **LPCM**: 2 channels, sample rates: [32, 44.1, 48, 88.2, 96, 176.4, 192] kHz (16bit/20bit/24bit)
- **AC-3 (Dolby Digital)**: 6 channels, sample rates: [32, 44.1, 48] kHz (max 640 kbps)
- **E-AC-3 (Dolby Digital Plus)**: 8 channels, sample rates: [44.1, 48] kHz (byte3=0x00)

**Extension Detailed Timing Descriptors:**
- 720x480 @ 59.94Hz (PCLK=27.00MHz)
- 720x576 @ 50.00Hz (PCLK=27.00MHz)
- 1280x720 @ 50.00Hz (PCLK=74.25MHz)

---

### Profile: `Mstar_EDID1_v1.4_2K.bin`

- **Size:** 256 bytes
- **MD5:** `99594ebc080a0f161bc8ddc5990cd37a`
- **SHA256:** `f8450becc457aa4f119ea20fca92a56cb73e37e0e739b444a00c1956d8184dcb`
- **Manufacturer / Product:** `MST` (Product Code: 48)
- **EDID Version:** 1.3, Extensions: 1

#### Base Block Detailed Timings (DTDs):
- 1920x1080 @ 60.00Hz (PCLK=148.50MHz)
- 1360x768 @ 60.02Hz (PCLK=85.50MHz)

#### CTA-861 Extension Capabilities:
- Basic Audio: True, YCbCr 4:4:4: True, YCbCr 4:2:2: True

**Supported CEA/CTA VIC Modes (Video Data Block):**

| VIC | Native | Resolution / Refresh Rate |
| --- | --- | --- |
| 1 | No | 640x480p @ 59.94/60Hz (4:3) |
| 3 | No | 720x480p @ 59.94/60Hz (16:9) |
| 4 | No | 1280x720p @ 59.94/60Hz (16:9) |
| 5 | No | 1920x1080i @ 59.94/60Hz (16:9) |
| 7 | No | 720(1440)x480i @ 59.94/60Hz (16:9) |
| 16 | Yes (Preferred) | 1920x1080p @ 59.94/60Hz (16:9) |
| 18 | No | 720x576p @ 50Hz (16:9) |
| 19 | No | 1280x720p @ 50Hz (16:9) |
| 20 | No | 1920x1080i @ 50Hz (16:9) |
| 22 | No | 720(1440)x576i @ 50Hz (16:9) |
| 31 | Yes (Preferred) | 1920x1080p @ 50Hz (16:9) |
| 32 | No | 1920x1080p @ 23.976/24Hz (16:9) |
| 34 | No | 1920x1080p @ 29.97/30Hz (16:9) |

**HDMI 1.4 Vendor-Specific Data Block (OUI 00-0c-03):**
- Physical Address: `1.0.0.0`
- Max TMDS Clock: 0 MHz

**Audio Capabilities (Short Audio Descriptors):**

- **LPCM**: 2 channels, sample rates: [32, 44.1, 48] kHz (16bit/20bit/24bit)
- **AC-3 (Dolby Digital)**: 2 channels, sample rates: [32, 44.1, 48, 96] kHz (max 640 kbps)

**Extension Detailed Timing Descriptors:**
- 720x480 @ 59.94Hz (PCLK=27.00MHz)
- 720x576 @ 50.00Hz (PCLK=27.00MHz)
- 1280x720 @ 50.00Hz (PCLK=74.25MHz)
- 1920x540 @ 50.04Hz (PCLK=74.25MHz)

---

