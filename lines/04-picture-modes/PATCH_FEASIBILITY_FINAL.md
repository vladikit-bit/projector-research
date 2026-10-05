# Final Patch-Feasibility Pass — 4K (3840×2160) HWC Enablement
**Target:** Thundeal TD98 Pro / C50A — MediaTek MT5889, Android 11 (SDK 30), Linux 4.9
**Binary:** `hwcomposer.mt5889.so` (ARM/Thumb-2, 32-bit ELF)
**Candidate architecture (accepted):** C1 = ctor adds 4× 4K `addConfig`; C2 = mode-change path drives the existing MI output-timing mechanism for the selected 4K config.
**Status of this pass:** FEASIBILITY ONLY. No patch bytes emitted. No binary modified. STOP.

---

## T1 — Direct ioctl feasibility (C2')
**RESOLVED — feasible, ZERO new imports.**

`readelf --dyn-syms` on `hwcomposer.mt5889.so` (versioned-symbol form `open@LIBC (2)` / `ioctl@LIBC (2)`, NOT `open@@`) confirms HWC already imports:
- `open@LIBC`, `ioctl@LIBC`, `__open_2@LIBC`, `fcntl@LIBC`, `write@LIBC`.

HWC does **not** currently open `/dev/mik!disp` and holds no reusable fd. `MI_DISP_GetController()` returns an **opaque MI handle** (via `ioctl(libmi3_private_fd, 0xc0081206, &handle)`) — it is **NOT** a Linux fd and cannot be reused by HWC.

**Therefore C2' = Method A:** HWC does `open("/dev/mik!disp", O_RDWR)` itself, caches the fd in a global slot, then `ioctl(fd, 0xc008122c, &{devId=0x4354, timingEnum})`. No new PLT/GOT/relocation entries are needed (open/ioctl already imported).

---

## T2 — Three wiring methods compared

| | **A. HWC → direct `/dev/mik!disp` ioctl** | **B. HWC → existing accessible MI wrapper** | **C. Export `MI_DISP_SetOutputTiming` via libmi3 + call from HWC** |
|---|---|---|---|
| Binaries touched | HWC only | HWC only (but no such wrapper exists) | **libmi3.so + HWC** (2 files) |
| New imports / PLT / GOT | **NONE** (open/ioctl present) | NONE claimed, but… | **NEW** import in HWC (new PLT, GOT slot, `.rela.plt`, dyn-sym, version node) |
| New data | 1 string `/dev/mik!disp` (16B) + 1 fd slot (4B) | — | libmi3 must export local fn `0x1323d4` (dyn-sym/.dynstr edit) |
| Code size (new) | ~2 code caves ≈ 178B + 8B branch overwrites + literal pools ≈ **~230B** | N/A | small HWC call (~10B) but **large ELF structural diff** in 2 files |
| ELF relocation impact | **NONE** | NONE | NEW `.rela.plt`/GOT entry + dyn-symbol (breaks libmi3 export ABI) |
| Runtime side effects | opens device node (SELinux risk, soft-failsafe) | — | same timing effect, but changes libmi3 exported set |
| Reversibility | trivial (restore bytes) | — | harder (2 binaries) |
| Verdict | ✅ **RECOMMENDED — smallest, evidence-backed** | ❌ **INFEASIBLE** — no HWC-imported MI function sets output timing; `MI_DISP_*` imports are GetController/OpenController/SetControlAttr/GetCaps/DeInit/… none do timing | ❌ LARGER, riskier — only if a wrapper were mandatory (it is not) |

**B is rejected:** there is no existing accessible MI wrapper for output timing (confirmed by dynsym sweep of both HWC and libmi3 — `MI_DISP_SetOutputTiming` is a LOCAL fn in libmi3, not exported, and not imported by HWC). B collapses to "misuse `SetControlAttr`" (which only sets FreeRunConfig, not resolution) → does not solve C2.

---

## T3 — devId argument verification
**RESOLVED — `0x4354` is a gate-only value; routing is forced to main display regardless.**

Chain: `GetController` → returns opaque handle (not used as devId for timing) → `SetOutputTiming` gate `(devId & 0x4354) != 0x4354 → return` (soft reject) → `_MI_DISP_SetOutputTiming(0, timingEnum)` **hardcodes devId=0** (main display).

`0x4354` is consumed **solely by the mask check**; it is not a meaningful deep device identifier. Passing `devId=0x4354` satisfies the gate AND targets the main display by construction. It is the guaranteed-pass value (also the value the firmware's own 4K paths use). **NOT marked UNKNOWN** — this is an evidence-backed stronger conclusion than prior hand-wave.

---

## T4 — Exact C1 patch feasibility
**RESOLVED — feasible, but a code cave is required (ctor has no inline slack).**

- Two stock 1080p configs are added in ctor `HWCDisplayDevicePrimary` @ 0x50a18 via veneer `0x7c7e0` (= `addConfig`) at **0x50cb8** and **0x50ccc**.
- `addConfig` signature = `(this=r0, W=r1, H=r2, vsyncNs=r3)`. Existing 1080p cfg uses `r3=0x1312D00 = 20,000,000 ns = 50 Hz` (clean vsync period).
- **Insertion point:** after the 2nd `addConfig` (0x50ccc). The next live instruction is `mov r0,r4` (0x50cd0) then `blx 0x7c7f0` (0x50cd2). Clean 4-byte swap: replace `blx 0x7c7f0` @ 0x50cd2 with `bl <cave_C1>`.
- **cave_C1 (~100B)** runs 4× `addConfig(3840, 2160, vsyncNs)` (configs become indices 2..5), then restores `mov r0,r4; blx 0x7c7f0; b 0x50cd6`.
- Per 4K addConfig call (immediates; no panel-field dependency): `mov r0,r4`(2) + `movw r1,#0xF00`(4) + `movw r2,#0x870`(4) + `movw r3,#lo; movt r3,#hi`(8) + `blx 0x7c7e0`(4) = **22B × 4 = 88B**.
- vsync_ns literals: 24Hz=`0x27C81AB`, 30Hz=`0x01FD1C49`, 50Hz=`0x1312D00` (reuse stock), 60Hz=`0x00FE5021`. W/H literals: 3840=`0xF00`, 2160=`0x870` — all fit `movw`.
- **No hardcoded 1080p reject in `addConfig`** (vector auto-grows); config_count derivation must be re-verified at patch-gen (R3).
- **Code cave required:** ~100B region from alignment gaps / unused function padding / `.so` tail (finalize location at patch-gen).

---

## T5 — Exact C2 patch feasibility
**RESOLVED — feasible, code cave required (function is 140B and tight).**

- Mode-change chokepoint `setActivePanelFrequency` @ 0x5244c. xref confirms sole caller = `eventWorkThread`.
- W/H is read from `this+0x70` at **0x524a8** (`ldrd r0,r1,[r5,#0x70]`). cfgId is in `r4` (arg r1, already present).
- **Injection point:** right after the W/H load (0x524a8). Replace the following 4-byte instruction `ldr r2,[sp,#0x1c]` @ 0x524ac with `bl <cave_C2>`.
- **cave_C2 (~70-80B):** `cmp r0,#0xF00; bne skip` (W==3840); `cmp r1,#0x870; bne skip` (H==2160); map `cfgId(r4)→timingEnum 13..16` (4K 24/30/50/60); load cached fd, `cbnz skip`, else `open("/dev/mik!disp")` + store; build 8-byte struct `{devId=0x4354, timingEnum}`; `ioctl(fd, 0xc008122c, &struct)`; `skip: b 0x524b0`.
- ioctl nr `0xc008122c` = 8-byte `{devId, timingEnum}` write to `/dev/mik!disp` (`_IOC_WRITE`, len 8).
- **Exact code size:** cave ≈ 70-80B + 4B branch overwrite; literal pool inside cave (0xc008122c, 0x4354, timingEnum values, struct addr) ≈ 24-32B.

---

## T6 — Duplicated invocation check  ← user-flagged "important"
**RESOLVED — SINGLE APPLICATION (no duplication).**

- `setActivePanelFrequency` veneer `0x7bd90` xref shows **exactly one caller**: `eventWorkThread` @ 0x48bf8 (call at 0x491ee), once per pending mode change (after the indirect `setActiveConfig` vtable call `[r0+0x44]`).
- `getDisplayVsyncPeriod` @ 0x52314 (full disasm reviewed): calls `0x7bcf0`, `0x7bac0` (×2), `0x7c3b0`, `0x7ba60`, `0x7bb00`, and two vtable slots `[+0x44]`/`[+0xd8]`. **None is `setActivePanelFrequency` (0x7bd90 / 0x5244c).** Its vtable calls are different slots; `setActivePanelFrequency` is a non-virtual member (called directly, not via vtable).
- **Conclusion:** adding timing programming inside `setActivePanelFrequency` executes **exactly once** per mode change. Control flow guarantees single application. No double-apply risk.

---

## T7 — FINAL DELIVERABLE (A–F)

**A. Recommended wiring method**
> **Method A** — HWC issues the ioctl `0xc008122c` to `/dev/mik!disp` directly. No new imports (open/ioctl/fcntl/write already imported), no libmi3 modification, no kernel change. Smallest, evidence-backed, trivially reversible.

**B. Exact functions to modify**
> - `hwcomposer.mt5889.so`: `HWCDisplayDevicePrimary` ctor @ **0x50a18** (C1: 4× `addConfig`).
> - `hwcomposer.mt5889.so`: `setActivePanelFrequency` @ **0x5244c** (C2: ioctl injection).
> - Kernel `mik.ko`/`xcker.ko`: **NO CHANGE** (4K timingEnum 13..17 → XC idx 17..21 already routed).
> - `libmi3.so`: **NO CHANGE** (Method A avoids exporting `MI_DISP_SetOutputTiming`).

**C. Exact instruction regions**
> - **C1:** replace `blx 0x7c7f0` @ **0x50cd2** (4B) with `bl <cave_C1>`; cave_C1 (~100B) = 4× `addConfig(3840,2160,vsyncNs)` then `mov r0,r4; blx 0x7c7f0; b 0x50cd6`. addConfig veneer = **0x7c7e0**. 4K configs become indices 2..5.
> - **C2:** replace `ldr r2,[sp,#0x1c]` @ **0x524ac** (4B) with `bl <cave_C2>`; cave_C2 (~70-80B) gates on `[this+0x70]==3840×2160`, maps `cfgId→timingEnum 13..16`, opens fd if needed, builds `{devId=0x4354, timingEnum}`, `ioctl(0xc008122c)`, returns to **0x524b0**.
> - Cave locations = scan for ~100B (C1) + ~80B (C2) unused region within the `.so` (alignment gaps / `.so` tail); finalize at patch-gen.

**D. Required new data / imports**
> - New imports: **NONE**.
> - New string/data: `"/dev/mik!disp"` (16B, rodata) + 1 fd slot (4B, `.bss`, zero-init).
> - New literals (in cave pools): `0xc008122c`, `0x4354`, `0xF00`, `0x870`, vsync_ns (`0x27C81AB`/`0x01FD1C49`/`0x1312D00`/`0x00FE5021`), timingEnum values 13..16, struct addr.
> - New ELF relocations: **NONE** (PC-relative literal pools; fd slot via existing GOT or PC-relative global).

**E. Estimated binary diff size**
> - Single `.so` (`hwcomposer.mt5889.so`). Net-new content ≈ cave_C1 (100B) + cave_C2 (78B) + string (16B) + fd slot (4B) + literal pools (~32B) ≈ **~230B**; overwrites ~8B at the two branch sites. **Total ELF Δ ≈ 230–260 bytes. No new PLT/GOT/relocation. libmi3 and kernel untouched.**

**F. Remaining risks**
> - **R1 (runtime):** `/dev/mik!disp` open permission / SELinux for the HWC process. If `open()` fails → soft-failsafe (no 4K timing; 1080p path still works). No crash.
> - **R2 (runtime):** physical-sink 4K acceptance (HDMI-TX / EDID / panel) — cannot be verified statically. Soft-failsafe.
> - **R3:** cfgId→timingEnum mapping assumes 4K configs appended as indices 2..5 (config_count→6). If firmware reorders/adds configs, reconcile at patch-gen (re-derive config_count in ctor).
> - **R4:** code-cave location — must find a genuinely-unused ~180B region; fallback = small appended section (no new segment required).
> - **R5 (low):** W/H gate uses `[this+0x70]` — consistent between `setActivePanelFrequency` (0x524a8) and `getDisplayVsyncPeriod` (0x52374); same field. Low risk.
> - **R6 (reversibility):** surgical + reversible (restore bytes / unload caves).

---

## Hard STOP
No ELF patch was generated. No binary was modified. Awaiting explicit authorization to emit the patch (C1 + C2 via Method A).
