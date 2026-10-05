# AUTONOMOUS MISSION LOG — P-BYP2..P-BYP5 (2026-09-16 night, GLM53)

Goal: real DTS passthrough (IEC61937 Pc=0x0B bursts + AVR lock). Method per the
user's autonomous-mission directive: find→prove→patch→test loop, minimal
reversible patches, AC3 regression after each step.

## Deployed state at session end (VERIFIED on device)

| Module | md5 | Content |
|---|---|---|
| mik.ko | `d49a3790e1d7a2a673032e625a728259` | mik-R5 base (pulled as DEVICE_mik_current.bin, md5 03fc2c0d) + **P-BYP5** (2 bytes: `bne .+8` @VA 0x977d4/file 0x9A2C8 → NOP) |
| utpa2k.ko | `2e251c98de213e08dc38c92f6c5009c9` | K-2 base + **P-BYP2** (row @0x44524C-58 → AC3-identical field writes) + **P-BYP3** (DTS-case @0x44D1DC restored to STOCK m6 engine) + **P-BYP4** (1 byte: SetDigitalOut mask 0x100C→0x140C @0x4437D4) |

All AC3 regressions PASS throughout (680/680, 447/447, 352/352, 352/352 + live
1.1MB dump). Artifacts: r57_npcm/utpa2k_K2_PBYP2_dts_ac3row.bin (a2efbf48),
utpa2k_PBYP3_m6_stock_dts_row.bin (ae775794), utpa2k_PBYP4_dts_attach_mask.bin
(2e251c98), mik_PBYP5_monitor_always.bin (d49a3790).
Rollbacks: utpa2k → utpa2k_K2_dts81.bin (5d2f2652) or DEVICE_prePBYP_utpa2k_k2.bin;
mik → DEVICE_mik_current.bin (03fc2c0d).

## Enum semantics decoded (from firmware's own string tables — sAudioSourceNameToEnumTable etc.)

- eAudioSource: 0:DTV 1:ATV 2:HDMI **3:ADC 4:MM**. The earlier "3 vs 4" was
  REAL but UNSTABLE across sessions (stale carry-over) — not the operative gate.
- CodecType latch: **0xA = DTS (the latch was CORRECT all along)**; 0xB =
  AU_DVB_STANDARD_MS10_DDT (Dolby!); 0x16 = DTSHD_ADO. The P-BYP-era idea that
  the row/table index was wrong is dead — the latch is right.
- SPDIF_OUT enum: 0=PCM, 2=NONPCM/AUTO, 3=BYPASS, 4=TRANSCODE, 5=NONE, 6=MULTIPCM.

## Step results

1. **P-BYP2** (DTS row writes AC3-success fields: strb 1→+0x570, +0x14=0, +0x18
   stays 3): AC3 PASS; DTS wire 0. Taught: rows aren't reached OR something
   later forces PCM.
2. **P-BYP3** (revert K-2 engine byte to STOCK: DTS → m6 engine 4):
   **DTS DECODE RESTORED — Decoder format:DTS, play state:PLAY, frame count
   717+ and growing** (first time since r56 era; the "hashkey UNSUPPORTED"
   verdict is cosmetic — decode runs anyway). AC3 PASS.
3. **Blocker isolated**: `audio_status` shows the SPDIF output PORT:
   AC3=`spdif[8bda00,b8ff00]` (buffers live) vs DTS=`spdif[000000,000000]`
   (port never attached) — while DecStatus=1, DecMute=0, source=4, everything
   else identical. The decode→PCM flows to DAC; the SPDIF port is never opened.
4. **P-BYP4** (1 byte: latch-mask 0x100C→0x140C so DTS joins the AC3-family
   SetDigitalOut path r4=1 attach): AC3 PASS; DTS port still dead —
   SetDigitalOut is NOT invoked per-stream.
5. **Delivery mystery resolved**: on a FRESH boot, DTS uses the SAME MM2-AES
   path as AC3 (start window: CheckAesInfo×24, InputAesFinished×12,
   SetCommInfo(1)×6). Earlier "PcmWrite-only" DTS observations were stale-session
   artifacts. mik sequences byte-identical except Codec:5/9 and ES BufSize
   1536/2012 (both correct).
6. **ROOT GATE FOUND**: the whole utpa2k SPDIF-config chain
   (SPDIF_Monitor ioctl → _MApi_Audio_SPDIF_Monitor → SetMode → BypassMode →
   ApplySetting → SND pushes → port attach) NEVER RUNS for DTS:
   - mik `_MI_AOUT_MonitorTask` @0x971a8 calls `MApi_Audio_SPDIF_Monitor`
     **only if `_bAllInsReleased==1`** (armed by MI_AOUT_Open/Close events);
   - utpa2k side gates on AudioVars+0x2098==1 ("DSP loaded" — set at boot by
     HAL_AUDSP_DspLoadCodeSegment — OK since boot).
   - Runtime proof: AC3 windows show the monitor ioctl + eAudioSource prints;
     DTS windows show ZERO (fresh boot, DTS first).
7. **P-BYP5** (mik 2 bytes: the bne→NOP, monitor call unconditional): AC3
   monitor still runs + wire live (no regression); **DTS STILL ZERO monitor
   calls** ⇒ the task's flow to the call region has one more condition, or the
   task loop doesn't iterate for DTS.

## The remaining open point (next session target — crisp)

In mik `_MI_AOUT_MonitorTask` (VA 0x971a8, size 0x14DC):
- Entry gates: `_stAoutMonitorTask+0xC==0`, `_bIsAoutInit!=0` — both pass.
- Body: event wait (`_s32AoutMonitorTaskEvent`, ~100ms), then the
  `_MI_AOUT_CheckInputChannelMuteStatus` loop (0x97618-0x976B0), the /100
  timer check at 0x97720, then 0x97770→mutex→(NOP'd flag)→SPDIF_Monitor call.
- With P-BYP5 ALL paths from 0x97720 reach the call — so the divergence is in
  HOW the per-iteration flow arrives there (or whether it iterates at all).
- **Hot lead**: the AC3 start windows uniquely contain
  `_MI_AOUT_NotifyDisconnectInput:9178]hAout:0x17000000,hInput:0x81000000`
  (a PCM disconnect event!) + `MI_PCM_Close` — DTS windows don't. The event
  `_s32AoutMonitorTaskEvent` is likely set by AOUT connect/disconnect churn,
  which the AC3 path generates and the DTS path does not. Trace
  `_s32AoutMonitorTaskEvent` setters (search relocs in mik) and the wait-loop
  semantics; candidate fix: force-set the event at DTS play, or make the task
  loop unconditionally (like the flag NOP already done).
- Debug knob attempt: `/sys/module/mik/parameters/mi_aout_amp_debuglevel`
  (default 32) feeds the AMP level, not `_u32AoutDbgLevel` (task prints) —
  no runtime knob found for the task prints.

## Causal map as of now (DTS, engine m6, all patches active)

```
Kodi .dts (Codec:9, MM2-AES delivery OK, ES chunks notify OK)
→ mik MapDecoderType 9→decSys 0xB → SetSystem2: engine byte 4 (m6 STOCK) + latch 0xA
→ DEC m6: DECODES DTS (format=DTS, PLAY, frames grow; hashkey verdict cosmetic)
→ PCM flows to DAC/speakers; SPDIF port NEVER attached:
   mik AOUT MonitorTask never calls MApi_Audio_SPDIF_Monitor for DTS
   (event-armed only by AOUT open/close churn absent from the DTS path;
    P-BYP5 flag NOP alone insufficient)
⇒ no SetMode/BypassMode/ApplySetting ⇒ no SND pushes ⇒ spdif[000000] ⇒ wire 0.
```

Success criteria NOT yet met (no valid DTS bursts). AC3 unaffected throughout.

## ADDENDUM — P-BYP6 result (07:30-07:50)

**P-BYP6** (utpa2k @0x3e77fc `popeq {r4,r5,fp,pc}` → NOP, 4 bytes; utpa2k md5
`f851355378cb67c596fe754c5e4a9d36`): the +0x2098 DSP-segment gate in
`_MApi_Audio_SPDIF_Monitor` bypassed — the monitor's real work now runs for DTS.

Verified after reboot:
- **DTS playback now produces the monitor chain**: eAudioSource prints appear
  (u8Spdif_mode=3, eAudioSource=4) and the SND pushes fire (shmType 6d/6e/9d)
  — IDENTICAL cadence to AC3 (~2-3 per 5s window for both). The mik+utpa2k
  monitor chain is now codec-symmetric.
- AC3 regression PASS (348/348, Pc=0x01, 2.1MB wire).
- BUT: DTS wire still 0 bytes; spdif[] port cells still 000000; [Driver:PCM].

## The remaining gap — precisely bounded

The SND SHM pushes (6d/6e/9d, all zeros) are NOT the differential (identical
for both codecs — they're the monitor's entry writes at ApplySetting
0x4468f0-0x446914). The real AC3-only artifacts that remain:
1. spdif[8bda00,b8ff00] — the SND SPDIF port read/write pointers advance
   (data flowing to the TX) only for AC3.
2. The **ApplySetting state machine** at AudioVars+0x25FC:
   - kicked at 0x446a90 (`str r6,[r1,#0xfb]` from r1=AV+0x2501 → state = 1 or
     2 depending on AV+0x2603) when the comparator (0x44692c-0x446a74, current
     vs shadow +0x14/+0x18/+0x570-584/+0x4FD-4FE) detects a config change;
   - states 1-7 (table @0x446f08): 1=mute SPDIF+ARC+state2 0x446f28;
     2=49ms-wait 0x446fc4; 3=reg 0x112C5F path 0x44701c; 4=DisOutSrm timer
     0x447064; **5=HAL_AUDIO_SPDIF_Tx_SetNonPCM(AV+0x2600 byte) 0x4470cc**;
     6=eARC 0x447150; 7=HDMI 0x4471a4. One state per ApplySetting call.
   - The SPDIF TX non-PCM channel-status enable (state 5) and the port data
     flow require the machine to WALK — multiple monitor ticks.
3. The mik AOUT MonitorTask under P-BYP5 calls SPDIF_Monitor on every body
   iteration, but the body runs only ~2×/5s (the WaitEvent has a long/infinite
   timeout [sp+0x4c]=-1, and/or the PCM churn gates the iteration rate).

## Next levers (ranked)

L1. Verify the state walk: instrument/patch so the DTS play-start apply block
    kicks the machine (it should — my row changes +0x570/+0x14 → comparator
    fires) AND the monitor ticks enough times to walk states 1→5. If the tick
    count is the problem, arm `_s32AoutMonitorTaskEvent` (set by MI_AOUT_Close
    at 0x7e768/0x7e770) or force the mik task loop cadence.
L2. The [Driver:] field: the mdb spdif_mode show prints [User:+0xC?][Driver:?]
    — DTS shows PCM despite my row leaving +0x18=3. Either [Driver:] reads a
    different field, or the state walk (which rewrites +0x14 in the 0x447120
    handler region) hasn't run for DTS. Proving the state walk (L1) likely
    resolves this.
L3. If the state walk is proven but the port stays dead: the port data flow
    itself (SND input channel → SPDIF TX port routing) — the
    HAL_AUDIO_DigitalOut_SetDataPath / SND port attach — trace which AC3 call
    attaches the port buffers (spdif[8bda00] = read pointers) and find its DTS
    equivalent.

State: mik d49a3790 (P-BYP5), utpa2k f8513553 (P-BYP2+3+4+6). All AC3
regressions PASS. DTS decode works (m6). Artifacts:
utpa2k_PBYP6_monitor2098gate.bin, all previous in r57_npcm/.

## ADDENDUM 2 — P-BYP7 & P-BYP8 experiments (08:00-08:40) — both reverted

**P-BYP7** (mik @0x971e4 `mvn r0,#0` → `mov r0,#0x64`; artifact mik_PBYP7_task_100ms.bin md5 299f0fd4):
hypothesis: [sp+0x4c]=-1 is the WaitEvent timeout (infinite → 100ms = task ticks
10Hz). RESULT: **ZERO monitor calls for both codecs — the wait never returned.**
LESSON: [sp+0x4c] is the WAIT-FLAGS mask (0xFFFFFFFF = match ANY event flag);
MI_OS_SetEvent(event, 1) from MI_AOUT_Close wakes it. With flags=0x64 nothing
matches → no wake (and no periodic timeout in that param). REVERTED to P-BYP5.
Wake semantics now proven: the task is EVENT-DRIVEN ONLY (AOUT Close events);
[sp+0x50]=3 = option (OR?), [sp+0x54]=0x64 = the 5th arg (unknown — possibly
timeout-in-ticks but empirically no periodic wake observed at any setting).

**P-BYP8** (utpa2k @0x446a78 `mov r6,#2`→`mov r6,#5` + @0x446a88 `movweq r6,#1`
(0x03006001 MOVW!) → NOP; artifact utpa2k_PBYP8_kick_to_state5.bin md5 c09ef909):
hypothesis: the state-machine kick (apply block, after comparator change) writes
state 1-or-2 → patch to write state 5 directly = SetNonPCM on the next tick.
RESULT: **AC3 WIRE DIED (0 bytes!)** — jumping to state 5 without walking states
1-4 (mute→49ms→reg 0x112C5F→timer) breaks the port flow for AC3 too. REVERTED
to P-BYP6. LESSON: the state walk 1→2→3→4→5 is REQUIRED sequentially; the
machine cannot be shortcut.

## Refined model after P-BYP7/8

The ApplySetting state machine MUST walk sequentially and needs ~6-8 monitor
ticks (one state per ApplySetting call; states 2 and 4 also have ms-timers).
Available DTS ticks: only the AOUT-Close-event wakes (~2-3 at play start).
The AC3 path gets repeated AOUT Close/PCM churn (MI_PCM_Close cycles) during
playback → many ticks → full walk. The DTS path's HAL never cycles the AOUT.

## Next levers (ranked, for the next session)

L1. SELF-RETRIGGER the mik task: at the loop-back (0x972bc `bne 0x9723c`),
    make the task signal its own event (MI_OS_SetEvent(_s32AoutMonitorTaskEvent,
    1)) at the end of each body iteration → continuous ticks while the task
    runs. Requires a cave trampoline (a BL insertion at a safe site near the
    loop-back with a cave in mik's .text padding), OR: patch MI_OS_SetEvent's
    caller — e.g. add the SetEvent to the per-buffer-cycle path
    (_MI_AUDIO_SetAttr is called ~3x/s during playback; insert via cave).
L2. Alternatively give utpa2k's ApplySetting the tick-independence: make the
    state-2/4 timers advance the machine MULTIPLE states per call (e.g. state 2
    handler, after the 49ms elapses, chains 3→4→5 in one call — needs a bigger
    code rewrite/cave).
L3. Or find the HAL/Kodi-side mechanism that cycles the AOUT during AC3 (the
    PCM reader renegotiation — likely _MI_AOUT_CheckInputChannelMuteStatus
    churn) and enable its DTS equivalent.

Stack state after this session: mik = d49a3790 (P-BYP5), utpa2k = f8513553
(P-BYP2+3+4+6). AC3 regression PASS on this stack (285/285, restored 1.75MB).
Artifacts: mik_PBYP7_task_100ms.bin (REVERTED-BAD), utpa2k_PBYP8_kick_to_state5.bin
(REVERTED-BAD — breaks AC3).

## ADDENDUM 3 — NIGHT SESSION 2026-09-16/17 (clean-base tests, hashkey discovery, regression caught)

### The hashkey discovery (MAJOR — changes the whole mission model)
- PRISTINE STOCK (after pkg reflash): hashkey = **(efe6bf, ffffff)** — the DTS
  license bits (mask 0x488: bits 3,7,10) are **SET** (0xefe6bf & 0x488 = 0x488).
- The OLD PATCHED-ERA device: hashkey = (ede227, fffefd) — the DTS bits CLEAR.
- CONCLUSION: **the old kernel patches corrupted the hashkey derivation and killed
  the DTS license bits. The license was in the efuse all along.** Kernel-module
  patching (mik/utpa2k) is now PROHIBITED for the mission (risk of re-corrupting).
- libmi3 (userspace) patches verified NOT to affect the hashkey (multiple reboots).

### Clean-base test matrix (stock kernel + libmi3 userspace)
| Test | Result |
|---|---|
| AC3 + libmi3_patched (2e34d0c9) | **WIRE PASS** 323/323 Pc=0x01 (1.99MB) |
| DTS + libmi3_patched | decode ✓ (m6 PLAY ~700fr), [Driver:BYPASS] **FIRST TIME**, wire 0 |
| DTS + libmi3_dts_v3 (4c199a5c) | decode ✓ (m6 PLAY), [Driver:BYPASS], wire 0 |
| AC3 + dts_v3 + IEC61937 policy | **REGRESSION: wire 0** (caught by user — no sound) |
| Revert both → stock policy + libmi3_patched | (state at power-off; verify on wake) |

### libmi3 patch mechanism (decoded from the 3-way diff @0x6163e)
- Stock: debug-log only. Patched: `OR 0x2E0` into the last word of a copied
  config struct (AC3-family format bits → the HAL's supported list → passthrough
  appears in the UI). dts_v3: `OR 0x070002E1` (+DTS/extra bits). Same site.
- Kodi settings verified: passthrough=true, dtspassthrough=true, device
  "AUDIOTRACK:AudioTrack (RAW)|Android IEC packer".

### Audio policy
- Device policy ALREADY has DTS/DTS_HD in offload+direct outputs. The corpus
  apc_patched adds IEC61937 → installed, NO effect on DTS (as user expected).
- REVERTED to stock (8504fa4b) after the regression.

### The regression (PENDING isolation)
After dts_v3 libmi3 + IEC61937 policy: AC3 wire went 0 (decode still AC3P PLAY).
Both changes reverted. Which one broke it — NOT yet isolated (test on wake:
AC3 → policy alone → AC3 → dts_v3 alone → AC3).

### Register bank dump — THE NEW NON-INVASIVE TOOL
- mdb: `echo 'reg_bank=0x112E' > /proc/utopia_mdb/audio` → dumps the live audio
  register bank (16-bit cells)! Also reg_bank=0x1603, 0x101E etc.
- IDLE: 0x98=0004, 0xB4=A494, 0xBC=000E
- AC3P PLAYING: 0x98=8401, 0xB4=C80F, 0xBC=0075
- DTS PLAYING: **NOT YET CAPTURED** (the critical missing diff!)
- write_mask_reg: the write tool exists; PARAM syntax being worked out
  (reg_bank=0x112E ✓ accepts; reg_value/mask_value ✓; write_mask_reg=0x98
  rejected — need the right PARAM format).

### The refined causal model
- The SDO copy (the SPDIF non-PCM data source) = a DDP-engine (0x81) feature.
- The m6 decodes DTS but has NO MM-path SDO copy (TV stocks: DTS passthrough =
  the HDMI-in crossover; the MM path was never wired).
- Historical P-A-era evidence: engine 0x81 + DTS bytes in the ES → SDO emitted
  VALID Pc=0x0B IEC61937 DTS bursts (the framer self-classifies by syncword).
- NEXT EXPERIMENT (data-collection-grade, no kernel patches): during a DTS
  session, flip the engine byte reg 0x112E98 → 0x81 via mdb write_mask_reg →
  if the DDP SDO copies the DTS ES → WIRE. If yes: the mission reduces to a
  runtime engine-byte flip (automatable, hashkey-safe).

### Session end state
Device powered off by user (sleep). Files: /vendor policy = STOCK (8504fa4b),
libmi3 = patched (2e34d0c9) — the DD-working combo. mik/utpa2k = pristine
stock. All tools + test media re-pushed to /data/local/tmp (rpc.sh=9090 raw
JSON version, test_ac3_51.ac3, test_dts_51.dts, libmi3_patched.so,
libmi3_dts_v3.so, audio_policy_STOCK_backup.xml).
