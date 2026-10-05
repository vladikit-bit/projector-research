# FINDING — 2026-09-30 (late): the HOST side is exonerated for the third time, now by three live measurements — and the DSP pump struct is NOT in the readable window

**Runtime session on a rebooted projector. Companions: `FINDING_host_tx_chain_20260930.md`,
`FINDING_latch_verification_20260930.md`.**

## 1. The host arms the output correctly for DTS (correcting my own same-session inference)

Fresh boot, **DTS played first**, `debug_level=4`:

```
HAL_AUDIO_SPDIF_BypassMode: AU_DVB_STANDARD_DTS(dec_id: 0)/E_AUDIO_INFO_MM_IN/
   (SPDIF_OUT_BYPASS/SPDIF_OUT_BYPASS), DDPE[0] MS11_DDE[0] DDP_Bypass[0] AAC_Bypass[0]
   MAT_Bypass[0] ATMOS Stream[0], AVR support DDP[0] DD[0] DTS[0] AAC[0], r2cmd[84]
```

versus AC-3 (identical except `DDP_Bypass[1]` — a descriptive flag of the content):

```
HAL_AUDIO_SPDIF_BypassMode: AU_DVB_STANDARD_AC3(dec_id: 0)/… /(SPDIF_OUT_BYPASS/SPDIF_OUT_BYPASS), … DDP_Bypass[1] … r2cmd[84]
```

`HAL_AUDIO_SPDIF_SetMode`'s jump table (`0x448B3C cmp r4,#4; ldr pc,[r0,r4,lsl#2]`):
mode 0 → `PcmMode`, 1 → `AutoMode`, **2 → `BypassMode`**, 3 → `TranscodeMode`,
then `HAL_AUDIO_eARC_SetMode` + tail `HAL_AUDIO_SPDIF_ApplySetting`.

**CORRECTION:** earlier this session I concluded "BypassMode is never called for DTS" from the
absence of its log line. That was a **log-absence artifact** — the mode was already armed by
the preceding AC-3 session, so no change and no log. On a fresh boot the DTS call happens and
arms BYPASS/BYPASS. `spdif_mode` readback agrees: `Driver:SPDIF_OUT_BYPASS` under both codecs
while playing, `SPDIF_OUT_PCM` only when idle.

## 2. The messages sent to the SND DSP are identical

From `ApplySetting` (`0x446900`, `0x446914`): `HAL_SND_R2_Set_SHM_PARAM(110/0x6E, …)` and
`(157/0x9D, …)`. Live, both codecs, every occurrence:

```
shmType:0x6e, id:0x0, param: [0x0, 0x0]      (18× under DTS, 43× under AC-3)
shmType:0x9d, id:0x0, param: [0x0, 0x0]
```

`[vars+0x4F4]==5` and `[vars+0x3A8]&1` are false for both. **No codec signal reaches the SND
core through this path.**

## 3. The host-visible audio registers are byte-identical

`reg_bank=0x112A`, `0x112B`, `0x1120` captured during DTS and AC-3 playback: **0 differing
words in all three banks.**

## 4. The passthrough-permit deny is real but not on our path

`MDrv_AUDIO_Get_PassthroughPermit` (called once, from `MDrv_AUDIO_SYSTEM_Control` @`0x412614`,
right after `MDrv_AUDIO_SPDIF_SetMode` and before `cmp r4,#0` @`0x412658`) compares command
names extracted from `.rodata`:

```
GetPassthroughPermitAac    -> 1        GetPassthroughPermitAc3   -> 1
GetPassthroughPermitAc4    -> 1        GetPassthroughPermitDts   -> 0   ← the deny
GetPassthroughPermitMpegH  -> 1        (everything else         -> 0)
```

It is a genuine source-level build decision naming DTS as unsupported. **But:** the strings
exist **only in the kernel module** — `libapiAUDIO.so`, `audio.primary.mt5889.so`,
`libutopia.so`, `libmi3.so` contain zero occurrences — and at runtime the function's
`unsupported cmd '%s'` log never fires in our path. **Not the active gate.**

## 5. The DSP pump struct is NOT in the mdb DM window (kills the write-flip plan)

Small reliable reads (`len≤0x50`) under both codecs:

| cell | meaning | DTS | AC-3 |
|---|---|---|---|
| 0x07C4 / 0x07D4 | B-latch `[+0x0]` / `[+0x10]` | 0 | 0 |
| 0x0290 | sub-object pointer | 0 | 0 |
| 0x4EE4 | pump progress latch | 0 | 0 |
| 0xA108 / 0xA124 | latch-pointer field / paradox `−0x5EDC` | 0 | 0 |
| 0x100C, 0x1038, 0x104C | the live 0x1000 block | `0x010BF0`/`0x1FAE`/`0x400` | `0x010BF0`/`0x1FAE`/`0x400` |

The pump's base `r11` is **not 0**, so the struct offsets are not cell indices. The
`write_dsp_sram_type` flip plan cannot target the latch/stall words. Only the 0x1000+ block is
live, and its codec differences (`0x1000–0x100B`, `0x100E/0x100F`) are activity counters.

**Capture-method trap (new):** a `len=0x1000` DM read (4096 words) overflows the dmesg ring —
the known-good cell `0x104C` read back as `0` in the full dump and `0x400` in a 0x50 read. Any
DM capture must use small reads (≤0x50) and be treated as untrustworthy if a control cell
comes back zero.

## 6. Where this leaves the block

Host side: **exonerated** (mode armed, SHM params equal, registers equal). The DSP side is the
only remaining live territory — Muse's thread: the pump's return-0 arms, the writer-unknown
`−0x5EDC` word, and the `{1,2}` type dispatch in `F_3D952`. Since the host cannot write those
words and they are not observable live, the only remaining handle is a **DSP-image patch** at
one of those sites (or the V083 reference-image diff), with the Pioneer as the oracle.
