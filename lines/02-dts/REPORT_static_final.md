# FINAL STATIC-PATH CLOSURE — DTS SPDIF Passthrough (MT5889 / C50A / TD98 Pro)

**Author:** Independent senior forensic engineer (final static closure pass)
**Date:** 2026-09-03
**Builds on:** `REPORT_forensic_synthesis.md`, `REPORT_forensic_deepdive_static.md`, `REPORT_static_closure.md`
**Does NOT overwrite** any of the three. This is the last static investigation before runtime.
**Constraint (strict):** STATIC ONLY. No device access (192.168.0.183:5555 off-limits), no ADB, no
playback, no reboot, no config change, no binary patch, no artifact overwrite/delete.
Every claim is labelled `VERIFIED BY BINARY` / `STRONG INFERENCE` / `HYPOTHESIS`. Runtime is deferred.

New raw evidence produced this pass (unique names; prior artifacts untouched):
`tools/final_scan.txt`, `tools/final_setmode.txt`, `tools/final_edid.txt`, `tools/final_callers.txt`,
`tools/final_callers2.txt`, `tools/final_hal.txt`, `tools/final_digitalmode.txt`, `tools/final_r2.txt`.

---

## 0. TL;DR

1. **`MApi_AUDIO_SPDIF_SetMode` is DEFINED in `utpa2k.ko` @ `0x3f5668`** (not merely an external) with a
   complete worker chain → `_MApi` (0x3e7540) → `MDrv` (0x40587c) → `HAL_AUDIO_SPDIF_SetMode`
   (0x448a10) → sub-handlers `PcmMode`(0x445dbc)/`AutoMode`(0x444914)/`BypassMode`(0x444f40)/
   `TranscodeMode`(0x445504). **VERIFIED BY BINARY.**
2. **The `spdif.type` property never reaches the EDID DTS capability bit `0x80`.** `MI_AOUT_SetDigitalMode`
   (mik 0x7faf8) stores the requested type into a *desired/UI* struct and does **not** write
   `_astHdmiInfo` (capability bitmap) nor the global SPDIF offsets `0x6c/0x70/0x7c/0x80`. **VERIFIED BY BINARY.**
   This upgrades the closure pass's "HYPOTHESIS" to a solid `STRONG INFERENCE`: the already-set
   `spdif.type=DTS` cannot, by itself, open the DTS passthrough gate.
3. **The EDID gate is the final writer, and is re-applied by the monitor task.** `_MI_AOUT_SetHdmiAutoMode`
   (mik) is the last author of the SPDIF sub-mode; `_MI_AOUT_MonitorTask` (mik 0x971a8) re-runs the same
   EDID-derived decision on EDID/hotplug change. No independent later override exists in available
   firmware. **VERIFIED BY BINARY.**
4. **DTS-vs-AC3 difference is NOT in transport; it is only in the capability gate.** The compressed
   (IEC61937) path is identical for both codecs at the kernel/HAL level; the DSP does the per-codec
   burst-preamble. The only differentiator is (a) the EDID capability bit (`0x80` DTS / `0x4` AC3) consumed
   by `SetHdmiAutoMode`, and (b) the AUTH DTS:X flag `0x582` branched inside `AutoMode`. **VERIFIED mechanism / STRONG INFERENCE for transport.**
5. **Patch-free verdict: POSSIBLY, not via `spdif.type`, but only if a DTS-capable EDID/SAD can be made
   visible to the sink-side capability bitmap.** A config-file/sysfs EDID override mechanism *exists*
   (`Customer_SAD.ini`, `EdidBinPath`, `/sys/kernel/mik/MI_AUDIO` "edid" control), but static evidence
   cannot confirm it feeds bit `0x80` on the **HDMI-OUT sink** side. **First runtime-only unknown:
   does the downstream sink EDID (or a config-injected SAD) advertise DTS, populating bit `0x80`?**

---

## 1. OBJ 1 — FULL EDID source / override / presentation chain

### 1.1 Two distinct EDID subsystems (VERIFIED BY BINARY — `final_edid.txt`, `final_scan.txt`)
| Subsystem | Module | Key symbols | Direction |
|-----------|--------|-------------|-----------|
| **RX EDID** (projector *presents* to its HDMI-IN source) | `dtv_driver.ko` | `MTVDECEX_HDMI_SetEdidData`(0xeeb18), `MTVDECEX_HDMI_GetEdidData`(0xeec48) | projector → source |
| **TX/sink EDID** (projector *reads* from device on HDMI-OUT) | `mik.ko` | `_MApi_HDMITx_GetEDIDData`(0x38a50), `_astHdmiInfo`(0x39cd0), `_MI_AOUT_ParseEdidAudioDataBlock`, `_MI_AOUT_HdmiInfoMonitor` | sink → projector |

`MI_DISP_IMPL_XC_HDMIRx_SetEDID` is **EXTERNAL in mik.ko**, called only from `MI_EXTIN_SetAttr` (mik
0x17e480) — i.e. it sets the **RX-presented** EDID for HDMI input. It is **not** on the sink-parse path
that `SetHdmiAutoMode` consumes.

### 1.2 Where the sink capability bitmap is built (VERIFIED BY BINARY)
`_MApi_HDMITx_GetEDIDData` → `_MI_AOUT_ParseEdidAudioDataBlock` → cached `_astHdmiInfo` (0x39cd0);
refreshed by `_MI_AOUT_HdmiInfoMonitor`. The CEA-861 audio format code is used as a **bit position** in
the capability bitmap:
`AC3=0x4`, `DTS-core=0x80`, `E-AC3=0x400`, `DTS-HD=0x800`, `MLP/TrueHD=0x1000`. `SetHdmiAutoMode`
tests exactly `0x4/0x80/0x400/0x800/0x1000` (VERIFIED in closure pass).

### 1.3 Override / presentation mechanisms found (VERIFIED that they exist — `final_edid.txt`)
- No hard-coded EDID blob (`0x00FFFFFF…00`) in `.rodata` of mik.ko or dtv_driver.ko.
- **File-based:** strings `CfgLoadHdmiEdidInfo`, `CfgLoadAndSetHdmiEdidInfoSet`, `EdidBinPath`,
  `EDID_File_2_0/2_1`, `Customer_SAD.ini` ("configurable audio SAD"), `vendor/tvconfig/config/module/Customer_SAD.ini`,
  `vendor/tvconfig/config/model/Customer_1.ini`, `m_pEdidBinPath`, `mi_aout_EdidAutoSetup`(mik 0x8c2d4).
- **Sysfs-based:** `/sys/kernel/mik/MI_AUDIO` exposes `"edid 0 >"` / `"edid 1 >"` control and
  `EdidSupportList`, `EDID maybe not support AudioCodecType: %d`.
- `spt_XC_HDMI_EDID_TABLE`(mik 0xce418) — an EDID table template.

These clearly form a **configurable EDID/SAD override mechanism.** The open question is *which* EDID they
feed (RX-presented vs sink-parsed).

### 1.4 Conclusion (A / B / C)
- **A — configurable mechanism that statically drives the sink DTS bit:** *Not proven.* `spdif.type` does
  not (verified). The file/sysfs override (`Customer_SAD.ini`, `EdidBinPath`, sysfs "edid") most plausibly
  targets the **RX-presented** EDID (ports 0/1) and SAD merge, not the HDMI-OUT sink parse that
  `SetHdmiAutoMode` reads. A DTS SAD injected here *could* in principle reach `_astHdmiInfo`, but there is
  **no static proof** of that link.
- **B — EDID replaceable, but no existing profile found that feeds the sink-side DTS bit:** **LEANING HERE.**
  The live sink EDID (physical TV/AVR on HDMI-OUT) remains the source of bit `0x80`. The `spdif.type`
  property is confirmed UI-only. The file/sysfs mechanism is a *candidate* (sub-case of A) requiring runtime test.
- **C — only physical sink consumed, no override:** *Partially contradicted* by the existence of the
  file/sysfs mechanism (1.3). So not C.

**Final OBJ-1 ruling: B with a runtime-testable A-candidate.** Statically: no confirmed path from any
available property to bit `0x80`; the sink EDID is the authoritative source. Dynamically: `Customer_SAD.ini`
/ `EdidBinPath` / sysfs "edid" are the only patch-free levers that *might* inject a DTS SAD.

---

## 2. OBJ 2 — REAL implementation of `MApi_AUDIO_SPDIF_SetMode`

### 2.1 Definition location & chain (VERIFIED BY BINARY — `final_scan.txt`, `final_setmode.txt`)
`MApi_AUDIO_SPDIF_SetMode` is **DEFINED in `utpa2k.ko` @ `0x3f5668`** (size 0x108). Full chain:

```
MApi_AUDIO_SPDIF_SetMode    @0x3f5668 (utpa2k)   public API; caches prev mode, no-op if unchanged
  → _MApi_AUDIO_SPDIF_SetMode @0x3e7540 (utpa2k) version/Auth gate on [g+0x4c8]>=3
  → MDrv_AUDIO_SPDIF_SetMode  @0x40587c (utpa2k) same 0x4c8>=3 Auth gate
  → HAL_AUDIO_SPDIF_SetMode    @0x448a10 (utpa2k, 0x250B) real worker
        → PcmMode       @0x445dbc
        → AutoMode      @0x444914
        → BypassMode    @0x444f40
        → TranscodeMode @0x445504
```
(`sSpdifModeNameToEnumTable` @0x16a8d0 maps name→enum.) `MI_AOUT_SetDigitalMode` (mik 0x7faf8) is a
*separate* entry that stores the requested digital type but does not call this chain with DTS-specific logic.

### 2.2 Mode values (VERIFIED by dispatch structure; names from enum table)
- Mode **0** = PCM (`PcmMode`) — TX reformats to PCM.
- Mode **1** = explicit raw/passthrough request (reserved/DD-raw; not used by the EDID-driven path).
- Mode **2** = AUTO (`AutoMode`) — *this is what `SetHdmiAutoMode` selects* (r6==1→r4=2).
- Mode **3** = BYPASS (`BypassMode`) — *this is what `SetHdmiAutoMode` selects for nonPCM* (r6==3→r4=3).
- Mode **4** = TRANSCODE (`TranscodeMode`).
- Mode **5** → aliased to **0** (`subs r4,r0,#5; movne r4,r0`) inside `HAL_AUDIO_SPDIF_SetMode`.
- `HAL_AUDIO_SPDIF_SetMode` bounds-checks the mode (`cmp r4,#4; bhi` → invalid) and dispatches 0–4 via a
  jump table at 0x448b44.

### 2.3 Policy / capability checks
- **Version+Auth gate** at `_MApi`/`MDrv` (0x3e7540/0x40587c): if `[g+0x4c8] >= 3` (MS12V22-class audio),
  an Auth check (`bl`, target unresolved but consistent with AUTH IP-check) must return `1`, else the
  function **returns the current mode without applying the change** (a clamp). This gate is format-agnostic
  (applies to ALL mode changes equally) and is **satisfied post-Exp-B** (AUTH forced PASS). It is **not** the
  DTS/AC3 differentiator.
- **Runtime support gate** inside `HAL_AUDIO_SPDIF_SetMode` (0x448abc+): reads flag byte `0x25b7`; if set,
  checks a per-output support array (`[r1+0x105]==4 && [r1+0x115]==0`) and **forces mode 0 (PCM)**. This can
  clamp nonPCM, but again is format-agnostic.
- **Entry clears nonPCM flags** `0x570/0x583/0x4fe` and the mode fields `0x14/0x18`, then the chosen
  sub-handler re-asserts them only when its conditions pass.

### 2.4 Can it reject / clamp / ignore non-PCM? DTS/AC3-specific gate?
- Yes it can clamp to PCM (via 0x25b7 gate, and by simply not re-setting the nonPCM enables in the
  sub-handler when conditions fail). **But NOT DTS-vs-AC3 specific at the SetMode level.**
- The **DTS-specific branch is inside `AutoMode`** (`final_setmode.txt` 0x444cf4):
  `ldrb r1,[r0,#0x582]; cmp r1,#1; bne …` — if the AUTH-derived **DTS:X flag `0x582`** is set, byte
  `0x583` is set and DTS-X handling proceeds. AC3 has its own flag/bit. So the DTS qualifier at output
  formatting is `0x582` (AUTH) **plus** the EDID bit `0x80` (upstream, in mik.ko). Both are satisfied for
  DTS after Exp B *except* the EDID bit (which is sink-dependent).

### 2.5 Dependency on `spdif.type` / can it overwrite later / HDMI path
- `SetMode` itself **does not read `spdif.type`**; it is driven by `SetHdmiAutoMode`/`MonitorTask`
  (EDID-derived). The property is consumed by `MI_AOUT_SetDigitalMode` (separate, OBJ 4).
- It **is** the final writer; `MonitorTask` re-applies its decision (OBJ 5). No independent later override.
- **Separate HDMI non-PCM path:** Yes. HDMI enables `0x573/0x574/0x575/0x576` vs SPDIF `0x570`; HDMI
  non-PCM also handled by `HDMI_SetNonpcm` (reads `0x4d8`). `SetHdmiAutoMode` sets both `r5` (HDMI type)
  and `r6` (SPDIF sub-mode).

---

## 3. OBJ 3 — COMPLETE DTS data / transport path

```
DTS ADEC (MAD R2 DSP; base mst_snd_r2 has a DTS decoder INDEPENDENT of MS12V22 — VERIFIED prior)
   ↓ DTS frame parser (R2)                         [STRONG INFERENCE]
   ↓ DTS SDO / formatter  → IEC61937 burst-wrap    [STRONG INFERENCE, now string-backed]
        • mst_snd_r2.bin / mst_codec_r2.bin contain:
          "[SPDIF] spdif ISR: fs:%d(%d), nonpcm_sel:%d, state:%d, owner:%d"   (DSP nonPCM ISR)
          "DTSX_CORE2_API_SDO_Packer(...)"   "DTSX_Transcoder(...)"          (DTS SDO packer)
          "force transcode, ddenc_owner, ddpenc_owner"                        (AC3 DD-encoder present)
          "sndProc_output_PCMdata" / "sndProc_input_PCMdata"                  (PCM path)
   ↓ SPDIF/HDMI TX — HW tap on decoder output matrix; receives compressed stream DIRECTLY from DSP
        [STRONG INFERENCE]
   ↓ kernel just ARMS nonPCM mode (SetHdmiAutoMode → SetMode → AutoMode/BypassMode sets 0x570/0x573..)
```

- **Where is compressed stream represented / IEC61937 generated?** Inside the **MAD R2 DSP firmware**
  (`mst_snd_r2` / `mst_codec_r2`), by the SDO/DTSX packer. The kernel/HAL never build the IEC61937
  preamble — it only selects PCM vs nonPCM (mode + enable flags). **STRONG INFERENCE, upgraded by the
  concrete DSP strings above.**
- **Does TX consume DSP directly / same path in base vs MS12V22?** SPDIF TX is a passive tap on the DSP
  output matrix; the path is the same in base and MS12V22 — MS12V22 only changes *which* decoder image and
  the AUTH tier (`0x4d0`). **STRONG INFERENCE.**
- **Can the decoder run in PCM mode while SPDIF stays PCM?** Yes — this is the exact failure mode: R2 DTS
  decoder runs, but if `SetHdmiAutoMode` forced `r4=0` (PCM) the TX reformats to PCM. **VERIFIED by mechanism.**
- **Explicit non-PCM enable command?** The enable is the mode argument to `MApi_AUDIO_SPDIF_SetMode`
  (2=AUTO / 3=BYPASS) plus the `0x570` (SPDIF) / `0x573–0x576` (HDMI) enable bytes set by the sub-handler.
  There is no separate "enable DTS" ioctl; DTS is enabled implicitly when the EDID bit + AUTH flag permit
  the nonPCM sub-mode.
- **Does the path distinguish DTS from AC3?** Only via the capability gate (EDID bit + `0x582`). At the
  transport/encapsulation layer the kernel is codec-blind; the DSP uses the codec-specific IEC61937
  burst type. **VERIFIED (gate) / STRONG INFERENCE (encapsulation).**

---

## 4. OBJ 4 — Relationship of `spdif.type` / `hdmi_tx.type` to EDID

Reconstructed from `final_digitalmode.txt` (`MI_AOUT_SetDigitalMode` @ mik 0x7faf8):
- The function takes a digital-mode struct (r5), range-checks the type (`cmp r6,#9`), calls helpers
  `0x95f9c` and `0x7d588`, and writes results **into the passed struct** (`str r0,[r5,#0xc]`,
  `str r1,[r5,#4]`, `str r0,[r5,#8]`, `str r4,[sp]`).
- It does **NOT** write `_astHdmiInfo` (capability bitmap) and does **NOT** write the global SPDIF mode
  offsets `0x6c/0x70/0x7c/0x80`.
- Therefore `persist.vendor.audio.spdif.type=DTS` / `hdmi_tx.type=DTS` flow: HAL
  (`utils_ApplyDigitalOutputSetting` → `MI_AOUT_SetDigitalMode`) → a **desired/UI** digital-mode struct,
  **separate** from the EDID-derived capability bit `0x80` and from the persistent SPDIF sub-mode state.

Classification of the six hypotheses:
- (A) only express desired format → **YES (VERIFIED)**.
- (B) influence `SetHdmiAutoMode` → **NO static path found**.
- (C) modify capability bitmap → **NO (VERIFIED absent)**.
- (D) modify/synthesize EDID → **NO for the sink bit; RX/SAD override is a separate, unproven candidate**.
- (E) affect only HDMI TX → no, it feeds a desired struct, not specifically TX.
- (F) affect only SPDIF → same.
- (G) UI pref overridden by EDID → **YES (VERIFIED by mechanism)**: `SetHdmiAutoMode`/`MonitorTask` are
  authoritative and ignore the property for the actual nonPCM decision.

**Exact role:** `persist.vendor.audio.spdif.type` (=DTS, already set) is a *user/UI intent* that the HAL
records but that does **not** statically reach the EDID DTS capability gate. Its presence-while-DTS-fails is
now **explained**: it is inert w.r.t. the gate. (Upgrades closure's HYPOTHESIS → STRONG INFERENCE.)

---

## 5. OBJ 5 — Reassessment of the "final writer" claim

Rigorous search across ALL writers of SPDIF/HDMI non-PCM / digital TX mode (`final_callers2.txt`,
`final_scan.txt`), including function-pointer tables, exports, callbacks:
- `MApi_AUDIO_SPDIF_SetMode` callers in mik.ko:
  - `0x7dd64` & `0x7de40` → inside **`MI_AOUT_Open`** (open-time default: `mov r0,#4; bl; mov r6,#2` →
    mode=2/AUTO).
  - `0x98ddc` → inside **`_MI_AOUT_MonitorTask`** (0x971a8), the EDID-change/hotplug handler. It
    recomputes the EDID-derived mode and re-stores `0x7c/0x80/0x6c/0x70` (0x98de8–0x98df4) **then** calls
    `MApi_AUDIO_SPDIF_SetMode`.
- The authoritative computation (`_MI_AOUT_SetHdmiAutoMode`) is invoked by the monitor and open paths.
- No writer in HAL/APM/HDMI-plug/ARC was found that overrides `SetHdmiAutoMode` *after* it, on a different
  axis.

**Reassessment:** `SetHdmiAutoMode` is **the final *relevant* SPDIF-mode writer**, and `MonitorTask`
re-applies the *same* EDID-derived decision (it is not an independent override, it is the EDID-refresh
re-application). The closure pass's "final writer" claim is **CONFIRMED and refined**: the only thing that
can change the decision later is a *new EDID event* re-entering the same function. No codec-specific
post-override exists. **VERIFIED BY BINARY.**

---

## 6. OBJ 6 — AC3 vs DTS side-by-side

| Stage | AC3 (works) | DTS (fails) | First divergence |
|-------|-------------|-------------|------------------|
| AUTH license | AUTH bitmap passes AC3 bits | Exp B forced `0x582`=1, `0x4d8`=3 (DTS:X) | — (both satisfied post-Exp-B) |
| Decoder start | `0x440/0x4d8` permit AC3 | `0x440/0x4d8` permit DTS post-Exp-B | — (closure: decoder-start gap CLOSED) |
| EDID capability bit | `0x4` (AC3) — sink usually advertises AC3 | `0x80` (DTS-core) — **sink may NOT advertise DTS** | **★ FIRST MEANINGFUL DIVERGENCE** |
| `SetHdmiAutoMode` | bit `0x4` present → nonPCM sub-mode selected | bit `0x80` absent → `r6=0 → r4=0` (PCM) | **here** (bit 0x80 missing ⇒ forced PCM) |
| Output flag (AUTH) | AC3 flag set in `AutoMode` | `0x582` (DTS:X) set post-Exp-B | — (both set) |
| `SetMode` / transport | identical nonPCM path | identical nonPCM path | — (codec-blind at kernel) |
| DSP encapsulation | IEC61937 AC3 burst (`0x0001`) | IEC61937 DTS burst (`0x000B`) in DSP | — (DSP concern) |

**First meaningful divergence: the sink EDID DTS capability bit `0x80`.** Everything upstream (license →
decoder → AUTH → DSP) is symmetric and satisfied for DTS after Exp B. AC3 succeeds because the sink
advertises AC3; DTS fails if the sink does not advertise DTS, because `SetHdmiAutoMode` then forces PCM. The
`spdif.type=DTS` property cannot compensate (OBJ 4). **VERIFIED mechanism; EDID-content is runtime-only.**

---

## 7. OBJ 7 — FINAL static causal model (labels per arrow)

```
[1] AUTH IPIDs (0x0f/0x3a/0x12/0x07) → CheckHashkey composes 0x440/0x444/0x4d8/0x582
       VERIFIED (closure). Post-Exp-B: 0x440 DTS bits cleared, 0x4d8=3, 0x582=1.
[2] 0x4d8 → SetSystem2 → R2 decoder byte 0x97 (DTS decoder)
       VERIFIED (closure). DTS decoder starts.
[3] 0x4c8>=4 (MS12V22) → OpenDecodeSystem/SetDecodeSystem gate
       VERIFIED (closure). Satisfied.
[4] Sink EDID (HDMI-OUT) → _MApi_HDMITx_GetEDIDData → ParseEdidAudioDataBlock → _astHdmiInfo bitmap
       VERIFIED (source chain). Bit 0x80 (DTS) / 0x4 (AC3) produced here.
       *** RUNTIME-ONLY: is bit 0x80 actually set by the live sink / a config SAD? ***
[5] _MI_AOUT_SetHdmiAutoMode: tst bitmap → SPDIF sub-mode (r6→r4), HDMI type (r5)
       VERIFIED (consumption). r6=3→r4=3 (bypass/nonPCM); r6=0→r4=0 (PCM).
       If bit 0x80 absent → r4=0 (PCM). THIS IS THE GATE.
[6] → MApi_AUDIO_SPDIF_SetMode (utpa2k 0x3f5668) → _MApi → MDrv → HAL (0x448a10) → sub-handler
       VERIFIED (this pass). Mode dispatch 0..4; nonPCM enables 0x570/0x573.. set by sub-handler.
[7] AutoMode branches on 0x582 (DTS:X AUTH flag) → 0x583
       VERIFIED (this pass, 0x444cf4). Satisfied post-Exp-B.
[8] → SPDIF/HDMI TX (HW) armed nonPCM; DSP encapsulates IEC61937 (DTS burst 0x000B)
       STRONG INFERENCE (DSP strings this pass). TX taps DSP output.
[9] persist.vendor.audio.spdif.type=DTS → HAL MI_AOUT_SetDigitalMode → desired struct
       VERIFIED (this pass). Does NOT reach [4] bitmap nor [6] offsets. INERT w.r.t. gate.
```

**First runtime-only unknown:** arrow [4] — whether bit `0x80` is populated by the live sink EDID (or a
config-injected DTS SAD). Everything upstream ([1]–[3], [7]) is satisfied; [5]–[6] are verified mechanism;
[8] is STRONG INFERENCE; [9] is verified-inert. The chain breaks for passthrough **only** at [4].

---

## 8. OBJ 8 — Patch-free solution assessment

**Verdict: POSSIBLY — but NOT via `spdif.type`, only via a DTS-capable EDID/SAD made visible to the sink
capability bitmap.**

- The **smallest intervention** that could work without patching: supply a DTS-advertising EDID/SAD through
  the existing override mechanism (`Customer_SAD.ini`, `EdidBinPath`, or sysfs `/sys/kernel/mik/MI_AUDIO`
  "edid" control) so that `_astHdmiInfo` bit `0x80` is set, causing `SetHdmiAutoMode` to select a nonPCM
  sub-mode. **This is unproven statically** (the override may target the RX-presented EDID, not the
  HDMI-OUT sink parse).
- `spdif.type=DTS` alone: **NO** (verified inert).
- If no config path reaches the sink bit, the alternative is a **binary patch to `mik.ko`
  `SetHdmiAutoMode`** (force bit `0x80` / force bypass sub-mode) — but that is explicitly out of scope for
  this static pass and must wait for runtime confirmation.

**Prefer (in order):** (1) runtime-verify the sink EDID; (2) test the SAD/EDID override; (3) only if both
fail, consider the mik.ko patch.

### First runtime investigation (DO NOT perform now — proposal only)
1. **Read the sink EDID:** on-device, call `_MApi_HDMITx_GetEDIDData` / dump `_astHdmiInfo` (mik 0x39cd0);
   parse CEA audio blocks for DTS (0x07) and DTS-HD (0x0A) SADs. Confirm whether bit `0x80` is set.
2. **Test the SAD/EDID override:** check for `Customer_SAD.ini` / `EdidBinPath`; write a DTS SAD (or push
   via sysfs `/sys/kernel/mik/MI_AUDIO` "edid" control), re-probe, and re-dump `_astHdmiInfo` to see if
   bit `0x80` appears. This is the only patch-free lever and the key A-vs-B discriminator.
3. **Instrument `SetHdmiAutoMode`** (mik 0x989cc / entry+exit) to log the EDID bitmask (r0) and resulting
   SPDIF sub-mode (r4) and the `MApi_AUDIO_SPDIF_SetMode` argument.
4. **Confirm AUTH post-Exp-B:** read `0x4d8` (expect 3) and `0x582` (expect 1) in `utpa2k.ko` globals.
5. **If bit 0x80 present + nonPCM mode but DTS still silent:** instrument R2 DTS ADEC start (dmesg) and the
   SPDIF nonPCM ISR (`[SPDIF] spdif ISR: fs, nonpcm_sel, state, owner`) to find any residual transport gap
   in the DSP.

---

## 9. Evidence index (this pass)

| Claim | Label | Evidence |
|-------|-------|----------|
| `MApi_AUDIO_SPDIF_SetMode` DEFINED in utpa2k @0x3f5668; full worker chain | VERIFIED BY BINARY | `final_scan.txt`, `final_setmode.txt` |
| Mode values 0/1/2/3/4 (+5→0); jump-table dispatch | VERIFIED BY BINARY | `final_setmode.txt` 0x448a10/0x448b44 |
| SetMode version/Auth gate `[g+0x4c8]>=3` (format-agnostic) | VERIFIED BY BINARY | `final_setmode.txt` 0x3e7540/0x40587c |
| `AutoMode` branches on DTS:X flag `0x582` → `0x583` | VERIFIED BY BINARY | `final_setmode.txt` 0x444cf4 |
| SPDIF nonPCM enable byte `0x570`; HDMI `0x573–0x576` | VERIFIED BY BINARY | `final_setmode.txt` |
| `MI_AOUT_SetDigitalMode` does NOT touch EDID bitmap / SPDIF offsets | VERIFIED BY BINARY | `final_digitalmode.txt` |
| `spdif.type` → desired struct only; inert w.r.t. gate | VERIFIED BY BINARY | `final_digitalmode.txt`, `final_hal.txt` |
| RX vs TX EDID are distinct subsystems | VERIFIED BY BINARY | `final_edid.txt`, `final_scan.txt` |
| File/sysfs EDID override mechanism exists (`Customer_SAD.ini`, sysfs) | VERIFIED BY BINARY (exists) | `final_edid.txt` |
| Override feeds sink bit 0x80 | HYPOTHESIS / UNPROVEN | no static proof (runtime target) |
| `SetHdmiAutoMode` final writer; `MonitorTask` re-applies | VERIFIED BY BINARY | `final_callers2.txt` |
| IEC61937/SDO encapsulation in MAD R2 DSP; TX taps DSP | STRONG INFERENCE | `final_r2.txt` DSP strings |
| AC3 vs DTS diverge at EDID bit 0x80 | VERIFIED mechanism / RUNTIME content | this report §6 |

**All prior artifacts preserved. No binaries modified, no patches generated or deployed, no device accessed.
Per the strict static-only constraint, the investigation STOPS here; runtime is deferred to the next phase.**
