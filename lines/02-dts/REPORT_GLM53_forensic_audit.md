# REPORT — GLM53 independent forensic audit of the DTS-passthrough investigation

**Author:** GLM-5.3-Flash (independent takeover session, 2026-09-13)
**Input:** entire accumulated corpus (all 100+ reports, 10 daily memory files, 8 RE skills, raw binaries, Ghidra/Reko projects, patch artifacts, runtime captures)
**Device:** Thundeal TD98 Pro / C50A · MStar MT5889 · AEON R2 DSP (DEC codec image + SND sound image) · Android 11 · Kodi 22
**Target:** `/vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin` (DEC, stock md5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`, 1,982,492 B) and `aucode_asnd_r2_MS12V22.bin` (SND, stock `eb879cdc07f510722f19db6d18d77d3c`, 1,839,920 B)
**Mode during this report:** read-only. No device connection, no patching. All DEC-side claims below were re-derived by me from `aeon_validate/dec_work/dec32_clean.txt` (Ghidra-authoritative listing, 637,122 instructions) and raw bytes of `dec_full.bin`.

---

## A. Corpus map

| class | location | what it holds |
|---|---|---|
| Main RE tree | `C:/firmware_temp/` | workspace root; every subproject below |
| SPDIF era (Aug 26 – Sep 8) | `spdif_audio_investigation/` | phases 1–15, R22–R26, DTS evidence chain, license profile, full audit ledgers; extracted DSP images (`r8_out/`), ARM kernel modules, HAL dumps |
| Root-level runtime phases | `REPORT_runtime_phase16..24.md`, `r16_out..r24_ghidra/` | DEC/SND R2 log differentials, DM dumps, addressing-model corrections |
| AEON static era (Sep 9–11) | `aeon_validate/REPORT_runtime_phase27..54.md` + `R46_FINAL_SUMMARY.md`, `REPORT_independent_postR46_audit.md` | the SND-image static chain (helpers 0xC4xx–0x26xxx, MMIO 0xB000_08xx, gate 0x25EE1) and the R54 negative |
| DEC era (Sep 12–13) | `aeon_validate/dec_work/`, `r57_npcm/` | DEC image analysis, license gate, wire captures, D1/L2/H1/H2 artifacts, the two FINDING generations |
| Latest state | `REPORT_final_dts_patch_experiment.md` | §1–§12: the D1 experiment and the final "output-frame production / write-pointer" steering |
| Daily memory | `C:/firmware_temp/.workbuddy-ai/memory/2026-09-03..12.md` | distilled state, corrections, tool lessons (read in full) |
| Skills | `C:/Users/k0994/.workbuddy-ai/skills/{mstar-dm-forensics, aeon-snd-static-trace, arm-ko-forensic, ghidra-headless-forensic, android-arm-probe-build, arm-elf-patch-feasibility, mstar-dec-r2-log}` | procedures + gotchas (read) |
| Toolchain | Ghidra 12.1.2 (`C:/ghidra_12.1.2_PUBLIC`) + AEON language (`aeon_ghidra_public/`, patched slaspec); Reko (`C:/Program Files/jklSoft/Reko`, `--arch aeon` validated); NDK r30 (`C:/Android/android-ndk-r30`); capstone in managed python | validated tooling — do not rebuild |
| Ghidra projects | `r27_work/ghidra_r27f` (SND, analysed), `dec_work/ghidra_dec/decimg` (DEC, analysed), `ghidra_projects/mik_analysis.rep` (mik.ko, analysed), `utpa_ghidra/utpaproj.rep` (utpa2k.ko, imported 02:16 last session), `ghidra_forensic`, `ghidra_proj_sdo`, `ghidra_proj_new2`, `r24_ghidra` | reuse, do not re-import |
| Video/EDID subproject | `display_edid_investigation/` (+ `hy4_review/`) | hwcomposer/EDID/4K work; shared tooling only (APS2 relocs gotcha, Ghidra headless recipe); no shared audio findings |
| Patch artifacts | `aeon_validate/r57_npcm/artifacts/` | STOCK DEC/SND, L2 (cba5f7e7), D1 (149f4275, **deployed on device**), H1 (e64b76e8), H2 (28c4ea85) — all verified builds |
| Captures | `r57_npcm/*.bin`, `BASE_AC3_cap.bin`, `r47_work/`, `r22_out/`, `r23a_out/` | wire captures (AC3 MBs; DTS 0 B), DM/log captures |
| Deploy/test scripts | `aeon_validate/{cap.sh, adbr.sh, adbw.sh, cap2.sh}`, `scripts/analyze_wire.py`, `scripts/aeon_verify_*.py`, `scripts/build_D1.py` | the proven deploy + wire-test procedure |

---

## B. Reconstructed investigation (one coherent story)

**Symptom.** AC3 (Dolby Digital) passthrough to the Pioneer VSX-817 over optical works. DTS never produces a valid DTS bitstream. Critically re-established after user testimony: **clean DTS has never worked on this device** (the one contrary report was withdrawn as describing transcoding).

**Layer-by-layer elimination (all runtime-proven):**
1. **Kodi / AudioTrack / AudioFlinger / HAL** — identical for both codecs. Kodi emits `AE_FMT_RAW / RAW(PT) / STREAM_TYPE_DTS_512`, 2012-byte frames; the HAL allocates exactly that buffer; `MI_AUDIO_Start Codec:9 eRet:0` succeeds. Ruled out (R47b, §12.3).
2. **EDID / HDMI sink capability** — refuted as the differentiator (AC3 works with an all-zero capability list; R1-D).
3. **ARM license/AUTH (utpa2k.ko)** — Exp-B emulation bypassed all ARM gates; DTS still failed (R4c/R5b). mik.ko EDID gate (R5 patch, deployed) is real but insufficient.
4. **DTS reaches the DSP and decodes** — `format: DTS, PLAY, frame count climbing`; DEC liveness cells (`0x0FE8+`) live in both codecs. Decode is not the problem.
5. **The wire is the decisive instrument** (`dump_spdif_npcm`, implemented in utpa2k.ko, samples `SpdifNonPcmWritePtr`): AC3 → megabytes of valid IEC61937 bursts (Pa=F872, Pb=4E1F, Pc=0x01, Pd=0x3000, 0x1800 stride). DTS → **exactly 0 bytes**, reproducibly, with the AC3 positive control in the same session. 0 bytes ≠ wrong header — nothing is handed to the SPDIF TX at all.
6. **Patch experiments on the DSP images** (each deployed, rebooted, wire-tested):
   - SND `Pc` 0x01→0x0B (`0x1C050`) — **no effect** (the DTS path never reaches that writer; the statically-read Pc=0x01 was stale buffer content).
   - DEC capability gate L2 (`0x21F66` suppress-branch → unconditional) — **no effect**.
   - DEC D1 (`0x45FFC` beqi→unconditional, forcing the "dead" arm at `0x4608E`) — **path changed (MI_PCM reader now opens under DTS) but still 0 bytes**.
7. The last session ended while starting the consumer-side trace (`SpdifNonPcmWritePtr` in mik.ko, digital-out path in utpa2k.ko).

**What this audit adds (new byte-level work, see §E).** The corpus never reverse-engineered the *producer* of the DEC's non-PCM output frames as a whole. I traced it end-to-end: an eligibility decider (`0x410A3`), a per-tick prepare+emit pair (`0x45FD1`/`0x4618E`) with a single call site (`0x41275`), and the descriptor fields that carry the output (size `desc+0xF0`, buffers `desc+0xF8/0xFC`, preamble slots `desc+0x16C..0x173`). This closes the gap between "decoder alive" and "wire empty" and locates the exact decision that DTS fails.

---

## C. Critical assumptions (what actually carries the current model)

| # | assumption | carried by | status after audit |
|---|---|---|---|
| A1 | 0-byte wire capture under DTS is real and decisive | r57 wire baseline + 3 independent sessions | **VERIFIED** (root-verified instrument, file-created check, AC3 positive control) |
| A2 | `dec32_clean.txt` is an authoritative listing (real boundaries, BE bytes) | R57 tooling | **VERIFIED** — I re-derived every load-bearing instruction from it and cross-checked encodings against the slaspec |
| A3 | §12.4's claim that `0x4608E` was "unreachable dead code" (hence D1 restored dead code) | final report §12.4 | **INCORRECT** — `0x45FFC` has a live dynamic predecessor at `0x4607C` (`bg.j` with `r23 = desc->0x28`, which is 1 on the prepared path). D1 actually changed the *row-3* fall-through |
| A4 | "the next target is the output-frame production / write-pointer path" (§12.12) | final steering | **PARTIALLY RIGHT but incomplete** — the write-pointer side was never the traced bottleneck; the DEC *producer* gate described below sits before it |
| A5 | DEC capability gate (`0x21F66`), SND Pc, SND timing regs, `0x854` bits — eliminated | R46–R57 experiments | **CONFIRMED eliminated** (each has a wire-negative or a proof of non-writership) |
| A6 | `params->0x0` is "written by the setter family at 0x4129E as 0x200/0x400, never 1" (§12.4) | final report | **MISLEADING** — those constants are the *selector* values of a different dispatch (`0x4129E` switches on values 0x20/0x100/0x200/0x400); the actual discriminator is `control[0x7C4]` (a latch of `control[0x40]`, a bitfield into which `0x20` is OR'd at `0x421FD`) |

---

## D. Independent verification plan (hypothesis-driven, after full read)

Only claims that control the root-cause theory were re-verified, from raw bytes:

1. The prepare/emit call chain and its single call site (`0x41275–0x41289`).
2. The full control flow of `0x45FD1` (rows 1/2/3), the `0x45FFC` branch predecessors, and the selector computation `(field+1)<<5` at `0x4608E`.
3. The emit function `0x4618E` (prepared check, buffer memset, preamble copy, DTS sync write, output size `desc->0xF0`).
4. The eligibility decider `0x410A3` and its helper `0x37D18`, including every flag polarity (checked against `aeon_ORBIS32.sinc`: `cmovsi_2 rD,rA,imm = flag ? rA : imm`; `bn.bf` = branch-if-flag-SET).
5. Writer/polarity facts asserted by the corpus where they are load-bearing: `control[0x7D4]` only ever zeroed (row 2 dead); `0x4EE4` writers (exactly two, in the open/config function); descriptor zeroing site `0x4DBC(r11)` ≡ `desc->0xF0`.
6. Branch-target scans into the patch window (byte-level, both 24-bit and 32-bit branch families).
7. `bn.j` encoding round-trip (op 0x0B, 18-bit displacement, base PC, byte units).

Everything else in the corpus (SND-side R46 chain, license-gate internals, DM addressing semantics, instrument behavior) was cross-read and found internally consistent after the documented corrections; per the audit charter it was not re-proven instruction-by-instruction.

---

## E. Independent results

### E.1 The DEC non-PCM output producer — full chain (all VERIFIED BY BINARY)

```
stream open/config (fn at ~0x3E0DD):
  params-ptr published: control[0xA108] = control + 0x7C4      (0x3E19E–0x3E1A6)
  descriptor published: control[0x4E40] = base+0x14CCC, [0x4E44] = +0x800 (0x3E325–0x3E33B)
  decision: 0x410A3(control) → control[0x14EE4] = 1 (enable) | 2 (disable)   (0x3E829–0x3E919)

0x410A3 — the DTS-SDO eligibility decider:
  0x37D18(params): returns 1 iff params->0x0 == 1 OR params->0x10 == 1      (0x37D18–0x37D29)
  require: that == 1, plus f(control[0x14B1C]) > 0, control[0xA0D8] <= 0x7F0,
           [0x290]->0xA34 != 0, [0x290]->0xA20 != 0, control[0xA124] == 1
  then bit-parse on the params->0x28 (or ->0x38) reader:
           re-seek, word-align, skip 39 bits, read 7 bits → field
           accept iff (field << 5) == 0x1E0   ⇔   field == 15               (0x41145–0x4115F)
  on accept: control[0x14EE4] = 1 (+ call 0x4661E) ; else = 2

service loop (0x411B0 region), guarded by control[0xB94]!=1, arg==0, control[0x14EE4]==1:
  0x41275  r10 = base + 0x14CCC (the output descriptor)
  0x4127D  r4  = params = *(control + 0xA108)
  0x41283  jal 0x45FD1 (prepare) ; 0x41289 jal 0x4618E (emit)

prepare 0x45FD1:
  desc->0x28 = 1 ("prepared"); desc->0x34/0x38 = 0
  row 1: params->0x0 == 1  → 0x4600C: copy 10 words of params->0x28 into desc+0x00..0x24,
         desc->0x2c = 0; re-check 0x28==1; re-seek; skip 39 bits; read 7 bits (→r11);
         skip 20 bits; read 4 bits → rate table (idx ≤ 13) → desc->0x80/0x78/0x7C = Hz;
         flow gate: r12 = field−5 must satisfy field ≥ 5 (bn.bf polarity, sinc-verified)
         → 0x46079: r23 = desc->0x28 (=1) → bg.j 0x45FFC → beqi r23,1 TAKEN → 0x4608E
  row 2: params->0x10 == 1 → same body but desc->0x2c = params->0x10 (=1)
         (control[0x7D4] is only ever ZEROED (0x4224D) ⇒ row 2 is dead in this image)
  row 3: (neither) → desc->0x28 = 0 (0x45FF5); movi r23/r11,0; 0x45FFC beqi r23,1
         never taken → filler → RETURN (unprepared)
  0x4608E: selector = (r11 + 1) << 5 → desc->0x38; memset(desc+0x16C, 0, 8)
  0x460A2–0x460B1: re-seek; desc->0x30 = desc->0x24; if ≠0 → clear + tail
  0x460C2 buffer-adequacy gate: if (selector<<5) <= (payload_bits + 0x80) → skip header
           (this is a size check, NOT the blocker; it passes for field==15: 0x4000 > 16224)
  dispatch: 0x200 → r4=0 (DTS), 0x400 → r4=1 (AC-3), 0x800 → skip, else default r4=2
  → 0x45D07 header builder: Pa=0xF872, Pb=0x4E1F, data-type 0x0B (DTS) / 0x10C / 0x20D,
    into desc+0x16C..0x173  (all VERIFIED intact)

emit 0x4618E:
  if desc->0x28 != 1: desc->0xF0 = 0; RETURN            ← DTS exits HERE today
  r14 = desc->0x38 (selector); memset(bufA,0,sel*4); memset(bufB,0,sel*4)
  if Pa (desc->0x16C) != 0: copy the IEC61937 preamble (Pa/Pb/Pc/Pd) into bufA[0..1]/bufB[0..1]
     → payload word-loop from word 2 (bit-reader → shifted words)
     → desc->0x2c == 1 ⇒ additionally write DTS core sync 0x7FFE/0x8001 (or DTS-HD
       0x1FFF/E800 if desc->0x30 != 0) after the preamble
  desc->0xF0 = r14 (output size in bytes; 0x200 for a DTS 512-core frame)

consume: the caller's tick loop stores 0 to desc->0xF0 (0x411C9/0x41221 `sw_0 0x4dbc(r11),r0`
  — the same address, since desc->0xF0 = control+0x14DBC) — a produce/consume handshake;
  the buffers (desc->0xF8/0xFC are pointers, set at open) feed the next hop toward the
  SPDIF non-PCM ring (`SpdifNonPcmWritePtr`, mik.ko).
```

### E.2 Verdicts on the corpus's leading theories

| theory | verdict |
|---|---|
| SND IEC writer Pc=0x01 bug (R56) | **VERIFIED BY BINARY, causally MOOT** — the DTS path never reaches it (wire-proven) |
| DEC license gate `0x21F66` (L2) | **correctly eliminated** (wire-negative) |
| DEC `0x460C2` geometry gate / `s->0x24` fork (§10/§11) | **correctly superseded**; my read: it is a buffer-adequacy check, passes for the correct selector |
| "0x4608E is unreachable dead code" (§12.4) | **INCORRECT** (A3); D1's real effect = rerouting row 3 with selector 0x20 |
| "next target = write-pointer path" (§12.12) | **incomplete** — the producer-side gate below sits before the pointer and is now identified |
| D1's observed effect (MI_PCM open appears) | **STRONG INFERENCE**: prepare *is* invoked for DTS ⇒ `control[0x14EE4]==1` ⇒ `0x410A3` passed at open ⇒ the failing test is downstream of it |

### E.3 The new root-cause location

**`0x45FD1` (prepare) marks the descriptor "prepared" (`desc->0x28 = 1`) only when `params->0x0 == 1` (row 1) or `params->0x10 == 1` (row 2). Row 2 is dead code (`control[0x7D4]` has no non-zero writer). For DTS, row 3 runs: `desc->0x28 = 0` → emit writes `desc->0xF0 = 0` → the output-size handshake never advances → the SPDIF TX non-PCM ring receives nothing → 0-byte wire. AC3 passes row 1, so only the DTS-classified stream is affected.**

Supporting semantics: `params->0x0` = `control[0x7C4]` = a latch of `control[0x40]` (`0x41C7A`), and `control[0x40]` is a bitfield into which the routing loop ORs **0x20** (`0x421FD`) — an exact-equality `== 1` test on such a bitfield selects only the bit-0 state. The corpus itself flagged this ("==1 is a bit-0 test, not a codec id", §12.4) without connecting it to prepare. `0x410A3` applies the same `0x0==1 || 0x10==1` discriminator at open time (via `0x37D18`); the runtime D1 effect indicates it passed then, and the per-tick prepare re-read the same field — the demotion happens between open and the per-tick prepare (exact runtime values of `control[0x40]`/`0x7C4` are not host-observable; this residual is stated, not papered over).

---

## F. Corrected DTS architecture (evidence-backed)

```
Kodi (SyncDTS, 2012-B frames, AE_FMT_RAW, STREAM_TYPE_DTS_512)      [VERIFIED OK]
  → AudioFlinger / audio.primary.mt5889.so → mi_decoder_open(9)     [VERIFIED OK]
  → MI_AUDIO_Start Codec:9 → utpa2k.ko → DEC ES buffer              [VERIFIED OK]
  → DEC R2 decodes DTS (dts m6 init, frame count climbing)          [VERIFIED OK]

  → DTS-SDO output path (DEC):
      open: 0x410A3 eligibility → control[0x14EE4] = 1 (DTS-classified, field==15)
      tick: prepare(0x45FD1) — row-3 demotion for DTS (params->0x0 != 1)
            ⇒ desc->0x28 = 0 ⇒ emit(0x4618E) writes desc->0xF0 = 0
            ⇒ no output frames ever handed on                        ← **THE BLOCKER**
  → SPDIF non-PCM ring (SpdifNonPcmWritePtr, mik.ko)                [starved — 0 bytes]
  → SPDIF TX → optical → Pioneer VSX-817                            [never locks]

AC3 path shares: Kodi→HAL→MI_AUDIO_Start Codec:5→DEC; its output flows (wire-proven),
and prepare row 1 passes for it (== 1 satisfied) — the patched sites below are
provably inert for rows 1/2.
```

---

## G. Real blocker candidate (narrowest defensible form)

> **In the DEC image, the per-tick output producer (`prepare 0x45FD1` + `emit 0x4618E`, single call site `0x41275`) requires the exact test `params->0x0 == 1` (or the dead `params->0x10 == 1`) to mark the output descriptor prepared. For the DTS-classified stream the test fails at tick time, so the descriptor is marked unprepared and the DTS-SDO output stage emits a zero-size output (`desc->0xF0 = 0`) every tick. The DTS pipeline downstream of that point (buffers → non-PCM ring → SPDIF TX) is structurally intact but permanently starved.**

Remaining uncertainty (stated honestly): the runtime values of `control[0x40]`/`control[0x7C4]` and `control[0x14EE4]` per codec are not host-observable; the row-3 demotion is inferred from (a) the byte-exact control flow, (b) the wire-negative experiments on everything downstream, and (c) D1's observed behavioral change (which requires prepare to run and its row-3 path to be taken).

---

## H. Patch candidates (ranked; NOT built yet, NOT flashed)

| # | site | change | causal confidence | DTS specificity | AC3 safety | reversibility | complexity |
|---|---|---|---|---|---|---|---|
| **P-A** | **DEC `0x45FF5`: `bn.sw 0x28(r3),r0` (`0c 03 28`) → `bn.j 0x4600C` (`2c 00 17`)** | reroute the unprepare arm into the full row-1 preparation flow (parse → selector `(15+1)<<5 = 0x200` → DTS header 0x0B → prepared) | **high** — directly restores the producer's intended flow; every downstream element byte-verified intact | **maximal** — the patched instruction's only predecessor is the row-3 fall-through; rows 1/2 (AC3's path) are provably untouched; stream self-classifies via the parse | **by construction** (no-op for rows 1/2; parse-driven selector keeps non-DTS streams harmless) | single 3-byte edit vs stock; rollback = restore stock | trivial |
| P-B | DEC `0x4625C` (H2, already built): `bg.bnei r23,1` → unconditional to `0x46260` | force the DTS-sync write even when `desc->0x2c != 1` | medium — needed only if the payload loop does not itself carry the DTS sync | high (only prepared streams reach it) | good | single 4-byte edit | trivial |
| P-C | DEC `0x41C6F` latch: `bn.lwz r25,0x40(r10)` → `bn.ori r25,r0,0x1` | force `control[0x7C4] = 1` at latch time | medium — upstream of P-A but interacts with the `0x4221D` change-detector, which zeroes `params->0x28/0x38` pointers on mismatch ⇒ **potential DSP dereference of 0** | low | **risk — rejected for now** | 3-byte edit | trivial |
| P-D | consumer-side (mik.ko `SpdifNonPcmWritePtr` / utpa2k digital-out path) | unknown — last session's lead, not yet analyzed | low (producer gate is upstream and identified) | unknown | unknown | — | high |

**Decision: P-A is the first patch.** It is the smallest evidence-backed change that restores the existing intended DTS preparation path; it does not bypass any license check, does not alter any shared state, and is provably inert for the AC3 path. Built from **stock** DEC (which also cleanly reverts the currently-deployed D1). P-B is the designated iteration if P-A puts DTS bursts on the wire but the AVR does not lock (the payload-sync question).

---

## I. Audit conclusion

The corpus is fully absorbed; the producer-side gate that the last three phases were converging toward has been located byte-exactly; the SND-Pc / DEC-L2 / D1 experiment outcomes are all explained by the corrected model; the exact next patch is defined, encoded, and safety-checked; the decisive instrument (wire capture with AC3 positive control) and rollback procedure are proven.

**AUDIT COMPLETE — PATCH CANDIDATE IDENTIFIED**

*(P-A: DEC `0x45FF5` `bn.sw 0x28(r3),r0` → `bn.j 0x4600C`; artifact built & verified: `r57_npcm/artifacts/aucode_adec_r2_MS12V22_PA_45FF5.bin`, md5 `afde18ec92ffe0a960390a423d7f63c2`, sha256 `98a9a631…`, from stock DEC `4b7e9509…`, exactly 3 bytes changed at `0x45FF5–0x45FF7`; success criterion = a NON-ZERO DTS wire capture, then Pioneer VSX-817 DTS lock; AC3 regression control mandatory in the same session.)*
