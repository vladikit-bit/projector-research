# SPACE BUNNY — DIRECT DTS OFFLOAD PROBE CORRECTION

**Date:** 2026-09-24  
**Device:** TD98 Pro / C50A, `192.168.0.183:5555`  
**Mode:** direct Android AudioTrack offload test; no firmware/module patch, no flash, no reboot  
**Patch restriction:** honored

## Why this correction exists

The earlier `Probe7` result was not a valid direct-stream verdict. Its frame-size calculation added an erroneous `+4`, and the next fixed version wrote zero padding between raw frames. The HAL saw an offload track, but the DTS/AC3 parsers saw invalid frame boundaries.

This report records the corrected exact-frame probe.

## 1. Probe correction

The local raw test files are:

```text
/data/local/tmp/test_dts_51.dts  7,545,000 bytes
/data/local/tmp/test_ac3_51.ac3  1,920,000 bytes
```

The corrected helper writes exactly one raw frame per `AudioTrack.write()`:

```text
DTS frame = 2012 bytes
AC3 frame = 1536 bytes
```

It does not insert IEC61937 headers, does not add zero padding between frames, and does not change the device volume.

The original tools were not modified. A separate temporary `Probe7Exact` helper was built and staged as `/data/local/tmp/probe7exact.jar`.

## 2. Corrected direct DTS result

Command shape:

```text
CLASSPATH=/data/local/tmp/probe7exact.jar app_process /system/bin \
  --nice-name=probe7exact Probe7Exact \
  /data/local/tmp/test_dts_51.dts dtsraw dts 1
```

Observed output:

```text
P7EXACT ... frame=2012
P7EXACT direct=true
P7EXACT track=1
P7EXACT written=7545000
P7EXACT done written=7545000
```

While active, `dumpsys media.audio_flinger` showed:

```text
Output thread ... type 4 (OFFLOAD)
HAL format: 0x0B000000 (AUDIO_FORMAT_DTS)
Processing format: 0x0B000000
flags: 0x11 (DIRECT | COMPRESS_OFFLOAD)
```

The corrected run produced no `dts_parser` bad-header or syncword errors. The earlier `LINE:1531` errors belonged to the malformed test carrier and are discarded.

## 3. Corrected AC3 control

The same helper was run with:

```text
/data/local/tmp/test_ac3_51.ac3 ac3raw ac3 1
```

Observed:

```text
P7EXACT ... frame=1536
P7EXACT direct=true
P7EXACT track=1
P7EXACT written=1920000
P7EXACT done written=1920000
```

AudioFlinger showed:

```text
type 4 (OFFLOAD)
HAL format: 0x09000000 (AUDIO_FORMAT_AC3)
flags: 0x11 (DIRECT | COMPRESS_OFFLOAD)
```

No `ac3_parser` sync errors were observed in the corrected run.

## 4. What this proves

The corrected probe proves that the device accepts valid raw DTS and AC3 elementary streams through Android's direct/compress-offload path:

```text
valid raw DTS -> ENCODING_DTS -> HAL format 0x0B000000
valid raw AC3 -> ENCODING_AC3 -> HAL format 0x09000000
```

This is stronger than the earlier Kodi/speaker test, but it still does **not** prove receiver lock or physical IEC61937 delivery.

## 5. Physical route limitation

During the corrected direct test:

```text
AudioFlinger output device: speaker
audio_hw_primary: speaker un-mute
audio_hw_primary: hdmi-arc mute
```

The Android policy declares HDMI Out, HDMI ARC, and Spdif, but the connected output state remains speaker. The exact-frame test therefore validates the bitstream/offload path, not the final SPDIF/ARC hardware route.

The remaining question is now sharply defined:

```text
How do we make the valid DTS offload stream reach the physical SPDIF/ARC output
and enter the correct non-PCM channel-status mode?
```

The known `MApi_AUDIO_SPDIF_SetMode` path exists in native code, but a standalone helper calling `libutopia.so` segfaulted because the vendor audio process/global state is not initialized in that process. That failed helper is not evidence and will not be retried.

The previously observed Android/Kodi `spdif_type`/`spdif_mode` properties and `AudioSystem.setParameters` path were already shown to be inert in the archived runtime work. No safe unpatched setting lever has yet been found that forces the physical digital route.

## 6. Final state and preservation

After the direct tests:

```text
active players = none
media volume = 83
Kodi passthrough = true
Kodi passthrough device = AUDIOTRACK:AudioTrack (RAW)|Android IEC packer
```

Temporary ADB forwarding was removed. No firmware, kernel module, vendor library, or persistent system audio property was patched. The test jars are isolated under `/data/local/tmp` and do not alter firmware.

## Corrected conclusion

The DTS stream itself is valid and reaches the hardware offload format. The unresolved blocker is no longer raw DTS parsing or AudioTrack acceptance. It is the **physical digital-output route/non-PCM mode handoff** after the offload stream reaches the MStar audio service.
