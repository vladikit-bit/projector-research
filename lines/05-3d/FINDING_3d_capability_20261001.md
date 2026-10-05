# FINDING — 2026-10-01: can the projector do native 3D? YES — the hardware path is in the build; only Android's plumbing is missing

Evidence read live from the device (root) plus the existing display/EDID reports in
`display_edid_investigation/`.

## 1. A dedicated 3D panel profile ships in the firmware, with explicit 3D driver flags

`m_p3D_120PPanelName` is present in the live configuration — in three `Customer_1.ini`
(`BD_MT5889_H2V1-B4-S/model`, `.../model_1`, `..._EWS/model`) and in `config/dataIndex/dataIndex_1.ini`:

```ini
m_p3D_120PPanelName = "/vendor/cusdata/bsp/common/panel/FullHD_VB1_8LANE_DLP_PROJECTOR_120_3D.ini";
```

That profile (`FullHD_VB1_8LANE_DLPC6540_2K_120_3D`) is 1920×1080, DCLK 594 MHz, and carries
**panel-driver 3D flags**:

```ini
bPanel3DFreerunFlag = 1;  # force freerun under 3D mode
bPanelReverseFlag   = 0;  # Set 3D L/R Switch under 3D mode
```

Free-run + L/R switch = **frame-sequential (shutter-glasses) 3D at 120 Hz**, implemented in the
panel driver, not faked in software. A second 3D panel profile also exists
(`FullHD_LG420EUF_120HZ_3DPASSIVE.ini`).

## 2. The currently mounted panel supports 120 Hz

`/vendor/cusdata/bsp/common/panel/UD_VB1_16LANE_CSOT_URSA.ini`:
`m_wPanelWidth=3840`, `m_wPanelHeight=2160`, `m_dwPanelDCLK=600`, HTotal/VTotal 4400/2260,
**`m_bVRR_en=1`, `m_wVRR_min=24`, `m_wVRR_max=120`** — a 4K panel with a 24–120 Hz range, i.e. the
hardware side of a 120 Hz 3D path exists.

## 3. The EDIDs advertised to external sources ARE 3D EDIDs

`/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/edid_cfg.ini`, HDMI port 1:

```ini
HDMI_EDID_File     = "Mstar_EDID1_v2.0_4K_2K_3D_HDR.bin";
HDMI_EDID_File_2_1 = "Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin";
HDMI_EDID_File_1_4 = "Mstar_EDID1_v1.4_3D_Frame_SideHalf_Top.bin";
```

`EDID_BIN/` holds ~36 3D profiles (4 ports × 3D_Frame_SideHalf_Top / 4K_2K_3D / 3D_HDR /
freesync / NoFRL, plus T31 variants). **SideHalf/Top-Bottom** is the HDMI-1.4-era SBS/TB 3D
format; HDMI 2.0/2.1 profiles add 3D+HDR and VRR.

So both standard 3D mechanisms are represented: **frame-sequential for glasses** (via the 3D panel
profile) and **side-by-half / top-bottom** (via the EDIDs).

## 4. What is actually missing is Android's plumbing

* Android exposes only **1920×1080@60 and @50** — `HWCDisplayDevicePrimary`'s constructor calls
  `addConfig()` exactly twice (hardcoded `20000000` ns and `0xFE502A` ns), and
  `getDisplayVsyncPeriod()` forces config 0 to 50 Hz. There is **no 120 Hz config**, so
  frame-sequential 3D is impossible from internal playback.
* Consequently the Android menu items **3D Mode** and **3D-to-2D**
  (`tv_picture_advance_video_3d_mode`, `_3d_to_2d`) render greyed — most likely a
  source-capability / mode-capability check, not a missing feature.

## 5. Verdict

The projector **can** do native 3D the way some TVs do — shutter glasses (frame-sequential 120 Hz,
hardware flags present, 120 Hz VRR panel) and SBS/TB for sources that negotiate HDMI 1.4. "Anaglyph
only" is what the **software path exposes today**, not what the hardware or the build can do.
Enabling it internally requires adding a 120 Hz config in `hwcomposer.mt5889.so` and selecting
the 3D panel profile — both are configuration/edits, not new hardware. With an **external source
over HDMI** the projector already advertises 3D, which is the path most likely to work today.

## 6. Bonus: the EDID audio descriptors explain the DTS-over-HDMI observation

From `EDID_FORENSIC_ANALYSIS.md` (Short Audio Descriptors in the active HDMI profiles):
**LPCM** (2ch, 32/44.1/48/88.2/96/176.4/192 kHz), **AC-3** (6ch, max 640 kbps),
**E-AC-3** (44.1/48 kHz) — **no DTS, no TrueHD/Atmos descriptor**. This matches the user's
recollection that with an external source Dolby Digital passed over S/PDIF while DTS did not: a
downstream device that reads these EDIDs has no reason to offer DTS passthrough.