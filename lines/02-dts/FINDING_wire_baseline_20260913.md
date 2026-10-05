# FINDING — WIRE BASELINE (2026-09-13 01:40-01:45), instrument-verified

## Instrument
`echo 'dump_spdif_npcm=1 path=1' > /proc/utopia_mdb/audio` (arm),
`=0` (disarm). Writes `/data/DUMP_audio_spdifNpcm_NN.bin`.
Requires `adb root` (file is root-only). dmesg marker:
`----- Start Dump Spdif Tx Npcm (mode:N) -----` / `Stop ...`.

## Measurement
A/B in ONE session, AC-3 then DTS, same instrument, same conditions.

| | AC-3 (`test_ac3_51.ac3`) | DTS (`test_dts_51.dts`) |
|---|---|---|
| dump size after 15-20 s | **5,148,672 B / 2,795,520 B** | **0 B** |
| IEC61937 preambles | **838** (100% data-type 0x01 = AC-3) | **0** |
| preamble byte stride | 0x1800 | n/a |
| preamble byte order | 16-bit **little-endian**: `72 f8 1f 4e` = Pa 0xF872, Pb 0x4E1F | n/a |
| `MI_AUDIO_Open AdecId` | 0 | 0 |
| `_MI_AOUT_NotifyConnectInput eDrvOut` | 4 | 4 |
| `Profile[pre,cur]` | [5,5] | [5,5] |
| `MI_AUDIO_Start Codec:` | **5** | **9** |
| `MI_PCM` reader opened | **YES** (`hPcm:0x81000000`) | **NO — zero MI_PCM lines** |
| player actually playing | yes | yes (speed 1, time advancing) |

## Conclusion
1. The instrument is proven sound: AC-3 yields a large, correctly-formed
   IEC61937 AC-3 burst stream.
2. **During DTS playback the SPDIF Tx engine emits literally NOTHING.**
   0 bytes over 12 s of confirmed playback.
3. The divergence is ABOVE the IEC61937 header layer: the `MI_PCM` output
   reader is never opened for DTS, while it is for AC-3. So the DEC blob is
   not producing an output frame/payload for DTS at all.
4. Therefore any fix confined to the DEC IEC61937 **header** builder
   (`0x45D07`) or its gate (`0x460C2`) cannot by itself make DTS appear on
   the wire — the failure is upstream of, or independent of, that builder.
5. `eDrvOut:4` is identical for both, so the HAL output-route selection is
   NOT the discriminator. The discriminator is the codec id
   (`Codec:5` AC-3 vs `Codec:9` DTS) and the consequent DEC behaviour.

## Reproduce
    ./cap.sh <tag> test_ac3_51.ac3 20      # -> MB of data, 838 preambles
    ./cap.sh <tag> test_dts_51.dts 12      # -> 0 bytes
    python scripts/analyze_wire.py <capture>
