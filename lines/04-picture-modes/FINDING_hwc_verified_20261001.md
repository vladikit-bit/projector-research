# FINDING — 2026-10-01: HWC mode list, verified from the binary (not from the reports)

**Library:** `/vendor/lib/hw/hwcomposer.mt5889.so` — **ELF32 ARM (Thumb), 475 016 B, stripped but
with a rich `.dynsym`**, so C++ symbols (`HWCDisplayDevice::addConfig`, `initConfigs`,
`getDisplayVsyncPeriod`) are resolvable by name. Pulled to `C:\firmware_temp\hwc\`.

## The two configs are added here — verified

```
_ZN23HWCDisplayDevicePrimaryC2ER16HWComposerDevice:            @ 0x40a18

  0x40c76  blx MI_DISPOUT_PanelGetAttr(0x1a, 0, …)
  0x40c82  blx MI_DISPOUT_PanelGetAttr(0x1b, 0, …)
  0x40c86  ldrh r0,[sp,#0x16] ; ldrh r1,[sp,#0x14]     ; panel H/V size
  0x40ca2  ldrd r1,r2,[r4,#8]        ; args 1,2 (width, height) from object members
  0x40ca6  ldrd r0,r3,[r4,#16]
  0x40caa  strd r0,r3,[sp]           ; args 5,6 on the stack
  0x40cae  movw r3,#0x2d00
  0x40cb4  movt r3,#0x131            ; r3 = 0x0131_2D00 = 20 000 000 ns  →  50 Hz
  0x40cb8  blx HWCDisplayDevice::addConfig       ← CONFIG #1, vsync HARDCODED 50 Hz

  0x40cbc  ldrd r3,r1,[r4,#4]        ; all args from object members
  0x40cc0  ldrd r2,r0,[r4,#12]
  0x40cc4  ldr  r6,[r4,#0x14]
  0x40cc6  strd r0,r6,[sp]
  0x40ccc  blx HWCDisplayDevice::addConfig       ← CONFIG #2, vsync from members
```

`addConfig(unsigned, unsigned, unsigned, int, int, int)` is declared in `.dynsym`
(`_ZN16HWCDisplayDevice9addConfigEjjiii`). The only third call site in the library is at
`0x46a38` inside `HWCDisplayDeviceVirtual` — not the internal panel.

**So the reports are right in substance but sloppy in detail:** only the **first** config's vsync
is a literal (20 000 000 ns = 50 Hz); the second one is loaded from object members. Both take
their **width/height from members**, which is why both are 1080p.

## Where the 1920×1080 comes from (verified live)

`/vendor/tvconfig/config/panel/UD_VB1_16LANE_CSOT_URSA.ini` (the profile `Customer_1.ini`
selects via `m_pPanelName`):

```ini
osdWidth  = 1920;
osdHeight = 1080;
b3DOSDLRSwitchFlag = 0;
```

The panel itself is 3840×2160 @ DCLK 600 MHz with VRR 24–120 Hz. **The OSD/UI size is what caps
HWC's configs at 1080p**, and it is a plain text INI under `/vendor`.

`Customer_1.ini` also sets:
```ini
m_p3D_120PPanelName = "/vendor/cusdata/bsp/common/panel/FullHD_VB1_8LANE_DLP_PROJECTOR_120_3D.ini";
```
i.e. the 120 Hz 3D DLP panel is wired up but the *active* panel is the 4K CSOT one.

## Implication — a text edit may be enough

Restoring modes has two independent gates:

1. **resolution/height** — `osdWidth`/`osdHeight` in the panel INI (1920×1080 → 3840×2160);
2. **extra refresh rates** — config #1's vsync is a hardcoded literal at `0x40cae/0x40cb4`
   (movw/movt pair = 0x01312D00); additional rates would need extra `addConfig` calls, which
   means injecting Thumb code (there is room after `0x40cd0` before the literal pool at
   `0x40dc8`).

Gate 1 is a one-line INI change and carries no code risk. Gate 2 is a code patch. The INI route
should be tried first — if 4K appears, the panel/scaler path is proven and only the rate list
remains.

Not done yet: whether the members at `[r4+4..0x14]` are filled from the INI OSD size or from the
panel native size — that is the next verification step, and it decides whether editing
`osdWidth` alone is sufficient.