# FORENSIC SPACE BUNNY — FULL CORPUS AUDIT

**Date:** 2026-09-23  
**Mode:** static/corpus-only, read-only  
**Scope:** non-TCL Space Bunny DTS/SPDIF investigation  
**Primary target:** `REPORT_phaseD2_final_static_residual_20260922.md`  
**New deliverable:** this report only

---

## 1. Executive verdict

The Phase D.2 report is **partially correct, but not fully correct**.

- **Correct:** ARM `SET_IPAUTH_GROUP` instruction order; the four DEC reads of `*(u16*)0xB000001E`; the `F_45FD1` conditional descriptor flow; the `desc+0xF0 ← desc+0x38` count-like value; pointer installation at `desc+0xF8/+0xFC`; and buffer preparation inside `F_4618E`.
- **Incorrect or materially overstated:** calling `F_45DE0` an allocator; correcting `movhi` to `0x1C01`; describing `0xDFB1` as record creation; claiming ambiguous `+0x2E4` stores; omitting the verified `+0x2E8` printer input; and naming opaque diagnostic/config fields as DTS license state.
- **Not closed:** ARM/SE transport → `0xB000001E`; the identity and producer of `*(r4+0x10)`; the exact DTS-config output-arm state; F8/FC → staging/TX bridge; and physical SPDIF emission.
- **Historically strongest endpoint:** AC-3 succeeds while clean DTS produces no TX-visible non-PCM bytes. Historical `Pc=0x000B` bursts are not evidence of a sustained DTS-configured path and are not proof of physical-line lock.

The strongest independent conclusion is:

> DTS decoding machinery exists and has decoded DTS in historical P-BYP3 runs, but no verified static or controlled-runtime edge joins the ARM authorization state to a DTS-configured, TX-visible output frame. Phase D.2 does not close that edge.

---

## 2. Scope, constraints, and exclusions

This audit obeys the following constraints:

- Static and historical-corpus evidence only.
- No device, ADB, runtime, breakpoint, memory-read, module-reload, reboot, settings, property, flash, or patch action.
- No modification of binaries, Ghidra projects, old reports, logs, JSONL, scripts, or backups.
- No proposed patch, bypass, flashing recipe, or remediation design.
- Historical runtime captures are admissible only with their provenance, deployment surface, and causal limitations stated.
- TCL/T615 evidence is excluded from the present causal verdict. It is not used as a donor or as proof for this non-TCL investigation.

Epistemic rule used throughout:

```text
instruction verified != data-flow verified != semantic naming verified != causal relation verified
```

---

## 3. Method, evidence hierarchy, and status vocabulary

### 3.1 Evidence hierarchy

1. Raw binaries and exact hashes.
2. Ghidra/Reko listings with raw instruction bytes.
3. ISA definitions used to decode immediate fields and load/store semantics.
4. Instruction-level data flow and bounded negative scans.
5. Historical runtime captures and logs.
6. Prior reports.
7. Agent/operator interpretation.

A later prose report does not override contradictory raw bytes or ISA semantics.

### 3.2 Status vocabulary

- `VERIFIED`: directly established by raw bytes, instructions, or controlled historical observation.
- `STRONGLY SUPPORTED`: multiple consistent static observations, no material contradiction.
- `LIKELY`: best explanation, but a direct edge is missing.
- `UNPROVEN`: plausible semantic or causal claim without sufficient evidence.
- `DISPROVEN`: contradicted by direct evidence.
- `SUPERSEDED`: historically reasonable, but replaced by better evidence.
- `ARTIFACT/ERROR`: report, decoder, provenance, or interpretation error.
- `INERT-SURFACE`: bytes were deployed but never executed on the real loading path.
- `OPAQUE`: a named boundary not crossed by the available static corpus.
- `RUNTIME REQUIRED`: cannot be resolved honestly from static evidence.

---

## 4. Artifact identity and integrity

Hashes were recomputed during this audit.

| Artifact | Size | MD5 | SHA-256 | Role |
|---|---:|---|---|---|
| `patch_baseline/utpa2k_stock.ko` | 25,381,336 | `2fc6e9fc46b6402d73aa6f84f9599d46` | `8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da` | Stock ARM audio module and embedded DSP images |
| `patch_baseline/mik_stock.ko` | 8,084,248 | `c1421040dbcfff415f9417f482189435` | `44b0b3b66b7255bd5c91c88fef7e9688737dee1c1a9c91667095e4a98d91e70d` | Stock middleware |
| `aeon_validate/dec_work/dec_full.bin` | 1,982,492 | `4b7e9509b4358fd3a130bd4d3b9cbe0a` | `530bfa684bc363e8513951b731cd42ccae97c96accef3c1d51eb431b61a53f5b` | Executing MS12V22 DEC image |
| `aeon_validate/r27_work/snd_full.bin` | 1,839,920 | `eb879cdc07f510722f19db6d18d77d3c` | `1da795fa47aec5aec2aa4cdf78d52028d62abe911f6993dac664206b9210c22d` | Executing MS12V22 SND image |

Authoritative decoding material:

- DEC: `aeon_validate/dec_work/dec32_clean.txt`.
- ISA field definitions: `aeon_ghidra_public/aeon/data/languages/aeon.slaspec:141` and `aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc:451`.
- ARM AUTH/IDMA: `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt:21`.
- Loader: `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm:330`.

Identity is therefore not the source of the D.2 contradictions.

---

## 5. Historical chronology

| Date / state | Direct observation | Corrected interpretation |
|---|---|---|
| 2026-08-30 | License profile reconstructed; IDs `0xB/0xC` associated with AC-3/DD+ | `0xB/0xC` are not DTS IDs; DTS group was `{0xF,0x3A,0x12,7}`. `spdif_audio_investigation/REPORT_full_license_profile.md:64` |
| 2026-09-06 | `0x97` versus stock `0x04`; OTP/license-like diagnostics identified | `0x4D8` is read; `0x97` is conditional, not proven root cause. `aeon_validate/dec_work/REPORT_phaseC_corpus_synthesis_20260922.md:37` |
| 2026-09-08 | EDID/mode path considered | Real mode-selection mechanism, but not a complete DTS cause. `aeon_validate/dec_work/REPORT_phaseC_corpus_synthesis_20260922.md:39` |
| 2026-09-12 | `dts_01.bin` captured during DTS-labelled window | AC-3-shaped, frozen residual; not a DTS burst. `FINDING_wire_capture.md:44` |
| 2026-09-13 | Root-verified same-session A/B: AC-3 emitted 838 valid bursts; DTS emitted 0 B | Canonical TX-visible software capture; still not a physical-line measurement. `FINDING_wire_baseline_20260913.md:9` |
| 2026-09-13 | P-A report observed intermittent `Pc=0x000B` slots | Observation retained; patch causality later disproved because the deployed `/vendor` DEC did not execute. `REPORT_GLM53_full_audit_and_dts_repair.md:21`, `REPORT_current_state_K1_verdict_20260915.md:44` |
| 2026-09-14/15 | K-1 patched embedded DEC; K-2 patched ARM with stock embedded DEC | K-1/K-2 were real-surface experiments, neither produced DTS TX output. `REPORT_current_state_K1_verdict_20260915.md:67` |
| 2026-09-15 | N3 autopsy: mixed AC-3/DTS preambles and stock-equivalent behavior | Historical `0x000B` bursts were AC-3-configured emission carrying DTS bytes, not clean DTS-config output. `REPORT_current_state_K1_verdict_20260915.md:129` |
| 2026-09-16/17 | P-BYP3 restored DTS decode; P-BYP6 made monitor cadence codec-symmetric | Decoding and monitor execution are insufficient; TX remained zero. `MISSION_LOG_dts_persuit_20260916.md:35`, `MISSION_LOG_dts_persuit_20260916.md:101` |
| 2026-09-22 | Phase C/D/D.1/D.2 static closure reports produced | D.2 is bounded but contains material decoder, allocation, mapping, and semantic errors. `REPORT_phaseD2_final_static_residual_20260922.md:31` |

Historical evidence is therefore a sequence of progressively better-controlled observations, not a single uninterrupted proof chain.

---

## 6. Executing image and loader truth

`HAL_AUDSP_DspLoadCode` selects embedded symbols, not files under `/vendor/lib/utopia/audio_bin`:

- `mst_codec_r2` / `mst_snd_r2`, or
- `mst_codec_r2_MS12V22` / `mst_snd_r2_MS12V22` on the `r4==0x4A` loader path when the later checks pass and `g_AudioVars2+0x4D0 == 4`.

Primary evidence:

- Embedded symbol selection and the `r4==0x4A` branch: `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm:323`.
- `cmp r1,#4` and conditional MS12V22 symbol selection: `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm:349`.
- `memcpy` of the selected embedded image: `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm:440`.
- Historical marker controls and dormant-file conclusion: `REPORT_current_state_K1_verdict_20260915.md:46`.

Embedded stock image locations:

| Embedded image | `utpa2k.ko` file offset | Size | Content identity |
|---|---:|---:|---|
| `mst_codec_r2_MS12V22` | `0xA611AC` | `0x1E401C` | `dec_full.bin` |
| `mst_snd_r2_MS12V22` | `0xDD3404` | `0x1C1330` | `snd_full.bin` |

Consequences:

- `/vendor/audio_bin` replacements are `INERT-SURFACE` unless separately proven loaded.
- V1/L2/D1/P-A/N-series conclusions based only on `/vendor` DEC deployment cannot be causal results for the executing embedded image.
- K-1 and K-2 are valid because they changed the executing embedded DEC or ARM code.
- Selecting MS12V22 changes the DEC/SND pair; that is not evidence that the pair fixes DTS.

---

## 7. ARM AUTH to SE-IDMA construction

Primary function: `MDrv_AUDIO_ApplyHashkey` / `SET_IPAUTH_GROUP` at `0x424BD8`.

Verified sequence:

1. Choose DSP ID `0x83` for `r0==0`, or `0x84` for `r0==1`.
2. Build base `0x112A82`.
3. Clear mask at `0x112AC0`.
4. Write/poll control words at `0x112A7E/0x112A80`.
5. Write `0x112A84 = 0x83/0x84`.
6. Write `0x112A85 = 0x1F`.
7. Write `mask[15:8]` to `0x112A82`.
8. Write `mask[23:16]` to `0x112A83`.
9. Overwrite `0x112A82` with `mask[7:0]`.
10. Overwrite `0x112A83` with zero.

Evidence: `spdif_audio_investigation/tools/closure_utpa2k.txt:178` and `dd_MDrv_AUDIO_ApplyHashkey.txt:66`.

If the writes are ordinary final-state stores, the final architectural bytes are:

```text
0x112A82 = mask[7:0]
0x112A83 = 0
0x112A84 = 0x83 or 0x84
0x112A85 = 0x1F
```

Status split:

| Claim | Status |
|---|---|
| Instruction order and overwrite order | `VERIFIED` |
| Concrete DSP ID and `0x1F` | `VERIFIED` |
| Final mapped bytes if stores are last-write-wins | `VERIFIED` conditionally |
| Which writes the SE samples as a burst/FIFO sequence | `OPAQUE` |
| Exact live mask value | `RUNTIME REQUIRED` |
| Delivery to `0xB000001E` | `OPAQUE` |

Sibling and bulk pushers share the address but have different payload logic; they are co-tenants, not duplicate implementations.

---

## 8. DSP gate reads and the missing producer

Four DEC sites read the same 16-bit MMIO word and compare it with `5`:

| Compare site | Read width | Address | Status |
|---|---:|---:|---|
| `0x21AC3` | `lh` | `0xB000001E` | `VERIFIED` |
| `0x21B19` | `lh` | `0xB000001E` | `VERIFIED` |
| `0x21F0E` | `lh` | `0xB000001E` | `VERIFIED` |
| `0x21F5C` | `lh` | `0xB000001E` | `VERIFIED` |

The fail path also reads `*(u8*)0xB000001D` and masks with `0x7F`. Therefore a claim that every non-`5` value immediately becomes the named error is too strong; the corpus contains a separate `0x1D` path.

Report evidence: `aeon_validate/dec_work/REPORT_phaseD1_residual_static_20260922.md:114` and `aeon_validate/dec_work/REPORT_phaseD2_final_static_residual_20260922.md:100`.

A bounded corpus scan found:

- no direct ARM/DEC/SND instruction constructing and storing `0xB000001E`;
- raw literal hits in data-like regions, not coherent writers;
- no identified `AbsWriteByte` implementation that explains the SE-to-DSP side effect.

Correct status:

```text
No direct writer found in the bounded corpus = OPAQUE BOUNDARY
```

It is not valid to promote this to “no writer exists,” and it is not valid to claim the ARM writes directly reach this word.

---

## 9. DEC preparation, descriptor fields, and count semantics

The relevant path is:

```text
0x41275  build descriptor base
0x4127D  r4 = *(base + 0xA108)
0x41283  call F_45FD1(descriptor, r4)
0x41289  call F_4618E(descriptor)
```

Primary evidence: `aeon_validate/dec_work/dec32_clean.txt:82513`.

### 9.1 `r4` identity

- `r4 = *(base+0xA108)` is verified on the `0x41275` caller path.
- The same function reads `*r4` and `*(r4+0x10)` as branch inputs.
- No bounded static path cited here proves the pointee’s identity, a stable alias to `base+0x40`, or a copy from `base+0xE0+0xC`.
- The pointed object's semantic identity and its `+0x10` producer remain opaque.

Therefore `r4 = codec ID` and `*(r4+0x10) = DTS license flag` are both unsupported.

### 9.2 `F_45FD1`

Verified operations at `aeon_validate/dec_work/dec32_clean.txt:89038`:

```text
desc+0x28 = 1
desc+0x34 = 0
desc+0x38 = 0
if (*r4 == 1)          -> prepared row
else if (*(r4+0x10)==1) -> alternate row
else                    -> desc+0x28 = 0
```

The later prepared row computes `desc+0x38 = (index+1)<<5` at `0x4608E`.

### 9.3 `F_4618E`

At `aeon_validate/dec_work/dec32_clean.txt:89181`:

- If `desc+0x28 != 1`, `desc+0xF0` receives zero and the function returns.
- If `desc+0x28 == 1`, `r14 = desc+0x38`.
- The function uses `r14` as a 32-bit count/length class.
- It zero-fills `N*4` bytes in the two pointed buffers, transforms samples, and writes sync/header fields.

`desc+0xF0` is therefore count-like, not proven output byte length and not license data.

A store at `0x4462F` writes a repeated sentinel to one function-local base `r11+0x10`, not a proven producer for the `r4` pointee. No producer of the pointee's `+0x10 == 1` was established. The license claim does not follow.

---

## 10. F8/FC buffers: preparation verified, meaning and destination unproven

`F_45DE0` at `aeon_validate/dec_work/dec32_clean.txt:88860`:

- clears descriptor regions;
- loads pointers from an input table;
- stores `table[0]` to `desc+0xF8`;
- stores `table[1]` to `desc+0xFC`;
- calls the zero-fill helper around those pointer installations.

`0x130D25` is a byte-fill helper: it handles word, halfword, and byte tails (`aeon_validate/dec_work/dec32_clean.txt:410711`). It is not an allocator.

`F_4618E` then:

- zero-fills `N*4` bytes through both pointers;
- calls bit/sample readers;
- performs sign-extension/sample transformations;
- writes IEC-like metadata;
- writes `0x7FFE/0x8001` or `0x1FFF/0xE800` into the pointed data.

Correct status:

| Claim | Status |
|---|---|
| F8/FC pointer installation | `VERIFIED` |
| Zero-fill and buffer transformation | `VERIFIED` |
| “Allocation” by `F_45DE0` | `DISPROVEN` |
| F8/FC contain compressed DTS payload | `UNPROVEN` |
| F8/FC feed a physical SPDIF emitter | `UNPROVEN/OPAQUE` |
| F8/FC are license storage | `UNPROVEN`; diagnostic naming is not semantic proof |

---

## 11. D.2-F: `movhi` and `0xDFB1` correction

Raw instruction: `aeon_validate/dec_work/dec32_clean.txt:28770`.

```text
018803  bg.movhi r11,0xe0 | c1601c01
018807  muli r23,r12,0x290
01880B  addi r11,r11,0x26B0
01880F  add r11,r23
018817  jal 0xDFB1
```

The SLEIGH definition places the 16-bit `movhi` immediate at bits `5..20` (`aeon.slaspec:141`). Applied to `0xC1601C01`, it yields `0x00E0`, matching the listing. The resulting base is:

```text
0x00E00000 + 0x26B0 = 0x00E026B0
```

D.2’s proposed correction to `0x1C01` / `0x1C0126B0` is therefore `ARTIFACT/ERROR`. The separate data literal `0x1C010000` elsewhere in the corpus cannot override the instruction encoding.

`0xDFB1` contains cache invalidate/flush and write-buffer synchronization operations (`aeon_validate/dec_work/dec32_clean.txt:15160`). It is not statically an allocator or record constructor.

D.2-F verdict:

```text
Stride/address arithmetic: VERIFIED
Base 0x1C0126B0: DISPROVEN
0xDFB1 as allocator: DISPROVEN
Exact purpose of the call: OPAQUE, but cache maintenance is directly visible
```

---

## 12. SHM/license semantics and the `+0x10C` record claim

### 12.1 Printer data flow

Near `0x1CA37`, the printer loads:

- `+0x2E8` at `0x1C9FF`;
- `+0x2E0` at `0x1CA12`;
- `+0x2E4` at `0x1CA16`;

then saves the words to outgoing stack slots and calls the formatter at `0x1CA37` (`aeon_validate/dec_work/dec32_clean.txt:34087`). This proves three late outgoing diagnostic values, but does not by itself prove their exact `lic[...]` index order. `+0x2E6` is not loaded by this printer.

Therefore:

- D.2’s proposed `+0x2E6` third diagnostic input is `DISPROVEN` for this printer.
- D.2’s claim that the two writes to `+0x2E4` have unresolved path/drift ambiguity is `ARTIFACT/ERROR`; the halfword store deterministically follows the word store (`aeon_validate/dec_work/dec32_clean.txt:48368`).
- `+0x2E0/+0x2E4/+0x2E8 = DTS license state` remains `UNPROVEN`.

### 12.2 Writer/clear family

- `0x279AD/0x279BD` deterministically assemble and write buffer-derived 16-bit values to `+0x2E4`.
- `0xAE290/0xAE294/0xAE298` clear `+0x2E4/+0x2E5/+0x2E6` from `r0`; this is reset/clear behavior, not a proven license writer.
- `r1+740` sites are stack spills, never structure fields.

### 12.3 `record+0x10C`

The historical chain is:

```text
record+0x10C -> 0x22EAB load -> work+0x120 at 0x22EC8
```

The read and copy are `VERIFIED`. The claim that this value is the DTS verdict or a controlling license field is `UNPROVEN`; it is used diagnostically in the decoded path.

### 12.4 License-semantic ledger

| Object / field | Directly proven role | Unsupported leap |
|---|---|---|
| ARM `AV+0x440/+0x444` | AUTH-derived mask state sent by `SET_IPAUTH_GROUP` | Exact DSP consumer and payload semantics |
| `*(r4+0x10)` | Conditional input controlling descriptor preparation | DTS license/output-enable flag |
| `desc+0x28` | Prepared/mode flag controlling `F_4618E` | License verdict |
| `desc+0x38` | Count/size class | License value |
| `+0x2E0/+0x2E4/+0x2E8` | Printer inputs | Runtime license oracle |
| `record+0x10C` | Copied diagnostic/config value | Executing DTS verdict |
| `0x4462F +0x10` | Sentinel-fill destination on another base | `=1` producer |
| `0xAE290` family | Zero/reset family | License initializer |

---

## 13. DEC IEC61937 clusters A–F

The corpus contains six static Pa/Pb-writing clusters. Queue/control size values are not automatically queue capacities.

| Cluster | Pa/Pb sites | Static role and constants | Evidentiary limit |
|---|---|---|---|
| A | `0x24C0C/0x24C19` | Descriptor `+0x54`; helper argument `0xC00` | Queue/control path, not proven physical emitter |
| B | `0x24C85/0x24C8E` | Pc=`0x15`; helper argument `0x3000` | Queue/control path |
| C | `0x2645E/0x2646E` | Pc=`1`; Pd=`r18<<4`; gated by `+0x414/+0x418` | Queue/control path |
| D | `0x2ABAC/0x2ABB3` | Pc=`7`; Pd=`r23<<3`; zero-fill `0xFF8`; later `0x800` calls | Queue/control path |
| E | `0x2B5FC/0x2B603` | Pc=`1`; Pd=`r24<<3`; `0xA00`, later `0xC00` | Queue/control path |
| F | `0x45D07` and arms at `0x45D7F/0x45DA0/0x45DBF` | Header builder: AC-3 `0x10C`, DTS `0x0B`, third `0x20D`; Pa `0xF872`, Pb `0x4E1F` | Descriptor/header preparation, not physical emission |

Primary evidence: `aeon_validate/dec_work/dec32_clean.txt:88799`, `aeon_validate/dec_work/dec32_clean.txt:88855`, and `aeon_validate/r57_npcm/FINDING_dec_dts_output_SUPERSEDES_s24.md:85`.

The DES-like stream-entry pool is statically bounded to two active entries:

- count `+0x7E3C`;
- base `+0x6E24`;
- stride `0x80C`;
- configured flag `entry+0x808`;
- registration refuses `count > 1`.

Evidence: `aeon_validate/dec_work/dec32_clean.txt:90151`.

The DTS header-builder and syncwriter code and expected constants exist. Functional correctness and execution under stock DTS remain unproven. Static presence does not prove that the runtime descriptor reaches the builder, nor that any builder output reaches a TX FIFO or physical SPDIF sink.

---

## 14. SND IEC path and physical-emitter boundary

The SND image contains an IEC header/payload preparation function:

- Pa/Pb written at `r10+0x250C/+0x250E`;
- Pc/Pd at `r10+0x2510/+0x2512`;
- payload begins at `r10+0x2514`;
- E-AC-3 slot `0x6000`;
- DTS slot `0x1800`.

Evidence: `aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm:17739`.

Additional static facts:

- `0xCB50` performs format/sample-rate-dependent updates to `0xB0000854`; `0xCC9B` clears related state. The exact bit semantics remain unresolved (`aeon_validate/r27_work/snd_full.reko/snd_full_code_0000.asm:5447`).
- `0x25EE1` reads `0xB0000C06` and related state, then can call shared helper `0x13292` with `r5=0x6009`. The helper is arithmetic/structure-store code; no direct MMIO, DMA, FIFO, or physical-output role is proven (`aeon_validate/r27_work/snd_full.reko/snd_full_code_0002.asm:4296`).
- `0xB000085C` is used as a read-modify-write control interface (`aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm:10595`).
- Helper `0xB6D8F` accepts an object, output-byte pointer, and length pointer; it returns `-1` for a null object and `0` after preparing packed bytes (`aeon_validate/r27_work/snd_full.reko/snd_full_code_000B.asm:12180`). In the phase-correct caller, the return of `0xB544A` is stored at `mem+0x1C`; that value is then passed to `0xB6D8F`, whose own return becomes `r14` before the `0x1F6DA` test (`aeon_validate/r27_work/ghidra_dump_1B000_5400.txt:18763`, `aeon_validate/r27_work/ghidra_dump_1B000_5400.txt:18883`).

These establish format/control candidates, but not a direct byte-level edge:

```text
IEC header/payload preparation
    -> [unresolved queue/FIFO boundary]
    -> TX-visible non-PCM software capture
    -> [missing physical-output boundary]
```

The static corpus does not prove that the DEC F8/FC buffers, SND working buffers, a TX FIFO, and a physical output sink are the same object. The capture does not identify a concrete ring structure. The physical emitter relation remains `OPAQUE`.

---

## 15. Historical wire evidence and causality

### 15.1 Instrument validity

The root-verified same-session A/B is strong:

- AC-3: MB-scale dump, 838 valid `Pc=0x0001` bursts.
- DTS: 0 B, no `MI_PCM` reader.
- Same instrument and session.

Evidence: `aeon_validate/r57_npcm/FINDING_wire_baseline_20260913.md:10`.

This verifies the software TX-visible non-PCM capture behavior. The instrument is not an AVR/physical-line measurement.

### 15.2 `dts_01.bin`

`dts_01.bin` has:

- `Pc=0x0001`;
- `Pd=0x3000`;
- AC-3 sync `0x0B77`;
- repeated/frozen content.

Evidence: `aeon_validate/r57_npcm/FINDING_wire_capture.md:44`.

Verdict:

```text
dts_01.bin = stale/mislabeled AC-3 residual, not a DTS burst
```

The later controlled zero-byte result supersedes any “DTS bytes with wrong Pc” reading.

### 15.3 P-A and N3

The P-A report genuinely recorded structurally valid `Pc=0x000B` slots, DTS syncwords, and source-matched payload during its session (`REPORT_GLM53_full_audit_and_dts_repair.md:21`). That observation is not erased.

Its causal interpretation is nevertheless superseded:

- P-A was deployed only to dormant `/vendor` DEC under the real loader.
- N3 showed the same class of output with stock-equivalent embedded code.
- K-1/N3 attributed the phenomenon to AC-3-first/PAPlayer configuration carrying DTS bytes.
- Historical bursts were intermittent and did not establish sustained DTS-configured operation or AVR lock.

Primary correction: `REPORT_current_state_K1_verdict_20260915.md:46` and `REPORT_current_state_K1_verdict_20260915.md:129`.

### 15.4 P-BYP

P-BYP3 restored DTS decode and growing frame count while TX remained zero. P-BYP6 made the monitor/config cadence codec-symmetric, yet TX remained zero (`aeon_validate/MISSION_LOG_dts_persuit_20260916.md:35`, `aeon_validate/MISSION_LOG_dts_persuit_20260916.md:103`).

This closes simplistic claims that “engine `0x81` alone” or “monitor calls alone” explain output. It does not identify the exact live state that fails.

---

## 16. Independent D.2-A through D.2-G verdict matrix

| D.2 item | D.2 claim | Independent verdict | Material correction |
|---|---|---|---|
| A | Exact `0x112A82–85` construction | **MIXED: VERIFIED + OPAQUE** | Software sequence/final mapped state verified; FIFO/SE sampling and hardware payload opaque |
| B | No code writer to `0xB000001E`; bridge opaque | **OPAQUE, wording too absolute** | No direct writer found in bounded corpus; separate `0x1D` fallback exists |
| C | `r4` provenance and `*(r4+0x10)` gate | **MIXED** | Data flow verified; pointee identity, `=1` producer, and license semantics unproven |
| D | `r14 → desc+0xF0` | **STRONGLY SUPPORTED** | Value is `desc+0x38`, count/size-like, not license or proven byte count |
| E | F8/FC allocation and fill | **MIXED; allocation claim false** | Pointers come from input table; `0x130D25` is fill, not allocation; downstream meaning opaque |
| F | `movhi 0x1C01`, base `0x1C0126B0`, allocator `0xDFB1` | **DISPROVEN / ARTIFACT-ERROR** | Correct immediate `0xE0`; base `0xE026B0`; `0xDFB1` is cache maintenance |
| G | `+0x2E0/+0x2E4/+0x2E6` and ambiguous overwrites | **MIXED; mapping claim false** | Printer also loads `+0x2E8`; `+0x2E6` is not its third value; `+0x2E4` overwrite is deterministic; license semantics unproven |

Phase-level verdict:

```text
D.2 is a useful bounded audit, but it is not a correct final closure.
```

---

## 17. Evidence ledger

| ID | Claim | Primary anchor | Status |
|---|---|---|---|
| E01 | Stock artifact identities | Recomputed hashes | `VERIFIED` |
| E02 | Loader selects embedded DEC/SND; MS12V22 choice is conditional on the `0x4A` path, later checks, and `+0x4D0==4` | `HAL_AUDSP_DspLoadCode.asm:323` | `VERIFIED` |
| E03 | `/vendor/audio_bin` files are dormant under this loader | Loader plus marker controls | `VERIFIED` |
| E04 | `0x112A82–85` ARM construction order | `closure_utpa2k.txt:178` | `VERIFIED` |
| E05 | Final mapped bytes `[mask7,0,83/84,1F]` if stores are ordinary | Same ARM sequence | `VERIFIED` conditionally |
| E06 | SE samples all writes in that order | No ISA/hardware proof | `OPAQUE` |
| E07 | Four DEC `==5` reads of `0xB000001E` | D.1 raw instruction table | `VERIFIED` |
| E08 | Separate `0xB000001D` fail read | D.1 raw instruction table | `VERIFIED` |
| E09 | Direct writer to `0xB000001E` in bounded corpus | Bounded scan | `NOT FOUND`, not absolute absence |
| E10 | `r4=*(base+0xA108)` | `aeon_validate/dec_work/dec32_clean.txt:82513` | `VERIFIED` |
| E11 | `r4` is codec ID / license object | No producer/consumer proof | `UNPROVEN/DISPROVEN as asserted` |
| E12 | `desc+0x28` conditional clear | `aeon_validate/dec_work/dec32_clean.txt:89038` | `VERIFIED` |
| E13 | `desc+0x38=(index+1)<<5` on prepared path | `aeon_validate/dec_work/dec32_clean.txt:89104` | `VERIFIED` |
| E14 | `desc+0xF0=0 or desc+0x38` | `aeon_validate/dec_work/dec32_clean.txt:89181` | `VERIFIED` |
| E15 | F8/FC pointer installation | `aeon_validate/dec_work/dec32_clean.txt:88860` | `VERIFIED` |
| E16 | F8/FC are allocated by `F_45DE0` | Function structure | `DISPROVEN` |
| E17 | F8/FC contain compressed DTS | No downstream proof | `UNPROVEN` |
| E18 | F8/FC feed physical SPDIF | No direct edge | `OPAQUE` |
| E19 | `0x18803` base `0xE026B0` | Raw word + ISA | `VERIFIED` |
| E20 | `0xDFB1` allocator | Cache ops in body | `DISPROVEN` |
| E21 | Printer inputs `+0x2E0/+0x2E4/+0x2E8` | `aeon_validate/dec_work/dec32_clean.txt:34087` | `VERIFIED` |
| E22 | D.2’s proposed `+0x2E6` printer input | No `+0x2E6` load in function; `+0x2E8` is loaded | `DISPROVEN` |
| E23 | Those values are DTS license state | Diagnostic format only | `UNPROVEN` |
| E24 | `record+0x10C -> work+0x120` | Historical config walk | `VERIFIED` |
| E25 | `record+0x10C` is executing verdict | No consumer proof | `UNPROVEN` |
| E26 | DTS header builder exists | `aeon_validate/dec_work/dec32_clean.txt:88799` | `VERIFIED` |
| E27 | DTS syncwriter exists | `aeon_validate/dec_work/dec32_clean.txt:89266` | `VERIFIED` |
| E28 | Stock DTS produces end-to-end TX-visible non-PCM output | Controlled same-session 0-B result | `DISPROVEN` |
| E28b | Header builder executes/reaches TX under stock DTS | No runtime hit or descriptor snapshot | `UNPROVEN/OPAQUE` |
| E29 | `dts_01.bin` is a DTS burst | AC-3 header/frozen content | `DISPROVEN` |
| E30 | P-A patch caused historical bursts | Inert deployed surface + N3 | `DISPROVEN` |
| E31 | P-BYP3 decodes DTS | Runtime telemetry | `VERIFIED` historically |
| E32 | P-BYP3 outputs DTS TX | TX remained 0 B | `DISPROVEN` |
| E33 | Physical SPDIF relation of A/F path | Missing ring/peripheral edge | `OPAQUE` |
| E34 | MS12V22 selection is itself a DTS fix | No causal evidence | `UNPROVEN` |

---

## 18. Corrections and supersessions

| Historical claim | Final status | Reason |
|---|---|---|
| IDs `0xB/0xC` are DTS | `DISPROVEN` | They belong to AC-3/DD+; AC-3 succeeds |
| `0x43D/0x43E` are DTS bits | `DISPROVEN` | Dolby-premium status |
| `0x4D8` is never read | `DISPROVEN` | Start-chain load exists |
| `0x97` is the root cause | `SUPERSEDED` | Conditional profile command; stock commonly uses `0x04`; cmd04 had no effect |
| EDID alone blocks DTS | `DISPROVEN/SUPERSEDED` | Real mode mechanism, incomplete causal explanation |
| Decoder status `-1` is the DTS signature | `DISPROVEN` | Prior audits and controlled A/B |
| `Invalid Spatif license` was observed live | `UNPROVEN` | String/static path exists; no captured runtime occurrence |
| `0x460B1 / s->0x24` is the DTS blocker | `SUPERSEDED` | Generic bitstream-reader field; `0x460C2` is a geometry guard |
| `0x460C2` is proven to be the final blocker | `UNPROVEN` | Static guard exists, but runtime descriptor values were not captured |
| P-A caused the first DTS bursts | `DISPROVEN` | Deployed `/vendor` image was inert; N3 explains historical bursts |
| N-series effects came from patched DEC files | `DISPROVEN` | Dormant surface; effects were player/session variance |
| Holding engine `0x81` solves DTS | `DISPROVEN` | K-2 held `0x81`, ES filled, play/TX stayed zero |
| SND Pc-only patch explains zero output | `DISPROVEN` | Header writer receives no DTS frame in controlled run |
| `r4` is codec ID | `DISPROVEN as asserted` | Only pointer provenance is proven |
| `*(r4+0x10)` is a license flag | `UNPROVEN` | Conditional input; identity/producer absent |
| `0xAE290` initializes license state | `DISPROVEN as asserted` | Zero-clear family only |
| `record+0x10C` is the DTS verdict | `UNPROVEN` | Diagnostic copy verified, semantics absent |
| F8/FC are allocated by `F_45DE0` | `DISPROVEN` | Input pointers plus zero-fill |
| `0xDFB1` allocates DM records | `DISPROVEN` | Cache maintenance body |
| D.2’s `+0x2E6` printer-input claim | `DISPROVEN` | Printer loads `+0x2E8`, not `+0x2E6` |
| “wire” means physical SPDIF/AVR | `DISPROVEN as terminology` | `dump_spdif_npcm` is a software TX-visible capture |
| MS12V22 selection fixes DTS | `UNPROVEN` | It selects a different image pair; no end-to-end proof |
| Hashkey bits alone prove DTS causality | `UNPROVEN` | Stock bits were present, yet TX remained zero |

---

## 19. Independent causal graph

```text
[AUTH inputs]
gIpAuthVars / AV+0x440/+0x444
        |
        v
ARM CheckHashkey -> SET_IPAUTH_GROUP @0x424BD8       [VERIFIED]
        |
        v
SE-IDMA architectural bytes @0x112A82..85            [VERIFIED]
        |
        |
        +==== O1: sampling semantics / bridge ======+ [OPAQUE]
        |
        v
DSP *(u16*)0xB000001E                               [reader VERIFIED]
        |
        +==== O2: writer/protocol not found =========+ [OPAQUE]
        |
        v
four `==5` branches                                 [VERIFIED]
        |
        +---- fail path also reads 0xB000001D         [VERIFIED]

[DEC preparation]
r4 <- *(base+0xA108)                                [VERIFIED provenance]
F_45FD1:
  desc+0x28 / +0x34 / +0x38                        [VERIFIED]
  branch on *r4 and *(r4+0x10)                     [VERIFIED]
  pointee identity and +0x10 producer              [OPAQUE]
        |
        v
F_4618E:
  N=desc+0x38; desc+0xF0=0 or N                    [VERIFIED]
  F8/FC zero-fill/transform/sync                    [VERIFIED]
        |
        |
        +==== O3: compressed-DTS meaning ==========+ [UNPROVEN]
        |
        +==== O4: F8/FC -> TX bridge ==============+ [OPAQUE]

[SND/IEC path]
header/payload preparation @+0x250C                [VERIFIED]
0xB0000854 format/rate-dependent RMW                 [VERIFIED relation, semantic UNPROVEN]
0x25EE1 state/control candidate                     [VERIFIED code, TX role UNPROVEN]
        |
        +==== O5: control/staging -> TX FIFO ======+ [OPAQUE]
        |
        v
TX-visible non-PCM software capture
  AC-3: valid bursts                                [VERIFIED]
  DTS: 0 bytes in controlled A/B                    [VERIFIED]
        |
        +==== O6: physical SPDIF/AVR relation =====+ [OPAQUE]

[Historical alternate path]
AC-3-configured pipeline carrying DTS bytes
  -> syncword-classified Pc=0x000B slots            [VERIFIED observation]
  -> not sustained DTS-configured passthrough        [VERIFIED limitation]
```

The graph does not contain a verified edge from ARM authorization to physical DTS emission. It also does not justify replacing that missing edge with a guessed “license gate.”

---

## 20. Three most consequential errors and residual unknowns

### 20.1 Error 1 — semantic overclaim

Diagnostic and conditional fields were repeatedly promoted to “license state” or “DTS gate” without a producer/consumer chain:

- `*(r4+0x10)`;
- `desc+0x28`;
- `record+0x10C`;
- `+0x2E0/+0x2E4/+0x2E8`.

Consequence: historical experiments were organized around a presumed license mechanism that the static corpus does not close.

### 20.2 Error 2 — D.2-E/F instruction and tool interpretation errors

- `F_45DE0` installs pointers; it does not allocate them.
- `0x130D25` is a fill helper.
- `movhi` immediate is `0xE0`, not `0x1C01`.
- Base is `0xE026B0`, not `0x1C0126B0`.
- `0xDFB1` is cache maintenance, not proven allocation.

Consequence: D.2-F’s DM/record-creation narrative is materially false.

### 20.3 Error 3 — conflating preparation, queue control, software TX capture, and physical emission

The corpus contains header builders, queue helpers, staging code, and a software TX-visible dump. It does not statically join those into one physical SPDIF chain. Historical `0x000B` captures further mixed AC-3 configuration with DTS payload.

Consequence: “producer exists,” “TX-visible bytes exist,” and “physical DTS output exists” were treated as interchangeable when they are distinct claims.

### Residual unknowns

| Unknown | Status |
|---|---|
| Which component writes `0xB000001E` under each mode | `OPAQUE` |
| Whether IDMA byte writes are FIFO/burst/last-write-wins at the SE | `OPAQUE` |
| Identity of `*(base+0xA108)` and producer of pointee `+0x10` | `OPAQUE` |
| Exact live `desc+0x08/+0x28/+0x30/+0x38` for stock DTS | `RUNTIME REQUIRED` |
| Whether F8/FC is compressed payload, intermediate samples, or another representation | `UNPROVEN` |
| F8/FC → SND/TX queue edge | `OPAQUE` |
| Physical ring/peripheral bridge | `OPAQUE` |
| Why P-BYP6 monitor symmetry did not advance the port/output state | `UNPROVEN` |

These are evidence boundaries, not recommendations.

---

## 21. Final short conclusions

1. **D.2 is not fully correct.** Its ARM sequence, gate reads, descriptor flow, count-like `desc+0xF0`, and F8/FC preparation are useful; its allocation, `movhi`, `0xDFB1`, and printer mapping claims are not.
2. On the verified `r4==0x4A` loader path with later checks passed, the executing image is the embedded MS12V22 DEC/SND pair in `utpa2k.ko`; `/vendor/audio_bin` deployments are dormant.
3. ARM writes to `0x112A82–85` are verified, but SE sampling and delivery to `0xB000001E` remain opaque.
4. `*(r4+0x10)`, `desc+0x28`, `record+0x10C`, and `+0x2E0/+0x2E4/+0x2E8` must not be called DTS license state without a complete semantic chain.
5. DTS header and sync machinery exists statically; existence is not execution proof.
6. `dts_01.bin` is stale AC-3-shaped residual, not a DTS burst.
7. P-A/N3 `Pc=0x000B` observations are genuine captures but reflect AC-3-configured emission of DTS bytes, not proven sustained DTS-configured passthrough.
8. P-BYP proves DTS decoding can run while TX remains empty; decoding and monitor cadence do not explain output.
9. The strongest static endpoint is an unresolved host-auth/DSP-state/output-control boundary, not a proven license gate.
10. No static or controlled historical evidence closes the path to physical SPDIF emission.
11. No remediation is proposed in this audit.

---

## Appendix A — Primary local references

- `aeon_validate/dec_work/dec32_clean.txt`
- `aeon_validate/dec_work/REPORT_phaseC_corpus_synthesis_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD_static_closure_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD1_residual_static_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD2_final_static_residual_20260922.md`
- `aeon_validate/r57_npcm/FINDING_wire_capture.md`
- `aeon_validate/r57_npcm/FINDING_wire_baseline_20260913.md`
- `aeon_validate/r57_npcm/FINDING_dec_dts_output_SUPERSEDES_s24.md`
- `aeon_validate/r27_work/snd_full.reko/snd_full_code_0000.asm`
- `aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm`
- `aeon_validate/r27_work/snd_full.reko/snd_full_code_0002.asm`
- `aeon_validate/r27_work/snd_full.reko/snd_full_code_000B.asm`
- `aeon_validate/r27_work/ghidra_dump_1B000_5400.txt`
- `aeon_validate/MISSION_LOG_dts_persuit_20260916.md`
- `aeon_ghidra_public/aeon/data/languages/aeon.slaspec`
- `aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc`
- `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm`
- `spdif_audio_investigation/tools/closure_utpa2k.txt`
- `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt`
- `spdif_audio_investigation/REPORT_full_license_profile.md`
- `spdif_audio_investigation/REPORT_r4a_otp_license_provenance_20260906.md`
- `REPORT_current_state_K1_verdict_20260915.md`
- `REPORT_GLM53_full_audit_and_dts_repair.md`
- `REPORT_final_dts_patch_experiment.md`

## Appendix B — Integrity statement

- No prior report, binary, listing, log, script, Ghidra project, or backup was edited.
- No device/runtime action was performed.
- This report is the only new deliverable from the resumed audit.
