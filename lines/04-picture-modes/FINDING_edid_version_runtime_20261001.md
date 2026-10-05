# FINDING — 2026-10-01: the EDID version advertised is a RUNTIME setting, and it was pinned to 1.4

Direct answer to the user's theory — which turned out to be right, one level deeper than expected.

## What was found

The Android TV Settings item «Версія HDMI EDID» is backed by a plain global setting:

```sh
settings get global tv_input_hdmi_edid_version     # -> 0
```

and the runtime value **overrides the configuration file**: `edid_cfg.ini` says
`eEdidVersion = 3` (EDID 2.1) on all four HDMI ports, but the live system was at **0**.

## Measured mapping (set the value, reopened the app, read the label)

| `tv_input_hdmi_edid_version` | Label shown |
|---|---|
| **0 (was the live state)** | **EDID 1.4** |
| 1 | EDID 2.0 |
| 2 | Автоматично визначати EDID (Auto) |

The user's observation — "у налаштуваннях Android зараз ми бачимо 1.4" — is exactly this: the
runtime sat at **0 = EDID 1.4**, not the 2.1 profile the config asks for.

## Why 1.4 matters

Per the existing capability matrix (`display_edid_investigation/REPORT_EDID_DISPLAY_MODES.md`):

| EDID profile | 4K@60 | 4K 24/30/50 | HDR10/DV/HLG | VRR/ALLM |
|---|---|---|---|---|
| **1.4** | ❌ | ❌ | ❌ | ❌ |
| 2.0 | ✅ | ✅ | ✅ | ❌ |
| 2.1 | ✅ | ✅ | ✅ | ✅ |

So at 1.4 the projector advertises **1080p-class only** to sources — the "1.4" state is
precisely the "максимум 1080p" state.

**Parsed from the binaries themselves:** `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin` (the profile the
config selects) declares a native DTD at **5940 MHz = 3840×2160@60**, VICs for 3840×2160p at
24/25/30/50/60, `3D present = yes`, `YCbCr 4:2:0 = yes`.

## The open half — and it does not contradict the user

The user reports that **with external sources the projector already offers many modes, nearly
everything**. That means the runtime value at 0 does **not** obviously cap what sources actually
see — either the served EDID comes from the per-port config (2.1) regardless of this setting, or
the source negotiates upwards. Not established either way.

## Current state

`tv_input_hdmi_edid_version` was changed **0 → 1 (EDID 2.0) → 2 (Auto)**; it is left at **2
(Auto)**, which is the sensible default and is reversible with
`settings put global tv_input_hdmi_edid_version 0`.

## Next measurement (needs an external source, costs one cable)

With a source plugged into any HDMI port, read the list of modes it offers, then set the setting
to 0 (1.4) and read again. If the list does not change, this setting is cosmetic and the served
EDID is fixed by `edid_cfg.ini`; if it shrinks to 1080p at 0, the setting is real and 2.1 is
worth pinning permanently.