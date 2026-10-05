# FINDING — The DEC capability gate, re-derived byte-exactly (pre-experiment)

**Date:** 2026-09-13
**Image:** `aucode_adec_r2_MS12V22.bin` ≡ `mst_codec_r2_MS12V22.bin` ≡ `dec_full.bin`
**Size:** 1,982,492 B (`0x1E401C`)  ·  **md5:** `4b7e9509b4358fd3a130bd4d3b9cbe0a`
**Address model:** **file offset == firmware address (no shift)**
**Method:** Ghidra ground truth (`dec_work/dec03_out.txt`, `dec06_out.txt`) + independent raw-byte re-encoding

---

## 0. Why this document exists

This is the **pre-experiment** derivation for the one change that has **never been tested**.
It supersedes the encoding guesses in `FINDING_rootcause_license_gate.md` §5, which were
written before the branch encoding was solved. Everything below is byte-verified against the
AEON slaspec (`aeon_ORBIS32.sinc`) and cross-checked against the Ghidra listing.

**No bytes have been written to any device image for this experiment yet.**

---

## 1. The exact instruction encoding (SOLVED — previously unknown)

From `aeon_ghidra_public/aeon/data/languages/aeon.slaspec`:

```
define token instr32(32)
   i32_opcode   = (26,31)
   i32_rD       = (21,25)
   i32_rA       = (16,20)
   i32_uimm0_3  = (0,2)
   i32_simm16_5 = (16,20) signed
   i32_uimm16_5 = (16,20)
   i32_rel_simm3_13 = (3,15) signed
```

From `aeon_ORBIS32.sinc`, `with : i32_opcode = 0x34`:

| `i32_uimm0_3` | mnemonic | semantics |
|---|---|---|
| 0 | `bg.blesi` | `if (rD s<= imm16) goto` |
| 1 | `bg.bleui` | `if (rD u<= imm16) goto` |
| **2** | **`bg.beqi`** | `if (rD == imm16) goto` |
| **3** | **`bg.011i`** | **`goto` (UNCONDITIONAL)** |
| 4 | `bg.bgtsi` | `if (rD s> imm16) goto` |
| 5 | `bg.bgtui` | `if (rD u> imm16) goto` |
| 6 | `bg.bnei` | `if (rD != imm16) goto` |
| 7 | `bg.b111i` | `goto` (also unconditional) |

### 1.1 The branch base is **PC**, not PC+4  (proven)

```
0x21F5C: D3650092  beqi r27,5   -> Ghidra target 0x21F6E
         rel field = 18 ;  0x21F6E - 0x21F5C = 18  ✓  (base = PC)
0x21F66: D31F0658  blesi r24,-1 -> Ghidra target 0x22031
         rel field = 203 ; 0x22031 - 0x21F66 = 203 ✓  (base = PC)
```

Encoding for re-use:

```
word32 = (0x34 << 26) | (rD << 21) | (imm5 << 16) | ((target - pc) & 0x1FFF) << 3 | u3
```

### 1.2 Immediate field widths (proven)

* `bg.blesi` takes `i32_simm16_5` = **5-bit signed** → `0x1F` means **−1**.
* `bg.beqi`  takes `i32_uimm16_5` = **5-bit unsigned** → `0x05` means **5**.

Both verified by re-encoding the stock words and matching byte-for-byte.

---

## 2. The gate function — authoritative control flow

Verified listing (`dec_work/dec03_out.txt`, window `0x021F60 – 0x022080`):

```
; ---- retry-loop head ----
021F52  bg.movhi r24, 0xb000
021F56  bn.ori   r27, r24, 0x1e
021F59  bn.lh    r27, 0x0(r27)        ; r27 = *(u16*)0xB000001E
021F5C  bg.beqi  r27, 0x5, 0x21F6E    ; if (*(0xB000001E) == 5) -> PASS
021F60  bn.ori   r24, r24, 0x1d
021F63  bn.lbs   r24, 0x0(r24)        ; r24 = *(s8*)0xB000001D
021F66  bg.blesi r24, -0x1, 0x22031   ; if (r24 s<= -1) -> ERROR   <<<< THE GATE
; ---- fall-through ----
021F6A  bg.lhz   r23, 0xee, r11
021F6E  bn.beqi  r23, 0x1, 0x21FA3
021F71  bg.bnei  r23, 0x2, 0x21E8D
021F75  bg.lhz   r23, 0xec, r11
021F79  bg.lwz   r24, 0x94, r11
021F7D  bt.addi  r23, -0x1
021F7F  bt.addi  r24, 0x1
021F81  bn.exthz r23, r23
021F84  bg.sw_0  0x94, r11, r24
021F88  bg.sw_1  0xec, r11, r23
021F8C  bn.beqi  r23, 0x0, 0x21F97
021F8F  bg.opcode_2E r26
021F93  bg.bles  r14, r26, 0x21E77
021F97  bg.sw_1  0xec, r11, r0        ; Z1
021F9B  bg.sh    0xec, r11, r0        ; Z1
021F9F  bg.j     0x21E77
...
021FA3  bg.lhz   r23, 0xec, r11       ; PASS landing from 021F6E
021FA7  bt.addi  r23, -0x1
021FA9  bn.exthz r23, r23
021FAC  bg.sw_1  0xec, r11, r23
021FB0  bn.beqi  r23, 0x0, 0x21FBE
021FB3  bn.sub   r23, r17, r25
021FB6  bg.opcode_2E r23
021FBA  bg.bles  r14, r23, 0x21FC6
021FBE  bg.sh    0xec, r11, r0        ; Z2
021FC2  bg.sw_1  0xec, r11, r0        ; Z2
021FC6  bt.movi  r24, 0x1
021FC8  bg.lhz   r23, 0xee, r11
021FCC  bn.sw   0x64(r10), r24
021FCF  bn.sw   0x74(r10), r0
021FD2  bg.j     0x21F71              ; loop back into the dispatch
;
02201F  bg.sh   0xec, r11, r0         ; Z3
022023  bg.sw_1 0xec, r11, r0         ; Z3
022027  bn.sw   0x64(r10), r0
02202A  bn.sw   0x74(r10), r0
02202D  bg.j     0x21F52              ; RETRY the whole gate
;
022031  bg.beqi  r23, 0x0, 0x21E8D    ; ERROR landing from the gate
022035  bg.sw_1  0xec, r11, r0        ; Z4  <<<< the license suppression
022039  bg.sh    0xec, r11, r0        ; Z4  <<<< the license suppression
02203D  bg.j     0x21E87
```

### 2.1 Gate truth table

| `*(u16*)0xB000001E` | `*(s8*)0xB000001D` | outcome |
|---|---|---|
| `== 5` | (not read) | **PASS** → `0x21F6E` |
| `!= 5` | `>= 0` | fall through → `0x21F6A` (normal processing) |
| `!= 5` | `<= -1` | **ERROR** → `0x22031` → **output zeroed** |

⇒ **The suppression is keyed on the SIGN of `0xB000001D`, not on `0xB000001E`.**
`0xB000001E == 5` is only a fast-path that skips the second test.

---

## 3. Reachability of the error path — exactly ONE branch

Aligned whole-image scan of the DEC image for any instruction whose target is `0x22031`:

```
0x021F66: d3 1f 06 58  blesi r24,-1 -> 0x022031     <<< the only one
```

⇒ **One branch, one site.** (The earlier finding's worry that "there are FOUR such sites,
patching only one may be insufficient" conflated the four `==5` *tests* with the error path.
The four tests are `0x21AC3`, `0x21B19`, `0x21F0E`, `0x21F5C`; only **`0x21F66`** can reach
`0x22031`.)

---

## 4. Zeroings of the output-size field `0xec(r11)` — five, only one is the gate

| # | sites | reached from | nature |
|---|---|---|---|
| Z1 | `021F97`/`021F9B` | `021F8C beqi r23,0` | **legitimate** (size decremented to 0) |
| Z2 | `021FBE`/`021FC2` | `021FB0 beqi r23,0` | **legitimate** |
| Z3 | `02201F`/`022023` | `022062 bles` | **legitimate** |
| **Z4** | **`022035`/`022039`** | **`021F66` gate only** | **the license suppression** |

⇒ Patching `0x21F66` disarms **only** Z4. All legitimate zeroings remain intact.

---

## 5. The change (the one untried patch)

### Option L2 — make the `<= -1` escape unreachable

```
file / fw offset 0x21F66:
   STOCK : d3 1f 06 58   bg.blesi r24, -0x1, ->0x22031
   PATCH : d3 1f 00 23   bg.011i r24, 0x1F, ->0x21F6A     (unconditional goto to the next insn)
```

Byte-level derivation (all fields re-verified):

```
pc     = 0x21F66
target = 0x21F6A        (the instruction immediately after)
rel    = 0x21F6A - 0x21F66 = 4
u3     = 3              (unconditional goto)
rD     = 24             (preserved; irrelevant for a goto)
imm5   = 0x1F           (preserved; irrelevant for a goto)

word32 = (0x34<<26)|(24<<21)|(0x1F<<16)|(4<<3)|3
       = 0xD31F0023
bytes  = D3 1F 00 23
```

**Semantics:** execution unconditionally continues at `0x21F6A`, exactly as if the `blesi`
had not been taken. The `<= -1` condition can no longer divert to `0x22031`, so Z4 becomes
unreachable. The `==5` fast path at `0x21F5C` is untouched (harmless either way).

**Diff size: 2 bytes changed** (`d3 1f 06 58` → `d3 1f 00 23`).

### Why not Option L1

L1 (make `0x21F5C beqi` unconditional) only forces the `==5` fast path. When
`*(0xB000001E) != 5` the code would still fall through to `0x21F60`–`0x21F66` and could still
escape to `0x22031`. **L1 is strictly weaker.** L2 removes the escape entirely.

### Why not patch the four `==5` sites

They are not an error path. They select the fast path vs. the slow path. Leaving them alone
preserves the original timing/behaviour as much as possible (minimal-change principle).

---

## 6. Rollback

```
Stock DEC  : md5 4b7e9509b4358fd3a130bd4d3b9cbe0a   (local: dec_work/dec_full.bin)
Patched    : aucode_adec_r2_MS12V22_GATE_L2.bin
On device  : /vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin
Rollback   : cat <stock copy> > <device path> && sync && reboot
```

**Prerequisite before any deploy:** stage an on-device stock backup
(`/data/local/tmp/adec_stock.bin`) — currently **absent**. The SND image also still carries the
inert V1 Pc patch (`785fd73b…`), which should be restored to stock
(`eb879cdc07f510722f19db6d18d77d3c`) first so the experiment has a clean baseline.

---

## 7. Success criterion

**A NON-ZERO DTS wire capture** from `dump_spdif_npcm` while `show_all_decoder_status` reports
`format: DTS / state: PLAY`. Not an AVR lock. An AC-3 capture must remain non-zero (regression
control).

---

## 8. Residual risk (stated honestly)

* `0xB000001D` / `0xB000001E` are **read-only MMIO** (0 firmware writers). We cannot make the
  hardware report a different value; we can only make the firmware **ignore the verdict**. If the
  same verdict is consumed elsewhere in a way that also matters, L2 will not be sufficient.
* The `0xec(r11)` state is written by many functions (§4 of the older finding). L2 does not
  change those writes.
* L2 is a **bypass**, not a fix. It is legitimate as a *diagnostic*: if the DTS capture becomes
  non-zero after L2, the gate is proven to be the blocker and a proper fix can be designed. If
  the capture stays 0 bytes, the gate is exonerated and the search moves on.

---

## 9. RESULT (2026-09-13, executed)

**Verdict: `PATCH HAD NO EFFECT`.**

* Artifact `artifacts/aucode_adec_r2_MS12V22_GATE_L2.bin` (md5 `cba5f7e795818462b55736543329c9ea`),
  diff = exactly 2 bytes (`0x21F68` `06`→`00`, `0x21F69` `58`→`23`), was deployed and the device was
  rebooted (verified via `/proc/uptime` reset); the patched word was confirmed on-device with `xxd`.
* Baseline SND was restored to stock first (`eb879cdc…`), removing the inert V1 Pc patch.
* Wire capture (after `adb root`; instrument proven by the AC-3 control in the same session):

| source | decoder status | wire capture |
|---|---|---|
| `test_dts_51.mp4` | `format: DTS`, `PLAY`, 48000 Hz, 10 ch, **frame count 1863** | **0 bytes** |
| `test_ac3_51.mp4` | `format: AC3P`, `PLAY`, **frame count 719** | **5,431,296 bytes** |

⇒ The DEC capability gate at `0x21F66` is **not** the last blocker. See
`REPORT_final_dts_patch_experiment.md` §9 for the full write-up and the corrected next direction.
