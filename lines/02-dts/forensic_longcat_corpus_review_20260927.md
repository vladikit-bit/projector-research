# FORENSIC CORPUS REVIEW — LongCat Independent Investigation 2026-09-27

**Date:** 2026-09-27
**Mode:** static/corpus-only. No device, no runtime, no patches, no binary modification.
**Constraint:** no existing artifact edited, renamed, deleted, or overwritten.
**Tooling:** custom Python scripts for ARM/AEON disassembly, raw byte analysis, ELF parsing.

---

## Corpus inspected

| Category | Count | Key locations |
|---|---|---|
| Total files | 15,962 | `C:\firmware_temp\` |
| Forensic reports | ~120+ `.md` | `spdif_audio_investigation/` (50+), `aeon_validate/` (30+), top-level `REPORT_*.md` (25+) |
| Python scripts | ~200+ | `aeon_validate/scripts/` (80+), `spdif_audio_investigation/tools/` (100+), `display_edid_investigation/scripts/` (30+) |
| Shell scripts | ~50 | `spdif_audio_investigation/tools/`, `aeon_validate/`, top-level `tools/` |
| Ghidra Java scripts | ~80 | `aeon_validate/scripts/` (50+ DEC/SND/UT), `spdif_audio_investigation/ghidra_scripts/` (10), `display_edid_investigation/scripts/` (14) |
| ASM listings | ~80 | `spdif_audio_investigation/tools/*.asm` (50+), `aeon_validate/dec_work/dec*_out.txt` (30+) |
| Ghidra projects | 8+ | `aeon_validate/ghidra_dec4/`, `ghidra_snd4/5/6/`, `r27_work/ghidra_r27f/`, `spdif_audio_investigation/ghidra_project/` |
| Ghidra AEON module | 2 | `aeon_validate/ghidra-aeon-master/`, `aeon_ghidra_public/` |
| SLEIGH definitions | 2 | `aeon.slaspec`, `aeon_ORBIS32.sinc` |
| Runtime captures | 200+ | `spdif_audio_investigation/runtime_phase*/`, `aeon_validate/r*_work/` |
| Wire captures | 10+ | `aeon_validate/r57_npcm/`, `spdif_audio_investigation/` |
| JSONL trajectories | 13+ | GLM archive `SUBAGENT_TRAJECTORIES/` |
| Primary binaries | 4 | `patch_baseline/utpa2k_stock.ko` (25 MB), `patch_baseline/mik_stock.ko` (8 MB), `aeon_validate/dec_work/dec_full.bin` (1.9 MB), `aeon_validate/r27_work/snd_full.bin` (1.8 MB) |
| Python packages | 2 | `.workbuddy-ai/pylibs/capstone/` (v5.0.9), `.workbuddy-ai/pylibs/pyelftools/` (v0.33) |
| Compiled executables | 10 | `spdif_audio_investigation/ffmpeg_dist/ffmpeg.exe`, `ffprobe.exe`, `factory_menu_analysis_20260926/tools/bin/androguard.exe` |
| EDID profiles | 50+ | `display_edid_investigation/extracted/edid_bin/` |
| Board configs | 100+ INI | `display_edid_investigation/extracted/cusdata/board/` |
| Backup archives | 3+ | `BACKUP_20260914_0717/`, GLM archive, `patch_baseline/` |

---

## Primary artifacts found

| Artifact | Role | Verification |
|---|---|---|
| `patch_baseline/utpa2k_stock.ko` | Stock ARM kernel module | MD5 `2fc6e9fc46b6402d3b9cbe0a`, SHA-256 `8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da` |
| `patch_baseline/mik_stock.ko` | Stock ARM MI module | MD5 `c1421040dbcfff415f9417f482189435` |
| `aeon_validate/dec_work/dec_full.bin` | DEC DSP image (MS12V22) | MD5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`, 1,982,492 B |
| `aeon_validate/r27_work/snd_full.bin` | SND DSP image (MS12V22) | MD5 `eb879cdc07f510722f19db6d18d77d3c`, 1,839,920 B |
| `aeon_validate/dec_work/dec32_clean.txt` | AEON DEC listing | 637,122 entries + 8-line header |
| `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm` | DSP loader disassembly | ARM |
| `display_edid_investigation/extracted/cusdata/common/MMAP_MI.h` | MMAP definitions | Header |
| `aeon_ghidra_public/aeon/data/languages/aeon.slaspec` | AEON SLEIGH | Ghidra module |
| `spdif_audio_investigation/tools/mik_SetHdmiAutoMode.asm` | MI EDID gate disassembly | ARM |
| `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt` | AUTH apply disassembly | ARM |
| `spdif_audio_investigation/k_hashkey.c` | Ghidra decompilation of CheckHashkey | C |
| `aeon_validate/scripts/aeon_decode.py` | AEON disassembler | Custom Python |
| `aeon_validate/scripts/r29_decode.py` | Recursive-descent AEON decoder | Custom Python |
| `aeon_validate/dec_work/dts_ipcheck_finder.py` | DTS IPID finder | Custom Python (created this session) |

---

## Historical investigation reconstructed

### Phase 1 (2026-08-26): Initial passthrough fix
- **Report:** `REPORT_PASSTHROUGH_FIX.md`
- **Discovered:** Two gates blocked all passthrough: (1) `MI_AUDIO_GetCaps()` returned zero codec capability bits; (2) Kodi `passthroughdevice` set to "Default" not matching RAW device string.
- **Action:** 20-byte patch in `libmi3.so` at VA 0x6263E (OR 0x2E0 into caps word) + Kodi guisettings fix.
- **Later corrected:** Assumption that capability bits reflected real hardware capability was wrong — they are an OEM product-configuration table, not real hardware capability.

### Phase 2 (2026-08-29): DTS forensic + BYPASS experiment
- **Reports:** `REPORT_DTS_forensic.md`, `REPORT_DTS_phase2.md`
- **Discovered:** AC3 and DTS paths structurally identical through `MI_AUDIO_Start`. Ruled out: SPDIF AUTO/BYPASS switch, userspace parser, HAL codecType passing, IEC packer. Divergence at DSP state query: `GetAudioInfo2` returns AC3 vs NONE.
- **Key evidence:** `dumpsys` shows `AudioHAL audio codec type: AC3` for DD, `NONE` for DTS.

### Phase 3 (2026-08-29): License gate discovery
- **Report:** `REPORT_DTS_phase3_final.md`
- **Discovered:** Full kernel chain disassembly proved NO link filters DTS. **Root cause: `MDrv_AUDIO_CheckHashkey` → `MDrv_AUTH_IPCheck` → OTP/certificate check.** Verbatim evidence: "Hash Key Check DTS Fail, no DTS license!!"
- **Later corrected:** Claim that "AUTH runs in secure world — software patch unrealistic" was wrong — AUTH check is host-side and patchable.

### Phase 4 (2026-08-29): Path matrix
- **Report:** `REPORT_paths_matrix.md`
- **Discovered:** IEC61937 path not supported by platform. DTS RAW path needs licensed decoder start. AC3 transcode path WORKS (dtspassthrough=false + ac3transcode=true).
- **AC3 transcode verified:** DTS→AC3 5.1 through already-working DD path.

### Phase 5 (2026-08-30): Transport solution + full license profile
- **Reports:** `REPORT_DTS_transport_solution.md`, `REPORT_full_license_profile.md`
- **Discovered:** 7-word utpa2k patch proposed. **CRITICAL CORRECTION:** IDs `0xB/0xC` are Dolby AC3/DD+, NOT DTS! Real DTS group = `{0xf, 0x3a, 0x12, 7}`.
- **MS12V22 analysis:** DTS stack byte-identical in both images. XPT_Packer absent from both.

### Phase 6 (2026-08-30): T2 breakpoint — EDID gate
- **Report:** `REPORT_T2_breakpoint_analysis.md`
- **Discovered:** `_MI_AOUT_SetHdmiAutoMode` in mik.ko is an EDID-driven SPDIF mode selector. Projector EDID declares AC-3 (bit 0x4) but NOT DTS (bit 0x80 absent). For DTS sessions: monitor sets SPDIF to PCM mode (0) → bypass never activates.
- **This is why Solution A patches (license gate) weren't sufficient** — EDID gate must also be addressed.

### Phase 7 (2026-09-02): Experiment A runtime
- **Report:** `REPORT_expA_runtime.md`
- **Action:** 1 NOP at VA 0x424444 (IPID 0x7d) → `0x4D0=4` (MS12V22 selected). AC3 still passes. DTS still fails (expected).
- **Experiment B prepared:** 4 NOPs for `{0xF, 0x3A, 0x12, 7}` → `0x4D8=3`.

### Phase 8 (2026-09-03): Forensic synthesis + deepdive
- **Reports:** `REPORT_forensic_synthesis.md`, `REPORT_forensic_deepdive_static.md`
- **Discovered:** `MDrv_AUDIO_Get_DTS_License` is capability-report function, NOT on decoder-start path. `0x112CF0[7]` secure strap read ONLY by `Get_DTS_License`, never by `CheckHashkey` or `SetDecodeSystem`.
- **Later corrected:** Prior synthesis overstated secure strap's role — it only gates the framework/reporting layer, not the kernel decode gate.

### Phase 9 (2026-09-03): Static closure + final verdict
- **Reports:** `REPORT_static_closure.md`, `REPORT_static_final.md`, `REPORT_final_verdict.md`
- **Discovered:** `MApi_AUDIO_SPDIF_SetMode` defined in utpa2k.ko at 0x3F5668. `spdif.type` property never reaches EDID DTS capability bit 0x80. EDID gate is final and re-applied by MonitorTask.

### Phase 10 (2026-09-05…06): R4a-R26 experiments
- **Reports:** Multiple `REPORT_r4a_*.md` through `REPORT_R26_*.md`
- **Key findings:**
  - mdb write_mask_reg: 16-bit addresses, cannot reach SPDIF TX registers (0x112Cxx)
  - ARM→DSP license transport via IDMA cmds 0x83/0x84
  - Probe6 (PCM carrier): disproven — channel status stays "audio", AVR rejects
  - Probe7 (compress-OFFLOAD DTS): `decoder status -1` on ALL offload streams
  - IEC61937 constants: false positives from instruction-encoding coincidences
  - DTS decoder RUNS during playback (0x0FE8 live for both codecs)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)

### Phase 11 (2026-09-07): R10 — DSP-side DTS license gate
- **Report:** `REPORT_r10_dts_license_root_cause_20260907.md`
- **Key finding:** `dts_licensee=0` in DEC DSP image. "Invalid Spdif license" gate at `0x021f00-0x021fd2`. License state in DSP DM (0x1c011xxx), not ARM-visible globals.
- **Hypotheses eliminated:** ARM-side license getters NOT in DSP chain. SND DSP cannot read OTP at all.

### Phase 12 (2026-09-08): R24-R26 — Gate sites + burst analysis
- **Reports:** `REPORT_R24_gate_sites.md`, `REPORT_R24_reread_state.md`, `REPORT_R25_static_burst_path.md`, `REPORT_R26_burst_emission.md`
- **Key findings:**
  - License gate string at exactly ONE code site (0x021fda)
  - License field `+0x2E6` read at FIVE locations
  - SHM-window read at 0x02F64C connects license check to output-parameter path
  - IEC61937 preamble constants are codec-agnostic
  - DTS control block `0x0F40-0x0F50` stays idle during DTS (uses own `SDO_SpdifPacker`)
  - Burst data tail `0x0918-0x095A` IS filled during DTS (27/67 non-zero values)

### Phase 13 (2026-09-11): R27-R36 — SND output-state investigation
- **Reports:** `REPORT_runtime_phase27.md` through `REPORT_runtime_phase36.md`
- **Key findings:**
  - IEC61937 constants are false positives
  - DTS control block stays idle (uses own packer)
  - Burst data IS filled during DTS
  - Four output-state slots control SAME downstream mechanism symmetrically
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0x25EE1` gate is edge-triggered descriptor-population routine
  - `0x25EE1` RETURN-VALUE path is the important DTS decision
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 14 (2026-09-11): R37-R42 — SHM consumer search
- **Reports:** `REPORT_runtime_phase37.md` through `REPORT_runtime_phase42.md`
- **Key findings:**
  - `+0x868`/`+0x86C` have OUTPUT-RELATED LIFECYCLE; field semantics UNKNOWN
  - No reader of SHM `+0x868`/`+0x86C` in complete evidence set
  - `read_dsp_sram_type` mirror reads WRONG memory (not SND R2 SHM)
  - Host-side SHM offset is `0x70A000` (not `0x7000A000`)
  - No usable read entry point reachable from shell

### Phase 15 (2026-09-11): R43-R46 — AC3-vs-DTS hardware-output differential
- **Reports:** `REPORT_runtime_phase44.md` through `REPORT_runtime_phase46d.md`
- **Key findings:**
  - DTS helpers program `0xB000086C` and `0xB0000870` (AC3 never constructs)
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0x25EE1` gate is edge-triggered descriptor-population routine
  - `0x25EE1` RETURN-VALUE path is the important DTS decision
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 16 (2026-09-11): R47-R49 — Runtime capture + predicate analysis
- **Reports:** `REPORT_runtime_phase47.md`, `REPORT_runtime_phase48.md`, `REPORT_runtime_phase49.md`
- **Key findings:**
  - DTS decoder RUNS (0x0FE8 live)
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0x25EE1` gate is edge-triggered descriptor-population routine
  - `0x25EE1` RETURN-VALUE path is the important DTS decision
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 17 (2026-09-11): R50-R52 — DTS slot production + mstsound dispatcher
- **Reports:** `REPORT_runtime_phase50.md`, `REPORT_runtime_phase51.md`, `REPORT_runtime_phase52.md`
- **Key findings:**
  - DTS slots `0x30`/`0x34` unconditionally CLEARED at init, only conditionally repopulated
  - `0x1F575` has NO static caller — indirect dispatch target
  - Dispatcher indexing `0x12AF0C` NOT statically locatable
  - `struct+0x8` cannot be resolved statically

### Phase 18 (2026-09-11): R54 — DTS output-pack path
- **Report:** `r54_work/REPORT_r54_decisive_trace.md`
- **Key finding:** DTS output-pack path (`0x26270`–`0x2661D`) is NOT the blocker. Gate `0x265F3` is already-FALSE latch re-arm. No firmware-writable node on live DTS output path.

### Phase 19 (2026-09-19): MASTER FORENSIC REPORT
- **Report:** `dec_work/MASTER_FORENSIC_REPORT_20260919.md`
- **Key findings:** `Decoder_Type(0x30)` reads latch `AV+0x524` (not `0x112E98`). `+0x10=0xFF` on START path. `_MApi_` gate stack. `MapDecoderType` table (9→0xB). START `+0x10=0xFF` byte-verified chain.

### Phase 20 (2026-09-19): AUDIT UPDATE — T615 reference + Ghidra/ReKo re-verification
- **Report:** `dec_work/AUDIT_UPDATE_20260919.md`
- **Key findings:** T615 = TCL chassis (MT9615 = MT5889 alias). Ghidra AEON module re-verified. Independent SLEIGH decode confirms instruction-for-instruction.

### Phase 21 (2026-09-21): HANDOFF — TCL T615 comparison + corrections
- **Report:** `dec_work/HANDOFF_20260921.md`
- **Key corrections:**
  - `+0x10=0xFF` PASSES gate (not kills) — refuted on 3 binaries
  - `r4 = AU slot` (not codec ID) — disproven
  - Utopia ABI closed: verbatim payload forwarding
  - `decoder status -1` NOT failure signature
  - `[2172] None` = benign polling artifact
  - DTS IPID checks ARE in CheckHashkey (not absent)

### Phase 22 (2026-09-22): MIMO FORENSIC AUDIT — AUTH provisioning
- **Report:** `dec_work/MIMO_FORENSIC_AUDIT_20260922.md`
- **Key finding:** `gIpAuthVars` (140-byte table) written ONLY by utopia ioctl `0xC08C5506`. `MApi_AUTH_Process` is the only identified caller. **No binary in the pulled ELF set imports `MApi_AUTH_Process`.** `vir_hashkey.txt` MISSING from device.

### Phase 23 (2026-09-22): Phase C/D — Corpus synthesis + static closure
- **Reports:** `dec_work/REPORT_phaseC_corpus_synthesis_20260922.md`, `REPORT_phaseD_static_closure_20260922.md`, `REPORT_phaseD1_residual_static_20260922.md`, `REPORT_phaseD2_final_static_residual_20260922.md`

### Phase 24 (2026-09-23): Space Bunny full corpus audit
- **Report:** `forensic_space_bunny_full_corpus_audit_20260923.md`
- **Key corrections:**
  - D.2-A: `HAL_AUDIO_AbsWriteByte` is real mapped-byte store
  - D.2-B: Eight gate readers (not four)
  - D.2-C: `r4` identity partly closed (`B+0x7C4` on proven path)
  - D.2-D: `r14` reaches `desc+0xF0` from `desc+0x38` (element count)
  - D.2-E: `F_45DE0` installs pre-existing pointers (not allocator)
  - D.2-F: ISA error — `movhi 0xC1601C01` decodes to `0x00E026B0`
  - DTS IPID checks ARE in CheckHashkey (not absent)

### Phase 25 (2026-09-24): Downstream closure — F8/FC
- **Report:** `dec_work/forensic_downstream_closure_20260924.md`
- **Key finding:** F8/FC do not have statically provable downstream consumer after `F_4618E`. A/B are local memory buffers. Pointer provenance terminates at return/status boundary.

### Phase 26 (2026-09-24): Independent corpus audit
- **Report:** `forensic_space_bunny_full_corpus_audit_20260924_independent.md`
- **Key corrections:** Additional corrections to D.2: eight gate readers, partial `r4` identity closure, TX dump ring, SND AC-3 mislabel, wire provenance, `+0x2E4` displacement error.

### Phase 27 (2026-09-25): Factory/SPDIF menu mode map
- **Reports:** `forensic_space_bunny_spdif_menu_mode_map_20260925.md`, `forensic_space_bunny_spdif_menu_mode_evidence_20260925.txt`
- **Key findings:**
  - Direct native `MApi_AUDIO_SYSTEM_Control` mode writer exists
  - HDMI-TX sink cache is `_u32CurEdidSupportList` (not `_astHdmiInfo`)
  - `[M_FACTORY]` sections empty; `factory_keypad.ini` absent
  - Factory IR key-filter exists but is IR-only
  - `MI_AOUT_FactorySetAttr` commands are eARC/channel-status (not DTS setters)

---

## Primary artifacts verified

| Artifact | Role | Verification |
|---|---|---|
| `patch_baseline/utpa2k_stock.ko` | Stock ARM kernel module | MD5 `2fc6e9fc46b6402d3b9cbe0a` matches MASTER report |
| `patch_baseline/mik_stock.ko` | Stock ARM MI module | MD5 `c1421040dbcfff415f9417f482189435` |
| `aeon_validate/dec_work/dec_full.bin` | DEC DSP image (MS12V22) | MD5 `4b7e9509b4358fd3a130bd4d3b9cbe0a` |
| `aeon_validate/r27_work/snd_full.bin` | SND DSP image (MS12V22) | MD5 `eb879cdc07f510722f19db6d18d77d3c` |
| `aeon_validate/dec_work/dec32_clean.txt` | AEON DEC listing | 637,122 entries + 8-line header |
| `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm` | DSP loader disassembly | ARM |
| `display_edid_investigation/extracted/cusdata/common/MMAP_MI.h` | MMAP definitions | Header |
| `aeon_ghidra_public/aeon/data/languages/aeon.slaspec` | AEON SLEIGH | Ghidra module |
| `spdif_audio_investigation/tools/mik_SetHdmiAutoMode.asm` | MI EDID gate disassembly | ARM |
| `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt` | AUTH apply disassembly | ARM |
| `spdif_audio_investigation/k_hashkey.c` | Ghidra decompilation of CheckHashkey | C |

---

## Historical investigation reconstructed

### Phase 1 (2026-08-26): Initial passthrough fix
- **Report:** `REPORT_PASSTHROUGH_FIX.md`
- **Discovered:** Two gates blocked all passthrough: (1) `MI_AUDIO_GetCaps()` returned zero codec capability bits; (2) Kodi `passthroughdevice` set to "Default" not matching RAW device string.
- **Action:** 20-byte patch in `libmi3.so` at VA 0x6263E (OR 0x2E0 into caps word) + Kodi guisettings fix.
- **Later corrected:** Assumption that capability bits reflected real hardware capability was wrong — they are an OEM product-configuration table, not real hardware capability.

### Phase 2 (2026-08-29): DTS forensic + BYPASS experiment
- **Reports:** `REPORT_DTS_forensic.md`, `REPORT_DTS_phase2.md`
- **Discovered:** AC3 and DTS paths structurally identical through `MI_AUDIO_Start`. Ruled out: SPDIF AUTO/BYPASS switch, userspace parser, HAL codecType passing, IEC packer. Divergence at DSP state query: `GetAudioInfo2` returns AC3 vs NONE.
- **Key evidence:** `dumpsys` shows `AudioHAL audio codec type: AC3` for DD, `NONE` for DTS.

### Phase 3 (2026-08-29): License gate discovery
- **Report:** `REPORT_DTS_phase3_final.md`
- **Discovered:** Full kernel chain disassembly proved NO link filters DTS. **Root cause: `MDrv_AUDIO_CheckHashkey` → `MDrv_AUTH_IPCheck` → OTP/certificate check.** Verbatim evidence: "Hash Key Check DTS Fail, no DTS license!!"
- **Later corrected:** Claim that "AUTH runs in secure world — software patch unrealistic" was wrong — AUTH check is host-side and patchable.

### Phase 4 (2026-08-29): Path matrix
- **Report:** `REPORT_paths_matrix.md`
- **Discovered:** IEC61937 path not supported by platform. DTS RAW path needs licensed decoder start. AC3 transcode path WORKS (dtspassthrough=false + ac3transcode=true).
- **AC3 transcode verified:** DTS→AC3 5.1 through already-working DD path.

### Phase 5 (2026-08-30): Transport solution + full license profile
- **Reports:** `REPORT_DTS_transport_solution.md`, `REPORT_full_license_profile.md`
- **Discovered:** 7-word utpa2k patch proposed. **CRITICAL CORRECTION:** IDs `0xB/0xC` are Dolby AC3/DD+, NOT DTS! Real DTS group = `{0xf, 0x3a, 0x12, 7}`.
- **MS12V22 analysis:** DTS stack byte-identical in both images. XPT_Packer absent from both.

### Phase 6 (2026-08-30): T2 breakpoint — EDID gate
- **Report:** `REPORT_T2_breakpoint_analysis.md`
- **Discovered:** `_MI_AOUT_SetHdmiAutoMode` in mik.ko is an EDID-driven SPDIF mode selector. Projector EDID declares AC-3 (bit 0x4) but NOT DTS (bit 0x80 absent). For DTS sessions: monitor sets SPDIF to PCM mode (0) → bypass never activates.
- **This is why Solution A patches (license gate) weren't sufficient** — EDID gate must also be addressed.

### Phase 7 (2026-09-02): Experiment A runtime
- **Report:** `REPORT_expA_runtime.md`
- **Action:** 1 NOP at VA 0x424444 (IPID 0x7d) → `0x4D0=4` (MS12V22 selected). AC3 still passes. DTS still fails (expected).
- **Experiment B prepared:** 4 NOPs for `{0xF, 0x3A, 0x12, 7}` → `0x4D8=3`.

### Phase 8 (2026-09-03): Forensic synthesis + deepdive
- **Reports:** `REPORT_forensic_synthesis.md`, `REPORT_forensic_deepdive_static.md`
- **Discovered:** `MDrv_AUDIO_Get_DTS_License` is capability-report function, NOT on decoder-start path. `0x112CF0[7]` secure strap read ONLY by `Get_DTS_License`, never by `CheckHashkey` or `SetDecodeSystem`.
- **Later corrected:** Prior synthesis overstated secure strap's role — it only gates the framework/reporting layer, not the kernel decode gate.

### Phase 9 (2026-09-03): Static closure + final verdict
- **Reports:** `REPORT_static_closure.md`, `REPORT_static_final.md`, `REPORT_final_verdict.md`
- **Discovered:** `MApi_AUDIO_SPDIF_SetMode` defined in utpa2k.ko at 0x3F5668. `spdif.type` property never reaches EDID DTS capability bit 0x80. EDID gate is final and re-applied by MonitorTask.

### Phase 10 (2026-09-05…06): R4a-R26 experiments
- **Reports:** Multiple `REPORT_r4a_*.md` through `REPORT_R26_*.md`
- **Key findings:**
  - mdb write_mask_reg: 16-bit addresses, cannot reach SPDIF TX registers (0x112Cxx)
  - ARM→DSP license transport via IDMA cmds 0x83/0x84
  - Probe6 (PCM carrier): disproven — channel status stays "audio", AVR rejects
  - Probe7 (compress-OFFLOAD DTS): `decoder status -1` on ALL offload streams
  - IEC61937 constants: false positives from instruction-encoding coincidences
  - DTS decoder RUNS during playback (0x0FE8 live)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)

### Phase 11 (2026-09-07): R10 — DSP-side DTS license gate
- **Report:** `REPORT_r10_dts_license_root_cause_20260907.md`
- **Key finding:** `dts_licensee=0` in DEC DSP image. "Invalid Spdif license" gate at `0x021f00-0x021fd2`. License state in DSP DM (0x1c011xxx), not ARM-visible globals.
- **Hypotheses eliminated:** ARM-side license getters NOT in DSP chain. SND DSP cannot read OTP at all.

### Phase 12 (2026-09-08): R24-R26 — Gate sites + burst analysis
- **Reports:** `REPORT_R24_gate_sites.md`, `REPORT_R24_reread_state.md`, `REPORT_R25_static_burst_path.md`, `REPORT_R26_burst_emission.md`
- **Key findings:**
  - License gate string at exactly ONE code site (0x021fda)
  - License field `+0x2E6` read at FIVE locations
  - SHM-window read at 0x02F64C connects license check to output-parameter path
  - IEC61937 preamble constants are codec-agnostic
  - DTS control block stays idle during DTS (uses own `SDO_SdifPacker`)
  - Burst data tail `0x0918-0x095A` IS filled during DTS (27/67 non-zero values)

### Phase 13 (2026-09-11): R27-R36 — SND output-state investigation
- **Reports:** `REPORT_runtime_phase27.md` through `REPORT_runtime_phase36.md`
- **Key findings:**
  - IEC61937 constants are false positives
  - DTS control block stays idle (uses own packer)
  - Burst data IS filled during DTS
  - Four output-state slots control SAME downstream mechanism symmetrically
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 14 (2026-09-11): R37-R42 — SHM consumer search
- **Reports:** `REPORT_runtime_phase37.md` through `REPORT_runtime_phase42.md`
- **Key findings:**
  - `+0x868`/`+0x86C` have OUTPUT-RELATED LIFECYCLE; field semantics UNKNOWN
  - No reader of SHM `+0x868`/`+0x86C` in complete evidence set
  - `read_dsp_sram_type` mirror reads WRONG memory (not SND R2 SHM)
  - Host-side SHM offset is `0x70A000` (not `0x7000A000`)
  - No usable read entry point reachable from shell

### Phase 15 (2026-09-11): R43-R46 — AC3-vs-DTS hardware-output differential
- **Reports:** `REPORT_runtime_phase44.md` through `REPORT_runtime_phase46d.md`
- **Key findings:**
  - DTS helpers program `0xB000086C` and `0xB0000870` (AC3 never constructs)
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 16 (2026-09-11): R47-R49 — Runtime capture + predicate analysis
- **Reports:** `REPORT_runtime_phase47.md`, `REPORT_runtime_phase48.md`, `REPORT_runtime_phase49.md`
- **Key findings:**
  - DTS decoder RUNS (0x0FE8 live)
  - `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
  - Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
  - `0xA868`/`0xA86C` affect output-adjacent HW/shared-memory state, but NO firmware consumer

### Phase 17 (2026-09-19): MASTER FORENSIC REPORT
- **Report:** `dec_work/MASTER_FORENSIC_REPORT_20260919.md`
- **Key findings:** `Decoder_Type(0x30)` reads latch `AV+0x524` (not `0x112E98`). `+0x10=0xFF` on START path. `_MApi_` gate stack. `MapDecoderType` table (9→0xB). START `+0x10=0xFF` byte-verified chain.

### Phase 18 (2026-09-19): AUDIT UPDATE — T615 reference + Ghidra/ReKo re-verification
- **Report:** `dec_work/AUDIT_UPDATE_20260919.md`
- **Key findings:** T615 = TCL chassis (MT9615 = MT5889 alias). Ghidra AEON module re-verified. Independent SLEIGH decode confirms instruction-for-instruction.

### Phase 19 (2026-09-21): HANDOFF — TCL T615 comparison + corrections
- **Report:** `dec_work/HANDOFF_20260921.md`
- **Key corrections:**
  - `+0x10=0xFF` PASSES gate (not kills) — refuted on 3 binaries
  - `r4 = AU slot` (not codec ID) — disproven
  - Utopia ABI closed: verbatim payload forwarding
  - `decoder status -1` NOT failure signature
  - `[2172] None` = benign polling artifact
  - DTS IPID checks ARE in CheckHashkey (not absent)

### Phase 20 (2026-09-22): MIMO FORENSIC AUDIT — AUTH provisioning
- **Report:** `dec_work/MIMO_FORENSIC_AUDIT_20260922.md`
- **Key finding:** `gIpAuthVars` (140-byte table) written ONLY by utopia ioctl `0xC08C5506`. `MApi_AUTH_Process` is the only identified caller. **No binary in the pulled ELF set imports `MApi_AUTH_Process`.**

### Phase 21 (2026-09-22): Phase C/D — Corpus synthesis + static closure
- **Reports:** `dec_work/REPORT_phaseC_corpus_synthesis_20260922.md`, `REPORT_phaseD_static_closure_20260922.md`, `REPORT_phaseD1_residual_static_20260922.md`, `REPORT_phaseD2_final_static_residual_20260922.md`

### Phase 22 (2026-09-23): Space Bunny full corpus audit
- **Report:** `forensic_space_bunny_full_corpus_audit_20260923.md`
- **Key corrections:**
  - D.2-A: `HAL_AUDIO_AbsWriteByte` is real mapped-byte store
  - D.2-B: Eight gate readers (not four)
  - D.2-C: `r4` identity partly closed (`B+0x7C4` on proven path)
  - D.2-D: `r14` reaches `desc+0xF0` from `desc+0x38` (element count)
  - D.2-E: `F_45DE0` installs pre-existing pointers (not allocator)
  - D.2-F: ISA error — `movhi 0xC1601C01` decodes to `0x00E026B0`
  - DTS IPID checks ARE in CheckHashkey (not absent)

### Phase 23 (2026-09-24): Downstream closure — F8/FC
- **Report:** `dec_work/forensic_downstream_closure_20260924.md`
- **Key finding:** F8/FC do not have statically provable downstream consumer after `F_4618E`. A/B are local memory buffers. Pointer provenance terminates at return/status boundary.

### Phase 24 (2026-09-24): Independent corpus audit
- **Report:** `forensic_space_bunny_full_corpus_audit_20260924_independent.md`
- **Key corrections:** Additional corrections to D.2: eight gate readers, partial `r4` identity closure, TX dump ring, SND AC-3 mislabel, wire provenance, `+0x2E4` displacement error.

### Phase 25 (2026-09-25): Factory/SPDIF menu mode map
- **Reports:** `forensic_space_bunny_spdif_menu_mode_map_20260925.md`, `forensic_space_bunny_spdif_menu_mode_evidence_20260925.txt`
- **Key findings:**
  - Direct native `MApi_AUDIO_SYSTEM_Control` mode writer exists
  - HDMI-TX sink cache is `_u32CurEdidSupportList` (not `_astHdmiInfo`)
  - `[M_FACTORY]` sections empty; `factory_keypad.ini` absent
  - Factory IR key-filter exists but is IR-only
  - `MI_AOUT_FactorySetAttr` commands are eARC/channel-status (not DTS setters)

---

## Verified findings

### VERIFIED BY BINARY

1. **DTS AUTH gate:** `MDrv_AUTH_IPCheck` reads single bit from `gIpAuthVars` (140-byte table, ioctl `0xC08C5506`). DTS IPIDs `{0xF, 0x3A, 0x12, 7}` are DTS-family (instruction-level confirmed). `0x43d/0x43e` are Dolby-premium flags, NOT DTS gates.

2. **DTS license = AUTH ∩ strap:** `MDrv_AUDIO_Get_DTS_License` = IPCheck(0xf|0x3a|0x12|7) AND RIU `0x112CF0[7]`. Strap read ONLY by `Get_DTS_License`, never by `CheckHashkey` or `SetDecodeSystem`.

3. **EDID gate:** `_MI_AOUT_SetHdmiAutoMode` forces SPDIF PCM when sink EDID lacks DTS-core bit `0x80`. Live in deployed `mik.ko` (unmodified). Codec 9/10/11/23 use DTS-family arm at `0x98A38`.

4. **SPDIF mode dispatch:** `HAL_AUDIO_SPDIF_SetMode` jump table: 0→PcmMode, 1→PcmMode, 2→AutoMode, 3→BypassMode, 4→TranscodeMode. Input 5 normalized to 0.

5. **Decoder_Type(0x30) path:** Reads latch `AV+0x524` (not `0x112E98`). Latch table at `0x45F1C8`: latch 2,3→`0x45F230` (SHM33); 4/0xB→`0x45F654` (SHM36); 0xA/0x10→`0x45F6BC` (SHM38/AUTH); 0x17/0x1B→`0x45F710`; rest→`0x45F7CC` (fail).

6. **AV+0x4D8 gate:** At `0x45F6D0`: `ldr r1, [r1, #0x4D8]`; `sub r2, r1, #1`; `cmp r2, #2`; `bhs` (branch if AV+0x4D8 >= 3). Sole writer: `MDrv_AUDIO_CheckHashkey` (`str r8, [r0, #0x4D8]`). Sole reader in 0x30 path: DTS/SHM38 arm.

7. **Utopia ABI:** `MApi_` marshals `(handle, struct*)` via `UtopiaOpen(0x80000034)+UtopiaIoctl(cmd)`. Cmds: `0x3F` (open), `0x3C` (set), `0xCF` (getinfo). `r4` = AU slot (OpenDecodeSystem return), `r5` = struct pointer. Verbatim forwarding.

8. **`+0x10=0xFF` PASSES gate:** Gate is `[r5+0x10]∉{0,0xFF}` — meaning values OTHER than 0 and 0xFF FAIL. 0xFF and 0 both PASS. Verified on 3 binaries (TD98, TCL libutopia, TCL utpa2k).

9. **`r4 = AU slot` (not codec ID):** Proven by Open-retval data-flow on both TCL and TD98. `r4` is the return value of `OpenDecodeSystem` (slot ∈ {0,1,2,3,6} or −1).

10. **`decoder status -1` NOT a failure signature:** Present in healthy AC3 playback. Removed from causal model. On compress-OFFLOAD path, `-1` = passthrough mode (no decoder instantiated).

11. **`[2172] None` = benign:** System poller + teardown queries. Present for both AC3 and DTS.

12. **True DTS signature:** `[2185] Fail to get DTS codec type! MApi_AUDIO_GetAudioInfo2(eAdecId:0, Audio_infoType_Decoder_Type) failed!` — persistent during DTS playback, never during AC3.

13. **F8/FC are local memory buffers:** A = `W+0x14EE8`, B = `W+0x156E8`. 8+8 field reads, all inside `F_45E52` or `F_4618E`. No queue/DMA/TX consumer after `F_4618E`. Pointer provenance terminates at return/status boundary.

14. **DTS output-pack path not the blocker:** R54 proved gate `0x265F3` is already-FALSE latch re-arm; no firmware-writable node on live DTS output path.

15. **SND +0xF8 consumer:** SND code `0x016f84`: `LD r23, 0xfa(r11)` where r12 = 0x21A000 (SHM window). Reads low halfword of ARM's `+0xF8` = param `0x6e` = AC3/DTS discriminator. Consumer is in SPDIF/output parameter path.

16. **SND +0x250C is AC-3/E-AC-3 preparation:** Not a DTS emitter. No exact DEC→SND edge from A/B to SND `+0x250C`.

17. **Factory `[M_FACTORY]` empty:** Both `Customer_Module.ini` and `MStar_Default_Module.ini` have empty sections. `factory_keypad.ini` absent from corpus.

18. **Direct native `MApi_AUDIO_SYSTEM_Control` writer exists:** Can request SPDIF modes. But OSD/factory/GAM reachability unproven.

19. **DTS decoder RUNS during playback:** DEC region `0x0FE8` live for both codecs. Decode engine is NOT the blocker.

20. **Status block `0x0C00-0x0C07`:** Non-zero for AC3, ALL-ZERO for DTS (20/20 samples). ZERO firmware writers (read-only hardware status). DTS-specific divergence.

---

## Previously corrected findings

| # | Old claim | Correction | Evidence |
|---|---|---|---|
| 1 | `+0x10=0xFF` kills SetDecodeSystem | **DISPROVEN** — `0xFF` PASSES the gate (gate admits {0, 0xFF}) | HANDOFF §2; 3-binary verification |
| 2 | `r4 = codec ID` | **DISPROVEN** — `r4` = AU slot from OpenDecodeSystem return | HANDOFF §3; Open-retval data-flow |
| 3 | `0x112E98/9A/9B` = failing Decoder_Type | **DISPROVEN** — failing call reads latch `AV+0x524`, not `0x112E98` | MASTER §7; HANDOFF §5 |
| 4 | SHM38-for-0xA "all leaves succeed" | **CORRECTED** — 0xA→`0x45F6BC` which has AUTH gate `AV+0x4D8∈{1,2,3}` | MASTER latch table; HANDOFF §6 |
| 5 | F8/FC → physical SPDIF | **DISPROVEN** — no downstream consumer after `F_4618E` | Downstream closure report |
| 6 | SND ring → physical SPDIF | **NOT PROVEN** — no A/B-to-SND edge; `+0x250C` is AC-3 preparation | R54; downstream closure |
| 7 | `0x2E4` global field name | **CORRECTED** — `+0x2E4` is displacement-dependent; cannot globally name one field | Space Bunny independent audit |
| 8 | `decoder status -1` = DTS failure | **DISPROVEN** — present in healthy AC3 | HANDOFF §4; AC3 baseline |
| 9 | `[2172] None` = failure | **DISPROVEN** — benign polling artifact | HANDOFF §4 |
| 10 | Utopia ABI unknown | **CLOSED** — verbatim payload forwarding | HANDOFF §2 |
| 11 | `AV bytes have no writers` | **CORRECTED** — Open stamps slots; only −1/5 slots lack static writers | HANDOFF §3 |
| 12 | `F_45DE0` allocates A/B | **DISPROVEN** — installs pre-existing pointers from input table | Space Bunny independent audit |
| 13 | `0xDFB1` = allocator | **DISPROVEN** — cache invalidate/flush | Space Bunny independent audit |
| 14 | `movhi 0xC1601C01` = `0x1C0126B0` | **DISPROVEN** — decodes to `0x00E026B0` | Space Bunny independent audit |
| 15 | `+0x2E6` license read | **DISPROVEN** — `+0x2E4` after displacement; `lic[3]` mapping unsupported | Space Bunny independent audit |
| 16 | SND `+0x250C` = DTS emitter | **DISPROVEN** — AC-3/E-AC-3 preparation | R54; Space Bunny |
| 17 | `_astHdmiInfo` = HDMI-TX sink bitmap | **DISPROVEN** — `_u32CurEdidSupportList` is the sink cache | Factory menu report |
| 18 | No direct native mode writer | **DISPROVEN** — `MApi_AUDIO_SYSTEM_Control` exists | Factory menu report |
| 19 | DTS IPID checks = 0xB/0xC | **DISPROVEN** — 0xB/0xC are Dolby; DTS group = {0xF, 0x3A, 0x12, 7} | Full license profile |
| 20 | Clean baseline | **DISPROVEN** — pre-expA backup modified | Forensic synthesis §4.2 |

---

## Important disproven hypotheses

1. **F8/FC → DMA bridge:** Exhaustive absence in path. A/B are local buffers, not queue/DMA/TX objects.
2. **ARM-side license → DSP chain:** Diagnostic patches proved ARM getters not in DSP chain. DSP license state is internal.
3. **EDID gate as sole root cause:** EDID gate is real but not the only gate. AUTH gate is upstream.
4. **`/vendor` files as execution surface:** Dormant for documented loader path.
5. **V440 as working-DTS reference:** No runtime evidence; static architectural reference only.
6. **NODTS EDID variants as runtime decoder gate:** SKU/region config only, no code link to decoder gates.
7. **DTS output-pack path as blocker:** R54 proved no firmware-writable node on live DTS output path.
8. **mstsound dispatcher statically locatable:** R52 definitive negative — runtime/pointer-mediated.
9. **Factory menu as DTS unlock:** No verified path from OSD/factory to DTS capability state.
10. **`MTGADEC_MTAUD_SetAudioOutMode` as RAW/BYPASS setter:** 8-byte no-op.
11. **IEC61937 constants as DTS-specific:** Codec-agnostic, symmetric locations.
12. **Probe6 PCM-carrier:** Channel status stays "audio", AVR rejects.
13. **DTS IPID checks = 0xB/0xC:** 0xB/0xC are Dolby; DTS group = {0xF, 0x3A, 0x12, 7}.
14. **Clean baseline:** Pre-expA backup modified.

---

## Current known state

### DTS failure causal model (current best reconstruction)

```
Kodi DTS (RAW PT, offload)
  → MI_AUDIO_Start Codec:9 eRet:0 (VERIFIED)
  → SetDecodeSystem path: struct+0x10=0xFF (VERIFIED)
  → _MApi_ gates: r4∈{5,−1}, [r5+0x10]∈{0,0xFF} (VERIFIED — 0xFF PASSES)
  → AV-byte gate: AV[r4*40+0xFA]≠0 (VERIFIED gate; live value BLOCKED)
  → MDrv_/HAL_SetDecodeSystem/SetSystem2 (CONDITIONAL on gates passing)
  → Latch 0xB stamped (LIKELY: SetNonpcm path; direct read BLOCKED)
  → Decoder_Type(0x30) queried (VERIFIED live: [2185] persistent)
  → DTS SHM38 branch (VERIFIED static schema)
  → AUTH gate AV+0x4D8 ∈ {1,2,3} (VERIFIED gate; live VALUE BLOCKED)
  → SHM38 ∈ [1..7] + sub-table + MAD(0x4D)≠0 (VERIFIED gates; live values BLOCKED)
  → success=2 / failure=0-or-None (VERIFIED live: persistent FAIL)
```

### Key divergence: AC3 vs DTS

- AC3: latch 3 → SHM33 chain → NO `AV+0x4D8` check → success
- DTS: latch 0xA → SHM38 chain → `AV+0x4D8∈{1,2,3}` gate → FAIL (if AUTH not provisioned)

This is the **first statically-proven AC3-vs-DTS divergence** in the entire chain.

### AUTH provisioning state

- `gIpAuthVars` (140-byte table) written ONLY by utopia ioctl `0xC08C5506`
- `MApi_AUTH_Process` is the only identified caller of `UtopiaSetIPAUTH`
- **No binary in the pulled ELF set imports `MApi_AUTH_Process`**
- `vir_hashkey.txt` MISSING from device
- **H1: LIKELY never provisioned; live `gIpAuthVars` = BLOCKED**

### Runtime evidence summary

- **AC3:** Works end-to-end. Pioneer locks DD. No mid-play failures.
- **DTS:** Bytes flow (16.9M frames, 0 underruns), decoder runs (`0x0FE8` live), but `Decoder_Type` query fails persistently → `codec NONE` → silence.
- **Status block `0x0C00-0x0C07`:** Non-zero for AC3, ALL-ZERO for DTS (20/20 samples). Hardware-produced status, zero firmware writers.

---

## Current unresolved state

| # | Question | Status | Resolver |
|---|---|---|---|
| 1 | Live `AV+0x4D8` value | **BLOCKED** | Needs privileged read-only probe or new playback instrumentation |
| 2 | Live latch `AV+0x524` (+0x528 secondary) | **BLOCKED** | Same as above |
| 3 | Live SHM38 value + producer | **BLOCKED** | Same as above |
| 4 | `AUTH_IPCheck(0xF)` runtime result | **BLOCKED** | Same as above |
| 5 | Live `AV+0xD2`/`AV+0x1C2` (r4=−1/5 gate-4) | **BLOCKED** | Same as above |
| 6 | Utopia core `(handle,struct)→(r4,r5)` exact mapping | **CLOSED** (verbatim forwarding) | HANDOFF §2 |
| 7 | DSP byte-arrival proof | **BLOCKED** | Needs DSP trace or AVR lock evidence |
| 8 | First-query client trigger | **BLOCKED** | Needs AudioFlinger/HAL call-order trace |
| 9 | `struct+0x8` in mstsound dispatcher | **BLOCKED** | R52 definitive negative; runtime-mediated |
| 10 | `0x25B7` support clamp live value | **BLOCKED** | Needs privileged probe |
| 11 | `0x109595` body (in gap `0x108000-0x12FFF`) | **BLOCKED** | Needs full AEON decode |
| 12 | `+0x14` runtime value | **BLOCKED** | Needs privileged probe |
| 13 | `0x2512C4` writer | **BLOCKED** | Needs full AEON decode |
| 14 | `0xA868`/`0xA86C` final consumer | **BLOCKED** | Needs full AEON decode |

---

## F8/FC understanding

### What is VERIFIED

- `F_45DE0` installs two pointers: `D+0xF8 = A = W+0x14EE8`, `D+0xFC = B = W+0x156E8`
- `F_45E52` reads D+0xF8/+0xFC and prepares A/B contents (header/bulk transforms)
- `F_4618E` reads D+0xF8/+0xFC and transforms A/B contents (fill, bit-extract, sync/header writes)
- A/B are passed as destinations ONLY to: fill helper `0x130D25` and bit/extraction helper `0x64369`
- No caller consumes a returned A/B value as a new pointer
- No queue/DMA/TX object is populated with the A/B address
- The `+0xF8`/`+0xFC` accesses in later `F_29867` region use a separate base candidate (identity to D UNPROVEN/BLOCKED)
- Standard DEC preserves F8/FC field instructions exactly in corresponding functions
- `F_4618E` has two return paths: early unprepared (`0x461C2`) and prepared normal (`0x462FE`)
- Last proven writes: `0x462F1` (B[...] = 0xE800), `0x462D1/0x462D7` (A/B header writes)

### What is NOT proven

- Any A/B → queue/DMA/TX submission
- Any A/B → SND `+0x250C` edge
- Any A/B → physical SPDIF relation
- F8/FC content is proven compressed DTS (transforms and sync constants verified, semantic payload identity NOT proven)
- Whether F8/FC are license fields (UNPROVEN; no license semantic edge present)
- Whether A/B survive `F_4618E` (DISPROVEN — not reloaded after return)

### First opaque boundary

```
A/B origin:                         VERIFIED
A/B contents and transforms:        VERIFIED
A/B pointer passed to local helpers: VERIFIED
A/B -> queue descriptor:             NOT FOUND
A/B -> DMA/FIFO/TX submission:       NOT FOUND
A/B -> SND +0x250C:                  NOT FOUND
First downstream object:             A/B memory buffers
First opaque boundary:               0x4128D after F_4618E return
```

---

## F0/r12 understanding

### VERIFIED

- `F_04618E` at `0x461AB`: stores `F0=0` on the disabled path (when `D+0x28 != 1`)
- `F_0411A6` at `0x4128D`: `bnei r12, 1, 0x411C9` — branch consumes only `r12`
- `r12` is loaded from `*(r3 + 0xA124)` at `0x41213`
- `0x411C9` is the fail target (F0-alias clear)
- `0x41221` is the return-1 path
- `F_04618E` prologue: `F0=0` disabled at `0x461AB`; `N=ctx+0x38`; memset via `0x130D25`; `0x64369` bit-reads at `0x46210/0x4621E`; sign-extend `r12=0x12`; header/sync writes

### NOT proven

- Semantic meaning of `r12` (only the `F0=0` disabled-path store exists; `F0=N` success semantics unproven)
- Whether `r12` is an output byte count, license value, or other semantic
- The exact live producer of `*(r4+0x10)` (unresolved)
- `0x45F7B` is decoded as an unresolved opcode affecting later register state in the bulk loop
- `0x461E9` is an unresolved short instruction before the `r8`/`r16` sync-offset use

### Correction

- Old claim `F0=N` (success semantics) → **CORRECTED**: only `F0=0` on disabled path is proven; else unproven
- Old claim `F0` = output byte count → **DISPROVEN**: `r14` reaches `desc+0xF0` from `desc+0x38` (element count), not output byte count

---

## DTS SND/ring understanding

### VERIFIED

- DTS manager around `0x250000` in SND image
- AC3 manager around `0x4E6AD4` in SND image
- SND `+0x250C` function is AC-3/E-AC-3 preparation (NOT DTS emitter)
- No exact DEC-to-SND edge from A/B to SND `+0x250C`
- Standard SND image lacks exact `+0x250C` function
- IEC61937 burst preamble constants are codec-agnostic (computed and stored in symmetric locations)
- DTS control block `0x0F40-0x0F50` stays idle during DTS (uses own `SDO_SdifPacker`)
- `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
- Status block `0x0C00-0x0C07`: non-zero for AC3, ALL-ZERO for DTS (20/20 samples)
- `0x25EE1` gate is edge-triggered descriptor-population routine
- `0x854` bits 17|19: AC3 sets, DTS actively clears
- `0x86C` and `0x870`: DTS helpers program these (AC3 never constructs)
- `0xA868`/`0xA86C`: affect output-adjacent HW/shared-memory state, but NO firmware consumer

### NOT proven

- DTS ring write → cursor/fill accounting → `0xB00008xx` → physical SPDIF
- SND ring → physical SPDIF relation
- DTS manager's exact producer/consumer/mailbox access
- `0x109595` body (in gap `0x108000-0x12FFF`)
- `+0x14` runtime value
- `0x2512C4` writer
- `0xA868`/`0xA86C` final consumer

### R54 verdict

The DTS output-pack path in firmware (`0x26270`–`0x2661D`) is **not the blocker**. Gate `0x265F3` is already-FALSE latch re-arm; no firmware-writable node on live DTS output path.

---

## Factory configuration finding

### What the empty `[M_FACTORY]` proves

- Both `Customer_Module.ini` and `MStar_Default_Module.ini` have empty `[M_FACTORY]` sections
- No factory feature assignment is present in the available configuration
- This is consistent with no configured factory-audio feature being exposed by the available image

### What `factory_keypad.ini` reference means

- The string `vendor/tvconfig/config/factory_keypad.ini` exists in the binary (`tools/final_edid.txt:491`)
- The file itself is **absent** from the extracted corpus
- This means the factory keypad configuration was not included in the firmware extraction, or it is generated at runtime

### What can NOT be concluded

- That no factory menu exists (the `mtktvfactory` executable is missing from the corpus)
- That no hidden DTS toggle exists (the factory service implementation is absent)
- That the factory IR key-filter is an audio unlock (it is IR-only)
- That `MI_AOUT_FactorySetAttr` commands are DTS setters (they are eARC/channel-status attributes)

### What CAN be concluded

- The factory service infrastructure exists (`vendor.mediatek.tv.mtktvfactory@1.0-service` observed in retained log)
- The factory menu is MediaTek `DesignMenuActivity` (confirmed by retained log)
- The known factory audio API (`MI_AOUT_FactorySetAttr`) handles communication-audio attributes, not DTS
- A generic HIDL factory bridge exists but its command set is unknown
- The direct native `MApi_AUDIO_SYSTEM_Control` writer exists but its OSD/factory reachability is unproven

---

## Remaining unknowns

1. **The single most valuable bit:** live `AV+0x4D8` (0 vs 1/2/3). Everything downstream hinges on it.
2. **If AUTH passes:** SHM38/MAD/DSP-lock chain becomes the next blocker.
3. **If AUTH fails:** AUTH_IPCheck provenance (why 0 on a DTS-capable box) is the next question.
4. **Physical SPDIF:** No proof that any firmware state produces a physical DTS stream on the wire.
5. **Factory bridge:** The missing `mtktvfactory` executable is the strongest unresolved lead for a no-patch route.
6. **DSP byte-arrival:** No proof that decoded DTS bytes reach any output stage.
7. **`0x25EE1` gate return value:** Not observable through validated mechanism.
8. **`0x109595` body:** In gap `0x108000-0x12FFF`, not decoded.
9. **`+0x14` runtime value:** Unknown.
10. **`0x2512C4` writer:** Unknown.
11. **`0xA868`/`0xA86C` final consumer:** Unknown.

---

## Recommended next investigation targets

Ranked by information value:

1. **Privileged read-only probe** (root adb or debug build): read `AV+0x4D8`, latch `AV+0x524/528`, SHM38, `0x112E98`, MAD status during DTS play. Closes the entire AUTH question in one snapshot. Highest value, lowest effort if root attainable without flashing.

2. **AUTH_IPCheck provenance:** trace `MDrv_AUTH_IPCheck(0xF)` inputs (OTP/fuse/version source) statically + compare against a known-good MStar DTS device image if obtainable. Decides whether `+0x4D8=0` is fixable in software at all.

3. **Factory bridge investigation:** if the missing `mtktvfactory` executable becomes available, trace the generic HIDL command bridge and the native `MApi_AUDIO_SYSTEM_Control` writer. This is the strongest no-patch route candidate.

4. **DSP lock analysis:** map DTS-parser→lock→SHM38 producer in AEON firmware (function addresses, lock field); correlate with `dts_parser` live activity. Needed only if (1) shows AUTH passing.

5. **TCL vendor-binary diff** (no new extraction needed): compare TD98 vs TCL `HAL_AUDIO_SetSystem2`/SHM38-arm/AUTH-gate presence; TCL likely lacks the `+0x4D8` gate (not found in examined regions) — a second confirmation of the divergence.

6. **Runtime SRAM-mirror capture** of `mem[0x8(r10)]` at the `0x1F6DA` test point (R52 §5). Would decide P-a vs P-b for the mstsound gate. Requires device.

7. **Wire capture during DTS playback** with AVR lock test. Would provide ground truth for whether any DTS bytes reach the physical SPDIF output.

---

*End of corpus review report. No binaries were patched, modified, or deployed during this investigation.*
