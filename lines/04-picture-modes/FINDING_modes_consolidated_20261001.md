# FINDING — 2026-10-01: consolidated picture-mode findings, verified against the binary

Verification of the ten `display_edid_investigation/` reports + `hy4_review/PATCH_*.md`, done by
reading them in full and re-deriving the load-bearing facts from `hwcomposer.mt5889.so` myself.

## 1. CORRECTION to what I told the user earlier today

I said the first gate was the panel INI (`osdWidth/osdHeight = 1920×1080`) and that editing it
might be enough. **The reports retract that** (later revision of `CAPABILITY_MATRIX` §B.10, and
`REVISED`/`FINAL_DISPLAY`): **HWC never reads those INI values for the mode list.** The
1920×1080 and the two refresh rates are **literals in the binary**. My statement was wrong.

## 2. Verified by me, byte-exact

**The literal pool at file offset `0x3BC88`:**
```
+0  0x00FE502A = 16 666 666 ns  (60 Hz)
+4  0x00000780 = 1920
+8  0x00000438 = 1080
+12 0x0004E200 = 320 000 (dpi)
```
plus the immediate in `.text`: `ELF 0x40CAE/0x40CB4` = `movw/movt #0x0131_2D00` = **20 000 000 ns
(50 Hz)**, feeding the first `addConfig` at ELF `0x40CB8`; the second at `0x40CCC` takes all
arguments from object members. **Only the first vsync is a literal** (the reports say both are —
they are not).

**The timing tables ARE present** (reports quote ELF `0x37738/0x377cc/0x37860`; the actual
location in this copy is **0x2000 lower**: file `0x376A8`, ELF `0x386A8`). Dumped as int32 the
width table contains **3840 (×5) and 4096 (×5)**, the height table **2160 (×9)**, and the
refresh table **60/50/24/25…**. So the 4K timings exist in the binary — only the `addConfig`
calls that would expose them are missing.

## 3. The two things I could not confirm

* `getDisplayVsyncPeriod`'s trigger: the old report family says "`config_idx == 0`"; the
  independent one says "**fires when the active Config's field at +4 is 0**". Different targets
  for a patch. Not resolved.
* The reports disagree on whether `setActivePanelFrequency` runs on a mode switch (old: never;
  independent: yes, from `eventWorkThread`). The independent one cites the single xref and is
  corroborated by the feasibility report.

## 4. The patch plan that survives all reports (C1 + C2)

* **C1** — in the Primary ctor, append `addConfig(3840,2160,vsync)` for 24/30/50/60 so the
  configs become indices 2..5. Byte sites: the `blx initConfigs` at ELF `0x40CD2` is redirected
  to a code cave. vsync literals: 24 Hz `0x027C81AB`, 30 Hz `0x01FD1C49`, 50 Hz `0x01312D00`,
  60 Hz `0x00FE502A`.
* **C2** — `setActivePanelFrequency` must map config→timing enum and issue the ioctl, because
  C1 alone is **proven insufficient**: that function collapses every non-zero config id to
  FreeRunConfig {1,2}, and the driver only has 4 free-run configs with **VRR enabled only on
  cfg 0 — which HWC never emits**. Path is closed end to end:
  `/dev/mik!disp`, ioctl `0xc008122c` (`_IOC(3,0x12,0x2c,8)`), payload `{devId=0x4354, timingEnum}`;
  4K enums **13/15/16/17** route to XC idx 17..21.
* **Two spec bugs to fix before generating bytes:** the enum map must **skip 14**
  (2→13, 3→15, 4→16, 5→17 — a linear 13..16 map would make the 60 Hz config ask for 4K@50),
  and the 60 Hz literal is `0x00FE502A`, not `0x00FE5021`.

## 5. Things that will survive an HWC patch (important)

* **Kodi has its own whitelist**: `videoscreen.whitelist` = `[{refresh 50.0, res 1920x1080},
  {refresh 60.0, res 1920x1080}]` — an application-level cap that no system patch fixes.
* **VRR is unreachable from Android**: only FreeRunConfig 0 turns VRR on, and HWC emits {1,2}.
* **4K@120 does not exist**: no `*_120P` timing enum anywhere in mik.ko; the 120 Hz panel
  profile in the firmware is a **1920×1080-class TI DLPC6540 DLP** (594 MHz / 2640×1875 = 120.000).
* **The imager is very likely 1920×1080-class with pixel shift (XPR)**: `board.ini`
  `[PixelShiftInfo] m_bPixelShiftStatus = TRUE`, HRange 5 / VRange 3. So "native 4K" is not
  proven — 4K would be produced by temporal sub-frame shifting, capped at 60 Hz.
  **This is the single biggest correction to the earlier reports and it changes the expected
  outcome of a 4K patch.**

## 6. Cheapest things to do first (no patching)

1. Read `/sys/class/graphics/fb0/modes` and try switching with a PC source — it already says
   `U:1920x1080p-51`, i.e. 51 Hz, which no report explains.
2. Read-only runtime capture of `/dev/mik!disp` ioctls — the reports name it as "the single
   highest-value experiment" and nobody has done it. It answers whether the kernel timing path
   is exercised at all today.
3. Check the EDID **CTA tag 0x0D** (DTS descriptor): the reports only transcribed the
   *short-audio-descriptor* lists (LPCM/AC-3/E-AC-3), so "no DTS in the EDID" is **not proven**
   — DTS is normally advertised in a separate 0x0D block that nobody parsed.