# FINDING — DEC-side DTS output path: the real blocker (SUPERSEDES the `s->0x24` fork reading)

**Status:** static analysis complete and cross-verified. **No patch built, no device touched.**
**Supersedes:** `FINDING_dec_dts_output_rootcause.md` §1–§3 and `REPORT_final_dts_patch_experiment.md` §10
(the "`s->0x24` fork at `0x460B1` is the DTS blocker" reading). That reading was a **misattribution**
of a generic bitstream-reader field to a DTS-specific gate.
**Method:** `dec_work/dec32_clean.txt` (Ghidra-authoritative: 637,122 instructions, real lengths + real bytes),
decoded with the `i24_opcode=0x03` load/store encoding from `aeon.slaspec`.

---

## 0. The answer in one paragraph

The DEC image contains a **complete, correct IEC61937 header builder** at `0x45D07` that already knows how to
emit the DTS header (`data-type 0x0B`) and the AC-3 header (`0x10C`). It is called from **exactly one place**
in the DTS/AC-3 output path: `0x460EF`. Whether that call happens is decided 30 bytes earlier by
**`0x460C2 bg.ble r25,r26,→0x460F3`**, where `r25 = selector<<5` and `r26 = payload_len + 0x80`.
**If that test fails, the function branches to `0x460F3` and no IEC61937 header is written at all.**
The `selector` and `payload_len` come from a **fixed-capacity stream-entry pool of exactly 2 entries**
(`0x80C` bytes each, base `+0x6E24`, count at `+0x7E3C`), and a per-entry "configured" flag at
**`entry+0x808`** which is set only by `0x46D81 sw 0x762C(r25),1` — and that function refuses to add an entry
when **`count > 1`** (`0x46D2C bgtui r23,0x1`). The header builder *requires* `entry+0x808 == 1`
(`0x45EE9 lwz r23,0x808(r23)` / `beqi r23,1,→0x45FBF`).

**So: the DTS wire is empty because the DTS stream entry is never registered as "configured"**
(either the pool is full with the AC-3 entry, or the DTS path never calls the registrar), and the
frame-geometry test at `0x460C2` then skips `0x45D07` entirely.

---

## 1. The single call site of the IEC61937 header builder

`0x45D07` (the per-codec header builder) has these inbound `jal`s inside `0x45xxx`/`0x46xxx`:

| caller | `r4` (codec sel) | `r6` (rate sel) | resulting data-type |
|---|---|---|---|
| `0x45FC9` | (varies) | (varies) | — (from the table walker) |
| **`0x460EF`** | **`movi r4,0x2`** | `0x2000` | **`0x20D`** |
| **`0x4616A`** | **`movi r4,0x0`** | `0x800` | **`0x0B` = DTS** |
| **`0x46186`** | **`movi r4,0x1`** | `0x1000` | **`0x10C` = AC-3** |

`0x45D07` itself (fully intact):

```
045d07  srli   r6,r6,0x2
045d0a  sfeqi  r6,0x1000
045d0e  ori    r23,r0,0x311
045d12  bf     →0x45d35
045d15  sfgti  r6,0x1000
045d19  bnf    →0x45d62
045d1c  sfeqi  r6,0x2000
045d20  ori    r23,r0,0x411
045d24  bf     →0x45d35
045d27  sfeqi  r6,0x4000
045d2b  ori    r23,r0,0x511
045d2f  bf     →0x45d35
045d32  ori    r23,r0,0x11
045d35  beqi   r4,0x1,→0x45d7f          ; AC-3 arm
045d38  beqi   r4,0x0,→0x45da0          ; DTS arm
045d3b  beqi   r4,0x2,→0x45dbf          ; third codec
045d3f  addi   r24,r0,-0x78e            ; = 0xF872  Pa
045d46  slli   r5,r5,0x3
045d49  cmov_0 r23,r23,r0
045d4c  sw_1   0x16c(r3),r24
045d50  addi   r24,r0,0x4e1f            ; = 0x4E1F  Pb
045d54  sh     0x170(r3),r5
045d58  sh     0x16c(r3),r24
045d5c  sw_1   0x170(r3),r23
045d60  jr     r9
...
045d7f  addi   r24,r0,-0x78e            ; AC-3
045d83  ori    r23,r0,0x10c
...
045da0  addi   r24,r0,-0x78e            ; DTS
045da4  movi   r23,0xb                  ; <-- data-type 0x0B = DTS
...
045dbf  addi   r24,r0,-0x78e
045dc3  ori    r23,r0,0x20d             ; third codec
```

**`bg.addi rN,r0,-0x78e` IS the literal `0xF872`.** This is why a byte/text search for `"f872"` in the
DEC image found only 8 unrelated `addi`-with-displacement false hits: the Pa/Pb constants are *always*
materialised as negative `addi` immediates, never as `ori rN,r0,0xf872`.

### 1a. All Pa/Pb emitter clusters in the DEC image (the complete set)

| # | Pa site | Pb site | notes |
|---|---|---|---|
| A | `0x24C0C` | `0x24C19` | writes `sh_1 0x0(r4)`, `sh_2 0x2(r4)`, then `jal 0x13602` with `r5=0xC00` (emit 0xC00 B) |
| B | `0x24C85` | `0x24C8E` | sibling of A |
| C | `0x2645E` | `0x2646E` | gated by `0x414(r10)==1` (`0x26457 beqi`) and `0x418(r10)` |
| D | `0x2ABAC` | `0x2ABB3` | `sh_2 0x6(r12),r24` with `r24=r23<<3`; `ori r5,r0,0xFF8` |
| E | `0x2B5FC` | `0x2B603` | `jal 0x87394` first, `r6=0xA00` |
| **F** | **`0x45D3F/45D7F/45DA0/45DBF`** | **`0x45D50/45D8E/45DAD/45DCE`** | **the per-codec builder `0x45D07` — the one in the DTS output path** |

Clusters A–E write into `0x54(r10)`/`0xC0(r3)`/`0xC0(r11)` output descriptors; cluster F writes into
`0x16C(r3)..0x173(r3)` of the descriptor passed in `r3`. **Only cluster F has a DTS arm that is reached from
the same function that computes the DTS frame geometry (`0x460B5`–`0x460F3`).**

---

## 2. THE GATE — `0x460C2` skips the header builder

```
0460b5  lwz  r24,0x8(r10)      ; r24 = payload / burst length
0460b8  lwz  r23,0x38(r10)     ; r23 = frame-type selector  (0x200 / 0x400 / 0x800)
0460bb  addi r26,r24,0x80      ; r26 = payload + 0x80
0460bf  slli r25,r23,0x5       ; r25 = selector << 5
0460c2  ble  r25,r26,→0x460f3  ; *** GATE *** if (sel<<5) <= payload+0x80  -> NO HEADER
0460c6  sfnei r23,0x200
0460ca  bnf  →0x46156          ; sel == 0x200 -> r4=0 -> DTS header (0x0B)
0460cd  sfnei r23,0x400
0460d1  bnf  →0x46172          ; sel == 0x400 -> r4=1 -> AC-3 header (0x10C)
0460d4  sfnei r23,0x800
0460d8  bf   →0x460f3          ; sel == 0x800 -> ALSO SKIP
0460db  addi r23,r24,0x7       ; round payload up
0460de  sflesi r24,-0x1
0460e1  cmov_0 r24,r23,r24
0460e4  srai r5,r24,0x3        ; r5 = payload/8  (burst length in words)
0460e7  mov  r3,r10
0460e9  movi r4,0x2
0460eb  ori  r6,r0,0x2000
0460ef  jal  0x45d07           ; <-- the header builder
0460f3  mov  r3,r10            ; (fall-through target)
```

**`0x460C2` is the last instruction that can prevent a DTS header from being emitted.** It is a *geometry*
test: it requires the descriptor's declared selector, scaled by 32, to exceed the payload length plus 128.
For a DTS burst (`payload` is a full 512–4096-sample core frame, i.e. much larger than a 0x200-class
selector<<5) the condition is **false** → branch to `0x460F3` → **header never written**.

> **This is the concrete form of "DTS is formed in the codec but never reaches SPDIF."**

The second skip (`0x460D8 bf →0x460F3`, taken when `sel == 0x800`) is the PCM/undifferentiated case.

---

## 3. Where `selector` (`0x38`) and `payload_len` (`0x8`) come from

Both are fields of the **per-stream entry** in a fixed-capacity pool:

```
0x465F6  lwz  r24,0x7e3c(r3)     ; count
0x465FC  ble  r4,r24,→0x46607    ; if (req_index >= count) -> return 0
0x46600  sw   0x0(r5),r0         ; *out = NULL
0x46605  jr   r9
0x46607  muli r4,r4,0x80C        ; *** ENTRY STRIDE = 0x80C ***
0x4660D  add  r23,r4
0x4660F  addi r23,r23,0x6e24     ; *** TABLE BASE = +0x6E24 ***
0x46613  sw   0x0(r5),r23        ; *out = &entry[idx]
0x46618  sw   0x7e3c(r3),r0      ; reset: count = 0        (this is the free-all routine)
```

and the registering routine:

```
0x46D28  lwz   r23,0x7e3c(r10)   ; count
0x46D2C  bgtui r23,0x1,→0x46cbc  ; *** CAPACITY GATE: count > 1 -> refuse ***
0x46D2F  lwz   r3,0x4(r10)
0x46D32  addi  r4,r1,0x10
0x46D35  jal   0x53804
0x46D39  lwz   r23,0x7e3c(r10)
0x46D3F  muli  r23,r23,0x80c
0x46D43  lwz   r4,0x6e20(r10)
0x46D47  add   r23,r10
0x46D49  sw    0x6e28(r23),r5    ; entry[idx].0x04 = r5
0x46D4D..0x46D59                 ; entry[idx].0x00 = r24
0x46D67  addi  r3,r3,0x6e2c
0x46D6B  jal   0x130e0d          ; init entry[idx].0x08 ...
0x46D75  muli  r25,r24,0x80c
0x46D7B  add   r25,r10
0x46D7D  lwz   r23,0x61cc(r10)
0x46D81  sw    0x762c(r25),r26   ; *** entry[idx].0x808 = 1 ***  (r26 = 1)
0x46D85  sw    0x7e3c(r10),r24   ; count += 1
```

**`entry+0x808`** (computed: `0x762C - 0x6E24 = 0x808`) is exactly the flag the header builder tests:

```
0x45EE9  lwz  r23,0x808(r23)
0x45EED  beqi r23,0x1,→0x45FBF   ; only if ==1 does it take the "emit" arm
```

and the **DTS/HD "DTS " ASCII ident** is also in this same function family:

```
0x45F47..0x45F44  16-bit byte-swap loop on 0xf8(r11)/0xfc(r11)
0x45F4D  addi r5,r4,-0x2         ; payload_len - 2
0x45F7F..0x45F82  lh r3,0x0(r27) ; lh r8,0x2(r27)  where r27 = r12 + 0x6E2C + idx*0x80C
```

---

## 4. Why this supersedes the `s->0x24` reading

| | old §10 reading | this finding |
|---|---|---|
| target | `0x460B1 bg.bnei r3,0,→0x4614C` (`s->0x24`) | **`0x460C2 bg.ble r25,r26,→0x460F3`** |
| `0x24` meaning | DTS-enable / licence gate | **a field of the bitstream-reader context** (`0x64296` = bit reader re-seek; `0x6434F` = `sw 0x24(r3),0`; `0x64354` = `lwz r3,0x24(r3)`) |
| `0x4614C` arm | "give up, clear `0x24`" | **a sample-rate-table dispatcher** — `0x7D00`=32000, `0x3E80`=16000, `0x1F40`=8000, `0x5622`=22050, `0x2B11`=11025, each `ori r23,..` then `j 0x46069` which stores to `0x80/0x78/0x7C(r10)` |
| `0x45D07` | "header builder, DTS arm `r4==0` → `0x0B`" | **confirmed correct** — keep |
| `0x4627B/0x46285` (`0x7FFE8001`) | DTS core syncwriter | **confirmed real**, but it is a *write of the sync into the payload buffer* — it is downstream of the header builder, and it too is unreachable when `0x460C2` skips |

`0x24(r10)` and `0x24(r15)` are **different structures**: `r10` is the local descriptor set at
`0x45FE9 bt.mov r10,r3`; `r15` is the DEC stream-manager context (`0x33AA5 sw 0x24(r15),r19` on the
DTS-HD arm, `0x336A2`/`0x33890` on the register arms). Neither is the DTS blocker.

The DTS **syncword classifier** in the stream state machine (`0x339F1`–`0x33BC8`) is intact and does
recognise all four DTS word types — converging on `0x33A01`:

```
033b92  movhi r25,0x7ffe / ori r25,r25,0x8001   ; 0x7FFE8001  DTS core
033b9a  beq   r23,r25,→0x33a01
033b9e  movhi_2 r26,0x1fff / ori r26,r26,0xe800 ; 0x1FFFE800  DTS-HD
033ba5  beq   r23,r26,→0x33a01
033ba9  movhi r27,0xff1f / ori r27,r27,0xe8     ; 0xFF1FE8
033bb0  beq   r23,r27,→0x33a01
033bb4  movhi r24,0x5864 / ori r24,r24,0x2520   ; 0x58642520  "DTS "
033bbc  beq   r23,r24,→0x33bcc
033bc0  movhi r25,0x6458 / ori r25,r25,0x2025   ; 0x64582025
033bc8  bne   r23,r25,→0x335cd                  ; none -> exit
```

and the DTS **input syncword normaliser** at `0x28A33`–`0x28AB7` is also intact:

```
028a33  lwz   r23,0x0(r4)
028a36  movhi_2 r25,0x180 / ori r25,r25,0xfe7f  ; 0x180FE7F  (DTS-HD byte-swapped)
028a3d  sfeq  r23,r25
028a46  bf    →0x28a88                          ; -> 4-byte byte-swap loop
028a49  movhi_2 r27,0xe8 / ori r27,r27,0xff1f   ; 0xE8FF1F
028a50  beq   r23,r27,→0x28a88
028a54  movhi r27,0x7ffe / ori r27,r27,0x8001   ; 0x7FFE8001  DTS core
028a5c  bne   r23,r27,→0x28ab0
028a60  srai  r25,r25,0x2
028a63  blesi r25,0,→0x28a86
028a66  opcode_2B r25
028a6a  lbz   r23,0x0(r24) / sbz 0x2(r26),r23   ; byte swap
028a70  lbz   r23,0x1(r24) / sbz 0x3(r26),r23
028a76  lbz   r23,0x2(r24) / sbz 0x0(r26),r23
028a7c  lbz   r23,0x3(r24) / sbz 0x1(r26),r23
```

**So recognition and normalisation both work. Only the *emission* is gated — by `0x460C2`.**

---

## 5. Candidate patches (NOT yet built, NOT yet tested)

All candidates are 2-byte in-place edits of the **new versioned artifact**; never overwrite stock
`dec_full.bin` (md5 `4b7e9509b4358fd3a130bd4d3b9cbe0a`).

### P1 — force the geometry gate to always pass (`0x460C2`)
```
0x460C2  stock: d7 3a 01 89   (bg.ble r25,r26,0x460f3)
proposed:       d7 3a 00 23   (bg.011i 0x460C6 ; rD=25, imm=0, u3=3, rel=4)
```
Encoded per `aeon.slaspec`: `w = (0x34<<26)|(rD<<21)|(imm5<<16)|(((target-pc)&0x1FFF)<<3)|u3`.
`target-pc = 0x460C6-0x460C2 = 4`, `rel=(4>>2)&0x1FFF = 1`, `w=(0x34<<26)|(25<<21)|(0<<16)|(1<<3)|3`.
Round-trip check against the stock word (`rD=25,imm=1,rel=0x31,u3=2` → `d73a0189`) passes trivially.
**Effect:** always falls through to the selector dispatch; `sel==0x200` → DTS header `0x0B` emitted.
**Risk:** the geometry test exists for a reason (it prevents emitting a header when the buffer is too small
for the declared frame). Forcing it may produce a malformed burst for the non-DTS cases. Mitigation: it
only *adds* a header; the payload copy is separate.

### P2 — force the DTS arm at the selector dispatch (`0x460C6`)
```
0x460C6  stock: c2 e0 40 04   (bg.sfnei r23,0x200)
```
Not attractive: replacing with an unconditional goto makes **every** stream take the DTS `0x0B` header.
Do **not** do this.

### P3 — make the third selector case (`0x460D8`) fall through to the default (`r4=2`)
```
0x460D8  stock: 20 00 6d      (bn.bf 0x460f3)
```
Also not attractive: `sel==0x800` is the PCM case; falling through would emit `0x20D` headers on PCM.

### P4 — raise the entry-pool capacity (`0x46D2C`)
```
0x46D2C  stock: 26 e6 43      (bn.bgtui r23,0x1,0x46cbc)
```
Changing `0x1` to a larger value would let more entries register, but the **pool is physically only 2
entries** (`(0x7E3C-0x6E24)/0x80C = 2`), so a 3rd entry would overlap the count field. **Reject.**

**Conclusion: P1 (`0x460C2`) is the only candidate that is small, targeted, reversible, and
directly on the DTS-specific path.** But before building it I must answer the one open question below.

---

## 5b. THE COMPLETE CHAIN, END TO END (closed loop)

Both writers of `0x28` are inside `0x45FD1`, and they *are* the fork:

```
045fdb  movi r23,0x1
045fdd  sw   0x28(r3),r23        ; *** desc->0x28 = 1 ***  ("prepared")
045fe0  lwz  r23,0x0(r4)         ; r23 = params->0x0        <-- DISCRIMINATOR #1
045fe3  sw   0x34(r3),r0
045fe6  sw   0x38(r3),r0         ; *** desc->0x38 = 0 ***
045fe9  mov  r10,r3              ; r10 = the descriptor
045feb  beqi r23,0x1,→0x4600c    ; params->0x0 == 1 ?
045fee  lwz  r11,0x10(r4)        ; r11 = params->0x10       <-- DISCRIMINATOR #2
045ff1  beqi r11,0x1,→0x46080
045ff5  sw   0x28(r3),r0         ; *** desc->0x28 = 0 ***   (the "unprepared" arm)
```

and the caller:

```
041275  movhi r23,0x1
041277  ori   r23,r23,0x4ccc
04127b  add   r10,r23            ; r10 = base + 0x14CCC
04127d  lwz   r4,-0x5ef8(r11)    ; r4 = *(r11 - 0x5EF8)   <-- the MODE/PARAMS STRUCT
041281  mov   r3,r10             ; r3 = the output descriptor
041283  jal   0x45fd1            ; prepare(descriptor, params)
041287  mov   r3,r10
041289  jal   0x4618e            ; then the pack/emit step
04128d  bnei  r12,0x1,→0x411c9
```
reached from `0x4121E beqi r23,0x1,→0x41275` with `r23 = 0x4EE4(r11)`.
`0x45FD1` has **exactly one caller in the whole image: `0x41283`.**

### The decision table

| `params->0x0` | `params->0x10` | path | `desc->0x38` | header? |
|---|---|---|---|---|
| `== 1` | — | `0x4600C` → `0x46016` → `0x4608E` | **`(idx+1)<<5`** | **emitted** |
| `!= 1` | `== 1` | `0x46080` → `0x46083` → `0x46087` (`desc->0x2c = r11`) | stays `0` | not on the `0x200` selector arm |
| `!= 1` | `!= 1` | `0x45FF5` (`desc->0x28 = 0`) → `0x46013` (`desc->0x2c = 0`) | **stays `0`** | **NEVER** |

Third row ⇒ `r25 = 0<<5 = 0` and `r26 = payload+0x80 > 0`, so
**`0x460C2 bg.ble r25,r26,→0x460F3` is taken deterministically and `0x45D07` is never called.**

### Why this is DTS-specific

`params->0x0` comes from `*(r11 - 0x5EF8)` where `r11` indexes the per-stream control block.
AC-3's negotiation sets it to 1 (row 1 → header `0x10C` emitted); DTS's negotiation does not
(rows 2/3 → no header). **The DTS stream is never "prepared", so its header builder is skipped.**

### Corrected patch preference

| option | edit | assessment |
|---|---|---|
| **Q1 — force the DTS descriptor to take row 1** | at `0x45FEB` make the `params->0x0==1` test unconditional | **best** — restores the *intended* path; does not fabricate geometry; only affects descriptors that call `0x45FD1` |
| **Q2 — stop `0x45FF5` from zeroing `0x28`** | nop/replace `0x45FF5 sw 0x28(r3),r0` | also good, but `0x46013` still zeroes `0x2c` and `0x38` was already zeroed at `0x45FE6` — so this alone is **insufficient**; `0x38` must be set, which only row 1 does |
| **Q3 — force the geometry gate `0x460C2`** | `d7 3a 01 89` → `d7 3a 00 23` | works, but patches the *symptom*; and it would emit a header with a `value/8` burst-length computed from a possibly-wrong `payload_len` |
| ~~Q4 — raise entry-pool capacity~~ | `0x46D2C` | **reject** — pool is physically 2 entries (`(0x7E3C-0x6E24)/0x80C = 2`) |

**Q1 is the correct fix: it restores the codec's own intended behaviour instead of overriding a
sanity check.**

### Q1 encoding

`0x45FEB` stock word: `22 e4 84` = `bn.beqi r23,0x1,0x0004600c`
(`i24_opcode=0x08`, `i24_uimm0_2=0`, `rD=23`, `rA=4`).

**The `bn.*` 24-bit family has NO unconditional-goto sub-op.** From `aeon_ORBIS32.sinc`
(lines 224–262, all marked "Verified on hw"):

```
with : i24_opcode = 0x08 {
    :bn.beqi rD,rA,rel  is i24_uimm0_2 = 0   -> if (rD == rA) goto rel
    :bn.bf   rel        is i24_uimm0_2 = 1   -> if (flag)  goto rel
    :bn.bnei rD,rA,rel  is i24_uimm0_2 = 2   -> if (rD != rA) goto rel
    :bn.bnf  rel        is i24_uimm0_2 = 3   -> if (!flag) goto rel
}
with : i24_opcode = 0x09 {
    :bn.blesi / bn.bleui / bn.bgtsi / bn.bgtui   (uimm0_2 = 0..3)
}
:bn.j rel is i24_opcode = 0x0B { goto rel; }        <-- the ONLY unconditional form
```

So a Q1 "make it unconditional in place" edit is **not available in the same instruction slot**.
Two workable forms:

- **Q1a — 4-byte re-encode to `bn.j`** (opcode `0x0B`, `i24_rel_iK18` in bits 0..17):
  `pc=0x45FEB`, `target=0x4600C`, `rel = target - pc = 0x21`.
  `word = (0x0B << 18) | (0x21 & 0x3FFFF)` → **`0x0B0021`** = bytes `02 C0 00 21` (24-bit).
  Requires length parity (4→3 bytes changes all following alignment) — **NOT viable**.
- **Q1b — swap the branch for the *other* condition** so the descriptor takes row 1 when
  `params->0x0 != 1`. Stock `bn.beqi r23,0x1,rel` with `r23 = params->0x0`:
  rewrite as `bn.bnei r23,0x1,rel'` targeting the *other* arm → effectively inverts the test.
  Also 3 bytes, in-place, no alignment change. **This is viable** but changes which arm is
  "normal", so it must be reasoned through per-caller. Deferred.

### H1 / H2 encoding — VERIFIED (these use the 32-bit `bg.*` group, which DOES have an unconditional form)

From `aeon_ORBIS32.sinc` line 479–492, `i32_opcode = 0x34`:
```
uimm0_3 = 0 bg.blesi  1 bg.bleui  2 bg.beqi  [3 bg.011i = UNCONDITIONAL GOTO]
uimm0_3 = 4 bg.bgtsi  5 bg.bgtui  6 bg.bnei  [7 bg.b111i = UNCONDITIONAL GOTO]
```
**`bg.011i` is only in the `0x34` group. `bg.bf` (`uimm0_3 = 3`) is in the `0x35` group and is a
FLAG test, not an unconditional goto.** Both H1 and H2 are in the `0x34` group, so the edit is sound.

Verified by `scripts/aeon_verify_patch_words.py` (faithful re-encode from the slaspec fields,
including a negative-displacement stock check):

```
0x460B1  stock 0xD06004DE  op=0x34 u3=6 rD=3  imm=0x0 rel=+155  -> bg.bnei  target 0x4614C
0x460B1  patch 0xD0600023  op=0x34 u3=3 rD=3  imm=0x0 rel=+4    -> bg.011i  target 0x460B5
0x4625C  stock 0xD2E1FA7E  op=0x34 u3=6 rD=23 imm=0x1 rel=-177  -> bg.bnei  target 0x461AB
0x4625C  patch 0xD2E00023  op=0x34 u3=3 rD=23 imm=0x0 rel=+4    -> bg.011i  target 0x46260
```

Both patch words differ from stock **only in the low byte**, so `rD`/`imm16`/`rel` are untouched and
the goto lands exactly on stock's fall-through target. **This is the minimal possible edit: 1 byte each.**

### Updated recommendation

| # | edit | bytes | assessment |
|---|---|---|---|
| **H1** | `0x460B1`: `d0 60 04 de` → `d0 60 00 23` (2 bytes: `0x460B3`,`0x460B4`) | **2** | forces the "rates already known" path, skipping the `s->0x24` re-zero. **Smallest safe first probe.** |
| **H2** | `0x4625C`: `d2 e1 fa 7e` → `d2 e0 00 23` (3 bytes) | **3** | forces entry to the DTS syncwriter regardless of `s->0x2c`. |
| Q1b | `0x45FEB`: `bn.beqi` → `bn.bnei` + retarget | 3 | fixes the *upstream* prepare decision; needs per-caller reasoning. **Second experiment.** |
| Q3 | `0x460C2` geometry gate | 4 | symptoms, not cause. Deprioritised. |

**Why H1/H2 are 2–3 bytes and not 1** (learned the hard way this session): the AEON 32-bit branch
word is big-endian in the file, and **`i32_rel_simm3_13` spans bits 3..15, which straddles the 3rd and
4th file bytes**. So changing *only* the low byte (`uimm0_3`) silently corrupts the displacement:
`d0600423` decodes to `rel=132` → target `0x46135` (an arbitrary wrong instruction), **not** the
intended `0x460B5`. Both fields must be written together. Verified numerically with
`scripts/aeon_verify_patch_words.py`.

### ⚠️ BASELINE NOTE for the flash log

H1 and H2 were built from **stock** `dec_full.bin` (`4b7e9509…`), **not** from the currently-deployed
L2 image (`cba5f7e7…`). The only difference between L2 and stock is 2 bytes at `0x21F68/9`:

```
H1 diff vs stock: 0x460B3  04 -> 00    0x460B4  de -> 23
H2 diff vs stock: 0x4625D  e1 -> e0    0x4625E  fa -> 00    0x4625F  7e -> 23
L2 diff vs stock: 0x21F68, 0x21F69
```

**So flashing H1 also silently reverts L2.** L2 is proven inert (§9: `PATCH HAD NO EFFECT`), so this is
harmless — but it must be recorded, and it means the DEC image after the H1 test will be
`e64b76e8b3ce05809ae8d50409e2b0ad` with **L2 reverted and H1 active**, not "L2 + H1". If the user
prefers to keep L2 in place, H1 must be rebuilt from the L2 image instead (a 3-line change to the
builder).

**Note:** H1/H2 target the *interpretation* of the fork; Q1b targets the *cause*. With the cause now
identified (`params->0x0` at the single caller `0x41283`), **Q1b is the more principled fix** — but
H1 is the smaller, safer first probe and will discriminate between "the fork logic is wrong" and
"the upstream state is wrong" in one reboot.

`0x460C2` is a **generic** test, so I must confirm that for DTS specifically `r25 <= r26`.
`r25 = 0x38(r10) << 5` and `r26 = 0x8(r10) + 0x80`. I do **not** yet have the runtime values.

Two ways to settle it, in increasing cost:

1. **Static**: find the writer of `0x38(r10)` in the entry-init path (`0x130E0D`, `0x53804`) and show
   that the DTS entry gets a small selector while AC-3 gets a large one. `0x460B8 lwz r23,0x38(r10)` is
   *also* written at `0x46097 sw 0x38(r10),r11` with `r11 = (loop_index+1) << 5` — i.e. `0x38` is
   written by this very function from its own loop index, which makes it suspicious that `0x460C2`
   ever passes.
2. **Runtime**: instrument. If `0x460C2` is taken for both AC-3 and DTS, then it is *not* the
   discriminator and the real gate is upstream (in `0x45EA8`'s stride-`0x80C` walk / `0x45EE9`'s
   `0x808` test). The cheap discriminator is to capture the two values once with the existing
   `dump_spdif_npcm` harness plus a Kodi AC-3 vs DTS A/B — which we already know produces
   AC-3 non-zero / DTS zero.

**RESOLVED (see §5a):** `0x38(r10)` is left **at zero** whenever the descriptor does not take the
`0x28(r10)==1` arm, and then `0x460C2` *always* skips the header builder. So the true discriminator to
chase is **who sets `0x28(r10) = 1`**, not `0x460C2` itself. Do not build P1 until that writer is
identified — otherwise it repeats the `0x21F66` mistake of patching a gate that was never the
discriminator.

---

## 7. Reproduce

```bash
# 1. the authoritative instruction listing (already produced)
wc -l /c/firmware_temp/aeon_validate/dec_work/dec32_clean.txt      # 637125 lines, 637122 instrs

# 2. any region (numeric, do NOT use string comparison)
awk -F' ' -v s=$((0x460b5)) -v e=$((0x460f6)) \
  '{a=strtonum("0x"$1); if(a>=s && a<e) print}' dec32_clean.txt

# 3. the gate
awk -F' ' -v s=$((0x460c2)) -v e=$((0x460c7)) \
  '{a=strtonum("0x"$1); if(a>=s && a<e) print}' dec32_clean.txt

# 4. the pool + the 0x808 flag
grep -n "0x7e3c\|0x6e24\|0x808(r23)" dec32_clean.txt

# 5. all Pa/Pb emitters (note: bg.addi rN,r0,-0x78e == 0xF872)
grep -n "0x4e1f\|0x78e" dec32_clean.txt
```

---

## 8. Artifacts

| file | what |
|---|---|
| `dec_work/dec32_clean.txt` | **authoritative** dump: `<addr> <len> <mnemonic> \| <bytes>`, 637,122 instructions |
| `dec_work/dec32_out.txt` | raw Ghidra log of the above |
| `scripts/dec33_exact_attrib.py` | exact base/disp attribution using real instruction boundaries |
| `dec_work/dec27_out.txt` | first listing of `0x45xxx`/`0x46xxx` (decision function) |
| `dec_work/dec28_out.txt` | decode of `0x64296`/`0x64354`/`0x6434F` |
| `FINDING_dec_dts_output_rootcause.md` | **superseded** — kept for the record |
| `REPORT_final_dts_patch_experiment.md` §10 | **superseded** by this file |
