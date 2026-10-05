# MUSE TASK — DEC firmware: how the codec descriptor init is dispatched, and where `type`/`cmd` live

**Date:** 2026-09-29 · **Priority:** high (this is the live blocker)
**Files (all local, no device needed):**
- Authoritative listing: `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (637 131 lines)
- DEC image: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (1 982 492 B, md5 `4b7e9509…`)
  — **file offset == firmware address**
- Prior analysis to read first, but re-verify everything against the raw listing:
  `C:\firmware_temp\aeon_validate\r57_npcm\FINDING_dec_dts_output_rootcause.md`
- Kernel module: `C:\firmware_temp\patch_baseline\utpa2k_stock.ko` (25 381 336 B)

**The project goal** is DTS passthrough to the optical S/PDIF output. The DTS bitstream is
already correct and reaches the decoder (measured 2026-09-29: 880 084 B of valid IEC61937
with 692 `7F FE 80 01` sync words and the `0x77` DTS subframe header). The optical
transmitter emits **0 bytes**. The break is inside the DEC/DSP output path.

---

## Established already — do not re-derive, but do spot-check

1. **The DTS IEC61937 burst builder in the DEC image is complete and byte-correct.**
   Header builder at `0x45D07` emits data-type `0x0B` (IEC61937 DTS) for `r4==0` and
   `0x10C` (AC-3) for `r4==1`; the DTS-HD arm writes `0x1FFFE800`.

2. **The whole output path forks on one field, at `0x460B1`:**
   ```
   0460AA  bg.jal 0x00064354      ; r3 = s->0x24
   0460AE  bn.sw  0x30(r10),r3    ; guard-B
   0460B1  bg.bnei r3,0x0,0x4614C ; s->0x24 != 0 -> clear it, write NO syncword
   0460B5  ...                    ; s->0x24 == 0 -> build the DTS burst
   04627B  bg.ori r25,r0,0x7ffe / 046285 bg.ori r25,r0,0x8001  ==> 0x7FFE8001
   ```
   `s->0x24 != 0` ⇒ empty wire, which is exactly the measured symptom.

3. **Two known-bad leads, already closed — do not repeat them:**
   - The **SND** Pc patch is deployed on the device (file `0x1C051` `0x21`→`0x2B`, making the
     DTS branch write `Pc=0x0B` instead of the AC-3 code) and is **inert** — DTS never reaches
     the SND header writer.
   - The prior report's list of "14 candidate writers of `s->0x24`" is a **mis-label**: all 14
     actually write `0x30(r10)` (guard-B). Verified individually.

4. **Methodological trap:** `0x24(r10)` is **not** one global field — `r10` is a different
   struct pointer in different functions. The cluster at `0xD6000` writes `0x24` as the 10th
   entry of an ascending 11-value table (`0x0a,0x05,0x0b,0x11,0x17,0x1f,0x2b,0x3e,0x59,0x6e,0x80`
   at offsets `0x00..0x28`) behind a computed dispatch — an unrelated struct. **Scope every
   search to the struct the function actually uses.**

---

## Q1 (highest value) — How is the codec descriptor init dispatched?

Two nearly identical descriptor-init bodies exist, differing mainly in one immediate:

```
… 0x13C8B..0x140B5 : 01404E  bt.movi r4,0x0   014053  bn.sw 0x24(r10),r0   → DTS   (r4=0)
… 0x140B9..0x14232 : 014144  bt.movi r4,0x1   014149  bn.sw 0x24(r10),r0   → AC-3  (r4=1)
```

Both **zero** `s->0x24`. `r4` is then consumed by the header builder's `r4==0 → DTS` /
`r4==1 → AC-3` selection.

**Measured dead ends:** there are **no `jal`/`bg.jal`/`bt.jal` instructions anywhere in the image
whose target lies in `0x13000–0x15000`**, and `dec_full.bin` contains **no 4-byte little-endian
value equal to `0x13C8B`, `0x140B9`, `0x13C03`, `0x13FD8`, `0x141D0` or `0x14000`** in a
pointer-like table. The only 4-byte values in `0x13C00..0x14300` in the whole binary are
entries of two **ascending module-wide function-index tables** at file `0x194C10`
(`0x10FDE … 0x1591C`, stride ≈ `0x3A8`) and file `0x1D0E10` (`0x15555 … 0x12928`, descending).
Those look like a handler index, not a codec dispatch.

**Deliverable:** find the actual dispatch mechanism that selects the DTS body over the AC-3
body, and report **how the selector is computed** — i.e. the code that produces the `r4` value
or chooses the function pointer. Look specifically for:
* register-indirect calls (`jalr`) and the register that carries the target;
* a jump table whose entries are computed at runtime rather than stored;
* any arithmetic that maps a codec id to `r4` (recall the mdb `decType` legend:
  `1:ac3/ac3p, 4:dts, 5:aac, 17:dtsX`, and the AC-3/DTS setters in userspace use
  **format 5 = AC-3 and format 9 = DTS** — check whether either numbering appears here);
* **if and only if** the selector is derived from a licence or capability read, name the
  exact register/shift/mask, because that would be the gate we are looking for.

## Q2 — Does the DTS gate feed the fork?

The DTS gate is at `0x21AA4`:
```
021aa4  bg.movhi r24,0xb000 / 021aa8 bn.ori r25,r24,0x1e  ; r25 = 0xB000001E
021aab  bn.lh  r26,0x0(r25)                              ; r26 = *(u16*)0xB000001E
021ac3  bg.beqi r26,0x5,0x00021ad9                       ; verdict == 5  -> pass
021ac7  bn.ori r24,r24,0x1d / 021aca bn.lbz r23,0x0(r24)
021acd  bn.andi r23,r23,0x7f
021ad0  bg.beqi r23,0x0,0x00021ad9                      ; (byte & 0x7F)==0 -> also pass
```
(AC-3 twin at `0x21F52`, error path `0x22031` which zeroes `[r11+0xEC]`.)

**Deliverable:** trace forward from `0x21AA4` and backward from `0x46040` and state whether
any dataflow connects them — specifically, **does the licence result ever become the `s->0x24`
fork input, or any field of the struct `0x46040` uses?** If they are disjoint, say so plainly.
The AC-3 error path at `0x22031` is loop-back control flow (it jumps back to `0x21F52`), so
check carefully whether it is a real suppressor or just a retry.

## Q3 — Where are the `type` and `cmd` words that `audio_status` prints?

Live reading, same harness, same `SPDIF_OUT_BYPASS` state:

| | DTS | AC-3 |
|---|---|---|
| `[1st decoder] type` | `04` (4=dts) | `01` (1=ac3/ac3p) |
| `[1st decoder] cmd` | `00` (stop) | `84` (freeRun) |
| `DecStatus` | `0` (not "dec ok") | `1` (dec ok) |

This is the **newest and most important measurement**, and it contradicts a 2026-09-12 note
that DTS "decodes at full rate" (`Decode frame count : 12569`). Resolving that contradiction
decides whether the block is at command dispatch (much earlier) or at the output stage.

**Deliverable:** in `utpa2k_stock.ko`, find the `audio_status` handler that prints
`[1st decoder] type:` and `cmd:`. The format string `[1st decoder]` is the symbol `.L.str.751`
at offset `0x66A56` in section `.rodata.str1.1`. It is referenced from `.text+0x415160` and
`.text+0x415170` (Thumb). Report:
* the two SHM/MAD offsets it reads to produce `type` and `cmd`;
* whether `cmd` is a host-written word or a DSP-written status word — **this is the crux**;
* the value-word that `DecStatus` is derived from.

In `dec_full.bin`, then locate those same offsets in the decoder shared-memory window
(`g_virDecR2shm = MAD_base + 0xE01000` on the host) and report **every writer** of the `cmd`
word, and whether any of them is codec-conditional.

## Q4 — Two untried byte-verified patch hypotheses (report any objection, do not deploy)

* **H1** `0x460B1`: `d0 60 04 de` → `d0 60 00 23` (unconditional goto `0x460B5`, ignore
  `s->0x24`). 2 bytes at `0x460B2`/`0x460B3`.
* **H2** `0x4625C`: `d2 e1 fa 7e` → `d2 e0 00 23` (unconditional goto `0x46260`, relax the
  `s->0x2c == 1` requirement; the immediate **must** be zeroed). 2 bytes.

**If your trace of Q1/Q2 shows either patch targets code that provably cannot execute for a
DTS input, say so and explain why** — that is a more valuable result than a patch. Note the
listing warning: `i32_rel_simm3_13` straddles the 3rd and 4th file bytes, so a branch cannot be
retargeted by editing one byte alone.

---

## Rules for this task

* **Every claim must cite a line of `dec32_clean.txt` or a file offset in a named binary.**
  Quote the listing line, do not paraphrase it.
* Distinguish clearly between *read from the listing*, *inferred*, and *not found*.
* If a search returns nothing, say what you searched (pattern, range, tool) and report the
  negative — a clean negative is a result here, and two earlier "narrowing" lists in this
  project turned out to be mis-labelled.
* Do **not** propose a patch without a byte-level round-trip verification
  (decode the stock word, re-encode, show the bytes match).
* Nothing here touches the projector. Do not suggest flashing anything.

---

## Q1 — RESULT (2026-09-29, static, listing-verified)

**Short answer:** there is no codec-id→`r4` arithmetic and no licence/capability read in
the chain. The selector is a **slot index (0 = DTS arm, 1 = AC-3 arm, else nothing)**,
threaded as `r3` through two nested calls. The "measured dead end" in the task brief
(no `jal` into `0x13000–0x15000`) is **DISPROVEN**: three `bg.jal 0x00013fbb` exist
(L27704, L28006, L32095), and `0x13FBB` is inside that range.

### The dispatcher: F_13FBB (entry L22960)

```
L22960: 013fbb 3 bn.addi r1,r1,-0x18 | 1c21e8
L22966: 013fc8 2 bt.mov r10,r4 | 8944
L22967: 013fca 4 bg.bnei r3,0x0,0x000140b7 | d060076e   ; r3==0 -> DTS arm (fall-through)
```
- **r3==0 (DTS) arm**, L22968–L23043: SE read-modify-write `*(u16*)0xB000087C &= ~0xF`
  (L22968–L22971 `movhi r11,0xb000 / ori r23,0x87c / lh`, L22983 `andi r24,0xFFF0`,
  L22990 `sh_1 0(r23),r24`); descriptor immediates differ from AC-3 arm;
  `L23009: 01404e 2 bt.movi r4,0x0` → `L23011: sw 0x24(r10),r0` →
  `L23012: jal 0x00020512` → second SE RMW on `0xB0000844`
  (L23013–L23018 `ori r11,0x844 / lwz r24 / op11_1 / sw 0(r11),r24`); returns L23036–L23043.
- **r3==1 (AC-3) arm**, from `0x140B7`:
  `L23044: 0140b7 3 bn.bnei r3,0x1,0x00014093 | 206772` (r3∉{0,1} → shared epilogue,
  builds nothing); different immediates + SE RMW on the same `0xB000087C`
  (L23045–L23049, mask `0xFF0F` at L23064).
- The `r4` consumed by the `0x45D07` header builder (`r4==0 → 0x0B`, `r4==1 → 0x10C`)
  is produced **inside** F_13FBB by its own `movi r4,0/1`, selected by the `r3` test.
  No mdb `decType` (1/4/5/17) or userspace format (5/9) numbering appears in F_13FBB.

### How `r3` (slot) is computed — full chain, caller to callee

F_13FBB's `r3` comes from `r11` of the enclosing function F_1760D (prologue at
`0x1760D`, L27344; `L27365: 017648 2 bt.mov r11,r3` — **r11 is F_1760D's input `r3`**):

| call site of F_13FBB | `r3` source (read from listing) | post-call slot guard |
|---|---|---|
| L27704 `017ab0 jal 0x13FBB` | `L27703: mov r3,r11` (with `r4=r18, r5=[r10+0x27C], r6=[r10+0x284]`, L27700–L27702) | L27705 `bleui r11,1 → 0x17E68` |
| L28006 `017e60 jal 0x13FBB` | `L28005: mov r3,r11` (with `r5=[r10+0x27C], r6=[r10+0x284], r4=r18`, L28002–L28004) | L28007 `bgtui r11,1 → 0x17AB8` |
| L32095 `01b1d2 jal 0x13FBB` | `L32089: lwz r3,0x0(r16)` (with `r4=r23, r6=[r16+0x284]`, L32090–L32091) | — (then `[r16+0x38]` check, L32096–L32097) |

F_1760D itself stores the slot sticky into its instance struct
(`L27538: 01787d sw 0(r10),r11`) and guards `L27464: bleui r11,1 → 0x181A1`.
Its own `r3` (slot) comes from 6 callers of `0x1760D`:

| caller | slot value (read) |
|---|---|
| L14844 `00dbb5 jal 0x1760d` | immediate: L14842–L14843 `movi r5,0 / movi r3,0`, struct `r4=0x465840` (L14841) |
| L14851 `00dbcb jal 0x1760d` | immediate: L14849–L14850 `movi r5,0 / movi r3,1`, struct `r4=0x465B90` (L14847–L14848) |
| L28408 `018372 jal 0x1760d` | forwarded: L28404–L28407 `r3=[r10+0], r5=[r10+4], r4=r10` (after `sw 0xC(r10),r11`, L28406) |
| L28761 `0187ec jal 0x1760d` | forwarded: L28758–L28760 `r3=[r10+0], r5=[r10+4], r4=r10`, ints masked (SPR 0x11, L28754–L28757) |
| L29768 `0194e4 jal 0x1760d` | forwarded: L29765–L29767 identical pattern, ints masked (L29761–L29764) |
| L30136 `019981 jal 0x1760d` | register: L30132–L30134 `r5=r15, r3=r11, r4=r10`, ints masked (L30128–L30131) |

Each init call is immediately followed by a paired `jal 0x230A3` with the **same** slot
(L14846 `movi r3,0 → jal`, L14853 `movi r3,1 → jal`). F_230A3(slot) (entry L42424):
`L42429 mov r10,r3; L42430 blesi r3,1 → 0x230BE` (slot>1 returns immediately), then
slot-strided table init (`L42437 muli r23,r3,0x78` into `0xE01BC8`, memset via
`0x130D25` at L42444) and SE shadow reads `0xB0000838/081C/083C → struct+0x48/4C/50`
(L42445–L42460), then `jal 0x2163B` with `r3=slot` (L42461–L42462).

### Answers to the brief's specific prompts

- **Register-indirect calls:** EXIST but are **not** the F_13FBB dispatch —
  `L27656: 017a19 bt.jalr r24` (target from `[r25+0x2AC]` table → `[r3+0]`, L27647–L27655)
  and `L27662: 017a29 bt.jalr r23` (target `[r3+0x10]`, L27659–L27661). Codec-op
  vtable pattern inside F_1760D's table-setup path. No `jalr` targets F_13FBB's arms.
- **Jump table with runtime-computed entries:** not found for this dispatch; the
  `0x194C10`/`0x1D0E10` index tables from the brief were not referenced by any of the
  three call sites above (INFERRED from absence along the traced paths, not exhaustive).
- **Licence/capability derivation:** NONE. The only SE reads in the chain are control-bit
  RMWs: `0xB000087C` (F_13FBB both arms), `0xB0000844` (DTS arm),
  `0xB0000C28 |= 0x4` (F_1760D, L27682–L27686), `0xB0000838/081C/083C` (F_230A3).
  No byte from `0xB000001D/1E` (the `0x21AA4` licence gate) reaches `r3/r11/r4` here.
- **Where the runtime decision actually lives:** one layer up, on the `decType` field
  `[r10+0xC]` — `L27693–L27694: lwz r23,0xC(r10); beqi r23,1 → 0x17E27`
  (==1 → site-2 path, else site-1 path) and
  `L27987–L27988: lwz r23,0xC(r10); bnei r23,1 → 0x17A92` (site-2 entry gate).
  The slot picks the descriptor *template per decoder instance*; `decType` picks the
  *call path*; the `0x45D07 r4` test picks the *burst header*. Conflating the three
  was the earlier mis-labelling trap.
- **Status vocabulary:** slot-index mechanism VERIFIED (init immediates + forwarding
  + sticky store + both arm entries all read from listing); licence-derivation
  DISPROVEN for this chain; vtable-`jalr` as arm selector DISPROVEN (targets are
  table-setup helpers, arms are reached by direct `jal` + `r3` test).

**Q4 implication (preliminary, pending Q2):** F_13FBB provably executes at init for
slot 0 (DTS arm zeroes `s->0x24` at L23011) — so a DTS input *can* reach code that
builds the DTS descriptor. Whether H1/H2 targets can execute for DTS input depends
on the Q2 link (gate `0x21AA4` → fork `0x460B1`), still open.

## Q2 — RESULT (2026-09-29, static, listing-verified): DISJOINT

**Short answer:** the licence verdict never becomes the fork input, nor any field the
fork path reads. The DTS gate's fail path does not even block — it only changes an
arithmetic multiplier. The AC-3 twin's `0x22031` is a real suppressor-exit, `0x22023`
is the retry loop.

### Fork input refined: it is a getter return, not a struct read

```
L89113: 0460a8 2 bt.mov r3,r10 | 886a
L89114: 0460aa 4 bg.jal 0x00064354 | e403c554
L89115: 0460ae 3 bn.sw 0x30(r10),r3 | 0c6a30   ; guard-B = fork result
L89116: 0460b1 4 bg.bnei r3,0x0,0x0004614c | d06004de
```
`F_64354` (dec28_out.txt L213–L215) is one instruction:
`064354 bn.lwz r3,0x24(r3)` + `jr` — i.e. **fork input = structB+0x24**, where structB
is F_45FD1's `r10` (= its `r3` arg, `L89048: 045fe9 mov r10,r3`, prologue L89038 `045fd1`).
Sibling micro-ops (dec28 L212/L216–221): `F_6434F: sw 0x24(r3),r0` (clearer),
`F_64359: [r3+8]+= [r3+4]; [r3+4]=0`, `F_64149: 10-word copy r4→r3 incl. +0x24`
(dec28 L52–L71).

### The only writers of the fork word

1. **Entry copy** `L89062–L89064`: `lwz r4,0x28(r4); jal 0x64149` (dst `r3`=structB) →
   structB+0x24 ← descSrc+0x24, where `r4` = F_45FD1's arg = `[r11−0x5EF8]`
   (single caller `L82516–L82518: lwz r4,-0x5EF8(r11); mov r3,r10; jal 0x45FD1`,
   with structB = caller-r10+`0x14CCC`, L82513–L82515). Then `sw 0x2C(r10),0`.
2. **Clearer F_6434F**, exactly 2 callers in the image (grep `jal 0x0006434f`):
   `L89161: 04614e` (post-fork nonedge: `mov r3,r10; jal; j 0x460F3`) and
   `L89804: 04690d` (`mov r3,r10; jal; j 0x4674E`, snapshot path after three
   `F_64149` stack copies L89788/L89793/L89799 — which also propagate guard-B+0x24
   to stack snapshots).
3. **Negative:** no direct `sw/sh/sbz …0x24(r…)` anywhere in `0x45xxx–0x46xxx`
   (grep `045…/046… .*sw.*0x24(r`, clean negative). F_13FBB's init-zeroing
   (L23011) targets the per-slot *descriptor* (`0x45A330+slot*0xD8` via `r18`,
   L27534–27537), a different address than structB — aliasing UNPROVEN, and the
   *runtime* setters of descriptor+0x24 remain the one unmapped door (see caveat).

### Licence fan-out stays inside the gate struct (structA ≠ structB)

Gate fn F_21170 (prologue `L39910: 021170 addi r1,-0x18`; `L39916: mov r10,r3`;
`[structA+0x1714]=r4`, L39915), 2 callers: `L27255: 0174ec` (structA=`[r10+0x18]`,
L27250–27255) and `L28115: 017fbd` (structA=`F_20D71(…)` return, L28100–L28115).
- DTS gate (L40659–L40677): `r26=*(0xB000001E)`; pass→`0x21AD9` on `==5` (L40668) or
  `(*(0xB000001D)&0x7F)==0` (L40669–L40672); fail path only recomputes
  `r27=byte&0x7F` (L40673–L40674) instead of the cmov const 4/`0xC` (L40666–L40667).
  `r26` is stored **only** to `[structA+0xE8]` (L40677); `r27` feeds
  `muls r24,r27,r16 → add r28 → bles r25,r28 → 0x21C47` size check (L40682–L40684)
  and, on the small side, `[structA+0xF0]=r27` (L40685–L40687). No trap, no +0x24.
- AC-3 twin (L41031+): fail (`!=5`, L41034) falls to
  `L41037: blesi r24,-1 → 0x22031` with `r24=0xB0000000` — always taken, i.e. a
  disguised unconditional jump. `0x22031` (L41095–L41098: `beqi r23,0 → 0x21E8D`
  else zero `[r11+0xEC]` word+half, `j 0x21E87`) is the **suppressor-exit**
  (single-shot zero + leave); `0x22023` (L41091–L41094: zero `[r11+0xEC]`,
  `[r10+0x64]`, `[r10+0x74]`, `j 0x21F52`) is the **retry loop**. Brief's question
  answered: both exist, they are different addresses with different exits.
- **Negative:** no read of `+0xE8/+0xF0/+0xEC` anywhere in the fork flow
  `0x45FD1–0x4618D` except `L89194: 0461ab sw 0xF0(r10),r14` with `r14=0`
  (L89192) in F_4618E's not-ready arm, and init writes `L88892/L88943` — i.e. the
  fork path never consumes a licence-derived field.
- structA bases (`[F_174xx-r10+0x18]`, `F_20D71` return) vs structB base
  (F_41xxx-r10+`0x14CCC`, single caller L82518): different functions, different
  derivations, no shared channel found in bounded trace → DISJOINT (STRONGLY
  SUPPORTED; full aliasing disproof would need runtime bases).

**Caveat (the remaining door):** who sets descriptor+0x24 at runtime after F_13FBB's
init-zero is still unmapped; the licence could only enter the fork word through
those setters, and none was found in the gate fan-out. The `0x13B` state machine
and D-series writers are the place to look, not the `0x21AA4` gate.

## Q3 — RESULT (2026-09-29, static, ARM disassembly of `utpa2k_stock.ko`, capstone+relocs)

All offsets below are `.text`/`.rodata.str1.1` file offsets (`ET_REL`, `e_machine=EM_ARM`;
`.text` file base `0xD654`, vaddr 0). `bl` targets resolved via `.rel.text`
(`R_ARM_CALL`=28, `R_ARM_MOVW_ABS_NC`=43, `R_ARM_MOVT_ABS`=44). Code is ARM mode,
not Thumb, in this region.

### `type` / `cmd` / `cpu_dec` — SE bytes, host-read

Print loop in `MDrv_AUDIO_Debug_Cmd_Read` (covers `.text+0x415160`):
- `r5 = 0x112E98` (`0x4150B8 movw/movt`); `r4 = AbsReadByte(0x112E98)` (`0x415140/44`),
  `r6 = AbsReadByte(0x112E99)` (`0x41514C/50`), `r0 = AbsReadByte(0x112E98)` (`0x415158/5C`);
- `MdbPrint(fmt=.L.str.751, r2=r4&0x7F, r3=r6, [sp]=r0>>7)` (`0x415160/64/68/6C/70/7C`):
  `type = [0x112E98]&0x7F`, `cmd = [0x112E99]` (raw), `cpu_dec = [0x112E98]>>7`.
- Second decoder identical at `+2`: `0x112E9A/0x112E9B` with `.L.str.753`
  (`0x415180–0x4151C8`, relocs `0x4151A4/1B8`).
- `HAL_AUDIO_AbsReadByte` (`0x44144C`, 8 insns): `r0-=0x100000`, byte-lane expand
  (`rsb r0,r1,r0,lsl#1`), `r0 = [remapped_base + r0]` — true physical SE addresses.
- Legends printed just below (`.L.str.755` = mdb `decType` numbering incl. `4:dts`;
  `.L.str.757` = `decCmd (0:stop … 80:freeRun)`, read as hex `0x80` — consistent
  with the measured AC-3 `cmd=0x84`, see below).

### `cmd` is HOST-written (the crux — command, not status)

- Bit7 ← `HAL_MAD_SetFreeRun @0x45FC40/0xB4` tail (`0x45FCD8–0x45FCF0`):
  `cmp r4,#2; bhs` (validate), `lsl r0,r4,#7; mov r1,#0x80; uxtb r2,r0;
  movw r0,#0x2E99; movt r0,#0x11; bl AbsWriteMaskByte` — bit7 of `0x112E99` = arg.
- Bit4 ← `HAL_MAD2_SetDecCmd @0x466FAC/0xD0`: set (`0x467014: mask 0x10,val 0x10`)
  vs clear (`0x46702C: mask 0x10,val 0`) on `0x112E99`, chosen by caller branches.
- Stop ← `HAL_AUDIO_WriteStopDecTable @0x44279C/0x6C` (4× MaskByte incl. `0x112E99`).
- Mirror in `HAL_AUDIO_SetDecCmd @0x44C144/0x73C`: `r6 = 0x112E99`
  (`0x44C24C/25C movw/movt`; `+2` for path 2 at `0x44C278`); small-cmd path
  (`sb≤0x20`, jump table `ldr pc,[r1,sb,lsl#2]` at `0x44C3A0`) does
  `Set_SHM_PARAM(0x91,…)` (`0x44C428/38`) **to the DSP** plus
  `AbsWriteMaskByte(r6, mask 0x6F, val 4)` (`0x44C43C/48`) — preserves bit7, sets bit2.
- Measured bytes decompose exactly: AC-3 `0x84 = 0x80` (freeRun bit7, host set via
  SetFreeRun) `| 0x04` (bit2, SetDecCmd mirror); DTS `0x00` = stop state, host never
  set bit7. **DTS is never commanded.**
- Dispatch gap scoped: `_MApi_AUDIO_SetCommand @0x3E5F6C/0x338` stubs call
  `MDrv_AUDIO_SetDecCmd(0, X)` (trampoline `0x4207A0→HAL_AUDIO_SetDecCmd`) with
  `X ∈ {0,1,2,3,4,5,6,8,9,0xA,0xB,0xF,0x10,0x20}` — no `0x80`/freeRun among them;
  freeRun arrives only via the `HAL_MAD_SetFreeRun` path, which the DTS flow never
  triggers (callers of setters enumerated: 20× `MDrv_AUDIO_SetDecCmd`,
  `0x4279A8/0x4674CC→HAL_MAD2_SetDecCmd`; per-codec arg mapping inside the stubs
  NOT fully traced — residual).

### `type` byte — no writer in either image (clean double negative)

- Host: full-`.text` `movw`-immediate scan for `0x2E98` + review of all 1337
  `AbsRead/Write/Mask` call sites → zero writers of `0x112E98` (STATUS: clean negative).
- DSP: no `0x2E98/0x2E99/0x2E9A/0x2E9B` (or `0x12E9x`) immediate anywhere in
  `dec32_clean.txt` (637 131 lines) → the DEC firmware never addresses the
  type/cmd bytes in any view (STATUS: clean negative).
- Therefore `type` (=`0x04` DTS / `0x01` AC-3 as observed) is written by HW or by a
  host path that does not use absolute SE addressing (e.g. a mapped struct store —
  not exhaustively excluded). It behaves as status; `cmd` behaves as command.

### `DecStatus` — value word partially traced (residual remains)

- Print `.L.str.795` = `DecStatus     : %x  (1:dec ok)\n` (`0x10C6DC`, size `0x22`);
  site `0x418A90/94` (movw/movt) + `MdbPrint @0x418AA0`, args `r0=r8=[sp,#0x94]`
  (saved query result), `r2=[sp,#0x88]&0xFF`.
- Upstream ingredients (all in `MDrv_AUDIO_Debug_Cmd_Read`):
  `HAL_DEC_R2_Get_SHM_PARAM(0x18/0x91)`, `HAL_DEC_R2_Get_SHM_INFO(0x3C/1)`,
  `HAL_MAD_GetCommInfo(0x814C/0x814D)`, `AbsReadByte(sl+0x4D/0x2D)`,
  `AbsReadReg(0x112C48/0x112C0C/0x112C40)`. The producer of `[sp,#0x94]` (hence the
  exact `DecStatus` source word) was NOT reached in this pass — residual, but the
  direction is fixed: DSP-SHM/host-query derived, consistent with `0` while
  `cmd=stop`.

### Old-vs-new contradiction resolved (INFERRED, consistent with all data)

2026-09-12 "DTS decodes at full rate" (frame counter) vs now (`cmd=00 stop`,
`DecStatus=0`, wire silent): the block is at **command dispatch, much earlier**
than the output stage. The host parks decoder0 in stop for DTS (never issues
freeRun), so the DSP never runs (`DecStatus=0`), the fork word never leaves its
init state, and the S/PDIF wire stays empty — while a frame counter elsewhere can
still free-run. The 09-12 note most likely sampled a different/latched state.
Consequence for the whole project: H1/H2 (Q4) would force a burst from a decoder
the host never started — treat them as diagnostic probes, not fixes; the fix
belongs in the host `SetDecCmd`/freeRun path for the DTS codec id.

## Q4 — RESULT (executability only; no patch proposed, no bytes changed)

Stock bytes verified against the listing (both match the brief exactly):
H1 `L89116: 0460b1 … | d06004de`, H2 `L89257: 04625c … | d2e1fa7e`. Re-encode of the
proposed words was NOT performed (no trusted AEON branch encoder in this session) —
byte round-trip stays OPEN, do not deploy on this analysis alone.

- **H1 (`0x460B1` force fall-through): no executability objection.** The fork branch
  is reached for every input that arrives at the output stage, DTS included: no
  codec-conditional branch precedes it in F_45FD1 (only the `F_643DC-return > 0xD`
  early exit L89085 and the `0x46073/0x45FFC` count-loop diverts, none codec-gated).
  Semantic caution (not a disproof): forcing the `==0` arm when the word is nonzero
  builds a burst from a "not ready" descriptor — symptom could change from
  silence to garbage rather than to sound.
- **H2 (`0x4625C` relax `+0x2C==1`): CAN execute for DTS input, but is likely
  INSUFFICIENT alone.** Full gate chain read from listing:
  `L89256–L89259: lwz r23,0x2C(r10); bnei r23,1 → 0x461AB; lwz r23,0x30(r10);
  bnei r23,0 → 0x462DE` → `L89260–L89270` syncword build (`0x7FFE`/`0x8001` stores).
  `+0x2C==1` holds exactly when `[descSrc+0x10]==1` at fork entry (only writers:
  entry-zero L89064 and `L89102: sw 0x2C(r10),r11` in the `0x46080` arm, where
  `r11==1` by the entry branch L89051). So H2 matters only in the
  (fork-word==0, `+0x2C`!=1) case; and whenever the symptom's cause is
  fork-word!=0 (guard-B, the very next check L89258–L89259), H2 changes nothing —
  execution still diverts to `0x462DE`. If a patch is ever tried, H1 dominates H2:
  H1-upstream-zero makes both H2 gates pass trivially… except `+0x2C`, which H1
  does not set (set only via the `0x46080` arm or the `F_64359`--adjacent paths —
  verify before combining).
- Path-to-H2 reachability for DTS: F_4618E (runs right after F_45FD1 per the single
  caller L82519–L82520) requires `[structB+0x28]==1` (L89190–L89193) to enter the
  output loop at all, then the `0x461FD` count loop (`ble r14,r16 → 0x46259`,
  L89224). None of these tests is codec-id-gated — state words only — so a DTS
  input is not provably excluded from H2's address. The exclusion, if any, is the
  *values* of those state words for DTS, which are runtime data.
