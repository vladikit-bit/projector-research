# R54 — Decisive trace of the DTS output-pack path (0x26270–0x2661D)

**Image:** `snd_full.bin` (AEON `aeon:LE:32:default`, MStar MT5889, Thundeal TD98 Pro)
MD5 `eb879cdc07f510722f19db6d18d77d3c`, 1,839,920 B
**Tool:** Ghidra `analyzeHeadless` (project `r27_work/ghidra_r27f` / `sndr27`, `-noanalysis`),
scripts `R54_TracePacker.java` + `R54b_FollowPack.java`
**Purpose:** decide whether a **firmware-writable node on the DTS output path** reaches an
existing output/SDO/TX arm action — i.e. whether a defensible single-instruction DTS patch
exists at all.

---

## 1. The function that contains the `0xC49D` gate at `0x265F3`

Entry `0x26270` (`bn.addi r1,r1,-0x48`), return `0x2646B` (`bt.jr r9`).
`getFunctionContaining(0x265F3)` → `<none: undefined region>` in the `-noanalysis`
project; the entry was recovered from the prologue at `0x26270` (R49 §2.4 said the same).

**Entry:**
```
0x26270  bn.addi r1,r1,-0x48
0x26273  bg.movhi r23,0xb000
0x26277  bn.sw 0x40(r1),r10
0x2627A  bn.sw 0x44(r1),r9
...
0x26289  bn.ori  r23,r23,0xa
0x2628C  bn.lh   r24,0x0(r23)      ; r24 = (s16)*(0xB000_000A)
0x2628F  bn.lwz  r23,0x58(r3)
0x26292  bt.mov  r10,r3
0x26294  bn.exthz r24,r24
0x26297  bg.bnei r23,0x0,0x264ff  ; dispatch
0x2629B  bg.andi r13,r24,0x200
0x2629F  bg.bnei r13,0x0,0x26480
```

---

## 2. The DTS-specific sub-block: `0x26574` → `0x2661D`

Reached from `0x2644A  bg.beqi r14,0x1,0x26574` — i.e. this is the **second branch of a
pair of tests on `0xB000_0814` bytes**, immediately after the first test
(`0x2643F  bg.beqi r23,0x1,0x2662b`) failed.

```
0x26574  bn.lwz r23,0x3c(r10)                 ; slot A
0x26577  bg.bnei r23,0x0,0x2644e              ; if slot A != 0 -> exit
0x2657B  bn.lwz r23,0x44(r10)                 ; slot B
0x2657E  bg.ori r25,r0,0xbb80                 ; 48000
0x26582  bg.bgts r23,r25,0x2644e              ; if slot B > 48000 -> exit
0x26586  bn.lwz r23,0x60(r10)                 ; slot C
0x26589  bg.movhi r12,0x4e
0x2658D  bn.bnei r23,0x0,0x265af              ; slot C != 0 -> other-helper path
0x26590  bg.addi r3,r12,0x6ad4
0x26594  bg.jal 0xca42                        ; init helper
0x26598  bg.addi r3,r12,0x6ad4
0x2659C  bg.jal 0xc55e                        ; init helper
0x265A0  bg.movhi r3,0x13
0x265A4  bn.sw  0x60(r10),r14
0x265A7  bg.addi r3,r3,-0x3994
0x265AB  bg.jal 0x10786e                      ; log
0x265AF  bg.addi r3,r12,0x6ad4                ; r3 = 0x004E6AD4  (codec ctx ptr)
0x265B3  bt.mov r11,r3                        ; r11 = ctx
0x265B5  bg.jal 0xc49d                        ; *** PREDICATE A #1 (pre-CC9B guard) ***
0x265B9  bn.lwz r23,0x14(r11)
0x265BC  bg.bnei r3,0x0,0x26773               ; if bits17|19 SET -> skip whole block
0x265C0  bg.sflesi r23,0x347f
0x265C4  bn.bf  0x2674e
0x265C7  bg.movhi r23,0x1
0x265CB  bg.addi r23,r23,-0x6000              ; r23 = 0x10000 - 0x6000 = 0xA000
0x265CF  bn.lwz r24,0x28(r11)
0x265D2  bg.lwz r25,0x884,r23                 ; counter
0x265D6  bt.addi r24,0x1
0x265D8  bt.addi r25,0x1
0x265DA  bg.movhi r3,0x1
0x265DE  bg.addi r3,r3,-0x577c                ; r3 = 0xA884
0x265E2  bt.movi r4,0x4
0x265E4  bg.sw_0 0x884,r23,r25                ; counter++
0x265E8  bn.sw  0x28(r11),r24                 ; ctx+0x28++
0x265EB  bg.jal 0xd4c8                        ; shared finalizer
0x265EF  bg.addi r3,r12,0x6ad4
0x265F3  bg.jal 0xc49d                        ; *** PREDICATE A #2 (canonical, gated) ***
0x265F7  bt.mov r14,r3
0x265F9  bg.addi r3,r12,0x6ad4
0x265FD  bg.jal 0xcc9b                        ; DTS-clear helper (clears 0x854 bits17|19)
0x26601  bg.bnei r14,0x0,0x26451              ; if predicate TRUE -> SKIP CB50
0x26605  bn.lwz r23,0x14(r11)                 ; size
0x26608  bg.sflesi r23,0x2fff
0x2660C  bn.bf  0x26451                       ; if size > 0x2FFF -> SKIP CB50
0x2660F  bt.mov r3,r11
0x26611  bg.ori r4,r0,0xbb80
0x26615  bg.jal 0xcb50                        ; *** AC3-SET helper ***
0x26619  bg.beqi r13,0x0,0x26454
0x2661D  bg.j 0x2648e
```

**This is the only site in the whole image where the `CC9B→CB50` pair sits on a
DTS-specific (format-20) path that is guarded by the `0xB000_0814` byte test at
`0x2644A`.** It is also the site whose `0xC49D` gate is `0x265F3` — the address the user
named.

---

## 3. Where the branch actually lands: `0x26451`

```
0x2643F  bg.beqi  r23,0x1,0x2662b      ; 0x814 byte0 == 1 ? -> 0x2662B
0x26443  bn.sw   0x5c(r10),r0
0x26446  bg.opcode_2E r14              ; (undecoded ALU)
0x2644A  bg.beqi  r14,0x1,0x26574      ; 0x814 byte? == 1 ? -> 0x26574  (our DTS block)
0x2644E  bn.sw   0x60(r10),r0
0x26451  bn.bnei  r13,0x0,0x2648e      ; ---- THE JOIN TARGET ----
0x26454  bt.movi  r3,0x0               ; <-- COMMON FUNCTION EPILOGUE
0x26456  bn.lwz   r9,0x44(r1)
...
0x2646B  bt.jr    r9                   ; RETURN
```

`0x26451` is the **join point of the DTS sub-block**. Reaching it means:
`CC9B` ran, `CB50` was skipped, and then — because `r13 == 0` — control falls straight into
the **common function epilogue at `0x26454` and RETURNS**.

That is: on the DTS path, the predicate's TRUE result does not merely skip a register write.
**It exits the entire packer function.** The DTS sub-block never reaches the frame-build
section that follows (`0x262A3`–`0x2632C`, entered only via `0x26480`/`0x2648A`, i.e. only
when `r13 != 0`).

---

## 4. The two outputs of the DTS sub-block

| Predicate A (`0x265F3`) | Path | Consequence |
|---|---|---|
| **TRUE** (bits 17\|19 already set) | `0x26601` taken → `0x26451` → `r13==0` → `0x26454` | `CB50` skipped; **function returns at `0x2646B`**; no frame produced |
| **FALSE** (bits clear) + size ≤ 0x2FFF | `0x2660C` not taken → `0x26615` `CB50` | `0x854` bits 17\|19 SET; falls to `0x26619`/`0x2661D` → `0x2648E` → frame builder |

So the *only* firmware-writable node on this path that changes downstream behaviour is
**`0x26601  bg.bnei r14,0x0,0x26451`**.

---

## 5. Why this is nonetheless NOT a viable DTS fix

### 5.1 The predicate is already FALSE during real DTS playback

`0x265F3` reads `0xB000_0854` bits 17|19. Per R46d/R49 the DTS-clear helper `0xCC9B`
**precedes** the predicate only at the *other* DTS caller (`0x1F75C`, R49 §3). Here, at
`0x265F3`, the ordering is `0x265F3` (P2) **first**, then `0x265FD` (`CC9B`). By R49 §3's
own cross-comparison table this site is in the "**P2 before CC9B**" group — i.e. the
predicate reads the *entry* latch, exactly like the AC3 caller `0x1EDF0`.

The runtime record (§5.3) shows the latch is **never set during either codec**, so the
predicate is **already FALSE** on the DTS path. `0x26601` is therefore **never taken**, and
replacing it with a NOP or an unconditional branch **changes nothing at runtime**.

### 5.2 R49-B: the bit pair is a reconciled latch, not a codec selector

R49 §4: the `CC9B → P2 → CB50` triplet is a **clear / re-read / conditionally-set latch
re-arm**; the decision lives upstream. Making `0x26601` unconditional would force the
AC3-set helper to overwrite the DTS-clear latch state — i.e. it would set an "AC3
configured" bit on a DTS stream. The blast radius extends to the two other consumers of
`0x854` bits 17|19 (`0xC4E0`, and the `0xC55E`/`0xC5C5` siblings), both of which are
**shared with the working AC3 path**.

### 5.3 Runtime ground truth — the divergence is upstream of the packer

From the R47 dense capture (`r47_dense_{ac3,dts}.log`), 20 samples each:

| Register | AC3 (non-zero samples) | DTS |
|---|---|---|
| `0x0C00` | 33 distinct non-zero values (`0xdb6f00`, `0xad6b00`, `0x9f3500`, `0x24f900`, …) | **0** |
| `0x0C01`–`0x0C05` | non-zero in 16 of 20 samples | **0** |
| `0x0C06` | non-zero in 4 samples; **byte0 bit7 SET in 3 of them** (`0xad`, `0xdb`, `0xe3`) | **0** |
| `0x0C07` | 4 non-zero samples (`0xf3e700` …) | **0** |
| `0x0FE8` (DEC liveness) | `0x014`,`0x020`,`0x004`,`0x002` | `0x018`,`0x020`,`0x01e`,`0x014`,`0x012` |

The `0x0C00–0x0C07` block is the **hardware output-status block** (R46b/R47e: **0 firmware
stores**, 10–13 firmware readers). Its contents are produced by the **transmit hardware**,
not by the DSP firmware.

**The `0x25EE1` gate's first condition — `bit7(byte@0x0C06)` — is therefore satisfied during
AC3 (3/20 samples) and *never* during DTS (0/20).** The decoder is demonstrably alive in
both cases (`0x0FE8` non-zero throughout DTS), so the DTS audio is decoded but the
**hardware-reported output status never populates**.

### 5.4 No firmware-writable node can influence it

- `0x0C00–0x0C07`: **0 firmware stores** (Ghidra-authoritative; `r53d_status.txt`).
- `0x0814` bit21 (the `0xCC9B`/`0xC49D` early-out guard): **0 firmware stores** — the
  R47 capture reads `0x0814` as `0x000000` in **20/20 samples for both codecs**, and the
  firmware never writes it.
- `0x0854` (the only firmware-written correlating field): reads `0x000000` in **20/20
  samples for both codecs** — the value the packer writes never becomes host-visible, and
  R49-B has already ruled it out as a selector.
- `0x086C` / `0x0870` / `0x0858`: `0x000000` in **20/20 for both codecs** (R46b §21.3: no
  functional firmware reader).

Writing *any* of these changes no downstream observable.

---

## 6. R54 verdict

**The DTS output-pack path in firmware (`0x26270`–`0x2661D`) is not the blocker.**

- Its gate (`0x265F3`) is an already-FALSE latch re-arm; the branch at `0x26601` is
  never taken at runtime.
- The DTS sub-block's real exit (`0x26451` → `0x26454`) is a **function return**, not an
  output arm; the frame-builder it bypasses is only reached on the `r13 != 0` path.
- The input that would make the block produce a frame — hardware output status
  `0x0C00–0x0C07` — is **all-zero during DTS and populated during AC3**, and it has
  **zero firmware writers**.

**⇒ No single firmware instruction in this region can restore DTS passthrough. There is no
firmware-writable node on the live DTS output path.** This independently confirms the
R47e/§D4.3 conclusion and closes STEP 1 with an honest negative.
