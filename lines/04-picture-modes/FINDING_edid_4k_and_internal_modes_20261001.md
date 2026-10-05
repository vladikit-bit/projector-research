# FINDING — 2026-10-01: the active EDID is NOT 1080p-limited; the Android 1080p cap has nothing to do with EDID

User's theory to test: "maybe the EDID currently in effect for Android is one where the maximum
is exactly 1080p50/60 — or those are the only modes at all."

**Verdict: refuted for both paths.**

## 1. Which EDID is active

`/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/edid_cfg.ini`, all four HDMI ports:

```ini
HDMI_EDID_File     = "Mstar_EDID{1..4}_v2.0_4K_2K_3D_HDR.bin"
HDMI_EDID_File_2_1 = "Mstar_EDID{1..4}_v2.1_4K_2K_3D_HDR.bin"
HDMI_EDID_File_1_4 = "Mstar_EDID{1..4}_v1.4_3D_Frame_SideHalf_Top.bin"
bEDIDEnabled = 1
eEdidVersion = 3          ; EDID_21 -> the 2.1 profile is the active one
```

## 2. What the active EDID actually declares (parsed directly from the binary)

`Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin` (256 B, EDID 1.3, manufacturer **MST**, product 0x30):

| Field | Value |
|---|---|
| Base DTD #1 | pixel clock **5940.00 MHz** = 3840×2160 @ 60 |
| Base DTD #2 | pixel clock 1485.00 MHz = 1920×1080 @ 60 |
| CTA video capability | native DTDs, **3D present = yes**, **YCbCr 4:2:0 = yes** |
| VIC list | **3840×2160p @ 24 / 25 / 30 / 50** (and 60 in the 2.1 profile), 1920×1080p @ 24/25/30/50/60, 1280×720p, 720×480/576 |

The 2.0 profile is the same minus the 4K60 VIC.

**So the projector advertises full 4K (up to 60 Hz), 3D and 4:2:0 to external sources.** It is
not a 1080p-only EDID.

## 3. Why it could not be the cause of Android's mode list anyway

The two paths are independent (see `display_edid_investigation/REPORT_EDID_DISPLAY_MODES.md`):

* **PATH A — HDMI RX EDID** = what the projector tells *external sources*. It is 4K-capable.
  It has no influence on what Android's own UI can display.
* **PATH B — internal panel timing** = the Android display. It is not described by an EDID at
  all; `HWCDisplayDevicePrimary`'s constructor calls `addConfig()` exactly twice with hardcoded
  1080p@50 and 1080p@60, and `getDisplayVsyncPeriod()` pins config 0 to 50 Hz.

`dumpsys display` confirms the internal display still reports exactly two supportedModes
(1080p60, 1080p50).

## 4. Consequence — the useful test is free

Since the EDID already advertises 4K and the panel is natively 3840×2160, **4K from an external
HDMI source should work today, with no patching at all.** Plugging a 4K source into any of the
four HDMI ports and checking whether the projector renders 4K is the cheapest possible test of
the whole "all modes" question, and it separates two different things:

* if external 4K works → the panel/scaler/receiver path is fine and the only thing missing is
  Android's HWC config list (a `.so` patch);
* if external 4K does **not** work → the 4K claim in the EDID is not backed by the actual
  pipeline, and the HWC patch alone would not help.

Either way it costs one cable.