# FINDING — 2026-09-30: the DTS block is DOWNSTREAM of decode, and the licence is a dead end (again, now proven by the DSP's own log)

**Status: live-measured. This supersedes the "DTS npcm = 0 B" measurements in the archive,
all of which were taken with a wedged audio monitor and prove nothing.**

---

## 1. Why every previous measurement in this project read zero

Two harness defects, both mine, both found today:

1. **Kodi's `Player.Stop` requires `playerid`.** Without it the call returns an error and the
   player never releases the audio device. Every "stop, then play the other format" cycle in
   the archive was actually "keep playing, open on top".
2. **`[Audio Mon Task]` wedges in state `D`.** Symptom: `CActiveAE::InitSink - failed to init`
   and `CDVDAudio::AddPacketsRenderer - timeout adding data to renderer` for *both* codecs,
   and every dump reads 0. Check: `ps -A -o PID,STAT,NAME | grep -aE 'MsOS_WaitEvent|Audio Mon Task'`
   — `D` is broken, `S` is alive. **Only a reboot clears it.**

## 2. The live differential (clean system, same player, same method)

| | decoder ES (`dump_es`) | optical TX (`dump_spdif_npcm`) |
|---|---|---|
| **DTS** | **2 883 196 B** — 1586 × `7F FE 80 01`, subframe `0x77` | **0 B** |
| **AC-3** | 714 240 B | **2 727 936 B** |

DTS head: `7f fe 80 01 fc 3c 7d b2 77 00 05 38 00 09 ef 7b`

**The DTS decoder receives a fully valid IEC61937 stream and produces a fully valid IEC61937
output. The optical transmitter emits nothing.** The block is downstream of the decode. Every
previous hypothesis in this project located the block at decoder *entry* or in the DEC output
*gate arithmetic*; the evidence now places it after the packer.

Kodi requests passthrough identically for both: `AE_FMT_RAW (AE) method: RAW (PT)`, with
`STREAM_TYPE_DTS_512` vs `STREAM_TYPE_AC3`.

## 3. The licence gate is FALSIFIED by the DSP's own log

`dump_r2_log_start=0 9E=0x16 PATH=1 8A(88)=0x1` dumps the **DEC R2 firmware's own log**
(73 KB, never used before in this project). Under DTS it prints:

```
dts m6 hook ok
dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0
```

and under AC-3:

```
ms11 ddp_only hook ok
ms11 ddp_only init ok, dd_licensee=0, ddp_licensee=0
```

**Every licence field is 0 in both cases, and DTS still decodes 2.88 MB.** The licence is
therefore not a gate in DEC R2 — it is informational, or it is read later by the SE. This
retires the whole IPAUTH / SE / `0xB000001C` line of investigation for the *decode* stage, and
is consistent with the earlier finding that `0x112CF0 = 0xFF00` (host licence open) while
`type<97>` (= IP-7 pass) appears in the R2 log.

The R2 log also shows the decoder genuinely running under DTS: `play=1`, `state=3,0`.

## 4. The static output stage, read from `dec33_realcode.txt`

`0x7FFE8001` appears at 31 sites; 29 are **comparisons** (sync-word detection in the parsers).
Only two **write** it, and one of those is the real packer:

**The IEC61937 packer — `0x46259..0x4629C`**
```
0x046259 lwz  r23,0x2c(r10)      ; packer-enable
0x04625C bnei r23,0x1,0x461ab    ; != 1 -> packer SKIPPED
0x046260 lwz  r23,0x30(r10)
0x046263 bnei r23,0x0,0x462de
0x04626A lwz  r5,0xf8(r10)       ; buffer A
0x046271 lwz  r23,0xfc(r10)      ; buffer B
0x04627B ori  r25,r0,0x7ffe
0x04627F sw   0x0(r24),r25       ; write sync
0x046282 add  r24,r23,r16
0x046285 ori  r25,r0,0x8001
0x046289 sw   0x0(r24),r25       ; write sync
```
`0x5381C` is a second, unrelated bitrate-silhouette packer with no reachable entry in this
image (its jumptable is absent) — a clean negative.

**The early exit that gates the whole output stage — `0x461A1`**
```
0x0461A1 lwz  r23,0x28(r3)       ; output-stage enable
0x0461A4 mov  r10,r3
0x0461A8 beqi r23,0x1,0x461c4    ; == 1 -> continue
0x0461AB sw_0 0xf0(r10),r14,     ; != 1 -> zero the size and LEAVE
0x0461BF addi r1,r1,0x20
0x0461C2 jr  r9                  ; return
```
This is the **only** early return on the output path before the packer, and it is a
single-instruction target. Writers of `+0x2C`: `0x46013` (stores 0) and `0x46087` (stores the
value computed at `0x46083 jal 0x64149`).

Known-good sites in the same function: size divert `0x4604F bgtui r3,0xd,0x46000`, and the
`+0x38 == 0x400` burst arm at `0x4608E..0x46097`.

## 5. What the mdb DM window is NOT (a retraction)

`read_dsp_sram_type=1` cells `0x100C` / `0x1038` / `0x103F` were reported as codec-dependent.
A time series with **nothing playing** shows them moving on their own (`0x1038` went
`0x1FAE -> 0x1F87` with no playback). They are free-running counters. `0x100E`/`0x100F`
(`0x202`/`0x1FF`) and `0x104C` (`0x400`) are constant. **The window does not discriminate
codecs and is not an instrument for this question.** `Decode frame count` (always 2560) and
`Decoder play state` (always STOP) are frozen constants for the same reason; only
`Decoder format` is reliable.

## 6. Status of the combined patch `f6594a6e`

The 3-byte patch (type tag 4→5 at `0x19861`, size divert → fallthrough at `0x46051/52`) was
built on the assumption that the block was at decoder entry. **The live measurement above
falsifies that assumption**: the DTS decoder already runs, and the ES output is already
complete and correctly framed. Flashing it would now be acting on a falsified model.

## 7. Next lever

The block is between a valid IEC61937 ES and the transmitter. The three concrete candidates,
in order of narrowness:

1. **`0x461A8 beqi r23,0x1`** — one instruction; the output stage leaves unless `+0x28 == 1`.
2. **`0x4625C bnei r23,0x1`** — one instruction; the packer is skipped unless `+0x2C == 1`.
3. Whatever arms the SPDIF TX (the codec word `AudioVars2+0x4F8`, still unread — it is
   kernel-only; the userspace route was measured dead).

Candidates 1 and 2 are reachable by patching the branch, are reversible, and target a site
that the live evidence now implicates directly. They are **untested**, and a patch here could
produce a silent or malformed stream rather than sound — the Pioneer remains the only oracle.

Raw logs: `C:\firmware_temp\R2_dts_clean.log`, `C:\firmware_temp\R2_ac3_clean.log`,
`C:\firmware_temp\aeon_validate\r57_npcm\DTS_clean_DUMP_ES1_from_DDR_00.bin`,
`...\AC3_clean_DUMP_ES1_from_DDR_01.bin`.

---

# ADDENDUM 2026-09-30 (evening) — verification of the Muse report, two corrections to THIS file, and the latch mechanism

All load-bearing claims of `MUSE_TASK_es_to_spdif_dead_end_20260930.md ## RESULTS` were
re-checked against the raw listing. Verdicts:

## Corrections to this file

1. **Candidate #2 (`0x4625C`) is dead as a lever — polarity inverted.** Confirmed byte-exact:
   the only `sw 0x2c(r10)` writers in the whole output-path range are `0x46013` (=0, the
   `0x4600C` copy arm) and `0x46087` (=r11=1, the `0x46080` arm). So `+0x2C==1` ⟺ the last
   pass took the `[r4+0x10]==1` arm, and **the IEC61937 packer runs on that arm only**. Forcing
   the branch at `0x4625C` would change nothing for DTS (already 1 on that arm) and corrupt the
   copy path (AC-3).
2. **The sync words in the DTS ES dump do NOT prove the DSP packer ran.** The source bitstream
   already carries IEC61937 framing (692 × `7FFE 8001` in the extracted track; the dump's
   1586 × 2012-byte frames ≈ 2.88 MB matches a pass-through of the framed source; Kodi's own
   log detects the stream by `syncword: 0x7ffe8001`). §2's "produces a fully valid IEC61937
   output" is reworded: the decoder **emits** the framed stream; whether the DSP re-packed it
   is unproven (R-TAP stands).
3. **CORRECTION (final, supersedes the rewording above): `dump_es` taps the decoder's INPUT
   elementary stream, not its output.** Proof: the AC-3 dump head is `0b 77 …` = the AC-3
   frame syncword **0x0B77** — raw AC-3 ES; and the mdb has a separate `dump_pcm`
   ("Dump Audio Decoder PCM") for the decoded output (the 9.6 MB DUMP_PCM1 files in the old
   archive are that). So the live differential reads: the decoder **receives** 2.88 MB of
   framed DTS (input side complete — already known from the R2 log: `dts m6 init ok`,
   `play=1`), and the TX emits 0 B. **The decoder's output toward SND has never been observed
   live.** The block spans the whole output stage, and the static chain (pump → latch → arms →
   packer → TX) is the primary evidence, not a confirmation.

## Confirmations

* `0x4121E beqi r23,0x1,0x41275` — verified; gates both `jal 0x45FD1` and `jal 0x4618E`.
  F_410A3 self-recursion at `0x41238 jal 0x410a3` — verified.
* B-latch arms — verified: `0x45FE0 lwz r23,0x0(r4)` → `0x45FEB beqi r23,0x1,0x4600C` (copy
  arm); `0x45FEE lwz r11,0x10(r4)` → `0x45FF1 beqi r11,0x1,0x46080`; else `0x45FF5 sw
  0x28(r3),r0` + return. **New detail: the prologue `0x45FDB movi r23,0x1 / 0x45FDD sw
  0x28(r3),r23` initializes `+0x28` to 1** — only the neither-arm path zeroes it. `+0x28==1`
  passes whenever any arm ran; candidate #4 matters only jointly with the B-latch.
* Pc builders — verified with the real preamble pair: `0x45D7F addi r24,r0,-0x78e` =
  **Pa `0xF872`**, `0x45D8E addi r24,r0,0x4e1f` = **Pb `0x4E1F`**; tags `0x45D83 ori
  r23,r0,0x10c`, `0x45DA4 movi r23,0xb` (DTS), `0x45DC3 ori r23,r0,0x20d`.
* Range gate `0x253C7` — verified, **refined beyond the report**: `0x2549D blesi r24,0x8,
  0x253CE` — the divert itself rejoins the pass path for Pc ≤ 8, and DTS (`0x0B − 0x0B = 0`)
  passes directly. The report's "an off-by-one here kills DTS only" is a hypothetical, not a
  finding; statically DTS passes this gate.
* Scanner `0x2417B` — verified: loop tests byte==`0x77` (flag) and exits on byte==`0x0B`,
  i.e. the `77 0B` adjacency.
* `0x36363636/0x5C5C5C5C` absent from realcode — confirmed (they were data).

## NEW — the R-J/#1 residual resolved: `[+0x4EE4]` is the pump's state latch

The full-corpus sweep for `0x4ee4` found the writers the report left unmapped:

```
0x3E827 mov r3,r11
0x3E829 jal 0x410A3            ; the pump
0x3E82D bnei r3,0x0,0x3E904    ; returned non-zero ->
0x3E831 movi r23,0x2           ;  else: [+0x4EE4] = 2
0x3E833 sw_0 0x4EE4(r10),r23,
   ...
0x3E917 movi r23,0x1           ;  success: [+0x4EE4] = 1
0x3E919 sw_0 0x4EE4(r10),r23,
0x3E91D movi r3,0x1 / j 0x3E7D2
```

Inside the pump, `0x4121E beqi r23,0x1,0x41275` runs the output stage only when the latch
reads 1 — i.e. only after a pass in which the pump returned non-zero. **So the gate word is
not a codec flag; it is the pump's progress latch.** The codec-conditionality the report
sought at #1 moves one step up: to whatever makes `F_410A3` return 0 under DTS.

Convergence: the pump's head (`0x410B5 lwz r3,-0x5EF8(r10)`) loads and immediately uses
(`jal 0x37D18`) the **same B-latch object** whose `[+0x0]`/`[+0x10]` the output stage's arms
test. Under DTS the B-latch reads 0 (prior work); every path that matters then hinges on
`[r4+0x10]==1`. **The narrowest open question is now: who writes the B-latch object's
`+0x10`, and is that write codec-conditional.** A full-corpus sweep of the latch writers is
the next mechanical step; no mdb route to the latch exists (the struct spans ±0x5EF8 ≈ 300 KB,
far outside the 64 K DM window).
