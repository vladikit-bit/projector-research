# Passthrough Repair Report — C50A / MT5889: Real AC3/E-AC3/DTS bitstream to SPDIF

Date: 2026-08-26 · Status: **WORKING** (AC3 ✓, E-AC3 ✓, DTS ✓ into the MStar DSP; physical receiver confirmation pending user)

---

## 1. Root cause — two stacked gates

### Gate 1 (primary): `MI_AUDIO_GetCaps()` reports zero codec capability bits
- Chain: Kodi `AudioTrack(ENCODING_AC3)` → AudioPolicy selects the declared `DIRECT|COMPRESS_OFFLOAD` profile → `AudioFlinger::openOutput` → HAL `adev_open_output_stream` → after a successful open, HAL runs `utils_common_out_get(stream, PARAM_9)` → `mi_common_raw_get` case 9 → `MI_AUDIO_GetCaps()` (ioctl `0xC0141002` to `/dev/mik`) and tests the returned mask word[12..16):
  - bit5 = AC3/E-AC3, bit6 = TrueHD, bit7 = AAC family, bit9 = DTS/DTS-HD
  - special rule: DTS/DTS-HD at ≤32 kHz rejected when `adev->0x3c98 == 3`
- On this unit bits were **0** ⇒ HAL logged `unsupport format (0x09000000)` and returned `-EINVAL`; framework surfaced `-38`, so `getMinBufferSize>0` but the AudioTrack constructor failed (`STATE_UNINITIALIZED`) — exactly Kodi's `VerifySinkConfiguration` gate, hence "No passthrough capabilities".
- The capability bits come from the kernel/DSP side; whether they are license-gated or misprogrammed for this product is UNKNOWN (the DSP clearly handles AC3/DTS once accepted — parsers and raw path work).

### Gate 2 (secondary, Kodi-side): passthrough device setting never persists
- `CActiveAE::LoadSettings()` sets `m_settings.passthrough=false` unless a passthrough-capable device exists (fine after Gate-1 fix), and `SupportsRaw()` resolves the device from the LIVE setting `audiooutput.passthroughdevice`.
- `ValidateOutputDevices()` rewrites it in memory each start ('Default' → 'AUDIOTRACK:AudioTrack (RAW)|Android IEC packer') but does **not save** it (`IsSameDeviceAs('Default', RAW)==false`), so `SupportsFormat("Default", …)` always failed → `GetPassthroughStreamType()` returned NULL → FFmpeg decode ("no pass-through").
- Fixed by writing the real device string into `guisettings.xml`.

Not blockers (verified): `FOR_ENCODED_SURROUND=FORCE_NONE` is default state, irrelevant; SPDIF/HDMI/ARC "disconnected" state is cosmetic on this platform — Android never connects them (no native `setDeviceConnectionState` callers; vendor MAudioPolicyManager returns -38) because digital outputs are managed inside the vendor HAL/DSP; routing to device "speaker" still reaches SPDIF TX through the DSP output matrix.

## 2. Fix implemented (smallest change)

1. **`/vendor/lib/libmi3.so` — 20-byte patch of `MI_AUDIO_GetCaps`**: replaced a debug-log-only block at VA 0x6263E with code that ORs `(1<<5)|(1<<6)|(1<<7)|(1<<9)` (0x2E0) into the caps word at `[dest+12]`, then jumps back to the epilogue. No other logic touched.
   - Original preserved at **`/vendor/lib/libmi3.so.orig`** (md5 c2b5c57d…); patched md5 2e34d0….
   - `/vendor` was remounted rw (userdebug); file change survives reboot. Copy also at `/data/local/tmp/libmi3_patched.so`.
2. **Kodi settings** (backup: `/data/local/tmp/guisettings.xml.bak`):
   - `audiooutput.passthroughdevice = AUDIOTRACK:AudioTrack (RAW)|Android IEC packer` (was Default)
   - `audiooutput.eac3passthrough=true`, `audiooutput.dtspassthrough=true`
   - `debug.showloginfo/extralogging=true` (diagnostics; can be turned off)

## 3. Verification evidence

| Test | Result |
|---|---|
| Probe: open AudioTrack AC3/EAC3/DTS/DTS-HD @48k stereo | ACCEPT ×4 (was REJECT) |
| Kodi enumeration | `m_streamTypes: AC3,EAC3,DTSHD_CORE,DTS×3,DTSHD,DTSHD_MA`; "Firmware implements AC3/EAC3/DTS/DTS-HD RAW" |
| AC3 mp4 via VideoPlayer | "Creating audio stream … pass-through"; sink `AE_FMT_RAW stream-type STREAM_TYPE_AC3`; AF OFFLOAD thread ACTIVE format 0x09000000; HAL `ac3_parser` running; dumpsys `AudioHAL audio codec type: AC3` |
| E-AC3 mp4 | pass-through; patch format 0x0a000000; `AudioHAL audio codec type: AC3P` |
| DTS mp4 | pass-through; patch format 0x0b000000; `CAEStreamParser::SyncDTS` detected; HAL parser streaming DTS frames (size 2012) continuously |
| Persistence | After HAL restart + adbd reconnect: probe still ACCEPT ×4; Kodi still enumerates all types |

## 4. Final architecture

```
Kodi 21.2 (AE_FMT_RAW, STREAM_TYPE_AC3/EAC3/DTS)
→ AudioTrack ENCODING_AC3/DTS (framework auto-DIRECT)
→ AudioPolicy offload output (DIRECT|COMPRESS_OFFLOAD)
→ mstar.hardware.audio.service → audio.primary.mt5889.so
→ ac3/dts parser → GRM resource → mi_decoder/raw path → HW DMA Reader
→ MStar MAD DSP (non-PCM; spdif mode AUTO passes it through)
→ SPDIF TX (IEC61937) → AV receiver        [receiver lock: confirm on AVR panel]
```
PCM/UI sounds continue on the primary mixer thread (speaker; DSP matrix also mirrors PCM to SPDIF when in PCM mode).

## 5. Revert instructions

```
adb connect 192.168.0.183:5555            # adb root if needed
mount -o remount,rw /vendor
cat /vendor/lib/libmi3.so.orig > /vendor/lib/libmi3.so && sync
stop vendor.audio-hal; start vendor.audio-hal
cp /data/local/tmp/guisettings.xml.bak /sdcard/Android/data/org.xbmc.kodi/files/.kodi/userdata/guisettings.xml
```

## 6. Caveats / notes
- Speaker will be silent (or noise-muted by the DSP mute matrix) during pure bitstream playback — normal for bypass on optical.
- If the receiver shows PCM instead of DD/DTS: set the projector's own digital-output mode to BYPASS (TV settings), since AUTO may transcode/detect per content; that setting was not touched during this session.
- DTS-HD MA/TrueHD exceed SPDIF bandwidth; Kodi falls back to DTS core (`dtshdcorefallback=true`).
- `mi_getCodecType: MI_AUDIO_GetAttr failed` errors pre-existed and are unrelated.
