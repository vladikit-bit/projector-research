# REPORT_runtime_phase7A — The Missing DTS-Specific Encoder Configuration

**Scope:** R7-A — Determine exactly what AC3-specific SHM/encoder configuration means, where its values are consumed, and whether that configuration is what identifies the compressed stream as AC3/DTS at the SPDIF/IEC61937 encoder.

**Device:** Thundeal TD98 Pro / C50A — MStar MT5889 (ARM) Android TV.
**Deployed binaries (MD5 re-verified, R7-A):**
- `utpa2k.ko` Exp-B = `kmods/utpa2k_expB_0x4d8_3_dts_license.ko` → `4c5e6fbb10e24abc3b8d2b0490507983`
- `mik.ko` R5 patch = `runtime_phase5/mik_r5_patched.ko` → `03fc2c0d22b231bc478e574557b85473`

**R7-A constraints (honored):**
1. No binaries patched, no EDID/settings/routing/properties changed.
2. `utpa2k.ko` and `mik.ko` left byte-identical to deployed images.
3. No new kernel instrumentation module. Read-only static + (already-captured) runtime evidence only.
4. **R7-B patch is explicitly NOT made in this task.** R7-B (mirror the codec-9 path → IEC61937 burst-info `0x0B`) is deferred pending this analysis.

**Methodology note (carried from R6 — critical for confidence):** In an *unlinked* `.ko`, every `bl`/`b` immediate is a **relocation placeholder** (`bl #<self>`); prior R-turns mis-decoded all call targets. This report uses the relocation-aware disassembler (`tools/r6_dis_reloc.py` + `tools/r6_reloc.py`) that resolves real callees via `.rel.*` (`VA = r_offset` because `sh_addr == 0`). SHM_PARAM dispatch is decoded by relocating the jump tables and resolving each handler.

---

## 1. The four AC3-associated SHM_PARAM writes — exact values & call sites

The AC3 machinery in the user brief references four param IDs: `0x6e`, `0x9d`, `0x9a`, `0x4c`. Two of these are written by `HAL_AUDIO_SPDIF_ApplySetting` (the per-port SPDIF apply function), two by `HAL_AUDIO_DigitalTx_ApplySetting` (the generic digital-transmit apply function). Confirmed call sites and argument values:

### 1a. `HAL_AUDIO_SPDIF_ApplySetting` (file `r6_out/HAL_AUDIO_SPDIF_ApplySetting_reloc.asm`)

Prologue (lines 17–22):
```
ldr  r1, [r0, #0x4f8]      ; field 0x4f8 (misc)
ldrb r1, [r0, #0x3a5]      ; field 0x3a5
ldr  fp, [r0, #0x4f0]      ; field 0x4f0
ldr  sl, [r0, #0x4f4]      ; ★ codec field (codec_id: 5=AC3, 9=DTS)
ldrb r5, [r0, #0x3a8]      ; ★ global field 0x3a8 (codec-independent)
mov  r0, r4                ; r4 = r0 = port/instance handle
```
`is-AC3` computation (lines 84–88):
```
0x004468d8: ldr  r1, [r1, #0x4f4]        ; r1 = codec field (reload)
0x004468e0: sub  r1, r1, #5   [CODEC_IMM 5]
0x004468e4: clz  r1, r1
0x004468e8: lsr  r8, r1, #5             ; r8 = (codec==5) ? 1 : 0  → IS-AC3 FLAG
```
Encoder channel-lock is then called **unconditionally** (both r3==4 and r3≠4 paths merge to `bl HAL_AUDIO_Encoder_Channel_Lock` at 0x4468ec).

Write #1 — **param `0x6e`** (lines 90–94):
```
0x004468f0: mov  r0, #0x6e
0x004468f4: mov  r1, r4   ; port/instance
0x004468f8: mov  r2, r8   ; ★ value = is-AC3 (1 for AC3, 0 for DTS)
0x004468fc: mov  r3, #0
0x00446900: bl   HAL_SND_R2_Set_SHM_PARAM
```
Write #2 — **param `0x9d`** (lines 95–99):
```
0x00446904: mov  r0, #0x9d
0x00446908: mov  r1, r4   ; port/instance
0x0044690c: mov  r2, r5   ; ★ value = (global field 0x3a8) & 1  (codec-independent)
0x00446910: mov  r3, #0
0x00446914: bl   HAL_SND_R2_Set_SHM_PARAM
```
(Note line 63: `and r5, r5, #1` — the 0x9d value is `field_0x3a8 & 1`, a global bit, identical for AC3 and DTS.)

### 1b. `HAL_AUDIO_DigitalTx_ApplySetting` (file `r6_out/HAL_AUDIO_DigitalTx_ApplySetting_reloc.asm`)

Write #3 — **param `0x4c`** (lines 268–275):
```
0x00446308: mov  r0, #0x4c
0x0044630c: mov  r2, r4   ; r4 = generic compressed/non-PCM status flag
                         ;   (derived from ldrh [r0,#0x25c8] compare, lines 262–266)
0x00446310: bl   HAL_DEC_R2_Set_SHM_PARAM
0x00446314: mov  r0, #0x4c
0x00446318: mov  r1, #1
0x0044631c: mov  r2, r4
0x00446324: bl   HAL_DEC_R2_Set_SHM_PARAM
```
→ **generic** (value depends on field 0x25c8, not on codec 5 vs 9).

Write #4 — **param `0x9a`** (lines 277–314):
```
0x0044632c: movw r1, #0x2501
0x00446330: ldrb r0, [r0, r1]    ; field 0x2501 = generic "compressed / NonPCM mode" flag
0x00446334: cmp  r0, #1
0x00446338: bne  #0x44638c        ; if compressed mode OFF → skip
0x0044633c: sub  r0, r8, #3   [CODEC_IMM 3]   ; ★ codec_id - 3 (per-codec enable-bit index)
0x00446340: mov  r1, #0x10
0x00446344: mov  r2, #0x10
0x00446348: bl   HAL_AUDIO_AbsWriteMaskByte   ; enable bit idx = codec_id-3
0x0044634c: mov  r0, #0x9a
0x00446350: mov  r1, #0
0x00446354: mov  r2, #1          ; ★ generic value 1 (compressed path)
0x0044635c: bl   HAL_DEC_R2_Set_SHM_PARAM
   ... (else branch 0x44638c sets value 0, same structure) ...
```
→ **generic** (gated by field 0x2501, the codec_id−3 index is only used as a *per-codec enable-bit index* shared by all compressed codecs).

**Conclusion of §1:** Of the four referenced params, **only `0x6e` is codec-specific**. `0x9d` is codec-independent (global bit); `0x4c` and `0x9a` are generic compressed-mode writes shared by every compressed codec (AC3 *and* DTS).

---

## 2. Consumers — param ID → SHM region → offset → handler

There are two independent R2 DSP shared-memory regions:
- **DEC SHM:** `g_virDecR2shm` = DSP MAD base + `0xe0001000`, written by `HAL_DEC_R2_Set_SHM_PARAM`.
- **SND SHM:** `g_virSndR2shm` = DSP MAD base + `0x70a0000`, written by `HAL_SND_R2_Set_SHM_PARAM`.

Both writers are jump-table dispatchers indexed by param ID; each param maps to a distinct SHM offset. The DSP firmware is the ultimate (opaque) consumer — it reads these fields and performs the burst packing.

### 2a. `HAL_SND_R2_Set_SHM_PARAM` (file `r6_out/HAL_SND_R2_Set_SHM_PARAM.asm`)
Prologue: `r6 = g_virSndR2shm` (0x459e04–0x459e14); jump table at `0x459e70`, indexed by `(param − 0x59)`; range `cmp r0, #0x76; bhi`.

| param | index | handler   | store instruction                  | SND SHM offset |
|-------|-------|-----------|------------------------------------|----------------|
| `0x6e`| 21    | 0x45a1a0  | `str r7, [r6, #0xf8]`             | **0xf8**       |
| `0x9d`| 68    | 0x45a2b4  | `clz r0, r7; lsr r0, r0, #5; str r0, [r6, #0x8c]` | **0x8c** |

- `0x6e`: `r7 = value arg = is-AC3`. **SND SHM offset 0xf8 = is-AC3 flag** (1 for AC3, **0 for DTS**).
- `0x9d`: `r7 = value arg = (field_0x3a8 & 1)`. Final stored value = `clz(value) >> 5`. Since `value ∈ {0,1}`, `clz(0)>>5 = 0` and `clz(1)>>5 = 0` — **always 0, codec-independent**. (This is a degenerate/no-op-style field; it never distinguishes AC3 from DTS.)

### 2b. `HAL_DEC_R2_Set_SHM_PARAM` (file `r6_out/HAL_DEC_R2_Set_SHM_PARAM.asm`)
Prologue: `r0 = g_virDecR2shm` (0xe0001000); jump table at `0x45a4c0`, indexed by param; range `cmp r5, #0xd0; bhi`. Per-instance region: base + `fp*14*64` (`rsb r0, fp, fp, lsl #4; add r0, r6, r0, lsl #6`).

| param | handler   | store instruction                  | DEC SHM offset |
|-------|-----------|------------------------------------|----------------|
| `0x4c`| 0x45ae60  | `str r1, [r0, #0xe60]`            | **0xe60**      |
| `0x9a`| 0x45b1b8  | `movw r1, #0x1158; … str sl,[r4,r0]` | **0x1158**  |

- `0x4c` → DEC SHM 0xe60 (generic compressed-status).
- `0x9a` → DEC SHM 0x1158 (generic compressed-mode enable).
- Note: `0x6e`/`0x9d` are **SND-only** params — they are absent from the DEC table and fall through to the no-op handler in the DEC dispatcher.

**Conclusion of §2:** The single encoder-facing (SND-region) field that differs between AC3 and DTS is **SND SHM offset 0xf8** (param `0x6e`), carrying the is-AC3 flag. Every other AC3 write is shared with DTS or degenerate.

---

## 3. Semantic meaning of each relevant parameter

| param | region | offset | value written by AC3 | value written by DTS | semantics |
|-------|--------|--------|----------------------|----------------------|-----------|
| `0x6e`| SND    | 0xf8   | **1** (is-AC3)       | **0** (is-AC3)       | Stream class flag: "this is AC3" vs "this is not AC3". **The only codec-type discriminator the LKM writes.** |
| `0x9d`| SND    | 0x8c   | 0 (degenerate)       | 0 (degenerate)       | `clz(global_0x3a8 & 1)>>5` → always 0. Codec-independent, carries no type info. |
| `0x9a`| DEC    | 0x1158 | 1 (compressed mode)  | 1 (compressed mode)  | Generic "compressed/NonPCM output enabled" — shared by all compressed codecs. |
| `0x4c`| DEC    | 0xe60  | generic status flag  | generic status flag  | Generic compressed-stream status — shared by all compressed codecs. |
| (per-codec enable bit) | DEC | codec_id−3 index (DTS→bit6, AC3→bit2) | bit set | bit set | `HAL_AUDIO_AbsWriteMaskByte(idx=codec_id−3, 0x10)` — enables the decoder/DSP path for the codec; set for **both** AC3 and DTS. |

---

## 4. Does any param control the IEC61937 burst/data type?

**Short answer: not explicitly in the LKM.** The IEC61937 burst-info value (AC3 = `0x15`, DTS = `0x0B`) is the data-type tag the encoder prepends to each compressed frame. We scanned both deployed modules for the IEC61937 sync-word constant (`0xF872` / `0x4E1F`) and for burst/pack/encoder function symbols:

- `r6_out/scan_iec_packer.txt` → **NO IEC61937 sync constant found in data/rodata of either `utpa2k_expB_*.ko` or `mik_r5_patched.ko`.**
- The only SPDIF/NonPCM/encoder symbols present are the known *wrappers*: `HAL_AUDIO_SPDIF_Tx_SetNonPCM`, `HAL_AUDIO_Encoder_Channel_Lock`, `MDrv_AUDIO_ENCODER_*`, `MApi_AUDIO_SPDIF_ChannelStatus_CTRL`. **No DTS-specific IEC61937 burst-info packer symbol exists** in the LKM; the DTS SDO packer strings (`DTSX_CORE2_API_SDO_Packer`, etc.) have no resolvable xref (opaque, as established in R6).

Therefore the burst-info generation is **delegated to the DSP firmware**, which consumes the SHM fields. The LKM's only type-level contribution is the **is-AC3 flag (offset 0xf8)** plus the **codec_id** (already passed to the DSP) and the **per-codec enable bit**. There is no LKM-level literal "DTS burst-info = 0x0B" to set.

---

## 5. Codec-ID transformation (codec_id−3, 16-bit masks, tables)

The only codec-ID arithmetic observed in the apply path is the **`codec_id − 3`** used as a bit index into a 16/32-bit enable mask:

- `HAL_AUDIO_DigitalTx_ApplySetting` line 281 / 301: `sub r0, r8, #3` → `HAL_AUDIO_AbsWriteMaskByte(idx = codec_id−3, mask = 0x10)`.
  - AC3 (5): idx 2 → bit 2.
  - DTS (9): idx 6 → bit 6.
- This is a **per-codec enable bit**, set for *both* codecs. It selects *which* decoder/DSP instance is active; it is **not** a stream-type tag.
- No `codec_id − 3` → table lookup, no 16-bit mask other than the enable bit, no separate "codec 9 → DTS" remap table exists in the apply path. The CODEC_IMM scan (R6 + R7-A) found immediate codec ids `3,5,10,11,12,13` but **never `9`** in any configuration-writing path — confirming DTS (9) is handled only by the generic `>=3` / `>=5` gates, never by a codec-9-specific branch.

---

## 6. DTS SDO packer dispatch (sub-task 5)

R6 established that the DTS SDO packer selection is **statically opaque**: the packer symbols (`DTSX_CORE2_API_SDO_Packer`, `HAL_AUDIO_DTSELoadCode` log wrapper, `ut_dtseload`) have no resolvable xref from the apply/SetMode/SetNonPCM paths. R7-A adds:

- `HAL_AUDIO_SPDIF_Tx_SetNonPCM` is reached for DTS (R6 runtime: codec 9 exercises the `SetNonPCM` branch under `codec_id >= 3`). It forwards to the generic non-PCM enable, **not** to a codec-9 packer.
- There is **no function-pointer / op-table / codec-indexed callback** in the LKM that routes codec 9 to a DTS IEC61937 burst packer. The routing decision is made inside the DSP firmware, reachable only via the SHM fields (§2) — and the only type field written for DTS is the *absence* of is-AC3 (offset 0xf8 = 0).

So at the LKM boundary, DTS is carried as "**generic compressed, is-AC3 = 0**" with its enable bit set, and **no positive DTS-type assertion** is ever written.

---

## 7. AC3 vs DTS comparison at the encoder-facing (SND SHM) boundary

| Field | AC3 (codec 5) | DTS (codec 9) | Equal? |
|-------|---------------|---------------|--------|
| SND SHM 0xf8 (is-AC3, param 0x6e) | **1** | **0** | **NO — first concrete divergence** |
| SND SHM 0x8c (param 0x9d) | 0 | 0 | yes (degenerate) |
| DEC SHM 0x1158 (param 0x9a) | 1 | 1 | yes (generic) |
| DEC SHM 0xe60 (param 0x4c) | status | status | yes (generic) |
| per-codec enable bit (idx 5−3 / 9−3) | bit 2 set | bit 6 set | both set (different index) |
| **is-DTS flag (any offset)** | — | **never written** | **DTS has no counterpart** |

**First concrete AC3-vs-DTS difference at the encoder-facing boundary:** SND SHM offset **0xf8** = `1` for AC3 vs `0` for DTS, combined with the **absence of any "is-DTS" field** anywhere in either SHM region.

---

## 8. Exact opaque boundary

The opaque boundary is the **DSP firmware**, the consumer of both R2 SHM regions:
- DEC SHM: `g_virDecR2shm` = DSP MAD base + `0xe0001000`.
- SND SHM: `g_virSndR2shm` = DSP MAD base + `0x70a0000`.

The LKM only **writes** these regions; the firmware **reads** them and performs the actual IEC61937 burst packing + burst-info tagging. Static analysis cannot enter the firmware, so the final mapping `(is-AC3 flag, codec_id, enable bit) → burst-info 0x0B/0x15` is **not statically verifiable**. This is the precise opaque boundary that remains after R7-A.

---

## 9. Confidence classification

| Finding | Classification |
|---------|----------------|
| `0x6e` is the only codec-specific SHM write; DTS receives `0` there | **PROVEN** — direct relocation-aware disassembly of both writers + call sites |
| `0x9d` is codec-independent/degenerate (always 0) | **PROVEN** — `clz(value)>>5` with `value ∈ {0,1}` |
| `0x9a` / `0x4c` are generic compressed-mode writes shared by AC3 and DTS | **PROVEN** — gated by field 0x2501, codec_id only used as enable-bit index |
| No codec-9-specific SHM param/field/branch is written anywhere in either module | **PROVEN** — exhaustive CODEC_IMM + param-ID scan (R6 + R7-A) |
| No IEC61937 sync constant / DTS burst-info literal in the LKM | **PROVEN** — `scan_iec_packer.txt` |
| The is-AC3 flag (0xf8) is the mechanism by which the encoder distinguishes AC3 from DTS | **STRONGLY SUPPORTED** — by elimination: it is the *only* type discriminator the LKM writes; DTS is left as "not-AC3" with no positive assertion |
| The missing is-DTS assertion is the root cause of "no physical DTS lock" | **STRONGLY SUPPORTED** — consistent with R6 symptom (DTS reaches kernel as codec 9, exercises SetNonPCM, but encoder never emits DTS burst-info 0x0B) |
| The flag → burst-info `0x0B` mapping occurs in DSP firmware and is the exact failing register | **NOT ESTABLISHED** — DSP firmware is opaque; cannot statically confirm the internal mapping |

---

## 10. Answer to the R7-A final question

**Q: What exact configuration does AC3 receive that DTS does not, and is that configuration actually responsible for identifying the compressed stream as AC3/DTS at the SPDIF/IEC61937 encoder?**

**A:**
1. **What AC3 receives that DTS does not:** Exactly **one** codec-specific item — the **is-AC3 flag** (`HAL_SND_R2_Set_SHM_PARAM(0x6e)` → SND SHM offset **0xf8** = `1` for AC3, `0` for DTS). All other AC3 SHM writes (`0x9d`, `0x9a`, `0x4c`) are codec-independent or generic compressed-mode and are written identically for DTS. DTS receives **no codec-9 counterpart flag** at any SHM offset.

2. **Is that configuration responsible for identifying the stream as AC3/DTS?** At the LKM level, **yes by construction**: the is-AC3 flag is the *only* stream-class assertion handed to the encoder path. The encoder (DSP firmware) has no other positive signal to distinguish DTS from the generic compressed class — it sees "is-AC3 = 0" and must fall back to a default. Because DTS is never positively tagged, the firmware most likely never selects the DTS IEC61937 burst packer (burst-info `0x0B`), which matches the observed "no physical DTS lock" symptom. The actual burst-info generation, however, happens inside the **opaque DSP firmware**, so the precise internal mapping cannot be statically proven (classification: STRONGLY SUPPORTED, not PROVEN).

**Net:** The configuration gap is a **missing positive DTS-type assertion** at the encoder-facing SHM boundary (offset 0xf8 is binary is-AC3 / is-not-AC3; there is no is-DTS). This is the most plausible root cause of the DTS passthrough failure and is the exact target for the deferred **R7-B patch** (mirror the AC3 path for codec 9 so a DTS-type flag reaches the encoder and the firmware selects burst-info `0x0B`).

---

## 11. Next steps (deferred — NOT executed)

- **R7-B (explicitly deferred):** Mirror the codec-9 path so the encoder receives a positive DTS-type signal at the SND SHM boundary (e.g., set offset 0xf8 to a DTS-specific value, or add a codec-9 equivalent `HAL_SND_R2_Set_SHM_PARAM` write), enabling the DSP to select the DTS IEC61937 burst packer (burst-info `0x0B`). Implementation and verification are out of scope for R7-A per the user's constraint #8.
- If a read-only runtime capture is later desired (constraint #7 allows it), the only safe exposures are the already-established fields: SND SHM 0xf8 (is-AC3), DEC SHM per-codec enable bits. The debug CLI is known silent/unusable (R5 §9e), so a static conclusion is preferred.
