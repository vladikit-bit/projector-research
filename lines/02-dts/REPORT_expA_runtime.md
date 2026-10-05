# Experiment A — Runtime Activation Report (CURRENT on device)

Дата: 2026-09-02 (runtime), 2026-09-03 (report)
Пристрій: Thundeal TD98 Pro / C50A / Android 11 (kernel 4.9)
ADb: 192.168.0.183:5555 (root = `adb root`)

## 1. Current on-device state (after Exp A activation)

| Item | Value |
|------|-------|
| `/vendor/lib/modules/utpa2k.ko` MD5 | `939cac247eff097f212fef872131df5c` |
| `/vendor/lib/modules/utpa2k.ko` SHA-256 | `0a35a0d086b2a53c60dbe38060d13f7ff7c8ad484fdbb77c2ef4893fbec5eec8` |
| `/vendor/lib/modules/utpa2k.ko` size | `25381336` bytes |
| Init load script | `/vendor/etc/init/init.fusion.rc:37`: `insmod /vendor/lib/modules/utpa2k.ko` |
| Boot marker | `/proc/boottime`: `utpa2k.ko start 5.965s` / `utpa2k.ko end 6.550s` (this boot) |
| Loaded module | `utpa2k 23715840 6 mik,misck,mwgifker,xcker,hdcp2x,hdcp1x, Live 0x00000000 (O)` |
| Holders | `mik`, `misck`, `mwgifker`, `xcker`, `hdcp1x`, `hdcp2x` (6) |
| Bind mount | none (clean after fresh boot) |
| `/vendor` | read-only, not modified |

## 2. Experiment A — what was changed

One ARM `BEQ` at VA `0x424444` (file offset `0x431a98`) replaced with ARM NOP `0xe320f000`.

| Field | Original word (LE) | Patched word (LE) |
|-------|--------------------|-------------------|
| bytes | `2f 00 00 0a` | `00 f0 20 e3` |
| word | `0x0a00002f` (`beq +0x5e` → `0x424508`, IPID `0x7d` fail-branch) | `0xe320f000` (ARM NOP) |

## 3. Static-reconstructed state after one CheckHashkey call

Other 42 `MDrv_AUTH_IPCheck` calls unchanged. The patch forces the IPID `0x7d` success path:

| Field | Value | Comment |
|-------|-------|---------|
| `0x4d0` | `4` | MS12V22 image |
| `0x4d4` | `8` | sub-tier for 0x7d |
| `0x4d8` | `0` | (IP 7 NOT patched, only 0x7d is) |
| `0x43d` | `1` | Dolby premium |
| `0x43e` | `1` | |
| `0x440` | (pre + 0x10) & ~0x10 = pre (no net change for 0x10 bit) | other bits unchanged |
| `0x57e`,`0x57f` | `1` | |
| `0x582` | `0` | (IP 7 not patched) |

Expected runtime observable:
- `HAL_AUDSP_DspLoadCode` switches to `mst_snd_r2_MS12V22` + `mst_codec_r2_MS12V22`
- `SetSystem2` sees `0x4d0==4` (adds `+0x1f` switch-table offset)
- DTS decode is **NOT** started (DTS AUTH missing bits `0x8/0x80/0x20000` still set in `0x440`)

## 4. Audio test results (manually confirmed by user)

| Path | BASELINE (pre-Exp-A) | EXPERIMENT A (now) |
|------|----------------------|---------------------|
| AC3/DD | PASS | **PASS** (manually confirmed) |
| DTS | FAIL | **FAIL** (expected — AUTH still missing) |

## 5. Rollback image (verified byte-for-byte)

`backups/2026-09-02_device_pre_expA/utpa2k_device_current.ko`
- MD5 `3782c6cf036785d1ce720456db47fa4a`
- SHA-256 `f8ba38fb859b3b4034156070a81449841de309f9a4a883a0049551f00a0297e2`
- size `25381336`

## 6. Artifacts produced (NOT touched, NOT deleted)

- `kmods/utpa2k.ko` (host original, MD5 `2fc6e9fc`)
- `kmods/utpa2k_expA_0x4d0_eq4.ko` (Experiment A)
- `backups/2026-09-02_utpa2k_pre_expA/utpa2k.ko` (host backup)
- `backups/2026-09-02_device_pre_expA/utpa2k_device_current.ko` (device baseline)
- `tools/expA_patch_0x7d.py` (generator)
- `runtime_logs/expA/*.log` (activation trace)
- `REPORT_ms12v22_patch_plan.md` (planning report)
- `REPORT_ms12v22_profile.md` (DTS/MS12V22 R2 analysis)

## 7. KEY INFERENCE (still pending runtime confirmation)

- `0x4d0==4` ⇒ MS12V22 R2 images are loaded
- DTS-license state unchanged ⇒ DTS decode should still not start
- AC3 passthrough verified working

---

# Experiment B — Verified Patch Report (NOT yet deployed)

## B.1 Source

- File: `kmods/utpa2k_expA_0x4d0_eq4.ko`
- MD5 `939cac247eff097f212fef872131df5c`
- SHA-256 `0a35a0d086b2a53c60dbe38060d13f7ff7c8ad484fdbb77c2ef4893fbec5eec8`
- size `25381336`

## B.2 Output

- File: `kmods/utpa2k_expB_0x4d8_3_dts_license.ko`
- MD5 `4c5e6fbb10e24abc3b8d2b0490507983`
- SHA-256 `5a5f561621ab9085ca272a5838a58a903b30b23002c1499c0fbfe78fb6219e39`
- size `25381336` (preserved)

## B.3 Byte-exact changes (4 new NOPs, 16 bytes total)

For each row, `BEQ` is replaced with ARM NOP `0xe320f000` (bytes `00 f0 20 e3`):

| IPID | VA | file offset | original bytes (LE) | original word | branch target (BEQ lands here) | semantics when NOPped (success path runs) |
|------|----|-------------|----------------------|---------------|---------------------------------|-----------------------------------------------|
| 0x0f (DTS core) | `0x423a84` | `0x4310d8` | `0d 00 00 0a` | `0x0a00000d` | `0x423ac0` | success: `0x440 |= 0x8` skipped, r5/sl/r6 written (DTS core path) |
| 0x3a (DTS fam)  | `0x423c00` | `0x431254` | `0c 00 00 0a` | `0x0a00000c` | `0x423c38` | success: `0x440 |= 0x80` skipped, r5=9,r6=3,sl=1 |
| 0x12 (DTS-HD)   | `0x423eec` | `0x431540` | `0f 00 00 0a` | `0x0a00000f` | `0x423f30` | success: r5=9,r6=3,sl=1; **r8=2** (then IP 7 overrides) |
| 0x07 (DTS:X)    | `0x4246ec` | `0x431d40` | `12 00 00 0a` | `0x0a000012` | `0x42473c` | success: **r8=3 (final 0x4d8=3)**, r6=4, sl=1, **0x582=1**, F57e/F57f set |

Total 16 bytes changed in 4 slots of 4 bytes each. No other bytes changed.

## B.4 Preserved

- Exp A's `0x424444` ARM NOP (IPID `0x7d`, `0x4d0=4` selector) — preserved exactly
- All other 38 `MDrv_AUTH_IPCheck` fail-branches — untouched
- All bytes outside the 4 NOP slots — identical byte-for-byte to source

## B.5 Expected state after one CheckHashkey call

| Field | Exp A only | Exp B (this) |
|-------|------------|--------------|
| `0x4d0` | `4` (MS12V22) | `4` (preserved) |
| `0x4d4` | `8` | `8` (preserved) |
| `0x4d8` | `0` (DTS AUTH missing) | `3` (DTS:X level — last-writer IP 7) |
| `0x43d` | `1` | `1` (preserved) |
| `0x43e` | `1` | `1` (preserved) |
| `0x440` | DTS-missing bits 0x8/0x80/0x20000 set (each set by a different IPID fail-branch) | corresponding success paths run; the 0x440|=0x8/0x80/0x20000 fail-side effects are skipped (the success paths do not set those bits); whether this constitutes complete DTS licensing is to be determined by runtime test |
| `0x444` | unchanged | Dolby premium bits written |
| `0x57e`/`0x57f` | `1` | `1` (preserved) |
| `0x582` | `0` | `1` (DTS:X flag) |

Net: Exp B adds `0x4d8=3` (DTS:X level), clears all DTS-missing bits, sets `0x582=1`, while keeping MS12V22 image.

## B.6 Generation tool

`tools/expB_patch_0x4d8_3_dts_license.py` (idempotent, refuses to overwrite existing output).

## B.7 Deployment status

**NOT deployed yet.** Awaiting user authorization. Rollback plan: `backups/2026-09-02_device_pre_expA/utpa2k_device_current.ko` is the baseline pre-Exp-A (still on `/vendor` would need to be reinstalled). For Exp B rollback, source `kmods/utpa2k_expA_0x4d0_eq4.ko` is available.

## B.8 What this experiment does NOT change

- EDID, audio_policy, HDMI/SPDIF routing — unchanged
- `audio.primary.mt5889.so` — unchanged
- `libmi3.so` — unchanged
- DTV/Dolby MS12 audio stack — fully intact
- Other 38 `AUTH_IPCheck` calls (0xb/0xc/0x50/0x52/0x51/0x54/8/9/0xa/0x66/0x7d/0x46/0xd/0x1e/0x41/0x49/0x3/0x37/0x45/0x1c/0x38/0x42/0x79/0x7e/0x7c/6/0x40/5/0x7f/0x53/0x73/0x75/0x74) — all fail-branches intact
