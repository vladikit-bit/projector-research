# FINDING — Host-side Codec 5 (AC3) vs Codec 9 (DTS) differential trace

Date: 2026-09-16. Directive: find the FIRST host-side state/configuration
difference after MI_AUDIO_Start() and before the R2 decoder consumes ES;
minimal parameter proven by static data-flow + runtime correlation; no patching.

Evidence grade: every address below is byte-verified in the actual module
binaries (mik.ko / utpa2k.ko, ARM capstone + relocation resolution) or in
the runtime logs (K2_DTS_decr2.log / K2_AC3DTS_decr2.log, r26 CSV, r56 FINDING).

## A. The complete call chain (static, closed)

```
Kodi mi_decoder_open (libmi3 @0x3f408):
  format 0x9000000 (AC3) → codec 5          format 0xb000000 (DTS) → codec 9
→ MI_AUDIO_Start(handle,&codec)  = ioctl 0xC0081005 {handle, codec}
→ mik.ko MI_AUDIO_Start @0xa437c:
    1. mi_hwcaps_GetAudioCaps(&caps)
    2. instance+0x958 = codec;  MapDecoderType(codec) @0xa33e0:
         codec 5  → decSystem 3   (case @0xa3568: mov r5,#3)
         codec 9  → decSystem 0xB (case @0xa374c: mov r5,#0xB)
    3. _aeCurAudioDecoderType[slot] = codec
    4. Build 0x28-byte AU OpenParam @sp+0x60:
         +0x10 = decSystem   (the SetSystem2 switch key)
         +0x0C = instance+0xAE0 (from MI_AUDIO_Open attrs — source type)
    5. MApi_AUDIO_SetDecodeSystem(path,&param)
→ utpa2k.ko: MDrv_AUDIO_SetDecodeSystem → pFuncPtr_Setsystem
→ HAL_AUDIO_SetDecodeSystem → HAL_AUDIO_SetSystem2(decId, decSystem) @0x44d0b0
     decSystem 3  → case @0x44d228: AbsWriteByte(0x112E98, 0x81)   engine 0x81 (DDP)
                   + AudioVars+0x524/0x528 latch = 3
     decSystem 0xB → case @0x44d1dc: AbsWriteByte(0x112E98, 4
                   or 0x97 if AudioVars+0x4d8==3)                    engine 4 (m6-DTS)
                   + latch = 0xA
→ then HAL_AUDIO_SPDIF_SetMode(AudioVars+0xc, AudioVars+0x1cc) @0x448a10:
     mode table (idx = AudioVars+0xc): 0/1→PcmMode, 2→AutoMode, 3→BypassMode, 4→Transcode
→ back in userspace, mi_decoder_open tunnel SetAttrs → mik mi_audio_SetAttr:
     attr 0x204 (tunnel byte) → MApi_AUDIO_SetAudioParam2(path,0x75,val)
     attr 0x205 (value 5)     → MApi_AUDIO_SetAudioParam2(path,0x76,5)
     attr 0x607               → (0x300-range handler)
→ utpa2k HAL_MAD_SetAudioParam2 @0x45c548 (134-entry table on param):
     0x75 → sub 0xA3; 0x76 → sub 0xA4 → HAL_DEC_R2_Set_SHM_PARAM(sub, adecId, val, 0)
     ⇒ writes into the DEC R2 SHM control block
→ ES feed (AOUT tunnel): host writes into DEC ES buffer — CONFIRMED ALIVE for
   DTS under K-2 (levels fluctuate) — the feed is NOT the blocker.
```

## B. The divergence point — SPDIF non-PCM output config rows

All three mode functions read the LATCH (code type) via
HAL_AUDIO_Get_CodeTypeByDecodeID(decId) (reads AudioVars+0x524/0x528 —
the value SetSystem2 stored: **3 for AC3, 0xA for DTS**) and dispatch into a
94-entry per-codec config-row table (indexed codeType−2).

### HAL_AUDIO_SPDIF_BypassMode @0x444f40 (SPDIF mode 3)
- AC3 row @0x445348: gate = **AudioVars+0x3A5 == 1** (AC3-detected status).
  PASS (DDP engine sets it) → writes AudioVars+0x14=0, **+0x570=1** (non-PCM
  enable) → ApplySetting pushes config to DSP → SDO live.
- **DTS row @0x445244: gate = AudioVars+0x399 == 2 (`cmp r7,#2; bne 0x445330`
  @0x445248). On fail → 0x445330: AudioVars+0x4FE=1 (+0x583=1) = force-PCM
  bail → non-PCM output config NEVER applied.**
- +0x385…+0x3B0 (incl. +0x399, +0x3A5) are the AUDIO_INFO detection block.
  No ARM writer exists in utpa2k (verified: only 1 bogus hit) ⇒ the fields are
  written by the DSP into the shared AudioVars (DSP DM base 0x1C010000 — same
  frame as the licensee struct at 0x1C012734=+0x2734, proven earlier).

### HAL_AUDIO_SPDIF_AutoMode @0x444914 (SPDIF mode 2)
- DTS row @0x444c44: gate = **HAL_DEC_R2_Get_SHM_INFO(0x3D, decId) == 3**
  (`cmp r0,#3; bne 0x444cd4` @0x444c50) — a DSP-side status; else consults
  HAL_MAD_GetDTSInfo(4) → g_u32bDTSCD.
- AC3/MPEG shared row @0x444b5c: gates on +0x3AD==1 / +0x3B0 bit0 / +0x2502==1.
- Same pattern: detection-gated.

### HAL_AUDIO_SetDigitalOut @0x4435e8
- Protected-codec mask test `tst 0x810, 1<<latch` @0x4439b4 → for latch 4/11
  calls **MDrv_AUTH_IPCheck(0xE)/(0xD)** (IP-auth!). DTS latch 0xA is in
  neither 0x810 nor 0x100C → all-zero data-path params.

## C. The minimal parameter (answer to the directive)

**The code-type latch (AudioVars+0x524/+0x528 = 3 vs 0xA) selects the non-PCM
output config row, and the DTS row is gated on a DTS-detection status byte
(AudioVars+0x399==2 in BypassMode / SHM_INFO 0x3D==3 in AutoMode) that this
device never produces, because no DTS decoder ever decodes: the m6 engine is
license-dead (licensee=0 from efuse HASH), and the DDP 0x81 engine cannot
sync DTS frames. When the gate fails, the row bails (+0x4FE=1 force-PCM) and
the SDO non-PCM config is never applied → wire silence. The ES feed itself is
alive.**

Every non-PCM output row in this firmware is detection-gated the same way;
AC3 passes because the DDP engine (licensed) really decodes AC3 and the DSP
fills the AC3 detection field.

## D. Runtime correlation (three independent eras, all consistent)

1. **r26 CSV (stock-era control-block diff)**: cells 0x0F46–0x0F4C —
   "DTS output config never set" (DTS == IDLE), while AC3 changes them ⇒
   the DTS output-config path does not run. (Same stock functions today;
   K-2 did not touch SetDigitalOut/BypassMode/AutoMode.)
2. **r56 FINDING**: with the DTS decoder RUNNING (format: DTS, PLAY, 12,569
   frames) the SPDIF TX still got **0 bytes** ⇒ decode working is not
   sufficient; the output-config gate is the blocker (the detection byte
   required by the row ≠ what the decoder writes).
3. **K-2 logs (this surface, current)**: DTS session — engine type[81],
   cmd<4>, "dec play !!", 158 s of live ES levels (0x1F7→0x3A5) and live STC
   (236181→332207), yet play=0, state=0,0(0), frmCnt=0000, PCM=0000 the whole
   window (2,468 telemetry lines). AC3 started later in the SAME boot with the
   SAME engine state (no re-init, no decType change, playCmd 0x0→0x84 only):
   play=1, state=3,0(9) within ~500k ticks, PCM live, SDO bursts.
   ⇒ engine + feed + STC are all fine; only the per-codec output config row
   differs.
4. **N3 (autopsy)**: full-AC3-configured session fed DTS bytes produced valid
   DTS bursts with **Pc=0x0B** — proving (a) the SDO/IEC framer
   self-classifies payload by syncword (0x7FFE8001→DTS1) and does NOT need a
   DTS decoder, and (b) the only thing DTS lacks is the applied non-PCM
   output config.

## E. Patch candidates (NOT built — awaiting approval)

**UPDATE 2026-09-16: P-BYP BUILT AND BYTE-VERIFIED (NOT INSTALLED).**

Active SPDIF mode determined at runtime (read-only, root adb):
```
/proc/utopia_mdb/audio → dmesg:
spdif_mode : [User:SPDIF_OUT_BYPASS] [Driver:SPDIF_OUT_PCM]
ARC-DigitalOutCodecCapability:[DD not support][DDP not support][DTS not support][AAC not support]
```
User mode = **BYPASS (3)** ⇒ HAL_AUDIO_SPDIF_BypassMode is the active path ⇒
**P-BYP is the correct primary candidate** (P-AUTO not needed as primary).
(The ARC capability line is the HDMI/ARC gate — untouched per directive.)

### P-BYP build record (artifact: r57_npcm/utpa2k_K2_PBYP_bypass_gate.bin)

| Item | Value |
|---|---|
| Source file | `r57_npcm/utpa2k_K2_dts81.bin` (= CURRENT DEPLOYED K-2; device md5 re-verified live: `5d2f265226c8707bc2d562124a03ca1c`) |
| Output file | `r57_npcm/utpa2k_K2_PBYP_bypass_gate.bin` |
| Output md5 | `ef18748f0f85d4ce05078f0070d76bfe` |
| Function | `HAL_AUDIO_SPDIF_BypassMode` (symbol st_value 0x444F40, size 0x5C4) |
| Site VA | **0x44524C** (`cmp r7,#2` gate at 0x445248; fail branch here) |
| File offset | **0x4528A0** (.text sh_offset 0xD654 + VA, sh_addr 0) |
| Old word | `0x1A000037` = `bne 0x445330` (bytes LE `37 00 00 1A`) |
| New word | `0xE1A00000` = `nop` (mov r0,r0) (bytes LE `00 00 A0 E1`) |
| Changed bytes | **3** (0x4528A0: 37→00, 0x4528A2: 00→A0, 0x4528A3: 1A→E1; byte 0x4528A1 is 0x00 in both words — coincidentally identical) |
| Effect | DTS config row @0x445244 always falls into the success path (0x445250: AudioVars+0x14=0, +0x18=0 → common tail → ApplySetting) instead of bailing to 0x445330 (+0x4FE=1 force-PCM) |

Diff proof (whole-file, source vs output): exactly 3 differing bytes, all within
the 4-byte site. File size unchanged (25,381,336 B). ELF header untouched
(first 64 B identical). ELF re-parse OK (46 sections; .text size 0x6287DC;
symbol table resolves). **K-2's own changes preserved byte-exact** (DTS-case
site @0x44D1DC region identical source↔output). AC3 row @0x445348 untouched.
No latch / MapDecoderType / EDID / Kodi / HDMI-ARC / other AUTH changes.

Before/after disassembly (0x445234–0x445264):
```
BEFORE:  00445248: cmp  r7,#2      ; AudioVars+0x399 (DTS detection) == 2 ?
         0044524c: bne  0x445330   <<< PATCH SITE  (fail → +0x4FE=1 force-PCM bail)
         00445250: mov  r1,#0
         00445254: str  r1,[r0,#0x14]   ; DTS success path
         00445258: str  r1,[r0,#0x18]
AFTER:   00445248: cmp  r7,#2      ; (gate still evaluated, result ignored)
         0044524c: nop             <<< PATCH SITE
         00445250..0x00445264: byte-identical to BEFORE
```

All other candidates (latch 0xA→3, MapDecoderType 9→3, P-AUTO) remain NOT built.

STATUS: **P-BYP TESTED 2026-09-16 — NO DTS WIRE; MISS EXPLAINED; FROZEN (no further patch).**

### Test results (2026-09-16, P-BYP deployed, md5 ef18748f verified pre/post reboot)

1. **AC3 regression ×2: PASS.** Capture 1: 4,595,712 B, 748/748 preambles, 100%
   Pc=0x01, stride 0x1800. Capture 2: 2,131,968 B, 347/347, 100% Pc=0x01.
   P-BYP is harmless to the AC3 path.
2. **Clean DTS test (MI_AUDIO_Start Codec:9, correct DTS pipeline): wire = 0 bytes.**
   NOT a success per the agreed criteria (no valid DTS IEC61937 bursts).
3. **Miss root cause (runtime-proven, 3 probes with debug_level=6 temporarily
   enabled then restored to 0):**
   - `HAL_AUDIO_SPDIF_SetMode: u8Spdif_mode=3, eAudioSource=3` during DTS
     playback (repeats ~10 Hz) vs **`eAudioSource=4` during AC3**.
     eAudioSource = AudioVars+0x1CC, set from mik instance+0xAE0.
   - `spdif_mode : [User:SPDIF_OUT_BYPASS] [Driver:SPDIF_OUT_PCM]` during DTS
     playback vs **[Driver:SPDIF_OUT_BYPASS]** during AC3.
   - Static routing in HAL_AUDIO_SPDIF_BypassMode: the mask tests at
     0x44500C/0x44501C dispatch **+0x1CC==3 via `tst 0x4A` bit3 DIRECTLY to
     0x445250** — which writes AudioVars+0x14=0 and **+0x18=0 (SPDIF_OUT_PCM)**
     and skips the 94-entry codeType table entirely. **The patched DTS-row gate
     at 0x44524C is never reached for source 3.** Source 4 goes via `tst 0x91`
     bit4 → the codeType table → AC3 row → +0x570=1 (non-PCM enable) — the
     working AC3 path.
   - Therefore the earlier hypothesis ("the DTS codeType row's +0x399==2 gate
     blocks the config") is DISPROVEN as the operative blocker for the MM path:
     the DTS row is dead code for eAudioSource=3.
4. **Revised minimal parameter:** **AudioVars+0x1CC (eAudioSource): 3 for DTS
   vs 4 for AC3** (origin: mik instance+0xAE0, writers MI_AUDIO_Open@0xa2ac4
   and _MI_AUDIO_NotifyConnectInput@0xa0b54 with the r4-derivation chain
   0xa0880–0xa0abc by input-handle magic 0x68/0x16/0xab). This value selects
   PCM-forced output (3 → 0x445250 direct) vs full non-PCM config table (4).
5. **Next lever (NOT built, NOT approved):** trace the NotifyConnectInput
   r4-chain to find where DTS gets source=3 (vs AC3 source=4); candidate
   surfaces thereafter: the value at its origin (mik) or the 0x4A/0x91 routing
   masks in BypassMode (utpa2k). P-BYP site 0x44524C remains inert.
6. Device state after test: P-BYP deployed (harmless; AC3 PASS), debug_level=0
   restored, dump files cleaned, players stopped. Rollback baseline:
   `utpa2k_K2_dts81.bin` md5 `5d2f265226c8707bc2d562124a03ca1c`
   (= r57_npcm/DEVICE_prePBYP_utpa2k_k2.bin, pulled from device pre-install).

Original candidate table (superseded by the build above):

| ID | Site | Change | Effect |
|---|---|---|---|
| **P-BYP** | HAL_AUDIO_SPDIF_BypassMode @0x44524C: `bne 0x445330` (gate fail branch) | 4-byte BNE → NOP (0xE1A00000) | DTS row always takes the success path (0x445250) → non-PCM output config applied without detection |
| **P-AUTO** | HAL_AUDIO_SPDIF_AutoMode @0x444C54: `bne 0x444cd4` | 4-byte BNE → NOP | same, for Auto mode |
| (rejected as primary) latch 0xA→3 | SetSystem2 DTS-case latch stores (`mov r1,#0xA` @0x44D218/0x44D27C) | imm 0xA→3 | routes DTS to AC3 rows — but the AC3 row gate (+0x3A5==1) then fails for pure-DTS sessions (no AC3 detection) — this only recreates N3, which requires AC3 priming |
| (rejected as primary) MapDecoderType 9→3 | mik.ko @0xA374C `mov r5,#0xB` | 1 byte | same problem as above |

P-BYP is the recommended candidate: it is the exact negation of the proven
minimal parameter, touches only the DTS row (AC3 row at 0x445348 untouched),
and composes with the deployed K-2 (engine 0x81 for DTS). If the device's SPDIF
mode is Auto rather than Bypass, P-AUTO is the analogous site (the active mode
must be confirmed at test time — check dmesg/SetMode prints or try P-BYP first,
then both).

Verification plan (when approved): deploy → AC3 regression (must stay PASS:
burst count/stride/Pc=0x01) → DTS test (wire capture: expect IEC61937 DTS
bursts Pa=F872 Pb=4E1F Pc=0x000B, payload=real DTS frames, AVR lock) → rollback
path = restore K-2 ko from BACKUP_20260914_0717.

## F. Answers to the standing questions

- **Why AC3 feeds ES and DTS does not** — framing correction: DTS *does* feed
  ES (proven under K-2). The real difference: nothing *consumes* DTS and the
  *non-PCM output config never runs* for DTS (detection-gated), while AC3
  decodes (licensed DDP), gets detected, and its config row applies.
- **Maven/Kodi audio packer**: not needed. The DSP-side IEC framer
  self-classifies payload syncwords (N3: DTS frames → Pc=0x0B bursts emitted
  under AC3 config). The blocker is purely the host-side config gate, not any
  Kodi-side packing.
- AudioVars+0x4D8 (DTS 0x97 special-case in SetSystem2): irrelevant under K-2
  (the case was rewritten); stock meaning still unidentified — parked.
