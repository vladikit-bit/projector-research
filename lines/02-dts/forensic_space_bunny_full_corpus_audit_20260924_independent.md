# FORENSIC SPACE BUNNY — FULL CORPUS INDEPENDENT AUDIT

**Date:** 2026-09-24  
**Phase:** 1 — static/corpus-only, read-only  
**Scope:** non-TCL Space Bunny DTS/SPDIF investigation  
**New deliverable:** this file only  
**Device/runtime:** no new ADB, device, playback, breakpoint, memory-read, module, settings, flash, or patch action  
**TCL/T615:** excluded from this audit; not searched, extracted, compared, or used as a donor/reference

---

## Executive Summary

The previous audit reached the correct high-level negative result: the corpus does **not** contain a verified end-to-end edge from ARM authorization state to a DTS-configured, physically emitted SPDIF frame. That negative result survives this independent review.

The current report is nevertheless not a valid “final closure” without corrections. The most important corrections are:

1. **D.2-A is partly too strong.** The ARM call sequence and overwrite order are verified. `HAL_AUDIO_AbsWriteByte` is an actual conditional mapped-byte store, not an unknown FIFO primitive. Therefore the final host-visible mapped target bytes are much better established than D.2 allowed; what remains opaque is SE sampling/bridge semantics and delivery to `0xB000001E`.
2. **D.2-B undercounts the gate readers.** The MS12V22 DEC image has **eight** coherent `*(u16*)0xB000001E == 5` comparison sites, not only the four later sites listed by D.1/D.2. Fourteen direct address constructions were found; the additional references are status reads. No direct construction-and-store writer was found in the bounded scan, so the bridge remains `OPAQUE`, not “no writer exists.”
3. **D.2-C is partly under-closed, but must remain path-scoped.** The direct `F_0411A6` expression is `r4=*(C+0xA108)` and `D411=C+0x14CCC`. Separate `F_03E0DD` stores reconstruct symbolic bases after helper calls; only on a path where those bases are proven equal may `r4` be identified as `B+0x7C4`. It is then a context/status object, not a codec ID or proven license object. The exact live producer of `*(r4+0x10)` remains unresolved.
4. **D.2-D is directionally correct but semantically overnamed.** `r14` reaches `desc+0xF0` from `desc+0x38` (or zero on the inactive path), and `desc+0x38` is used as a word/element count. It is not proven to be an output byte count or license value.
5. **D.2-E is wrong about allocation.** `F_45DE0` installs pre-existing pointers from an input table and fills them through `0x130D25`; it is not the allocator. The local F8/FC transform path is real, but no direct edge from those pointers to a physical SPDIF sink is proven.
6. **D.2-F contains a material ISA error.** The raw `movhi` word `0xC1601C01` decodes to immediate `0x00E0`, giving base `0x00E026B0`, not `0x1C0126B0`. `0xDFB1` is cache invalidate/flush and write-buffer synchronization, not an allocator or record constructor.
7. **The SHM/“lic” story contains a displacement error and a format-order error.** The old `0x21FD6` “`+0x2E6` license read” is actually `+0x2E4` after applying the AEON displacement field. The 0x1CA37 printer loads `+0x2E8`, `+0x2E0`, and `+0x2E4`, not `+0x2E6`; the three visible outgoing values map to the nested `frmCnt[...]` diagnostic group, not the final `lic[%d,%d,%d]` group. `lic[3] ↔ +0x2E6` remains disproven for this printer.
8. **The emitter conclusion must be narrower.** Clusters A–E are descriptor/ring/queue/control preparation paths. Cluster F is a header builder. The SND `+0x250C` function cited in earlier reports is AC-3/E-AC-3 preparation, not a proven DTS emitter. None is proven to be the physical SPDIF sink.
9. **The historical wire evidence is mixed and provenance-sensitive.** Raw captures contain real `Pc=0x000B` frames with DTS sync/payload, but the N3 capture also contains AC-3-labelled frames. The exact host configuration that produced the DTS-labelled frames is not encoded in the raw file. “AC-3-configured pipeline carrying DTS bytes” is a plausible explanation, not a raw-verified fact. P-A causality is disproven for the `/vendor` surface.
10. **A concrete TX-visible dump ring is statically identifiable.** `MDrv_AUDIO_Dump_SpdifNpcm_Monitor` constructs a vendor-labelled ring at `DSP base + 0x672000` with a `0x66000` span. This proves what the dump instrument reads, not that the ring is the physical transmitter or that a DTS-configured producer reaches it.

The strongest current conclusion is therefore:

```text
DTS decoding and IEC-style staging machinery exist.
Several historical software-TX captures are real.
But the ARM→SE→DSP authorization bridge, the live DTS descriptor/config state,
the F8/FC-to-output handoff, and the final physical SPDIF relation remain open.
```

No remediation or patch is proposed in this phase.

---

## 1. Corpus reviewed

### 1.1 Primary binaries and authoritative listings

| Artifact | Identity / role |
|---|---|
| `patch_baseline/utpa2k_stock.ko` | ARM stock module; MD5 `2fc6e9fc46b6402d73aa6f84f9599d46`; SHA-256 `8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da` |
| `patch_baseline/mik_stock.ko` | ARM stock MI module; MD5 `c1421040dbcfff415f9417f482189435` |
| `aeon_validate/dec_work/dec_full.bin` | Executing MS12V22 DEC image used for the D.2 listing; MD5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`; size `0x1E401C` |
| `aeon_validate/r27_work/snd_full.bin` | MS12V22 SND image; MD5 `eb879cdc07f510722f19db6d18d77d3c`; size `0x1C1330` |
| `aeon_validate/dec_work/dec32_clean.txt` | 637K-line AEON listing; primary oracle for DEC instructions and raw bytes |
| `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt` | ARM AUTH/ApplyHashkey disassembly |
| `spdif_audio_investigation/tools/closure_utpa2k.txt` | ARM xrefs and helper call census |
| `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm` | Embedded-image loader disassembly |
| `aeon_ghidra_public/aeon/data/languages/aeon.slaspec` | AEON field definitions |
| `aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc` | AEOV instruction semantics, including `movhi`, stores, cache operations, and displacement fields |
| `aeon_validate/r27_work/snd_full.reko/` | SND Reko/Ghidra-derived listings and raw cross-checks |

### 1.2 Historical/runtime artifacts

The corpus was indexed exhaustively as a file tree (over 1,100 text artifacts) and searched by address, field, hash, filename, and causal terms. Load-bearing artifacts were read directly, including:

- Phase C, D, D.1, and D.2 static reports;
- `MASTER_FORENSIC_REPORT_20260919.md`, `HANDOFF_20260921.md`, and the v2–v4 autonomous traces;
- ARM AUTH, CheckHashkey, ApplyHashkey, loader, SetSystem2, MI Start, and GetAudioInfo2 dumps;
- DEC/SND helper and emitter listings;
- EDID/display capability reports;
- K-1, K-2, P-A/N-series, P-BYP, and historical wire findings;
- raw wire captures including `dts_01.bin`, `N3_DTS_dts.bin`, `N3_AC3_ctl.bin`, `PA_probe1_dts.bin`, `K1_DTS_wire.bin`, and `K1CTL_AC3_wire.bin`;
- backup manifests and embedded-image extractions.

Lower-value raw runtime logs and archived JSONL trajectories were used for provenance and chronology, not treated as primary instruction evidence. No archived report remains modified; a temporary background-agent edit was restored byte-for-byte from the pre-task snapshot.

### 1.3 Evidence hierarchy

1. Raw binary bytes and exact ELF/AEON/ARM instruction listings.
2. ISA/SLEIGH field definitions and calling-convention definitions.
3. Instruction-level producer/consumer data-flow, with base-register provenance.
4. Historical runtime observations, provided deployment surface and capture provenance are explicit.
5. Prior reports and agent interpretations.
6. Narrative/archive material without direct artifact support.

A later report does not outrank contradictory bytes. A repeated claim is not independent evidence.

### 1.4 Status vocabulary

`VERIFIED`, `STRONGLY SUPPORTED`, `LIKELY`, `UNPROVEN`, `DISPROVEN`, `SUPERSEDED`, `ARTIFACT/ERROR`, `INERT-SURFACE`, `OPAQUE`, `BLOCKED`, and `RUNTIME REQUIRED` are used as in the user brief. “No writer found” means no writer found in the stated bounded scan, not proof that no hardware or indirect writer exists.

---

## 2. Investigation timeline

| Period | Observed experiment/result | Interpretation made at the time | Later correction/current state |
|---|---|---|---|
| 2026-08-26…30 | AC-3 worked; early DTS attempts produced silence; HAL/EDID and license hypotheses formed | DTS might be blocked by a host capability, decoder, or mode gate | AC-3 is a useful control, but early “DTS once worked” claims were later retracted because the path was transcoding or misclassified |
| 2026-08-30 | Static license reconstruction | IDs `0xB/0xC` were initially treated as possible DTS IDs | Primary `Get_AC3_License` paths establish `0xB/0xC` as AC-3/DD+ IDs; DTS group is `{0xF,0x3A,0x12,7}` |
| 2026-09-05…06 | EDID/mode and HAL gate experiments; `0x97` versus `0x04`; ARM AUTH/CheckHashkey analysis | EDID and a single command byte were promoted as likely root causes | EDID/mode mechanism exists but is not a closed DTS cause; `0x97` is conditional on `AV+0x4D8==3` and is not the sole blocker |
| 2026-09-08 | Full independent audit report claimed a broad ARM→DSP closure | ARM, DSP, and output layers appeared closed | Later work found unlinked SE/DSP state, inert surfaces, and conflated producer/consumer claims; closure was downgraded |
| 2026-09-11…13 | `dump_spdif_npcm` instrument discovered; AC-3 captures produced valid bursts; DTS captures were empty or stale | Wrong `Pc`, malformed burst, or stale DTS buffer | Root-verified software-TX A/B strongly supports AC-3 output versus DTS zero; `dts_01.bin` is AC-3-shaped/frozen, not a DTS burst |
| 2026-09-13…14 | P-A and N-series produced intermittent `Pc=0x000B` frames | P-A opened the DTS output path and license state was next blocker | P-A was deployed to dormant `/vendor` files on the observed loader path; the bursts were real bytes but patch causality is disproven |
| 2026-09-14…15 | K-1 embedded DEC and K-2 ARM experiments; N3 autopsy | K-1/K-2 isolated a real embedded/DSP or ARM cause | K-1 was a real-surface test and did not fix output; K-2 showed `0x81` plus ES activity is insufficient; N3 is mixed evidence |
| 2026-09-16…18 | P-BYP2…6, codec-differential traces, v2–v4 static paths | Decoder engine, monitor cadence, F8/FC, and A/B handoff became the main boundary | P-BYP3/P-BYP6 details are partly report-only; general DTS decode capability is historically supported, but the final TX relation remains open |
| 2026-09-19…22 | HOST AUTH, SHM38, D.2, D.2 residual reports | A “final” static closure was declared | D.2 contains real ISA, allocation, movhi, printer, and absence overclaims; this audit corrects them |
| 2026-09-23 | Prior full-corpus audit challenged D.2 | Central negative conclusion survived | This audit finds additional corrections: eight gate sites, partial `r4` identity closure, TX dump ring, SND AC-3 mislabel, wire provenance, and `+0x2E4` displacement error |
| 2026-09-24 | This independent report | No single verified DTS root cause | Current boundary is distributed across host AUTH/state, SE delivery, DSP descriptor/config, output attachment, and physical TX |

---

## 3. Loader truth and executing-surface audit

### 3.1 Embedded images

`HAL_AUDSP_DspLoadCode` directly selects embedded symbols and copies them. On the documented branch:

- `r4 == 0x4A` at `HAL_AUDSP_DspLoadCode.asm:323`;
- `g_AudioVars2+0x4D0 == 4` selects the MS12V22 symbols at `:354–365`;
- `memcpy` of the selected embedded image is visible at `:440–442`.

Embedded locations in `utpa2k_stock.ko`:

| Symbol | File offset | Size |
|---|---:|---:|
| `mst_codec_r2` | `0x7C50B8` | `0x29C0F4` |
| `mst_codec_r2_MS12V22` | `0xA611AC` | `0x1E401C` |
| `mst_snd_r2` | `0xC54ABC` | `0x17E948` |
| `mst_snd_r2_MS12V22` | `0xDD3404` | `0x1C1330` |

The standalone images in `spdif_audio_investigation/r2img/` are byte-identical to the corresponding embedded blobs, as independently hash-checked.

### 3.2 Consequence

A replacement under `/vendor/lib/utopia/audio_bin/` is `INERT-SURFACE` for the observed loader path unless an independent file-loading path is proven. This applies to the historical V1/L2/D1/P-A and N-series deployments that changed only those files.

This does **not** prove that every historical boot selected the same branch. `BACKUP_20260914_0717/README.txt` records an ALT-image execution/selection confusion, and the loader branch is conditional. The safe statement is:

```text
For the documented embedded-loader path, /vendor files are dormant snapshots.
Session-by-session image selection still requires the corresponding runtime evidence.
```

K-1 is different: it changed the embedded DEC blob and is a valid executing-surface experiment. K-2 changed real ARM text and restored the embedded DEC blob, so it is also not an inert `/vendor` test.

---

## 4. ARM → SE-IDMA audit

### 4.1 CheckHashkey state

Primary dump `spdif_audio_investigation/tools/ut_checkhashkey_full.asm` shows:

- `MDrv_AUDIO_CheckHashkey @ 0x423494`;
- final stores at `0x424870–0x424878`:
  - `AV+0x4D0 = r6`;
  - `AV+0x4D4 = r5`;
  - `AV+0x4D8 = r8`;
- `r8` is assigned `0`, `1`, `2`, or `3` along IP-check branches, including the DTS group checks.

`0x4D8` is therefore directly observed as an AUTH-derived level/state word. Calling it “the DTS license” is stronger than the bytes justify. Its downstream use in `SetSystem2` is verified; its complete entitlement semantics are not.

### 4.2 SET_IPAUTH_GROUP software sequence

`spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt:20–88` and stock ARM relocations establish this call sequence:

```text
base = 0x112A82
AbsWriteMaskReg(0x112AC0, 0xFFF0, 0)
AbsWriteMaskByte(0x112A7E, 1, 1)
CheckSeIdmaReady(8)
CheckSeIdmaReady(0x10)
AbsWriteByte(0x112A84, dspId)       ; 0x83 or 0x84
AbsWriteByte(0x112A85, 0x1F)
AbsWriteByte(0x112A82, mask[15:8])
AbsWriteByte(0x112A83, mask[23:16])
AbsWriteByte(0x112A82, mask[7:0])    ; overwrite
AbsWriteByte(0x112A83, 0)            ; overwrite
CheckSeIdmaReady(0x10)
```

The extraction and overwrite order are `VERIFIED`.

For a historical mask such as `0x00EDE227`, the software event stream is `E2 ED 27 00` for the first three mask bytes followed by the zero overwrite of `0x112A83`; this is compatible with the separately printed key1 value, but it is not a captured SE transaction and must not be called proof of the live payload.

The final **host mapped target state**, assuming the helper calls reach their store paths and no asynchronous hardware mutation occurs, is:

```text
0x112A82 = mask[7:0]
0x112A83 = 0
0x112A84 = 0x83 or 0x84
0x112A85 = 0x1F
```

### 4.3 AbsWriteByte is a real mapped-byte helper

`patch_baseline/utpa2k_stock.ko` exports `HAL_AUDIO_AbsWriteByte` at `0x441CDC`, size `0x218`. Independent ARM disassembly establishes:

- input address transformation: `addr - 0x100000`;
- aligned mapped byte offset calculation;
- read of the current mapped byte;
- conditional early return if the desired/current/shadow state already agrees;
- terminal mapped write at `0x441E7C`: `strb r4,[r7,r0]`.

Thus D.2’s suggestion that the final `strb` body was entirely unknown was too pessimistic. The host-side helper and final mapped-byte state are statically identified. The unresolved boundary is what the SE hardware samples from those writes and how/where that state reaches DSP MMIO.

There is no `0xB000001E` construction in the helper body. A direct ARM helper-to-`0xB000001E` writer is not established.

### 4.4 Sibling and bulk paths

These are co-tenants of the base, not duplicate implementations of `SET_IPAUTH_GROUP`:

| Function | Address range | Directly observed behavior |
|---|---:|---|
| `MDrv_Write_DSP_sram` | `0x423298` | Special `0x1F83/0x1F84` cases; generic path writes `0x112A84` from the low byte of `r6`, then different high/mask lanes and zeroes `0x112A83` |
| `HAL_Copy` | `0x42D7B4` | Writes `0x112A84/0x112A83` from `r4`, then streams array words into `0x112A82/0x112A83` with readiness checks |

The same address does not imply the same payload or the same caller.

There are also data-driven co-writers that a literal-only code scan would miss. `AudioInitTbl_0` in stock `utpa2k.ko` contains mask/value records for `0x112A82..85`, and `HAL_AUDIO_WriteInitTable` dispatches those records to the AbsWrite helpers. The boot sequence calls the init-table writer before CheckHashkey/ApplyHashkey. This reinforces the bounded-absence rule: no direct instruction literal is not proof that no writer exists.

---

### 4.5 Host `_MApi_AUDIO_SetDecodeSystem` and the `+0x10` claim

The host-side function is a separate ABI and must not be merged with the DEC `F_45FD1` `r4`.

In stock `utpa2k_stock.ko`, `_MApi_AUDIO_SetDecodeSystem` is at `0x3E52A8` (the older Ghidra dump is address-shifted). Its raw control flow is:

```text
r4 == 5 or r4 == -1
    -> bypasses the struct+0x10 test
    -> takes a different early path and does not take the direct tail-call at 0x3E546C

r4 != 5 and r4 != -1
    -> reads struct+0x10
    -> 0 or 0xFF enters the rejection/no-dispatch path
    -> another value continues to the AV[r4*0x28+0xFA] check
```

Therefore the universal statements “`+0x10=0xFF` always kills the gate” and “`+0x10=0xFF` always passes” are both too broad. The direct START argument value is supplied through `[instance+0xADC]` (`k_Start.asm:311–313`); its live value is not fixed by the static corpus. The universal old correction is `SUPERSEDED`; the path-specific dispatch condition is `VERIFIED`, while the live branch taken during DTS is `RUNTIME REQUIRED`.

---

## 5. SE-IDMA → DSP audit: `0xB000001E`

### 5.1 Reader inventory

The authoritative MS12V22 DEC listing contains **eight** coherent `== 5` comparison sites:

```text
0x138AE
0x13904
0x13D97
0x13DE5
0x21AC3
0x21B19
0x21F0E
0x21F5C
```

The first four are duplicate/earlier decoder paths omitted by the D.1/D.2 four-site table. The later four are the sites most often cited in the prior reports.

The broader listing has fourteen direct address constructions to `0xB000001E`. The other six are status/load references, including examples at `0xDCBD`, `0xEBE6`, `0xF853`, `0x12D9D`, `0x19575`, and `0x1A2B9`. They are reads, not stores.

Every explicit `== 5` failure path also reads `0xB000001D` as a separate byte/status value and masks it with `0x7F`. The two addresses must not be merged.

### 5.2 Writer scan

Fresh raw scans of the primary artifacts found:

| Artifact | Result |
|---|---|
| `utpa2k_stock.ko` | No direct `0xB000001E` literal in `.text`; hits are in `.ARM.exidx`/`.data`/relocation material |
| `dec_full.bin` | One LE literal hit at file offset `0x178D41`, in a data-like/table region; the surrounding decode is not a coherent writer |
| `snd_full.bin` | No 32-bit literal hit; address-construction regions found are status reads |
| `HAL_AUDIO_AbsWriteByte` | No `0xB000001E` construction; terminal operation is a mapped `strb` at the translated SE address |

No direct immediate-construction-and-store writer was found in the bounded corpus. That is not proof that no indirect writer, hardware producer, SE firmware path, or table-mediated writer exists.

The strongest separate DSP-side destination candidates are `0xB0000838` and `0xB000081C`, read by `r2_decoder_select` and printed as key fields. They are distinct addresses from `0xB000001E`; no direct producer chain from `0x112A82..85` to either key field or to `0xB000001E` is proven. The SND dump also shows an E3/F3-style request/ack use of `0xB000001E`, reinforcing that it behaves as a status/handshake word rather than a proven 24-bit key field.

**Status:** reader/data-flow `VERIFIED`; producer/bridge `OPAQUE`; the old absolute wording “zero code writers exist” is `ARTIFACT/ERROR`.

---

## 6. D.2-C — `r4`, `*(r4+0x10)`, `desc+0x28`, `desc+0x34`, `desc+0x38`

### 6.1 Path-specific r4 identity

The caller path is:

```text
0x41210  r11 = r3 + 0x10000
0x41275  r10 = 0x14CCC
0x4127D  r4 = *(r11 - 0x5EF8) = *(base + 0xA108)
0x41283  F_45FD1(r10, r4)
0x41289  F_4618E(r10)
```

The direct `F_0411A6` path has a precise symbolic identity. Let `C` be its entry `r3`:

```text
0x41210  r11 = C + 0x10000
0x41275  r10 = C + 0x14CCC
0x4127D  r4  = *(C + 0xA108)
```

`F_03E961` passes its `r11` entry value into `F_0411A6`, so if that value is called `B` on the specific call path, then `C=B` and the descriptor/r4 expressions become `B+0x14CCC` and `*(B+0xA108)`.

The apparent initialization stores at `0x3E1A6` and `0x3EA38` are real, but their effective bases are reconstructed after intervening helper calls. The safe symbolic statement is:

```text
[store_base@site - 0x5EF8] = source_base@site + 0x7C4
```

It is not automatically `[B+0xA108] = B+0x7C4` for every path. On a path where that store-base identity is proven, the `r4` pointee is a context/status object at `B+0x7C4`; otherwise its exact live pointee remains conditional.

The older D.2 statement that the pointee identity was wholly opaque is therefore too weak for the directly traced `F_0411A6` expression, but the cross-function initialization identity must remain symbolic. `*(r4+0x10)` is the adjacent field `B+0x7D4` only on that same path.

The pointer is a context/status object. `F_03E961` and the helper around `0x37D18/0x37D2D` read its `+0x00`, `+0x10`, and `+0x24` fields as status predicates. Nothing in this flow proves that the object is a codec ID or a license entitlement.

### 6.2 F_45FD1 branch

Primary instructions:

```text
0x45FDB  desc+0x28 = 1
0x45FE0  r23 = *r4
0x45FE3  desc+0x34 = 0
0x45FE6  desc+0x38 = 0
0x45FEB  if *r4 == 1 -> first row
0x45FEE  r11 = *(r4+0x10)
0x45FF1  if r11 == 1 -> alternate row
0x45FF5  otherwise desc+0x28 = 0
```

Supported interpretation:

- `desc+0x28` is a conditional prepared/mode flag.
- `*(r4+0x10)` is a boolean branch operand.
- Neither is proven to be a DTS license flag, capability bit, or entitlement.
- The direct `0x4462F` store is a sentinel/fill sequence, not a proven `+0x10 == 1` producer. Other same-offset stores exist, but they are not automatically the same base instance.

`desc+0x34` starts at zero and is later replaced by a result derived from the bit-reader helper. It is not a license field.

`desc+0x38` is set through a parser path as a value of the form `(source + 1) << 5`. The source is not proven to be a previous descriptor index.

### 6.3 Host `r4` is a different register

The ARM `_MApi_AUDIO_SetDecodeSystem` `r4` and the DEC `F_45FD1` `r4` are unrelated registers in unrelated ABIs. The host `r4` is the first API argument; the DEC `r4` is a pointer to a context object. Reusing the old “r4=codec ID” correction across both layers is invalid.

---

## 7. D.2-D — `r14` → `desc+0xF0`

Primary flow:

```text
0x461A1  r23 = desc+0x28
0x461A6  r14 = 0
0x461A8  if desc+0x28 == 1 -> 0x461C4
0x461AB  desc+0xF0 = r14

0x461C4  r14 = *(desc+0x38)
0x461D2/0x461DE  fill *F8 and *FC with r14<<2 bytes
0x46222…0x4624B  local word/element loop using r14
0x46259/0x4625C  conditional rejoin
0x4629C  metadata rejoin
```

The direct data-flow is `VERIFIED`:

```text
inactive path: desc+0xF0 = 0
prepared path: desc+0xF0 = desc+0x38 = N
```

Because `N<<2` is used for clearing and `N` is used as a loop bound, the strongest semantic label is:

```text
COUNT-LIKE / WORD-OR-ELEMENT-COUNT VALUE
```

It is not proven to be:

- an output byte count;
- a physical-ring length;
- a license value;
- a DTS verdict.

---

## 8. D.2-E — F8/FC preparation and the missing handoff

### 8.1 F_45DE0

Primary instructions at `dec32_clean.txt:88860–88890`:

```text
0x45DE9  r13 = input table
0x45DF7  fill helper, descriptor-sized clear
0x45E04  fill helper, +0x3C region
0x45E08  r3 = table[0]
0x45E17  desc+0xF8 = table[0]
0x45E21  fill helper at table[0], size 0x200
0x45E25  r3 = table[1]
0x45E2F  desc+0xFC = table[1]
0x45E39  fill helper at table[1], size 0x200
```

`0x130D25` is a word/halfword/byte fill helper with alignment and tail handling. It is not an allocator and does not return a newly allocated pointer.

**D.2-E allocation claim:** `DISPROVEN`.  
**Pointer installation and fill:** `VERIFIED`.

The upstream table is constructed in `F_03E0DD` using a symbolic base/offset expression. Exact absolute identity of every descriptor instance must remain path-scoped; numeric equality of `+0xF8`/`+0xFC` in another object is not an alias proof.

### 8.2 F_45E52 modes

The three call sites in the `F_0411A6` family (`0x411F5`, `0x41259`, `0x4126D`) are alternative setup/consumer branches. They are not three unconditional calls that must all execute before `F_45FD1`. The direct `0x41275 → 0x45FD1 → 0x4618E` path can bypass them.

Within the directly traced `F_0411A6` path, the calls use the descriptor base constructed for that path. The symbolic base relation between this path and the separate `F_03E0DD` construction at `0x3E325` is not established: intervening helpers overwrite `r10/r11/r14`, and the construction uses a symbolic `U + 0x14EE8/0x14F6C/0x14CCC`. Therefore the stronger claim “F_45DE0 initializes the exact F8/FC pointers later consumed by F_4618E” is withdrawn. The exact state/mode that invokes the alternative branches is also not established for the failing DTS session.

### 8.3 F8/FC downstream boundary

After `F_4618E` finishes its local fill, bit extraction, transformations, and sync/header writes, the immediate caller does not reload or pass `desc+0xF8`, `desc+0xFC`, or their pointees to a proven queue/DMA/SND consumer.

A later `R`-based object also uses numeric `+0xF8/+0xFC` offsets, but a different base is not an alias. SND `+0x250C` has the same Pa/Pb constants but no decoded pointer/call edge from the DEC buffers.

**Status:** local staging/transform `VERIFIED`; compressed-DTS meaning `UNPROVEN`; F8/FC→TX/physical `OPAQUE`.

---

## 9. D.2-F — stride, `movhi`, and `0xDFB1`

Raw instruction window:

```text
018800  lwz   r12,0(r3)
018803  movhi r11,0xe0       | c1601c01
018807  muli  r23,r12,0x290
01880b  addi  r11,r11,0x26b0
01880f  add   r11,r23
018813  ori   r4,r0,0x290
018817  jal   0xDFB1
```

SLEIGH defines `movhi` immediate as bits 5..20:

```text
(0xC1601C01 >> 5) & 0xFFFF = 0x00E0
0x00E0 << 16               = 0x00E00000
base + 0x26B0               = 0x00E026B0
```

Therefore:

- `0x1C01` is a low-word/incorrect extraction, not the `movhi` immediate;
- `0x1C0126B0` is `DISPROVEN`;
- the stride `0x290` and address arithmetic are `VERIFIED`;
- the nearby `0x1C010000` data literal cannot override the instruction field.

`0xDFB1` contains cache setup, `flush_invalidate`, `syncwritebuffer`, and looped cache maintenance. It is not an allocator or record constructor. The caller-specific reason for the cache operation remains opaque.

---

## 10. +0x2E0 / +0x2E4 / +0x2E6 audit

### 10.1 Correct state base

The printer at `0x1C93E` constructs:

```text
S(i) = 0x465840 + i*0x350
```

using the sequence around `0x1C958–0x1C98E`. It then loads:

```text
0x1C9FF  r23 = *(S+0x2E8)
0x1CA12  r25 = *(S+0x2E0)
0x1CA16  r24 = *(S+0x2E4)
```

and saves those values to outgoing stack slots before the formatter call at `0x1CA37`.

### 10.2 A direct `+0x2E4` producer/consumer

`F_021C6B` uses the same `S(i)` construction:

```text
0x21C74…0x21C8C  S(i) = 0x465840 + i*0x350
0x21CDA          *(S+0x2E4) = function argument r4
0x21FD6          r4 = *(S+0x2E4)
0x21FDA…0x21FE2  print "Invalid Spdif license:%d, output_spdifSz:%d"
```

The old `REPORT_dec_state_transition_20260906.md` decoded the raw displacement as `+0x2E6`; the authoritative listing and SLEIGH field give **`+0x2E4`**. This is a concrete `ARTIFACT/ERROR` in the earlier report.

The direct string/argument relationship makes `S+0x2E4` a plausible diagnostic license value for this function. It does not prove that the field controls the output branch, nor that it is the same object as every other `+0x2E4` site.

### 10.3 Printer argument order

The format string at `0x1427CE` is:

```text
arm->r2 %d:
state:%x(chg:%d,req=%d),
done:%x, ack:%x,
frmCnt[%04x,%04x(%04x,%04x),%04x,%04x]
smpRate[%d,%d,%d]
ovrFetchEs:%d,
spdif[s:%d,r:%d], hdmi[s:%d,r:%d],
outSz:[%d,%d,%d,%d],
calling:%04x,
lic[%d,%d,%d]
```

There are 28 conversions. The three SHM values saved at outgoing slots corresponding to the nested `frmCnt[...]` group are not the final three `lic[...]` arguments. The exact register/stack ABI is supported by the AEON calling convention and visible saves, so the frame-counter mapping is `STRONGLY SUPPORTED`; the key direct fact is stronger:

```text
+0x2E6 is not independently loaded by this printer.
lic[3] ↔ +0x2E6 is DISPROVEN for printer 0x1CA37.
```

D.2 itself already recorded the absence of a `+0x2E6` printer load. The actual omission in D.2 was the verified `+0x2E8` input, not a newly discovered `+0x2E6` input.

### 10.4 Other writers and stack false positives

The `r3`-based buffer path at `0x27982–0x279BD` deterministically performs a word store to `+0x2E4` followed by a halfword store to the same address. It is not a branch/path ambiguity; the second store overwrites one half of the previous word according to AEON memory semantics.

Other bases and paths exist, including `r10+0x2E4`, `r11+0x2E4`, and pointer-derived clear code. They must not be merged solely because the displacement is identical.

`0xAE290/0xAE294/0xAE298` clear bytes `+0x2E4/+0x2E5/+0x2E6` from `r0`. This is reset/clear-like behavior, not a proven license initializer.

The `r1+0x2E4` occurrences at `0xCE66B` and `0x129128` are stack spills (`sp+0x2E4`, decimal 740), not SHM fields. They remain excluded permanently.

### 10.5 D.2-G verdict

| Subclaim | Status |
|---|---|
| Buffer-derived writes to a `+0x2E4/+0x2E0/+0x2E8` family | `VERIFIED` on the cited bases |
| Printer loads `+0x2E0/+0x2E4/+0x2E8` | `VERIFIED` |
| Printer loads `+0x2E6` | `DISPROVEN` for this function |
| Printer values are `lic[]` values | `DISPROVEN/UNPROVEN` depending on the claimed index; format order strongly rejects the old mapping |
| `+0x2E4` is the value printed by the invalid-license diagnostic | `VERIFIED` on the `S(i)` instance |
| `+0x2E4` controls output | `UNPROVEN` |
| `0xAE290` writes license state | `DISPROVEN as asserted` |

---

## 11. License-semantics ledger

“License” below means only a field with a direct AUTH/license relationship. Diagnostic strings, conditional fields, and same-sized buffers are not enough.

| Field/object | Direct evidence | Diagnostic/branch use | Semantic interpretation | Confidence |
|---|---|---|---|---|
| ARM AV `+0x440/+0x444` | CheckHashkey mask accumulation; ApplyHashkey loads them | Passed to SET_IPAUTH_GROUP | AUTH-derived mask words | `VERIFIED` role; exact feature names partly inferred |
| ARM AV `+0x4D0` | CheckHashkey writes it; loader uses it for image selection | Chooses embedded image pair | DSP/profile selector | `VERIFIED`; not a DTS license by itself |
| ARM AV `+0x4D8` | CheckHashkey writes 0/1/2/3; SetSystem2 reads it | Chooses `0x97` versus `0x04`; DTS/SHM38 paths read it | AUTH-derived DTS level/state | `STRONGLY SUPPORTED`; “license” alone overclaims |
| SE bytes `0x112A82–85` | SET_IPAUTH_GROUP and sibling/bulk sequences | Mapped-byte helper; readiness polling | AUTH mask/SE command payload | `VERIFIED` software sequence; live SE semantics `OPAQUE` |
| `0xB000001E` | Eight `==5` reads plus other status reads | Gate branch and `0x1D` fallback | DSP capability/status word | Reader `VERIFIED`; producer and exact license meaning `OPAQUE` |
| DEC `r4` | Direct `F_0411A6`: `*(C+0xA108)`; separate `F_03E0DD` stores are symbolic/path-scoped | `F_45FD1` branches on `*r4` and `*(r4+0x10)` | Context/status object when the initialization base is proven | Expression `VERIFIED`; live pointee identity conditional; license/codec name `UNPROVEN` |
| `*(r4+0x10)` | Conditional equality-to-1 operand | Selects alternate descriptor row | Boolean mode/status gate | `VERIFIED` role; producer/semantics `OPAQUE` |
| `desc+0x28` | Set to 1 then conditionally cleared | Controls early emit path | Prepared/mode flag | `VERIFIED`; license verdict `UNPROVEN` |
| `desc+0x34` | Initial zero; later bit-reader-derived result | Internal state | Bit-reader/stream state | `VERIFIED` role; license `UNPROVEN` |
| `desc+0x38` | Parser-derived `(source+1)<<5`; used as loop/count | Fill size and loop bound | Count-like value | `VERIFIED` data-flow; exact semantic class `STRONGLY SUPPORTED` |
| `desc+0xF0` | `0` or `desc+0x38` | Written at emit return | Count/handshake-like value | `STRONGLY SUPPORTED`; output bytes/license `UNPROVEN` |
| `desc+0xF8/+0xFC` | Installed from table, cleared, transformed | Local A/B staging | Pre-existing staging pointers | `VERIFIED`; compressed DTS/TX meaning `UNPROVEN` |
| `S(i)+0x2E0` | Buffer/table-derived word; printer frame-counter slot | `frmCnt[...]` diagnostic | Frame/status counter | `STRONGLY SUPPORTED`; license `DISPROVEN` as generic claim |
| `S(i)+0x2E4` | Function argument written at `0x21CDA`; read by invalid-license print and printer | Diagnostic license text and frame-counter group | State/diagnostic value with partial license association | `STRONGLY SUPPORTED`; control semantics `UNPROVEN` |
| `+0x2E6` | Byte/halfword in a 32-bit field; no printer load | Clear/read in other paths | Unnamed byte/field portion | `UNPROVEN`; not `lic[3]` for 0x1CA37 |
| `S(i)+0x2E8` | Buffer/table-derived word; printer load | `frmCnt[...]` diagnostic | Frame/status counter | `STRONGLY SUPPORTED`; license `UNPROVEN` |
| `record+0x10C` | Historical layout/copy path to `work+0x120` | Diagnostic/config copy | Record coordinate/value | Copy `VERIFIED`; “DTS verdict” `UNPROVEN` |
| `0x4462F +0x10` | Repeated sentinel fill on another base | Fill operation | Sentinel/fill data | Not a `+0x10==1` license producer |
| `0xAE290` family | Zero/clear byte stores | Initialization/reset-like | Not a license initializer | `DISPROVEN as asserted` |

---

## 12. Claims independently verified

- Embedded-image loader selection and `/vendor` inertness on the documented path.
- CheckHashkey mask construction, SET_IPAUTH_GROUP call order, overwrite order, sibling/bulk co-writers, and host mapped-byte state.
- `movhi` immediate `0xE0`, computed `0xE026B0` base, and `0xDFB1` cache-maintenance role.
- Eight DEC `0xB000001E == 5` comparison sites and the separate `0xB000001D` fallback.
- Direct `F_0411A6` expressions for `D411=C+0x14CCC` and `r4=*(C+0xA108)`.
- F8/FC pointer installation and fill behavior; D.2-E allocation is disproven.
- `r14`/`desc+0xF0` data flow and count-like use of `desc+0x38`.
- Printer loads of `+0x2E8/+0x2E0/+0x2E4`, absence of a `+0x2E6` printer load, and the `0x21FD6 → +0x2E4` displacement correction.
- Raw mixed wire-capture classifications and the existence of a software TX dump ring.

## 13. Claims superseded

- “P-A caused the first DTS bursts” → superseded by loader truth and mixed-capture evidence.
- “`0x460B1` is the DTS blocker” → superseded; it is generic bitstream-reader context.
- “`0xAE290` initializes license state” → superseded; it clears bytes.
- “`0xB/0xC` are DTS IDs” → superseded; they are AC-3/DD+ IDs.
- “`0x97` is the root cause” → superseded as a sole-cause claim.
- “`+0x10=0xFF` universally kills or universally passes the host gate” → superseded; branch scope matters.
- “F8/FC are allocated by `F_45DE0`” → superseded; it installs and fills pointers.
- “D.2 `movhi=0x1C01` / base `0x1C0126B0` / `0xDFB1` allocator” → superseded by ISA and helper body.

## 14. Claims disproven or not established

- `lic[3] ↔ +0x2E6` for printer `0x1CA37` → disproven.
- P-A `/vendor` causality → disproven for the observed loader path.
- A direct `0xB000001E` writer in the bounded corpus → not found; universal nonexistence is not proven.
- A DTS-configured physical SPDIF output path → not established.
- AC-3-configured emission of every N3 `Pc=0x000B` frame → plausible/likely, but not raw-verified.
- `F_45DE0 → F_4618E` same-pointer handoff → not established after symbolic re-check.
- A universal “no writer/no consumer/no physical path” claim → disproven as an evidence-standard overreach.

## 15. Possible overclaims

The principal overclaims are semantic inflation (`license`, `verdict`, `DTS-configured`), surface inflation (`/vendor` equals executing firmware), physicality inflation (software TX dump equals SPDIF line), and decoder inflation (a raw immediate suffix or same offset equals the same field). These are not cosmetic wording issues: each can redirect an investigation toward the wrong component.

---

## 16. Independent Phase D.2-A through G matrix

| Item | D.2 author claim | Primary verification | Independent status | What remains missing |
|---|---|---|---|---|
| D.2-A | Exact SE payload bytes at `0x112A82–85` | ARM sequence, overwrite order, helper body, and mapped-byte final path verified | **PARTLY VERIFIED / PARTLY OPAQUE** | SE sampling, FIFO/burst behavior, and bridge to DSP MMIO |
| D.2-B | SE→`0xB000001E`; no code writer found | Eight `==5` sites, fourteen direct constructions, no direct construction-and-store in bounded scan | **OPAQUE; D.2 reader count incomplete** | Producer, indirect writer, hardware/SE bridge |
| D.2-C | `r4` provenance and branch; pointee opaque | Direct path gives `r4=*(C+0xA108)`; separate `F_03E0DD` stores are symbolic/path-scoped | **PARTLY CLOSED / CONDITIONAL** | Live initialization base and producer of `+0x10==1`; no license naming |
| D.2-D | `r14` from `desc+0x38`, `F0` count/size-like | Direct load, fill, loop, and rejoin verified | **STRONGLY SUPPORTED** | Exact count/handshake consumer; never call it output bytes |
| D.2-E | F8/FC allocation and fill | Pointer installation and fill verified; allocator claim false | **MIXED; allocation DISPROVEN** | Buffer semantics and downstream handoff |
| D.2-F | `movhi=0x1C01`, base `0x1C0126B0`, `0xDFB1` allocator | SLEIGH field and cache-maintenance body contradict all three | **MATERIAL ERROR** | Only broader caller-specific cache purpose remains opaque |
| D.2-G | Buffer/SHM copies; no `+0x2E6` printer input; ambiguity | `+0x2E8` load verified; `+0x2E4` sequence deterministic; format order rejects generic lic mapping | **MIXED; semantic mapping corrected** | Exact `lic[]` source/consumer and live field values |

**Phase D.2 verdict:** useful bounded static work, not a correct final closure. It should be treated as a source of hypotheses and anchors, not as proof of DTS output causality.

---

## 17. IEC61937 emitter-cluster audit

All six requested clusters write the constant pair `Pa=0xF872`, `Pb=0x4E1F`. That fact alone does not identify a physical sink.

| Cluster | Pa/Pb | Pc/Pd and source | Helper/structure | Caller/producer status |
|---|---|---|---|---|
| A `0x24C0C/0x24C19` | `F872/4E1F` | Pc `1`; Pd `(*(u16*)(r10+0x5C)-4)<<4`; header `P=*(r10+0x54)`, source `Q=*(r10+0x248)` | `0x24C2E → 0x13602` with `r5=0xC00`; `0x24C4C → 0x130E0D` copies `Q` to `P+8` | Direct caller region around `0x266D7/0x24BD5`; producer of P/Q not closed; queue/control path |
| B `0x24C85/0x24C8E` | `F872/4E1F` | Pc `0x15`; Pd `(*(u16*)(r10+0x5C)-4)<<1`; same P/Q family | `0x24CAC → 0x13602` with `r5=0x3000`, then same copy path | Same caller family; no independent producer or physical proof |
| C `0x2645E/0x2646E` | `F872/4E1F` | Pc `1`; Pd `*(u16*)(r10+0x96)<<4`; header `P=*(r10+0x50)` | `0x12250`; optional `0x13B08/0x13602`; indirect/state-machine reachability | Containing function begins around `0x24652`; producer of +0x50/+0x96/+0x414/+0x418 unresolved |
| D `0x2ABAC/0x2ABB3` | `F872/4E1F` | Pc `7`; Pd `*(u32*)(r10+0x170)<<3`; header `P=*(r3+0xC0)` | `0x130D25` fills `0xFF8-count`; conditional `0x13B08/0x13602` | Callers around `0x2C32D/0x2C39A`; no codec-specific producer proven |
| E `0x2B5FC/0x2B603` | `F872/4E1F` | Pc `1`; Pd `r24<<3`; source/header `r11+0xBC/r11+0xC0` | `0x87394` preparation with `0xA00`; six `0x130E0D` copies; optional queue helpers | Callers around `0x2C204/0x2C384`; producer and physical sink unresolved |
| F `0x45D07` and arms | `F872/4E1F` | `r4=1 → 0x10C` internal arm word; `r4=0 → 0x0B`; `r4=2 → 0x20D`; Pd `r5<<3` | Writes descriptor header metadata at `+0x16C..+0x173`; callers `0x45FC9/0x460EF/0x4616A/0x46186` | Header builder/staging path, not physical emitter; live DTS branch selection unproven |

Important address correction: `0x24C28` and `0x24C35` are argument-setup instructions, not calls. The actual A-path calls are `0x24C2E → 0x13602` and `0x24C4C → 0x130E0D`. The B path uses `0x24CAC → 0x13602` and rejoins the same copy path.

### 13.1 Helper classification

| Helper | Static behavior | Correct label |
|---|---|---|
| `0x130D25` | Repeated byte/halfword/word fill with tail handling | Fill helper; not allocator |
| `0x130E0D` | Generic byte copy with alignment handling | Copy helper; not TX submission |
| `0x13602` | Cursor/count/index/list manipulation and copy helpers | Ring/queue/control candidate; not proven physical emitter |
| `0x13B08` | Similar ring/queue path with cursor/state updates | Ring/queue/control candidate |
| `0x87394` | Stack-based packet/frame preparation | Preparation helper; no physical sink proof |
| `0x64369` | Reads bitstream words and writes caller-supplied outputs | Local extraction/transform, not downstream consumer |

### 13.2 SND path

The cited SND function at `snd_full.reko/snd_full_code_0001.asm:17739–17884` writes a header at `r10+0x250C` and payload at `+0x2514`, but its directly observed branches are:

- E-AC-3: Pc `0x15`, Pd `0x6000`;
- AC-3: Pc `0x0001`, Pd `0x1800`;
- AC-3 sync check for `0x770B` byte-swapped form.

It must not be cited as a proven DTS emitter. No decoded pointer/call edge connects the DEC F8/FC buffers to this SND function.

---

## 18. Physical SPDIF relation and the TX dump ring

### 14.1 A concrete software TX-visible ring exists

The stock module exports `MDrv_AUDIO_Dump_SpdifNpcm_Monitor` at `0x421654` (size about `0x1CC`). Independent ARM disassembly shows:

- reads a DSP SRAM/control cell at `0x1114`;
- obtains a DSP base;
- constructs a ring source at `base + 0x672000` (`0x421690–0x42169C`);
- handles ring portions and wrap;
- uses span `0x66000` at `0x421788`;
- writes portions to the dump file.

This is a concrete vendor-labelled source for the software TX-visible capture. It corrects the earlier statement that the capture identifies no ring at all.

What it does **not** prove:

- that this ring is the physical SPDIF transmitter;
- that a DTS-configured producer writes it;
- that a DTS `Pc=0x0B` frame reaches it;
- electrical/optical output or receiver lock.

**Status:** dump-ring provenance `VERIFIED/STRONGLY SUPPORTED`; ring-to-physical-TX relation `OPAQUE`.

### 14.2 Historical wire evidence

Independent raw scan:

| File | Direct result | Safe interpretation |
|---|---|---|
| `dts_01.bin` | 44 `F872` preambles; all Pc `0x0001`, Pd `0x3000`; AC-3 sync `0x0B77`; repeated/frozen blocks | AC-3-shaped frozen/mislabeled capture; not a DTS burst. “Stale origin” is not proven without an exact prior-frame match |
| `PA_probe1_dts.bin` | 114 Pc `0x000B`, Pd `0x3EE0`, DTS sync/payload | Real DTS-framed TX-visible bytes existed in that historical capture |
| `N3_DTS_dts.bin` | 152 Pc `0x000B` DTS frames plus 273 Pc `0x0001` AC-3 frames | Mixed capture; coexistence is verified, exact producer/configuration is not |
| `N3_AC3_ctl.bin` | 114 Pc `0x000B` plus 476 Pc `0x0001` | AC-3 control capture is also mixed; not a clean DTS-config proof |
| `K1_DTS_wire.bin` | 408 Pc `0x0001`, AC-3 sync | Residual/stale AC-3 content despite filename; no new DTS bytes |
| `K1CTL_AC3_wire.bin` | 680 Pc `0x0001` | Valid AC-3 software-TX control |

The same-session report’s “838 AC-3 bursts / DTS zero” is directionally useful, but the preserved `ab_AC3/D_04.bin` contains 816 bursts; 838 belongs to a different earlier capture. The exact DTS zero-byte file was not preserved; the zero is supported by `files.txt` and lifecycle logs. The claim “no `MI_PCM` reader” is not preserved and is contradicted by later `MI_PCM_Open` records. The correct status is:

```text
AC-3 versus DTS software-TX contrast: STRONGLY SUPPORTED.
Exact 838 count and no-MI_PCM subclaim: PROVENANCE ERROR / NOT VERIFIED.
```

### 14.3 AC-3-configured DTS-byte explanation

K-1/N3 reports and player-path observations make the following explanation plausible:

```text
AC-3 configuration/priming plus DTS payload bytes
→ framer classifies by DTS syncword
→ Pc=0x000B frames appear in the software-TX capture
```

The raw N3 file proves mixed output and real DTS payloads, but it does not encode which host configuration emitted each frame. Therefore:

- `DTS-configured sustained output`: `UNPROVEN`;
- `AC-3-configured pipeline carrying DTS bytes`: `LIKELY/STRONGLY SUPPORTED`, not raw-verified;
- `physical SPDIF output`: `UNPROVEN`.

---

## 19. Historical “first DTS bursts” audit

The old P-A report’s observation was not fabricated: `Pc=0x000B`, correct Pd, DTS sync, and changing payload were present in `PA_probe1_dts.bin`. The causal interpretation was wrong or at least unproven:

1. P-A was deployed under `/vendor`, inert on the documented embedded-loader path.
2. N3 later produced mixed AC-3/DTS frames under a player/session history.
3. K-1 and K-2 did not produce a clean DTS-configured TX stream.
4. P-BYP3/P-BYP6 details are partly preserved only as report claims, not complete primary runtime pairs.

**OLD STATE:** P-A opened the DTS output path; first bursts were caused by the patched DEC.  
**CONTRADICTING PRIMARY EVIDENCE:** loader truth, raw mixed N3 capture, K-1/K-2 logs, and P-BYP reports.  
**CORRECTED STATE:** DTS-framed bytes definitely appeared in some software-TX captures; P-A causality is disproven, and a DTS-configured physical/direct output path is not proven.  
**STATUS:** `SUPERSEDED` for “P-A caused DTS output”; `UNPROVEN` for the exact configuration that produced the bursts.

---

## 20. Standard vs MS12V22

### 16.1 What is verified

- The loader has distinct embedded Standard and MS12V22 image pairs.
- The selection is conditional on the documented `r4`/profile path and `AV+0x4D0==4` checks.
- The D.2 addresses and much of the detailed listing refer to the MS12V22 DEC image.
- The standalone Standard and MS12V22 images have different sizes, hashes, layouts, and string inventories.

### 16.2 What is not verified

The raw Standard image contains `F872/4E1F` constructions at different offsets and DTS sync constants at different addresses, so analogous IEC machinery exists in both images. That does not prove one-to-one function identity, identical control flow, or that switching to MS12V22 fixes DTS.

The earlier “same heads/identical behavior” statements are bounded comparisons, not proof of semantic equivalence across the full images. The loader branch itself is not evidence that MS12V22 is a DTS solution.

**Status:** image selection `VERIFIED` subject to branch conditions; semantic equivalence `UNPROVEN`; MS12V22-as-DTS-fix `UNPROVEN/DISPROVEN as a causal claim without an A/B experiment`.

---

## 21. Reasoning and methodology audit

The following reasoning errors recur in the corpus and materially affect conclusions:

1. **Wrong execution surface.** Assuming a reboot made `/vendor/audio_bin` active ignored the embedded loader truth.
2. **Temporal correlation promoted to causation.** `type[81] → type[4]`, hashkey tuple changes, and wire bursts were treated as proof of the mechanism that produced them.
3. **Decoder/field drift.** Linear AEON sweeps, raw immediate suffixes, and halfword displacements produced errors such as `movhi=0x1C01` and `+0x2E6` instead of `+0x2E4`.
4. **Same offset treated as same object.** `r1+0x2E4`, `r3+0x2E4`, `r10+0x2E4`, and S(i)+0x2E4 were merged without base provenance.
5. **Same constant treated as same semantic payload.** Pa/Pb constants do not prove that DEC, SND, and TX rings are the same object.
6. **Diagnostic string treated as control proof.** `Invalid Spdif license`, `lic[]`, `licensee`, and `UnsupportedType` do not by themselves identify the writer, branch, or output consequence.
7. **Decode success treated as output success.** P-BYP/K-2 show DTS decoding/ES activity can coexist with zero TX output.
8. **Allocation/fill conflation.** `0x130D25` fills memory; it does not allocate the buffer.
9. **Bounded absence treated as universal absence.** “No direct writer found” was sometimes rewritten as “no writer exists.”
10. **Software TX called physical wire.** `dump_spdif_npcm` is not an oscilloscope, optical receiver, AVR, or electrical-line measurement.
11. **Multi-patch confounding.** P-A/N-series results mixed image deployment, player state, AC-3 priming, and code changes.
12. **Cross-layer `r4` aliasing.** Host API `r4` and DEC descriptor `r4` were treated as one object.
13. **Diagnostic history overread.** “Invalid Spatif license” string presence was treated as a live observed failure without a captured occurrence.
14. **Configuration inference from bytes.** Mixed wire frames do not identify the host configuration that produced each frame.

---

## 22. Top 3 consequential possible errors

### 18.1 Wrong execution surface

**OLD CLAIM:** P-A, L2, D1, and N-series image patches tested the live DTS output path; P-A caused the first DTS bursts.

**WHY IT MATTERS:** It invalidated an entire experimental branch and made later “the site is disproven” conclusions unsafe.

**PRIMARY EVIDENCE:** `HAL_AUDSP_DspLoadCode.asm:323–365,440–442`; embedded symbol offsets; backup marker controls; K-1/K-2 distinction.

**CORRECTION:** `/vendor` replacements were dormant on the documented loader path. K-1 embedded DEC and K-2 ARM tests are the valid real-surface experiments. P-A burst causality is `DISPROVEN`; the bursts themselves remain real historical observations.

**CONFIDENCE:** `VERIFIED`.

### 18.2 Producer/consumer and physical-sink conflation

**OLD CLAIM:** F8/FC or a Pa/Pb cluster is the physical SPDIF emitter; a `Pc=0x000B` capture proves DTS-configured output.

**WHY IT MATTERS:** It encouraged patching the wrong layer and treated a diagnostic/framer observation as proof of a physical DTS path.

**PRIMARY EVIDENCE:** F8/FC local fill/transform ending at `0x462FE`; `0x130D25/0x130E0D/0x13602` helper bodies; six emitter clusters; SND `+0x250C` AC-3/E-AC-3 branches; `MDrv_AUDIO_Dump_SpdifNpcm_Monitor` ring at `+0x672000`; raw N3 mixed capture.

**CORRECTION:** Clusters A–E are queue/control candidates; F is a header builder; SND +0x250C is AC-3/E-AC-3 preparation. A vendor-labelled software TX ring exists, but no physical transmitter relation or DTS-configured producer edge is proven.

**CONFIDENCE:** `VERIFIED` for the static classifications; `OPAQUE` for the physical relation.

### 18.3 Field/decoder semantic overclaim

**OLD CLAIM:** `*(r4+0x10)`, `desc+0x28`, `+0x2E6`, `movhi=0x1C01`, and `0xDFB1=allocator` can be used as a DTS-license/root-cause chain.

**WHY IT MATTERS:** These labels turn ordinary control/status fields into a presumed entitlement model and hide the actual unresolved boundaries.

**PRIMARY EVIDENCE:** `F_45FD1` branch; direct `F_0411A6` expression `r4=*(C+0xA108)`; symbolic/path-scoped `F_03E0DD` stores; SLEIGH displacement/immediate fields; `0x21FD6` raw word; format string argument order; `0xDFB1` cache operations.

**CORRECTION:** `r4` is a path-specific context/status pointer; `+0x2E4` is a diagnostic/state field; `+0x2E6` is not the printer’s lic input; `movhi` is `0xE0`; `0xDFB1` is cache maintenance. No field may be called a DTS license without a complete producer/consumer/cause chain.

**CONFIDENCE:** `VERIFIED` for the corrections; semantic interpretation remains bounded.

---

## 23. Independent current causal graph

```text
AUTH INPUTS
  gIpAuthVars / AUTH IPCheck
          |
          v
ARM MDrv_AUDIO_CheckHashkey
  AV+0x440 / AV+0x444
  AV+0x4D0 / AV+0x4D4 / AV+0x4D8
          |
          v
MDrv_AUDIO_ApplyHashkey
          |
          v
SET_IPAUTH_GROUP @0x424BD8
  software sequence + mapped-byte target
  0x112A82..85
          |
          |  [SE sampling / burst / delivery semantics: OPAQUE]
          v
[unresolved SE/DSP bridge]
          |
          +--> possible DSP key/status fields 0xB0000838 / 0xB000081C [producer OPAQUE]
          |
          +--> separate status/handshake word 0xB000001E [producer NOT FOUND]
          v
DEC MMIO gate
  eight *(u16*)0xB000001E == 5 comparisons
  fail -> separate 0xB000001D status path
          |
          v
DEC descriptor state
  r4 = *(C+0xA108)
  initialization to C+0x7C4 only on a separately proven base path
  *r4 / *(r4+0x10) select descriptor row
  desc+0x28 / +0x34 / +0x38
          |
          v
F_45DE0
  install F8/FC pointers
  fill local regions through 0x130D25
          |
          v
F_4618E
  N=desc+0x38
  fill/transform/sync writes in F8/FC
  desc+0xF0 = 0 or N
          |
          |  [no proven F8/FC -> queue/DMA/SND/TX handoff]
          v
[opaque output attachment]

Separate host/output branch:
  MI_AUDIO_Start Codec:9
    -> MapDecoderType 9 -> 0xB
    -> MApi/HAL SetDecodeSystem
       [param1 and +0x10 gate branch scope matters]
    -> SetSystem2
       command 0x04/0x97, latch 0xA
    -> monitor/SetMode/ApplySetting
    -> historical type[81]/type[4], spdif[] state
       [P-BYP/P-BYP6 details partly report-only]

Separate emitter/preparation branches:
  DEC clusters A-E: queue/control candidates
  DEC F: descriptor header builder
  SND +0x250C: AC-3/E-AC-3 preparation
  vendor TX dump ring: base +0x672000, span 0x66000
          |
          |  [relation to physical transmitter: OPAQUE]
          v
Software TX-visible capture
  AC-3: valid Pc=0x0001 bursts
  DTS: zero-byte result strongly supported, exact file provenance qualified
          |
          |  [no electrical/AVR lock proof]
          v
Physical SPDIF
  OPAQUE / RUNTIME REQUIRED
```

The graph intentionally does not contain an unverified arrow labeled “ARM license → physical DTS output.”

---

## 24. Remaining opaque boundaries

1. Live AUTH provisioning and the actual `gIpAuthVars` value.
2. Exact live `AV+0x440/+0x444/+0x4D0/+0x4D4/+0x4D8` values.
3. The exact first argument reaching `_MApi_AUDIO_SetDecodeSystem` and whether the `+0x10` reject branch is taken.
4. SE sampling semantics for `0x112A82–85`.
5. The producer/protocol for DSP-visible `0xB000001E`.
6. Whether any `*(r4+0x10)==1` producer exists on the live path.
7. Live `desc+0x08/+0x28/+0x30/+0x38` values during stock DTS.
8. Whether F8/FC contain compressed DTS, intermediate samples, or another representation.
9. The queue/DMA/SND handoff after F8/FC.
10. Which of clusters A–E, if any, is the actual software TX ring producer for a DTS-configured frame.
11. Whether the identified `+0x672000` ring is the physical transmitter source.
12. Exact configuration active during N3 `Pc=0x000B` frames.
13. Preserved primary evidence for the full P-BYP3/P-BYP6 claims.
14. Electrical SPDIF/receiver lock behavior.

---

## 25. Runtime-required questions

No runtime was performed in this phase. A later authorized phase would need, at minimum, read-only observations that answer:

1. What are the SE-side values and sampling behavior after SET_IPAUTH_GROUP?
2. What writes or hardware-produces `0xB000001E` in AC-3 versus DTS?
3. What are the live `r4` object fields and `desc` fields on the failing DTS path?
4. Does the F path execute, and what values are in F8/FC at its return?
5. Which producer advances the identified TX dump ring?
6. What is the exact host configuration during a clean DTS-configured attempt?
7. Does the ring reach the physical SPDIF peripheral, and does the receiver lock?

These are questions, not patch proposals.

---

## 26. Final audit conclusion

### What is genuinely proven

- The stock embedded loader selects image pairs conditionally; `/vendor` replacements were inert on the documented path.
- ARM AUTH mask construction and the SET_IPAUTH_GROUP software write order are real.
- `HAL_AUDIO_AbsWriteByte` is a conditional mapped-byte store, and the host-side final target bytes are statically identifiable.
- The MS12V22 DEC image contains eight `0xB000001E == 5` comparison sites, not four; no direct writer was found in the bounded scan.
- The DEC descriptor `r4` is a context/status pointer on the traced initialization paths, not a proven codec ID/license object.
- `desc+0x28`, `desc+0x34`, `desc+0x38`, and `desc+0xF0` have concrete control/data-flow roles, but not the license semantics previously assigned to them.
- `F_45DE0` installs and fills F8/FC pointers; it does not allocate them.
- `movhi` is `0xE0`, base is `0xE026B0`, and `0xDFB1` is cache maintenance.
- The printer loads `+0x2E8/+0x2E0/+0x2E4`; it does not load `+0x2E6`.
- The old invalid-license diagnostic read is `+0x2E4`, not `+0x2E6`.
- Historical DTS-framed `Pc=0x000B` bytes existed in some software-TX captures; `dts_01.bin` is not a DTS burst.
- A vendor-labelled software TX dump ring is statically identifiable, but its physical relation is not.

### What is probably true

- The N3/P-A-era DTS-framed bursts were produced by a mixed or AC-3-configured/primed framer path rather than a clean DTS-configured output path.
- The unresolved failure boundary is distributed across host state, SE/DSP delivery, DSP descriptor/config state, and output attachment.
- The existing IEC machinery is not missing merely because the DTS wire is empty.

### What remains unproven

- Any single DTS license root cause.
- ARM AUTH → `0xB000001E` delivery.
- Live meaning/producer of `*(r4+0x10)`.
- F8/FC semantic payload and downstream consumer.
- Exact emitter-to-physical-SPDIF relation.
- Exact configuration that produced historical `Pc=0x000B` frames.
- Universal absence of a writer or physical path.

### What previous agents got wrong or overclaimed

- They treated `/vendor` deployment as execution without loader proof.
- They promoted strings, diagnostics, and same-offset fields into license semantics.
- They mixed host and DEC `r4` values.
- They confused header/queue preparation with physical emission.
- They treated mixed wire bytes as DTS-configured output.
- They made bounded decoder scans sound universal and introduced concrete ISA/displacement errors.
- They used “allocation” for a fill helper and “wire” for a software TX dump.

### Where the real investigation boundary currently is

```text
The corpus proves the existence of substantial DTS/IEC machinery and several
real software-TX observations. It does not yet prove which live state selects
a DTS-configured producer, how that producer reaches the TX ring, or how the
ring reaches physical SPDIF. That boundary is unresolved; this phase does not
select or authorize a patch.
```

---

## Appendix A — Primary local references

- `patch_baseline/utpa2k_stock.ko`
- `patch_baseline/mik_stock.ko`
- `aeon_validate/dec_work/dec_full.bin`
- `aeon_validate/dec_work/dec32_clean.txt`
- `aeon_validate/r27_work/snd_full.bin`
- `aeon_validate/r27_work/snd_full.reko/snd_full_code_0001.asm`
- `spdif_audio_investigation/tools/dd_MDrv_AUDIO_ApplyHashkey.txt`
- `spdif_audio_investigation/tools/closure_utpa2k.txt`
- `spdif_audio_investigation/r8_out/HAL_AUDSP_DspLoadCode.asm`
- `spdif_audio_investigation/r8_out/a1_SetSystem2.txt`
- `spdif_audio_investigation/tools/k_utpa2k_decomp.c` (stock `_MApi_AUDIO_SetDecodeSystem`, address-shifted Ghidra dump)
- `spdif_audio_investigation/tools/k_Start.asm`
- `aeon_ghidra_public/aeon/data/languages/aeon.slaspec`
- `aeon_ghidra_public/aeon/data/languages/aeon_ORBIS32.sinc`
- `aeon_validate/dec_work/REPORT_phaseC_corpus_synthesis_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD_static_closure_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD1_residual_static_20260922.md`
- `aeon_validate/dec_work/REPORT_phaseD2_final_static_residual_20260922.md`
- `aeon_validate/dec_work/forensic_trace_autonomous_20260918_v2.md`
- `aeon_validate/dec_work/forensic_trace_autonomous_20260918_v3.md`
- `aeon_validate/dec_work/forensic_trace_autonomous_20260918_v4.md`
- `aeon_validate/dec_work/MASTER_FORENSIC_REPORT_20260919.md`
- `aeon_validate/r57_npcm/FINDING_wire_baseline_20260913.md`
- `aeon_validate/r57_npcm/FINDING_wire_capture.md`
- `aeon_validate/r57_npcm/FINDING_dec_dts_output_SUPERSEDES_s24.md`
- `REPORT_current_state_K1_verdict_20260915.md`
- `REPORT_GLM53_full_audit_and_dts_repair.md`
- `aeon_validate/MISSION_LOG_dts_persuit_20260916.md`
- `aeon_validate/r57_npcm/dts_01.bin`
- `aeon_validate/r57_npcm/PA_probe1_dts.bin`
- `aeon_validate/r57_npcm/N3_DTS_dts.bin`
- `aeon_validate/r57_npcm/N3_AC3_ctl.bin`
- `aeon_validate/r57_npcm/K1_DTS_wire.bin`
- `aeon_validate/r57_npcm/K1CTL_AC3_wire.bin`

## Appendix B — Integrity statement

- This is a new report. No prior report, raw log, JSONL, binary, Ghidra project, backup, or firmware image remains modified, renamed, moved, deleted, or overwritten.
- A background agent temporarily edited five historical reports during an attempted read-only audit; those files were restored byte-for-byte from pre-task snapshots, and their final SHA-256 values match the pre-task state.
- No new ADB/device/runtime operation was performed.
- No patch, flash, module reload, reboot, settings change, or physical-line test was performed.
- TCL/T615 was not used as evidence, donor, or comparison target.
