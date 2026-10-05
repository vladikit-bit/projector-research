# MASTER FORENSIC REPORT — Current Muse Chat Session (DTS passthrough, Thundeal TD98 Pro / C50A)

Date of record: 2026-09-19. Status: READ-ONLY investigation, no patches, no firmware writes, no config changes, no module reloads. Playback runs No.1–5 were user-authorized state changes (media push + VIEW intents); everything else strictly read-only.

> Provenance rule for this document: **CURRENT SESSION** = this Muse chat only.
> **EXTERNAL / HISTORICAL UNION-ALPHA ARCHIVE** = `C:\firmware_temp\Бекап розслідування\GLM_archive_20260918\` (prior GLM/stealth session, snapshot 2026-09-18T20:03:27Z). Archive is used ONLY as cross-check; its 12 subagents are NOT counted in this chat's agents.

---

# PART I — MASTER REPORT ON THIS CHAT

## 1. SESSION SCOPE / PROVENANCE

- **Subject:** this current AI-assisted forensic RE session (OpenCode harness, Muse Spark model `muse-spark-1.3-contributor-free`), working directory `C:\firmware_temp`, device `192.168.0.183:5555` (adb `-P 5038`).
- **Time range:** 2026-09-18 (inheritance + v4 report + archive re-read) through 2026-09-19 ~19:38 EDT (device clock; last runtime run). Prior session summary referenced a `forensic_trace_autonomous_20260918_v3.md`; this chat produced v4 and follow-ups.
- **Available in-chat material:** full tool-call transcript (Read/Grep/Glob/Bash/Write/Edit/TodoWrite/Task), 12 own Task-subagent sessions (IDs below), created files under `C:\Users\k0994\AppData\Local\Temp\opencode\` (analysis scripts `v4_*.py syms.py disas.py funcref.py mikcodec.py mikdis*.py sodisas.py mkreq.py chk*.py`, `shot.png` pulled then device copy removed) and `C:\firmware_temp\aeon_validate\dec_work\forensic_trace_autonomous_20260918_v4.md` (written by this chat).
- **Imported from Union Alpha/Stealth predecessor:** problem statement (native DTS passthrough broken), baseline hashes, DSP listing `dec32_clean.txt` + `dec_full.bin`, stock `.ko` baselines, test media in `spdif_audio_investigation/testmedia/`, `kodi_rpc.sh` pattern, Kodi guisettings copy, and the whole GLM archive itself (external).
- **Own findings of this chat:** bitstream-reader field layout `0x64369`; `ctx+0xEC=6` store in `F_45DE0`; `F_028AFE`/`F_014F63` output-path verification incl. dual-entry `0x288D9` + second tail `0x1476A`; dead `r7`-bridge proof; `R+0x158A8 ≡ ctx+0x3C` identity; `R+0xD9C=0` writer; slot pass-through proof; `Decoder_Type(0x30)`=latch (not `0x112E98`) discovery with full fail-condition enumeration; `_MApi_` gate stack; `MapDecoderType` table (`9→0xB/5→3/invalid→0x1F`); START-struct `+0x10=0xFF` byte-verified chain; OpenDecodeSystem no-stamp proof; all 5 runtime playback runs + OFFLOAD/`-1`/GetCodecType-fail evidence; actor map (`mstar.hardware.audio.service`); contamination quarantine procedure.
- **Merely inherited (not re-derived here unless noted):** register base addresses, B=`0x22500000` algebra, selector/CheckHashkey table, builder byte-identity, TEE/SHM33 details, toolchain caveats.

## 2. AGENTS / SUBAGENTS IN THIS CHAT (N=12, established from Task records)

| # | Task ID | Role / assignment | Inputs analyzed | Key findings | Errors → corrections | Integration |
|---|---|---|---|---|---|---|
| T1 | ses_f4d9372f… | Callers of F_03E961 | dec32_clean.txt | Single caller `0x37687` in F_0375BE; F_03E961 tail has no F8/FC/F0 reads, status-only returns | none | full |
| T2 | ses_f4d8ee08… | F_0375BE caller chain | dec32_clean.txt | Single caller `0x29A9F`; root-context map; 4 output slots on success path | none | full |
| T3 | ses_f4d88901… | DSP HW output interface | dec32_clean.txt | MMIO `0xB000081C/838/83C`+`0xB000000A`; DMA setup; F_028AFE dispatch; r10+0xD4… slots | MISSED second tail (`0x1476A`) + misattributed `0x29DEA→0x288D9` as F_028AFE → corrected by me (dual entry of F_02863D) | partial → corrected |
| T4 | ses_f49c6c56… | Read archive chunks 001–002 | GLM 001/002 | Early-phase reconstruction (onboarding, 0x112E98, hashes, IEC captures) | none | full |
| T5 | ses_f49c6c50… | Read chunks 003–004 | GLM 003/004 | SetSystem2/SHM33/audit-phase reconstruction; noted mmap/B/TEE absent there | none | full |
| T6 | ses_f49c6c4e… | Read chunks 005–007 | GLM 005/006/007 | B/mmap/TEE/builder/0x411A6/stopping-point reconstruction | none | full |
| T7 | ses_f49c6c4b… | Read all subagent trajectories | 13 files | Per-agent roles (truncated in delivery) | TRUNCATED → I completed missing agents by direct reads | partial → completed |
| T8 | ses_f49c6c49… | Spot-check raw extracts | raw_db_extract/raw_jsonl/SHA | All counts OK; **live_snapshot 35 rows vs documented 28 (MISMATCH)**; hashes 4/4 match | none (finding itself) | full |
| T9 | ses_f492764d… | P1 .so DTS setup chain | audio.primary.mt5889.so | PLT map; call sites; `mi_decoder_open` DTS→9; Parser_Open/Flush 0 sites; offload flag bit | none material | full |
| T10 | ses_f492764b… | P2/P4 slot + version in .so | same .so | No slot constants on DTS path; NO AV+0x4D8 mechanism (VERIFIED absence) | over-strong "pure offload skips all" (live flags are 0x11, so 0x606 runs) → corrected by me | partial → corrected |
| T11 | ses_f4927649… | P3 DTS vs AC3 | same .so | Shared setup path; codec values 9 vs 5; SPDIF immediates differ | none | full |
| T12 | ses_f4927648… | P5 ordering/init-race | .so + both .ko | Reset-before-stamp only in Resume; Start Get-before-Set; cached-stale branch; no async proof → F3b UNRESOLVED | none | full |

### HISTORICAL EXTERNAL AGENTS — UNION ALPHA ARCHIVE (not counted above)
12 sessions: ad256c2b (reports locator), dd885e7d (SetSystem2/SPDIF audit), c5d9b117 ×2 files (DSP/SDO history), 3866b9db (RIU scratch), 64d2c412 (DEC images), 9f05c8e9 (utpa2k focus), fc9171c8 (SetDspBaseAddr producer), 7280ff60 (mmap 0x13B→MI_MAD_ADV_BUF), c15b682b (SHM33 slots), b1eab034 (selector), 4a1b4982 (TEE chain), ba3dda29 (output handoff → stopping boundary). Used only as cross-check/provenance.

## 3. INVESTIGATION HISTORY (this chat only)

1. **Inheritance + corrections:** opened from prior summary (F_45DE0 direction, F8/FC provenance, A/B downstream). First act: independent re-verification; corrected `r13=r4/r10=r3` roles; traced F8/FC writers to `ctx+0x4E40/0x4E44` self-referential stores.
2. **DTS internals:** identified `0x64369` bitstream reader (struct +0x0/+0x4/+0x8), `F_04618E` frame-header parser role, DTS sync words `0x7FFE/0x8001`, sample-rate tables; wrote v4 report (5-level chain down to `F_028AFE→F_014F63→DMA/MMIO`).
3. **Archive stage (user-directed):** full independent re-read of GLM archive (7 chunks + 13 trajectories + raw + hashes) via T4–T8 + direct reads (`ba3dda29` fully, `4a1b4982` partially, tails of others); produced reconstruction + self-audit; found archive never traced `F_028AFE/F_014F63` itself (only stage-2 spec) → downgraded v4 HW-path claims to my-session evidence.
4. **Bridge hunt:** verified `jal F_028AFE@0x29D1B`, `r7=*(r10+0xF8)` → PROVED dead (zero reads in `[0x28AFE,0x28BC9)` before overwrite); found dual-entry `0x288D9` + tail `0x1476A`; output slots are POINTERS (`0xACC0/0xBA0/0xCE8/0x158A8`); exact identity `slot3 = ctx+0x3C`; Block A/B validation mirror; `ctx+0xEC=6` writer found; no F8/FC→DMA data bridge (VERIFIED ABSENCE in path).
5. **Runtime (user-approved, 5 runs):** idle baseline → push `test_dts_51.mp4` → Kodi intents (chooser incident + Kodi force-stop, disclosed; Kodi restarted) → JSON-RPC dead (8080 held by uid-0 non-HTTP) → screenshot proved playback → asound-blindness discovered → OFFLOAD thread found (`AudioOut_5D`, DTS `0xB000000`, FrmRdy, 0 underruns) → `MI_AUDIO_Start Codec:9` + `GetCodecType fail` + `codec NONE` + `status -1` + silence (user-confirmed) → dense early-window runs fixing fail at Start+14–18 s mid-play.
6. **Static closure:** `Decoder_Type(0x30)`=latch discovery; fail-condition enumeration (F1 excluded, F2/F3 live); slot pass-through proof; `_MApi_` gate stack; `MapDecoderType` table; START `+0x10=0xFF`; OPEN no-stamp proof; `.so` has no slot/version mechanism; actor map single process; contamination quarantine (mik/utpa2k file confusion → file-script re-verification); release-latch `1→0` correction.
7. **Current model:** dead stamp/auth chain by construction (below §18).

## 4. TOOLS / METHODS ACTUALLY USED

- **Python (extensive):** ELF parsing (pyelftools), Capstone disassembly (ARM host code; Thumb for .so), raw-byte scans (struct.unpack jump tables/STR encodings/BL targets), log arithmetic. Verified independently by: byte-level spot checks, cross-tool agreement (capstone vs raw decode), archive cross-check. Pitfalls hit: PowerShell mangling of long `python -c` one-liners (quote stripping, `head` absent) → migrated to file scripts; linear-sweep desync on embedded jump tables → targeted reads + byte scans; wrong-file reads (mik vs utpa2k) → quarantine + file-based re-verification (chk*.py).
- **Read/Grep/Glob (this harness):** primary listing navigation; Grep used for address/pattern search (regex, with false-positive awareness for offset-coincidence across structs — explicitly handled via base-provenance rule).
- **ADB (`-P 5038`, `192.168.0.183:5555`):** connect, push/pull, shell reads (`cat /proc/asound`, `dumpsys`, `logcat -d`, `dmesg`, `ls`, `ps`, `getprop`, `screencap`, `am start`), `dumpsys media.audio_flinger` (key instrument). Never: `logcat -c`, writes outside `/sdcard/Movies`+`/data/local/tmp`, settings, reloads, `/dev/mem`.
- **logcat/dmesg/dumpsys//proc:** baselines + 5 playback runs; thread-name drift handled (`5D→6D`).
- **Hash verification:** `md5sum` on device (stock match), `SHA256SUMS.txt` recheck (4/4), byte-identity checks.
- **screencap:** one screenshot proving playback (then removed from device).
- **NOT used in this chat:** Ghidra, Reko (referenced only via archive/toolchain docs), firmware extraction (used existing corpus), custom OpenCode skills (none invoked), WebFetch/WebSearch, TEE/pkg tooling, decoders beyond consulting `aeon_ORBIS32.sinc` semantics (`bg.ble`, `jal/j`, `sw_0`, `cmov_*`).
- **Known issues (this chat):** Ghidra `bg.jal→0x130CEA` bug is ARCHIVE knowledge, not re-encountered; Capstone desync on jump tables (mitigated); PowerShell-vs-Python quoting (mitigated via files); wrong-file contamination (mitigated via chk-scripts + size assertions); `Select-Object` instead of `head`; `mov r0,#0x80000034`-style Utopia constants read as-is.

## 5. ARTIFACT CORPUS (current chat usage)

- `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` — primary DSP listing (637122 entries + 8-line header; byte-matched `dec_full.bin`). Read-only. All DSP findings.
- `dec_full.bin` — DSP image (SHA256 `530bfa68…` per archive). Matched only.
- `C:\firmware_temp\patch_baseline\utpa2k_stock.ko` — MD5 `2fc6e9fc46b6402d73aa6f84f9599d46` (also live on device). All ARM host RE. Unmodified.
- `C:\firmware_temp\patch_baseline\mik_stock.ko` — MD5 `c1421040dbcfff415f9417f482189435` (also live). MI-layer RE. Unmodified.
- `C:\Users\k0994\AppData\Local\Temp\opencode\audio.primary.mt5889.so` — pulled 265604 B from `/vendor/lib/hw/`. Host HAL RE (Thumb). Local copy only; device file untouched.
- `C:\firmware_temp\spdif_audio_investigation\testmedia\test_dts_51.mp4` (18345456 B) — pushed to `/sdcard/Movies/` (user-authorized; remains). `.dts/.ac3/.eac3` siblings listed, not used.
- `aeon_ORBIS32.sinc` (+ `aeon.slaspec` lineage) — semantics reference only.
- `kodi_rpc.sh` — read as pattern reference (its `nc ::1 8080` path verified DEAD: port held by uid-0 non-HTTP service); `kodi22_guisettings.xml` — webserver=true:8080 on file, not effective.
- `mst_codec_r2*.bin`, `r2img/`, other baselines — referenced for provenance, not re-analyzed here.
- Created: `forensic_trace_autonomous_20260918_v4.md` (this chat), `C:\Users\k0994\AppData\Local\Temp\opencode\*.py` (analysis scripts), `shot.png` (pulled, viewed, device copy removed), `krpc_ping.txt` (temp), this master report file.
- GLM archive — external read-only cross-check (Sec. IV).

## 6. DEVICE / SYSTEM CONTEXT (facts only)

- Thundeal TD98 Pro / C50A, MT5889 (MStar), Android 11, Linux 4.19.116 armv7l (SMP PREEMPT, Sep 10 2025 build), uptime ~3:49→5:49 across session, load ~60+.
- Audio: `MStar-MAD-No00` card0 (`pcm0-3p/7p/0-2c`; NO compr nodes; NO debugfs); card2 DMIC. Stock `utpa2k`/`mik` loaded with dependents.
- Userspace: `mstar.hardware.audio.service` (/vendor/bin/hw, uid 1041, PID 308 + HwBinder threads 406/407/726 + worker proc 2249 "writer"); `audioserver` PID 331 (AudioFlinger); `system_server` 936; Kodi `net.kodinerds.maven.kodi22` (+ `org.xbmc.kodi` installed) — PIDs 28579 → 28920 → 29402 → 1650 across kills/restarts.
- Kodi 21–22 era: OMX `MS.DTS.Decoder` + `MS.Passthrough.Decoder` enumerated; passthrough device `AUDIOTRACK:AudioTrack (RAW)|Android IEC packer`; framework `AudioCapabilities` lists DTS/Passthrough mimes unsupported.
- External chain: SPDIF/HDMI/AVR present by project premise; acoustics per user: silence during DTS playback. No AVR lock evidence collected.

## 7. STATIC DISCOVERY — COMPLETE LEDGER

### Decoder / RIU (host ARM, utpa2k_stock.ko; all fresh-verified this chat unless noted)
- `0x112E98` (decoder-type byte), `0x112E99` (cmd), `0x112E9A` (slot-2 alias), `0x112E9B`. Decomposition `type&0x7F/cpu>>7` (archive; consistent).
- `HAL_AUDIO_SetSystem2@0x44D0B0/0x85C`: addr select (`r5==2→0x112E9A`); DTS branch `@0x44D1DC`: `AV+0x4D8==3 ? 0x97 : 4`; latch `0xA` primary `@0x44D27C→0x44D8F8` / secondary `@0x44D218→0x44D6F0`; AC3 latch 3; default `0x1C`; **`r5==1` early-return with NO stamp**; `r5==2` secondary only.
- `HAL_MAD_GetDTSInfo@0x464980`: `read98 &0x1F, cmp 4` then `Get_SHM_INFO(1)` — legacy path, NOT the failing one.
- `GetAudioInfo2` chain: `MApi@0x3FD0B0` (ioctl `0xCF`) → `_MApi@0x3ED960` → `MDrv@0x427910` (via `g_FuncPrt_Hal_GetAudioInfo2` := `HAL_MAD_GetAudioInfo2`, installed in `HAL_MAD_Init` — VERIFIED reloc) → `HAL_MAD_GetAudioInfo2@0x45E34C` (infoType jump table).
- **`Decoder_Type = infoType 0x30`** (VERIFIED: both `_MI_AUDIO_GetCodecType` queries carry `r1=0x30`; case target `0x45EA5C`): `fp=Convert_DecId_to_ADECId(0)=0` → reads **primary latch `AV+0x524`** → jump table on `(latch−2)`, base `0x45F1C8=pc+8` (corrected after own off-by-8 error): latch 2,3→`0x45F230`; 4/0xB→`0x45F654`; **`0xA/0x10→0x45F6BC`** (needs `AV+0x4D8∈{1,2,3}` + `SHM38∈[1,7]`, then sub-dispatch — all sub-cases return 1); 0x17/0x1B→`0x45F710`; rest/out-of-range→`0x45F7CC` fail-with-log (`Unknown eDecoder (%d)…`, prints latch — never seen live, Utopia-gated). **`0x112E98 is NOT read in this path`** (only case `0x3C` + legacy GetDTSInfo read it).
- `Convert_DecId_to_ADECId@0x44485C`: eAdecId=0→0 (VERIFIED; F1 excluded).

### Authorization / version
- `MDrv_AUDIO_CheckHashkey@0x423494/0x16FC`: `str r6,[AV+0x4D0]@0x424870`, `r8∈{0,1,2,3}` → `AV+0x4D4/0x4D8`, latch `+0x503` (archive; consistent with re-reads). `AV+0x4D8` = DTS version (0 none … 3 DTSX); writers: CheckHashkey only (in-image). `HAL_AUDIO_CheckHashkeyDone` = stub. `[dts] ver(%d)` MdbPrint reads it (string verified; never seen live).

### DSP image / mmap / TEE / SHM (mostly archive-inherited, spot-checked)
- `B=0x22500000` via `MI_SYS_GetMmapLayout(0x13B)→MI_MAD_ADV_BUF→SetDspBaseAddr(2)→AV+0xA0/A4` (archive VERIFIED; not re-derived here).
- `SHM 0x33` (hex): host formula verified in archive; DSP-side relation: NO PROVEN RELATION (kept). Not pursued (correctly secondary).
- TEE cmd3 / image selector (`+0x4D0==4→MS12V22`): archive; loader-branch≠selector correction kept. Not pursued.
- Builder `[0x45D07,0x45DE0)` 217 B = std `0x50AFE` (archive; accepted). Callers `0x45FC9/0x460EF/0x4616A(r4=0→0x0B)/0x46186` (accepted).

### DSP DTS path (dec32_clean.txt; mine unless noted)
- `F_03E0DD@0x3E0DD–0x3E392`: prologue `r14=r3/r11=r4/r10=r4+0x10000/r12=r5`; 5 callers (`0x37810/0x37986/0x37A61/0x3F1AE/0x3F7D2`); DTS invocation via `F_03E961@0x3F1AE` with `(R, desc, *(R+0xACC8))`.
- `F_45DE0`: `F8=*(t+0)→ctx+0xF8 @0x45E17`, `FC→ctx+0xFC @0x45E2F`, **`ctx+0xEC=6 @0x45E3D/0x45E3F`** (my correction), misc `0x3E/0x40/0x7C/0x80` writes.
- `F_45E52@0x45E52` (×3 from `0x411A6`), `F_45FD1@0x45FD1` (parser init), **`F_04618E@0x4618E`**: `F0=0` disabled `@0x461AB`; `N=ctx+0x38`; memset via `0x130D25`; `0x64369` bit-reads `@0x46210/0x4621E`; sign-extend `r12=0x12`; header/sync writes.
- **`0x64369` bitstream reader**: state `ctx+0x0(ptr)/+0x4(off)/+0x8(remaining)`; returns 1/0 (mine).
- `F_0411A6@0x411A6`: `0x41283→0x45FD1`, `0x41289→0x4618E` (same `root+0x14CCC` descriptor), `0x4128D r12-gate→0x411C9` (F0-alias clear), return-1 `0x41228`. Upstream single caller `0x3EF64` (T1).
- `F_0375BE` (single caller `0x29A9F`): success slots `*(r22)=*(R+0xACC0)`, `*(r21)=R+0xBA0`, `*(r20)=R+0xCE8`, `*(r19)=R+0x158A8` (`@0x376D9–0x376F7`, null-guarded) — POINTERS, not data.
- `F_03E961`: `0x3EF64→0x411A6`; `desc+0x1FC(R+0xD9C)=0 @0x3EF4B` (cond `[0xA50]≠1`); `desc+0xB4=0 @0x3EF56` (cond); returns status.
- `F_029867@0x29867` (no direct caller — indirect entry VERIFIED ABSENCE): `R=r3`, `r11=*(R+0)`, MMIO IRQ `0xB000081C/838/83C` + `0xB000000A`; `jal F_0375BE@0x29A9F`; slots reload; struct0 gates/validation (Block A `0x29BE9`); `jal F_028AFE@0x29D1B` (`r3=r11,r4=1,r5=r14,r6=r20,r7=*(R+0xF8)`); second entry `jal 0x288D9@0x29DEA` (F_02863D interior, `r3=r21`); Block B `0x2A11C` on r19.
- `F_028AFE@0x28AFE`: `0xB0(r5)` channel dispatch + traps; tails `r4≠0→0x28B57: jal 0x14F63` / `r4==0→0x28BBA: jal 0x1476A` (16-caller generic). **Incoming `r7` never read before overwrite** (full-range scan) → r7-bridge DEAD.
- `F_014F63`, `F_02863D@0x2863D`, geometry gate `0x460C2` (`bg.ble (N<<5) vs (L+0x80)`; polarity left AMBIGUOUS per archive task 309 — my earlier "correction" downgraded to partial).
- Identity (exact arithmetic VERIFIED): `ctx = R+0xBA0+0x14CCC = R+0x1586C`; `ctx+0x38=R+0x158A4`; **`slot3 = R+0x158A8 = ctx+0x3C`**; `ctx+0xF8=R+0x15964`.
- Sample rates, `0x7FFE/0x8001` + `0x1FFF/0xE800` sync words, `0xBB80`=48000 defaults.

### MI/HAL/Utopia (mik+utpa2k; file-verified)
- OPEN: `MI_AUDIO_Open@0xA256C` → `MApi_AUDIO_OpenDecodeSystem(handle, struct)` (0xA29F4) → Utopia cmd **`0x3F`** → `_MApi@0x3E56E0` (gates `[r5+0x10]∉{0,0xFF}`, `[r5+3]≠0`, `[r5+0xC]∉{5,3}`) → `MDrv@0x405CC8` (`bx gOpenDecodeSystemFuncPtr`) → `HAL@0x43FF20` (decID/priority, `struct+0x18` write). **No latch/SetSystem2/CheckHashkey anywhere in open chain** → open-time stamp DEAD by construction.
- START: `MI_AUDIO_Start@0xA437C` → `MApi_AUDIO_SetDecodeSystem(handle, struct=sp+0x60)` (0xA4858) → Utopia cmd **`0x3C`** → `_MApi@0x3E52A8` (gates `r5≠0`, `r4∈{5,−1}`, `AV[r4*40+0xFA]≠0`, `[r5+0x10]∉{0,0xFF}`) → `MDrv@0x424E78` → `pFuncPtr=HAL_AUDIO_SetDecodeSystem` (init-installed, VERIFIED reloc) → `HAL@0x4407F4` (r5 passthrough VERIFIED) → `SetSystem2(r5=slot)`.
- `struct+0x10`: OPEN = `MapDecoderType(global[sl])` ∈ {`0xB`(←9 DTS), `3`(←5), `0x1F`(invalid), 0(skipped)} — static mapper `0xA33E0` fully table-read; START = overwritten with `[sp+0x18]=[sp+0x24]=**0xFF**` (single store `0xA44BC`, byte-verified `e3a000ff…e58d1070` chain) — **gate-killer for `_MApi_`**.
- `AV+0x1C2/AV+0xD2` (AV-byte gate): **zero writers** in image (byte-scan of STRB/STR imm across cond/rn/rd) → gate fails if reached.
- `_MApi_` reachable ONLY via Utopia dispatcher (zero direct BL — file-verified scan); dispatcher arg mapping (kernel Utopia core, outside both modules) = final static boundary.
- `_MI_AUDIO_GetCodecType@0xB3BE8` (static, mik): `infoType 0x30` ×2 queries; cached-stale branch; callers `MI_AUDIO_GetCodecParams` (0xA8BEC/0xA8E68) + `MI_AUDIO_GetAttr` (0xAA3F4/0xAB694). `mi_audio_SetCodecType@0xB07F8` (slot/codec globals, monitor/ioctl paths only).

## 8. CURRENT OPEN PATH (reconstruction)
`MI_AUDIO_Open` (validate/mutex/memset) → mapper-gated `MapDecoderType(global[sl])→struct+0x10` (DTS input 9→`0xB` IFF global holds 9; init-0→`0x1F`; skipped→0) → `MApi_OpenDecodeSystem(handle, struct)` → Utopia `0x3F` → `_MApi_` struct-gates → `MDrv` → `HAL_OpenDecodeSystem`: decID select + priority + `struct+0x18`. **Latch untouched end-to-end.** `CheckHashkey` untouched. `SetSystem2` untouched.

## 9. CURRENT START PATH (reconstruction)
`MI_AUDIO_Start` (validate) → `GetDecodeSystem` fills struct → **local overwrite `struct+0x10=[sp+0x18]=[sp+0x24]=0xFF`** → `MApi_SetDecodeSystem(handle, struct)` → Utopia `0x3C` → `_MApi_`: `r5≠0 ✓` → `r4∈{5,−1}` (dispatcher-dependent) → `[r5+0x10]∉{0,0xFF}` (**0xFF FAILS here**) → AV-byte (no writers → would FAIL) → `MDrv_` → `HAL(slot=r4-chain)` → `SetSystem2`. Reaching `MDrv_` requires passing ALL gates; MI-provided values fail at least the `+0x10` gate deterministically.

## 10. STRUCT +0x10 (ledger)
- OPEN: `= MapDecoderType(...)`: DTS 9→`0xB`, AC3-family 5→3, invalid→`0x1F`, skipped→0. Gate needs ∉{0,0xFF}: 0xB/3/0x1F pass; 0 fails.
- START: `= 0xFF` (exact store chain byte-verified) or entry-stale (single store in function; bypass paths leave entry value — entry memset of that slot not established). Gate needs ∉{0,0xFF}: **0xFF deterministically fails**.
- Open-derived decoder info is discarded/overwritten before the stamp-relevant call.

## 11. UTOPIA ABI (proven vs boundary)
- Proven: `MApi_` marshals `(handle, struct*)` via `UtopiaOpen(0x80000034)+UtopiaIoctl(cmd)`; cmds `0x3F` (open) / `0x3C` (set) / `0xCF` (getinfo); `_MApi_` handlers take `(r4, r5)` with the gate stacks above; zero direct BLs to `_MApi_` (dispatcher-only).
- NOT proven (exact boundary): kernel dispatcher's `(handle, struct) → _MApi_.(r4, r5)` unpacking (Utopia core outside analyzed modules). All downstream conclusions are therefore stated CONDITIONAL on dispatcher forwarding, with the independent kills (`+0x10`, AV-bytes, latch-set) holding under every consistent reading.

## 12. DECODER-TYPE READ PATH (complete)
`_MI_AUDIO_GetCodecType` (mik, static 0xB3BE8; adecId from `[r4+0xADC]`, dmesg eAdecId:0) → `MApi_GetAudioInfo2(0, 0x30)` → ioctl `0xCF` → `_MApi` → `MDrv` (via g_FuncPrt) → `HAL_MAD_GetAudioInfo2@0x45E34C` → case `0x30@0x45EA5C` → `fp=Convert(0)=0` → `r8=AV+0x524` → table `(latch−2)` [latch 2,3→SHM33-chain; 4/0xB→SHM36-chain; 0xA/0x10→SHM38+`AV+0x4D8∈{1,2,3}`-chain (all SHM38 leaves return 1); 0x17/0x1B→0x45F710; else→`0x45F7CC` fail+log]. Return: `r4` (1 success / 0 fail); `_MI_` treats 0 as fail (`None` [2172] → `Fail DTS` [2185] printks).
- Failure ⟺ {`AV+0x4D8∉{1,2,3}`} ∪ {latch→`0x45F7CC` entries/out-of-range} (Convert-fail and SHM38-fail EXCLUDED by construction).

## 13. RUNTIME FORENSICS (all runs, read-only except authorized playback)

Conventions: wall clock EDT 2026-09-18/19; kernel-time→wall anchor via `uptime` (boot≈13:32:30–13:32:59, ±60 s slop acknowledged). asound `/proc` reads generate permissive SELinux denials (allowed).

- **Baseline (17:22):** all `pcm0-3p/7p/0-2c` closed (status+hw_params); stock module hashes match; no debugfs; dmesg MI3 idle + `MI_PCM_Close hPcm:0x81000000`; logcat 21633 lines (Kodi DTS/OMX enumeration 17:17:08; `AudioCapabilities: Unsupported mime audio/vnd.dts(+hd/passthrough)`).
- **Media prep:** pushed `test_dts_51.mp4` (18345456 B) → `/sdcard/Movies/` (remains, authorized). `kodi_rpc.sh` path DEAD (`nc ::1 8080` refused; :8080 held by uid-0 non-HTTP service; Kodi uid 10051 has no :8080 listener; saved guisettings webserver=true not effective).
- **Run 1 (17:27:56–17:29:26, Kodi 29402):** explicit VIEW intent (after implicit-intent chooser incident that force-stopped Kodi 28920 — disclosed; Kodi restarted). `SyncDTS (6ch/48kHz/syncword 0x7ffe8001/framesize 2012)` → sink `AE_FMT_RAW/RAW(PT)/STREAM_TYPE_DTS_512` (17:27:56); prior GUI sink PCM float 48 kHz. Screenshot proved video render (01:00/01:30, removed afterward). EOF 17:29:26 → PCM teardown sink + `AudioTrack stop (16.9M frames)` + `dts_parser_flush` (pid 308). asound polls missed window (closed).
- **Run 2 (17:30:55–~17:32:25):** new RAW sink 17:30:55; `AudioOut_5D` handle 93 (OFFLOAD DIRECT|COMPRESS_OFFLOAD, DTS `0xB000000`): active track Id 112 (client 29402=Kodi), `FrmRdy 19904/32192`, `Underruns 0`, **`decoder status: -1`**, HAL `codec type: NONE`; `MI_AUDIO_Start Codec:9 hAudio:0x19000000 eRet:0` (14xxx s); GetCodecType-fail line @14382ks (≈17:32:44, post-EOF).
- **Run 3 (17:37:00–~17:38:30):** RAW sink 17:37:00; fails @14738/14746/14757ks (≈17:38:40/48/59, post-EOF); Stop→Start(Codec:9)→Stop churn @14762ks.
- **Run 4 (19:16:39–~19:18:09, Kodi 1650):** sink 19:16:39; Start @20651ks (19:16:41, +2 s); **fail triplet @20665ks (19:16:55, +14 s, MID-PLAY)**; reopen Start @20741ks (19:18:11); Nones @20747/20761ks.
- **Run 5/final (19:21:59 intent):** Start @20972ks (19:22:31); **fail @20990ks (19:22:49, +18 s, MID-PLAY)**; early poll: fresh thread `Frames written 3293184`, active track, status −1 from first sample; thread renamed `5D→6D` (handle 109) — poll-by-type lesson recorded.
- **Invariant across runs:** sink RAW-PT always opens; `Codec:9` always starts OK; Decoder_Type fails (mid-play AND teardown); `codec NONE` + `status −1` always; frames written (16.8–16.9M) with 0 underruns; user-confirmed silence; asound `pcmXp` closed even mid-play (offload bypasses ALSA nodes — explains baseline blindness).
- **Actor PIDs:** 308 = `mstar.hardware.audio.service` (PPid 1, uid 1041) incl. `dts_parser` tag; 406/407/726 = its HwBinder threads (NOT separate clients — corrected mid-investigation); 2249 = same binary ("writer", PPid 1, 15 threads); 331 = audioserver; 936 = system_server; 29899 = dead/unattributable; 658 = kernel AOUT thread. `/proc/308/fd` blocked (SELinux).
- Post-session state: asound closed (=baseline), Kodi alive, only test video added, screenshot removed, no `logcat -c`, no reloads, no `/dev/mem`.

## 14. ACTOR / PROCESS ANALYSIS
`Kodi → AudioFlinger(audioserver:331) → audio HAL service(308+threads) → MI_*(mik.ko) → MApi_/Utopia → utpa2k.ko → (DSP)`. Second binary: same-binary worker (2249). OMX/codec2 side: enumerated but not traced (no evidence it issues the failing query; GetCodecParams has no caller in the HAL .so — the _MI_ queries originate from mik-API clients: HAL service and/or mediaserver via ioctl; exact first-query client unresolved → BLOCKED micro-point, irrelevant to gate verdicts). PID≠client lesson recorded.

## 15. DSP F8/FC → DMA INVESTIGATION (separate branch, CLOSED as causal explanation)
Proven arrows: compressed stream → `F_45E52` prep → `F_45FD1` → `F_04618E` bit-extraction into F8/FC + sync/header writes (all inside descriptor `root+0x14CCC`); `F_0411A6` returns status only. Output side: `F_0375BE` slot-pointers → `F_029867` validation/dispatch → `F_028AFE` (dual entries, dual tails `0x14F63`/`0x1476A`) → MMIO/DMA regs.
NOT proven / absent: ANY store/load moving F8/FC values or data into output structs (exhaustive offset+base scans in chain); `r7=*(R+0xF8)` discarded before use (full-range scan); slot3 (`ctx+0x3C`) validated-not-forwarded (Block B); `R+0xACC0` pointer + `R+0xD9C/R+0xDA4` fields unwritten in image (host-side ⇒ host-boundary, BLOCKED beyond); `ctx+0x58…` counters unwritten (Block B vacuous-pass claim REJECTED).
Why no longer primary: physical symptom (mid-play Decoder_Type fail + codec NONE) is fully explained upstream at the HAL type query; DSP compressed-output path was never reached with valid frames in this flow (no evidence any builder invocation consumed F8/FC toward SDO). Kept as structural reference, not cause.

## 16. REJECTED / CORRECTED HYPOTHESIS LEDGER
| OLD CLAIM | EVIDENCE | WHY WRONG | CORRECTION | STATUS |
|---|---|---|---|---|
| `0x97 vs 0x04` decides Decoder_Type | case `0x30` body has NO `0x112E98` read (reads latch) | wrong read-site | relevant only to legacy GetDTSInfo/case 0x3C | REJECTED for failing call |
| geometry gate = root cause | gate exists, semantics partial (polarity ambiguous per archive task 309) | no link to observed fail | capacity-adjacent staging only | REJECTED |
| `0x10C = AC-3` | P3: `0x0B000000`=DTS, AC3=`0x09/0x0A…` (name tables) | mislabel | DTS family | REJECTED |
| `0x13B = MI_MAD_DEC_BUF` | P-subagent: indexed binding → `MI_MAD_ADV_BUF` | wrong table entry | corrected (archive; kept) | REJECTED |
| `tee_mode=optee ⇒ MS12V22` | loader branch ≠ selector | conflation | independent | REJECTED |
| SHM33 causal role | host formula only; NO PROVEN RELATION to DSP path | no link | structural only | REJECTED |
| slot==1 (F3a) | `_MApi_` gate admits only r4∈{5,−1}; no writer of 1 on path | unreachable value | EXCLUDED on _MApi_ path | REJECTED (path-scoped) |
| slot==2 secondary mismatch | no writer of 2 anywhere found | no evidence | UNLIKELY | REJECTED as claim |
| init race (threads) | no thread/workqueue links query↔stamp; order is structural (gates), not temporal | wrong mechanism class | dead-path replaces race | REJECTED |
| teardown-fail as primary cause | mid-play fails VERIFIED (Start+14–18 s) | sampling artifact of late reads | symptom, not cause | REJECTED |
| F8/FC→DMA bridge exists | exhaustive absence in path | assumed connectivity | VERIFIED ABSENCE (path-scoped) | REJECTED |
| F0 success semantics (`F0=N`) | only `F0=0` disabled-path store exists | over-read | F0=0 on disabled; else unproven | CORRECTED |
| Ghidra `bg.jal` targets trusted | const `0x130CEA` bug (archive) | decoder bug | Reko/byte-arithmetic for calls | CORRECTED (inherited) |
| post-release latch=1 (mine) | `SetDecoderX_Type(0,1)` stores r4=0 (r1 is condition) | misread operands | latch=0 | CORRECTED |
| pure-offload skips all SetAttr (mine) | live flags `0x11` → isDirect=1 → `0x606` runs | assumed flags=0x10 | corrected; slot unaffected | CORRECTED |
| multi-actor (mine) | PIDs = one process + threads | PID≠process | single HAL service | CORRECTED |
| live_snapshot 28 rows | file has 35 rows / 11412497 B | doc drift | recorded mismatch | CORRECTED record |
| v4 HW-path as archive fact | archive never traced F_028AFE (stage-2 spec only) | misattribution | my-session evidence, verified independently after | CORRECTED |
| wrong-file reads (mine) | mik `.text` ends `0x2CA2D8`; phantom `0x3E59xx` outputs | PowerShell/script confusion | quarantine + file-script re-verification (chk*.py); utpa2k findings re-confirmed | CORRECTED, procedure kept |
| `0x45F7CC`-latch-table base | my off-by-8 (`0x45F1C0` vs pc+8 `0x45F1C8`) | anchor error | raw-word table re-read; latch 0xA→`0x45F6BC` confirmed | CORRECTED |

## 17. EVIDENCE CLASSIFICATION

### VERIFIED (selection; each with address-level proof in §§7–15)
Kodi SyncDTS+RAW-PT sink; OFFLOAD DTS thread flow (FrmRdy/underruns/frames); `Codec:9` start; mid-play Decoder_Type fail + `codec NONE` + `status −1` + silence; full `_MI_→HAL case 0x30` chain; latch (not byte) read-site; `Convert(0)=0`; fail-condition set {`AV+0x4D8∉{1,2,3}`, latch→fail-entries}; SHM38/Convert exclusion; slot pass-through; `_MApi_` gate stack; `MapDecoderType` table; START `+0x10=0xFF`; OPEN no-stamp; `ctx+0xEC=6`; slot pointers/identities; dead r7-bridge; dual-entry/dual-tail output fns; stock hashes; actor identities; 0x97-irrelevance; post-release latch=0; teardown-fail mechanism.
### VERIFIED ABSENCE (bounded scans stated)
F8/FC→output data bridge (chain functions); `r7` use; `R+0xACC0`/`R+0xD9C/R+0xDA4`/`ctx+0x58…`/`AV+0x1C2/AV+0xD2` writers (spellings listed); `Parser_Open/Flush` call sites; `SetDecodeSystem/MApi` imports in `.so`; AV+0x4D8 mechanism in `.so`; direct BL to `_MApi_SetDecodeSystem`; direct caller of `F_029867`; `logcat/dmesg` fail-print/`[dts] ver` lines.
### LIKELY
Utopia forwards struct (struct-shaped gates; shared kernel space); `AV+0x4D8==0` live (corpus boot precedent + necessity under dead-stamp); slot on live path ∈{0,?} normal-operational; `-1` = never-locked codec-NONE state; `.so` GetAttr sites avoid attr0-branch.
### BLOCKED (what→resolver)
- Live `AV+0x4D8`/latch values → needs readable diagnostic or new playback instrumentation (denied/unavailable read-only).
- Utopia core `(handle,struct)→(r4,r5)` mapping → needs kernel Utopia image (outside corpus).
- Register-offset/indexed writers (residual) → needs full data-flow engine (Ghidra PCODE/Reko with AEON).
- DSP-side arrival of bytes → needs DSP trace/`/dev/mem` (forbidden) or AVR lock evidence.
- First-query client (attr0 path trigger) → needs AudioFlinger/HAL call-order trace.
- `r1` at `MApi_OpenDecodeSystem@0xA29F4`, `r7`/`sl` full provenance in `MI_Open`, `sp+0x24` bypass-path values → deeper mik data-flow (diminishing ROI).

## 18. CURRENT CAUSAL MODEL (no overclaim)
`Kodi DTS detect (syncword 0x7FFE8001)` → `RAW PT DTS_512 + OFFLOAD DIRECT|COMPRESS` → `MI_AUDIO_Start Codec:9` → `SetDecodeSystem path: _MApi_ gates (struct+0x10=0xFF on Start path; AV-bytes unwritten) fail → MDrv_/HAL_SetDecodeSystem/SetSystem2 never run → latch never stamped 0xA AND CheckHashkey never runs via this path (ver stays init-0)` → `Decoder_Type(0x30)` reads stale latch → `ret 0` (mid-play VERIFIED) → `codec NONE` → `status −1` → silence (user-confirmed), while bytes stream (underruns 0). Teardown adds deterministic fails (`Release→latch=0`). OPEN path never stamps by construction. F2 and F3b-never-stamped are one dead chain (auth-side / stamp-side).

## 19. EXACT REMAINING BOUNDARY
(1) Utopia core arg mapping (kernel, outside corpus). (2) Live AV/latch values (no read-only surface). (3) Register-offset writer residual. (4) DSP byte-arrival proof (needs trace/`/dev/mem`/AVR). (5) First-query client trigger. Nothing else is load-bearing; no artificial blockers added.

## 20. WHAT SHOULD NOT BE RE-INVESTIGATED (why closed)
- F8/FC→DMA data bridge (absence proven in path); TEE/SHM33/geometry as causes (no link to failing call); `0x97` (wrong read-site); slot∈{1,2} on `_MApi_` path (gate-excluded); SHM38 (all leaves succeed); Convert (returns 0); init-race-as-threads (no links); Ghidra-call-target trust (use byte arithmetic); v4-as-archive (reclassified); `Parser_Open`-in-.so (0 sites).

## 21. REPRODUCIBILITY (key claims)
- Decoder_Type fail live: `adb logcat/dmesg | grep GetCodecType` during Kodi DTS playback (needs playback).
- Latch table: `struct.unpack('<I', utpa2k .text[0x45F1C8+k*4])`, anchor bytes `E28F1000/E791F100` at `0x45F1C0/0x45F1C4`.
- Gates: capstone disasm `utpa2k 0x45EA5C / 0x45F6BC / 0x45F7CC / 0x3E52A8 / 0x44D1DC / 0x440D70`; `mik 0xA33E0 table 0xA33FC / 0xA44BC / 0xA47E4 / 0xA4858`; byte patterns `e3a000ff…e58d1070`, `E5804524`-family scans, `0xE791F100`-style table validation.
- OFFLOAD live: `dumpsys media.audio_flinger | grep -A40 'type 4 (OFFLOAD)'`; asound closed-check.
- Hashes: `md5sum /vendor/lib/modules/utpa2k.ko mik.ko`; `SHA256SUMS.txt` recheck.
- Scripts: `C:\Users\k0994\AppData\Local\Temp\opencode\chk*.py, disas.py, sodisas.py, syms.py` (file-based only; inline `python -c` under PowerShell is contamination-prone).

---

# PART II — REPAIR ASSESSMENT (analysis only; nothing changed)

## L0 — Engineering Menu / hidden vendor settings
No evidence examined either way in this chat (menu never inspected). Structural view: the dead path is CODE gates (unwritten state), not a setting flag — UNLESS a menu entry persists something that (a) sets `AV+0x4D8≠0` (auth store is hashkey-derived; menu cannot forge AUTH), or (b) pre-fills latch/SHM state (no menu→AV path known). Verdict: **no basis to expect menu fix; check is cheap but hope is ungrounded.**

## L1 — Persistent configuration
Same reasoning: `ver` derives from hashkey/AUTH IP checks at runtime; properties/DBs don't feed `AV+0x4D8` or latch in any traced writer set. **Unlikely repair level.**

## L2 — `audio.primary.mt5889.so` (host HAL)
Could it form the missing state? It CAN write open-struct fields (it builds `out+0xD0…` incl. codec) and already reads DTS version (`GetAttr 0x5002→dev+0x3C98`, gate `==3` in raw_get). A repair here would mean: ensure the kernel-visible decode-system struct carries passing values (`+0x10∉{0,0xFF}` etc.) or add the missing `SetDecodeSystem`-equivalent trigger. WHAT: struct formation/open path. WHY: only host component on the path with write access to the structs. EFFECT: gates could pass → stamp → identify. RISKS: behavior change for all codecs; needs struct-layout certainty (currently partial: `r1` at Open call, `r7/sl` provenance gaps). PREREQUISITES: close those micro-gaps; validate on AC3 first (shared path!). HOW TO VALIDATE: same 5-run protocol (sink → Start → NO fail lines → status≠−1 → AVR lock).

## L3 — `mik.ko` / `utpa2k.ko` (kernel; most concrete points)
Candidate points (no bytes given): (a) `_MApi_` gate set (documented, reviewable); (b) `MI_Start` overwrite `+0x10=0xFF` (single store `0xA44BC` — the most surgical point in the whole graph: it deterministically kills the gate); (c) latch init/reset defaults; (d) `_MI_` cached-stale branch. WHAT/WHY/EFFECT: (b) flipping the overwrite to a passing value (or preserving `GetDecodeSystem` output) would open the gate with minimal blast radius — IF downstream (AV-byte, ver) cooperates. RISKS: kernel-module signing/rebuild chain, boot risk, hashkey interplay. PREREQUISITES: stock backups + hashes (have), UART/serial recovery path (NOT confirmed — do not touch without it), exact `sp+0x24` path-condition audit (which branch writes 0xFF vs stale).

## L4 — DSP image (`mst_codec_r2`/MS12V22)
**Not needed per current evidence.** The builder/parser chain exists and was never reached with valid frames in this flow; failure is strictly upstream (HAL type query). Touching DSP would be maximal risk for zero causal leverage. (If a future AVR test showed IEC bursts emitted but malformed, revisit.)

## L5 — Combination
Host struct fix (L2) + kernel gate review (L3) is the rational combo; DSP excluded; config/menu excluded absent new evidence.

## MINIMUM THEORETICAL FIX
Change `Decoder_Type(0x30)→0` into `→DTS` requires SIMULTANEOUSLY: (i) latch holding a success-mapped value at query time (e.g. `0xA` via a `SetSystem2` that actually runs), (ii) `AV+0x4D8∈{1,2,3}`, (iii) SHM38∈[1,7] (already success-neutral). Prerequisites: a living stamp path (currently dead by gates/init-state) + passing version state. Authorization/version IS required (gate `0x45F6DC` is unconditional on the latch-`0xA` path). A single host-side state change is INSUFFICIENT as long as `_MApi_` gates fail first — the gates must pass (or be reviewed) too. No hex provided, per constraints.

## ENGINEERING MENU ASSESSMENT
No — not as first step: nothing in evidence connects menu settings to the dead code path (gates keyed on driver state + AUTH, not user settings). A menu check costs little but cannot substitute for the state/gate analysis above; do it opportunistically, not primarily.

## PATCH TARGET ASSESSMENT (most rational candidate)
**`mik.ko` `MI_AUDIO_Start` struct overwrite (`0xA44BC`)** — single store, maximal leverage, minimal blast radius among code options; runner-up: `_MApi_` gate review in `utpa2k.ko`. `.so` second (struct formation). DSP image: no. Config/menu: no.

## VALIDATION / ROLLBACK PLAN (future, safe sequence)
1. Full backup: `utpa2k.ko`, `mik.ko`, HAL `.so`, `persist`/audio configs; record MD5/SHA256 + `getprop` + `dumpsys` baselines; confirm serial/fastboot recovery availability (STOP if absent).
2. One change at a time, reversible (keep stock file for `adb push` restore or reflash path).
3. Verify with the fixed 5-run protocol: sink type → Start → `dmesg` GetCodecType lines (expect absence of fail + ideally success-path evidence) → `decoder status` → `codec type` → AVR lock/sound (user-confirmed ground truth).
4. Roll back on any regression (AC3/PCM paths must keep working — shared machinery!).

---

# PART III — SELF-CRITIQUE: WHAT COULD STILL BE WRONG?

- **Utopia ABI:** the `(handle,struct)→(r4,r5)` unpacking is inferred from gate shapes, not observed. If the dispatcher synthesizes scalars (e.g. `r4∈{5,−1}` always), the START path could reach `MDrv_` and my dead-path verdict weakens to «dies at AV-byte/`+0x10` gates» (still dead given unwritten AV-bytes + `0xFF`, but via fewer legs). Falsifier: dispatcher RE.
- **Struct mapping:** `r1` at Open call, `r7/sl` origins, `sp+0x24` bypass values, GetDecodeSystem body — residual gaps listed; each could theoretically carry a passing value. Counter: live failure is systematic across runs/clients, favoring structural over coincidental causes.
- **`struct+0x10` writers:** MI_Open writes via mapper (codec-dependent ✓), MI_Start overwrites `0xFF`/stale; a bypass-path stale value ∉{0,0xFF} is possible but unevidenced + contradicted by systematic failure.
- **External writer possibility:** a third kernel client writing AV/latch/structs (ATV paths exist but weren't traced per scope freeze) could change state — noted, not chased (user-scoped out).
- **`AV+0x4D8`:** never observed live; corpus boot precedent (0) is same-lineage but not this boot. A nonzero live value would kill F2 but leave F3b standing.
- **`status=-1` semantics:** inferred as never-locked-NONE; could equally be «no decoder engaged by design in passthrough». If so, the TRUE fault may sit downstream (DigitalTx/SPDIF enable) with codec-NONE as a parallel benign symptom — the M2 branch stays honestly open.
- **Multi-client:** first-query client unattributed; an OMX/mediaserver query ordering could still surprise.
- **Silence causation:** frames-written + underruns-0 prove HAL consumption, not SPDIF emission nor DSP arrival. An independent SDO/SPDIF blocker AFTER decoder-type failure is possible but currently gratuitous (no evidence for it; codec-NONE suffices).
- **Falsifiers, summarized:** (a) any observed `Decoder_Type` success / `status≠−1` / `codec≠NONE` mid-play; (b) AVR lock with `codec NONE` (would prove output independent of type query → M2); (c) a writer of `AV+0x1C2/0xD2` or `AV+0x4D8≠0` found; (d) dispatcher proof of scalar passing with valid values.

---

# PART IV — EXTERNAL UNION-ALPHA CROSS-CHECK

1. **Confirmed in this chat:** stock hashes; `B`/mmap/selector/CheckHashkey structure; builder byte-identity + 4 callers; `0x411A6` parser→output pair + descriptor + r12-gate; F8/FC writers/readers layout; `GetDTSInfo &0x1F==4` gate text; `0x010C≠AC-3`; latch addresses; `0x45FD1/0x4618E` bounds; stopping boundary (no downstream consumer) — all consistent, several re-derived byte-identically.
2. **Corrected:** `F0=N` success claim → disabled-only; geometry polarity → ambiguous; post-release latch `1→0`; v4 HW-path-as-archive → reclassified to own evidence then independently verified; `0x97`-as-cause → rejected for this call; init-race-as-threads → replaced by structural dead-path.
3. **Historical-only (not re-derived, kept as context):** TEEB/OPTEE internals, SHM33 slot arithmetic, predecessor-session lineage (`sess_dfab4598` etc.), toolchain validation details, subagent-internal reasoning.
4. **Contradictions current-vs-archive:** none on facts; two on emphasis fixed above (F0, latch value); one on scope (archive stopped at descriptor boundary; this chat continued through HAL offload to the failing type query — extension, not contradiction).

---

## CURRENT BEST MODEL
Kodi DTS (RAW PT, offload) → `MI_AUDIO_Start Codec:9` → `SetDecodeSystem` chain dead by construction (open: no stamp code; start: `_MApi_` gates fail on init-zeroed/0xFF state; AV auth/version bytes unwritten) → latch never `0xA`, `AV+0x4D8` stays init → `Decoder_Type(0x30)` reads stale latch → ret 0 mid-play (VERIFIED) → `codec NONE` → `status −1` → silence with live byte-flow. Teardown fails add deterministically (`Release→latch=0`).

## CURRENT VERIFIED STATE
Sink formats, offload flow counters, Codec:9 start, mid-play type-query failure, codec NONE/−1, full `_MI_→HAL` read path + fail set, slot pass-through, `_MApi_` gate stack, mapper table, `+0x10=0xFF` chain, open no-stamp, actor identities, all absences listed in §17.

## CURRENT HARD BLOCKERS
Utopia dispatcher mapping; live AV/latch values; register-offset writer residual; DSP byte-arrival proof; first-query client trigger. (Each with resolver in §17.)

## MOST LIKELY REPAIR LEVEL
Kernel (`mik.ko` struct overwrite + `_MApi_` gates review), host `.so` second; DSP/config/menu excluded on current evidence.

## MINIMUM MISSING EVIDENCE
One observed `AV+0x4D8`/latch value at first-query time, OR dispatcher proof — either decides F2-vs-F3b residue and unlocks safe patch design.

## WHAT I WOULD DO NEXT (recommendation only)
`dmesg`-archaeology for `SetDecodeSystem`-adjacent prints across full boot (no new playback), then — only if inconclusive — a single instrumented-by-observation playback aimed at the first 15 s with per-5-s `dumpsys` snapshots to catch any transient non-`−1` state; parallel AVR-lock check as independent ground truth for M2.

*End of master report. Nothing was modified, patched, or reconfigured to produce it.*

