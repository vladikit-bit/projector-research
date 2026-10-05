# FINDING — The last blocker for DTS passthrough: a MALFORMED capability mask in the active libmi3.so

**Date:** 2026-09-12 (corrected)
**Device:** Thundeal TD98 Pro / C50A (MStar MT5889) · Android 11 · Kodi `22.0-BETA1`
**AVR:** Pioneer VSX-817 (optical SPDIF)

---

## 0. ⚠️ RETRACTION — read this first

An earlier draft of this document claimed a **regression**: that `libmi3.so` md5 `2e34d0c9…` had made
DTS passthrough work (2026-08-26) and that a later `4c199a5c…` broke it.

**That claim is WITHDRAWN.** It rested entirely on `REPORT_PASSTHROUGH_FIX.md` §3
("DTS mp4 → pass-through; HAL parser streaming DTS frames (size 2012) continuously").
**The user has stated that report is WRONG — the sound was only transcoding.**

Consequences:
- There is **no evidence DTS passthrough ever worked** on this device with any `libmi3` patch.
- The "streaming DTS frames (size 2012)" line was **not** real DTS bitstream.
- **No regression may be attributed** to `4c199a5c` (or to `2e34d0c9`). That comparison is void.
- This is consistent with the user's own statement: *"А от чистого дтс ще не було"*.

What follows is the corrected analysis. §1–§2 (the malformed patch) **remain valid as facts about the
binary**; only the "it used to work" framing is retracted.

---

## 1. What the active libmi3.so actually contains (valid, byte-exact)

Three builds, all differing at the **same** file offset `0x06163E` inside `MI_AUDIO_GetCaps`:

```
STOCK      0x06163E: 13 48 78 44 00 78 40 28 | 06 d3 11 48 00 21 00 24 78 44 4c f0 56 ee 00 e0
patched    0x06163E: 55 f8 04 1c | 40 f2 e0 22 | 11 43 | 45 f8 04 1c | 00 f0 03 b8
 (2e34d0c9)          ldr r1,[r5,#4]  MOVW r2,#0x02E0  orr r1,r1,r2  str r1,[r5,#4]  b.w
dts_v3     0x06163E: 55 f8 04 1c | 40 f2 e1 22 | c0 f2 00 72 | 11 43 | 45 f8 04 1c | 00 24 ...
 (4c199a5c)          MOVW r2,#0x02E1  MOVT r2,#0x7200      <-- WRONG MASK      MOVS r4,#0  <-- CLOBBER
```

Decoded (ARM32 Thumb):
- `55 f8 04 1c` = `ldr r1,[r5,#4]`
- `40 f2 e0 22` = `movw r2,#0x02E0` → mask bits **5,6,7,9** (AC3/E-AC3, TrueHD, AAC family, DTS/DTS-HD)
- `11 43`       = `orr r1,r1,r2`
- `45 f8 04 1c` = `str r1,[r5,#4]`
- `00 f0 03 b8` = `b.w` (return to epilogue)

In the **active** build (4c199a5c):
1. The mask is `0x720002E1` (`movw r2,#0x02E1` + `movt r2,#0x7200`) instead of `0x2E0`. The extra
   bit 0 and the high half `0x7200_xxxx` are not the capability bits the HAL expects.
2. `00 24` = `movs r4,#0` **zeroes r4** inside a function that continues executing.

⇒ A suspect, unverified patch whose capability word is malformed. It has **never been shown** to
deliver clean DTS.

## 2. Device inventory (valid)

```
/vendor/lib/libmi3.so                    4c199a5c739e61741ba731534fbb3bf4   <- active, malformed mask
/vendor/lib/libmi3.so.orig               c2b5c57dba4f89228da72a29c71f0c06   <- stock
/vendor/lib/hw/audio.primary.mt5889.so   8c11c348ac7c0ce5bf48ccb36a83ab9c   <- STOCK (HAL patches reverted)
/vendor/lib/modules/mik.ko               03fc2c0d22b231bc478e574557b85473   <- R5 (r6=1), inert for DTS
/vendor/lib/modules/utpa2k.ko            f44ad0a4bcb411eacbe8f677e0d3780f   <- cmd04+isac3
No DEC/SND firmware image is patched (both stock: 4b7e9509… / eb879cdc…).
```

Host copies: `C:/firmware_temp/spdif_audio_investigation/libs/libmi3_{orig,patched,patched_v2,dts_v3}.so`

## 3. Candidate fixes (smallest change, fully reversible)

### Option A — restore stock, establish a clean baseline
```sh
adb root; adb remount
adb shell 'cat /vendor/lib/libmi3.so.orig > /vendor/lib/libmi3.so && sync'
adb shell 'stop vendor.audio-hal; start vendor.audio-hal'
```
Stock md5 `c2b5c57d…`. Use this to observe the **unpatched** behaviour (expected: HAL rejects the
AC3/DTS formats, Kodi reports no passthrough) so the baseline is honest.

### Option B — correct mask `0x2E0` (the `2e34d0c9` build)
```sh
adb push libs/libmi3_patched.so /data/local/tmp/     # md5 2e34d0c9…
adb shell 'cat /data/local/tmp/libmi3_patched.so > /vendor/lib/libmi3.so && sync'
adb shell 'stop vendor.audio-hal; start vendor.audio-hal'
```
Note: this build is **untested for real DTS output** (see §0). It is a candidate, not a fix.

### Option C — patch the active image in place
At file `0x061648` change `e1 22` → `e0 22` and remove the `c0 f2 00 72` (`movt`); restore the
clobbered byte `24` → `21`. Simpler to just use Option B.

## 4. Open question this answers

> **Does DTS bitstream leave userspace as DTS on this device at all?**

Everything found this session (the frozen AC-3 header on the wire under DTS, the complete-but-never-
invoked DEC DTS writer) is consistent with "no". Until a correct capability mask is in place and the
AVR is read live, this remains **unproven in both directions**.

If, after Option B, DTS frames do reach the DSP/SPDIF and the AVR still shows DD/PCM, then the next
and only remaining target is the DEC capability gate described in
`FINDING_rootcause_license_gate.md` (`*(0xB000001E)==5`, sites `0x21AC3`, `0x21B19`, `0x21F0E`,
`0x21F5C`).

## 5. Method note

The Ghidra ground truth used above (`dec03_out.txt` … `dec21_out.txt`) supersedes the earlier Python
decoder output in this area, which invented operands because the region mixes 3-byte `bn.*` and
4-byte `bg.*` instructions. Always confirm AEON operands with Ghidra + the slaspec.
