# PATCH-ORIENTED STATIC ANALYSIS — 4K Timing Exposure
## Target: Thundeal TD98 Pro / C50A (MediaTek MT5889, Android 11/SDK 30, Linux 4.9)
## Surface: `hwcomposer.mt5889.so` + `libmi3.so` + `mik.ko` + `xcker.ko`
## Mode: STATIC ONLY. No device mods, no reboot, no runtime ioctl, no EDID mod, no ELF patch, no generated deployment patch.

---

## EXECUTIVE FINDING (the linchpin)

The previous "hy4_vtable.py" interpretation of `vtable+0x44` was **WRONG**. The `_ZTV23HWCDisplayDevicePrimary`
symbol has the standard 2-slot RTTI header (offset-to-top + typeinfo), so:

> **object-vtable[k] = `_ZTV` symbol slot (2 + k)**

Re-resolved via Ghidra 12.1.2 relocated memory (job hTBcbV → `HY4_vtable_resolved.txt`):

| Object-vtable offset | `_ZTV` slot | Function | Raw addr |
|---|---|---|---|
| `vtable+0x44` | slot 19 (off 0x04c) | **`HWCDisplayDevice::setActiveConfig`** | 0x4d3bc |
| `vtable+0xd8` | slot 56 (off 0x0e0) | `getRefreshPeriod` | 0x54a80 |

**Corroboration (doubles the correctness):** `getDisplayVsyncPeriod` @ 0x52314 calls `vtable+0x44` (line 325) then
`vtable+0xd8` (line 330). `vtable+0xd8 = getRefreshPeriod` is exactly what a vsync-period getter would call
→ the +2 mapping is correct. The old mapping (`vtable+0x44` = `getDisplayRequests`, slot 17) was off by the
2-slot RTTI header and is **discarded**.

**Consequence:** the config-activation virtual call resolves to `setActiveConfig` — an in-memory config-pointer
swap (see below). It does **NOT** select output timing or resolution. The configId path never reaches the kernel
4K timing path.

---

## TASK 1 — CONFIG → TIMING RELATIONSHIP  →  ANSWER **C**

Trace: `setActiveConfigWithConstraints(cfgId)` → stores `*(this+0x60)=cfgId`, arms `*(this+0x58)=1`.
`eventWorkThread` @ 0x48bf8 (and `getDisplayVsyncPeriod` @ 0x52314) on pending-change fire:

```
(**(code**)(*(int*)this + 0x44))(this, *(this+0x60));   // == setActiveConfig(this, cfgId)   [IN-MEMORY]
HWCDisplayDevicePrimary::setActivePanelFrequency(this, cfgId);  // == FreeRunConfig only
```

- `setActiveConfig` @ 0x4d3bc: validates `cfgId < config_count`, compares config W/H (config+0x70/0x74) vs
  display W/H (this+0x70/0x74) for **change detection only**, swaps `*(this+0xdc)` (current-config pointer).
  **No hardware side effect.**
- `setActivePanelFrequency` @ 0x5244c: `local_1c = (cfgId==0)?1:2;` → `MI_DISP_SetControlAttr(ctrl,1,&local_1c)`.
  Sets **FreeRunConfig** (free-run VSync frequency) only. No resolution/timingEnum selection.

**Resolution/timing is selected through a SEPARATE, UNWIRED path.** Therefore adding 4K `addConfig` entries in the
constructor would enumerate 4K configs in Android's HWC2 list, but **activating them would not change the physical
output timing** — the panel would keep running at its 1080p/FreeRun state. Answer **A** (refresh/free-run only) and
**B** (configId also selects timing) are both **false** for the HWC config path.

---

## TASK 2 — AUDIT `vtable+0x44`  →  RESOLVED (correct APS2/RELR relocation)

Resolved via Ghidra's *relocated* memory (not raw file bytes). See linchpin table above.
- `vtable+0x44` = `HWCDisplayDevice::setActiveConfig` @ 0x4d3bc.
- It takes `(this, cfgId)`, performs an in-memory config-pointer swap, issues **no MI_* / XC / kernel call**.
- The old `hy4_vtable.py` reading (which read `.rel.dyn` as `Elf32_Rel` → all-zero slots) has been superseded;
  the relocated-table read is authoritative.

---

## TASK 3 — TRACE `MI_DISP_SetOutputTiming` BACKWARDS  →  NEGATIVE (no path exists)

`readelf --dyn-syms` sweep:

- **HWC imports** these `MI_DISP_*` (UND): `MI_DISPOUT_*`, `MI_DISP_Init/DeInit/GetHandle/GetCaps/GetController/
  OpenController/SetControlAttr/GetOutputTiming/DisableB[...]`. **`MI_DISP_SetOutputTiming` is NOT imported.**
- **libmi3 exports** 95 `MI_DISP_*` FUNC symbols. **`MI_DISP_SetOutputTiming` is absent** (grep empty across all
  bindings). The exported `MI_DISP_SetOutpu[...]` is `MI_DISP_SetOutputRect` (a window/rect setter, *not* timing).
- `MI_DISP_SetOutputTiming` @ 0x1323d4 is a **LOCAL** function inside libmi3, reachable only through
  `mik.ko` → `/dev/mik!disp` → **ioctl 0xc008122c** → kernel `xcker.ko` → XC.

Conclusion: there is **no** HWC config → timingEnum → `MI_DISP_SetOutputTiming` path anywhere in the userspace
display stack. We do not claim absence merely from "no direct ELF import" — we verified the symbol is also
non-exported (local-only), so it is structurally unreachable from HWC. The 4K output-timing setter is effectively
dead from Android's perspective unless explicitly wired.

---

## TASK 4 — REAL PATCH CANDIDATES

| # | File / function | Addr | Current behavior | Required change | Downstream effect | Sufficiency |
|---|---|---|---|---|---|---|
| **C1** | `hwcomposer.mt5889.so` ctor `HWCDisplayDevicePrimary::HWCDisplayDevicePrimary` | 0x50a18 | 2× `addConfig` reusing panel-derived `this+8`/`this+c` (1920×1080), vsync 50/60 Hz only | Add 4× `addConfig(3840,2160,vsync_ns,dpiX,dpiY)` for 24/30/50/60 Hz (vsync_ns = 41666667 / 33333333 / 20000000 / 16666667) | Android enumerates 4K@24/30/50/60 HWC2 configs | **NECESSARY, NOT SUFFICIENT** |
| **C2** | `hwcomposer.mt5889.so` `setActivePanelFrequency` (mode-change chokepoint, already calls `MI_DISP_*`) | 0x5244c | maps cfgId→FreeRunConfig idx only | Extend: when activated config W/H == 3840×2160, map cfgId→timingEnum (13..17) and invoke the output-timing setter | Drives kernel 4K timing path on mode switch | **NECESSARY, NOT SUFFICIENT alone** |
| **C3** | `libmi3.so` `MI_DISP_SetOutputTiming` | 0x1323d4 (local) | LOCAL; unreachable from HWC | Make it reachable: export from libmi3 + add to HWC imports (OR have HWC issue ioctl 0xc008122c directly) | Enables C2 to actually call the setter | **NECESSARY if wiring via libmi3** |
| **C4** | kernel `mik.ko` / `xcker.ko` | n/a | 4K timingEnum 13..17 already routed to XC idx 17..21 | **No change required** | — | already present |
| **C5** | `mik.ko` `MI_DISP_SetOutputTiming` devId gate `(devId & 0x4354)==0x4354` | 0x1323d4 | gates DEVICE id, not timing | Verify controller handle (from `MI_DISP_GetController`) satisfies mask; likely already satisfied | — | assumed OK, verify |

---

## TASK 5 — NECESSARY vs SUFFICIENT

- **C1 (enumerate 4K configs):** NECESSARY. NOT SUFFICIENT (activation never changes output timing).
- **C2 (wire output timing on mode change):** NECESSARY. NOT SUFFICIENT alone (needs C1 enumeration + C3 API reachability + kernel path C4).
- **C3 (make setter reachable):** NECESSARY *iff* wiring goes through libmi3; otherwise fold into C2 via direct ioctl.
- **C4 (kernel path):** already present; assumed SUFFICIENT on the kernel side.
- **Together (C1+C2+C3):** NECESSARY, and **likely SUFFICIENT modulo hardware capability** (UNKNOWN — see risks).
- **No single candidate is SUFFICIENT by itself.**

---

## TASK 6 — HIDDEN VALIDATION (sweep)

- **No hardcoded 1920/1080 reject** in `addConfig` (0x4ce90) or `setActiveConfig` (0x4d3bc). `addConfig` only stores
  W/H in a map + vector (`this+0xd0`); `setActiveConfig` only does index-bounds + W/H change-detect (no rejection).
- **Config count** is vector-derived (`(this+0xd4 - this+0xd0)>>3`); no hardcoded "2 configs" assumption — adding
  configs is safe.
- **devId gate** `(devId & 0x4354) == 0x4354` validates DEVICE, not timingEnum. Must verify the `MI_DISP_GetController`
  handle satisfies it (C5).
- **`_MI_DISP_CheckTxSupportTiming(timingEnum)`** is **warning-only** (does not abort) — 4K timingEnums not blocked here.
- **RESIDUAL UNKNOWN:** `validate` @ 0x53af8 and `present` @ 0x54200 (HWC2 layer validation) may check layer geometry
  against the display's reported W/H (`this+0x70/this+0x74`). If display W/H is not updated to 4K, a 4K layer could be
  forced to client composition or rejected. **Needs verification** — likely requires display W/H to track the active
  config W/H (which the `setActiveConfig` pointer-swap + `getCurrentTiming` flow should already do).

---

## TASK 7 — 3840×2160 PATH (timingEnum 13..17)

- `mik.ko` `_MI_DISP_XC_SetOutputTiming` @ 0x0fa8b0: `switch(timingEnum)` → `case 0x0d..0x11` (13..17) set
  `local_50 = 0x11..0x15` (XC idx 17..21) → `MI_DISP_IMPL_XC_ChangeOutputResolution_EX(_astXcDeviceId + devId*8, local_50)`.
  **No switch-level reject** for 13..17; default → XC idx 6.
- `mik.ko` `MI_DISP_SetOutputTiming` @ 0x1323d4: devId gate (device-only) + warning-only `CheckTxSupportTiming`, then
  `_MI_DISP_SetOutputTiming` → `SetPanelChangeParameters` → `ChangeOutputResolution_EX`.
- **RESIDUAL UNKNOWN:** `xcker.ko` `MI_DISP_IMPL_XC_ChangeOutputResolution_EX` / `MApi_SC_SetFreerunVFreq` DCLK/VTotal/
  panel-timing **reject conditions** for 4K were not fully traced. The 4K enums exist in the switch, implying the XC
  scaler can emit 4K timing, but physical DLP-DMD is 1080p — the 4K path is most plausibly the **HDMI-out** signal
  (MT5889 SoC supports 4K@60). **This hardware-capability question is the dominant residual risk and cannot be resolved by static analysis alone.**

---

## TASK 8 — SMALLEST PLAUSIBLE PATCH SETS + BEST CANDIDATE

### PATCH SET A — enumeration only (INSUFFICIENT)
Add 4× `addConfig(3840,2160,...)` in ctor (C1).
**Result:** Android *exposes* 4K@24/30/50/60, but activating them = `setActiveConfig` (pointer swap) +
`setActivePanelFrequency` (FreeRunConfig). Physical output stays 1080p. **Does not select 4K. REJECTED as insufficient.**

### PATCH SET B — enumeration + wiring (NECESSARY, likely sufficient modulo HW)
- C1: add 4K `addConfig` calls in ctor.
- C2: extend `setActivePanelFrequency` (or the post-`setActiveConfig` call in `eventWorkThread` 0x48bf8) to map the
  activated 4K config → timingEnum 13..17 and invoke the output-timing setter.
- C3: make `MI_DISP_SetOutputTiming` reachable from HWC (export from libmi3 + import, or direct ioctl 0xc008122c with
  the `MI_DISP_GetController` device handle).
- C5: confirm devId satisfies `(devId & 0x4354)==0x4354`.

### BEST PATCH CANDIDATE (single highest-leverage injection site)
**`HWCDisplayDevicePrimary::setActivePanelFrequency` @ 0x5244c** — extend it so that, when the activated config's
W/H == 3840×2160, it additionally maps `cfgId → timingEnum (13..17)` and calls the output-timing setter
(`MI_DISP_SetOutputTiming`, made reachable per C3).

Why it is the best single site:
1. It is **already on the per-mode-change path**, invoked by both `eventWorkThread` and `getDisplayVsyncPeriod`
   immediately after the config pointer swap, with `cfgId` in hand.
2. It **already calls `MI_DISP_*`** (`MI_DISP_GetController` / `MI_DISP_OpenController` / `MI_DISP_SetControlAttr`),
   so the MI client handle/device-id plumbing is already present in this function — minimal new scaffolding.
3. It centralizes the configId→hardware mapping (today: FreeRunConfig; extended: + output timing), keeping the change
   to one function rather than scattering logic across ctor/eventWorkThread/present.

Classification: **NECESSARY, NOT SUFFICIENT alone** — it requires C1 (enumeration) and C3 (API reachability) as
companions. No single change is sufficient.

---

## STOP — AWAITING AUTHORIZATION

Per instruction, **no binaries have been modified and no deployment patch has been generated.** The above identifies
the minimal evidence-backed change set and the best single injection site. Before any patch generation, open questions
to resolve (static or with user input):
1. **Hardware capability:** does the target expose 4K on HDMI-out (not the internal 1080p DLP)? Static analysis cannot
   confirm the physical sink.
2. **C5 devId mask:** confirm the `MI_DISP_GetController` handle satisfies `(devId & 0x4354)==0x4354`.
3. **TASK 6 residual:** verify `validate`/`present` layer-geometry checks vs display W/H don't reject 4K layers.
4. **C3 mechanism:** export-from-libmi3 vs direct ioctl 0xc008122c — pick one before writing the patch.

---

## DE-RISK RESOLUTION (static, 2026-09-06)

All 4 residual unknowns resolved via Ghidra decompilation (jobs on `ghidra_hwc` / `ghidra_libmi3` / `ghidra_mik` / `ghidra_xc`):

**U1 — 4K HDMI-out capability: CONFIRMED SUPPORTED.**
hwcomposer strings: `HDMITX_RES_4K2Kp_24/25/30/50/60Hz`, `DACOUT_4K2KP_*`, `RAPTORS_RES_4K2Kp_*`, `RESOLUTION_4K2KP`.
xcker.ko: `OC Set to 4LANE (3840x2160) Vfreq:30Hz`, `_msAPI_XC_HDMITx_Set4K2KMode`, `MDrv_HDMI_3D_4Kx2K_Process`.
SoC/HDMI-TX fully supports 4K2K output. Internal DLP is 1080p, but HDMI-out to a 4K sink is a real, intended path.
→ residual risk downgraded to "physical sink must accept 4K" (not provable statically; SoC clearly capable).

**U2 — 0x4354 devId gate: RESOLVED (safe).**
`MI_DISP_SetOutputTiming` (mik.ko 0x1323d4) gate `(devId & 0x4354)==0x4354` early-returns on failure
(soft reject, no crash). Internally it calls `_MI_DISP_SetOutputTiming(0, timingEnum)` — devId hardcoded to 0
(main display). Therefore passing `devId = 0x4354` satisfies the mask AND targets main display.
`MI_DISP_GetController` (libmi3 0x7aee8) returns a kernel-assigned handle via ioctl `0xc0081206`; that handle is
also usable, but `0x4354` is the guaranteed-pass value. → no blocker.

**U3 — validate/present/capabilities/getDisplayConfigs: NO 4K reject/cap.**
- `validate` (0x53af8): iterates layers, handles composition types & secure layers; does NOT compare layer
  W/H vs display W/H. No 4K gate.
- `present` (0x54200): `HWComposerOSD/MMVideo/TVVideo::present`; uses active W/H (this+0x170/0x174) for OSD
  region; no reject.
- `getDisplayCapabilities` (0x51ebc): returns capability list (composition types etc.), NOT a resolution cap;
  `initDisplayCapabilities` once.
- `getDisplayConfigs` (0x4ecf4): count = `(this+0xd4 - this+0xd0)>>3`, iterates config vector → adding 4K
  `addConfig` entries auto-flows to Android enumeration. No hardcoded max.
→ adding 4K configs is safe and self-propagating.

**U4 — ChangeOutputResolution_EX (xcker.ko 0x37cbc): NO 4K reject.**
Rejects only if XC idx `> 0xff` (4K idx 17..21 pass) or invalid device id (param_1[1] < 2). The
`Check panel timing if panel timing is invalid, ignore set panel timing` path is a SOFT ignore (no crash).
timingEnum 13..17 → XC idx 17..21 applied unconditionally. → kernel 4K path robust.

**REVISED PATCH SET B (de-risked, minimal — hwcomposer-only):**
- **C1:** ctor (0x50a18) add 4× `addConfig(3840,2160, vsync_ns, dpiX, dpiY)` for 24/30/50/60 Hz
  (vsync_ns = 41666667 / 33333333 / 20000000 / 16666667).
- **C2' (replaces C2+C3):** in `setActivePanelFrequency` (0x5244c) [or right after `setActiveConfig` in
  `eventWorkThread` 0x48bf8], when the activated config W/H == 3840×2160, issue ioctl `0xc008122c` to
  `/dev/mik!disp` with 8-byte `{devId=0x4354, timingEnum (13..17)}`. This drives the kernel 4K timing path
  DIRECTLY, avoiding the need to export `MI_DISP_SetOutputTiming` from libmi3. (If preferred, C3 = export
  `MI_DISP_SetOutputTiming` and call it with `devId=0x4354` instead.)
- Touches ONLY `hwcomposer.mt5889.so` (one binary). libmi3 unchanged.

**BEST PATCH CANDIDATE (unchanged site, refined mechanism):** `setActivePanelFrequency` @ 0x5244c — extend to
map 4K config → timingEnum 13..17 → ioctl `0xc008122c` (devId=0x4354). Necessary-but-not-sufficient on its own
(needs C1 enumeration); together with C1, likely sufficient modulo physical-sink acceptance.

**Overall risk after de-risk:** LOW brick-risk. Worst cases are soft: (a) gate fails → timing not applied, display
stays 1080p (no crash); (b) sink/panel rejects 4K timing → "invalid panel timing ignored" → stays 1080p. No code
path identified that crashes or corrupts the display stack.

---

## STOP — STILL AWAITING AUTHORIZATION (patch generation)

Residual unknowns are now statically resolved. No binaries modified; no patch generated. The only remaining item
before generating the patch is the **physical-sink capability** (U1), which only runtime/testing can confirm.
Awaiting your authorization to generate the actual hwcomposer patch (C1 + C2').
