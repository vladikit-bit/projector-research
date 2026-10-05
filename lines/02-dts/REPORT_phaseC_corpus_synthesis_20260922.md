# PHASE C — CORPUS SYNTHESIS & CAUSAL RECONCILIATION (2026-09-22)

**Mode:** static corpus read-only. No device/ADB contact, no patches, no writes to any
prior artifact. This is a **NEW file**; nothing was deleted or overwritten.

**Corpus read (this phase):** Stealth/Union Alpha archive 001–007 + 13/13 subagent
trajectories; Muse `forensic_trace_*` (5 files) + MASTER/HANDOFF/AUDIT_UPDATE;
Phase A/B `MIMO_FORENSIC_AUDIT_20260922.md`; TCL/T615 via AUDIT_UPDATE §1 + HANDOFF §1
+ Phase B B7; `REPORT_final_dts_patch_experiment.md` (full 1068 L);
`REPORT_DTS_final_evidence_chain.md`; `REPORT_full_license_profile.md`;
`REPORT_r4a_otp_license_provenance`; wire findings (R57 capture + baseline);
`REPORT_GLM53_full_audit_and_dts_repair.md` (full); `AUDIT_full_20260908.md`;
`REPORT_dec_dsp_forensic` + `REPORT_dec_state_transition` (09-06);
`REPORT_current_state_K1_verdict_20260915.md`; `REPORT_cmd04_deploy`;
`REPORT_16bit_gate_pass`; `REPORT_T2_breakpoint`; `REPORT_hal_gate1_experiment`;
evidence-chain-era reports already open (ExpA, r10, rootcause, SUPERSEDES_s24,
chronology ledger, final_verdict, codec5_vs_9, mission log, session index, BACKUP
manifest/README); archive-extracted hash-verification message (local md5sum, full table).
Lower-value remainder (runtime_phase27–52 detail, early phase A–E raw, some 06-06
side reports) inventoried but not line-read — noted as scope limit in §1.

**Claim vocabulary:** `VERIFIED | STRONGLY SUPPORTED | LIKELY | UNPROVEN | DISPROVEN |
SUPERSEDED | ARTIFACT/ERROR | INERT-SURFACE (test ran, bytes never executed)`.

**State labels:** `STOCK / EXP-A / EXP-B / cmd04 / D1 / L2 / P-A / N-series / K-1 / K-2 /
P-BYP / R5 / dts_v3 / other`.

---

## §1 Corpus & history reconstruction (chronological, labeled)

| Era | Date | State label | What happened | Evidence anchor |
|---|---|---|---|---|
| Static chain built | 08-29…08-30 | (device Solution A era: utpa2k+libmi3 patched) | Evidence chain: no host bypass of license gate; SDO packer only inside R2 image; T2 finds EDID-driven SPDIF_SetMode overwrite in mik; HAL gate1 bind-mount experiment (rolled back) | `REPORT_DTS_final_evidence_chain`, `REPORT_T2_breakpoint`, `REPORT_hal_gate1_experiment` |
| License profile | 08-30 | research-only | IDs 0xb/0xc = AC3/DD+ (**not DTS**); true DTS group {0xf,0x3a,0x12,7}; 0x43d/0x43e = Dolby-premium; 0x4d8 = DTS level 1/2/3; strap 0x112cf0 bit7 read-only; 4-NOP emulation designed | `REPORT_full_license_profile` |
| R4-A | 09-06 | static | OTP bounding/hash print exists only in SND-MS12V22; `dts m6 licensee` print in DEC, storage UNDECODED; join verdict D (insufficient) | `REPORT_r4a_otp_license_provenance` |
| DEC forensic + state | 09-06 | **EXP-B state on device** (mik R5 + utpa2k Exp-B + libmi3 dts_v3) | cmd 0x04 vs 0x97 decoded (stock DTS=0x04; 0x97 only if AV+0x4d8==3); `Invalid Spatif license` + `lic[]` live only in DEC; CheckHashkey AbsWrite census = 0 DSP-visible regs; SpecifyDigitalOutputCodec dead; +0x2e6 load → rept/skip machine; `0x97→0x04 changed nothing` | `REPORT_dec_dsp_forensic`, `REPORT_dec_state_transition` |
| cmd04 deploy | 09-06 | **cmd04** (Exp-B `4c5e6fbb` → `0fb4ffeb`) | Single byte 0x97→0x04 in loaded ko; host path identical; AVR lock pending→later treated as no-effect | `REPORT_cmd04_deploy_test_results` |
| Full independent audit | 09-08 | (device state as then) | ARM-license/EDID/transport/mode/physical TX **CLOSED** by live tests; +0x2e6 gate UNPROVEN nearest edge; print never observed; best-next = pull existing AudioDECR2 logs | `AUDIT_full_20260908` |
| Wire instrument + patch marathon | 09-12…09-13 | V1 SND, then **L2**, **D1**, **P-A** (/vendor audio_bin) | dump_spdif_npcm discovered; DTS wire 0 B (root); frozen-AC3 `dts_01.bin` re-explained as stale; L2/D1 deployed→**later shown INERT** (loader truth); packer-gate invalid; s→0x24 SUPERSEDED by 0x460C2; libmi3 regression RETRACTED | `REPORT_final_dts_patch_experiment` §7–§12; wire findings; K-1 §C |
| GLM53 N-marathon | 09-13…09-14 | **P-A + N-1…N-15** as /vendor DEC+SND files; N-11/G active | "First DTS bursts", engine 81→4 switch, OTP classifier, mailbox, HASH patches — **causality later DISPROVEN**: /vendor files never executed; effects = player-path/variance | `REPORT_GLM53_…` + **K-1 §C/F.3** |
| K-1 / K-2 | 09-14…09-15 | **K-1** `58d9e8d0` (embedded DEC blob patch); **K-2** `5d2f2652` (ARM 17 B, embedded DEC stock) | **LOADER TRUTH**: DspLoadCode memcpy's embedded blobs in utpa2k.ko; /vendor audio_bin = dormant. K-1: no frames to type[81]; K-2: type[81] held + ES fills but play=0/wire 0. N3 autopsy: all historical DTS bursts = AC3-config pipeline fed DTS bytes (TrySyncAC3 misdetect / PAPlayer), **not** DTS-config emission | `REPORT_current_state_K1_verdict_20260915` |
| Codec5-vs-9 + P-BYP | 09-16 | **K-2→P-BYP2/3/4** utpa2k `2e251c98`, mik `d49a3790` (R5+P-BYP5) | Static chain closed (5→0x81 / 9→0x4|0x97); P-BYP3 restored stock m6 engine → **DTS decode works**, wire still 0; r26 "DTS output config never set"; 0x81-hold insufficient; P-BYP7/8 reverted | `FINDING_codec5_vs_9_host_differential`, `MISSION_LOG_dts_persuit_20260916` |
| Hashkey discovery / clean base | 09-16/17 | reflash toward **STOCK** | `STOCK_snapshot_boot` (hashkey `(efe6bf,ffffff)`, spdif PCM); DTS wire 0 on restored base; **112E dump during DTS PLAY = GAP**; MISSION_LOG "pristine utpa2k" claim-only until 09-18 runtime md5 | archive current-baseline msg; `STOCK_*` pulls |
| Idle verification | 09-18 | **STOCK** (utpa2k `2fc6e9fc`, mik `c1421040`) | forensic_trace_autonomous adb md5 = stock modules; all PCM closed; no DTS in logcat/dmesg idle | `forensic_trace_autonomous_20260918` |
| Phase A/B | 09-22 | research-only | H1 static AUTH chain closed to gate flag; MApi callers found; gIpAuthVars writers = ioctl only | `MIMO_FORENSIC_AUDIT_20260922` |

---

## §2 Evidence reconciliation (conflicts resolved)

### R1 — THE decisive structural fact: INERT vs REAL surface (K-1 §C)
`HAL_AUDSP_DspLoadCode` (0x46f5a8) memcpy's the **four ELF-symbol blobs embedded in
utpa2k.ko `.data`**; no `filp_open`/`request_firmware`. `/vendor/lib/utopia/audio_bin/*.bin`
are **dormant snapshots**. Proven by XMARK/XALT/STOCKDEC controls (marker strings in
/vendor files never appeared in R2 logs).

**Consequence:** every experiment that wrote only `/vendor/.../aucode_*.bin` —
**V1-Pc, L2, D1, P-A, N-1…N-15 (as /vendor deploys), H1/H2 (not flashed)** — is
`INERT-SURFACE`. Their "PATCH HAD NO EFFECT" verdicts are tautological (bytes never
ran); they **do not** disprove the patched code site's causal role on the executing
surface. Re-labeling applied in §3/§4.

Experiments on surfaces that **did** execute:
- ARM `.text` of the loaded utpa2k.ko / mik.ko: **Exp-A, Exp-B, cmd04, alllic_diag,
  dtslic_pass, K-1(embedded), K-2, P-BYP2/3/4(+6), R5, dts_v3, HAL gate1** → their
  negatives/positives are valid for those sites.
- K-1 put the N-era DEC edits **into the embedded blob** (real surface) → valid test of
  those DEC sites → still failed (licensee/keys/frames).

### R2 — "First DTS bursts" causality
- GLM53: bursts = P-A patch effect. **SUPERSEDED/DISPROVEN** by K-1 §C + N3 autopsy:
  bursts coincide with raw `.dts` through Kodi PAPlayer / AC3-first priming +
  TrySyncAC3 misdetect (Codec:5 pipeline consumes DTS bytes; SDO re-emits with Pc=0x0B
  because the framer classifies by syncword). Stock-equivalent embedded code produced
  the same bursts (`N3_DTS_dts.bin`: 152 DTS + 273 AC3 preambles).
- Standing rule: **every historical "DTS burst on wire" observation is AC3-config
  emission of DTS bytes — never DTS-config output.** (N3 autopsy, K-1 F.3.)

### R3 — license-ID mapping
Evidence-chain break point `IPCheck(0xb/0xc)=DTS fail` is **DISPROVEN** (license_profile
same-day correction): 0xb/0xc = AC3/DD+ (and they PASS — AC3 works). True DTS IDs =
{0xf, 0x3a, 0x12, 7}. Structural claims of the evidence chain (no host bypass path;
packer only inside DSP; ES-buffer only data path) **stand**.

### R4 — 0x4d8 in start chain
license_profile: "0x4d8 not read in SetDecodeSystem/SetSystem2/SetDecodeCmd" →
**DISPROVEN** by `dec_dsp_forensic` byte proof: case 0x44d1dc `ldr r0,[r0,#0x4d8]; cmp #3;
movweq r1,#0x97` → AbsWriteByte(0x112E98). Stock DTS emits **0x04**; 0x97 only under
Exp-B-forced 0x4d8==3 (or full DTS license).

### R5 — SND and OTP
AUDIT_full: "SND DSP cannot read OTP" → **DISPROVEN** by GLM53 N-2: SND classifier
reads OTP bounding mirror `0xB000_083C` (=0xff00 runtime) and HASH `0xB000_0838`;
check1 `(B&0x1001)==0x1000` **fails** at runtime → skips mailbox config-notify →
DEC never gets DTS-capable config → downgrade byte 4 at `0xB000_0018` → type[4].
(Both statements can look true only if "read OTP" meant the OTP controller directly;
the mirror read is proven.)

### R6 — ARM license force vs DSP licensee
Exp-B (0x4d8=3), alllic_diag (keys 0xffffff), dtslic_pass (Get_DTS_License→0) all left
**DSP licensee fields = 0** (runtime). Consistent with: CheckHashkey AbsWrite census =
0 DSP-visible registers; the only designed 20-byte digital-output identity command is
dead; SE-IDMA mask push (0x424bd8 → 0x112A82-85) is a **separate, still-unlinked**
transport (tension recorded in §7). => **ARM-side license emulation is not sufficient**
to populate DSP-side DTS authorization (audience: hypotheses A vs C).

### R7 — Wire "frozen AC3 ×26" vs "0 bytes"
`dts_01.bin` (270 KB, stale AC3 header) = early capture under V1 state; later
properly-rooted same-session A/B (wire_baseline, L2-era, D1-era) = **0 bytes under DTS**.
Unified reading (patch_experiment §8.2): no DTS writes ever reach the TX; what was
captured earlier was residual non-PCM buffer content. **0-byte DTS is the canonical
wire observation** (when instrument armed with adb root + same-session AC3 control).

### R8 — Instrument validity lesson
Any "0 bytes" captured **without adb root** is INVALID (`/proc/utopia_mdb/audio` is
root-only; §9.0 of patch_experiment retroactively voids an earlier both-codec-0 run).
Standing protocol: uid=0 → uptime-verified reboot if firmware touched → AC3 positive
control in the same session → DTS capture.

### R9 — TCL/T615 separation
- ARCH: same mega-lib AUTH pattern (UtopiaSetIPAUTH body, zero internal callers of
  MApi_AUTH_Process), +0x10=0xFF gate passes, factory codes, stripped utpa2k (shoff=0),
  AArch64 modules vs TD98 ARM32, entry 5→7 vs 5→3, utpa2k 33 vs 25 MB.
- RUNTIME-PROVEN: **none** — no evidence V440 (or any T615 FW) ever emitted DTS/AC3
  passthrough. Phenomenology (per-FW codec matrices on identical silicon) = existence
  proof that **vendor software decides DTS availability** — supports software-block
  model without proving any TD98 mechanism.
- Status: `ARCH only`; never a donor; user constraint inherited.

---

## §3 Corrections / disproven / superseded (consolidated)

| # | Old claim | Status | Correcting evidence |
|---|---|---|---|
| 1 | +0x10=0xFF kills gate | DISPROVEN | gate `cmp #0xFF; cmpne #0; bne FAIL` — 0xFF passes (TCL + TD98 stock @0x3E53B8) |
| 2 | latch 0xB arms output | DISPROVEN | live tests |
| 3 | V440 = working-DTS reference | DISPROVEN/CONSTRAINT | no runtime proof; user rule |
| 4 | MApi_AUTH_Process has zero importers | DISPROVEN | libmi GOT/PLT + BL @0xAB32C in MI_SYS_Init (Phase B) |
| 5 | midaemon only HASHKEY_Verify | SUPERSEDED | midaemon BL MI_SYS_Init @0x2184 first (Phase B) |
| 6 | libhashkey_ca → gIpAuthVars | DISPROVEN | TZCTL hashkey ≠ AUTH (Phase B) |
| 7 | s→0x24 fork @0x460B1 = DTS blocker | SUPERSEDED | bitstream-reader field; real gate family 0x460C2/params (SUPERSEDES_s24); and whole DEC-image patch class later INERT/EXHAUSTED |
| 8 | Packer gate 0x265F3 patchable | ARTIFACT | already-FALSE bits; TRUE path = return (PATCH TARGET INVALID) |
| 9 | SND Pc=0x0B fix | INERT + no-effect | /vendor file never ran; path never reached writer (0-byte proof) |
| 10 | L2 capability gate is last blocker | INERT-SURFACE | /vendor DEC never executed (loader truth) — site untested on real surface; separately, capability gate role reduced by K-1/K-2 |
| 11 | D1 / P-A caused path change / bursts | INERT-SURFACE + DISPROVEN causality | /vendor inert; bursts = N3/PAPlayer mechanism |
| 12 | N-series effects caused by patches | DISPROVEN | K-1: flicker independent of deploy; XMARK/XALT |
| 13 | Holding engine 0x81 suffices for DTS | DISPROVEN | K-2 held 0x81: ES fills, play=0, wire 0 |
| 14 | 0x97 wrong-command is root blocker | DISPROVEN (as sole cause) | stock sends 0x04; cmd04 test equal; `0x97→0x04 changed nothing` |
| 15 | 0xb/0xc are DTS IP IDs | DISPROVEN | GetLicenseAc3 checks 0xb first; AC3 passes |
| 16 | 0x4d8 never read in start chain | DISPROVEN | 0x44d1dc load |
| 17 | 0x43d/0x43e are DTS bits | DISPROVEN | Dolby-premium status (license_profile) |
| 18 | libmi3 regression broke working DTS | RETRACTED | user: never clean DTS; PASSTHROUGH_FIX wrong (transcode) |
| 19 | EDID/mode gate blocks DTS alone | CLOSED | R5 deployed → no DTS |
| 20 | Decoder status -1 = DTS fail signature | DISPROVEN | prior audits |
| 21 | Four independent writers of AV+0x4D8 | SUPERSEDED | CheckHashkey tail store @0x424878 (sole g_AudioVars2 writer site found); ioctl writes gIpAuthVars not 0x4D8 |
| 22 | SND cannot read OTP | DISPROVEN | reads 0xB000_083C (N-2) |
| 23 | License-gate numbers 146/1624 | DISPROVEN | Python decoder drift |
| 24 | "0 callers" license-gate scan | DISPROVEN | 12 jal/j into 0x21E00–0x21F60 |
| 25 | 'Invalid Spatif license' observed in logs | UNPROVEN (never seen) | grep across workspace logs: only static mentions; print never captured |
| 26 | Host mirror proves DSP license fields | DISPROVEN as method | mirror↔DSP-VA unreliable (reads 0 while DSP state nonzero) |
| 27 | MISSION_LOG end-state "pristine utpa2k" | ARTIFACT/claim-only (then) | no stock pull until 09-18 adb md5 confirmation |
| 28 | HAL planned IEC61937 via ADEC | DISPROVEN as design | gate1 experiment: format 0x0d unmapped; MStar path absent |
| 29 | Full 43-check license profile as DTS fix | NOT RECOMMENDED / likely redundant | switches DSP images (0x4d0=4 side effect); ARM force already exhausted (R6) |
| 30 | 4-NOP DTS-IPCheck patch as next fix | LOW-VALUE retest | equivalent class already tested via Exp-B/alllic/dtslic without DSP effect (R6); only differs if 0x424bd8 mask push is the missing link — which is a **transmission** question, not a NOP question |

---

## §4 Historical experimental states — hashes & labels

All md5 verified locally where noted (host files, not live device unless stated).

### Kernel modules (utpa2k.ko / mik.ko)
| md5 | Label | Surface | Note |
|---|---|---|---|
| `2fc6e9fc46b6402d73aa6f84f9599d46` | **STOCK utpa2k** | real | `patch_baseline/utpa2k_stock.ko`; device confirmed 09-18 |
| `f44ad0a4bcb411eacbe8f677e0d3780f` | installed pre-K1 | real | `patch_baseline/utpa2k_installed.ko` (= DEVICE_utpa2k) |
| `939cac247eff097f212fef872131df5c` | **EXP-A** | real ARM | BEQ→NOP force +0x4d0=4 class |
| `4c5e6fbb10e24abc3b8d2b0490507983` | **EXP-B** | real ARM | 0x4d8=3 → cmd 0x97; DTS still FAIL |
| `0fb4ffebed5b7059665bbfb92d940656` | **cmd04** | real ARM | 0x97→0x04; no better |
| `58d9e8d03d2f378456814ba3ac6da9e6` | **K-1** | real (embedded DEC) | 100 B / 24 regions in embedded MS12V22 blob |
| `5d2f265226c8707bc2d562124a03ca1c` | **K-2** (=DEVICE_prePBYP) | real ARM | DTS case→AC3-mirror 17 B; embedded DEC stock |
| `ef18748f0f85d4ce05078f0070d76bfe` | P-BYP (gate NOP) | real ARM | BypassMode bne→NOP |
| `a2efbf48` | P-BYP2 | real ARM | DTS row→AC3-success fields |
| `ae775794` | P-BYP3 | real ARM | DTS case→stock m6(4): decode restored |
| `2e251c98de213e08dc38c92f6c5009c9` | P-BYP2+3+4 | real ARM | session-end ADDENDUM14 |
| `f8513553` | +P-BYP6 | real ARM | monitor leveled (ADDENDUM15) |
| `c1421040dbcfff415f9417f482189435` | **STOCK mik** | real | confirmed 09-18 + STOCK_mik_pristine pull |
| `03fc2c0d22b231bc478e574557b85473` | **R5** mik | real | EDID→mode gate; no DTS |
| `d49a3790e1d7a2a673032e625a728259` | R5+P-BYP5 | real | monitor always |
| `299f0fd4` / `c09ef909` | P-BYP7 / P-BYP8 | real, REVERTED | |

### DSP images (`/vendor/lib/utopia/audio_bin/` — DORMANT unless noted)
| md5 | Label | Note |
|---|---|---|
| `4b7e9509b4358fd3a130bd4d3b9cbe0a` | **STOCK DEC MS12V22** | executing content also embedded in stock ko |
| `e26ce887c6d3573d6fd5c243881cb448` | **STOCK ALT DEC (mst_codec_r2)** | 2,736,372 B; BACKUP README: ALT was the log source when 0x4d0≠4 confusion |
| `eb879cdc07f510722f19db6d18d77d3c` | **STOCK SND MS12V22** | 1,839,920 B |
| `42c1cb7991e88c8a97bf926bfb47884e` | **STOCK SND base** | |
| `785fd73bcae39a88e6b6029445190e98` | **V1** SND Pc | INERT surface |
| `cba5f7e795818462b55736543329c9ea` | **L2** DEC | INERT surface |
| `149f4275e9f1ef88ba079f0f32652cee` | **D1** DEC | INERT surface |
| `afde18ec92ffe0a960390a423d7f63c2` | **P-A** DEC | INERT surface as /vendor; **valid** inside K-1 embedded |
| `e64b76e8…` / `28c4ea85…` | H1 / H2 | built, never flashed |
| `ea7cd454…` | N-1-G SND | INERT surface |
| `a1b2b5d5…` | N-2a SND | built |
| `e317d82e…`→`d123f2f2…` | N-3…N-10 DEC | INERT as /vendor |
| `b2a7e2465ef01ff8a4b87b8d21794e1f` | **N-11 SND** | INERT; BACKUP DEVICE asnd MS12V22 |
| `51392f238db9d5f27e03bcc04203eb72` | BACKUP DEVICE adec ALT | N-12-patched ALT @ backup time |
| `2a279c8b1769e9bceb287aa139a0ba99` | BACKUP DEVICE adec MS12V22 | N15_X marker build @ backup |

### Userspace
| md5 | Label |
|---|---|
| `c2b5c57dba4f89228da72a29c71f0c06` | libmi3 **STOCK** |
| `2e34d0c973472405197dd7c7f20bdb92` | libmi3 patched (mask 0x2E0 era) |
| `d71a330f…` | libmi3 v2 |
| `4c199a5c739e61741ba731534fbb3bf4` | libmi3 **dts_v3** (active late; caps mask malformed 0x720002E1, r4 clobber — suspect unverified) |
| `8c11c348…` | audio.primary **STOCK** |
| `8504fa4b6d3ca83f92c339cbb43514b7` | audio_policy.xml (post-restore stock claim) |

### BACKUP_20260914_0717 (deployment record)
DEC-ALT `51392f23` ACTIVE; DEC-MS12V22 `2a279c8b` inactive-but-patched; SND `b2a7e246`;
asnd-alt stock `42c1cb79`. Rollback stocks: `4b7e9509` / `e26ce887` / `eb879cdc`.
**Deploy problem recorded:** R2 logs printed from ALT (0x4d0 select confusion).

---

## §5 STOCK now vs historical states

| Component | STOCK (current, 09-18+) | Worst/most informative historical |
|---|---|---|
| utpa2k.ko | `2fc6e9fc` (adb-confirmed idle) | K-1 `58d9e8d0` (valid embedded DEC test); P-BYP stack `2e251c98`/`f8513553` |
| mik.ko | `c1421040` | R5+P-BYP5 `d49a3790` |
| libmi3 | stock `c2b5c57d` (assumed; dts_v3 was standing late-era) | dts_v3 `4c199a5c` (unverified quality) |
| HAL | `8c11c348` stock throughout (gate1 rolled back) | — |
| DSP /vendor files | stock snapshots (dormant) | N-era patches (dormant/inert) |
| Embedded DEC/SND in ko | **stock** (K-2 restored; reflash) | K-1 embedded N-set (valid, failed) |
| persist props (era-dependent) | spdif/hdmi BYPASS+DTS at D1-era; stock boot showed User/Driver **PCM** in STOCK_snapshot (verify live before test — see §10) | |
| Policy XML | `8504fa4b` stock-claimed | IEC61937 experiment policy (rolled) |

**Cleaner stock state:** YES — current STOCK is the first state where (a) all loaded
surfaces are stock, (b) the inert-surface confusion is understood, (c) the valid
negatives (Exp-B/class, K-2/P-BYP class, R5, cmd04) are catalogued, and (d) the missing
instruments (same-session R2 log + optional SHM/AV reads + 112E snapshot) are known.

---

## §6 Current causal graph (per-edge confidence)

Edges for **stock DTS attempt** (AC3 contrast in parentheses):

```
E1  Kodi raw DTS → HAL (Codec:9, 2012 B/frame)                VERIFIED (runtime)
E2  MI_AUDIO_Start → MapDecoderType 9→0xB → SetSystem2        VERIFIED (static byte)
    → 0x112E98 = 0x04 (stock; 0x97 iff AV+0x4d8==3)           VERIFIED host / UNPROVEN DSP accept
    (AC3: 5→3 → 0x112E98 = 0x81)                               VERIFIED
E3  SetDecodeSystem → CheckHashkey                             VERIFIED
    gIpAuthVars NULL on stock → IPCheck(DTS IDs) fail          STRONGLY SUPPORTED (Phase B chain;
                                                                live NULL = UNREAD)
    → 0x440 |= 0x8|0x80|0x20000 ; 0x4d8 = 0 ; 0x582 = 0       STRONGLY SUPPORTED (static+ExpB inverse)
E4  license masks → R2 via SE-IDMA push 0x424bd8 → 0x112A82-5  STRONGLY SUPPORTED host write /
    (AC3: 0xb passes → Dolby levels; real silicon license)      DSP consumption = UNPROVEN
E5  DspLoadCode loads EMBEDDED MS12V22 pair (0x4d0==4)         VERIFIED (+loader truth)
    (/vendor audio_bin never read)                             VERIFIED
E6  MI_AUDIO_Write → ES buffer (format-agnostic)               VERIFIED (both codecs)
E7  Engine byte: stock DTS → m6 type[4] (via SetSystem2=4)     VERIFIED
    (AC3 → 0x81 DDP type[81])                                  VERIFIED
    external downgrade 0xB000_0018←SND OTP-fail (no MB notify) VERIFIED static (N-2)
E8  licensee fields (DM 0x1C012734+idx·0x290) = 0              VERIFIED runtime print
    SND→DEC config compose fills verdict=2/licensee=0          STRONGLY SUPPORTED (static chain);
                                                                composer site UNPROVEN
E9  m6 session: decode runs (P-BYP3: PLAY, frames) but output-  VERIFIED runtime
    config row never set (r26 0x0F46-0x0F4C) / es:0-class      
E10 DEC frame-output machine reads arm→R2 +0x2e6 (=0 zero-init, UNPROVEN dynamic execution
    no live writer found) → 'Invalid Spatif license' gate →    (print NEVER observed);
    rept/skip suppress                                          architecture STRONGLY SUPPORTED
E11 SpdifNonPcmWritePtr does not advance for DTS-config        STRONGLY SUPPORTED (0-B captures,
                                                                root, AC3 control)
E12 SPDIF TX wire: DTS = 0 B ; AC3 = MB-scale valid bursts     VERIFIED (instrumented A/B)

E13 (alt path, N3 only): AC3-config engine + DTS bytes in ES →  VERIFIED (N3 autopsy + captures)
    framer syncword-classifies → Pc=0x0B bursts                 but NOT DTS-config; no AVR lock
```

**Binding bottleneck (stock, current best):** E3→E8 **transmission gap** and/or
E10 gate — i.e. between hypothesis **A** (missing ARM provisioning) and **C** (DSP-side
lic/output gate), with **E** (non-PCM arm / write-pointer) still open downstream if a
frame were ever produced under DTS-config. **B** (producer machinery missing) is
effectively closed (machinery exists; feed/config condition is the issue). **D** (0x97)
DISPROVEN as root.

---

## §7 Remaining unresolved (ranked)

1. Live stock values: gIpAuthVars NULL? AV+0x440/0x444/0x4d8/0x582; strap 0x112cf0 bit7.
2. Does 'Invalid Spatif license' (or any rept/skip diagnostic) print in DEC R2 log during
   stock DTS playback? (Never captured; workspace R2 DTS logs from stock era are empty/absent.)
3. Does 0x424bd8 mask push actually land in DSP-visible state read by output machine
   (+0x2e6 / lic[])? — the A→C bridge.
4. Writer/init of arm→R2 +0x2e6 (zero-fill model vs mailbox init item set).
5. `lic[3]` slot ↔ +0x2e6 mapping; consumers behind undecoded 16-bit gate ops.
6. SND message composer (0x1F44F body / join path) that fills verdict+licensees — flow-
   following decode still blocked (Ghidra OSGi; aeon_decode drift).
7. 0x112E98 value during DTS-PLAY (corpus: no 112E dump while DTS playing — GAP).
8. Condition under which SpdifNonPcmWritePtr advances for a non-AC3 payload with
   engine 0x81 vs 0x04 (N3 shows payload-path works under 0x81+AC3-config; K-2 showed
   0x81 alone insufficient).
9. Stock-boot hashkey `(efe6bf,ffffff)` vs DTS-select `0xede227/0xfffefd` provenance
   (derived AV440/444 vs efuse — archive note; not closed).
10. Whether Tier0 debug ioctl can dump AV+/SHM without new code (16bit report Tier0).

---

## §8 Still static-closeable (no device needed)

| Target | Why closeable | Entry |
|---|---|---|
| SND composer 0x1F44F + join 0x1F55C–0x1F620 | bounded region; needs flow-follow decode or fixed Ghidra sweep (SND02_Sweep.java ready) | `snd_full.bin` stock |
| 0xB6D8F (license query feeding 0x1F67F andi 0x200) | helper decode | SND |
| op at SND 0x1F509 (bytes 89 43) | slaspec 2-byte op | SND |
| 0x424bd8 mask push → DSP reader of 0x112A82-85 | ARM static + DEC/SND scan for 0x112A82 | utpa2k + both DSP images |
| DEC boot stub DM-init copy table (0x0–0x206) | phase-aware decode → exact DM↔blob map → locate static verdict/licensee bytes | dec_full.bin |
| 16-bit gate compare constants @0x021f00–0x021fd2 | partial idioms already cracked (8300 82e2); continue bounded | DEC |
| Mask-push vs 'no license-carrying mailbox' tension (R6) | reconcile census vs 0x424bd8 path | static only |
| K-2/P-BYP site re-audit on **stock ARM bytes** (not K-2 bytes) | byte diffs already on disk | host files |

**Do not spend static effort on:** re-proving loader truth, N3 mechanism, Exp-B class
negatives, EDID closure, 0x97-as-root, /vendor DEC patch effects, mirror-read proofs.

---

## §9 Runtime-needed (only what static cannot close)

- The four live values of §7.1–7.3 and print presence §7.2 (one instrumented session).
- 0x112E98 during DTS-PLAY (§7.7).
- Wire confirmation on STOCK with protocol R8 (stock DTS-0 was measured 09-16/17 but
  **without** same-session R2 log license-print capture and without AV/SHM reads).

No other runtime work is justified until §10 returns.

---

## §10 Minimal test proposal + opinion

### The single proposed session (read-only / reversible, uses existing instruments)

**Preconditions:** adb root (`uid=0` verified); STOCK modules md5-checked
(utpa2k `2fc6e9fc`, mik `c1421040`); **no** file writes, no patches, no reboot required
unless user wants a cold-stock confirmation boot; props/settings untouched.

**Protocol (one session, both codecs):**
1. Arm instrument: `echo 'dump_spdif_npcm=1 path=1' > /proc/utopia_mdb/audio`
2. Start DEC R2 log capture (existing `dump_r2_log` path used in PA/N/K eras).
3. **AC3 positive control** — play `test_ac3_51.ac3` ~15–20 s → expect MB-scale dump,
   100% Pc=0x01, Codec:5. Disarm dump; keep log.
4. Re-arm dump; **DTS** — play `test_dts_51.dts` (or `.mp4` DTS) ~15–20 s →
   Codec:9, frame count climbing; pull `/data/DUMP_audio_spdifNpcm_*.bin` +
   `/data/AudioDECR2*` (+ SND log if cheap).
5. Stop; disarm. Optional Tier0 (if an existing debug/proc read exposes it **without**
   new code): snapshot AV+0x440/444/4d8/582 + gIpAuthVars pointer + 0x112E98 during
   step 4 — only if already readable; **build nothing**.

**Record:** dump sizes; R2 log lines for `type[`, `licensee=`, `Invalid Spatif license`,
`s*, es:, play=`, OTP/HASH; engine transitions; 0x112E98 if available.

### What ONE observation discriminates hypotheses A–E

| Observation (DTS window vs AC3 control) | Discriminates |
|---|---|
| **'Invalid Spatif license…' prints (with output_spdifSz=0 or rept/skip)** | **C** is the executing gate; if AV shows 0x4d8=0/0x440-DTS-bits set → A feeds C (A→C edge live); if AV shows license-like state nonzero → C independent of A |
| No print + licensee=0 + AV DTS-bits-missing + wire 0 | Binding edge = **A→E8 transmission** (A active, C architecture idle/unreached) → next static = 0x424bd8→DSP bridge (§8) |
| No print + licensee=0 + AV license-looking state forced-but-0 on DSP | same as above (already known from Exp-B class — prefer stock AV read first) |
| licensees/AV OK-ish + wire 0 + SpdifNonPcmWritePtr static + props not BYPASS | **E** (arm/owner/mode) |
| licensees/AV OK + write-pointer moves + wire content wrong/absent bursts | **B** (producer/feed under DTS-config) — residual |
| 0x112E98 ≠ expected 0x04/0x81 during play | **D**-class engine misprogram (stock expectation: 0x04) |

This is **one session, one class of observation** (wire + log + optional existing reads),
fully read-only, with the mandatory AC3 control (protocol R8).

### Opinion — do prior tests suffice? Does clean STOCK justify repeating?

1. **Prior tests do NOT suffice**, for two structural reasons:
   - A large fraction of DEC-path negatives are **INERT-SURFACE** (§2 R1) — they never
     tested the site on the executing image. Honest status: those sites are
     *untested-on-real-surface*, not *disproven*.
   - The tests that **were** real (Exp-B/cmd04/alllic/dtslic, K-2, P-BYP*, R5) closed
     **D**, most of **B-as-missing-machinery**, and EDID/mode/physical layers — but
     were run in **confounded multi-patch eras** (R5+dts_v3+Exp-B, K-2+P-BYP chains),
     never as a clean stock A/B with license-print + AV read + wire together.
2. **AC3-needed-a-patch precedent** (user's framing) applies to *host capability/policy*
   layers already explored; the current binding unknowns are *live license/print state*
   which no clean-stock triple-measurement ever captured.
3. **Wrong-activation-branch vs wrong-patch-site:** evidence now favors **wrong
   activation/transmission state** (A→C bridge / C reachability) over a missing
   producer (B) or wrong command (D). That is exactly what §10 measures without
   choosing a patch site.
4. **Cleaner STOCK state justifies ONE repeat:** yes — because (i) inert-surface
   confusion is gone, (ii) stock md5s are confirmed, (iii) the protocol is read-only and
   cheap, (iv) corpus has a **hole** exactly at "stock DTS + R2 license print + AV
   values + wire in one session" (§7.2, §7.7, §9).
5. **If §10 returns the no-print + A-active pattern:** stop patching; static-only next
   step = close §8 bridge (0x424bd8→+0x2e6). **If print-with-sz0:** C is the gate →
   static close of §8 composer/gate constants before any further patch discussion.
   **Do not** re-enter N-series / /vendor DEC patching / 0x97 / EDID / Kodi tracks
   (DO-NOT list, §3 + §8 exclusions).

### DO-NOT-REINVESTIGATE (sufficiency-checked)

Sufficient evidence exists — do not reopen unless §10 returns a contradicting datum:

| Item | Sufficiency |
|---|---|
| +0x10=0xFF kill; latch 0xB arm; status−1 fail; [2172]; V440-donor; raw ioctl-byte forgery plan; four-writers-of-0x4D8; r4=codecID; per-codec enable bit; 146/1624 counts; s→0x24-as-blocker; P-A/N causality; 0x81-hold-sufficiency; 0x97-as-root; 0xb/0xc=DTS-IDs; 0x4d8-unread; 0x43d/0x43e=DTS; libmi3-regression; EDID-as-sole; transport/GetHandle; physical TX; Kodi (user-exhausted); /vendor DEC effects; mirror-as-DSP-proof; full-43-profile as DTS fix; HAL-IEC-via-ADEC design | see §3 rows |
| 4-NOP IPCheck set as *next* experiment | redundant vs Exp-B/alllic/dtslic class unless §10 shows mask push unconsumed **and** user reopens transmission patching |
| Further N-series / embedded-DEC bit experiments | K-1 exhausted licensee/keys sourcing; DEC patching declared exhausted on real surface |
| TCL binary porting / V440 as working reference | ARCH only, user constraint |

---

*End of Phase C synthesis. No device was contacted. No prior file was modified.*
