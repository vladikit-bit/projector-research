# CURRENT-STATE REPORT — K-1 verdict (2026-09-15, session GLM53-takeover)

Evidence-grade: every claim below is byte/timestamp/opcode verified. Prior-agent
narratives were re-derived from artifacts, not trusted.

## A. Current installed state (device 192.168.0.183, verified read-only 01:10–02:30)

| Item | Value | Verified via |
|---|---|---|
| `/vendor/lib/modules/utpa2k.ko` | md5 `58d9e8d03d2f378456814ba3ac6da9e6` = K-1 artifact (BACKUP_20260914_0717/utpa2k_K1_patch.bin) | on-device md5sum == host artifact md5 |
| K-1 deployed & booted | YES (deployed ~08:20 Sep 14; current boot epoch started ~21:05 Sep 14, uptime 14,620 s at 01:12) | /proc/uptime + on-device log mtimes |
| `/vendor/.../aucode_adec_r2_MS12V22.bin` | stock `4b7e9509` (restored during STOCKDEC control 08:10) | on-device md5sum |
| `/vendor/.../aucode_adec_r2.bin` (ALT) | stock `e26ce887` | on-device md5sum |
| `/vendor/.../aucode_asnd_r2_MS12V22.bin` | N-11 `b2a7e246` (still deployed; inert — SND /vendor files never execute either... see §C) | on-device md5sum |
| `/vendor/.../aucode_asnd_r2.bin` | stock `42c1cb79` | on-device md5sum |
| Other standing patches | mik.ko R5 `03fc2c0d`; libmi3.so dts_v3 `4c199a5c`; HAL stock | corpus inventory, re-confirmed by backup README |

Active DSP selection: MS12V22 pair (loader field 0x4d0==4), now loaded from the
EMBEDDED blobs inside utpa2k.ko (= K-1 surface).

## B. K-1 exact composition (diff of DEVICE_utpa2k.ko f44ad0a4 → utpa2k_K1_patch.bin)

100 changed bytes / 24 regions, ALL inside the embedded `mst_codec_r2_MS12V22`
blob (ko 0xA611AC..0xC451C8; region = ko−0xA611AC):

| DEC offset | old → new | Meaning | Origin |
|---|---|---|---|
| 0x045FF5 | `0c0328`→`2c0017` | P-A: `bn.sw 0x28(r3),r0` → `bn.j 0x4600C` | P-A (Sep 13) |
| 0x0DD18, 0x1E277, 0x1E38E, 0x22DC2, 0x24A47, 0x2605F, 0x2790C, 0x2AEE4, 0x2B93C, 0x2C6D6, 0x2EC9F, 0x319AB, 0x32604, 0x327D7, 0x32B36 (15×3B) | `lwz rN,HASH` → `bn.ori rN,r0,0x88` | HASH-read neutralization (19-site N-9/N-10 set minus the 0x159xx/0x230xx sites K-1 skipped) | N-9/N-10 |
| 0x01593C, 0x0230DF, 0x02972F+0x02975A, 0x029AE6 | ori→0x488 + nop lwz + m6-init block fixes | type3-select / m6-init HASH blocks | N-4/N-5/N-6 |
| 0x01768D | `0ef702`→`52e088` | applier 0x179E9-chain site | N-3 era (see note) |
| 0x01993E (4B) | `d4ef026a`→`e400cdc4` | monitor fn 0x198BD body: HASH-arg reroute for the houseKeeping type switch | N-15 new |
| 0x020020 (27B) | large rewrite (bf→branch chain around the 0xB000_0018 downgrade-byte writer at 0x2071F neighborhood; incl. `e7ff32` = bg.j rel −3226*4 → skip-write) | prevents/alters the monitor's engine-downgrade write | N-15 new |
| 0x020001..13 | NOT in K-1 (N15 had extra 0x20001-13 edits; K-1 = stock there) | — | K-1 selection |

K-1 = closest to N13surgical (81 bytes apart) + N15's two semantic patches
(0x1993E, 0x20020) − N15's bulk 62-region set. NOT equal to any N artifact.
No ARM/.text changes; no SND/ALT blob changes; sizes/sections untouched.

Build/deploy records: no build script or report file was ever written (the
session died before documentation — only artifacts exist). Backup manifest:
BACKUP_20260914_0717 (README + md5s + DEVICE_* files + DEVICE_utpa2k.ko).

## C. Loader truth (independently verified, matches R8 of 2026-09-06)

- All four DSP images are embedded in utpa2k.ko `.data` as ELF symbols
  (byte-identical to the /vendor files): mst_codec_r2 ko+0x7C50B8 (2,736,372 B),
  mst_codec_r2_MS12V22 ko+0xA611AC (1,982,492 B), mst_snd_r2 ko+0xC54ABC
  (1,567,048 B), mst_snd_r2_MS12V22 ko+0xDD3404 (1,839,920 B).
- HAL_AUDSP_DspLoadCode @0x46f5a8 memcpy's the EMBEDDED arrays; NO file I/O
  (no filp_open/request_firmware/kernel_read). Selection: `g_AudioVars2+0x4d0==4`
  → MS12V22 pair. /vendor/lib/utopia/audio_bin/*.bin are byte-identical dormant
  snapshots — the loader NEVER reads them.
- PROOF the /vendor DEC files never executed (three independent controls, Sep 14):
  1. XMARK: X-marker in /vendor MS12V22 `decType change` string → log shows `decType` (0× XecType).
  2. XALT: X-marker in /vendor ALT same string → still `decType` (0× XecType).
  3. STOCKDEC: /vendor DEC both stock → log byte-identical transitions.
  Also: the "N-11 SND effect" (type3 key1 0xede227→0) and the "N-7 boot effect"
  (0xefe7ff) flicker independent of deployment state — runtime variance, not patch effects.
- Reinterpretation of history: every prior /vendor DEC patch (V1/L2/D1/P-A/N-1..N-15)
  was INERT (deployed bytes never executed). The P-A-era "first DTS bursts" and
  N-era 114–152-burst captures coincide with the first-ever raw `.dts` playback
  through Kodi's PAPlayer (type[81] passthrough engine) — a player-path change,
  not a patch effect. Stock produces the same bursts on the same path (N3_DTS_dts.bin:
  152 DTS + 273 AC-3 preambles with stock-equivalent embedded code).

## D. Runtime baseline after K-1 (fresh tests this session)

AC3 regression (×2: 08:39 Sep-14 capture + 01:52 Sep-15 my control):
PASS — 4,177,920 B / 680 preambles, 100% Pc=0x01, stride 0x1800. No regression.

DTS under K-1 (two runs: R2-armed + clean cap.sh, raw `.dts` via Kodi PAPlayer):
- Engine: `type[3]→type[81]` at play; **`81→4` did NOT occur during playback**
  (the 2.2-s kill is gone; the switch only appears at log tail post-stop, tick 15496912).
- BUT: wire capture = **0 new bytes** (dump file unchanged, md5 identical to prior
  AC3 content). DEC type<81> telemetry during playback: `ES=0000 PCM=0000 play=0
  frmCnt:0000` for all 492 lines — the passthrough engine never received frames.
- licensee=0 ×2, es:0, cB:0 in type[4] window; type4-select key1 STILL 0xede227
  (K-1's embedded HASH patches did NOT zero it — the m6 SDK reads the efuse HASH
  via an unmodelled mechanism, per addendum 9, now confirmed under the real surface).
- Net DTS result: WORSE than stock-embedded behavior: stock produced intermittent
  (114–152) valid DTS bursts via PAPlayer; K-1 produces none.

## E. Decision

**K1-CHANGED-PATH-BUT-NOT-FIXED** (and DTS wire regressed vs embedded-stock).

- The 81→4 kill: REMOVED during playback (0x20020/0x1993E monitor patches work —
  or the switch simply occurs post-stop; both consistent with data).
- The wire: EMPTY during the 81 window — because the type[81] engine never got
  frames (play=0, frmCnt 0000 whole session). The bottleneck moved UPSTREAM of
  DEC: into the host/Kodi→DEC passthrough feed, NOT the DEC image.
- Root-cause implication: with the embedded surface now being the real one, the
  meaningful next lever is the ARM/host layer of utpa2k.ko (the frame delivery /
  SDO feed into type[81]), or Kodi's PAPlayer passthrough negotiation. DEC-image
  patching has been exhausted (N-1..N-15 + K-1 all failed on licensee/keys —
  these come from the m6 SDK's own efuse reads, not the image).

## Artifacts added this session (nothing overwritten except the one accident, repaired)

- r57_npcm/K1BOOT_decr2.log (08:25 boot log pulled from device)
- r57_npcm/K1_DTS_decr2.log + K1_DTS_wire.bin (my K-1 DTS test)
- r57_npcm/K1CTL_AC3_wire.bin (my AC3 control)
- r57_npcm/N2A_DTS_sndr2_00.log, N2B_DTS_sndr2_01.log (device-only SND logs preserved)
- INCIDENT: first pull of N2B overwrote host N2_DTS_sndr2.log (was a copy of _00,
  identical md5 d7c21ee9 preserved in N1G_DTS_sndr2.log + N2A_DTS_sndr2_00.log —
  zero data loss).

## F. ADDENDUM (2026-09-15 evening): K-2 + N3 autopsy + NEW DIRECTIVE

### F.1 K-1 post-verdict audit findings
- K-1's 0x20020 cave remapped the WRONG register: r15 = current engine state,
  r7 = incoming request → the 4→0x81 remap never engaged as designed.
- The cave also destroyed live code at 0x20020 (a copy-loop: muls / jal 0xE0CF /
  lwz / beq chain) — the "works" effect is partly damage, partly post-stop timing.
- Mirror↔DSP-VA data mapping is UNRELIABLE (config/licensee fields read 0 even
  during live DTS) — do not use the host mirror to prove DSP-side config state.

### F.2 K-2 (current installed state)
- Approach changed surfaces: ARM .text of utpa2k.ko, DTS case @0x44d1dc rewritten
  to mirror the AC3 case (orr r0,r4,#1; uxtb r1,r0; mov r0,r6; nop; nop) —
  17 bytes in ARM text; embedded DEC blob restored FULLY STOCK inside K-2 ko.
- Results: type[81] HELD on both DTS-video and raw-.dts paths; ES buffer fills
  (es:2560 appears on raw .dts); BUT PCM=0, play=0, 0 LvL, wire 0 bytes.
- AC3 regression under K-2: PASS (3874 es:1536 lines, 7.5MB dump).
- AC3→DTS direct switch: the decode loop DIES at the switch (es:2560 never seen
  after switch); play→stop→play→pause cycles; PCM=0.

### F.3 N3 autopsy (why the historical 114–152 bursts happened) — CLOSED
- N3 success = AC3 playback FIRST (engine 0x81 already primed by the running AC3
  decoder in cmd<4> state), then the DTS stream was switched INTO the live 0x81
  engine. The bursts came from Kodi's TrySyncAC3 misdetect opening .dts as
  Codec:5: the whole pipeline was configured for AC3/DDP and consumed DTS frames
  as "AC3" payload; the SDO re-emitted them with Pc=0x0B.
- CONSEQUENCE: every historical "DTS burst" observation is explained by the AC3
  configuration path feeding DTS bytes. No DEC image ever emitted DTS on its own
  DTS-configured path.

### F.4 NEW DIRECTIVE (user, 2026-09-15) — replaces all N3/Kodi plans
- N3 replication / Kodi misdetect / historical burst artifacts: STOPPED.
- Baseline = current K-1/K-2 runtime difference: AC3 feeds ES + wire PASS;
  DTS gets type[81] + ES fill but play=0/PCM=0/wire=0.
- TASK: find the FIRST host-side state/configuration difference between Codec ID 5
  and Codec ID 9 after MI_AUDIO_Start() and before the R2 decoder consumes ES.
- Trace chain: MI_AUDIO_Start(5/9) → subsequent MI_AUDIO/Utopia calls →
  SetAttr / SetDecodeSystem / InputProcess configuration → SHM/mailbox writes →
  R2 DEC input/feed.
- Deliverable: the minimal parameter/state explaining why AC3 feeds ES and DTS
  does not — PROVEN by (a) static data-flow AND (b) runtime correlation,
  BEFORE any patching.
- Ignore (unless reopened by this trace): EDID, SPDIF mode, MI_AUDIO_GetHandle,
  generic transport investigations.
- User note: the only remaining Kodi-side idea is a Maven audio packer/encoder
  (tested, also does not work) — Kodi side is effectively exhausted; focus is
  the firmware host path.

### F.5 State snapshot at directive time
- Device /vendor/lib/modules/utpa2k.ko = K-2 (ARM 17B DTS-case + stock DEC blob).
- BACKUP_20260914_0717 = full rollback point (K-1 era + stock images).
- DEC-image patching declared EXHAUSTED (N-1..N-15, K-1 all failed on
  licensee/keys sourced from m6 SDK efuse reads, not from the image).
