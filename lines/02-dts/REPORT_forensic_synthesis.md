# Forensic Synthesis — DTS SPDIF Passthrough Failure (MT5889 / C50A / TD98 Pro)

**Author:** Independent senior forensic engineer (continuation pass)
**Date:** 2026-09-03
**Scope:** Independent re-audit of the DTS-vs-AC3 SPDIF passthrough failure.
**Constraint:** Runtime investigation DEFERRED. No binaries patched, no device modified,
no artifacts deleted/overwritten. All conclusions labelled VERIFIED BY BINARY /
VERIFIED BY RUNTIME / STRONG INFERENCE / HYPOTHESIS.

---

## 0. TL;DR

1. The **original** root cause — DTS decode gated behind 4 AUTH IPIDs
   `{0xf, 0x3a, 0x12, 7}` via `MDrv_AUDIO_Get_DTS_License` / `MDrv_AUTH_IPCheck` —
   is **VERIFIED BY BINARY** (not assumed). When unlicensed, `0x4d8=0` and
   `0x440` carries DTS-missing bits `0x8/0x80/0x20000`; the R2 DTS decoder
   program does not start, so no compressed bitstream reaches SPDIF TX.
2. The 4 IPIDs are **genuinely DTS-family** checks (instruction-level confirmed).
   We did **not** take the "DTS license" label on faith.
3. `0x43d`/`0x43e` are **NOT** DTS gates — they are Dolby-premium capability
   flags, written at the final capability assembly, driven by *premium-family*
   IPIDs (incl. `0x7d`), orthogonal to the DTS set. VERIFIED.
4. **NEW (this pass): a SECOND, independent, still-live gate.** `mik.ko`
   `_MI_AOUT_SetHdmiAutoMode` (VA `0x989cc`) is an EDID-capability-driven SPDIF
   mode selector that **forces SPDIF to PCM** when the sink EDID lacks the
   relevant compressed-audio bit. The deployed `mik.ko` is the **unmodified host
   original** (md5 `c1421040`). This means **even a perfect DTS license
   emulation (Exp B) will NOT yield DTS SPDIF passthrough on this projector**,
   because its EDID does not advertise DTS. This refines — and partially
   corrects — the prior "license is the sole root cause" narrative.
5. The **device baseline was NOT clean**: the pre-expA backup `3782c6cf` ≠ host
   original `2fc6e9fc`. Experiments were layered on an already-modified module
   (prior "clean baseline is false" finding, now hash-verified).
6. What remains **UNPROVEN** (runtime deferred): whether Exp B's license
   emulation actually *starts* the R2 DTS ADEC at runtime, and whether the
   EDID gate is the decisive blocker in practice.

---

## 1. Phase 1 — Investigation Chronology (reconstructed, read-only)

| # | Report | Claim / state | Status after this pass |
|---|--------|---------------|------------------------|
| 1 | `REPORT_PASSTHROUGH_FIX` | libmi3 GetCaps patch opens AC3/EAC3/DTS; early "DTS works" later overturned | Caps patch VERIFIED BY BINARY (§3.6) |
| 2 | `REPORT_DTS_forensic` / phase2 / phase3 | DTS fails at `g_AudioVars2[0x43d]` AUTH gate; decoder doesn't start | Partly right; `0x43d` is Dolby flag, not the DTS gate (§3.3) |
| 3 | `REPORT_paths_matrix` | AC3-transcode via Kodi works; DTS RAW fails | Consistent |
| 4 | `REPORT_DTS_transport_solution` / `phase5_spdif_tx_research` | No compressed bitstream reaches SPDIF TX without active DTS ADEC; only DTS SDO packer is inside R2 | STRONG INFERENCE, consistent |
| 5 | `REPORT_full_license_profile` | 27-IPID map; claims `0xb/0xc`=Dolby, `{0xf,0x3a,0x12,7}`=DTS; `0x43d/0x43e`=Dolby-premium | VERIFIED BY BINARY (§3.2–3.4) |
| 6 | `REPORT_ms12v22_profile` | `0x4d0=4` selects MS12V22 images; DTS works on base images too | VERIFIED (§3.5) |
| 7 | `REPORT_solutionA_prepared` | "ПРИГОТОВЛЕНО, НЕ ВСТАНОВЛЕНО" (prepared, not installed) | **CONTRADICTED** by hash: pre-expA device backup is modified (§4.2) |
| 8 | `REPORT_expA_runtime` | Exp A = 1 NOP @`0x424444` (IPID `0x7d`); AC3 PASS / DTS FAIL | VERIFIED (Exp A patches Dolby `0x7d`, not DTS) |
| 9 | `REPORT_T2_breakpoint_analysis` | `_MI_AOUT_SetHdmiAutoMode` forces SPDIF PCM on EDID | VERIFIED BY BINARY (§3.4) |
| 10 | expB logs / summary | Exp B (4 DTS IPID NOPs) deployed; AC3 PASS / DTS FAIL | Deployment VERIFIED BY RUNTIME (logs); see §4.3 inconsistency |

---

## 2. Phase 2 — Independent DTS Path Audit (gates enumerated)

End-to-end path: `Kodi → AudioTrack → AudioFlinger → AudioPolicyManager →
audio.primary.mt5889.so (HAL) → libmi3.so (MI_AUDIO_*) → libutopia.so (Utopia2K)
→ kernel mik.ko / utpa2k.ko / dtv_driver.ko → MStar MAD R2 DSP → HDMI/SPDIF TX`.

Plausible rejection / transcode / mute gates, **tested independently of prior labels**:

- **G1 — Kodi IEC61937 packing.** Kodi 21/22 packs IEC61937; the R2 AC3/DTS
  decoder expects raw frames → sync failure on IEC framing. (Prior; AC3-RAW
  works, Kodi-IEC is silent.) *Not a DTS-specific gate.*
- **G2 — Framework/Policy capability gate.** Does Android offer DTS at all?
  Gated by `libmi3` GetCaps (§3.6) and `audio_policy`. VERIFIED that device
  libmi3 = v2 caps patch (OR `0x000002E1`) → DTS *offered*.
- **G3 — HAL `audio.primary.mt5889.so` digital-mode gate.** Mode selector, not
  data injection. VERIFIED BY PRIOR audit; no data-injection API exists.
- **G4 — DTS AUTH license gate (utpa2k.ko).** `MDrv_AUDIO_Get_DTS_License` /
  `CheckHashkey` / `MDrv_AUTH_IPCheck`. **VERIFIED BY BINARY (§3.1–3.3).** This
  is the *original* gate.
- **G5 — R2 DSP image / decoder-start gate.** `HAL_AUDIO_SetSystem2` selects
  decoder program + DSP byte from `0x4d0`/`0x4d8`. VERIFIED BY BINARY (§3.5).
  Decoder only *runs* if `0x4d8>0` (license granted) **and** a decode program
  is launched.
- **G6 — mik.ko SPDIF/HDMI mode gate (EDID).** `_MI_AOUT_SetHdmiAutoMode`
  forces SPDIF PCM when sink EDID lacks compressed-audio capability. **VERIFIED
  BY BINARY (§3.4).** This is the *second* gate and is **live** in the deployed
  module.
- **G7 — SPDIF TX data tap.** SPDIF TX is a tap on the DSP decoder output
  matrix; no host API injects bitstream. STRONG INFERENCE (prior phase5).

---

## 3. Phase 3 — Targeted Binary Verification (independent disassembly)

Tooling: `tools/forensic_elf.py`, `tools/forensic_dis.py`, `tools/relocs.py`
(fixed: `r_offset` is section-relative, not file offset), `tools/audit_phase3.py`.
capstone 5.0.7; pyelftools; ARM32 manual disassembly with relocation-aware
call targets. All `.ko` are ARM; `libmi3.so` is **Thumb-2** (the earlier
"garbage" decode was the wrong ISA — corrected).

### 3.1 `MDrv_AUTH_IPCheck` @ `0x1390c` — reads ONE bit  ✅ VERIFIED BY BINARY
```
ldrb r2,[r1,#9]            ; r1 = gIpAuthVars (140-byte AUTH table)
sub  r3,r2,#0x10 ; blo …   ; range check on tbl[9] (field size)
mvn  r3,r0,lsr #3          ; r0 = id
add  r2,r2,r3              ; byte = tbl[9] - (id>>3) - 1  (reverse-indexed)
ldrb r1,[r1,#0xa]          ; byte at tbl[10 + byte]
and  r4,r2,r1,lsr r0       ; bit = (byte >> (id&7)) & 1
```
Confirms: license = a **single bit** in the secure AUTH table (ioctl
`0xC08C5506`, 140 bytes). No host crypto. Matches `REPORT_full_license_profile` §1.

### 3.2 The four DTS IPIDs `{0xf, 0x3a, 0x12, 7}` — genuinely DTS-family  ✅ VERIFIED BY BINARY
Re-derived (not assumed) from `CheckHashkey` (`0x423494`) field writes:
- `0xf`  → fail sets `0x440|=0x8` (DTS core missing); pass `0x4d8=1`
- `0x3a` → fail sets `0x440|=0x80` (DTS family missing)
- `0x12` → fail sets `0x440|=0x20000` (DTS-HD missing); pass `0x4d8=2`, clears `0x440&=~0x400`
- `7`    → pass `0x4d8=3` (DTS:X), `0x582=1`, `0x444&=~0x100`

None of these write `0x43d`/`0x43e`/`0x4d0`. **They are DTS-capability checks,
confirmed at the instruction level** — answering the explicit instruction not to
assume the "DTS license" label.

### 3.3 `0x43d` / `0x43e` — Dolby-premium flags, NOT DTS gates  ✅ VERIFIED BY BINARY
Final capability assembly in `CheckHashkey` (`0x424870`–`0x424880`):
```
str  r6,[r0,#0x4d0]   ; Dolby/DSP-tier
str  r5,[r0,#0x4d4]   ; sub-tier
str  r8,[r0,#0x4d8]   ; DTS level   (written by {0xf,0x3a,0x12,7})
strb r7,[r0,#0x43e]   ; Dolby-premium "active"
strb sl,[r0,#0x43d]   ; Dolby-premium "top active"
```
`0x43d`/`0x43e` are written from `r7`/`sl`, which are set by *premium-family*
IPIDs (e.g. `0x50`, `0x52/0x75`, `0x51/0x74`, `0x54`, `8`, `0x66`, `9`, `0xa`,
`0x7d`) — **orthogonal** to the DTS set. Consistent with
`REPORT_full_license_profile` §3. They are status/reporting flags, not the
decoder-start gate (which reads `0x4d8`, not `0x43d/0x43e`).

### 3.4 `MDrv_AUDIO_Get_DTS_License` @ `0x422aa0` — AUTH bits AND secure strap  ✅ VERIFIED BY BINARY
```
mov r0,#0xf  ; bl MDrv_AUTH_IPCheck
mov r0,#0x3a ; bl …
mov r0,#0x12 ; bl …
mov r0,#7    ; bl …
   → r4 = bit0(DTS core)|bit1(0x3a)|bit2(0x12)|bit3(0x7)
ldr r0,[r7]  ; g_AudioVars2
ldr r0,[r0,#0x4c8] ; version >=4 (MS12V22 path) check
movw r0,#0x2cf0 ; movt #0x11 → r0 = 0x112cf0  (HW strap register)
bl HAL_AUDIO_AbsReadReg
and  r6,r5,r0,asr #31   ; extract strap bit (bit7 = DTS strap)
ands r4,r6,r4           ; license AND strap → final
```
**Confirms:** DTS licensing = AUTH IPID bits **AND** a secure hardware strap at
`0x112cf0[7]`. The same trailing helpers also reveal more strap bits
(`0x112cf0` bit10 → IPID `0x41`/DRA, bit11 → IPID `0x1e`/WMA). The DTS gate is a
compound of (secure-fuse AUTH) ∩ (hardware strap) — emulating only the AUTH
side (Exp B) leaves the strap AND at 0 unless the strap bit is also set.

### 3.5 `HAL_AUDIO_SetSystem2` @ `0x44d0b0` — DSP image / decoder byte select  ✅ VERIFIED BY BINARY
```
ldr r1,[r0,#0x4d0] ; cmp r1,#4          ; MS12V22 tier select
...
ldr r0,[r0,#0x4d8] ; cmp r0,#3          ; DTS level
mov r1,#4  ; movweq r1,#0x97            ; base=4, MS12V22=0x97
bl <dsp-write>                        ; pushes byte to R2 via SE-IDMA
```
`0x4d0==4` selects MS12V22 R2 images; `0x4d8` (DTS level) selects the decoder
byte. (Prior report cited a `0x43d`-branch @`0x44d2c0` writing byte 5/0xd — a
complementary sub-case; exact byte value is minor INFERENCE, structure
confirmed.) DTS decoder machinery exists in the **base** `mst_snd_r2` image
(per `REPORT_ms12v22_profile`), so DTS does not require MS12V22.

### 3.6 `libmi3.so` caps patch @ VA `0x6263e` (Thumb-2)  ✅ VERIFIED BY BINARY
The patch site is the `GetCaps` capability-word builder:
- **base** `libmi3.so`: calls the caps-populate helper with `r1=0` → DTS bits **not** set.
- **device** `libmi3.so.device` = `libmi3_patched_v2.so` (md5 `d71a330f`):
  `ldr r1,[r5,#-4]; movw r2,#0x2e1; orrs r1,r2; str r1,[r5,#-4]` → ORs **`0x000002E1`**.
- `libmi3_dts_v3.so`: `movw r2,#0x2e1; movt r2,#0x700; orrs` → ORs **`0x070002E1`**.

This **verifies the prior "v3 OR `0x070002E1`" claim** (it is split across
`movw`/`movt`, which is why a contiguous-byte scan missed it). The caps patch
only makes DTS *selectable* in the framework — it does not by itself start the
DTS decoder.

### 3.7 mik.ko `_MI_AOUT_SetHdmiAutoMode` @ `0x989cc` — EDID gate (T2)  ✅ VERIFIED BY BINARY
19-way switch on requested HDMI mode (`sub r1,r1,#4; cmp #0x13; ldr pc,[table]`),
driven by an **EDID-derived capability bitmask `r0`**:
```
tst r0,#0x800 ; bne …     ; compressed-audio capability bits
tst r0,#0x80  ; bne …
tst r0,#0x1000; bne …
tst r0,#0x404 ; …
   → selects (r5,r6) = (HDMI digital type, SPDIF sub-mode)
   → r6==3 → r4=3 ; r6==1 → r4=2 ; r6==0 → r4=0 (PCM)
bl MApi_AUDIO_SPDIF_SetMode(r4)        ; @0x98dd4
str r6,[r0,#0x7c]; str r5,[r0,#0x80]   ; global SPDIF/HDMI state
```
When the matching capability bit is absent (projector EDID lacks DTS/DD), the
function selects `r6=0 → r4=0` → **SPDIF PCM mode**. This is the T2 breakpoint:
SPDIF non-PCM passthrough is gated on **EDID**, independent of the AUTH license.
**The deployed `mik.ko.device` (md5 `c1421040`) is the unmodified host original,
so this gate is LIVE.** (Exact mapping of which EDID bit = DTS specifically is
HYPOTHESIS — needs `_MI_AOUT_ParseEdidAudioDataBlock` @`0xd332b`; the gate
*mechanism* is proven.)

---

## 4. Phase 4 — Forensic Synthesis

### 4.1 Proven (VERIFIED BY BINARY)
- DTS decode is gated on 4 AUTH IPIDs `{0xf,0x3a,0x12,7}` + secure strap `0x112cf0[7]`
  (`MDrv_AUDIO_Get_DTS_License`). Unlicensed → `0x4d8=0`, DTS-missing bits set,
  R2 DTS ADEC does not start.
- The 4 IPIDs are DTS-family (instruction-level confirmed). `0x43d/0x43e` are
  Dolby-premium flags, not DTS gates.
- `HAL_AUDIO_SetSystem2` gates R2 image/decoder selection on `0x4d0`/`0x4d8`.
- `mik.ko` forces SPDIF PCM via an EDID-capability switch; gate is live in the
  deployed (unmodified) module.
- `libmi3` device build = v2 caps patch (OR `0x000002E1`); v3 = OR `0x070002E1`.

### 4.2 Proven (VERIFIED BY RUNTIME / HASH)
- Device **baseline was modified**: pre-expA backup `3782c6cf` ≠ host original
  `2fc6e9fc`. The prior "clean baseline" narrative is false (CONFIRMS the
  contradiction flag). Whether the 33-byte delta is exactly "Solution A" is
  STRONG INFERENCE (not re-diffed this pass).
- Device pre-expB backup = expA module `939cac24` → Exp A was deployed.
- `mik.ko.device` = host original `c1421040` → EDID gate unmodified.
- `libmi3.so.device` = v2 caps patch `d71a330f`.
- Exp B module `4c5e6fbb` **reported deployed** per runtime logs (summary). ⚠️
  **Doc inconsistency:** `REPORT_expA_runtime.md` B.7 marks Exp B "NOT deployed
  yet". Treated as: deployed per logs, but the formal report predates deployment.

### 4.3 What prior agents got RIGHT
- Primary/original root cause = DTS AUTH license gate (now binary-verified).
- The 4 IPIDs are DTS-family; `0x43d/0x43e` are Dolby-premium (binary-verified).
- Architecture couples SPDIF non-PCM to decoder synchronization (STRONG
  INFERENCE, consistent with all gates found).
- Exp A/B static NOP sites are correct (binary-verified).

### 4.4 What prior agents got WRONG / INCOMPLETE
- **Incomplete root-cause narrative.** The investigation framed the AUTH
  license as *the* (sole) root cause and expected "fix the license → DTS
  passthrough works." Independent verification shows a **second, independent,
  still-live gate** (`mik.ko` EDID switch) that forces SPDIF PCM regardless of
  license. On a projector whose EDID lacks DTS, **Exp B alone cannot enable DTS
  SPDIF passthrough** — the EDID gate must also be addressed (or the sink EDID
  must advertise DTS). This is the most likely reason DTS still FAILS after Exp B.
- **"Clean baseline" contradiction** (§4.2) — experiments layered on a
  pre-patched module; affects reproducibility/attribution, not the DTS analysis.
- **Doc inconsistency** on Exp B deployment status (§4.2).

### 4.5 First genuinely unproven failure point (refined)
After the (now bypassable) license gate, the **next live gate is the `mik.ko`
EDID switch** (`_MI_AOUT_SetHdmiAutoMode`) that forces SPDIF PCM. This is the
**current decisive blocker** (VERIFIED as a gate; its runtime decisiveness is
HIGHLY LIKELY but formally UNPROVEN because runtime is deferred).

### 4.6 Open / HYPOTHESIS (runtime deferred)
- Whether Exp B's license emulation actually *starts* the R2 DTS ADEC at runtime
  (license→decoder-start link). Unproven.
- Exact EDID bit → DTS format mapping inside `_MI_AOUT_ParseEdidAudioDataBlock`.
- Whether license-emulation + DTS-capable EDID (or patched `mik.ko`) yields
  working DTS SPDIF. Unproven.
- Whether the `0x4d8=3` (DTS:X) level from Exp B is required, or `0x4d8=1`
  (DTS core) suffices for passthrough. Unproven.

### 4.7 Single highest-information next experiment (CONCEPTUAL — not performed)
With Exp B deployed, at runtime (adb root, read-only):
1. Dump `g_AudioVars2` via `MDrv_AUDIO_Debug_Cmd_Read` / CheckHashkey debug
   print (`dbg≥3`): confirm `0x4d8==3`, `0x440` DTS bits clear, `0x582==1`
   (license emulation took effect).
2. Play a DTS source; capture `dmesg` for R2 DTS ADEC start / IEC61937 SPDIF
   mode entry.
3. Read the EDID-gate result: `MApi_AUDIO_SPDIF_SetMode` argument and global
   state at offsets `0x6c/0x70/0x7c/0x80` — does SPDIF get forced to PCM by EDID?

This single experiment resolves both remaining unknowns (license→decoder link
AND EDID-gate decisiveness). Complementary: patch `mik.ko` `_MI_AOUT_SetHdmiAutoMode`
to force the compressed SPDIF sub-mode (`r6=1`/`3`) regardless of EDID bit,
combined with Exp B, to test DTS SPDIF directly (also deferred).

---

## 5. Evidence Index
| Claim | Label | Evidence (this pass) |
|-------|-------|----------------------|
| IPCheck reads 1 bit | VERIFIED BY BINARY | `0x1390c` disasm (§3.1) |
| 4 IPIDs = DTS | VERIFIED BY BINARY | `CheckHashkey` field writes (§3.2) |
| `0x43d/0x43e` = Dolby | VERIFIED BY BINARY | `0x424870-80` assembly (§3.3) |
| `Get_DTS_License` = AUTH∩strap | VERIFIED BY BINARY | `0x422aa0` disasm (§3.4) |
| `SetSystem2` image/byte sel | VERIFIED BY BINARY | `0x44d0b0` disasm (§3.5) |
| libmi3 caps OR `0x2E1`/`0x070002E1` | VERIFIED BY BINARY | `0x6263e` Thumb disasm + md5 (§3.6) |
| mik.ko EDID forces PCM | VERIFIED BY BINARY | `0x989cc` disasm (§3.7) |
| Device baseline modified | VERIFIED BY RUNTIME | md5 `3782c6cf`≠`2fc6e9fc` (§4.2) |
| mik.ko unmodified in deploy | VERIFIED BY RUNTIME | md5 `c1421040` = host (§4.2) |
| SPDIF TX = decoder tap | STRONG INFERENCE | prior phase5 (§2 G7) |
| Exp B deployed | VERIFIED BY RUNTIME (logs) + doc inconsistency | §4.2 |

**No artifacts were modified, deployed, or deleted. Runtime work remains deferred.**
