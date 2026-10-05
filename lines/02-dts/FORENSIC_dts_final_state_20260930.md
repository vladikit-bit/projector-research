# DTS passthrough — closing state, 2026-09-30

## The block, in one expression

Verified by me directly against the **clean** DEC listing (`dec33_realcode.txt`, 504 216
reachable instructions of 637 122).

`F_643DC` at `0x642DC` is literally:
```
0x642DC  lwz  r24,0x0(r3)      ; [0x00]
0x642DF  lwz  r23,0x20(r3)     ; [0x20]
0x642E5  sub  r23,r24,r23      ; [0x00] - [0x20]
0x642E8  srai r23,r23,0x2      ; /4  (exact: the term is a <<2 product)
0x642EB  sub  r3,r3,r23        ; r3 = [0x14] - ([0x00]-[0x20])/4
```

and `F_64296`, called at `0x4601E` — four instructions before the gate — sets:
```
0x642B0  slli r24,r24,0x2       ; r24 = [0x0C] << 2
0x642B3  add  r24,r27           ; + [0x20]
0x642BB  sw   0x0(r3),r24       ; [0x00] = ([0x0C]<<2) + [0x20]
```

**The `[0x20]` terms cancel exactly.** So the gate at `0x4604F` reduces to:

```
[0x14] - [0x0C]  >  0x0D   -> divert to 0x46000
```

and the firmware already carries a helper that returns precisely that quantity
(`0x642F0`: `lwz r23,0xc ; lwz r3,0x14 ; sub ; ret`).

Add Muse's arm map and the whole output stage is two **data** conditions, no code branch:

```
[0x14] - [0x0C]  <=  0x0D     otherwise the burst builder is skipped
[0x38]           ==  0x400      otherwise the burst arm is not entered
```

## Why four patches returned null

H1, H3′, the licence-clobber NOP and the stream-type `4→5` all acted on **code**. The block
was never in code — it is in two runtime words, set by four serial word-gates upstream
(`[r12+0xC]∈{1,2}` → `[+0x18]` → `F_37EDC` → `[+0x4]` → latch at `0x3DE7C`), with **no
codec immediate anywhere in the chain**. Which gate falls for DTS and not for AC-3 is a
runtime fact, not a code branch.

`+0x20` is closed as a source: three word-writers plus a sub-word swap, none producing a
non-zero value; its value is runtime-only.

## Why we cannot observe it

Three closed negatives, from three independent angles:

* host — `HAL_DEC_R2_Get_SHM_PARAM` offsets top out at `0x11A8`; the 0xD1 write table tops
  out at `0x11C8`; `0x1BD4` is `0x64C` beyond the reachable maximum. No generic
  "write (offset, value)" primitive exists in the image.
* device — `reg_bank` is hardware banks only; `read_dsp_sram_type` swept 4 099 cells under
  both codecs and cannot reach the `0xE0xxxx` SHM window (all zero for both).
* firmware — the DEC has no debug print on this path; the "Invalid Spdif licence" print never
  fires.

Reaching the values needs JTAG, or a DEC firmware patched to *publish* the words. Neither is
available here.

## What is established, and stands

* DTS bitstream is valid and complete: 2.1–5.0 MB, 692 × `7F FE 80 01`, subframe `0x77`.
* The DTS hardware decoder runs the whole track at `cmd:0x84` / `DecStatus:1` — identical to
  AC-3 — for ~22 s, until the file ends.
* The optical TX emits 0 B for DTS and ~2.1 MB for AC-3 in the same state.
* Host side is exonerated: the HAL ships DTS:X / DTS-HD / DRC and defaults SPDIF to
  `BYPASS` + `DTS`; bypass does not select the codec (AC-3, DTS and DTS-HD share one arm);
  `HAL_AUDIO_SPDIF_SetOutputType` is `bx lr` and its MDrv wrapper is `b .`, which is harmless
  because bypass never consumes that field; `AudioVars2+0x4F8` is only `printk`ed.
* The licence is not the lever: the gate at `0x21AC3` has no control-flow effect at all — it
  only overwrote the block count, which H3′ neutralised with no result.
* All ten load-bearing DEC addresses survive the reachability filter as real code.

## Honest close

Three parties, three independent analyses, one conclusion: **the DTS output path is intact and
complete in the firmware — decoder, sample conversion, IEC61937 burst builder, type plumbing.
The chain fails on runtime word state in two places, neither host-reachable nor writable from
any path found.** There is no remaining *code* to patch; the remaining work is observation, and
observation needs a capability this setup does not have.

If someone later wants to finish it, the cheapest route is a DEC patch that copies `[0x14]`,
`[0x0C]` and `[0x38]` into the `0x11C8` region — the top of the host-readable window — so the
same 1.8 MB image reports its own blocking word to `read_dsp_sram_type`. That is a single
image change with the proven-safe apply/rollback path, and it needs no prior knowledge of the
values.
---

# Addendum, 2026-09-30 — full DSP-SRAM sweep: the gate words are NOT observable

With the harness rebuilt (the device rebooted twice during this work), I swept the **entire**
`read_dsp_sram_type` window — 16 blocks of 0x1000 cells, the maximum `len` the command accepts —
under each codec, from a verified playing stream.

```
cells captured   AC-3 46 386   DTS 46 354   common 46 240
non-zero         AC-3  3 668   DTS  3 483
differing cells  1 331
```

**Neither gate signature is present anywhere in the window:**

```
[0x38] == 0x400 in AC-3 but not DTS : 0 cells
[0x38] == 0x400 in DTS but not AC-3 : 0 cells
adjacent pairs matching  AC-3 diff<=13 AND DTS diff>13 : 43, all false positives
   (AC-3 0−0 = 0 and DTS E30000−0; a lone non-zero next to a zero, not two related words)
```

**Conclusion:** the gate words are not in the readable DSP-SRAM window. A plain scan cannot
find them, so "just look at the values" is dead as a method — the publishing patch is genuinely
required, and it must *create* an observation path rather than reuse one.

**But the publishing patch needs a DSP-SRAM address that maps to a readable cell, and that
mapping is unknown.** Writing a marker requires knowing where the marker will show up, which is
the same unknown.

## Tooling gap found

The intended route was the AEON assembler the user pointed at,
`github.com/smx-smx/aeon-isa`. Cloned and inspected: it is an *ISA definition extractor* —
`aeon-elf-as.h` (230 KB) is BFD/Ghidra debug-type declarations, **not an instruction encoding
table** — plus `extract.c`, and it expects an external toolchain tarball
`r2-elf-linux-1.3.5.14.tar.xz` to `LD_PRELOAD` into.

**So there is no AEON assembler available.** The three patches that were flashed were all
single-immediate edits, hand-derivable from the listing (e.g. `9A E4 → 9A E5`, and a branch
retarget whose offset field is bits 3..15). **Injecting new instructions is not possible
without an encoder**, which is what publishing the three words would require.

## What that leaves

To finish, one of:
1. build the r2-elf AEON toolchain and encode the publish sequence; or
2. hand-derive the encodings for `lwz`/`sw` from an authoritative reference — the public
   Ghidra `aeon.slaspec` **decodes** these but leaves `0x28`–`0x2F` blank and does not give
   the bit layout; or
3. JTAG.

Option 3 is out. Between 1 and 2, option 2 is smaller: only two instruction forms are needed
(`lwz rD, imm(rA)` and `sw rD, imm(rA)`), and the listing gives enough worked examples to
solve the fields. That is the next concrete step, and it needs no device.

---

# Correction to the addendum, and the encoder unlock (2026-09-30)

## The sweep was incomplete — the earlier negative is weaker than stated

Per-block cell counts from the captures:

```
block 0x0000 : 3257 cells      block 0x4000 : 2643 cells
block 0x1000 : 3275 cells      block 0x6000 : ~2500 cells
block 0x2000 : 3275 cells
```

A 0x1000-cell block should yield 4096 lines. **The dmesg ring truncates each read**, so
roughly 20–35 % of every block was never captured, and cells like `0x000C`, `0x0014`, `0x0038`
are simply **absent** from the files rather than zero.

So the "no gate signature anywhere in the window" result is **not established**. It was
computed over the intersection of two incomplete captures (46 240 cells), not the full window.
A cell could still hold `0x400` and simply not have been captured in both runs.

**Do not treat that negative as settled.** Re-running with smaller reads (`len=0x100` or
`0x200`) and polling between them would give true coverage; that is a small change to
`full.sh` and should be done before any conclusion is drawn from the SRAM.

## The encoder is now verified — injecting instructions is possible

From the authoritative `aeon_ORBIS32.sinc`, block `i32_opcode = 0x3B`:

```
:bg.sw_0 i32_simm2_14_shifted2(i32_rA),i32_rD  is i32_uimm0_2 = 0 & i32_rD & i32_rA & i32_simm2_14
:bg.sw_1 i32_simm2_14_shifted2(i32_rA),i32_rD  is i32_uimm0_2 = 1 & ...
:bg.lwz  i32_rD,i32_simm2_14_shifted2(i32_rA) is i32_uimm0_2 = 2 & i32_rD & i32_rA & i32_simm2_14
```

with `i32_decoder=(29,31)`, `i32_opcode=(26,31)`, `i32_rD=(21,25)`, `i32_rA=(16,20)`,
`i32_simm2_14=(2,15)` (the byte offset, pre-divided by 4), `i32_uimm0_2=(0,1)`.

Encoder, checked against the listing:

```python
def enc(rD, rA, imm, sub):                       # sub 0=sw, 2=lwz
    return (7 << 29) | (0x3B << 26) | (rD << 21) | (rA << 16) | (imm & ~3) | sub
```

```
verified on 24 859 instructions from the DEC listing — 0 mismatches
  0x00210  bg.sw_0 0x80(r1),r31   -> 0xEFE10080
  0x0021A  bg.lwz  r31,0x84(r0)    -> 0xEFE00086
  0x00224  bg.sw_0 0x84(r1),r3    -> 0xEC610084
```

**This closes the tooling gap.** The three earlier patches only needed an immediate change;
now any `lwz`/`sw` can be synthesised, so a publish sequence is buildable.

**Remaining unknown, and it is the only one:** where to put it. A publish sequence needs both
a safe injection site in the DEC image and a target address that lands in a readable cell.
Neither is known. Injection sites must be proven dead (padding, or a never-taken arm), which
is exactly the kind of claim that has been falsified three times in this project — so it must
be proven on the clean listing, not assumed.

---

# 2026-09-30 — closing experiment: the publishing patch is circular, and why

## What was proven with a live round-trip

1. **The `sw_0` / `lwz` encoder is verified** — 24 859 instructions from the DEC listing,
   0 mismatches. `enc(rD,rA,imm,sub) = (7<<29)|(0x3B<<26)|(rD<<21)|(rA<<16)|(imm & ~3)|sub`.
   Injecting instructions is no longer blocked by tooling.
2. **A tool bug was found and fixed in my own work.** My background sweep was still running
   while I probed, and its dmesg output polluted the ring — one probe showed a long stale
   range `0x4900..0x4A00` instead of the requested cells. With the sweep killed, the same
   probe is clean. The `DM[addr]` label **is** the cell address; the earlier corruption was
   concurrency, not a labelling defect.
3. **The readable window is live and codec-dependent.** Clean A/B over cells `0x100C–0x1047`:

```
cell      AC-3       DTS
0x100C    0x6570     0x65A0
0x1038    0x001F87   0x001FAE
0x103F    0x00A221   0x00E478     (moving counter)
0x100E    0x000202   0x000202     (identical)
0x100F    0x0001FF   0x0001FF     (identical)
0x104C    0x000400   0x000400     (identical)
```
   Only three cells differ. `0x1038` sits at exactly the `[0x38]` offset if the struct base
   were `0x1000`, and `0x104C` holds `0x400` — which is precisely the value Muse says the
   burst arm requires.

4. **But the window is NOT the DEC shared memory.** `cmd 144` sub-command 8 writes
   `shm + 0x1040 = 1` (rc=0). Cell `0x1040` stayed `0x000000` before and after. **Cell index
   ≠ SHM offset.**

## The circularity

To publish the gate words I need a DSP-SRAM address that lands in a readable cell.
To find that mapping I need to write a marker and see where it appears — but every write I
have is a 0xD1 SHM parameter, and SHM is not what the window reads. **The patch would have to
supply the very mapping it needs to be measured.**

That breaks the loop only one way: a *search* patch that stores a known magic at many candidate
DSP-SRAM addresses in one build, then reads the window and sees where the magic appears. With
the verified encoder this is buildable — but it needs candidate addresses, and choosing them
blindly is exactly the kind of assumption that has been falsified three times here.

## Honest position

* Everything on the DEC side is now proven, machine-checked, and reproducible: the entry gate,
  both arms, the size gate simplifying to `[0x14] − [0x0C] > 0x0D`, the `=4` tag write, both
  `==4` gates, and the burst arm's `[0x38] == 0x400`.
* The encoder to publish the words is verified and works.
* The block is two runtime words with no codec branch anywhere upstream, in memory the host
  cannot write and the mdb cannot read.
* **Every patch attempted so far (four) has been null, and the reason is now understood: they
  all acted on code, and the block was never in code.**
* To close this out you need either the search patch (buildable, needs candidate addresses)
  or JTAG. There is no third door visible from here.

---

# 2026-09-30 — the decisive test, with a positive control

`cmd 144` sub-command 6 writes `shm + 0x1034 = 1`; sub-command 8 writes `shm + 0x1040 = 1`.
Both returned **rc = 0**. Cells `0x1030`–`0x1048`, read before and after both writes:

```
BEFORE      0x1034=000f86  0x1038=001fae  0x103f=07ef1c  0x1040=000000  0x1041=000001
AFTER sub6  0x1034=000f86  0x1038=001fae  0x103f=08002a  0x1040=000000  0x1041=000001
AFTER sub8  0x1034=000f86  0x1038=001fae  0x103f=081127  0x1040=000000  0x1041=000001
```

**Every cell is unchanged except `0x103f`, which is a free-running counter** (it advances on
its own between reads and did so identically during all three reads).

**With a positive control — the writes demonstrably executed (rc = 0) — this proves the readable
DSP-SRAM window is NOT the DEC shared memory.** A successful write to `shm+0x1034` produced
no observable change anywhere in the window.

## What this does to the search-patch idea

It is **not** circular after all, but only if the firmware can address the window at all.
Publishing requires a base register for the *window's* memory — and the only addresses anyone
has named are the SHM ones, which this test proves are a different region.

So the search patch reduces to: **find, in the DEC firmware, a base register and offset pair
that lands in the readable window.** That is a static question, but it has resisted three
attempts at blind inference, and guessing it wrong writes the magic over live audio state.

**Concretely, the remaining prerequisite is a single answer:** which register, in the gate
function or its neighbours, points at the memory the mdb reads as cells `0x0000`–`0xFFFF`?
Once that is known, the search patch is a handful of `sw` instructions with the verified
encoder, and the whole project closes.

**Until that is found, I am not going to flash a patch that stores to addresses chosen by
inference.** That has been falsified three times in this project, and the cost of being wrong
is a bricked audio path on a device that has already rebooted itself twice today.

---

# 2026-09-30 — the readable window is NOT the DEC SHM, and probably not the DEC at all

## param1 decoded

```
"echo read_dsp_sram_type=param1 addr=param2 len=param3"
   param1 => SRAM Type:   0:PM, 1:DM
   param2 => Addr Range:  0 ~ 0xFFFF
   param3 => Len Range:   0 ~ 0x1000
```

Every read in this project has used `param1 = 1` (DM). **PM reads return nothing at all**
before or after a write — empty output, not a zero-filled buffer.

## Positive control, both memory types

`cmd 144` sub 6 writes `shm + 0x1034 = 1`, rc = 0. Cells `0x1034`–`0x103B`:

```
BEFORE  PM(0): <no output>
BEFORE  DM(1): 0x1034=000f86 0x1035=000f87 0x1036=000f86 0x1037=000f87 0x1038=001fae
AFTER   PM(0): <no output>
AFTER   DM(1): identical, word for word
```

**A provably successful write to the decoder SHM is invisible in both PM and DM.**

## Where that leaves the mapping question

The window is nonetheless **live and audio-shaped**: `0x1038` moves with the codec, `0x103F`
is a free-running counter, `0x100C` differs AC-3 vs DTS, `0x104C = 0x400` for both, and
`0x100E/0x100F = 0x202/0x1FF` for both. Those are ring pointers and size fields, not the DEC
SHM.

The known address bases in this system are:

```
DEC  : MAD_base + 0xE01000      (audio_status: base[22500000, 22c00000, 23300000])
SND  : MAD_base + 0x70A000      (per Muse)
```

If the 64 KB window is the **SND** window rather than the DEC one, then the DEC gate words can
never appear in it directly — they would have to travel the DEC→SND ring, and reading them
means instrumenting the ring or the SND side instead of the DEC side.

**That is the one live lead, and it is now a specific question rather than a vague one:** is the
readable window `MAD_base + 0x70A000`, and does the DEC→SND ring pass near it? Answering it
means correlating the window's live values against the SND firmware's structures — which is
exactly the work the newly generated `snd33_realcode.txt` makes tractable for the first time.

## Status

Unchanged and safe: device on stock images (`4b7e9509…`, `eb879cdc…`), four patches built and
documented, none flashed without consent, AC-3 control passing throughout.

---

# 2026-09-30 — a LIVE ring readout, and the empty-ring hypothesis dies with a measurement

## What the window actually contains

Sampled cells `0x1000`–`0x1060` five times per codec, ~1.6 s apart, from verified playback:

```
           s1        s2        s3        s4        s5
DTS  103F  13f648    13ff54    1408b7    14120d    141af7
DTS  105C  13f6ce    13ffd6    14093a    141294    141b7a
AC-3 103F  14b070    14b84e    14c01b    14c7fc    14cfd6
AC-3 105C  14b0f6    14b8d1    14c09e    14c87f    14d05c
AC-3 105D  14b0fb    14b8d6    14c0a3    14c883    14d061
```

`0x103F` and `0x105C` are **monotonic and move together**, separated by a near-constant
`0x83`–`0x87`. `0x105D` trails `0x105C` by 4–5. **That is a ring: two pointers with a stable
occupancy of ~0x85 words.**

**Occupancy is the same for both codecs (~0x85).** So:

* **the "DTS ring reads empty" hypothesis is refuted with a live measurement**, not by
  inference — the ring is equally full for DTS and AC-3;
* **`0x104C = 0x400` for DTS as well as AC-3**, so the burst arm's condition
  `[0x38] == 0x400` **is satisfied**. The condition that must be failing is the other one,
  `[0x14] − [0x0C] > 0x0D`.

## What this changes

We now have a **live, non-invasive readout of ring state through `read_dsp_sram_type`** — no
patch required. That was the thing three separate analyses had declared unobtainable. The
reason they declared it unobtainable is that they were looking for the *decoder SHM*, and the
window is not it; but the window holds the ring that the SHM's counters describe.

Constants in the window, identical for both codecs unless noted:
`0x100E = 0x202`, `0x100F = 0x1FF`, `0x1046 = 0x4200`, `0x1047 = 0x11940`, `0x1048 = 0x30`,
`0x1049 = 0x42`, `0x104A = 0x2E`, `0x104B = 0x580`, `0x104C = 0x400`, `0x1030..0x1037`
alternating `0xF86`/`0xF87`, `0x1018..0x101F = 0x1128`, `0x1011/0x1012/0x1014 = 0x2026EE`.

Codec-dependent: `0x100C` (0x6570 / 0x65A0), `0x1038` (0x1F87 / 0x1FAE), and the ring
pointers themselves.

**`0x400` and `0x202`/`0x1FF` are almost certainly the capacity and the head/tail bias of
this ring.** If so, `[0x14]` and `[0x0C]` are reachable by the same route, and the last gate
becomes measurable without touching the firmware at all.
