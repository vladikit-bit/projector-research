# REPORT_runtime_phase5 — DTS Output-Gate Validation → Minimal Experimental Unlock

**Device:** Thundeal TD98 Pro / C50A (MStar MT5889, MAD R2 DSP)
**Stack:** Kodi 22 (net.kodinerds.maven.kodi22) → AudioTrack → AudioFlinger → `audio.primary.mt5889.so` → `libmi3.so` → `libutopia.so` (Utopia2K fp-dispatch) → kernel `mik.ko` (ARM A32, EDID/SPDIF auto-mode gate) + `utpa2k.ko` (ARM A32, DTS licence/AUTH).
**Date of investigation:** 2026-09-05 (captures), report written post-phase-D.
**Author:** forensic session (Agent mode), per the R5 authorized brief.

---

## 0. Objective & method (recap of the brief)

Determine whether the remaining DTS passthrough failure is explained by:

1. DTS **licence / AUTH** state, and/or
2. **EDID-gated digital-output mode selection** (the kernel `mik.ko` choosing *PCM* vs *compressed/non-PCM* for the SPDIF/HDMI path based on the sink EDID capability mask).

Then, **only if** the evidence is strong, perform the *smallest possible* experimental change and verify it — without deploying until authorized.

Method order mandated by the brief: **runtime → static → minimal differential experiment**. Full static xref was explicitly deprioritized because Utopia2K dispatch blocks call-graph recovery at the HAL layer.

**Constraint compliance (R5):**
- Phase A forbade any change to EDID / HDMI / ARC topology / properties / settings / routing / state-words / binaries, and forbade `dumpsys` inside playback windows. ✓ Honored.
- Phase D forbade: touching AUTH/EDID/HDMI/AC3/E-AC3/DTS-HD/logdecoder; and **deploying** the patch without verification + authorization. ✓ Honored — patch built locally, **not deployed**.
- Device restored to baseline at end of session. ✓ (see §6).

---

## 1. Phase A — Read-only runtime proof: **BLOCKED**

**Goal:** instrument `_MI_AOUT_SetHdmiAutoMode` @0x989cc, `MApi_AUDIO_SPDIF_SetMode`, `HAL_AUDIO_SPDIF_Tx_SetNonPCM`, `HAL_AUDIO_SPDIF_SetOutputType` during AC3 vs DTS playback windows; capture EDID capability mask, selected mode, non-PCM decision, TX path.

**Every non-invasive observation channel was exhausted and found unavailable:**

| Channel | Result | Why unusable for Phase A |
|---|---|---|
| `CONFIG_KPROBES` | not set | no kprobe/uprobe |
| `CONFIG_FTRACE` | not set | `available_tracers` = `nop` only |
| tracefs `kprobe_events` | absent | tracefs mounted but no event writes |
| `/proc/kallsyms` | masked to `00000000` | `kptr_restrict=2` |
| `/dev/kmem`, `/dev/mem`, `/dev/kcore` | absent | `CONFIG_DEVMEM`/`CONFIG_DEVKMEM` not set |
| `mik.ko` debug params (`mi_aout_amp_debuglevel`, `misck/api_audio_amp_debuglevel`, …) | writable as root only | they gate the **AMP** submodule only — *not* the SPDIF/EDID mode selector |
| `/sys/kernel/mik/MI_AUDIO` debug CLI | bails with `0x9` | requires a decoder handle named `MI_DEMO_LIVE_PATH_0`; Kodi does **not** register one (same mechanism as R3). Error: `_MI_DEBUG_AUDIO_GetHandle(178) failed … E_MI_AUDIO_DECODER_MAIN! u32ErrCode:0x9` |
| `/sys/kernel/mik/MI_AOUT` debug CLI (`edid`, `PrintAudioConfig`, `PrintGetCaps`, `SPDIF`, `ARC`) | **silent** | no output at idle *or* during AC3 playback |
| Live sink EDID capability mask | not exposed | MStar HDMI-TX driver; only the projector's own `EDID_BIN/*` is present on device, not the AVR/downstream sink mask the gate reads |

**Conclusion:** The runtime EDID capability mask and the live `MApi_AUDIO_SPDIF_SetMode(arg)` value **cannot be observed on this device without modifying code** (e.g. teaching the debug CLI to accept Kodi's handle, or adding prints) — which Phase A forbids. The AC3 playback capture `R5a_20260905_092219_ac3.{dmesg,logcat}` is retained as evidence that the MI_AUDIO CLI cannot be used as an observation channel. **Phase A therefore yields no direct runtime measurement.**

> This is an honest "not established" result, not a failure of analysis. It reframes the investigation onto its strong leg: static binary proof of the gate mechanism.

---

## 2. Phase B — DTS licence / AUTH reconciliation: **CONFIRMED INSUFFICIENT**

**Goal:** reconcile whether the deployed DTS licence/AUTH state alone explains the failure.

**Established facts (R1–R4, re-confirmed):**
- On-device `utpa2k.ko` md5 = `4c5e6fbb…` = **Exp B**.
- Exp B state words: `0x4d0 = 4`, `0x4d8 = 3`, `0x582 = 1`, DTS **AUTH bits at 0x440/0x444 cleared**.
- Under Exp B, the DTS parser **runs** (R2 static proof) and the decoder **opens** (R3/R4), yet DTS still fails while **AC3 passes**.

**Logical deduction:** Exp B presents a *fully emulated* DTS licence (DTS-HD capability bit set at 0x582, support tables populated). If licence/AUTH were the blocker, AC3 (which depends on the *same* licence/AUTH path for compressed transport) would also fail. AC3 passes. Therefore **DTS licence/AUTH alone is insufficient** to explain the DTS-specific failure. No contradiction found between the licence state and the observed AC3-pass/DTS-fail asymmetry.

**Result:** DTS licence/AUTH is *not* the gate. The failure is downstream of licence — in the SPDIF/HDMI output-mode decision, which is exactly what Phase C confirms.

---

## 3. Phase C — Static confirmation of the EDID → output-mode gate: **DECISIVE**

**Method:** full ARM (A32) disassembly of the deployed `mik.ko` (`libs/mik_dts_hdmienable.ko`, md5 `647b08db…`) over the gate region `VA 0x98684 .. 0x9a7dc`, using the project `relocs.py` + `forensic_elf.py` (capstone 5.0.7, `md.detail=True`). Capability mask keyed by CEA-861 SAD format code as bit position in `r0`:

| bit | SAD format |
|---|---|
| `0x4`    | AC3 (0x06) |
| `0x80`   | DTS-core (0x07) |
| `0x400`  | E-AC3 (0x10) |
| `0x800`  | DTS-HD (0x0B) |
| `0x1000` | MLP (TrueHD) |

### 3.1 Capability-bit test (deployed binary)
```
0x00098a38: tst  r0, #0x800      ; CAP BIT 0x800 = DTS-HD(0x0B)
0x00098a3c: bne  #0x98ad4        ; DTS-HD present -> HD path
0x00098a44: tst  r0, #0x80       ; CAP BIT 0x80  = DTS-core(0x07)   <-- the DTS gate
0x00098a50: bne  #0x98c04        ; DTS-core PRESENT  -> r5=7, r6=1 (compressed)
0x00098a54: mov  r5, #1          ; DTS-core ABSENT  default
0x00098a58: mov  r6, #0          ;        r6 = 0  (compressed flag = 0 -> PCM)
```

### 3.2 Mode selection (driven by `r6`)
```
0x00098c64: cmp  r6, #1
0x00098c68: bne  #0x98c74        ; r6 != 1 -> PCM
0x00098c6c: mov  r4, #2          ; r6 == 1 -> r4 = 2 (COMPRESSED)
0x00098c74: mov  r4, #0          ;            r4 = 0 (PCM)
```

### 3.3 Dispatch into the SPDIF driver
```
0x00098dd8: mov  r0, r4
0x00098ddc: bl   MApi_AUDIO_SPDIF_SetMode     ; arg = r4 (0=PCM, 2=compressed)
0x00098de8: str  r6, [r0, #0x7c]
0x00098dec: str  r5, [r0, #0x80]
```
(branch-reloc resolution confirms `bl@0x98ddc -> MApi_AUDIO_SPDIF_SetMode`; `MApi_AUDIO_SetAC3PInfo` resolved at `0x98d70`.)

### 3.4 Mechanism (proven by binary)
- If the sink EDID advertises **DTS-core** (`0x80` set): `r6 = 1` → mode selector sets `r4 = 2` → `MApi_AUDIO_SPDIF_SetMode(2)` = **compressed/non-PCM** → DTS bitstream passes.
- If the sink EDID advertises **AC3** (`0x4`) but **not DTS-core** (`0x80` clear): the `bne #0x98c04` is *not* taken → fall-through `mov r6,#0` → mode selector sets `r4 = 0` → `MApi_AUDIO_SPDIF_SetMode(0)` = **PCM** → DTS is decoded to PCM and the AVR never sees DTS.

This is **exactly** the observed AC3-pass / DTS-fail asymmetry, and it is fully explained by a single EDID capability bit tested in the kernel gate.

### 3.5 Deployed gate is UNMODIFIED (critical)
The deployed `mik.ko` is `mik_dts_hdmienable.ko` (md5 `647b08db…`). Diff vs pristine `kmods/mik.ko` (md5 `c1421040…`) shows the **only** on-device delta is a 4-byte patch at `0x9a334` (`bne`→`nop`) in `_MI_AOUT_MonitorTask` — **not** the gate. The gate at `0x98a38`/`0x98a44`/`0x98a58` is **pristine/unmodified** on the device. So the device is failing *because of stock firmware behavior*, not a broken patch.

---

## 4. Phase D — Minimal experimental patch: **PREPARED, NOT DEPLOYED**

**Rule:** start from the **current active Exp-B binary**; change **only** the DTS output-mode decision; preserve Exp-B, MonitorTask, AC3/E-AC3/DTS-HD/logdecoder; produce source+output hash, offsets, old+new 4-byte instruction, changed-byte count. **Do not deploy until verified.**

**Patch:** force the compressed flag `r6 = 1` in the DTS-core-absent fall-through, so the selector emits `r4 = 2` (compressed) even when the sink lacks the DTS capability bit.

| field | value |
|---|---|
| file | `runtime_phase5/mik_r5_patched.ko` |
| base (deployed) | `libs/mik_dts_hdmienable.ko` md5 `647b08db…` |
| output | md5 `03fc2c0d22b231bc478e574557b85473` |
| file offset | `0x9b54c` (→ VA `0x98a58`, the `mov r6,#0` default in the gate) |
| old 4-byte instr | `00 60 a0 e3`  (`mov r6, #0`) |
| new 4-byte instr | `01 60 a0 e3`  (`mov r6, #1`) |
| changed bytes | **1** |
| dependency | strict superset of existing `mik_dts_patched.ko` (md5 `ebebb3b3…`, 1-byte at `0x9b54c`); preserves all Exp-B + MonitorTask(`0x9a334`) changes |

**What it does / does not do**
- Does: makes the DTS-core-absent path report *compressed* so `MApi_AUDIO_SPDIF_SetMode` gets mode 2.
- Does not: touch AUTH/licence, EDID data, HDMI/ARC routing, AC3/E-AC3/DTS-HD paths, logdecoder, or any other instruction.

**Honest residual risk (why Phase E is required before trusting it):**
The `mov r6,#0` at `0x98a58` is the default written on the DTS-core-absent branch; whether it is shared with the AC3 entry's `r6` default cannot be 100% excluded from static reading alone. A regression test (Phase E) is the only way to confirm AC3 is not perturbed. The patch is built but **remains local-only**; the device is on the unmodified baseline.

---

## 5. Phase E — Controlled regression test: **NOT RUN (authorization-gated)**

Per the brief, deployment + regression requires explicit user authorization. Not performed. Device left at baseline. If authorized later, procedure (from `tools/r5_phaseA_capture.sh` template):
1. `adb root`; push `mik_r5_patched.ko` to `/vendor/lib/modules/mik.ko`; `insmod`/reboot.
2. Reboot; run **AC3 control** first (`test_ac3_51.mp4`), capture `dmesg`+`logcat`, confirm AVR still locks AC3 (regression check).
3. Run **DTS main test** (`test_dts_51.mp4`), capture logs, verify AVR locks DTS and **records DTS (not PCM)** in its own status display.
4. If DTS locks and AC3 still locks → patch validated. If AC3 regresses → revert immediately.

---

## 6. Device state at end of session (baseline restored)

- `mik.ko` on device = `647b08db…` (hdmienable, **unchanged** at the gate).
- `utpa2k.ko` on device = `4c5e6fbb…` (Exp B, **unchanged**).
- mik debug levels reset to `32`.
- Kodi stopped.
- `mik_r5_patched.ko` exists **locally only** (`runtime_phase5/`), not deployed.

---

## 7. Proven / Strongly supported / Not established

### PROVEN (directly demonstrated)
1. **EDID→mode gate mechanism** in the deployed `mik.ko`: DTS-core capability bit `0x80` tested at `0x98a44`; present → `r6=1` → `SetMode(2)`=compressed; absent → `r6=0` → `SetMode(0)`=PCM. Exact addresses and instructions cited (§3).
2. **The deployed gate is unmodified** vs pristine except the unrelated MonitorTask `0x9a334` patch (§3.5).
3. **DTS licence/AUTH is insufficient**: Exp B emulates the licence yet DTS fails while AC3 passes (§2).
4. **The Phase D patch is a clean 1-byte change** on the deployed base: `0060a0e3`→`0160a0e3` @ file `0x9b54c`, output md5 `03fc2c0d…` (§4).
5. **Phase A runtime observation is impossible** on this device without code changes (§1).

### STRONGLY SUPPORTED (multiple observations, not directly measured)
1. **The connected sink lacks the DTS-core EDID capability bit (`0x80`).** Inferred because: (a) the gate mechanics prove a missing `0x80` forces PCM; (b) the observed AC3-pass/DTS-fail asymmetry matches the missing-`0x80` case exactly; (c) projectors commonly omit DTS SADs in default EDID. *Not directly measured* because Phase A is blocked (the runtime mask is unobservable).
2. **The 1-byte patch will flip DTS to compressed output.** Strongly supported by the gate logic (forcing `r6=1` makes the selector take the `r4=2` branch). *Not directly measured* — requires Phase E.

### NOT ESTABLISHED
1. **Actual runtime EDID capability mask** read by the gate (unobservable — Phase A blocked).
2. **Actual live `MApi_AUDIO_SPDIF_SetMode(arg)` value** during DTS playback (unobservable).
3. **Whether the patch preserves AC3** (no regression run — Phase E not executed).
4. **Whether the AVR physically locks DTS** after the patch (Phase E not executed).
5. **Whether `HAL_AUDIO_SPDIF_Tx_SetNonPCM` / `SetOutputType` add a second gate** below `SetMode` (not reached statically; would need runtime or deeper xref).

---

## 8. Final matrix

| # | Question | Status | Evidence |
|---|---|---|---|
| 1 | DTS parser runs? | **Yes** | R2 static proof |
| 2 | DTS decoder opens? | **Yes** | R3/R4 |
| 3 | DTS licence/AUTH sufficient? | **No** | Exp B emulates licence; AC3 passes, DTS fails (§2) |
| 4 | Compressed audio reaches kernel? | **Yes (AC3)** | AC3 locks on AVR (R1–R4) |
| 5 | Output-mode gate exists in `mik.ko`? | **Yes (proven)** | §3 disassembly, exact VAs |
| 6 | Gate chooses PCM when DTS cap absent? | **Yes (proven by binary)** | §3.1–3.3 |
| 7 | Sink advertises DTS capability (`0x80`)? | **Strongly No (inferred)** | §7.1; Phase A unobservable |
| 8 | Enters non-PCM/SDO for DTS? | **No (when cap absent)** | follows from §3 + §7.1 |
| 9 | Minimal output-mode experiment prepared? | **Yes (not deployed)** | §4, md5 `03fc2c0d…` |
| 10 | Patch restores DTS lock? | **Not established** | Phase E not run |
| 11 | AC3 regression after patch? | **Not established** | Phase E not run |
| 12 | Physical DTS AVR lock observed? | **No** | requires Phase E + AVR |

---

## 9. Recommendation

The evidence is **strong enough to justify the experiment but not strong enough to deploy**:

- The DTS failure is **explained** by the EDID-gated output mode (AC3 passes because its cap bit is present; DTS fails because its cap bit is absent → forced PCM). Licence is ruled out.
- The **minimal patch is ready and verified as a clean, isolated 1-byte change** on the live binary.
- **Next step (requires your explicit authorization):** run Phase E — deploy `runtime_phase5/mik_r5_patched.ko`, reboot, AC3 control test → DTS main test, capture `dmesg`+`logcat`, confirm AVR identifies DTS. If AC3 regresses, revert immediately.

**Nothing has been deployed.** The device is on the unmodified baseline.
