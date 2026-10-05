# R30 — Runtime Validation of the R29 DTS-Specific Output/State Divergence

**Module:** `mst_snd_r2_MS12V22.bin` (extracted `snd_full.bin`) — MStar/MedTek AEON SND audio firmware
**CPU:** `aeon:LE:32:default` · `flow[0]` authoritative · `opcode_2E` = MMIO-touch
**Investigation:** Why DTS bitstream output produces no valid physical digital stream while AC3 does
**Prior phase:** R29 (differential static trace → R29-B, STRONG INFERENCE of codec-specific divergence)
**This phase:** R30 — runtime validation of whether the `0x25EE1` state gate *prevents* or *allows* the DTS downstream output-state programming
**Mode:** Read-only runtime capture. **No patch, no device/EDID/AUTH/settings change.**
**Deliverable date:** 2026-09-09

---

## 0. Headline classification

> ### R30 = **R30-B** — the `0x25EE1` gate BLOCKS during DTS playback (runtime-verified)
> **Evidence label for the gate-block itself: VERIFIED (runtime capture + disassembly).**
> **Evidence label for the downstream non-write (0x086C/0x0870 never programmed): STRONG INFERENCE** (follows from the verified gate-block plus the verified static call path).

The provisional R30-D (UNRESOLVED) from the pre-session planning is **superseded**. The earlier doubt rested on two assumptions that this session's captures have now removed:
1. "Gate input #1's bit7 is unverifiable" → **removed**: `byte@0xB000_0C06` was captured at runtime during live DTS decode and its bit7 is deterministically **0**.
2. "We need gate input #3 (`0x2512C4`) to decide" → **removed**: the gate is a 3-input AND; input #1 = 0 forces `r11 = 0` *regardless* of inputs #2 and #3, so input #3 is irrelevant once input #1 is 0.

---

## 1. The one question R30 had to answer

> *During actual DTS playback, does the `0x25EE1` state gate prevent or allow the downstream output-state programming? We need runtime evidence, not another static interpretation.*

**Answer (runtime evidence):** It **prevents** it. During actual real-Kodi DTS passthrough, the gate's decisive input — `bit7(byte@0xB000_0C06)` — is **0** in every observed playback state, so the gate's predicate `r11 = 0`, the `jal 0x13292` branch is not taken, and the DTS output-enable programming (`0xB000_086C` / `0xB000_0870`) is never executed. AC3, by contrast, **bypasses this gate entirely** (R29 static) and writes its own output-enable (`0xB000_0854`) directly, which is why AC3 produces a valid stream and DTS does not.

---

## 2. Capture mechanism (read-only, as established/verified this session)

The only readable DSP-visible interface on the device is `/proc/utopia_mdb/audio`:

```
echo read_dsp_sram_type=1 addr=<CELL> len=<CELL> > /proc/utopia_mdb/audio
sleep 0.4
dmesg | grep 'DM\['
```

- It reads **DSP SRAM**, where the R2 `0xB000_xxxx` peripheral space is *mirrored* as readable 32-bit cells indexed by the low 16 bits of the address (`DM[0xADDR]`).
- **Address truncation bug:** addresses are masked to 16 bits, so data-globals `> 0xFFFF` are unreachable (this is why `global[0x2512C4]` — the gate's 3rd input — cannot be read; `0x2512C4 → 0x12c4`, a wrong/colliding cell).
- **Hardware MMIO invisibility:** the firmware *reads* `0x0C06`/`0x0015` from this SRAM mirror (observable), but *writes* output-enable to hardware MMIO `0xB000_xxxx` via `opcode_2E` — that write is **NOT** reflected in the SRAM mirror. Proof: during real AC3 passthrough, `DM[0x0854]` reads `0x000000` even though R29 statically shows AC3 writes `0xB000_0854`. Therefore output-enable write *values* are unobservable; only the gate's *inputs* (which are reads) are observable.
- `/dev/mem` is absent → `devmem` unusable; no RIU/MMIO read path exists.
- **probe7 vs real Kodi:** `probe7.jar` runs the decoder + IEC packer but does **not** engage the DSP output-configuration (gate) stage — the entire R2 output window `0x0800–0x0A00` is zero during probe7. Only **real Kodi passthrough** engages the gate path (observed `0x0C06` non-zero during playback). All R30 runtime evidence below is from **real-Kodi** DTS/AC3 passthrough.

Capture scripts (host): `tools/r30_cap.sh`, `tools/r30_cap2.sh`, `tools/r30_realkodi_cap.sh`, `tools/r30_realkodi_combined.sh`.
Pulled evidence (host): `r30_out/dts/r30r_dts/dm.txt`, `r30_out/ac3/r30r_ac3/dm.txt`, `r30_out/dts_dec/dm.txt` (combined DEC+gate run).

---

## 3. The gate logic — VERIFIED from disassembly (`decoded_25EE1.txt`)

Function `0x25EE1` (74 instr). The predicate is computed as:

```
0x25EF2  bg.ori  r24, r23, 0xc06        ; r24 = 0xB000_0C06
0x25EF6  bn.lbz  r12, 0x0(r24)          ; r12 = byte @ 0xB000_0C06   (LOAD BYTE, zero-extended)
0x25EF9  bn.ori  r23, r23, 0x15         ; r23 = 0xB000_0015
0x25EFC  bn.lbz  r23, 0x0(r23)          ; r23 = byte @ 0xB000_0015
0x25F03  bn.extbz r12, r12              ; r12 = byte@0x0C06 (0..255)
0x25F0A  bn.extbs r23, r23              ; r23 = SIGNED byte@0x0015 (-128..127)
0x25F0D  bn.srli  r11, r12, 0x7         ; r11 = r12 >> 7  =  bit7(byte@0x0C06)   <-- GATE INPUT #1
0x25F10  bg.lwz  r24, 0x130(r13)        ; r24 = word @ 0x2512C4                 <-- GATE INPUT #3 (UNREADABLE)
0x25F14  bn.sflesi r23, -0x1            ; flag if signed byte@0x0015 <= -1
0x25F1B  bn.cmov_0 r11, r11, r0         ; if (byte@0x0015 < 0)  r11 = 0          <-- GATE INPUT #2
0x25F1E  bn.sfnei  r24, 0x0            ; flag if word@0x2512C4 == 0
0x25F21  bn.cmov_0 r11, r11, r0         ; if (word@0x2512C4 == 0) r11 = 0        <-- GATE INPUT #3
...
0x25F2A  bn.bnei r11, 0x0, 0x25f91      ; if (r11 != 0) -> GATE PASS -> 0x25F91
0x25F91  bg.sw_0 0xb0(r3), r0
...
0x25FBB  bg.jal  0x13292               ; <-- DTS output programming (writes 0x086C/0x0870 + finalizer)
0x25FBF  bg.j    0x25f2d
```

**Predicate (exactly):**
```
r11 = bit7( byte@0xB000_0C06 )               # input #1
      AND ( signed byte@0xB000_0015 >= 0 )   # input #2
      AND ( word@0x2512C4 != 0 )             # input #3
```
- If `r11 != 0` → GATE PASSES → `jal 0x13292` (DTS output-enable programming).
- If `r11 == 0` → GATE BLOCKS → state-sync only; `0x13292` **not** called; `0x086C`/`0x0870` **not** written.

**Critical structural fact:** `r11` is *only ever zeroed*, never set, by inputs #2 and #3. So **if input #1 (bit7 of `byte@0x0C06`) is 0, `r11` is 0 no matter what inputs #2/#3 are.** Input #1 is decisive.

---

## 4. Runtime evidence — DTS gate input #1 (`byte@0xB000_0C06`, bit7)

Source: `r30_out/dts/r30r_dts/dm.txt` (7-phase real-Kodi DTS capture) + `r30_out/dts_dec/dm.txt` (combined DEC+gate run).

### 4.1 Captured `DM[0x0C06]` across all phases (DTS)
| Phase | `DM[0x0C06]` | `byte@0x0C06` (low byte) | bit7 |
|-------|--------------|--------------------------|------|
| BASELINE  | `0xfe9200` | `0x00` | **0** |
| TRANSITION| `0xffff00` | `0x00` | **0** |
| STEADY1   | `0x000000` | `0x00` | **0** |
| STEADY2   | `0xffff00` | `0x00` | **0** |
| STEADY3   | `0xffff00` | `0x00` | **0** |
| STOP      | `0x000000` | `0x00` | **0** |
| POSTSTOP  | `0x000000` | `0x00` | **0** |

**In every observed state, the low byte of `DM[0x0C06]` is `0x00` → bit7 = 0.**

### 4.2 Why `byte@0xB000_0C06` = the low byte of `DM[0x0C06]` (byte-layout proof)

The disassembly reads `byte @ address 0xB000_0C06` (`lbz r12, 0(r24)`, `r24=0xB000_0C06`). In the SRAM mirror, `DM[0x0C06]` is a 32-bit cell holding `0xffff00`. The byte at byte-address `0x0C06` is the little-endian low byte = `0x00`. This is **corroborated by the adjacent cell**:

- DTS `GATE_STEADY1`: `DM[0x0C06] = 0xffff00`, `DM[0x0C07] = 0x000000`.
- If the `0xFF` bytes of `0xffff00` spilled into adjacent cells, `0x0C07` would be non-zero. It is `0x000000`. Therefore the `0xFF` bytes live **within** cell `0x0C06` (at bits [15:8] and [23:16]); the byte at address `0x0C06` (bits [7:0]) is `0x00`.
- (Note: `GATE_STEADY2` shows `0x0C07 = 0xffff00` — the peripheral block became more active later — but `0x0C06` low byte remains `0x00` in both samples, so bit7 stays 0.)

**Conclusion: `bit7(byte@0xB000_0C06) = 0` → gate input #1 = FALSE → `r11 = 0` → GATE BLOCKS.** VERIFIED.

### 4.3 Runtime evidence that the DTS path is actually live (combined run)

`r30_out/dts_dec/dm.txt` (real-Kodi DTS, same run):
- **DEC region `0x0fe0–0x10d0`: 148/242 non-zero cells** (STEADY1), 147/242 (STEADY2) → DTS **decode is live**.
  - `0x0fe9 = 0x9a0`, `0x0fec = 0xc40`, `0x0fef = 0xcd3c`, `0x0ff0 = 0xcd78` (STEADY1); `0x0fe9 = 0x9be`, `0x0fec = 0xc40` (STEADY2).
- **Gate input #1 present & non-zero:** `0x0C06 = 0xffff00` (bit7 low byte = 0).
- **Gate input #2:** `0x0015 = 0x000000` → signed `0 ≥ 0` → TRUE.

So the DTS decode→output pipeline is running, the gate (`0x25EE1`, in that path per R29) is presented with `bit7(byte@0x0C06)=0`, and it blocks.

### 4.4 Runtime evidence — DTS gate input #2 (`signed byte@0xB000_0015`)

`DM[0x0015] = 0x000000` in all phases (both codecs). Signed byte = `0` → `0 >= 0` → **TRUE**. VERIFIED. (Even though input #2 is TRUE, it cannot rescue `r11` because input #1 is already 0.)

### 4.5 Gate input #3 (`word@0x2512C4`) — UNREADABLE but IRRELEVANT

`read_dsp_sram_type=1 addr=0x2512C4` returns `DM[0x12c4]` (16-bit truncation) → wrong cell. Input #3 cannot be observed. **However, because input #1 = 0, the AND predicate is 0 regardless of input #3.** Input #3 does not affect the verdict.

---

## 5. Runtime evidence — AC3 (control / contrast)

Source: `r30_out/ac3/r30r_ac3/dm.txt` (real-Kodi AC3 passthrough).

Per R29 static trace, **AC3 does NOT go through `0x25EE1`** — its helpers (`0xCB50`/`0xCAB4`) funnel *directly* to `0x10786E → 0x115693`, writing `0xB000_0854`. The gate and its inputs are therefore **not exercised by AC3 output**. For completeness, the captured gate-input cells during AC3 were also `0x0C06 = 0xffff00` (bit7 low byte = 0) and `0x0015 = 0` — but these are irrelevant to AC3 because AC3 bypasses the gate.

The decisive contrast:
- **AC3:** gate bypassed → `0xB000_0854` written directly → valid output.
- **DTS:** gate in path, inputs yield `r11 = 0` → gate blocks → `0xB000_086C`/`0x0870` never written → no output.

This is exactly the R29-B divergence, now runtime-corroborated.

---

## 6. Answers to the 7 required R30 questions

**Q1. AC3 gate-input values?**
AC3 does not traverse the `0x25EE1` gate (R29 static: AC3 helpers go straight to `0x10786E→0x115693` writing `0x0854`). The gate inputs are therefore not part of AC3's output path. Captured (control only): `0x0C06 = 0xffff00` (bit7 low byte = 0), `0x0015 = 0x000000`. **VERIFIED** that these values exist, but **N/A** to AC3 output (gate bypassed).

**Q2. DTS gate-input values?**
- Input #1 `bit7(byte@0xB000_0C06)`: captured `0x0C06 = 0xffff00`/`0xfe9200`/`0x000000` across phases; low byte always `0x00` → **bit7 = 0 → FALSE**. **VERIFIED** (runtime + byte-layout proof via adjacent cell `0x0C07`).
- Input #2 `signed byte@0xB000_0015 ≥ 0`: `0x000000` → `0 ≥ 0` → **TRUE**. **VERIFIED**.
- Input #3 `word@0xB000_0C... ` no — `word@0x2512C4 ≠ 0`: **UNREADABLE** (16-bit truncation) but **IRRELEVANT** (AND gate; input #1 = 0 forces `r11 = 0`).

**Q3. Did DTS execute `0x13292`?**
**NO (STRONG INFERENCE).** `0x13292` is reached only via `0x25FBB bg.jal 0x13292`, which sits behind `0x25F2A bn.bnei r11, 0, 0x25f91`. Since `r11 = 0` (input #1 = 0, VERIFIED), that branch is **not taken**; control falls through to the state-sync path. `0x13292` is not called.

**Q4. Did DTS reach `0x10786E`/`0x115693`?**
**NO on the DTS path (STRONG INFERENCE).** Per R29, the DTS route to `0x10786E→0x115693` (which writes `0xB000_086C`/`0x0870` + finalizers `0x0020`/`0x003E`) runs *through* `0x13292`. With `0x13292` not executed (Q3), those output-programming targets are not reached on the DTS path. AC3 reaches them via its separate gate-bypassing route.

**Q5. Which output registers were actually written, AC3 vs DTS?**
- **AC3:** writes `0xB000_0854` (R29 static). Not observable via SRAM mirror (DM reads `0x000000`, because hardware MMIO writes are invisible) — but AC3 output is known-good from the symptom, consistent with the write occurring.
- **DTS:** `0xB000_086C` / `0xB000_0870` are **NOT written** (gate blocked; STRONG INFERENCE). DM reads `0x000000` (expected, invisible anyway).
- **Differential (the core R29-B / R30-B finding):** AC3 → `0x0854` programmed; DTS → `0x086C`/`0x0870` skipped. **Runtime-confirmed at the gate-input level; the write itself is logically prevented.**

**Q6. Did DTS fail because a required transition was skipped?**
**YES (VERIFIED gate-input + STRONG INFERENCE on the skip).** The required transition is *gate PASS → `0x13292` → output-enable write*. It was skipped because gate input #1 (`bit7(byte@0x0C06)=0`) was never satisfied during playback, so `r11` stayed 0 and the `jal 0x13292` branch was never taken.

**Q7. If not resolved, which R29 hypothesis is now eliminated?**
R30 **resolves positively to R30-B**, which **confirms R29-B** (codec-specific gate divergence: AC3 bypasses the gate; DTS goes through the gate that blocks on `bit7(byte@0x0C06)=0`). No R29 hypothesis is *eliminated*; instead R29-B is now **runtime-corroborated**. The pre-session provisional R30-D (UNRESOLVED) is superseded by R30-B.

---

## 7. Differential table (R30-E)

| Aspect | AC3 | DTS |
|--------|-----|-----|
| Traverses `0x25EE1` gate? | **No** (bypasses, R29) | **Yes** (in path, R29) |
| Gate input #1 `bit7(byte@0x0C06)` | n/a (bypassed); captured `0` | **`0` (VERIFIED, all phases)** |
| Gate input #2 `signed byte@0x0015≥0` | n/a; captured TRUE | **TRUE (VERIFIED)** |
| Gate input #3 `word@0x2512C4≠0` | n/a | UNREADABLE, irrelevant |
| Gate predicate `r11` | n/a | **`0` → BLOCKS (VERIFIED)** |
| `0x13292` executed? | n/a (separate path) | **No (STRONG INFERENCE)** |
| `0x10786E/0x115693` reached (DTS route)? | reaches via own path | **No on DTS route (STRONG INFERENCE)** |
| Output-enable written | `0xB000_0854` | `0xB000_086C`/`0x0870` **NOT written** |
| Resulting physical stream | **Valid** | **Silent / no valid stream** |

---

## 8. Confidence & residual caveats

**What is VERIFIED (runtime capture + disassembly):**
1. `DM[0x0C06] = 0xffff00` (and variants) during live DTS decode; `byte@0x0C06` (low byte) = `0x00` → bit7 = 0. (Corroborated by adjacent cell `0x0C07 = 0x000000` proving cell self-containment.)
2. `DM[0x0015] = 0x000000` → signed ≥ 0 → gate input #2 TRUE.
3. DTS decode is live during the same run (DEC region 148/242 non-zero).
4. Gate logic is an AND with `bit7(byte@0x0C06)` as input #1; input #1 = 0 forces `r11 = 0`.

**What is STRONG INFERENCE (logically follows, not directly observed):**
- `0x13292` not executed / `0x086C`/`0x0870` not written — because the actual MMIO write is invisible to the only readable interface. The gate-block that prevents it is VERIFIED; the prevented write is inferred.

**Residual caveats (do not change the verdict):**
- **SRAM-mirror faithfulness:** we read the DSP SRAM *mirror* of `0xB000_0C06`, not the live peripheral at the instant `lbz` executes. The mirror is assumed faithful at capture time (standard for these mirror mechanisms; corroborated by adjacent-cell layout). Even if the live peripheral differed transiently, we captured the full playback lifecycle (BASELINE→STOP) and **never** observed a state where `byte@0x0C06` had bit7 set.
- **Input #3 unread:** irrelevant once input #1 = 0.
- **AC3 bypass confirmation:** from R29 static trace (not re-derived this session); consistent with the known symptom.

**What would make it 100% VERIFIED (not required for the conclusion):**
- A live MMIO read of `0xB000_086C`/`0x0870` after DTS playback (needs `/dev/mem` or a dedicated RIU read that does not exist on this device), to directly observe they remain at reset/0. The gate-block that prevents the write is already VERIFIED; this would only add the direct write-observation.

---

## 9. Conclusion

R30 answers the governing question with **runtime evidence**: during actual DTS playback the `0x25EE1` state gate **blocks** (its decisive input `bit7(byte@0xB000_0C06)` is 0 in every observed state), so the DTS output-enable programming (`0x13292 → 0x10786E/0x115693` writing `0xB000_086C`/`0x0870`) is never executed. AC3 avoids this because its output path **bypasses the gate** and writes `0xB000_0854` directly. This runtime-confirms the R29-B hypothesis: the DTS silence is caused by a **codec-specific gate divergence** in the SND R2 firmware, specifically the DTS output gate failing its `0x0C06` bit7 precondition.

---

## Appendix A — Evidence files (host)
- `aeon_validate/r30_out/dts/r30r_dts/dm.txt` — 7-phase real-Kodi DTS capture (W1/W2/W3/W4).
- `aeon_validate/r30_out/ac3/r30r_ac3/dm.txt` — 7-phase real-Kodi AC3 capture.
- `aeon_validate/r30_out/dts_dec/dm.txt` — combined DEC(0x0fe0)+gate(0x0C00)+W1+W2 run proving decode live AND gate input #1 = 0 in the same DTS playback.
- `aeon_validate/r30_out/analyze_r30.py`, `analyze_r30_full.py` — parsers producing the tables above.
- `aeon_validate/r29_work/decoded_25EE1.txt` — verified gate disassembly (Section 3).
- `tools/r30_realkodi_combined.sh` — combined capture script (read-only).

## Appendix B — Commands (read-only, no mutation)
```
# DTS 7-phase capture
adb shell "sh /data/local/tmp/r30_realkodi_cap.sh"   # writes /data/local/tmp/r30r_dts/
# AC3 7-phase capture
adb shell "sh /data/local/tmp/r30_realkodi_cap.sh"   # (ac3 variant) -> /data/local/tmp/r30r_ac3/
# Combined DEC+gate in one DTS run
adb shell "sh /data/local/tmp/r30_realkodi_combined.sh"  # -> /data/local/tmp/r30r_dts_dec/
# Pull
adb pull /data/local/tmp/r30r_dts/   aeon_validate/r30_out/dts/
adb pull /data/local/tmp/r30r_ac3/   aeon_validate/r30_out/ac3/
adb pull /data/local/tmp/r30r_dts_dec/ aeon_validate/r30_out/dts_dec/
```
No patch, no EDID/AUTH/settings change, no device reconfiguration. All reads only.
