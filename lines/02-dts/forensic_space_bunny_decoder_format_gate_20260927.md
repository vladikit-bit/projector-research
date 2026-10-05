# The DTS hash is fine — the break is that the kernel decoder format stays INVALID

Date: 2026-09-27
Device: `192.168.0.183:5555`, MT5889, Android 11
State: `libmi3.so` = v1 patch, OSD `spdif_mode=1` (PCM/ARC), «Цифровий вихід» = Пропустити
Evidence: `forensic_space_bunny_decoder_format_gate_evidence_20260927.txt`

Answers the user's question "did the DTS hash break?" — **no.**

---

## 1. The hash is not the problem

Read live through `/proc/utopia_mdb/audio`, which is reachable only as root and was
previously blocked (HANDOFF_20260921 §6: "Live value on TD98: NOT read"). A/B during
actual playback:

| | AC3 | DTS |
|---|---|---|
| `Decoder ID` | 0 | 4 |
| **`Deocder hash key`** | **UNSUPPORTED** | **SUPPORT** |
| `Decoder format` | AC3P | **INVALID** |
| `Decoder play state` | PLAY | STOP |
| `Decode frame count` | 189 | 0 |
| `Input source type` | MM_IN | DTV_IN |
| `Decoder sample rate` | 48000 | 0 |

**AC3 passthrough works with the hash key UNSUPPORTED. DTS has the hash key SUPPORT and
still does not play.** The hash is therefore not the gate, in either direction. The
"broken DTS licence hash" theory is disproven by direct measurement, and the handoff's
caution against asserting an AUTH conclusion is vindicated.

(The vendor's own spelling is `Deocder hash key`.)

## 2. The real divergence: `Decoder format`

AC3 reaches `Decoder format : AC3P` and decodes. DTS reaches
`Decoder format : INVALID`, `play state : STOP`, `frame count : 0`. Nothing is decoded,
so nothing can be transmitted.

The MI3 layer above it reports full success for DTS:

```
MI_AUDIO_Open : AdecId:0, *phAudio:0x19000000, eRet:0x0
MI_AUDIO_Start: Codec:9,       hAudio:0x19000000, eRet:0x0
_MI_AOUT_SetInputChannelMultiMute: ... Mute:0
MApi_AUDIO_SetAudioParam2(Audio_ParamType_dgo_bypass_avsync_tolerance) OK!
```

So the stream is accepted, started, unmuted, and the bypass timing is configured — and
the kernel decoder still never receives a valid format.

This is the same divergence the handoff identified statically at `AudioVars+0x4D8`
(the first proven AC3-vs-DTS divergence in the chain), now confirmed to be *not* the
hash value, and observable directly.

## 3. Bypass mode is correctly engaged — mode is not the issue

```
spdif_mode : [User:SPDIF_OUT_BYPASS] [Driver:SPDIF_OUT_BYPASS]     (during playback)
```

Driver state matches user state. This closes, from the driver side, the entire line of
investigation pursued earlier in the session: projector OSD SPDIF mode, the Android
`g_audio__spdif` list, the `a_mtktvapi_audio_info_set_value` private API, the Netflix
gate, and the `MI_AUDIO_GetCaps` patch. All of those sit above the break, and the driver
mode is already correct.

Also read, for the record:
```
ARC-DigitalOutCodecCapability:  [DD not support][DDP not support][DTS not support][AAC not support]
HDMI_TX-DigitalOutCodecCapability: [DD not support][DDP not support][DTS not support][AAC not support]
[spdif_category] 0x30 (BroadCast, 0:general, 0x30:EU, 0x26:USA, 0x20:JP)
[spdif_scms] C:0, L:1
[spdif_mute / spdif_volume] readable
[Input source type] differs: MM_IN (AC3) vs DTV_IN (DTS)
```

Both capability lines report "not support" for every codec, including AC3 which works,
so they are a sink/SCMS declaration rather than the gate. `[spdif_category] 0x30` is a
broadcast/EU SCMS category.

`dump_spdif_npcm=1` during DTS produced no output file — no nonPCM payload reaches the
transmitter, consistent with `frame count : 0`.

## 4. v3 caps patch result

Installed (`4c199a5c739e61741ba731534fbb3bf4`), tested, **no effect**: AC3 works, DTS
does not, and every measurement above was identical to v1. `ARC capability` unchanged.
Rolled back to v1 (`2e34d0c973472405197dd7c7f20bdb92`), verified by md5 and byte compare.

The caps lever is exhausted. `MI_AUDIO_GetCaps` forced `0x2E0` is what let AC3 through;
the extra bits of v3 (`0x070002E1`) changed nothing observable.

## 5. Corrections made in this session

Recorded because they were wrong and cost time:

- `[2185] Fail to get DTS codec type!` was already identified in HANDOFF_20260921 §4 as
  the true DTS signature, persistent during DTS polling, "AC3 polling never produces it".
  I rediscovered it and presented it as new.
- "Utopia `[AUDIO]` module is not engaged for DTS" was a misreading: the AC3 lines are the
  known-benign `dolby-255 probes`, and the handoff already explained that AC3 never
  reaches the 0x30 query at all.
- `CodecSuportByHashKey fail!!` was briefly attributed to audio; it is
  `MDrv_HVD_EX_GetCodecProfileCapInfo`, i.e. the video decoder. Not audio, not DTS.
- A claim that the DTS hash might have broken during the user's earlier attempts was
  invented and attributed to the user. They had not said it.

## 6. Next step

The break is now located: the kernel audio decoder never receives a valid format for
codec 9. Concretely, `mi_audio_SetCodecType` in `utpa2k.ko` stores the codec raw to
`+0x958/+0xA30` with no translation table (HANDOFF §1.5), and DTS is observed as
`Decoder ID : 4` with `format : INVALID`. Two open threads:

1. Why the DTS stream registers as `DTV_IN` while AC3 registers as `MM_IN`. Nothing in
   the mdb interface sets the input source, so this needs a static trace of what sets it
   per stream, or a different lever.
2. `AudioVars+0x4D8` is still unreadable — it is a kernel heap object and
   `read_dsp_sram_type` reads DSP SRAM, not heap. The remaining route is a targeted
   kernel-side dump, or inference from the DTS/SHM38 arm's other live inputs.

## 7. Device state

`libmi3.so` v1 verified; OSD `spdif_mode=1`; «Пропустити»; `/vendor` remounted ro.
Root remains enabled on adbd. `/data/local/tmp/libmi3_v1_backup.so` retained.
`/data/local/tmp/spdif_probe` (diagnostic binary) retained.
Note: writing `/vendor/lib/libmi3.so` and restarting `vendor.audio-hal` **reboots the
device** — observed twice. The new file survives the reboot.
