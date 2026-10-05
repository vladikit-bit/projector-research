# DTS reaches the HAL in passthrough format — the break is below AudioFlinger

Date: 2026-09-26
Device: `192.168.0.183:5555`, MT5889, Android 11
Test state: Android «Цифровий вихід» = **Пропустити** (Bypass), OSD `spdif_mode` = **1 (PCM/ARC)**, `libmi3.so` patched v1
Evidence: `forensic_space_bunny_dts_passthrough_reached_evidence_20260926.txt`

No patch applied by this test. No kernel/vendor change. The only device writes were
`Settings.Global` keys and the `action_value` provider row, all itemised in §6.

---

## 1. The test method was wrong twice before; this one is measured

Earlier runs reported `HAL format: AUDIO_FORMAT_PCM_16_BIT` and
`Output devices: AUDIO_DEVICE_OUT_SPEAKER`, and I read that as "no passthrough".
Both readings were artefacts:

- Kodi was not actually playing. The deep-link form
  `am start -n …/.Splash -d "file://…"` lands on Kodi's home screen; the library is
  empty. Playback only starts with an explicit action:
  `am start -a android.intent.action.VIEW -d file:///sdcard/Movies/<file> -t video/mp4 -n net.kodinerds.maven.kodi22/.Splash`
- `AUDIO_DEVICE_OUT_SPEAKER` and the frozen `spdif mode / spdif type` lines are **not**
  indicators here. AC3 — which the user confirms works — shows exactly the same
  readings. Those fields are noise on this configuration.

The field that does discriminate is the **output thread's HAL format**.

---

## 2. Measured result

**AC3 5.1 / 384 kbps** (`test_ac3_51.mp4`):
```
  Standby: no
  HAL format:         0x9000000 (AUDIO_FORMAT_AC3)
  Processing format:  0x9000000 (AUDIO_FORMAT_AC3)
```

**DTS 5.1 / 1536 kb/s** (`test_dts_51.mp4`, Kodi OSD shows it playing 00:08/01:30):
```
  HAL format:         0xb000000 (AUDIO_FORMAT_DTS)
  Processing format:  0xb000000 (AUDIO_FORMAT_DTS)
```

`0x9000000` and `0xb000000` are the AudioFlinger passthrough (IEC61937) encodings —
**DTS is handed to the HAL in exactly the same passthrough form as AC3.**

User report: AC3 decodes on the receiver, DTS does not.

**Therefore the break is strictly below AudioFlinger.** Kodi passthrough, AudioFlinger
format negotiation and the MediaTek HAL's input stage are all confirmed working for
DTS. Whatever refuses it is the decoder/output path beneath.

---

## 3. What the applied patch actually is

`reapply_pt_fix.sh` replaces `/vendor/lib/libmi3.so`. Device confirms it is applied:

```
$ adb shell md5sum /vendor/lib/libmi3.so
2e34d0c973472405197dd7c7f20bdb92     == MD5_PATCHED in the script
```

Byte deltas against the pristine copy, all in one function:

| variant | file offset | change | installed |
|---|---|---|---|
| `libmi3_patched.so` | `0x06163e` | `*(r5-4) \|= 0x2E0` | **yes** |
| `libmi3_patched_v2.so` | `0x06163e` | `*(r5-4) \|= 0x2E1` | no |
| `libmi3_dts_v3.so` | `0x06163e` | `*(r5-4) \|= 0x070002E1` | no |

The enclosing symbol is **`MI_AUDIO_GetCaps` @ 0x625b5**, size 224 — a *capability
query*, not a mode setter.

**Original:**
```asm
0x6263f  ldr  r0, [pc,#0x4c]
0x62641  add  r0, pc
0x62643  ldrb r0, [r0]            ; global byte
0x62645  cmp  r0, #0x40
0x62647  blo  #0x62657             ; below 0x40 -> return 0 without filling caps
...
0x62657  movs r4, #0               ; return 0
```

**Patched (v1, installed):**
```asm
0x6263f  ldr  r1, [r5, #-0x4]      ; 32-bit field at offset 12 of the caps payload
0x62643  movw r2, #0x2E0
0x62647  orrs r1, r2
0x62649  str  r1, [r5, #-0x4]      ; force the bits on
0x6264d  movs r4, #0
0x6264f  b.w  #0x62659             ; always take the success path
```

So the patch does two things: it **removes a refusal** (a global flag below `0x40` made
the function bail out and report no capabilities) and it **forces constant bits into
the capability field**.

That is why AC3 works: the platform is now told it supports those output formats, so
it passes AC3 through instead of decoding to PCM. It is a **capability declaration
lie**, not a routing change.

---

## 4. Why DTS is still refused

The installed bits are `0x2E0` (bits 5, 6, 7, 9). The uninstalled DTS variant
`libmi3_dts_v3.so` is `0x070002E1` — the installed bits plus bit 0 and bits 24–26.

Given that v1 already unlocks AC3 and v3 is the variant explicitly named for DTS, the
extra bits 24–26 are the natural place where "DTS output is supported" is declared.
The stream already arrives as `AUDIO_FORMAT_DTS`; if the caps word does not claim DTS
output, the path below has no licence to emit it.

This is a hypothesis, not a proven bit mapping — the caps struct header was not located
and the semantic meaning of offsets/bits is inferred from the variant naming and the
observed AC3-only outcome. It is, however, directly testable.

---

## 5. Proposed next step — needs your consent

Swapping `/vendor/lib/libmi3.so` to `libmi3_dts_v3.so` (already present on the device
at `/data/local/tmp/libmi3_dts_v3.so`, md5 `4c199a5c739e61741ba731534fbb3bf4`) and
re-running the same DTS test.

This is a **patch change** and I will not do it without your explicit go-ahead, per the
standing rule. It is reversible — the script pattern re-copies the file and restarts
`vendor.audio-hal`, and the pristine copy is in the workspace at
`spdif_audio_investigation/libs/libmi3.so`.

Expected observations either way:
- If DTS decodes on the receiver → the caps bits were the gate; then the remaining work
  is narrow and the earlier "layer 4 decoder/Auth" concern is largely resolved.
- If still silent → the caps are not the gate, and the next probe is the `mik.ko` EDID
  arm (`_u32CurEdidSupportList`, `dts:0` in ARC capability).

---

## 6. Device state left behind

| what | final state |
|---|---|
| `action_value.spdif_mode` | `1` (PCM/ARC) — set by this test |
| Android «Цифровий вихід» | **Пропустити** (Bypass) — restored by the user during this session |
| `libmi3.so` | v1 patch, untouched by this test |
| `Settings.Global.running_package_name` | `com.bkdisplay.screensaver` |
| `Settings.Global.pre_package_name` | `null` |
| Kodi | force-stopped after the capture |
| `/data/local/tmp/test_dts_51.mp4` | removed; the original `/sdcard/Movies/*.mp4` files left in place |

The «Цифровий вихід» row was left on «Автоматично» by an earlier step in this session,
which is why there was no sound at one point — the patch is only effective in
«Пропустити». That has been corrected.
