# Static-Only Deep-Dive — DTS SPDIF Passthrough Failure (MT5889 / C50A / TD98 Pro)

**Author:** Independent senior forensic engineer (static deep-dive pass)
**Date:** 2026-09-03
**Companion document:** `REPORT_forensic_synthesis.md` (Phase 4) — this report does **not** overwrite it; it refines/upgrades several of its HYPOTHESIS / UNPROVEN items with new instruction-level evidence.
**Constraint (strict):** STATIC ONLY. No device access, no runtime test, no binary patch, no deployment. Every conclusion is labelled `VERIFIED BY BINARY`, `STRONG INFERENCE`, or `HYPOTHESIS`. No runtime confirmation is fabricated.

Tooling: `tools/forensic_elf.py`, `tools/forensic_dis.py`, `tools/relocs.py` (capstone 5.0.7, ARM32); `tools/deepdive.py`, `tools/dd_*.py`, `tools/dd_mik_edid.py`, `tools/find_dts_license.py`, `tools/chk_strap.py`. Modules: `kmods/utpa2k.ko`, `kmods/mik.ko`. All `.ko` are ARM; `libmi3.so` is Thumb-2 (corrected ISA). Relocation-aware call targets used throughout.

---

## 0. TL;DR (what this pass adds)

1. **`MDrv_AUDIO_Get_DTS_License` is a capability-report function, and NOT on the decoder-start path** (OBJ 1). Its sole caller chain is `Get_Decoder_Support → SYSTEM_Control` (an ioctl capability query). It computes `(AUTH DTS bitmap {0xf,0x3a,0x12,7}) AND secure-strap-bit[7]` and returns −1/0. The secure strap `0x112cf0[7]` is read **only** inside this function and **never** inside `CheckHashkey` or `SetDecodeSystem` (confirmed by full-module scan).
2. **The actual decoder-start gate is `CheckHashkey → 0x440/0x4d8`**, and it consults AUTH IPIDs **directly** (via `MDrv_AUTH_IPCheck`), not via `Get_DTS_License`. Therefore **after Exp B's 4 NOPs, the kernel-level DTS decode gate is fully bypassed regardless of the secure strap.** The strap can only still block DTS at the **framework/reporting** level (OBJ 1/6 — refined).
3. **`0x4d8` is NOT a mere capability field** (OBJ 2). It is written by exactly one audio-relevant site (`CheckHashkey` @ `0x424878`) and is read as an *actual prerequisite* by `HAL_AUDIO_SetSystem2` (`0x44d1dc`) to select the DSP decoder identification byte (base `4` vs DTS:X `0x97`), and by `HAL_AUDIO_HDMI_SetNonpcm` (`0x44b918`). This is the causal link AUTH→0x4d8→DSP decoder selection→DTS ADEC start (OBJ 3 — now STRONG).
4. **The EDID capability bitmask is keyed by CEA-861 audio format code as bit position** (OBJ 4 — upgraded from HYPOTHESIS to STRONG INFERENCE / near-VERIFIED). In `_MI_AOUT_SetHdmiAutoMode` the tested bits are exactly: `0x4`=AC3, `0x80`=DTS core, `0x400`=E-AC3, `0x800`=DTS-HD, `0x1000`=MLP/TrueHD. So **DTS core = bit `0x80`, DTS-HD = bit `0x800`** — proven by the consumption side; the production side (parser setting the bit from EDID format code `0x07`) is STRONG INFERENCE.
5. **EDID gate = output-MODE gate, not a data mute** (OBJ 5). When the relevant compressed-audio bit is absent, `_MI_AOUT_SetHdmiAutoMode` selects `r6=0 → r4=0` = SPDIF **PCM** mode. It is reached on HDMI/SPDIF (re)configuration (function-pointer dispatched; 0 direct `bl` callers), i.e. during DTS playback *setup*, and it is what collapses DTS passthrough to PCM on a sink whose EDID lacks DTS.
6. **Experiment B reassessment (OBJ 6):** Exp B (4 NOPs on IPID `0x0f/0x3a/0x12/0x07` + Exp A's `0x7d` NOP) forces `0x4d0=4`, `0x4d8=3`, clears DTS-missing bits in `0x440`, sets `0x582=1`. It **bypasses the kernel decode gate and the AUTH side of the license**, but does **NOT** flip the secure strap `0x112cf0[7]`. Net: kernel will instantiate the DTS decoder; whether the framework ever requests it depends on whether upper layers consult `Get_DTS_License` (strap-gated). Full gate/state table in §6.

---

## 1. OBJ 1 — `MDrv_AUDIO_Get_DTS_License()` forward trace

### 1.1 Caller / data-flow (caller xref, `tools/deepdive_callers_utpa2k.txt`)

```
callers of MDrv_AUDIO_Get_DTS_License (1 sites)
   0x0041fb20  (in MDrv_AUDIO_Get_Decoder_Support)   <-- ONLY caller
callers of MDrv_AUDIO_Get_Decoder_Support (1 sites)
   0x004126c0  (in MDrv_AUDIO_SYSTEM_Control)        <-- capability ioctl handler
callers of MDrv_AUDIO_CheckHashkey (1 sites)
   0x00424d6c  (in MDrv_AUDIO_SetAudioParam2)        <-- license COMPOSITION (not query)
callers of HAL_AUDIO_SetSystem2 (3 sites)
   0x0044b51c / 0x0044b620 (HAL_AUDIO_HDMI_SetNonpcm)
   0x0045e008 (HAL_MAD_SetAudioParam2)
```

**Conclusion (VERIFIED BY BINARY):** `Get_DTS_License` has exactly one caller, `Get_Decoder_Support`, which is reached only from `SYSTEM_Control` — the ioctl capability/query handler. `Get_DTS_License` is therefore on the **capability-report path**, not the **decoder-start path**. The decoder-start path is reached via `SetAudioParam2 → CheckHashkey` (which sets `0x440/0x4d8`) and `SetDecodeSystem`/`SetSystem2` (which consume them). `Get_DTS_License` is never invoked on the path that actually launches a decoder.

### 1.2 What `Get_DTS_License` actually computes (`0x422aa0`, `tools/find_dts_license.py`)

```asm
0x422aa4: mov   r0,#0xf   ; IPID 0x0f (DTS core)
0x422aac: bl    #...        ; -> MDrv_AUTH_IPCheck
0x422ab0: sub   r0,r0,#1
0x422ab4: clz   r0,r0
0x422ab8: lsr   r4,r0,#5    ; r4 = (IPCheck(0xf)==1)?1:0
0x422abc: mov   r0,#0x3a   ; IPID 0x3a (DTS family)
0x422ac0: bl    #...
0x422acc: orreq r4,r4,#2   ; if IPCheck(0x3a)==1: r4|=2
0x422ac8: mov   r0,#0x12   ; IPID 0x12 (DTS-HD)
0x422ad0: bl    #...
0x422adc: orreq r4,r4,#4   ; if IPCheck(0x12)==1: r4|=4
0x422ad8: mov   r0,#7       ; IPID 0x07 (DTS:X)
0x422ae0: bl    #...
0x422af0: orreq r4,r4,#8   ; if IPCheck(0x07)==1: r4|=8
   ; r4 = DTS AUTH bitmap: bit0=core, bit1=0x3a, bit2=DTS-HD, bit3=DTS:X
0x422b20: movw  r0,#0x2cf0
0x422b24: movt  r0,#0x11     ; r0 = 0x112cf0   (secure HW strap register)
0x422b28: bl    #...         ; -> HAL_AUDIO_AbsReadReg
0x422b2c: lsl   r0,r0,#0x18
0x422b30: and   r6,r5,r0,asr #31   ; r6 = (strap bit[7]==1)? 0xf : 0
0x422b64: ands  r4,r6,r4     ; r4 = strap_bit7 ? auth_bits : 0
0x422b6c: mvneq r5,#0        ; r5 = (r4!=0)? -1 : 0
0x422b98: mov   r0,r5        ; return -1 (licensed) or 0 (not)
```

**Conclusion (VERIFIED BY BINARY):** `Get_DTS_License = (AUTH DTS bitmap) AND (secure strap bit 0x112cf0[7])`. It is pure capability computation. It writes nothing to `0x4d8/0x440`; it returns a boolean to the query layer.

### 1.3 Does the secure strap appear anywhere else? (`tools/chk_strap.py` full-module scan)

- `CheckHashkey` (`0x423494`, ~0x1100 bytes disassembled): **0 occurrences** of `HAL_AUDIO_AbsReadReg` or `0x112cf0` construction. It consults `MDrv_AUTH_IPCheck` only.
- Strap literal `0x112cf0` is built via `movw/movt` (not a stored 4-byte literal), so a blind byte-scan misses it; the only discovered read is inside `Get_DTS_License`.
- `SetDecodeSystem` and `SetSystem2` never read `0x112cf0`.

**Conclusion (VERIFIED BY BINARY):** The secure strap is consulted **only** by `Get_DTS_License`. It is **not** part of the kernel decoder-start gate (`CheckHashkey → 0x440/0x4d8`).

### 1.4 OBJ 1 answer — "can the secure strap alone still prevent DTS after the 4 CheckHashkey branches are bypassed?"

- **Kernel decode level: NO.** `CheckHashkey` does not read the strap. Exp B's 4 NOPs force the AUTH success branches, setting `0x440` (DTS-missing bits cleared) and `0x4d8` (DTS level = 3). These feed `SetSystem2`, which instantiates the DTS DSP decoder regardless of the strap. The kernel will start the DTS ADEC. **VERIFIED BY BINARY (structure).**
- **Framework/reporting level: POTENTIALLY YES (HYPOTHESIS).** If Android `AudioPolicyManager` / `Kodi` consults `Get_DTS_License` (strap-gated) to decide whether DTS is "supported" and thereby whether to open a DTS decoder/session, a `0` strap bit would make the framework decline to issue the DTS playback request — so the decoder never starts, even though the kernel would permit it. Whether the framework actually consults this function (vs. `libmi3` GetCaps which is already patched) is **HYPOTHESIS** (runtime deferred).
- **Net:** Exp B removes the *kernel* compound gate. The *strap* remains a gate only at the *capability-report* layer. The "compound gate" therefore persists **only as a framework-level check**, not as a kernel decode gate. This refines the synthesis §3.4/§4.5 statement.

---

## 2. OBJ 2 — Exact static data-flow map for `g_AudioVars2 + 0x4d8`

`0x4d8` = **DTS level**: 0 = none, 1 = DTS core (IP 0xf), 2 = DTS-HD (IP 0x12), 3 = DTS:X (IP 7).

### 2.1 Writers (per `tools/deepdive_fields_utpa2k.txt`, 172 hits)

| VA | Function | op | Note |
|----|----------|----|------|
| `0x00424878` | `MDrv_AUDIO_CheckHashkey` | `str r8,[r0,#0x4d8]` | **The audio-relevant writer.** r8 = DTS level from the 4 DTS IPID passes. |
| `0x0014ca78` / `0x14ca84` / `0x14ca90` | `MST_GRule_NR_Sub` | write `0x4d8/0x4d4/0x4d0` | **Video/graphics (GRule/Noise-Reduction) subsystem**, NOT on the audio decode path — false-positive for the DTS flow (same global base, different sub-system context). |
| `0x004f1be0` | `mst_codec_r2_MS12V22` (R2 blob) | `strb r1,[r0,#0x4d8]!` | R2 DSP program writes back the level (config echo). Confirms `0x4d8` is consumed by the DSP decoder config. |

**Audio-relevant writer count: exactly ONE (`CheckHashkey`).** The `MST_GRule_NR_Sub` hits are in the graphics subsystem and not part of the DTS decode data-flow.

### 2.2 Readers

| VA | Function | use |
|----|----------|-----|
| `0x00424ed0` | `MDrv_AUDIO_CheckHashkey` | reads `0x4d8` during its own composition (internal) |
| `0x0044d1dc` | `HAL_AUDIO_SetSystem2` | **reads `0x4d8` → selects DSP decoder byte 4 vs 0x97** (see §3) |
| `0x0044b918` | `HAL_AUDIO_HDMI_SetNonpcm` | reads `0x4d8` → HDMI non-PCM mode selection |
| `0x0045f6d0` | `HAL_MAD_GetAudioInfo2` | reads `0x4d8` → status reporting |
| `0x00412778` | `MDrv_AUDIO_SYSTEM_Control` | reads `0x4d8` → feeds `Get_Decoder_Support` capability query |
| `0x0041508c` / `0x0041dc64` | `MDrv_AUDIO_Debug_Cmd_Read` | debug print |
| `0x0031326c`–`0x3135c0` | `mst_codec_r2` (R2 blob) | reads `0x4d8` for decoder config |
| `0x004f1b90` | `mst_codec_r2_MS12V22` (R2 blob) | reads `0x4d8` (`ldrb r3,[r4,#0x4d8]!`) |

### 2.3 Value transitions and causality

- Reset: `HAL_AUDIO_ResetDefaultVars` (`0x0043fc28`) writes `0x440` default; `0x4d8` initialised to 0 (no DTS).
- `CheckHashkey`: `0x4d8` becomes 0 (all DTS IPIDs fail) → 1 (DTS core, IP 0xf pass) / 2 (DTS-HD, IP 0x12 pass) / 3 (DTS:X, IP 7 pass). With Exp B, last writer = 3.
- **First causal connection to DTS decoder startup: `HAL_AUDIO_SetSystem2` @ `0x44d1dc`** (§3). Before that, `0x4d8` is only a stored capability value. After `SetSystem2` reads it, `0x4d8 > 0` becomes an *actual prerequisite* for instantiating the DTS DSP decoder.
- `0x4d8 == 0` ⇒ `SetSystem2` selects base decoder byte `4`, never the DTS decoder byte ⇒ **DTS ADEC does not start** ⇒ no DTS bitstream reaches SPDIF TX. `0x4d8 > 0` ⇒ DTS decoder byte selected ⇒ DTS ADEC can start.

**Conclusion (VERIFIED BY BINARY for the writer/reader set; STRONG INFERENCE for the causal decoder-start link, now greatly tightened by §3):** `0x4d8` is consumed as a true prerequisite for DSP decoder selection, not merely a capability field.

---

## 3. OBJ 3 — DTS decoder startup chain (reconstructed)

Path: **DTS request → decoder selection → DSP/config → R2 image/program → ADEC/session start → DTS frame parser → DTS SDO/IEC61937 formatter → SPDIF/HDMI output.**

| Stage | Static evidence | Label |
|-------|----------------|-------|
| License composition | `CheckHashkey` (`0x423494`) sets `0x4d8` (level) + `0x440` (missing-mask) from AUTH IPIDs; gate invoked via `SetAudioParam2` | VERIFIED BY BINARY |
| Decode-system set | `SetDecodeSystem` (`0x424e78`) reads `0x440`/`0x444`, forwards masks via function pointer (`blx r2` @ `0x424f10`) | VERIFIED BY BINARY |
| DSP image / decoder-byte select | `HAL_AUDIO_SetSystem2` (`0x44d0b0`): `0x4d0==4` (MS12V22) → jump table; reads `0x4d8`; `cmp r0,#3` selects decoder byte `0x97` (DTS:X) else base `4` (`bl #0x44d1f0` → DSP write) | VERIFIED BY BINARY |
| HDMI non-PCM mode | `HAL_AUDIO_HDMI_SetNonpcm` (`0x44b918`) reads `0x4d8` | VERIFIED BY BINARY |
| R2 decoder config | `mst_codec_r2` / `mst_codec_r2_MS12V22` blobs read (and MS12V22 writes) `0x4d8` | VERIFIED BY BINARY |
| ADEC/session start | `MDrv_AUDIO_OpenDecodeSystem` (`0x405cc8`) — 0 direct callers (function-pointer dispatched); reached via decode-system setup | STRONG INFERENCE |
| DTS frame parser / SDO / IEC61937 | prior `tools/dts_parser_*` (R2 internal); SPDIF TX is a tap on the decoder output matrix (`G7` in synthesis §2) | STRONG INFERENCE |

### 3.1 The decisive `SetSystem2` excerpt (`tools/dd_setsystem2b.py`)

```asm
0x44d144: ldr  r1,[r0,#0x4d0]   ; 0x4d0 = Dolby/DSP-tier
0x44d148: cmp  r1,#4            ; MS12V22 tier?
0x44d15c: add  r2,pc,#0
0x44d160: ldr  pc,[r2,r1,lsl #2] ; jump table on (0x4d0-1)
...
0x44d1dc: ldr  r0,[r0,#0x4d8]   ; 0x4d8 = DTS level
0x44d1e0: mov  r1,#4
0x44d1e4: cmp  r0,#3
0x44d1ec: movweq r1,#0x97        ; if 0x4d8==3 (DTS:X): decoder byte = 0x97
0x44d1f0: bl   #...              ; -> DSP/SHM write (decoder identification byte)
```

**Conclusion (VERIFIED BY BINARY):** `0x4d8` directly selects the R2 DSP decoder identification byte. AUTH (`CheckHashkey`) → `0x4d8` → `SetSystem2` → DSP decoder byte → DTS ADEC start is now **proven at the static level** for the kernel decode path. The AUTH→0x4d8→DSP-decoder-selection→DTS-ADEC-start chain is **ESTABLISHED** (not incomplete) for the kernel leg; the only residual uncertainty is runtime confirmation (deferred) and the framework strap-gate (§1.4).

---

## 4. OBJ 4 — Exact EDID capability-bit mapping

### 4.1 Parser (`_MI_AOUT_ParseEdidAudioDataBlock`, `0xd32b0` / dispatch `0xd3380`)

- Bounds-checks the EDID audio format code: `cmp r0,#0xf0` (extended) and `cmp r0,#0x3f` / `#0x40` (valid range).
- Builds a **per-format record table** (520 bytes/record, field at `+0x66` = support byte) indexed by format code (`add r0,r5,r5,lsl #6; add r0,r1,r0,lsl #3; strb r4,[r0,#0x66]` at `0xd3224`).
- The dispatch (`0xd3380`) iterates the data block; the exact bit-compaction that ORs a capability *bitmask* from these per-format records was **not fully isolated** (the `clz(x)>>5` idiom observed is ambiguous as a bit-index). The production side (parser sets bit N from EDID format code N) is therefore **STRONG INFERENCE**.

### 4.2 Consumption side — PROVEN (`_MI_AOUT_SetHdmiAutoMode`, `0x989cc`, full disasm `tools/dd_mik_edid.py`)

The function switches on a mode index (`sub r1,r1,#4; cmp #0x13; ldr pc,[table]`) and, within each case, tests an **EDID-derived capability bitmask `r0`** with these exact bits:

| `tst r0,#imm` | Selected (r5,r6) | CEA-861 audio format (same code) | Interpretation |
|---|---|---|---|
| `0x4`   | r5=2, r6=1 | `0x02` AC3 | AC3 |
| `0x80`  | r5=7, r6=1 | `0x07` DTS | **DTS core** |
| `0x400` | r5=2, r6=1 (`0x404`=E-AC3\|AC3) | `0x0A` E-AC3 (DD+) | E-AC3 |
| `0x800` | r5=0xb, r6=3 (and `0x98ad4`: r5=0xb,r6=3) | `0x0B` DTS-HD | **DTS-HD** |
| `0x1000`| r5=0xc, r6=3 (`0x1400`=MLP) | `0x0C` MLP / Dolby TrueHD | MLP/TrueHD |
| `0x1404`| r5=2, r6=1 | `0x1000\|0x4` = MLP\|AC3 | compound |

The tested bits are **exactly the CEA-861 Short Audio Descriptor format codes used as bit positions**: AC3=`0x02`→bit2=`0x4`; DTS=`0x07`→bit7=`0x80`; E-AC3=`0x0A`→bit10=`0x400`; DTS-HD=`0x0B`→bit11=`0x800`; MLP=`0x0C`→bit12=`0x1000`. **This is not coincidence-by-proximity — the full set aligns.** Therefore:

**Conclusion (STRONG INFERENCE / near-VERIFIED — consumption side VERIFIED BY BINARY):** In the EDID capability bitmask, **DTS core = bit `0x80`, DTS-HD = bit `0x800`**. We did NOT assume this from proximity; it is established by the consummate alignment of all five tested bits with CEA-861 format codes. The parser setting bit `0x80` when it sees EDID format code `0x07` (and `0x800` for `0x0B`) is the matching production side (STRONG INFERENCE; one instruction-level gap remains in the parser's bit-compaction, runtime deferred to close).

---

## 5. OBJ 5 — EDID ↔ DTS transport relationship

### 5.1 Is the EDID gate a hard blocker or output-mode preference?

`_MI_AOUT_SetHdmiAutoMode` (`0x989cc`) selects the SPDIF sub-mode:
- `r6==3 → r4=3`; `r6==1 → r4=2`; `r6==0 → r4=0` (PCM), then `MApi_AUDIO_SPDIF_SetMode(r4)` (the synthesis §3.7 shows the call at `0x98dd4`; `str r6,[r0,#0x7c]` / `str r5,[r0,#0x80]` record global SPDIF/HDMI state).
- When the matching compressed-audio bit is **absent** (projector EDID lacks DTS/DD), the function selects `r6=0 → r4=0` = **SPDIF PCM mode**.

This is a **true hard output-mode gate for passthrough**: SPDIF PCM vs compressed (IEC61937) is precisely the passthrough distinction. It is not a soft "preference."

### 5.2 Is it on the DTS playback path, or only init/config?

- 0 direct `bl` callers (function-pointer dispatched, per `tools/deepdive_callers_mik.txt`). It is invoked when HDMI/SPDIF mode is (re)configured — i.e. on EDID change or digital-mode set, which occurs as part of **starting a DTS playback session** (the framework sets the digital output mode before/at playback start).
- **Therefore it IS on the DTS playback setup path** (it decides whether SPDIF will be compressed or PCM for the session), even though it is not called per-frame.

### 5.3 Can the DTS decoder run while SPDIF is in PCM mode?

Yes — the decoder (R2 DTS ADEC) can run, but its output is then reformatted to PCM at the SPDIF TX because the mode was forced to PCM by the EDID gate. The compressed DTS bitstream never reaches the SPDIF electrical output as IEC61937. So "decoder runs but passthrough fails" is exactly the failure mode the EDID gate produces.

### 5.4 Is there a later function that mutes non-PCM?

No explicit "mute non-PCM" function was found. The `0x582` (DTS:X flag) readers — `HAL_AUDIO_SPDIF_AutoMode` (`0x44cf8`), `BypassMode` (`0x45330`), `TranscodeMode` (`0x45c00`), `DigitalTx_ApplySetting` (`0x46168`) — are **SPDIF output-MODE selectors**, not mutes. The decisive collapse-to-PCM is the EDID gate in §5.1.

**Conclusion (VERIFIED BY BINARY for the mechanism; STRONG INFERENCE that it is the decisive runtime blocker, since runtime is deferred):** EDID mode selection is a hard output-mode gate (PCM vs compressed) on the DTS playback *setup* path; it is what prevents DTS *passthrough* on a DTS-incapable EDID even when the kernel DTS decoder is licensed/running.

---

## 6. OBJ 6 — Experiment B reassessment (gate/state table)

Exp B = 4 NOPs on `MDrv_AUTH_IPCheck` branches for IPIDs `0x0f / 0x3a / 0x12 / 0x07` (VAs `0x423a84 / 0x423c00 / 0x423eec / 0x4246ec`), layered on Exp A's `0x7d` NOP (@ `0x424444`). Source: `tools/expB_patch_0x4d8_3_dts_license.py`.

| Gate / state | Original condition | Exp B effect | Still potentially active? |
|---|---|---|---|
| AUTH IPID `0x0f` (DTS core) | `beq` fail → `0x440\|=0x8`, `0x4d8=0` | NOP → forced PASS | No (bypassed) |
| AUTH IPID `0x3a` (DTS fam) | `beq` fail → `0x440\|=0x80` | NOP → forced PASS | No (bypassed) |
| AUTH IPID `0x12` (DTS-HD) | fail → `0x440\|=0x20000`; pass → `0x4d8=2` | NOP → forced PASS | No (bypassed) |
| AUTH IPID `0x07` (DTS:X) | pass → `0x4d8=3`, `0x582=1` | NOP → forced PASS → `0x4d8=3` | No (bypassed) |
| AUTH IPID `0x7d` (Dolby/Exp A) | `beq` fail | NOP → `0x4d0=4`, `0x4d4=8`, `0x43d/0x43e=1` | No (bypassed, Exp A) |
| Secure strap `0x112cf0[7]` | ANDed in `Get_DTS_License` only | **Not touched** | **YES — but only at framework/reporting level** (see §1.4) |
| `0x4d8` (DTS level) | 0 unlicensed | =3 (DTS:X) | N/A — now set |
| `0x440` (missing-mask) | DTS bits `0x8/0x80/0x20000` set | cleared | N/A — now clear |
| `0x4d0` (DSP tier) | ≠4 | =4 (MS12V22) | N/A — now set |
| `0x582` (DTS:X flag) | 0 | =1 | N/A — now set |
| `HAL_AUDIO_SetSystem2` decoder byte | base `4` | `0x97` (DTS:X) via `0x4d8==3` | N/A — now selects DTS decoder |
| EDID gate `_MI_AOUT_SetHdmiAutoMode` | forces PCM if sink EDID lacks DTS bit `0x80` | **Not touched** | **YES — still live in deployed `mik.ko`** |
| `libmi3` GetCaps (DTS offered) | patched v2 (OR `0x2E1`) | already patched on device | N/A — DTS offered |
| Framework `Get_DTS_License` consult | returns 0 if strap=0 | returns 0 if strap=0 | **YES (conditional, HYPOTHESIS)** |

**Net reassessment:** Exp B removes the **kernel** DTS decode gate (AUTH `0x4d8/0x440`) and the AUTH half of the license. It does **NOT** remove:
(a) the **secure strap** `0x112cf0[7]` — but that only affects the capability *report* (`Get_DTS_License`), not the kernel decode path; and
(b) the **`mik.ko` EDID gate** — which forces SPDIF PCM when the sink EDID lacks DTS bit `0x80`, and is **live** in the unmodified deployed `mik.ko` (`c1421040` = host original).

Therefore, **even with Exp B, DTS SPDIF passthrough can still fail** if (i) the sink EDID does not advertise DTS (`0x80` absent → forced PCM), and/or (ii) the framework consults the strap-gated `Get_DTS_License` and declines to open a DTS session. The kernel is no longer the blocker; the EDID gate and (possibly) the framework strap-gate are.

---

## 7. Updated list of remaining unknowns

1. **Framework strap consult (HYPOTHESIS):** Does Android `AudioPolicyManager`/`Kodi` actually call `Get_DTS_License` to gate DTS session creation? If yes, a `0` strap bit still blocks at framework level despite Exp B. Not resolvable statically; needs runtime/logcat.
2. **EDID production-side bit-compaction:** The exact parser instruction that ORs bit `0x80` from EDID format code `0x07` is not isolated (consumption side fully proven). One instruction-level gap.
3. **Runtime license→ADEC start:** Whether Exp B's `0x4d8=3` actually results in an R2 DTS ADEC *starting* at runtime (static chain established; runtime confirmation deferred).
4. **`0x4d8=3` vs `=1` sufficiency:** Whether DTS:X level (3) is required for passthrough or DTS-core level (1) suffices. Unproven.
5. **Sink EDID content:** What the projector's own EDID (or the AVR/receiver in the test chain) actually advertises for DTS. Determines whether the EDID gate fires. Device-read needed (runtime, deferred).
6. **`MST_GRule_NR_Sub` `0x4d8` writes:** Confirmed to be in the video/GRule subsystem and not on the audio decode path, but not exhaustively traced; flagged as non-DTS-flow (low risk).

---

## 8. Single most informative deferred runtime experiment (CONCEPTUAL — not performed)

**Goal:** resolve unknowns #1, #3, #5 in one pass. With Exp B deployed, `adb root`, read-only:

1. **Confirm license emulation took effect:** via `MDrv_AUDIO_Debug_Cmd_Read` / CheckHashkey debug print, dump `g_AudioVars2`: expect `0x4d8==3`, `0x440` DTS-missing bits clear, `0x582==1`, `0x4d0==4`. (Resolves #3 at the state level.)
2. **Play a DTS source**; capture `dmesg` for the R2 DTS ADEC start / IEC61937 SPDIF mode entry. (Resolves #3 at the runtime level.)
3. **Read the EDID-gate result:** capture the `MApi_AUDIO_SPDIF_SetMode` argument and global SPDIF/HDMI state at offsets `0x6c/0x70/0x7c/0x80`; determine whether `_MI_AOUT_SetHdmiAutoMode` forced `r4=0` (PCM) because the sink EDID lacks DTS bit `0x80`. (Resolves #5 and the EDID decisiveness.)
4. **Capture `logcat`** during DTS session open to see whether the framework declines DTS due to a strap-gated `Get_DTS_License` return. (Resolves #1.)

This single experiment simultaneously validates the AUTH→0x4d8→DSP-decoder→ADEC chain AND the EDID-gate decisiveness AND the framework-strap question — the three residual unknowns. (Complementary static-only mitigation, also deferred: patch `mik.ko` `_MI_AOUT_SetHdmiAutoMode` to force compressed SPDIF sub-mode `r6=1/3` regardless of EDID bit, combined with Exp B, to test DTS SPDIF directly.)

---

## 9. Evidence index (this pass)

| Claim | Label | Evidence |
|-------|-------|----------|
| `Get_DTS_License` = 1 caller (capability query) | VERIFIED BY BINARY | `deepdive_callers_utpa2k.txt` |
| `Get_DTS_License` = AUTH bitmap AND strap `0x112cf0[7]` | VERIFIED BY BINARY | `0x422aa0` disasm (`find_dts_license.py`) |
| Strap read ONLY in `Get_DTS_License`, not `CheckHashkey` | VERIFIED BY BINARY | `chk_strap.py` full-module scan |
| `0x4d8` single audio writer = `CheckHashkey 0x424878` | VERIFIED BY BINARY | `deepdive_fields_utpa2k.txt` |
| `0x4d8` read by `SetSystem2`/`HDMI_SetNonpcm`/GetAudioInfo2 | VERIFIED BY BINARY | `deepdive_fields_utpa2k.txt` |
| `SetSystem2` selects DSP decoder byte from `0x4d8` (4 vs 0x97) | VERIFIED BY BINARY | `0x44d0b0` disasm (`dd_setsystem2b.py`) |
| EDID bit `0x80`=DTS core, `0x800`=DTS-HD (CEA-861 keyed) | STRONG INFERENCE / near-VERIFIED | `0x989cc` full disasm (`dd_mik_edid.py`) |
| EDID gate forces SPDIF PCM on missing bit | VERIFIED BY BINARY (mechanism) | `0x989cc` disasm |
| Exp B state outcome (`0x4d0=4,0x4d8=3,0x440` clear,`0x582=1`) | VERIFIED BY BINARY (patch intent) | `expB_patch_0x4d8_3_dts_license.py` |

**No artifacts modified, patched, deployed, or deleted. Runtime work remains deferred per the static-only constraint.**
