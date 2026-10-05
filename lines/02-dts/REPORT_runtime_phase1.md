# REPORT_runtime_phase1.md
## Runtime Phase 1 — Differential AC3-vs-DTS characterization on the patched experimental baseline

**Device**: Thundeal TD98 Pro / C50A — MediaTek MT5889, Android 11, kernel 4.19.116
**Transport**: ADB over TCP, `192.168.0.183:5555`, `adb root` (uid=0, `u:r:su:s0`)
**Session window**: 2026-09-03 22:50 → 23:16 (device local, Europe/Helsinki)

---

## A. Baseline declaration — what this device is

> **This is the patched experimental baseline used for differential AC3-vs-DTS runtime characterization.**

It is **not** a clean stock device, and it is **not** fully documented by the previously supplied
baseline description. The modifications present are treated as **known controlled prerequisites**,
not as contamination. Two modifications were found that were **not** in the supplied description.

### A.1 Deployed component hashes (verified on-device with `md5sum`)

| Component | Deployed md5 | Classification |
|---|---|---|
| `/vendor/lib/modules/utpa2k.ko` | `4c5e6fbb10e24abc3b8d2b0490507983` | **Experiment B** (confirmed deployed) |
| `/vendor/lib/modules/mik.ko` | `647b08db19dcbc20b7d5c6a4a4e36e9b` | ⚠ **UNDOCUMENTED PATCH** — see A.2 |
| `/vendor/lib/modules/dtv_driver.ko` | `6d72fbebe034a21340c85a01f2a25be5` | Stock (matches local original) |
| `/vendor/lib/libmi3.so` | `4c199a5c739e61741ba731534fbb3bf4` | **libmi3_dts_v3.so** (DTS capability patch v3) |
| `/vendor/lib/hw/audio.primary.mt5889.so` | `8c11c348ac7c0ce5bf48ccb36a83ab9c` | **STOCK** (mtime 2009-01-01) |
| `/vendor/etc/audio_policy_configuration.xml` | — | ⚠ **UNDOCUMENTED MODIFICATION** — see A.3 |

**Important resolution of the "other previous HAL/APM changes" ambiguity:** the HAL
(`audio.primary.mt5889.so`) is byte-identical to the stock copy. **No APM or HAL binary patch is
live.** The only patched components are `utpa2k.ko`, `mik.ko`, `libmi3.so`, plus one XML config
file.

### A.2 ⚠ UNDOCUMENTED MODIFICATION #1 — `mik.ko` is patched

```
deployed  /vendor/lib/modules/mik.ko      = 647b08db19dcbc20b7d5c6a4a4e36e9b
workspace kmods/mik.ko ("host original")  = c1421040dbcfff415f9417f482189435
workspace libs/mik_dts_hdmienable.ko      = 647b08db19dcbc20b7d5c6a4a4e36e9b   <-- IDENTICAL TO DEPLOYED

DIFF: exactly 4 bytes at file offset 0x9A334  (.text + 0x97840)
  original: 03 01 00 1A   ->  bne #0x97c54
  deployed: 00 00 A0 E1   ->  mov r0, r0   (NOP)
  Size-preserving: 8084248 bytes both.

CONTAINING FUNCTION: _MI_AOUT_MonitorTask  @ .text + 0x971a8  (+0x698)
  0x97838: ldrb r0,[g2]
  0x9783c: cmp  r0,#1
  0x97840: bne  #0x97c54        <<< NOP'ed out
```

Effect: the original control flow required **two** conditions to reach the HDMI/DTS enable block
(`g1 == 0` **AND** `g2 == 1`). The NOP removes the second condition, so the block now executes
whenever `g1 == 0`.

**Consequence for this investigation:** the `_MI_AOUT_MonitorTask` gate is **already partially
bypassed**, yet DTS still fails. Therefore **DTS failure cannot be attributed to an untouched
MonitorTask gate** — that lever has already been pulled and was insufficient.

### A.3 ⚠ UNDOCUMENTED MODIFICATION #2 — `audio_policy_configuration.xml` is modified

`-rw-r--r-- 1 root root 35616 2026-08-29 21:45 /vendor/etc/audio_policy_configuration.xml`

Every other `/vendor/etc/*.xml` carries the stock build timestamp **2009-01-01**. This one file was
rewritten on **2026-08-29 21:45**.

Content consequence: `DTS` and `DTS_HD` profiles were added to the `"offload output"` and
`"hw_av_sync output"` **mixPorts**, but **not** to the `devicePort Speaker`:

```xml
<devicePort tagName="Speaker" type="AUDIO_DEVICE_OUT_SPEAKER" role="sink">
    <profile format="AUDIO_FORMAT_PCM_16_BIT" .../>
    <profile format="AUDIO_FORMAT_AC3"        .../>
    <profile format="AUDIO_FORMAT_E_AC3"      .../>
    <profile format="AUDIO_FORMAT_E_AC3_JOC"  .../>
    <!-- NO DTS, NO DTS_HD -->
</devicePort>
```

**This edit is not the DTS blocker** — see section D.3, where Kodi is shown enumerating full DTS
capability at runtime. It is recorded here so the baseline is honest.

### A.4 Runtime services / topology

- `audioserver` (pid 351), `vendor.audio-hal` (pid 331 → `mstar.hardware.audio.service`),
  `cpu-audio` (pid 449), `vendor.cec-hal-1-0` (pid 328)
- Kernel audio threads present: `[AUDIO]` (293), `[Audio Mon Task]` (347)
- Kernel log is **99% flooded** with `** CEC system busy!!!`
  (3127 of 3160 lines in the baseline capture). All kernel captures in this phase were
  CEC-filtered.

---

## B. EDID / SAD — the highest-priority runtime target

### B.1 What does **not** exist

| Item | Result |
|---|---|
| `Customer_SAD.ini` | **Does not exist anywhere on the device** (searched all mounts) |
| Any `*SAD*` file | **None** |
| `EdidBinPath` / `EDIDBin` | **None** |
| Downstream-sink EDID sysfs node | **None** |
| `/sys/kernel/mik/MI_AUDIO` | **`-EACCES`** — kernel handler refuses, see B.3 |
| MStar diagnostic binaries (`mi_*`, `utopia*`, `tvservice`) | **None** on `/vendor/bin` or `/system/bin` |

### B.2 What **does** exist — and its correct classification

`/vendor/cusdata/bsp/common/EDID_BIN/` contains 36 × `Mstar_EDID*.bin` + `VGA_EDID.bin`, selected
by `/vendor/cusdata/bsp/board/BD_MT5889_H2V1-B4-S/edid_cfg.ini`:

```ini
[HDMI_EDID_1] HDMI_EDID_File=... HDMI_EDID_File_2_1=... HDMI_EDID_File_2_0=... HDMI_EDID_File_1_4=...
              bEDIDEnabled=1  eEdidVersion=3  bUseDefaultValue=0
[HDMI_EDID_2..4]  (same shape, CEC physical addrs 0x1000 / 0x2000 / 0x3000 / 0x4000)
[VGA_EDID] ...
```

> **CLASSIFICATION: RX / presented — configuration template.**
> These 36 files are the EDID the **projector presents on its own HDMI inputs**. They describe what
> the projector tells *connected sources* it can accept. They have **nothing to do with the
> downstream audio sink**. Do not reason about DTS using these files.

### B.3 The mik sysfs debug nodes are hard-gated

All 24 nodes under `/sys/kernel/mik/` (`MI_AUDIO`, `MI_AOUT`, `MI_PCM`, `MI_DISP`, `MI_CEC`, …) are
`-rw-r--r-- root root` yet return `Permission denied` **even as uid=0 `u:r:su:s0`**, including via
`dd`. This is the kernel `show` handler returning `-EACCES`, not a permission-bit problem. The EDID
state is therefore **not reachable through sysfs**.

The only related knob found is `/sys/module/mik/parameters/mi_aout_amp_debuglevel` (default **32**),
plus `mi_sys_cfg_debuglevel` (32), `mi_sys_cfg_file_debuglevel` (0). Raising the AOUT level 32 → 63
did **not** unlock any SPDIF / EDID / SetMode message class (verified, then restored to 32).

### B.4 The real EDID API — found in the deployed `libmi3.so`

```
mi_aout_GetCurEdid      -> ioctl MI_DEV_IOC_AOUT_INNER_GET_CUR_EDID        -> pu32CurEdidSupportList
mi_aout_EdidAutoSetup   -> ioctl MI_DEV_IOC_AOUT_INNER_EDID_AUTO_SETUP
mi_hwcaps_GetAudioCaps  -> MI_AUDIO_GetCaps / MI_DEV_IOC_AUDIO_GETCAPS
```

This confirms the static model's predicted mechanism (an EDID-derived capability bitmap consumed by
an auto-mode function) **exists and is reachable from userspace** via `libmi3.so`. It is only the
*observation* of its value that is blocked by B.3.

### B.5 The EDID-derived capability **as seen by the HAL** — all zero

From `dumpsys media.audio_flinger`, section `===== Audio HAL Dump =====`, captured **during
playback**, identical in the AC3 and the DTS trace:

```
Supported ARC capability: dd:0, ddp:0(atmos:0), dts:0, aac:0, mat-truhd:0(Byte3:0), dtshd:0,
```

**Every field zero.** The HAL string is `Supported ARC capability: ` and the HAL contains
`utils_SADFormat2AudioFormat(sadCodecType)`, `utils_SADSupportSamplerate`,
`utils_SADSupportChannelMask`, `utils_SADFormat2HdmiEncodingsString` — i.e. this line is the
SAD/EDID-derived capability list.

> **Interpretation (evidence-bounded):** the live EDID-derived capability list advertises
> **nothing** — not AC3, not DTS. This single observation invalidates any theory in which AC3 works
> *because* the sink advertises AC3. AC3 works **despite** an all-zero capability list. See D.6.

### B.6 HDMI topology — there is no HDMI output at all

`dumpsys hdmi_control`:

```
mPortInfo:
  port_id: 1, type: HDMI_IN, address: 0x1000, cec: true,  arc: true,  mhl: false
  port_id: 2, type: HDMI_IN, address: 0x2000, cec: true,  arc: false, mhl: false
  port_id: 3, type: HDMI_IN, address: 0x3000, cec: true,  arc: false, mhl: false
  port_id: 4, type: HDMI_IN, address: 0x4000, cec: true,  arc: false, mhl: false
HdmiCecLocalDevice #0:
  mDeviceType: 0                      (TV)
  mArcEstablished: false              <<< ARC NOT established
  mSystemAudioActivated: false        <<< System Audio Control NOT active
  mArcFeatureEnabled: {1=true, 2=false, 3=false, 4=false}
  CEC devices:
    CEC: logical_address: 0x00 device_type: 0 display_name: Android physical_address: 0x0000
CEC message history:
  [S] Report Physical Address / Device Vendor Id / Request Active Source / Active Source
  [H] hotplug port=1 connected=true     (20:18:32)
  -- no [R] received messages at all --
```

`dumpsys tv_input` lists only `com.mediatek.tvinput/.hdmi.HDMIInputService/HW5..HW8` — inputs.
`AudioPolicy` reports exactly one available output device: `AUDIO_DEVICE_OUT_SPEAKER` (0x2).

**Findings:**
1. **All four HDMI ports are inputs.** There is no HDMI TX port. `_MApi_HDMITx_GetEDIDData` has no
   physical TX sink to interrogate.
2. Something **is** physically connected on port 1 (the only ARC-capable port) — hotplug fired.
3. But **ARC is not established**, **System Audio is not activated**, and **no peer CEC device was
   ever discovered** — the only CEC entry is the projector itself. The CEC log contains only
   transmitted (`[S]`) frames; nothing was ever received (`[R]`).
4. Combined with B.5, this is consistent: **no downstream-sink EDID is being read.**

> **Architectural consequence:** the "downstream sink EDID" assumed by the static analysis does not
> exist in the form assumed. Audio leaves this device over a path with **no EDID negotiation at
> all** (SPDIF/optical, or ARC that is not established). Any fix strategy premised on "make the sink
> advertise DTS" has no runtime object to act upon unless ARC is first established.

---

## C. UI audio mode → property → HAL → output mode mapping

### C.1 The `persist.vendor.audio.*` properties are INERT — proven

Set on the device:

```
[persist.vendor.audio.spdif.mode]   : [BYPASS]
[persist.vendor.audio.spdif.type]   : [DTS]
[persist.vendor.audio.hdmi_tx.mode] : [BYPASS]
[persist.vendor.audio.hdmi_tx.type] : [DTS]
[persist.vendor.audio.hdmi_arc.mode]: [BYPASS]
[persist.vendor.audio.hdmi_arc.type]: [DTS]
```

Yet **no deployed audio binary contains the string `vendor.audio.spdif`**:

```
/vendor/lib/hw/audio.primary.mt5889.so : 0 occurrences
/vendor/lib/libmi3.so                  : 0 occurrences
/vendor/lib/libutopia.so               : 0 occurrences
```

> **This is direct runtime confirmation of the static model's conclusion.** `spdif.type = DTS` does
> **not** reach the output-mode layer, because nothing reads it. These six properties are
> experimental leftovers and are **not a lever**.

### C.2 What the HAL actually reports, during playback, in both traces

```
 - hdmi arc mode: ui(auto), current(auto)
 - hdmi arc type: ui(not-specified), current(not-specified)
 - hdmi tx  mode: ui(auto), current(auto)
 - hdmi tx  type: ui(not-specified), current(not-specified)
 - spdif     mode: ui(auto), current(auto)
 - spdif     type: ui(not-specified), current(not-specified)
 - Dolby version: 10
 - Dolby bypass: 0
 - DTS DRC scale: 0
 - spdif delay: 0
```

**Identical for AC3 and DTS.** The `ui(...)` and `current(...)` values track each other in every
case — the HAL's effective state equals its UI state, and both are the *default*.

### C.3 What the HAL *can* do, and which keys it wants

The HAL binary contains:

```
SetSpdifOutputMode=PCM      SetSpdifOutputType=NONE
SetSpdifOutputMode=AUTO     SetSpdifOutputType=AC3
SetSpdifOutputMode=BYPASS   SetSpdifOutputType=DTS
SetSpdifOutputMode=TRANSCODE
keys: spdif_mode  spdif_type  hdmi_tx_mode  hdmi_tx_type  HDMI_ARC  sound_type
fmt : "%s: spdif mode = ui(%s), current(%s); spdif type = ui(%s), current(%s); digital setting = Mode(%s), Type(%s)"
```

Settings DB contents (global) — note what is **missing**:

```
sound_spdif_type=4     sound_spdif_delay=0    sound_earc=0
hdmi_arc_control_enabled=1     hdmi_system_audio_control_enabled=1     hdmi_control_enabled=1
sound_dts_drc=0  sound_dts_studio_enable=0  sound_advanced_dolby_atmos=0  sound_advanced_dolby_ap=0
-- NO key named  spdif_mode / spdif_type / hdmi_tx_mode / hdmi_tx_type --
```

**Conclusion (R1-C):** the UI "Pass-through" path that would force a nonPCM hardware mode is
**never exercised**. The HAL is left in `AUTO` / `not-specified` for all three outputs. There is no
evidence that any UI setting currently forces a nonPCM hardware mode; the output mode is whatever
`AUTO` resolves to internally, and the keys that would override it are absent from the settings DB.

---

## D. AC3 control trace vs DTS failure trace — the differential

Identical procedure for both (`tools/r1_trace.sh`): force-stop Kodi → clear logcat + dmesg → launch
the file → settle 14 s → capture **during** playback.

- Subject: Kodi **21.2** (`net.kodinerds.maven.kodi22`, UID 10061, Kodinerds build)
- Kodi audio config: `audiooutput.passthrough=true`,
  `audiooutput.passthroughdevice=AUDIOTRACK:AudioTrack (RAW)|Android IEC packer`,
  `ac3passthrough=true`, **`dtspassthrough=true`**, `eac3passthrough=false`,
  `dtshdpassthrough=false`, `dtshdcorefallback=true`, `ac3transcode=true`

### D.1 Playback was genuinely active in both captures (not idle dumps)

**AC3** — 23:09:03:
```
VideoPlayer::OpenFile: /sdcard/testmedia/test_ac3_51.mp4
CAEStreamParser::TrySyncAC3 - AC3 stream detected (6 channels, 48000Hz)
Creating audio stream (codec: ac3, channels: 6, sample rate: 48000, pass-through)
Trying to open: samplerate: 48000, channelMask: 12, encoding: 5
CAESinkAUDIOTRACK::Initializing with: m_sampleRate: 48000 format: AE_FMT_RAW (AE)
    method: RAW (PT) stream-type: STREAM_TYPE_AC3  min_buffer_size: 24576
    m_frames: 24576  m_frameSize: 1  channels: 2
```

**DTS** — 23:10:01:
```
VideoPlayer::OpenFile: /sdcard/testmedia/test_dts_51.mp4
CAEStreamParser::SyncDTS - dts stream detected (6 channels, 48000Hz, 16bit BE, period: 512,
    syncword: 0x7ffe8001, target rate: 0x18, framesize 2012)
Creating audio stream (codec: dts, channels: 6, sample rate: 48000, pass-through)
Trying to open: samplerate: 48000, channelMask: 12, encoding: 7
CAESinkAUDIOTRACK::Initializing with: m_sampleRate: 48000 format: AE_FMT_RAW (AE)
    method: RAW (PT) stream-type: STREAM_TYPE_DTS_512  min_buffer_size: 32192
    m_frames: 32192  m_frameSize: 1  channels: 2
```

`encoding: 5` = `ENCODING_AC3`, `encoding: 7` = `ENCODING_DTS`.
**Kodi opens the DTS passthrough AudioTrack successfully. There is no error at this layer.**

### D.2 AudioFlinger — both reach the HAL as OFFLOAD, both are actively writing

| | **AC3 (working)** | **DTS (failing)** |
|---|---|---|
| Thread | `AudioOut_45`, tid 11625, type 4 **OFFLOAD** | `AudioOut_4D`, tid 11890, type 4 **OFFLOAD** |
| I/O handle | 69 | 77 |
| HAL format | `0x9000000` **AUDIO_FORMAT_AC3** | `0xb000000` **AUDIO_FORMAT_DTS** |
| HAL frame count / buffer | 784 / 784 B | 4096 / 4096 B |
| Flags | `0x11` DIRECT\|COMPRESS_OFFLOAD | `0x11` DIRECT\|COMPRESS_OFFLOAD |
| Output devices | `0x2` Speaker | `0x2` Speaker |
| Standby | no | no |
| Total writes | 1712 | 1505 |
| Frames written | 395 920 | 1 568 768 |
| Last write occurred | 88 ms | 31 ms |
| Timestamp rate | **0.999837** | **0.976836** |
| localSR | **47999.6** | **47917.8** |
| jitter ms (max) | **2.41** | **15.11** |

> **⚠ CORRECTION to the static-phase notes.** The earlier reports recorded
> `AUDIO_FORMAT_IEC61937 = 0x09000000`. That is **wrong**. The actual constants are
> `AC3 = 0x09000000`, `E_AC3 = 0x0A000000`, `DTS = 0x0B000000`, `DTS_HD = 0x0C000000`,
> `IEC61937 = 0x0D000000`. The earlier "Output 61, Format 0x09000000" observation was **AC3**,
> not IEC61937. Any static reasoning that used the old constant must be re-checked.

Bitrate sanity check — DTS is delivered at full rate, it is not being starved:
- AC3: 395 920 B over ≈6 s ≈ 63 kB/s ≈ 505 kbps (AC3 5.1 448–640 kbps) ✓
- DTS: 1 568 768 B over ≈8 s ≈ 196 kB/s ≈ 1569 kbps (DTS core 1509 kbps) ✓

### D.3 The app-side capability list is NOT the blocker

Kodi enumerates full DTS capability at startup:

```
Enumerated AUDIOTRACK devices:
  Device 1  AudioTrack (IEC)   m_deviceType: AE_DEVTYPE_HDMI
    m_streamTypes: STREAM_TYPE_AC3, STREAM_TYPE_DTSHD_CORE, STREAM_TYPE_DTS_1024,
                   STREAM_TYPE_DTS_2048, STREAM_TYPE_DTS_512, STREAM_TYPE_EAC3,
                   STREAM_TYPE_DTSHD, STREAM_TYPE_DTSHD_MA, STREAM_TYPE_TRUEHD
  Device 2  AudioTrack (RAW)   (same set, different order)
```

So the missing `DTS` entry in `devicePort Speaker` (A.3) does **not** prevent Kodi from offering or
attempting DTS. Recorded and dismissed as a cause.

### D.4 HAL dump — byte-for-byte identical except the codec

Full diff of the `===== Audio HAL Dump =====` block, AC3 vs DTS:

```diff
24c24
<  - source: id(30) ... format(0x09000000), mix hw_module(10) handle(69) stream(-1)
---
>  - source: id(34) ... format(0x0b000000), mix hw_module(10) handle(77) stream(-1)
37c37
<  - handle: 69
---
>  - handle: 77
42c42
< AudioHAL audio codec type: AC3
---
> AudioHAL audio codec type: DTS
```

**Nothing else differs.** Same `Dolby version: 10`, `Dolby bypass: 0`, `DTS DRC scale: 0`, same
`auto`/`not-specified` for all three outputs, same all-zero `Supported ARC capability`, no rejected
`setParameters` in either trace.

> **The HAL correctly identifies DTS (`AudioHAL audio codec type: DTS`) and passes it down through
> an identical configuration. There is no HAL-level gate that treats DTS differently.**

### D.5 Kernel — identical sequences, DTS decoder starts successfully

Normalized diff (PIDs and addresses masked) of all audio kernel messages:

```diff
< <UTPA_ERR>[Utopia][[AUDIO][ERROR]]: [MAD]HAL_MAD_GetAudioInfo2: Unknown dolby type (255) adec_id(0)
< <UTPA_ERR>[Utopia][[AUDIO][ERROR]]: [MAD]HAL_MAD_GetAudioInfo2: Unknown dolby type (255) adec_id(0)
< <MI3_DEBUG>[PID:X][MI_AUDIO_Start:9311]Codec:5,hAudio:0xX,eRet:0xX
---
> <MI3_DEBUG>[PID:X][MI_AUDIO_Start:9311]Codec:9,hAudio:0xX,eRet:0xX
50c46
< <MI3_INFO>reallocate memory: _u32DefaultWriteBufferSize:0 -> stWriteParams.u32BufSize:1536
---
> <MI3_INFO>reallocate memory: _u32DefaultWriteBufferSize:0 -> stWriteParams.u32BufSize:2012
```

- `MI_AUDIO_Start Codec:9, eRet:0x0` — **the DTS decoder starts and returns success.**
  (`Codec:5` = AC3, `Codec:9` = DTS, correlated against the HAL's own codec-type line.)
- `u32BufSize: 2012` — exactly the DTS frame size Kodi's parser reported. **A structurally valid
  DTS elementary stream is being ingested.**
- `HAL_MAD_GetAudioInfo2: Unknown dolby type (255)` appears **only in the AC3 trace** — it is a
  Dolby-specific info query. Its absence for DTS means **DTS never enters any equivalent
  codec-specific output negotiation**.

**No `SPDIF_SetMode`, `SetHdmiAutoMode`, `SetDigitalMode` or nonPCM message appears in either
trace** — including the verbose re-run at `mi_aout_amp_debuglevel = 63`.

### D.6 ⭐ FIRST MEANINGFUL RUNTIME DIVERGENCE

The divergence is **not a failure point**. DTS never errors anywhere. Everything through
`MI_AUDIO_Start` is successful and equivalent. The first *behavioural* differences are:

1. **The output mode never reacts to DTS.** In both traces the HAL keeps all three outputs at
   `AUTO` / `not-specified`. AC3 additionally triggers the Dolby-specific `HAL_MAD_GetAudioInfo2`
   path; **DTS triggers no comparable output-mode negotiation.**

2. **The DTS offload hardware clock is wrong.** AC3: rate `0.999837`, localSR `47999.6`, jitter max
   `2.41 ms` (locked). DTS: rate `0.976836`, localSR `47917.8` (−0.17%), jitter max `15.11 ms`.
   The DSP is **not consuming the DTS stream at real-time rate**, while it consumes AC3 correctly.

3. **Position: between the MAD/MI decoder layer and the final SPDIF/HDMI transmit stage.**
   Decoder started (`Codec:9`, `eRet:0`); frames delivered at full bitrate (≈1569 kbps); but the
   hardware timestamp runs slow and no transmit-mode transition occurs.

### D.7 ⭐ The EDID theory is REFUTED as the AC3/DTS differentiator

The static model predicted: *first divergence = sink EDID missing DTS bit 0x80.*

Runtime says otherwise. **AC3 works while the EDID-derived capability list is all-zero — including
`dd:0`.** If the EDID capability gated passthrough, AC3 could not work either. Therefore:

- The EDID/ARC capability list is **not** what enables AC3.
- Consequently **DTS failure cannot be explained by a missing EDID DTS bit**, because AC3 succeeds
  without its own EDID bit.
- The DTS/AC3 asymmetry must come from a **DTS-specific gate that is EDID-independent**, located
  below `MI_AUDIO_Start`.

The two remaining candidate mechanisms, both DTS-specific and both below the observed layer:

| # | Candidate | Why it fits | How it would show |
|---|---|---|---|
| 1 | **AUTH / DSP-license state** (`g_AudioVars2`: `0x440` missing-mask, `0x444` avail-mask, `0x4d0` DSP tier, `0x4d8` DTS level, `0x582` DTS:X flag) | Experiment B was *intended* to set `0x4d0=4`, `0x4d8=3`, clear DTS bits in `0x440`, `0x582=1`. If the bypass is incomplete, DTS decoding is licensed off while AC3 (Dolby) is licensed on. | Decoder starts but produces no/incorrect output; clock drift |
| 2 | **DSP transmit packer not instantiated for DTS** (`DTSX_CORE2_API_SDO_Packer`, `DTSX_Transcoder`, `force transcode, ddenc_owner, ddpenc_owner`) | AC3 has the Dolby MS12 / `ddenc` path (`Dolby version: 10`, `HAL_MAD_GetAudioInfo2`) as an EDID-independent fallback; DTS has no equivalent. | Decoder starts, no IEC61937 burst emitted, hardware clock never locks |

Both are consistent with all observations. **This phase cannot distinguish them** — that requires
R2.

---

## E. Interpretation-rule compliance check

| Rule | Verdict |
|---|---|
| Do not assume `spdif.type=DTS` means EDID advertises DTS | ✅ **Proved inert** — no deployed binary reads `vendor.audio.spdif` (C.1) |
| Do not assume DTS failure means AUTH bypass failed | ✅ Not assumed — listed as candidate #1, unverified (D.7) |
| Do not assume AC3 success proves every DTS prerequisite | ✅ AC3 success used only to *refute* the EDID theory (D.7) |
| Do not assume decoder start proves valid DTS bitstream transmitted | ✅ `Codec:9 eRet:0` + `u32BufSize:2012` + full bitrate = **valid ingest**; transmit **not** proven |
| Do not assume PCM output proves decoder never started | ✅ N/A — DTS decoder demonstrably started |
| Do not treat an idle dump as evidence of active playback | ✅ Both captures verified active via Kodi log + `Last write occurred` 88/31 ms (D.1) |
| Do not assume an EDID file/profile is consumed by SetHdmiAutoMode | ✅ 36 EDID_BIN files classified as RX templates and excluded (B.2) |

---

## F. Stop-condition compliance

- ✅ **No binary patch created or deployed.** No `.ko`, `.so`, or HAL/APM binary was written.
- ✅ **No EDID invented or replaced.** No EDID file was touched.
- ✅ **No irreversible change.** One reversible runtime knob was used:
  `/sys/module/mik/parameters/mi_aout_amp_debuglevel` 32 → 63 → **restored to 32** (verified).
- ✅ All device interaction was read-only (`getprop`, `lsmod`, `md5sum`, `ls`, `cat`, `dumpsys`,
  `ps`, `dmesg`, `logcat`, `adb pull`, `adb push` of test media to `/sdcard/testmedia/`).
- ⚠ Playback control used `am force-stop` / `am start` on Kodi — required by R1-D, non-destructive.

---

## G. Artifacts (all new, uniquely named, nothing overwritten)

Directory: `spdif_audio_investigation/runtime_phase1/`

**Baseline (R1-A)** — `R1_baseline_identity.txt`, `R1_baseline_lsmod.txt`,
`R1_baseline_modules.txt`, `R1_baseline_libs.txt`, `R1_baseline_props.txt`,
`R1_baseline_audioflinger.txt`, `R1_baseline_audiopolicy.txt`, `R1_audio_policy_xml.txt`,
`R1_baseline_playing_state.txt`, `R1_baseline_dmesg.txt`, `R1_baseline_logcat.txt`

**EDID hunt (R1-B)** — `R1_edid_discovery.txt`, `R1_edid_cfg.txt`, `R1_edid_MI_AUDIO_sysfs.txt`,
`R1_sysfs_hdmi.txt`, `R1_sysfs_hdmitx.txt`, `R1_mik_ko_patch_analysis.txt`,
`pulled/mik.device.ko`

**Differential traces (R1-D)** — per-format, per-timestamp:
```
R1_trace_ac3_20260903_230852_{audioflinger,audiopolicy,dumpsysaudio,ps,logcat,kodilog,dmesg}_during.txt
R1_trace_dts_20260903_230951_{...}_during.txt
R1_verbose_dts_20260903_231624_dmesg_verbose.txt
R1_verbose_ac3_20260903_231624_dmesg_verbose.txt
```
Scripts: `tools/r1_trace.sh`, `tools/r1_verbose_trace.sh`

---

## H. Answers to the specific DTS questions

1. **Does the sink EDID advertise DTS?** — **No.** It advertises nothing at all (`dd:0, dts:0,
   dtshd:0, …`). And there is no HDMI TX port, so there is arguably no sink EDID to read (B.5/B.6).
2. **Does `spdif.type` reach the output mode?** — **No.** Nothing reads it (C.1).
3. **Does the DTS AudioTrack get created?** — **Yes**, `ENCODING_DTS`, offload, no error (D.1/D.2).
4. **Does the DTS decoder start?** — **Yes**, `MI_AUDIO_Start Codec:9 eRet:0x0` (D.5).
5. **Is a valid DTS bitstream delivered?** — **Yes**, 2012-byte frames at ≈1569 kbps (D.2/D.5).
6. **Does the HAL treat DTS differently?** — **No.** HAL dump is identical except codec label (D.4).
7. **Where does DTS diverge?** — Below `MI_AUDIO_Start`, at the transmit stage: no output-mode
   transition + hardware clock never locks (D.6).

---

## I. What remains unknown after R1

1. The literal value of `pu32CurEdidSupportList` — blocked by the sysfs `-EACCES` gate (B.3).
   Inferred all-zero from the HAL ARC capability line, not read directly.
2. Runtime values of the AUTH/DSP license words `0x440 / 0x444 / 0x4d0 / 0x4d8 / 0x582`.
3. Whether the DSP DTS transmit packer (`DTSX_CORE2_API_SDO_Packer`) is instantiated.
4. The physical output path actually in use (SPDIF optical vs unestablished ARC) — not determined;
   no readable state node exists for either.

---

## J. Smallest viable next intervention (for R2 — not executed in R1)

Ranked by (evidence strength × reversibility) ÷ risk:

1. **Establish ARC first.** `mArcEstablished: false`, no peer CEC device, no received CEC frames.
   Until a downstream sink exists, no EDID-based mechanism has an object. This is a
   **configuration/topology** step, not a patch.
2. **Populate the missing settings keys.** The HAL wants `spdif_mode` / `spdif_type` /
   `hdmi_tx_mode` / `hdmi_tx_type` and has `SetSpdifOutputType=DTS` +
   `SetSpdifOutputMode=BYPASS` implemented; those keys are **absent** from the settings DB while
   the HAL sits at `AUTO`/`not-specified`. Writing them is a pure, reversible **R2-C
   configuration-only** experiment and directly tests whether the output mode can be forced to
   nonPCM for DTS.
3. **Only if 1–2 fail:** revisit candidate #1 (AUTH words) and #2 (DTS packer). Both require
   reading kernel state that is currently unobservable — an observation capability must be built
   first.

**Do not** re-attempt the `mik.ko` MonitorTask NOP: it is already deployed and DTS still fails.

---

## K. Corrections carried forward to the static reports

1. `AUDIO_FORMAT_IEC61937` is **`0x0D000000`**, not `0x09000000`. `0x09000000` is **AC3**.
   (`REPORT_static_final.md` and earlier artifacts using the old value must be re-checked.)
2. `mik.ko` is patched (A.2) — add to the baseline record in all future reports.
3. `/vendor/etc/audio_policy_configuration.xml` is modified (A.3) — add to the baseline record.
4. The EDID gate is **not** the AC3/DTS differentiator (D.7). The static model's "first divergence
   = sink EDID DTS bit 0x80" is **refuted**; the divergence is EDID-independent and sits below
   `MI_AUDIO_Start`.

---

## L. Status

**R1-A** complete (with two undocumented baseline modifications added to the record).
**R1-B** substantially complete; one residual unknown (literal EDID support-list value, sysfs-gated).
**R1-C** complete — the property→HAL path is proven inert.
**R1-D** complete — differential captured with verified active playback; divergence localized.
**R1-E** this report.

**STOP CONDITION HONORED:** no binary patch created or deployed, no EDID invented or replaced,
no irreversible change performed. Awaiting authorization to proceed to R2.
