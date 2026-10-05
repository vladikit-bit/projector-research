# FINDING — The SPDIF output gate that blocks DTS (DEC image) — corrected & proven

**Date:** 2026-09-12
**Image:** `mst_codec_r2_MS12V22.bin` (DEC/CODEC R2), 1,982,492 B = `0x1E401C`, md5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`
**Address model:** file offset == firmware address (no shift)
**Method:** Ghidra + `aeon.slaspec` (ground truth) — outputs in `aeon_validate/dec_work/dec0*.txt`

---

## 0. CORRECTION to the earlier version of this document

The first version of this finding claimed the gate compared against **`146` (0x92)** and **`1624`**,
and that the gate function had **zero callers**. **Both claims were WRONG** — they were artefacts of my
Python decoder (which covers only ~87% of this region and drifts out of phase).

Ghidra ground truth (byte-exact, `dec06_out.txt`) gives:

| claimed earlier | actual (Ghidra) |
|---|---|
| `bg.beqi r27, 146, ->0x21F6E` | `bg.beqi r27, **0x5**, ->0x21F6E` |
| `bg.blesi r24, 1624, ->0x22031` | `bg.blesi r24, **-0x1**, ->0x22031` |
| "0 callers" | **12 jal/j targets** into `0x21E00–0x21F60`, plus intra-function `bn.beqi` — the gate IS reachable |

The earlier "0 callers" came from a scan that only matched `bg.jal` with `bit0==0` and ignored the
**3-byte `bn.j` / `bn.beqi` family**. Do not repeat that scan pattern.

---

## 1. The gate — byte-exact

```
021F52  bg.movhi r24, 0xb000        [C3160001]   ; r24 = 0xB0000000 (audio MMIO base)
021F56  bn.ori   r27, r24, 0x1e     [53781E]     ; r27 = 0xB000001E
021F59  bn.lh    r27, 0x0(r27)      [0B7B01]     ; r27 = *(u16*)0xB000001E
021F5C  bg.beqi  r27, 0x5, 0x21F6E  [D3650092]   ; if (*(0xB000001E) == 5) -> 0x21F6E   (PASS)
021F60  bn.ori   r24, r24, 0x1d     [53181D]     ; r24 = 0xB000001D
021F63  bn.lbs   r24, 0x0(r24)      [171800]     ; r24 = *(s8*)0xB000001D
021F66  bg.blesi r24, -0x1, 0x22031 [D31F0658]   ; if (*(s8*)0xB000001D <= -1) -> 0x22031 (ERROR)
```

So: **`0xB000001E` must read exactly `5`; otherwise a negative `*(s8*)0xB000001D` sends execution to
the error path at `0x22031`.**

The error path ends in the diagnostic print whose string is
`"Invalid Spdif license:%d, output_spdifSz:%d"` @ rodata `0x144904` (single code site: `0x21FDA`,
materialised as `movhi r3,0x0281` + `ori r3,r3,0x4904` → VA `0x2814904`).

---

## 2. This gate is NOT alone — the same probe is used EIGHT times

`dec05_out.txt` (full MMIO base-composition sweep of the DEC image) shows **271 MMIO compositions
total, of which all 8 inside the gate window `0x21400–0x22200` are exactly two registers**:

```
0219A0  => 0xB000000A     (bit-5 test)
021A7D  => 0xB000000A
021AC7  => 0xB000001D
021B1D  => 0xB000001D     <-- inside the DTS handling branch
021E6A  => 0xB000000A
021EC8  => 0xB000000A
021F12  => 0xB000001D
021F60  => 0xB000001D
```

The `0xB000001E == 5` test appears at four sites:

```
0x21AC3  bg.beqi r26, 0x5, ->0x21AD9
0x21B19  bg.beqi r27, 0x5, ->0x21B2B     <-- DTS branch
0x21F0E  bg.beqi r26, 0x5, ->0x21F24
0x21F5C  bg.beqi r27, 0x5, ->0x21F6E
```

⇒ `0xB000000A` / `0xB000001D` / `0xB000001E` form a dedicated **hardware SPDIF capability/status
block**, polled repeatedly by the whole digital-output section. This is consistent with
"the SoC's SPDIF/Audio-peripheral reports whether the attached sink/output is DTS-capable".

---

## 3. The gated state IS firmware-written (⇒ a patch can move it)

`dec04_out.txt` — "writers of state fields used by gate":

```
--- stores to displacement 0xec (the field the gate reads at 0x21F6A/0x21F75) ---
  013DD1  bg.sh  0xec(r10),r23
  013DD5  bg.sw_1 0xec(r10),r26
  013E30  bg.sw_1 0xec(r10),r23
  013E3F  bg.sh  0xec(r10),r0      <- explicit zeroing
  ... (11+ stores in 0x013DD1-0x013F85)
```

The field `0xec(r10)` — read by the gate via `lhz r23,0xee/0xec,r11` — has **many firmware writers**.
It is therefore **internal firmware state, not a read-only hardware latch**, and the branch depending
on it is **patchable**.

---

## 4. The DTS-relevant branch

At the DTS site (`dec06_out.txt`, window `0x21A90–0x21AD0`):

```
021AA0  bg.lwz  r23, 0x118c, r21     ; r23 = a per-stream descriptor field
021AA4  bg.movhi r24, 0xb000
021AA8  bn.ori  r25, r24, 0x1e
021AAB  bn.lh   r26, 0x0(r25)        ; r26 = *(0xB000001E)
021AAE  bg.muli_maybe r25, r23, 0x60 ; r25 = r23 * 0x60
021AB2  bn.sfnei r23, 0x0            ; r23 == 0 ?
021AB5  bg.ori  r23, r0, 0x4800      ; default value 0x4800 (= 18432)
021AB9  bn.cmov_0 r25, r25, r23      ; r25 = (r23!=0) ? r25 : 0x4800
021AC3  bg.beqi r26, 0x5, 0x21AD9    ; <-- the SAME 0xB000001E == 5 gate
021AC7  bn.ori  r24, r24, 0x1d
021ACA  bn.lbz  r23, 0x0(r24)        ; r23 = *(0xB000001D)
021ACD  bn.andi r23, r23, 0x7f
```

Note `0x4800` = **18432 bytes** — a frame-size-shaped constant selected when the descriptor field
is zero, and the same path then tests the SPDIF capability block. `output_spdifSz` in the error string
is consistent with this being the frame-size variable.

---

## 5. Patch candidates (minimal, reversible, one change)

All from the **stock DEC image** `mst_codec_r2_MS12V22.bin` (md5 `4b7e9509…`). Never overwrite the
stock file; write a new versioned artifact.

### Option L1 — force the licensed branch at the SPDIF gate (4 bytes)
At `0x21F5C`: `d3 65 00 92` = `bg.beqi r27,0x5,->0x21F6E`.
Replace with an unconditional branch to `0x21F6E`, or make the compare always-true
(e.g. compare `r27` with itself). Effect: the `*(0xB000001E)==5` test always passes; the error path
`0x22031` becomes unreachable for this site.

**Caveat:** there are FOUR such sites (§2). Patching only one may be insufficient; the DTS branch site
is `0x21B19`. Patching all four is a larger change (still only 4 bytes each) and must be justified
by which site the DTS path actually traverses.

### Option L2 — neutralise the negative-value test only
At `0x21F66`: `d3 1f 06 58` = `bg.blesi r24,-0x1,->0x22031`.
Remove the branch so a negative `*(s8*)0xB000001D` no longer escapes to the error path.
Leaves the `==5` test intact.

### Blocking question before writing any byte
`0xB000001E`/`0x1D` are **read-only MMIO** from the firmware's point of view (0 writers in the image —
the 8 sites are all reads). Per the project's own pre-patch rule (skill §10, step 3), a field with zero
firmware writers is hardware-driven: **we cannot make the hardware return `5`; we can only make the
firmware ignore the verdict.** That is exactly what L1/L2 do — but it must be stated as
"bypass the verdict", not "fix the capability".

Whether the bypass is *sufficient* depends on what else consumes the same verdict downstream
(the `0xec(r10)` state is written by many functions — see §3). This is the residual risk.

---

## 6. Why every earlier patch failed (reconciled)

| patch | why it missed |
|---|---|
| **V1** (`asnd_r2_MS12V22.bin`, 1 byte `0x1C051`) | **wrong image** (SND, not DEC) and **wrong function** (MS12V2 DDP encoder node). Verdict: `PATCH TARGET INVALID` |
| **R8-Exp-1** (`utpa2k_cmd04_isac3.ko`, forced `is-AC3=1`) | the only proven `+0xf8` consumer is the speaker/DMX downmix handler, not the digital output. Outcome C: no change |
| **R5** (`mik.ko`, `r6=1` at `0x98a58`) | DTS dies *upstream* in Kodi's own `no pass-through` decision; the kernel never sees DTS |
| **Probe6/7** (IEC61937 over a PCM16 carrier) | never enters this DEC function, so the gate is never reached |
| **R10's diagnostic kernels** (`alllic_diag`, `dtslic_pass`) | targeted ARM-side license getters; the verdict is derived in the DSP, not from those |

---

## 7. Runtime instruments available

### 7.1 `dump_spdif_npcm` — wire capture of the SPDIF TX
```sh
echo 'dump_spdif_npcm=1 path=1' > /proc/utopia_mdb/audio   # param1: 0=stop,1=start; param2: 0=/tmp,1=/data
echo 'dump_spdif_npcm=0 path=1' > /proc/utopia_mdb/audio
```
Writes `/data/DUMP_audio_spdifNpcm_NN.bin`; dmesg: `----- Start Dump Spdif Tx Npcm (mode:0) -----`.

**GOTCHA (proven):** the capture is a **global singleton**. Re-arming while active returns
`[Err] dump_spdif_npcm is already activated!!`, and if the file is `rm`'d while still armed the writer
keeps writing to a **deleted inode** → every subsequent file is **0 bytes forever**.
Always send `dump_spdif_npcm=0 path=1` first; never delete while armed.

### 7.2 Wire format (proven)
Each 16-bit word is byte-swapped on the wire. AC-3 burst header reads
`72 f8 1f 4e 01 00 00 30 77 0b…` → Pa=`0xF872`, Pb=`0x4E1F`, **Pc=`0x0001` (AC-3)**, Pd=`0x3000`,
payload sync `0x0B77`.

### 7.3 Proven wire differential (01:53 capture set)
| file | size | content |
|---|---|---|
| `ac3_03.bin` | 571,392 B | live stream, 93 blocks, 3 unique |
| `dts_01.bin` | 270,336 B | **same AC-3 header** (`Pc=0x0001`, `Pd=0x3000`, sync `0x0B77`) with a **frozen payload block repeated 26×** |

⇒ the DTS IEC61937 writer is **never invoked**; the SPDIF peripheral retransmits stale buffer content.

### 7.4 Decoder status
```sh
echo show_all_decoder_status > /proc/utopia_mdb/audio   # then dmesg
```
DTS → `Decoder format : DTS`; AC-3 → `AC3P`; idle → `INVALID`/`STOP`.

### 7.5 MMIO cannot be read directly
`/dev/mem` absent; `mknod /dev/mem c 1 1` succeeds but reads fail with
`No such device or address` (kernel STRICT_DEVMEM). `devmem ADDR WIDTH [DATA]` takes WIDTH in
**bytes**. `reg_bank=0` is rejected. ⇒ the gate inputs cannot be measured; only the code path can.

---

## 8. Reusable tooling facts

- **`bg.jal` / `bg.j` displacement:** `target = pc + s25((word32 >> 1) & 0x1FFFFFF)`,
  bit0 of the word must be `0` for `jal`, `1` for `j`; opcode field `(w>>26)&0x3F == 0x39 (57)`.
  Use the **4-byte** word (a 3-byte listing silently truncates byte 3).
- **Instruction phase is NOT uniform.** This region mixes 3-byte `bn.*` and 4-byte `bg.*`.
  A fixed-stride sweep will land mid-instruction and invent operands (this is what produced the
  bogus `146`/`1624`). **Always verify with Ghidra + the slaspec before trusting an operand.**
- **Immediates rendered as registers:** `bn.sflesi` / `bg.blesi` style ops take an **immediate**, but
  the Python decoder printed them as `r31`/plain numbers. Cross-check every predicate operand.
- **Address models:** DEC `file_offset == fw_addr`; SND `file_offset = fw_addr + 0x16F00`.
- **Ghidra recipe (this sandbox):** reuse the analysed project; set `JAVA_HOME` + `APPDATA`;
  `-noanalysis`; `-scriptPath` required. Ghidra *does* decode AEON here via the installed slaspec
  (script outputs `dec03_out.txt` … `dec21_out.txt` were produced this way). The old claim
  "Ghidra cannot decode AEON" (R24 §1) is **outdated** — the language is present now.
