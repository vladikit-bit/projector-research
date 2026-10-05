# DEC-side DTS IEC61937 output path — ROOT CAUSE LOCATED (static, proven)

**Date:** 2026-09-13 · **Image:** `dec_full.bin` = `aucode_adec_r2_MS12V22.bin` = `mst_codec_r2_MS12V22.bin`
(md5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`, 1,982,492 B). **file offset == fw addr.**
**Method:** Ghidra 12.1.2 headless, `-noanalysis`, real disassembler (DEC22/DEC23/DEC24/DEC26/DEC27/DEC28).
**Status:** READ-ONLY static analysis. Nothing patched. Nothing deployed.

---

## 0. The answer to "where exactly does it break"

The DEC image contains a **complete and correct DTS IEC61937 burst builder** — it is *not* missing,
*not* broken, and *not* license-gated at the syncwriter. It is **reached or not reached depending on a
single runtime field**:

> **`s->0x24`** (a 32-bit word in the output-descriptor struct, base register **r10**).
>
> * `s->0x24 == 0` → **DTS core syncword `0x7FFE8001` IS written** (`0x46259`…`0x4628B`)
> * `s->0x24 != 0` → branch to `0x4614C` → clears the field → goes to `0x460F3` → **no syncword at all**

This is the fork. Everything downstream of it (the "decoder is alive but the wire is empty" symptom)
is explained by this branch, not by any header field.

---

## 1. The function that decides — `SUB` at `0x46040`..`0x46160`

Verified instruction by instruction (DEC27). Load-bearing lines only:

```
046080  bn.lwz  r4,0x38(r4)              ; arg: r4 = src->0x38
046083  bg.jal  0x00064149               ; -> writes back a 0x28-byte descriptor
046087  bn.sw   0x2c(r10),r11            ; guard-A  = r11   <-- (see §3)
04608A  bg.j     0x00046016
...
0460A2  bt.mov  r3,r10
0460A4  bg.jal  0x00064296               ; r3 returned UNCHANGED (struct ptr, non-zero)
0460A8  bt.mov  r3,r10
0460AA  bg.jal  0x00064354               ; r3 = s->0x24          <-- THE VALUE
0460AE  bn.sw   0x30(r10),r3             ; guard-B = s->0x24
0460B1  bg.bnei r3,0x0,0x0004614c        ; *** s->0x24 != 0  ->  0x4614C  ***
0460B5  bn.lwz  r24,0x8(r10)
0460B8  bn.lwz  r23,0x38(r10)            ; r23 = s->0x38  (frame/sample size selector)
0460BB  bg.addi r26,r24,0x80
0460BF  bn.slli r25,r23,0x5
0460C2  bg.ble  r25,r26,0x000460f3       ; size too small -> 0x460F3 (no sync)
0460C6  bg.sfnei r23,0x200
0460CA  bn.bnf  0x00046156               ; ==0x200 -> 0x46156 (r4=1,r6=0x1000)
0460CD  bg.sfnei r23,0x400
0460D1  bn.bnf  0x00046172               ; ==0x400 -> 0x46172 (r4=1,r6=0x1000)
0460D4  bg.sfnei r23,0x800
0460D8  bn.bf   0x000460f3               ; !=0x800 -> 0x460F3 (no sync)
0460E9  bt.movi r4,0x2
0460EB  bg.ori  r6,r0,0x2000
0460EF  bg.jal  0x00045d07               ; builder, r4=2 (r6=0x2000)
```

### `0x4614C` (the "not DTS" arm) — verified bytes

raw `0x4614C`: `88 6a  e4 03 c4 02  e7 ff ff 43`
```
04614C  bt.mov  r3,r10
04614E  bg.jal  0x0006434f               ; SUB_0006434F:  bn.sw 0x24(r3),r0 ; jr r9   <- CLEARS s->0x24
046152  bg.j     0x000460f3               ; -> the "no syncword" tail
```

### why `0x64296` is a red herring
`SUB_00064296` (`0x64296`..`0x642C1`) reads `s->0x14`,`s->0x18`,`s->0xc`,`s->0x10`,`s->0x20`, writes
`s->0x8`,`s->0x0`,`s->0x4`, and does `bt.jr r9` **without ever touching r3**. Its "return value" is the
incoming `r3` (the struct pointer) — always non-zero. **The value that actually lands in guard-B comes
from the next call, `0x64354`.**

---

## 2. The syncword writer is intact and correct

Verified at `0x46250`..`0x4628C` (DEC22). This is the code the wire *should* be seeing:

```
046259  bn.lwz  r23,0x2c(r10)            ; s->0x2c  (guard-A / "is DTS" flag)
04625C  bg.bnei r23,0x1,0x000461ab       ; != 1  -> leave the DTS branch entirely
046260  bn.lwz  r23,0x30(r10)            ; s->0x30  (guard-B, set from s->0x24)
046263  bn.bnei r23,0x0,0x000462de       ; != 0  -> DTS-HD sync path
046266  bg.lhz  r23,0x16c(r10)           ; s->0x16c = data-type (0x0B=DTS, 0x10C=AC-3)
04626A  bg.lwz  r5,0xf8(r10)             ; r5 = out cursor A
04626E  bn.sfnei r23,0x0
046271  bg.lwz  r23,0xfc(r10)            ; r23 = out cursor B
046275  bn.cmov_1 r16,r8,r0
046278  bn.add   r24,r5,r16
04627B  bg.ori   r25,r0,0x7ffe            ; *** 0x7FFE ***
04627F  bn.sw    0x0(r24),r25
046282  bn.add   r24,r23,r16
046285  bg.ori   r25,r0,0x8001            ; *** 0x8001  ==> 0x7FFE8001 = DTS core sync ***
046289  bn.sw    0x0(r24),r25
04628C  bt.add   r5,r16
...
0462DE  bg.lwz   r5,0xf8(r10)            ; DTS-HD arm:
0462E2  bg.ori   r24,r0,0x1fff            ; 0x1FFF
0462EA  bn.sw    0x0(r5),r24
0462ED  bg.ori   r24,r0,0xe800            ; 0xE800  ==> 0x1FFFE800 = DTS-HD sync
0462F1  bn.sw    0x0(r23),r24
```

**Conclusion: the DTS syncwriter exists and is byte-correct. It is simply not being entered.**

---

## 3. The header builder — `0x45D07` (verified, and it is *not* the bug)

```
045D07  bn.srli r6,r6,0x2                 ; r6 = samples>>2
045D0A  bg.sfeqi r6,0x1000                ; 4096
045D0E  bg.ori  r23,r0,0x311              ; -> data-type 0x311
045D15  bg.sfgti r6,0x1000
045D19  bn.bnf  0x00045d62
045D1C  bg.sfeqi r6,0x2000                ; 8192  -> 0x411
045D27  bg.sfeqi r6,0x4000                ; 16384 -> 0x511
045D32  bn.ori  r23,r0,0x11               ; default 0x011
045D35  bn.beqi r4,0x1,0x00045d7f         ; r4==1 -> data-type 0x10C
045D38  bn.beqi r4,0x0,0x00045da0         ; r4==0 -> data-type 0x0B  <= DTS !
045D3B  bg.beqi r4,0x2,0x00045dbf
045D3F  bg.addi r24,r0,-0x78e             ; 0xF872 = Pa
045D50  bg.addi r24,r0,0x4e1f             ; 0x4E1F = Pb
045D4C  bg.sw_1 0x16c(r3),r24             ; slot +0x16D
045D54  bg.sh   0x170(r3),r5              ; slot +0x171 (Pd)
045D58  bg.sh   0x16c(r3),r24
045D5C  bg.sw_1 0x170(r3),r23             ; slot +0x173 (data-type)
045DA0  bg.addi r24,r0,-0x78e             ; r4==0 arm: Pa=0xF872
045DA4  bt.movi r23,0xb                   ; *** data-type 0x0B = IEC61937 DTS ***
```

So the builder knows how to emit DTS (`r4==0` → data-type `0x0B`) and AC-3 (`r4==1` → `0x10C`).
The callers set `r4` from `0x4635C`..`0x4618E` (`r4=0` and `r4=1`), and one path uses `r4=2`.

**This matches `FINDING_wire_capture.md` §4 addresses exactly.** That finding is confirmed correct.

---

## 4. Why the earlier "0 byte-pattern hits" search was a false alarm

A raw byte search for `7FFE8001`, `1FFFE800`, `58642520`, `80017FFE`, `FE7F0180`, `00E8FF1F` returns
**0 hits in both `dec_full.bin` and `snd_full.bin`**. This is *expected*: AEON builds these constants
from **split instruction immediates** (`bg.ori r25,r0,0x7ffe` + `bg.ori r25,r0,0x8001` at `0x4627B` /
`0x46285`), so the 32-bit value never appears as contiguous bytes. Do **not** conclude "the DTS
mechanism is absent" from a byte scan — use the disassembly.

---

## 5. Guard fields, definitions (all verified)

| field | read at | written at | meaning |
|---|---|---|---|
| `s->0x2c` | `0x46259` (must == 1 to stay in DTS) | `0x46087` (from `SUB_00064149`), `0x21D68`, … | "this frame is a DTS frame" |
| `s->0x30` | `0x46260` (must == 0 for DTS core) | `0x460AE` (from `SUB_00064354` = `s->0x24`), `0x14048`, `0x1413E` | DTS-HD / extension selector |
| `s->0x24` | `0x64354` | `0x6434F` (`sw 0x24(r3),r0`), `0x460F5`… | **the actual fork input** |
| `s->0x38` | `0x460B8`, `0x46080` | `0x46097` (`r11<<5`) | frame/sample-size class (`0x200`/`0x400`/`0x800`) |
| `s->0xf8`, `s->0xfc` | `0x46222/6A/A0/DE` | many | output write cursors (IEC61937 buffer) |
| `s->0x16c…0x172` | `0x46266`,`0x462A8…` | `0x45D4C…` | Pa/Pb/Pd/data-type header slots |

---

## 6. Where the search now stands

**New, precise next target (not yet investigated):**

> Determine **what sets `s->0x24` to a non-zero value** in the DTS playback case.
> Candidate writers found by full-image scan (DEC24) with base **r10**:
> `0x23A7E`, `0x28F3A`, `0x2914E`, `0x291FC`, `0x2D3C4`, `0x2D43C`, `0x2D5D4`, `0x2EFD6`,
> `0x3C49C`, `0x460AE`, `0x1413E`, `0x14048`, `0x10AD0`, `0x157CA`.
> `0x460AE` is the one inside the decision function itself; the others are upstream state writers.
> The one to chase is whichever these writes `s->0x24` **from the codec/stream negotiation** when the
> input is DTS.

**Two concrete, *new* patch hypotheses (neither tried yet):**

* **H1** — at `0x460B1`, force the fall-through so `s->0x24` is ignored:
  `d0 60 04 de` (`bg.bnei r3,0,→0x4614C`) → **`d0 60 00 23`** (`bg.011i` unconditional goto to
  `0x460B5`; `u3=3`, `rel = 0x460B5 − 0x460B1 = 4`). Diff = 2 bytes at `0x460B2`/`0x460B3`.
  *Verified by round-trip:* stock decodes to rD=3, imm=0, rel=155 (`→0x4614C`), u3=6; re-encoding the
  stock word reproduces `0xd06004de` exactly.
  *Caveat:* this only helps if the DTS path is entered at all; if `s->0x2c != 1` is the real gate the
  syncwriter is still skipped at `0x4625C`.
* **H2** — at `0x4625C`, relax the `s->0x2c == 1` requirement:
  `d2 e1 fa 7e` (`bg.bnei r23,0x1,→0x461AB`) → **`d2 e0 00 23`** (`bg.011i` unconditional goto to
  `0x46260`; rD=23, imm=0 — **the immediate MUST be zeroed**, `rel = 0x46260 − 0x4625C = 4`).
  *Verified by round-trip:* stock decodes to rD=23, imm=1, rel=8015 (`→0x461AB`), u3=6; re-encoding the
  stock word reproduces `0xd2e1fa7e` exactly.
  *Caveat:* same class as the already-inert L2 gate patch; must be tested on the **wire**.

**Hard prerequisite for any test (learned at cost — SKILL §14):** `adb root` + verify `uid=0(root)`,
a **positive control (AC-3) in the same session**, and a real reboot verified via `/proc/uptime`.
A 0-byte capture without root is INVALID.

---

## 7. Reproduce

```bash
JAVA_HOME=.../jdk-21 APPDATA=... HOME=... \
  analyzeHeadless.bat <decproject> decimg -process dec_full.bin -noanalysis \
  -scriptPath scripts -postScript DEC22_DtsOutTrace.java      # 0x45CD0..0x46380
  # DEC23_BuilderCallers / DEC24_GuardWriters / DEC26_StructMap / DEC27_DtsDecision / DEC28_DecisionFn
```
Outputs: `dec_work/dec22_out.txt` … `dec_work/dec28_out.txt`.

## 8. Artifacts

| path | content |
|---|---|
| `dec_work/dec22_out.txt` … `dec28_out.txt` | the authoritative listings used above |
| `scripts/DEC22_DtsOutTrace.java` … `DEC28_DecisionFn.java` | the tracers |
| `dec_work/dec_full.bin` | the DEC image (`4b7e9509…`) |
| this file | the finding |
