# Static-Only Forensic Closure — DTS SPDIF Passthrough (MT5889 / C50A / TD98 Pro)

**Author:** Independent senior forensic engineer (final static closure pass)
**Date:** 2026-09-03
**Builds on:** `REPORT_forensic_synthesis.md` (Phase 4) and `REPORT_forensic_deepdive_static.md` (deep-dive).
**Does NOT overwrite** those reports. This pass closes the decoder-start gap, completes the
state model, traces the output-mode machine, investigates the EDID *source*, traces format-7
end-to-end, traces the DTS data path, reassesses the experiments, and builds the final gate map
+ decision tree.
**Constraint (strict):** STATIC ONLY. No device access, no playback, no reboot, no patch, no
deploy. Conclusions labelled `VERIFIED BY BINARY` / `STRONG INFERENCE` / `HYPOTHESIS / UNPROVEN`.
Runtime remains deferred. No artifact was modified, overwritten, or deleted.

New raw evidence this pass: `tools/closure_utpa2k.txt`, `tools/closure_mik.txt`,
`tools/closure_setmode.txt` (unique names; prior artifacts untouched).

---

## 0. TL;DR — can the complete DTS passthrough path now be explained statically?

- **Decoder-start gap: CLOSED (VERIFIED).** After Exp B, **no remaining static state in
  `utpa2k.ko` can reject or suppress a DTS decoder request.** The only license-derived gates on
  the start path are `0x440` (missing-mask, read by `SetDecodeSystem`) and `0x4d8` (DTS level,
  read by `SetSystem2`/`HDMI_SetNonpcm`) — both composed by `CheckHashkey` and both forced to the
  "DTS permitted" state by Exp B. No `MDrv_AUTH_IPCheck` / `Get_DTS_License` call exists on the
  start path. `Get_DTS_License` is strictly capability reporting (1 caller → `SYSTEM_Control`).
- **Output-mode gap: the EDID gate is the final writer, NOT overridden later (VERIFIED).**
  `_MI_AOUT_SetHdmiAutoMode` computes the SPDIF sub-mode from the EDID capability bit, stores it
  to the global state (`[0x7c]/[0x80]/[0x6c]/[0x70]`) and calls `MApi_AUDIO_SPDIF_SetMode`
  (external HDMI-TX). No later static writer overrides it in the DTS setup path.
- **A configurable audio profile MECHANISM EXISTS (VERIFIED that it exists).** `persist.vendor.audio.spdif.type=DTS`,
  `spdif.mode=BYPASS`, `hdmi_tx.type=DTS`, `hdmi_tx.mode=BYPASS`, `hdmi_arc.type/mode=DTS/BYPASS`
  are read by `audio.primary.mt5889.so` (`utils_get_digital_output_mode` /
  `utils_ApplyDigitalOutputSetting` → `MI_AOUT_SetDigitalMode`). The prior experimental APMs
  (`apm_iec_patched.so`, `apm_gate_patched.so`) already consume these.
- **Whether that profile solves the problem WITHOUT patching is UNPROVEN (the first remaining
  unknown).** `spdif.type=DTS` is *already set* on the device, yet DTS passthrough fails — which
  strongly suggests the profile does **not** override the EDID gate. But it is also possible the
  profile controls the **presented** EDID the source reads, and the source simply isn't sending
  DTS. **This is the single point runtime must target.**

---

## 1. OBJ 1 — Close the decoder-start gap

### 1.1 `MDrv_AUDIO_OpenDecodeSystem` @ `0x405cc8` (VERIFIED)
The symbol is a **thunk**: `ldr r1,[r1]` (r1 = address-0 global) then `bx r1` — it jumps to a
function pointer stored in `g_AudioVars2[0]` (the registered decode-system HAL callback). The real
body (`0x405d74`) gates on `[g_AudioVars2+0x4c8] >= 4` (MAD/version, i.e. MS12V22-present), **not**
on the DTS license. `0` direct callers via reloc → function-pointer dispatched (consistent with the
thunk). **After Exp B, `0x4d0=4` ⇒ MS12V22 ⇒ `0x4c8>=4` is satisfied**, so this gate does not block.

### 1.2 `MDrv_AUDIO_ApplyHashkey` @ `0x424b90` (VERIFIED)
```
0x424b90: ldr r0,[r0,#0x440] ; read missing-mask
0x424bc0: bl #0x424bd8        ; push mask (r0=0 = "missing")
0x424bc8: ldr r1,[r0,#0x444] ; read avail-mask
0x424bcc: mov r0,#1
0x424bd0: b  #0x424bd8       ; push mask (r0=1 = "avail")
; 0x424bd8 body writes to HW via HAL_AUDIO_AbsWriteMaskReg @0x112Axx and
;   HAL_AUDIO_AbsWriteByte (r6 base = 0x112a82) — the DSP/SE register space.
```
`ApplyHashkey` **only reads `0x440`/`0x444` and forwards them to HW** (called by `DspReboot`). It
**does not write** `0x440`, and therefore **cannot independently suppress DTS** — it propagates
whatever `CheckHashkey` put there. After Exp B the DTS-missing bits in `0x440` are cleared, so the
propagated mask will not suppress. (Note: the HW base touched is `0x112Axx`, a *different*
register block from the secure strap `0x112cf0` — `ApplyHashkey` never touches the strap.)

### 1.3 `MDrv_AUDIO_SetDecodeSystem` @ `0x424e78` (VERIFIED)
- Reads `0x440` (`0x424ed0`) and `0x444` (`0x424ee0`); forwards the masks via the push helper.
- At `0x424f0c` performs `blx r2` where `r2 = [g_AudioVars2+0]` — a function pointer (the
  registered "set decode system" HAL callback). **This is the dispatch that ultimately sends the
  decode-system / decoder-start command to R2** (via the registered callback; the target itself is
  a runtime-registered pointer, not statically resolvable, but it is the documented Utopia2K
  decode-system hook).
- No `MDrv_AUTH_IPCheck` / `Get_DTS_License` anywhere in the function.

### 1.4 `HAL_AUDIO_SetSystem2` @ `0x44d0b0` (VERIFIED)
```
0x44d144: ldr r1,[r0,#0x4d0] ; 0x4d0 = DSP tier
0x44d148: cmp r1,#4          ; MS12V22?
0x44d1dc: ldr r0,[r0,#0x4d8] ; 0x4d8 = DTS level
0x44d1e4: cmp r0,#3
0x44d1ec: movweq r1,#0x97    ; if 0x4d8==3 (DTS:X) → decoder byte 0x97 else base 4
0x44d1f0: bl #... -> HAL_AUDIO_AbsWriteByte   ; R2 DSP command interface
```
Call targets in `SetSystem2`: `MDrv_AUDIO_SHM_Init`, `HAL_AUDIO_AbsWriteByte`, `printk` — **no
second license/IPCheck**. `0x4d8` directly selects the R2 DSP decoder identification byte. After
Exp B `0x4d8=3` ⇒ decoder byte `0x97` ⇒ DTS decoder instantiated.

### 1.5 `MDrv_AUDIO_Get_Decoder_Support` / `Get_DTS_License` (VERIFIED)
`Get_DTS_License` has exactly **one** caller (`Get_Decoder_Support` → `SYSTEM_Control`, a capability
ioctl). It is **strictly capability reporting**. It is **never** invoked on the start path
(`SetDecodeSystem`/`OpenDecodeSystem`/`SetSystem2` do not call it). Therefore `Get_DTS_License`
can affect decoder creation **only indirectly**, via the framework's decision to *request* DTS — and
only if the framework consults it. Since it ANDs the AUTH bitmap with the secure strap `0x112cf0[7]`,
a `0` strap bit would make it return "not licensed" and could lead the framework to decline the DTS
session (HYPOTHESIS whether the framework actually consults it — see §8).

### 1.6 CodecType = 9 sufficient? (VERIFIED for kernel; precondition satisfied by Exp B)
`CodecType=9` (DTS) selects the DTS decoder path, but the start path additionally requires the
license-derived state to permit it: `0x440` (missing-mask, read by `SetDecodeSystem`) and `0x4d8`
(read by `SetSystem2`). After Exp B both are in the "permit" state, so **CodecType=9 + Exp B state
⇒ DTS selected**; no other static prerequisite remains in `utpa2k.ko`.

### 1.7 Is there a second license/capability check between framework request and R2 start?
**No (VERIFIED).** The start path (`SetDecodeSystem → blx r2` callback; `OpenDecodeSystem → 0x4c8`
version gate; `SetSystem2 → 0x4d8 → AbsWriteByte`) contains no `MDrv_AUTH_IPCheck`, no
`Get_DTS_License`, no `0x4d8==0` rejection beyond what `CheckHashkey` already set. The only
license-derived gates are `0x440`/`0x4d8`, both bypassed by Exp B.

### 1.8 Distinctions (per instruction)
- **capability reporting**: `Get_DTS_License`, `Get_Decoder_Support` (ioctl query).
- **decoder selection**: `SetDecodeSystem` reads `0x440/0x444`; `SetSystem2` reads `0x4d8`.
- **decoder initialization**: `OpenDecodeSystem` (version gate `0x4c8`); `blx r2` callback.
- **decoder execution**: R2 DTS ADEC (DSP program selected by `SetSystem2`'s byte).
- **output formatting**: `SPDIF_AutoMode`/`BypassMode`/`TranscodeMode`/`DigitalTx_ApplySetting`
  (consume `0x582`), `SetHdmiAutoMode`, `HAL_AUDIO_HDMI_SetNonpcm`.

---

## 2. OBJ 2 — `0x440 / 0x444 / 0x4d8 / 0x582` state model

| Field | Init | Write(s) | Read(s) | Value range | Consumer subsystem | Affects |
|-------|------|----------|---------|--------------|--------------------|---------|
| `0x440` missing-mask | `ResetDefaultVars` | **only `CheckHashkey`** (≈40 sites; e.g. `0x423494`…`0x424878`) | `SetDecodeSystem`(`0x424ed0`), `ApplyHashkey`(`0x424bb8`), `OpenDecodeSystem` region, `Debug_Cmd_Read` | bitmask; DTS-missing bits `0x8/0x80/0x20000` | **decoder SELECTION gate** | capability + selection |
| `0x444` avail-mask | `ResetDefaultVars` | **only `CheckHashkey`** | `SetDecodeSystem`(`0x424ee0`), `ApplyHashkey`(`0x424bc8`) | complement of `0x440` | decoder availability | capability + selection |
| `0x4d8` DTS level | `ResetDefaultVars` | **only `CheckHashkey` `0x424878`** (+ `MST_GRule_NR_Sub` in *video* subsystem — not DTS flow; + R2 blob `0x4f1be0` echo) | `SetSystem2`(`0x44d1dc`), `HDMI_SetNonpcm`(`0x44b918`), `GetAudioInfo2`, `SYSTEM_Control`, R2 blobs | 0/1/2/3 (none/core/HD/X) | **DSP decoder SELECTION (byte) + HDMI non-PCM + status** | decoder config |
| `0x582` DTS:X flag | `ResetDefaultVars` | **only `CheckHashkey`** (`0x424e48`,`0x4246f8`) | `SPDIF_AutoMode`/`BypassMode`/`TranscodeMode`/`DigitalTx_ApplySetting`(`0x582`), `GetAudioInfo2` | 0/1 | **OUTPUT FORMATTING** (Bypass vs Transcode for DTS:X) | output mode |

- **Four concepts, one combined machine.** `0x440/0x444` = *capability/availability*
  (decoder-selection gate); `0x4d8` = *which DTS tier decoder to instantiate* (decoder config);
  `0x582` = *DTS:X output-format mode* (output formatting). All four are composed together by
  `CheckHashkey` and consumed at different stages.
- **After Exp B, all four are in the "DTS permitted" state** (`0x440` DTS bits cleared, `0x444`
  DTS bits set, `0x4d8=3`, `0x582=1`).
- **Remaining static state in `utpa2k.ko` that can prevent DTS decoder selection/start: NONE (VERIFIED).**
  The only residual is (a) the secure strap consumed by `Get_DTS_License` — framework/reporting
  only, not in the decode path — and (b) the EDID gate in `mik.ko` — a *separate module*.

---

## 3. OBJ 3 — SPDIF/HDMI output-mode state machine

```
EDID capability bitmap (bit 0x80 = DTS)
   ↓  (_MI_AOUT_ParseEdidAudioDataBlock → _astHdmiInfo cache; read via _MApi_HDMITx_GetEDIDData)
requested digital mode (HAL: persist.vendor.audio.spdif.type/mode → MI_AOUT_SetDigitalMode)
   ↓
_MI_AOUT_SetHdmiAutoMode @0x989cc   (mode index r1; EDID bitmask r0)
   → selects (r5 = HDMI digital type, r6 = SPDIF sub-mode)
   → r6==3→r4=3 ; r6==1→r4=2 ; r6==0→r4=0 (PCM)
   → str r6,[r0,#0x7c]; str r5,[r0,#0x80]; str r6,[r0,#0x6c]; str r5,[r0,#0x70]   (persistent state)
   → bl MApi_AUDIO_SPDIF_SetMode(r4)   (EXTERNAL HDMI-TX function, sym value 0x0 = exported)
   ↓
HAL_AUDIO_HDMI_SetNonpcm @0x44b51c (utpa2k) — reads 0x4d8, configures HDMI non-PCM
   ↓
actual transport mode: SPDIF PCM (r4=0)  OR  compressed/IEC61937 (r4=1/3)
```

- **`MApi_AUDIO_SPDIF_SetMode` is an EXPORTED external** (symbol value `0x0`, defined in the
  HDMI-TX/demux driver, not in `mik.ko`). `SetHdmiAutoMode` is its effective caller.
- **Last writer in the DTS setup path: `_MI_AOUT_SetHdmiAutoMode` (VERIFIED).** It writes the
  global SPDIF/HDMI state (`0x7c/0x80/0x6c/0x70`) and calls `MApi_AUDIO_SPDIF_SetMode`. The
  `AutoMode`/`BypassMode`/`TranscodeMode`/`DigitalTx_ApplySetting` functions are **alternative
  mode handlers** (selected by which is invoked), not sequential post-overrides — they read `0x582`
  and the stored mode but do not overwrite the HW mode unless re-invoked. No later static writer
  that overrides `SetHdmiAutoMode`'s choice was found.
- **Therefore the EDID gate is the final mode decider in the static trace.** If the sink EDID lacks
  DTS bit `0x80`, `r6=0 → r4=0` ⇒ SPDIF forced to PCM. This is a true hard passthrough gate (not
  a soft preference), and it is live in the unmodified deployed `mik.ko` (`c1421040` = host).

---

## 4. OBJ 4 — Can a different EDID profile solve it WITHOUT patching?

### 4.1 Where the sink EDID comes from (VERIFIED source chain)
- Read from the HDMI receiver: `_MApi_HDMITx_GetEDIDData` (`0x38a50`), `_MApi_HDMITx_GetRxAudioFormatFromEDID`
  (`0x24b70`/`0x38a40`), `_MApi_HDMITx_EDID_HDMISupport` (`0x290b4`/`0x38a48`).
- Parsed by `_MI_AOUT_ParseEdidAudioDataBlock` (`0xd332b`) into the cached struct `_astHdmiInfo`
  (`0x39cd0`); monitored/refreshed by `_MI_AOUT_HdmiInfoMonitor` (`0xa8b36`).

### 4.2 A configurable audio profile MECHANISM EXISTS (VERIFIED it exists)
From `props_all.txt` (device runtime properties) and the HAL:
```
persist.vendor.audio.spdif.mode   = BYPASS
persist.vendor.audio.spdif.type   = DTS
persist.vendor.audio.hdmi_tx.mode = BYPASS
persist.vendor.audio.hdmi_tx.type = DTS
persist.vendor.audio.hdmi_arc.mode= BYPASS
persist.vendor.audio.hdmi_arc.type= DTS
```
Consumed by `audio.primary.mt5889.so`: `utils_get_digital_output_mode`,
`utils_ApplyDigitalOutputSetting` → `MI_AOUT_SetDigitalMode`; format string
`spdif mode=ui(%s),current(%s); spdif type=ui(%s),current(%s); digital setting=Mode(%s),Type(%s)`;
strings `SetSpdifOutputType=AC3`, `SetHdmiTxOutputMode=BYPASS`, `SetSpdifOutputMode=PCM`,
`isTranscodeSupported`, `AudioDigitalMonitor`. The prior experimental APMs
(`apm_iec_patched.so`, `apm_gate_patched.so`) already reference `spdif.type`/`hdmi_tx.type`.
Additionally, `MI_DISP_IMPL_XC_HDMIRx_SetEDID` (external) indicates the projector can **SET its own
EDID** (RX emulation / forwarding).

### 4.3 Does the profile solve it? — UNPROVEN (the crux)
- `spdif.type=DTS` is **already set on the device**, yet DTS passthrough still fails. This is strong
  evidence that the property alone does **not** override the EDID gate (→ leans Case B).
- **However**, two EDIDs are in play and static evidence cannot tell which one `_MI_AOUT_SetHdmiAutoMode`
  tests:
  - (a) the **physical downstream sink** EDID (the TV/AVR on HDMI OUT) — a property cannot change
    this; and
  - (b) the **presented/emulated EDID** the projector reports to its HDMI *input* source — which
    `spdif.type=DTS` / `MI_DISP_IMPL_XC_HDMIRx_SetEDID` could plausibly influence, and which
    determines whether the *source* sends DTS in the first place.
- **Static conclusion:** A configuration/profile mechanism exists; it is a credible candidate to make
  the system DTS-capable **if** `spdif.type=DTS` propagates to the DTS capability bit (`0x80`) that
  `SetHdmiAutoMode` consumes (either via the presented EDID or directly). This is **not statically
  provable** — it is the primary runtime question. There is **no evidence** that *no* such mechanism
  exists; the mechanism exists, its efficacy vs the EDID gate is runtime-unverified.
- **Answer to the objective question:** *Could a DTS-advertising EDID be selected through an existing
  configuration/profile mechanism, making the binary patch unnecessary?* — **Statically plausible,
  not confirmed.** The configuration point is `persist.vendor.audio.spdif.type` / `hdmi_tx.type`
  (and possibly the EDID-emulation path). If runtime shows that setting `spdif.type=DTS` populates
  bit `0x80`, the patch is unnecessary.

---

## 5. OBJ 5 — CEA format 7 → DTS-capable SPDIF mode (end-to-end)

```
CEA SAD format code 0x07 (DTS)
  ↓ _MI_AOUT_ParseEdidAudioDataBlock @0xd32b0
  ↓ per-format support structure (≈520 B/record, support byte at +0x66), bounds 0x3f/0xf0
  ↓ capability bitmap  (bit position = CEA format code)
  ↓ bit 0x80  ← CONSUMPTION side VERIFIED (SetHdmiAutoMode tests exactly 0x4/0x80/0x400/0x800/0x1000)
  ↓ _MI_AOUT_SetHdmiAutoMode @0x989cc : tst r0,#0x80 → r5=7, r6=1 (DTS-capable SPDIF sub-mode)
  ↓ MApi_AUDIO_SPDIF_SetMode(compressed)
```
**Missing link (explicit, kept as STRONG INFERENCE, NOT upgraded to verified):** the exact
`orr`/`strb` inside the parser that sets bit `0x80` from format code `0x07` is not isolated — the
`clz/lsr` idiom observed in the dispatch (`0xd3380`) is ambiguous as a bit-index. The *consumption*
side (which bit `SetHdmiAutoMode` tests) is VERIFIED; the *production* side is STRONG INFERENCE.
No further disassembly was forced (per budget guidance).

---

## 6. OBJ 6 — DTS data path after decoder start

```
DTS ADEC (R2, base mst_snd_r2 image contains a DTS decoder INDEPENDENT of MS12V22 — VERIFIED per ms12v22 profile; 0x4d0=4 selects MS12V22 but DTS does not require it)
  ↓ DTS frame parser (R2)
  ↓ DTS SDO / formatter (R2)  — IEC61937 framing generated in the DSP (STRONG INFERENCE)
  ↓ SPDIF/HDMI TX  — HW tap on the decoder output matrix; receives the compressed stream directly from the DSP (STRONG INFERENCE)
```
- **Can DTS decoding succeed while SPDIF remains PCM mode?** **Yes (VERIFIED by mechanism).** The
  R2 DTS decoder can run; if `SetHdmiAutoMode` forced `r4=0` (PCM), the TX reformats the decoder
  output to PCM — this is precisely the EDID-gate failure mode.
- **Second non-EDID transport gate after `SetHdmiAutoMode`?** **None found.** SPDIF TX is a passive
  tap; the only gate is the mode selection (PCM vs compressed) performed by `SetHdmiAutoMode`/the
  HAL `spdif.type` setting.
- Classifications: R2 base-image DTS decoder = VERIFIED; IEC61937 framing in DSP = STRONG INFERENCE;
  TX taps DSP output = STRONG INFERENCE.

---

## 7. OBJ 7 — Reassessment of previous experiments (avoid hindsight)

| Exp | Definitely changed | Definitely did NOT change | Layer | Hypothesis actually tested | Legitimate conclusion from PASS/FAIL |
|-----|--------------------|---------------------------|-------|----------------------------|--------------------------------------|
| **A** (1 NOP @`0x424444`, IPID `0x7d`) | `0x4d0=4` (MS12V22), `0x4d4=8`, `0x43d/0x43e=1` (Dolby premium) | DTS state (`0x4d8`, `0x440` DTS bits) | Dolby / decoder-tier | "Does enabling MS12V22/Dolby fix DTS?" | DTS still FAILED ⇒ the DTS gate is on a different axis than `0x7d`/Dolby. Test was on the wrong axis. |
| **B** (4 NOPs IPID `0x0f/0x3a/0x12/0x07` + Exp A `0x7d`) | `0x4d8=3`, `0x440` DTS bits cleared, `0x582=1`, `0x4d0=4` | secure strap `0x112cf0[7]`; EDID gate (`mik.ko` unmodified); `libmi3` caps (already patched) | AUTH/DTS license (kernel decode gate) + Dolby tier | "Does forcing AUTH DTS PASS enable the DTS decoder to start?" | DTS PASSTHROUGH still FAILED ⇒ the kernel decode gate is bypassed, but a SECOND gate (EDID/output-mode, and/or framework strap-gate) remains. The FAIL does **not** mean Exp B was ineffective at the kernel decode level. |

---

## 8. OBJ 8 — Final static gate map

| Layer | Gate / state | Static condition | Exp B status | Remaining uncertainty |
|-------|--------------|------------------|--------------|----------------------|
| Capability | `libmi3` GetCaps | OR `0x2E1`/`0x070002E1` (device = v2) → DTS offered | already patched | none (VERIFIED) |
| Capability | `Get_DTS_License` (`0x422aa0`) | AUTH bitmap `{0xf,0x3a,0x12,7}` AND strap `0x112cf0[7]` | AUTH forced PASS; **strap untouched** | **framework may consult it** (HYPOTHESIS) — only indirect/reporting influence |
| Decoder state | `CheckHashkey` / `0x440` | missing-mask; DTS bits `0x8/0x80/0x20000` | **cleared** | none (VERIFIED) |
| Decoder state | `0x4d8` (DTS level) | 0=none…3=X | **=3** | none (VERIFIED) |
| Decoder config | `SetSystem2` (`0x44d0b0`) | `0x4d0==4`→jump tbl; `0x4d8==3`→decoder byte `0x97` | satisfied | none (VERIFIED) |
| Decoder init | `OpenDecodeSystem` (`0x405cc8`) | `0x4c8>=4` (version) | satisfied (MS12V22) | none (VERIFIED) |
| Decoder start | `SetDecodeSystem`/`OpenDecodeSystem` (`blx r2` callback) | function-pointer dispatch; no license check | open | none (VERIFIED) |
| EDID | DTS capability bit `0x80` | set iff sink/presented EDID advertises DTS (0x07) | **not changed** (EDID gate live) | **which EDID is read + does `spdif.type` drive bit 0x80?** (HYPOTHESIS) |
| Output mode | `_MI_AOUT_SetHdmiAutoMode` (`0x989cc`) | `tst r0,#0x80` → compressed vs `r4=0` PCM | **not changed** | none on mechanism (VERIFIED it is final writer); input (EDID bit) is the uncertainty |
| HDMI/SPDIF | `MApi_AUDIO_SPDIF_SetMode` / `HDMI_SetNonpcm` | mode arg from above | not changed | none (VERIFIED) |
| Transport | DTS SDO / IEC61937 → SPDIF TX | DSP tap; IEC61937 in DSP | n/a | none found (STRONG INFERENCE) |
| **Profile** | `persist.vendor.audio.spdif.type/mode` (+ `hdmi_tx.*`, `hdmi_arc.*`) | read by HAL `utils_ApplyDigitalOutputSetting` | already `type=DTS` | **does it populate EDID bit 0x80?** (HYPOTHESIS) |

### First remaining unknown in the causal chain
**Does the configurable audio profile (`persist.vendor.audio.spdif.type=DTS`, already set) drive
the DTS capability bit `0x80` that `_MI_AOUT_SetHdmiAutoMode` tests — or is that bit sourced solely
from the physical sink EDID (which a property cannot change)?**

This is the first point where static evidence stops. Everything upstream (license → `0x440/0x4d8`
→ `SetSystem2` → R2 DTS decoder start) is now fully explained and satisfied by Exp B. The chain
breaks (for passthrough) only at the EDID/output-mode stage, and the *single* open question there
is whether the **profile** can satisfy the EDID bit. Runtime must target exactly this.

---

## 9. OBJ 9 — Decision tree before runtime

- **Case A — A standard/configurable DTS-capable EDID profile exists.**
  `spdif.type=DTS` / `hdmi_tx.type=DTS` (and/or `MI_DISP_IMPL_XC_HDMIRx_SetEDID`) can populate the
  DTS capability bit `0x80`. → **Runtime should first test profile selection** (set
  `persist.vendor.audio.spdif.type=DTS` / `hdmi_tx.type=DTS`, toggle `spdif.mode`, reboot-the-audio
  service, re-probe EDID + `MApi_AUDIO_SPDIF_SetMode` argument). If bit `0x80` appears and SPDIF
  leaves PCM, **no binary patch is needed.**
- **Case B — No profile mechanism feeds the EDID bit; the live EDID clearly lacks DTS.**
  → **Runtime should verify the sink EDID content** (`_MApi_HDMITx_GetEDIDData` / `_astHdmiInfo`)
  and the resulting `MApi_AUDIO_SPDIF_SetMode` argument. If bit `0x80` is absent and `r4=0`, the
  physical sink (or the projector's presented EDID) does not advertise DTS; connect a DTS-capable
  AVR or patch `mik.ko`.
- **Case C — EDID advertises DTS, but a static gate remains unproven.**
  → **Runtime should target that specific gate** (e.g., instrument `SetHdmiAutoMode` entry/exit and
  `SetSystem2` decoder-byte write; confirm `0x4d8→0x97` and the HW SPDIF mode).
- **Case D — All static gates appear satisfied but DTS still fails.**
  → **Runtime should instrument decoder startup and actual IEC61937 transport** (dmesg for R2 DTS
  ADEC start, SPDIF TX mode, IEC61937 sync) to find the residual runtime gap.

**Do NOT patch anything yet.** Runtime remains deferred per the static-only constraint.

---

## 10. Evidence index

| Claim | Label | Evidence |
|-------|-------|----------|
| `OpenDecodeSystem` gates on `0x4c8` (version), not license | VERIFIED BY BINARY | `0x405d74` body, `closure_utpa2k.txt` |
| `ApplyHashkey` only reads+forwards `0x440/0x444` to HW (no suppression) | VERIFIED BY BINARY | `0x424b90`, `closure_utpa2k.txt` |
| `SetDecodeSystem` `blx r2` = decode-system dispatch to R2 | VERIFIED BY BINARY | `0x424f0c`, `closure_utpa2k.txt` |
| `SetSystem2` `0x4d8`→decoder byte via `HAL_AUDIO_AbsWriteByte` | VERIFIED BY BINARY | `0x44d1dc`/`0x44d2d8`, `closure_utpa2k.txt` |
| No AUTH/`Get_DTS_License` on start path | VERIFIED BY BINARY | caller scan (`closure_utpa2k.txt`) |
| `Get_DTS_License` = 1 caller, capability reporting only | VERIFIED BY BINARY | prior deep-dive + `closure_utpa2k.txt` |
| `SetHdmiAutoMode` final SPDIF-mode writer (state + `MApi_AUDIO_SPDIF_SetMode`) | VERIFIED BY BINARY | `0x98de8-0x98df4`, `closure_setmode.txt` |
| `MApi_AUDIO_SPDIF_SetMode` is exported external | VERIFIED BY BINARY | sym value `0x0`, `closure_mik.txt` |
| Configurable audio profile exists (`persist.vendor.audio.*`) | VERIFIED BY BINARY | `props_all.txt`, HAL strings, `apm_iec_patched.so` |
| EDID source chain (GetEDIDData→Parse→_astHdmiInfo) | VERIFIED BY BINARY | `closure_mik.txt` symbols |
| `0x440/0x444/0x4d8/0x582` composed by CheckHashkey, 4 distinct concepts | VERIFIED BY BINARY | `deepdive_fields_utpa2k.txt` + this pass |
| DTS data path: R2 base decoder independent of MS12V22 | VERIFIED BY BINARY | `REPORT_ms12v22_profile` (prior) |
| IEC61937 framing in DSP; TX taps DSP output | STRONG INFERENCE | synthesis §2 G7, §3.5 |
| Parser sets bit `0x80` from format `0x07` | STRONG INFERENCE (production side) | `closure_mik.txt` parser region |
| Profile (`spdif.type=DTS`) populates EDID bit `0x80` | HYPOTHESIS / UNPROVEN | no static proof; first runtime target |

**All prior artifacts preserved. No binaries modified, no patches generated or deployed, no device accessed. Runtime remains deferred.**
