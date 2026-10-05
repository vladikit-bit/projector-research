# R6 — DTS post-SetMode(2) differential trace

**Date:** 2026-09-06 (continuation)   ·   **Investigator:** forensic static trace
**Device:** Thundeal TD98 Pro / C50A (MStar MT5889), Kodi 22 → AudioTrack → `audio.primary.mt5889.so` → `libmi3.so` → `libutopia.so` → kernel `mik.ko` + `utpa2k.ko`

---

## 0. Scope, binaries, and method

### 0.1 Deployed binaries (md5 re-verified against R5b — §9a discipline)
| Role | File | md5 | Status |
|---|---|---|---|
| kernel SPDIF/IEC gate (R5-patched) | `runtime_phase5/mik_r5_patched.ko` | `03fc2c0d22b231bc478e574557b85473` | **DEPLOYED (R5b)** |
| Utopia2K DTS-licence (Exp-B) | `kmods/utpa2k_expB_0x4d8_3_dts_license.ko` | `4c5e6fbb10e24abc3b8d2b0490507983` | **DEPLOYED (Exp-B)** |

Stock (NOT deployed, listed only to avoid the prior md5-confusion trap): `kmods/mik.ko`=`c1421040…`, `kmods/utpa2k.ko`=`2fc6e9fc…`.

**No binary, EDID, HDMI/ARC, setting, property, or routing was changed.** R5b runtime state is the reference: Kodi DTS passthrough = YES; AudioTrack DTS RAW/PT = YES; HAL `0x0B000000` = YES; vendor codec id 9 = YES; `MI_AUDIO_Start Codec:9` = YES; DTS parser runs = YES; kernel output-mode gate **forced to compressed** by R5 patch; **Pioneer AVR DTS lock = NO.**

### 0.2 Method — critical correction vs prior R-turns
Prior R-turns disassembled the raw `.ko` directly. In an **unlinked** `.ko` every `bl`/`b` immediate is a **relocation placeholder** (the real callee is in the `.rel.*` section), so `bl #<self>` decodes as a self-loop and **all call targets were wrong**. R6 built a relocation-aware disassembler (`tools/r6_dis_reloc.py`, `tools/r6_reloc.py`) that resolves real callees and MOVW/MOVT symbol loads, and a literal-pool / data-xref scanner (`tools/r6_litxref.py`, `tools/r6_packref.py`, `tools/r6_dataref.py`). **All findings below use relocation-resolved code.** VA = `file_off − 0xd654` (utpa2k) / `− 0x2af4` (mik).

---

## 1. Differential table — AC3 (codec 5) vs DTS (codec 9) after SetMode(2)

| Stage / function | AC3 (5) | DTS (9) | Divergence? |
|---|---|---|---|
| `HAL_AUDIO_SPDIF_SetMode` (mode 2 = compressed) | switches on sub-mode idx `0x1c4`, sets `r5=2`, calls per-mode apply helpers | **identical** | **NO** — codec-agnostic |
| `HAL_AUDIO_SPDIF_SetOutputType` | empty stub `bx lr` | **identical (no-op)** | **NO** — pure stub |
| `HAL_AUDIO_SPDIF_Tx_SetNonPCM` | gate `codec_id ≥ 3` (5 passes) → `HAL_AUDIO_DigitalOut_SetChannelStatus` | gate `codec_id ≥ 3` (**9 passes**) → same `SetChannelStatus` | **NO** — symmetric; NonPCM bit = caller flag `<<6` |
| `HAL_AUDIO_DTSELoadCode` | n/a (AC3) | `codec_id ≥ 4` gate passes → `UtopiaLogSystem` + `printk` (log wrapper only) | n/a — not a gate |
| `HAL_AUDIO_Encoder_Channel_Lock` | called | **called unconditionally** (`r3==4` and `r3≠4` both reach `0x4468ec`) | **NO** |
| `HAL_AUDIO_DigitalTx_ApplySetting` | enable bit idx `5−3=2`; `codec_id ≥ 5` gate passes | enable bit idx `9−3=6`; `codec_id ≥ 5` gate passes | **NO** — symmetric per-codec mask bit |
| `HAL_AUDIO_SPDIF_ApplySetting` | **explicit** `mov #5` / `cmp #5` branches; `is-AC3` SHM param (`r8`); `SetNonPCM` via `codec_id ≥ 3` | **NO `mov #9`/`cmp #9` anywhere**; rides generic `codec_id ≥ 3`/`≥5`; `SetNonPCM` + eARC channel-status still called | **YES (asymmetry)** — see §3 |
| DTS:X capability block (DigitalTx) | sets DTS-core cap bit `0x80` | `0x582`(DTS:X)=1 → `0x80`, guarded by `codec_id ≥ 5` (both pass) | **NO** — advertisement, not blocking |
| DTS SDO packer (`DTSX_CORE2_API_SDO_Packer` etc.) | AC3 has its own packer path | **no resolvable code/data/reloc xref** from codec-9 path | **YES (opaque)** — see §4 |

**First real AC3-vs-DTS divergence:** there is **none in any gate or mode/NonPCM/SetMode function**. The first *asymmetry* appears in `HAL_AUDIO_SPDIF_ApplySetting`, where **AC3 alone receives explicit codec-specific configuration** (`mov #5`, `cmp #5`, the `is-AC3` SHM parameter) while **DTS (9) is handled only by the generic `codec_id ≥ 3` compressed-mode path and never gets a DTS-specific encoder-type / burst-info** (§3). The actual IEC61937 packer selection is opaque to static analysis (§4).

---

## 2. Per-function evidence (relocation-resolved)

### 2.1 `HAL_AUDIO_SPDIF_SetMode` @ `0x448a10` (size `0x250`) — codec-agnostic
Prologue loads `field_0x1c4`, `add r1,r1,#1`, switch (`cmp r1,#6; bhi …`) on a **device sub-mode index**, sets internal `r5=2` for mode 2, then stores `r4` to `0x14`/`0x18`, zeroes `0x570`/`0x573`/`0x4fe`/`0x583`, and dispatches to per-mode apply helpers (`bl #0x448c34` …). **No `cmp`/`sub` against codec id 5 or 9.** → SetMode(2) = "compressed mode", identical for AC3 and DTS. (REPORT_phase5 `SET_MODE` analysis confirmed `r5=2` → compressed.)

### 2.2 `HAL_AUDIO_SPDIF_SetOutputType` @ `0x44a2c4` — empty stub
`bx lr` only (size `0x4`). **No-op for every codec.** It is not a DTS gate.

### 2.3 `HAL_AUDIO_SPDIF_Tx_SetNonPCM` @ `0x44428c` (size `0x10c`) — codec-agnostic, symmetric
```
r6 = g_AudioVars2                         ; global audio object
r4 = r0                                  ; caller NonPCM flag
ldr r0,[r6]; cmp r0,#0; beq …            ; null guard
ldr r0,[r0,#0x4c8]                       ; codec_id field
cmp r0,#3; blo <common>                  ; *** gate: codec_id < 3 (PCM) skips to common path ***
movw r0,=.L.str.6; bl UtopiaLogSystem    ; codec_id >= 3 → log only
cmp r0,#1; beq <printk-block>            ; logging return, converges back
<common>:                                  ; *** BOTH AC3(5) and DTS(9) arrive here ***
  ldr r1,[r0,#0x282]; ldr r2,[r0,#0x286]; ldr r0,[r0,#0x289]
  cmp r4,#0; movwne r4,#1                ; NonPCM flag from caller
  lsl r0,r4,#6                           ; NonPCM enable bit = (flag≠0)<<6
  strb r0,[sp]
  bl HAL_AUDIO_DigitalOut_SetChannelStatus   ; *** real IEC61937/AES channel-status config ***
```
**Conclusion:** the only codec test is `codec_id ≥ 3` — both AC3 and DTS satisfy it; the channel-status config is reached identically. NonPCM enable is driven solely by the caller's flag. **Not a DTS blocker.** It *is* invoked from `HAL_AUDIO_SPDIF_ApplySetting` (`bl @ 0x4470d8`, under `codec_id ≥ 3`, lines 578–596).

### 2.4 `HAL_AUDIO_DTSELoadCode` @ `0x444010` (size `0x54`) — log wrapper, not a gate
```
ldr r0,[g_AudioVars2]; cmp r0,#0; beq ret
ldr r0,[r0,#0x4c8]; cmp r0,#4; poplo ret   ; codec_id < 4 → return
movw r0,=.L.str.2; bl UtopiaLogSystem       ; codec_id ≥ 4 (DTS 9 passes) → log
cmp r0,#1; beq 0x44404c
… ; b printk                                 ; tail-call printk
```
No DTS encoder load, no licence check that blocks. **Not a gate.**

### 2.5 `HAL_AUDIO_Encoder_Channel_Lock` in `SPDIF_ApplySetting` @ `0x4468ec` — called unconditionally
```
0x4468d0: cmp r3,#4; bne 0x4468ec          ; r3≠4 jumps straight to the call
0x4468d8: ldr r1,[r1,#0x4f4]; sub r1,r1,#5 ; r3==4 → compute is-AC3
0x4468e4: clz r1,r1; lsr r8,r1,#5          ; r8 = 1 iff field==5 (AC3)
0x4468ec: bl HAL_AUDIO_Encoder_Channel_Lock ; *** reached on BOTH paths ***
… mov r2,r8 ; bl HAL_SND_R2_Set_SHM_PARAM  ; r8 (is-AC3) feeds SHM param, not the lock gate
```
The `is-AC3` boolean (`r8`) only feeds downstream `HAL_SND_R2_Set_SHM_PARAM` (param ids `0x6e`, `0x9d`); it does **not** gate the channel lock. **Not a DTS blocker.**

### 2.6 `HAL_AUDIO_DigitalTx_ApplySetting` @ `0x446190` (367 insns) — symmetric per-codec mask
`sub r0, r8, #3` computes index `codec_id − 3`; `HAL_AUDIO_AbsWriteMaskByte(…, idx, #0x10)` writes the per-codec enable bit (`0x10` width = 16 codecs). DTS → bit **6**, AC3 → bit **2** — both set. `HAL_DEC_R2_Set_SHM_PARAM(0x9a, …)` and a `codec_id ≥ 5` gate (`cmp r0,#5; blo`) — both AC3 and DTS pass. Callees are only `HAL_DEC_R2_Set_SHM_PARAM` (×20), `HAL_AUDIO_AbsWriteMaskByte` (×12), `MDrv_HDMI_Ctrl`, `HAL_AUDIO_PcmRenderControl`, `printk`/`UtopiaLogSystem`. **No DTS packer, no SetNonPCM, no codec-9-specific branch.**

### 2.7 `HAL_AUDIO_SPDIF_ApplySetting` @ `0x4465xx` (1391 insns) — the asymmetry lives here
- `cmp r0,#3` appears **~30×** as the generic `codec_id ≥ 3` compressed gate — symmetric.
- **AC3-specific:** `sub r1,r1,#5` (`is-AC3`, line 86→`r8`), `mov r0,#5` (lines 764, 921), `mov r0,#0xa` (AAC, lines 740/799/872/875/880), `mov r0,#0xb` (codec 11, line 1283), `cmp r0,#5` (lines 181/196/215/337 in `DigitalTx`). **DTS (9) is never the subject of a `mov`/`cmp`.**
- `HAL_AUDIO_SPDIF_Tx_SetNonPCM` called @ `0x4470d8` under `codec_id ≥ 3` (line 579) — DTS enters; `ldrb r0,[obj,#0xff]` is the NonPCM flag, `HAL_AUDIO_eARC_TxOutputSetChannelStatus` also called. Symmetric.
- `g_u32bDTSCD` (line 1059) only copies a "DTS-CD" config flag into `field 0x24bc` — config propagation, not a gate.
- **Exhaustive CODEC_IMM scan of the audio region** (`SetMode`, `SetNonPCM`, both `ApplySetting`, `DTSELoadCode`, `AutoMode`): codec ids referenced are **3, 5, 10, 11, 12, 13** — **never 9 (DTS).** DTS has no codec-specific configuration anywhere in the output path.

### 2.8 `mik.ko` side-check — `cmp #9` is debug-only
A module-wide scan of the deployed R5-patched `mik.ko` found `cmp #9`/`mov sb,#9` clusters, but function-symbol resolution shows they all live in **`MI_DEV_DEBUG_Util`** and **`MI_DEV_EVENT_NotifyWithReturnCallback`** — i.e. codec-name *logging*, not the SPDIF/IEC output gate. The R5 patch already forces the compressed mode in `_MI_AOUT_SetHdmiAutoMode`; the kernel side does not contain a DTS-specific output-block either.

---

## 3. Where AC3 and DTS first diverge (the answer to R6)

**They do not diverge in any gate, mode, or NonPCM/SetMode function.** The configuration/apply layer (`SetMode`, `SetOutputType`, `SetNonPCM`, `DTSELoadCode`, `Encoder_Channel_Lock`, both `ApplySetting`, the DTS:X cap block, and `mik.ko`'s mode gate) is provably **AC3/DTS-symmetric**, and DTS (9) receives the per-codec enable bit and channel-status config identically to AC3 (5).

The first and only **asymmetry** is that **AC3 receives explicit codec-specific encoder configuration** (`is-AC3` SHM parameter, `cmp #5`/`mov #5` branches, `mov #0xa`/`#0xb` for neighbour codecs) whereas **DTS (codec 9) is handled exclusively by the generic `codec_id ≥ 3` compressed-mode path and is never given a DTS-specific encoder type / IEC61937 burst-info.** Every concrete DTS (9) configuration slot that exists for AC3 (5) is **absent** for DTS in the static output path.

---

## 4. The DTS SDO packer is opaque to static analysis (R4 §3.3)

The packer name strings — `DTSX_CORE2_API_SDO_Packer`, `DTSDecSDOPacker_API_Process`, `Mstar_DTS_Hdmi_Packer`, `DTSX_Transcoder` — **exist** in `utpa2k.ko` (`.rodata` and literal pools), but:
- **No code `bl`/call xref**, **no MOVW/MOVT relocation**, and **no data/registration-table dword** references any of them (verified by `r6_packref`, `r6_litxref`, `r6_dataref`).
- All SDO/SetMode/NonPCM/ApplySetting functions have **0 direct callers** (R4 §3.3) — they are reached only via Utopia function-pointer dispatch tables, and the packer selection is an opaque DSP-firmware / fp-dispatch mechanism.

Therefore the **static reachability of the DTS packer from the codec-9 path cannot be established**: we can prove it is *not blocked* by the configuration layer, but we **cannot prove it is engaged** for DTS. This is the single open link.

---

## 5. Answer to R6's final question

> *After R5 forces DTS into compressed output mode, what exact code/state prevents codec 9 from becoming a valid physical SPDIF/IEC61937 output?*

**The configuration/apply/licence layer does NOT prevent it** — that layer is AC3/DTS-symmetric and DTS is correctly placed into compressed output mode (SetMode 2), gets its channel status set, and gets the per-codec enable bit. The prevention is in the **IEC61937 packer / SPDIF encoder burst-info selection for codec 9**, which is:

1. **never configured by any codec-9-specific code** in `utpa2k.ko` (no `mov #9`/`cmp #9`; only generic `≥3`/`≥5`), whereas AC3 is explicitly configured; and
2. **opaque to static resolution** — the DTS SDO packer is selected via Utopia fp-dispatch / DSP-firmware with no resolvable xref, so we cannot confirm it is actually invoked for codec 9.

**Most likely concrete mechanism:** the SPDIF/IEC61937 encoder (R2 DSP, driven by SHM params from `utpa2k` + the mode gate in `mik.ko`) is told "compressed mode" but is **not told "codec = DTS, burst-info = 0x0B"** through any DTS-specific encoder-type field. AC3 gets that explicit type; DTS does not. The encoder therefore either defaults to AC3 framing (IEC61937 PC/pa = `0x01`) or emits unframed bytes. The Pioneer AVR sees wrong/missing IEC61937 preambles → **no DTS lock**. (Exp-B's licence patch — `0x4d8=3` DTS:X, `0x582=1` — is *necessary but not sufficient*: it lets DTS reach the kernel + parser, but does not supply the DTS encoder type.)

---

## 6. Confidence classification

| Claim | Class |
|---|---|
| SetMode(2)/SetOutputType/SetNonPCM/DTSELoadCode/Encoder_Channel_Lock/ApplySetting are AC3/DTS-symmetric | **PROVEN** (direct relocation-resolved disassembly) |
| No `cmp #9`/`mov #9` DTS-specific handling anywhere in the utpa2k audio output path | **PROVEN** (exhaustive CODEC_IMM scan of all target functions) |
| DTS SDO packer not statically reachable from codec-9 path | **PROVEN** (no code/data/reloc xref to packer strings) |
| DTS burst-info / encoder-type not set for codec 9 → AVR no-lock | **STRONGLY SUPPORTED** (absence of any DTS encoder config + AC3's explicit config + opaque packer) |
| Exact failing register/state (which register holds burst-info, what value DTS shows) | **NOT ESTABLISHED** (requires runtime read of encoder burst-info or DSP-firmware internals) |

---

## 7. Next exact experiment (R7) — *outside R6's no-patch constraint*

**Hypothesis to test:** DTS output fails because the SPDIF/IEC encoder is never given the DTS burst-info (`0x0B`) / encoder type, only "compressed mode".

- **R7-A (read-only, preferred first):** capture the SPDIF/IEC61937 encoder "non-PCM audio type / burst-info" register and the R2 SHM params written in `ApplySetting` (`0x6e`, `0x9d`, `0x9a`, `0x4c`) **during DTS playback**, and diff against AC3 playback. Expected smoking gun: AC3 → burst-info `0x01`; DTS → `0x01` (defaulted to AC3) or `0x00` (unset), never `0x0B`. *Caveat:* per R5 §9e, ftrace/kprobe/kallsyms/devmem and the debug CLI are unavailable → a tiny **read-only** observability method (e.g., a temporary kprobe module that only reads the encoder register) would be required, which is itself a deployable change and thus outside R6's no-patch rule.
- **R7-B (patch experiment):** for codec 9, replicate the AC3-specific encoder-type/burst-info configuration — i.e. set the DTS encoder type / IEC burst-info `0x0B` in the SPDIF/IEC output config (mirror the `mov #5` path) — and observe Pioneer AVR DTS lock. This is the decisive test of the hypothesis and the natural successor to R6.

---

## 8. Appendix — tools produced this turn
- `tools/r6_reloc.py` — relocation-aware ARM call/MOVW-MOVT resolver (real callee symbols).
- `tools/r6_dis_reloc.py` — relocation-aware function disassembler; annotates callees + flags codec-id immediates.
- `tools/r6_packref.py` — lists instructions referencing a symbol substring, mapped to containing function.
- `tools/r6_litxref.py` — PC-relative literal-pool xref (catches `ldr pc` string loads).
- `tools/r6_dataref.py` — data-section dword xref (registration tables).
- `r6_out/*.asm` — relocation-resolved dumps of `HAL_AUDIO_SPDIF_SetMode`, `HAL_AUDIO_SPDIF_Tx_SetNonPCM`, `HAL_AUDIO_DTSELoadCode`, `HAL_AUDIO_SPDIF_AutoMode`, `HAL_AUDIO_SPDIF_ApplySetting`, `HAL_AUDIO_DigitalTx_ApplySetting`.
