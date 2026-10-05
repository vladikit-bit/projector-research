# FINDING — 2026-10-01: Dolby Vision on this unit decodes but has NO tone-mapping tuning data

Found by reading the device's own Dolby configuration and the PQ engine's runtime log. This
answers the user's two symptoms (wrong colours when starting DV content, and the DV menu
appearing) with a concrete mechanism rather than a guess.

## 1. The panel's Dolby Vision data IS in the image

`/vendor/tvconfig/config/HDR_BIN/dolby_factory.cfg` (2 767 B) is a complete Dolby Vision VSVDB v2
panel profile:

```ini
Tmax = 281            Tmin = 0.23      TEOTF = POWER      Tgamma = 2.2   TContrast = 1221
TPrimaries = 0.6291 0.3425  0.3100 0.6106  0.1508 0.0496  0.3124 0.3319
GlobalDimming = 1     BLULight = 0.23 70 140 211 281
vsvdb_version = 2     vsvdb_dm_version = 4.0_VSEM
vsvdb_Tmax = 281      vsvdb_Tmin = 0.23
support_normal_dolbyvision = 1        # "Support tunneled picture"
```

**These `Tmax / Tmin / gamma / TPrimaries` are exactly the 11 fields that are ZERO in the
projector's «Dolby Vision PQ Calibration → End-user calibration» menu** (Tmax, Tmin, screen
Gamma, Rx, Ry, Gx, Gy, Bx, By, Wx, Wy). The factory data for them exists; the calibration store
is empty.

The file also carries **three Dolby picture modes**:
`[PictureMode 0] Dark` (Tmax 281, GlobalDimming 1) · `[1] Bright` (Tmax 140) · `[2] Vivid` (Tmax 210)
— these are almost certainly what the «Режим зображення» preset list maps to.

## 2. But the PQ tuning binary is a stub

`dolby.bin` — 14 235 bytes, and **every byte is `0x01`** (header included). It is a placeholder,
not tuning data.

## 3. And the engine says so at runtime

```
[PQ] MHal_PQ_Get_GameMode_TableIndex: [GameMode] Pls check size input H:1920,V:1080
[PQ] QM_InputSourceToIndex: [Load PQ] Panel index=0, Source index=181
[PQ] MDrv_PQ_Set_CustomerIp_Parameter_U2: Customer IP is not set as customer mode
<MI3_ERR>_MI_DISP_SetHdrPanelNits: Set E_PQ_XC_IP_TMO Customer Data fail.
[PQ] MDrv_PQBin_GetGRule_GroupIPNum: =PQ_BIN_NOT_ENABLE!!!
```

The PQ engine is alive and loading PQ metadata (source index 181), but: the customer IP/TMO
(Dolby Vision Tailor-Made Operating) data is **not set**, and the PQ bin is **not enabled**.

## 4. What this means

* Dolby Vision **decode** is real (the `OMX.MS.DOLBY_VISION.*` family is declared; the banner
  appears; the PQ engine processes metadata).
* Dolby Vision **tone mapping is not calibrated** on this unit — no IP/TMO, stub PQ bin.
* So the DV menu being visible does **not** mean DV works; it means the *path* exists and the
  *tuning* is missing. That is a precise, testable explanation for the wrong colours the user
  saw when starting DV content in Kodi.

## 5. What could be tried (none of it done — this is a proposal)

1. Find what selects "customer mode" for the PQ engine (not a key in this cfg — it must be a
   vendor setting, a property, or a build flag) and switch it on, so the panel data in
   `dolby_factory.cfg` is actually consumed.
2. Check whether the customer IP is expected to arrive from the **decoder/DRM path** at
   playback (source index 181 suggests a metadata source) rather than from a file — in which
   case the DV content being played may not be carrying a usable IP.
3. Fill the 11 calibration fields from `dolby_factory.cfg` (Tmax 281, Tmin 0.23, TPrimaries,
   TEOTF POWER, Tgamma 2.2) through the visible calibration menu — a purely
   user-visible action requiring no patching, and it would immediately tell us whether the
   colour error is the calibration or something else.

Item 3 is the cheapest decisive test of the whole Dolby Vision line and is available to the
user without any binary patching.

## 6. Relation to the display-mode work

Independent of the mode list: `BW Window:0, PQSource:9, Hsize:1920, Vsize:1080` confirms the
panel is 1920×1080-class at runtime — the earlier static conclusion (1080p imager + pixel shift)
holds. The mode-list blocker (HWC collapses config ids to FreeRunConfig 1/2 → 50/60 Hz) is
separate and still open; it is not caused by the PQ problem.