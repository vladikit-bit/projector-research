# REPORT: R5 Phase E — Deployment & Regression Test Results

**Device:** Thundeal TD98 Pro / C50A (MStar MT5889) · Android TV (Kodi 22)
**AVR:** Pioneer VSX-817 (optical/coax SPDIF, expects IEC61937 DTS/AC3)
**Date:** 2026-09-06
**Authorization:** explicit (user authorized Phase E deployment of the verified R5 patch)
**Patch:** `runtime_phase5/mik_r5_patched.ko` (md5 `03fc2c0d…`), 1 byte at file `0x9b54c` / VA `0x98a58`.

---

## 1. Authorization & scope

User authorized deployment of the **already-verified** R5 `mik.ko` patch only. No changes to
EDID / HDMI / ARC / routing / settings / properties / `utpa2k.ko` / `libmi3.so` / any other module.
No second patch was made during the experiment.

## 2. Before / after module hashes

| | module | MD5 |
|---|---|---|
| **Before** (baseline, pre-deploy) | `/vendor/lib/modules/mik.ko` | `647b08db19dcbc20b7d5c6a4a4e36e9b` |
| **After** (active, post-reboot) | `/vendor/lib/modules/mik.ko` | `03fc2c0d22b231bc478e574557b85473` |
| Source patch | `runtime_phase5/mik_r5_patched.ko` | `03fc2c0d22b231bc478e574557b85473` |
| `utpa2k.ko` (Exp-B, unchanged throughout) | `/vendor/lib/modules/utpa2k.ko` | `4c5e6fbb10e24abc3b8d2b0490507983` |

Pre-deploy baseline hash matched the expected `647b08db…`. Post-reboot active hash matched the
expected `03fc2c0d…`. `utpa2k.ko` (Exp-B) is **unchanged**.

## 3. Deployment steps performed

```bash
adb root                                   # already root; adbd uid 0
adb remount                                # /vendor was ro (dm-1, verity); remount succeeded -> rw
adb shell md5sum /vendor/lib/modules/mik.ko          # 647b08db... (baseline confirmed)
adb pull /vendor/lib/modules/mik.ko backup/mik.ko.baseline_647b08db   # backup (host)
adb push mik_r5_patched.ko /data/local/tmp/mik_r5_patched.ko
adb shell cp /data/local/tmp/mik_r5_patched.ko /vendor/lib/modules/mik.ko
adb shell chmod 644 /vendor/lib/modules/mik.ko && sync
adb reboot
# after reboot:
adb root
adb shell md5sum /vendor/lib/modules/mik.ko          # 03fc2c0d... (deployed confirmed)
adb shell lsmod | grep mik                            # mik loaded (901 uses, pulled by dtv_driver)
```

`adb` binary: `C:\Android\platform-tools-latest-windows\platform-tools\adb.exe` (device over TCP
`192.168.0.183:5555`). Backup retained at
`runtime_phase5/backup/mik.ko.baseline_647b08db` (host, md5 `647b08db…`).

## 4. Test procedure

For each window: `am force-stop kodi` → `logcat -c` → `am start VIEW test_*.mp4` → play 16 s →
capture `dmesg` + `logcat -d` (no `dumpsys`) → `force-stop`.

| window | clip | purpose |
|---|---|---|
| E1 | `test_ac3_51.mp4` | AC3/DD control (regression gate) |
| E2 | `test_dts_51.mp4` | DTS main test |
| E3 | `test_dts_51.mp4` | DTS repeat (rule out one-off) |
| E4 | `test_ac3_51.mp4` | AC3 repeat (regression control) |

Captures in `runtime_phase5/phaseE/`: `E1_ac3_ctrl`, `E2_dts_main`, `E3_dts_repeat`, `E4_ac3_repeat`
(each `.dmesg` + `.logcat`).

## 5. Results

### 5.1 AC3 — E1 (control) and E4 (repeat)
Kodi plays AC3 as **true passthrough**:
```
E1: CAEStreamParser::TrySyncAC3 - AC3 stream detected (6 channels, 48000Hz)
E1: Creating audio stream (codec: ac3, channels: 6, sample rate: 48000, pass-through)
E1: CAESinkAUDIOTRACK::Initializing ... format: AE_FMT_RAW (AE) method: RAW (PT)
    stream-type: STREAM_TYPE_AC3  channels: 2
E4: (identical) codec: ac3 ... pass-through, stream-type: STREAM_TYPE_AC3
```
**AC3 regression: NONE.** AC3 passthrough is fully intact after the patch.

### 5.2 DTS — E2 (main) and E3 (repeat)
Kodi plays DTS as **no pass-through → transcoded to AC3**:
```
E2: Creating audio stream (codec: dts, channels: 6, sample rate: 48000, no pass-through)
E2: CActiveAESink::OpenSink - initialize sink
E2: CAESinkAUDIOTRACK::Initializing ... format: AE_FMT_RAW (AE) method: RAW (PT)
    stream-type: STREAM_TYPE_AC3  channels: 2
E2: CAEEncoderFFmpeg::Initialize - AC3 encoder ready
E3: (identical) codec: dts ... no pass-through ... stream-type: STREAM_TYPE_AC3 ... AC3 encoder ready
```
**DTS result: FAILED to pass through.** Both runs, Kodi detects `codec: dts` but decides
**`no pass-through`**, then spin-up-encodes the DTS to **AC3** (`CAEEncoderFFmpeg::Initialize - AC3
encoder ready`) and opens the sink as `STREAM_TYPE_AC3`. The bitstream delivered to the audio HAL is
**AC3, not DTS**. The R5 patch does not change this — the decision is made in the Kodi/Android audio
path (userspace), which the kernel `mik.ko` 1-byte change cannot affect.

### 5.3 Repeated DTS — consistent
E3 reproduces E2 exactly (`codec: dts … no pass-through … stream-type: STREAM_TYPE_AC3 … AC3 encoder
ready`). Not a one-off.

### 5.4 Repeated AC3 — consistent, no regression
E4 reproduces E1 exactly (`codec: ac3 … pass-through … STREAM_TYPE_AC3`). System stable.

### 5.5 Kernel HAL traces (dmesg)
All four windows show the identical MStar HAL path (`MI_PCM_Open HW DMA Reader1`,
`_MI_AOUT_NotifyConnectInput hAout:0x17000000`, `_MI_PCM_SetChVolFadingMode`) — no SPDIF/SetMode/DTS
divergence is visible in kernel logs. This is the logging blind spot established in Phase A: the
`mik.ko` output-mode decision does not emit observable trace lines, so it cannot confirm the AVR lock
directly. The decisive evidence is the **Kodi userspace** sink decision above.

## 6. Log evidence (key lines)

| signal | AC3 (E1/E4) | DTS (E2/E3) |
|---|---|---|
| Kodi stream creation | `codec: ac3 … pass-through` | `codec: dts … no pass-through` |
| AudioTrack sink type | `RAW (PT) STREAM_TYPE_AC3` | `RAW (PT) STREAM_TYPE_AC3` |
| Encoder | (none — true DD passthrough) | `CAEEncoderFFmpeg::Initialize - AC3 encoder ready` |
| Format delivered to HAL | AC3 IEC61937 | **AC3 IEC61937 (transcoded from DTS)** |

## 7. Pioneer VSX-817 result

I cannot read the AVR display directly. The device-side evidence is unambiguous, however: for the DTS
file Kodi delivers **AC3** (STREAM_TYPE_AC3, IEC61937-wrapped, AC3-encoded), so the VSX-817 will
**identify/lock Dolby Digital (AC3), not DTS**. For AC3 it locks DD as before.

This is consistent with — and stronger than — a "trust the AVR" check: the AVR receives AC3 because
Kodi transcodes; it cannot receive DTS because Kodi never emits it. The AVR will confirm **AC3**,
**not DTS**. (User should still glance at the VSX-817 front panel to confirm DD vs PCM; the log
evidence says DD-from-AC3-transcode, i.e. not DTS.)

## 8. Hypothesis verdict: **FALSIFIED** (as the explanation for the DTS failure)

- The R5 patch was built and deployed **correctly** (1-byte change, md5 `03fc2c0d…`, module loaded,
  CFG-identical to source, AC3 disjoint — per `REPORT_phaseD_patch_verification.md`).
- AC3 **still works** → no regression; the patch is safe and the kernel gate is genuinely codec-scoped.
- **DTS passthrough is NOT restored.** Kodi still reports `no pass-through` for DTS and transcodes to
  AC3. Therefore the bitstream never reaches `mik.ko` as DTS, so the patched compressed-output branch
  is never exercised for DTS. The AVR locks AC3, not DTS.

**Interpretation.** Phase C proved the kernel EDID→mode gate *mechanism* is real (DTS-cap-absent →
PCM). But the R5 experiment shows this gate is **not the blocking layer** for the observed failure:
DTS dies *upstream*, in the passthrough/format decision made by Kodi / the Android audio policy from
the sink EDID capability advertisement. The kernel patch flips the downstream gate, but the bitstream
format (DTS vs AC3) is already decided before it, so the patch is moot. The EDID/output-mode gate is a
**real but incidental** detail, not the cause of the DTS-specific failure.

**Likely true root cause (next investigation, not changed here):** the Pioneer VSX-817 / sink EDID as
seen by the audio HAL does not present DTS as a passthrough-capable stream that Kodi will use, so
Kodi falls back to AC3 transcode. Static confirmation: the AudioTrack device enumeration lists DTS
stream types, yet Kodi still chose `no pass-through` — pointing at a Kodi passthrough-policy /
`AudioCapabilities` gating step (userspace), not the kernel `mik.ko` mode selector.

## 9. Failure / revert

The revert trigger conditions (AC3 regression, module fails to load, system unstable) did **NOT**
occur:
- AC3 passthrough intact (E1/E4). ✓ no regression
- `mik` module loaded cleanly (`lsmod`: 901 uses). ✓
- No crashes / instability in any window. ✓

Per the brief, revert is only mandatory on those conditions, so **the patch remains deployed** (as
authorized). Backup is retained at `runtime_phase5/backup/mik.ko.baseline_647b08db`
(md5 `647b08db…`) for an on-request revert:
```bash
adb root; adb remount
adb push backup/mik.ko.baseline_647b08db /data/local/tmp/mik.ko
adb shell "cp /data/local/tmp/mik.ko /vendor/lib/modules/mik.ko && chmod 644 /vendor/lib/modules/mik.ko && sync"
adb reboot
# verify: adb shell md5sum /vendor/lib/modules/mik.ko  -> 647b08db...
```

## 10. Constraint compliance

- Only the verified R5 `mik.ko` patch deployed; exactly one changed byte; no other module/file touched.
- EDID / HDMI / ARC / routing / settings / properties unchanged.
- `utpa2k.ko` (Exp-B) and `libmi3.so` untouched.
- No second patch made.
- Deployment authorized; no code changes after the patch.

---

## Summary

| question | answer |
|---|---|
| Before hash `647b08db…`? | yes (confirmed pre-deploy) |
| After hash `03fc2c0d…`? | yes (confirmed post-reboot, module loaded) |
| Deployment status | **DEPLOYED & ACTIVE** (safe, no instability) |
| AC3 result | **pass-through works** (no regression) |
| DTS result | **no passthrough; Kodi transcodes DTS→AC3; AVR gets AC3** |
| Repeated DTS | identical (not a one-off) |
| Repeated AC3 | identical (no regression) |
| Log evidence | Kodi `codec: dts … no pass-through` → `STREAM_TYPE_AC3` + `AC3 encoder ready` |
| Pioneer AVR | receives **AC3** (DD), **not DTS** |
| R5 EDID/output-mode hypothesis | **FALSIFIED** as the cause of the DTS failure |
| Next step | investigate upstream Kodi/Android passthrough & sink-EDID DTS capability (userspace), not the kernel gate |
