# MUSE TASK — DEC: the `+0x20` writer, the B-latch, and the unprobed arms past `0x46156`

**Date:** 2026-09-30 · **Priority:** highest — your last report closed the ring branch and
pointed the whole investigation at a small set of named words
**Files:**
- `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt`, `dec_full.bin` (md5 `4b7e9509…`, file offset == fw addr)
- **New:** `C:\firmware_temp\aeon_validate\snd_work\snd32_clean.txt` — the SND listing (293 718 insns), generated 2026-09-30; treat the SND side the way you treat the DEC side
- Read first: `C:\firmware_temp\MUSE_TASK_ring_remaining_20260930.md` → `## RESULTS`

---

## Where your report leaves us — and the synthesis I want tested

Your Q2 result (no DTS divert branch anywhere; the ring is a shared codec-blind pump) and your
A/B verdict (bytes-per-SetCommInfo 1.56 vs 1.88 KB — per-byte progress for DTS, no spin
excess) together remove the "empty ring" explanation and point the block at **word state**, as
you said. I want that made concrete and then actionable.

**`F_643DC` returns `[0x14] − ([0x0] − [0x20])/4`**, and you established **`[0x0]` (the B-latch)
is 0 for DTS.** That arithmetic predicts the failure exactly, and it predicts where:

```
04604b  jal  0x000643dc          ; r3 = [0x14] − ([0x0] − [0x20])/4
04604f  bgtui r3,0xd,0x00046000  ; r3 > 13 -> divert
```

With `[0x0] == 0` and `[0x20]` positive, `([0x0] − [0x20])` is **negative**, so the subtraction
in `F_643DC` *adds* instead of subtracting, and `r3` comes out **larger than the 0x0D bound** —
the size gate diverts for DTS. For AC-3 the latch is non-zero, the arithmetic yields a small
value, and the gate passes. **This is a hypothesis formed from your numbers; test it or kill
it.** If it holds, the block is a single **word-state** difference, not a missing code path.

## Q1 (decisive). Confirm or refute that arithmetic

* Read `F_643DC` (`0x642DC`) again and state the exact expression and the **sign/width
  behaviour** of each operand, with the listing lines.
* For AC-3 vs DTS, what are `[0x14]`, `[0x0]` and `[0x20]`? If any is codec-conditional,
  that is the block. `[0x20]` was left as a residual in your Q1 — **close it.**
* Then state plainly: **is `r3 > 0x0D` true for DTS and false for AC-3?** If yes, the size gate
  at `0x4604F` is the block, it is one comparison, and the fix is to make the B-latch behave —
  not to touch the ring.

## Q2. The B-latch `[0x0]` — who writes it, and why only AC-3?

Your Q1 said `[0x0] ← B-latch (0 for DTS)`. Now trace it to its writer:

* every writer of `+0x0` on the structure `F_643DC` reads;
* for each, **what condition sets it**, and whether that condition is codec-conditional;
* name the single site where AC-3 gets it set and DTS does not. **That site is the block**,
  and if it is a `beq/bls` on a codec field it is a one- or two-byte patch.

## Q3. The unprobed arms past `0x46156`

You left `0x46156+` unprobed and, with the fork exonerated (`0x4614C` = clear + rejoin
refill, which explains why H1 was null), **those arms are now the prime suspect.**

Decode from `0x46140` to the end of that function: every branch, what each arm does, and which
one the DTS path would take after the size gate. Specifically: **is there an arm after
`0x46156` that AC-3 takes and DTS cannot?** H1 forced `0x4614C` and got nothing, which only
rules out that one arm — say explicitly which arms H1 did and did not cover.

## Q4. `F_295BC` linkage and the `+0x24` setter (your residuals)

Close both. `+0x24` is the fork input at `0x460B1`, so who writes it — and is that writer
codec-conditional — is the same question as Q2 from the other end.

## Runtime facts you can use

All measured on the device with verified playback and an early capture window:

```
AC-3 : dump_spdif_npcm = 2 119 680 B
DTS  : dump_spdif_npcm =       0 B   (decoder alive, dump_es = 2.08 MB valid IEC61937)
DTS  : decoder cmd:0x84 DecStatus:1 for ~22 s, then cmd:0x00 when the file ends
```

Three one/two-byte patches were flashed and **all returned null**, each with the AC-3 control
passing: H1 `0x460B1` fork forced; H3′ `0x21AD3` licence clobber NOPed; and the stream-type
patch `0x19861 E4→E5` (forging `5` for the DTS instance). All reverted; the device is on
stock. **Do not re-propose output-stage patches — yours and mine both now have three nulls.**

Runtime A/B that constrains your answers:
```
            total lines   SetCommInfo   AUR2   HAL_MAD
DTS           124 246           1 106     1 371   3 465
AC-3          105 780             340       604   1 368
```
DTS drives the R2 ~3.3x harder than AC-3 and moves ~4x the elementary-stream bytes while
emitting zero on the wire. **An empty ring would mean DTS does *less*, not more.**

Also: `read_dsp_sram_type` cannot reach this state — swept 4 099 cells under both codecs, only
92 differ, all short runs of live counters/pointers, and the `0xE0xxxx` SHM window reads
all zero for both. **Treat "observe the word" as unavailable unless you can show a path.**

## Tooling now available

* **Ghidra works for both images.** `analyzeHeadless` recipe, with the three fixes that were
  needed (see the note in `td98-dec-output-entry-gate.md`). The SND listing exists; use it.
* `reko.exe disassemble --arch aeon --data <HEX>` for the `0x28`–`0x2F` MAC group, which the
  public Ghidra slaspec leaves blank.

## Method notes

Seventh trap, same family: the listings render `bg.sw_0 0xa0(r1),r30,` — mnemonic **with a
suffix and a trailing comma** — so `\bsw\s+0xa0\(` returns **0** on a file containing 568
occurrences of `0xa0`. Probe the format before trusting any count. Six earlier counting
passes were silently wrong for this reason.

Function-boundary discipline: a `jr r9` can be the *end* of a fragment, not a prologue —
`0x190F0` turned out to be a return and the code after it belonged to a tail-callee. That error
produced two false conclusions in this project. Derive entries from `jal` targets and the
instruction after a 2-byte `jr r9`, then verify with a walk that models calls **and** returns.

## Rules

* Every claim cites a listing line or a binary offset.
* Q1 is confirm-or-refute. If the arithmetic does not explain DTS, say so — that would reopen
  whatever else is consistent with your A/B.
* Every claim cites a listing line or a binary offset.
* Q1 is confirm-or-refute. If the arithmetic does not explain DTS, say so — that would reopen
  whatever else is consistent with your A/B.
* Do not flash anything. Static only.

---

## RESULTS (2026-09-30, static only)

Notation: `@0x4604b` = AEON addr = `dec_full.bin` offset = `dec32_clean.txt` 6-hex
field. `bg.jal` caller counts are exact greps (8-digit lowercase). Codec-blind =
full-body read, no compare on a codec value. F_45FD1 = `[0x45FD1,0x462D8]`
(entry decoded this session, `045fd1..46015`); F_410A3 = `[0x410A3,0x41354]`;
F_3D952 = `[0x3D952,…]` (prologue + arg moves `03d952..03d98c` VERIFIED).

### Q1 verdict: literal hypothesis REFUTED, mechanism REFINED (and stronger)

Sign/width first (`0642dc..0642ee`): `bn.sub` = 32-bit wrap (mod 2³²);
`bn.srai` = arithmetic (sign-extending) shift — distinct mnemonic from `srl`
(`064432`) and `sra` (`0644a4`), both present in the listing; `bg.bgtui` =
UNSIGNED greater-than-immediate. So `0x4604f` diverts on large-positive AND on
negative-wrapped `r3`. That half of the hypothesis is correct.

The literal half is not: **`[0x0]` is not the B-latch at `0x4604b`.** `F_45FD1`
re-derives it first: `0x4601c mov r3,r10; 0x4601e jal 0x64296`, and `F_64296`
(`064296..0642c1`) overwrites `+0x0/+0x4/+0x8` with
`[0x0] := ([0xC]<<2)+[0x20]`, `[0x4] := [0x10]-byte`,
`[0x8] := ([0x14]−[0xC])<<5 + [0x18] − [0x10]` (`0642b8..0642be` stores).
Both queries (`0x4602f`, `0x4604b`) run AFTER this (and after two `F_644F6`
produces, which only advance `+0x0/+0x4/+0x8`). Substituting:
`([0x0]−[0x20])/4 ≈ [0xC] + w` (w = produced words, ≈1–2), so at `0x4604f`:

```
r3 ≈ [0x14] − [0xC] − w  >  0x0D (unsigned)  →  return via 0x46000
```

The B-latch acts ONLY at entry (`0x45feb beqi [r4+0],1 → 0x4600c`). The size
gate tests `+0x14` vs `+0xC`. Writers, all codec-blind by body (no codec
immediate at any site): `+0x14 ← 0x3DB2C (F_37F32 query)`, `+0xC ← 0x3DB54
(F_37F32 query)`, `+0x10 ← 0x3DB62 (F_37F54)`, `+0x28 ← 0x3DB21`,
`+0x1c ← 0x3DB42`, `+0x2c ← 0x3DB38 (F_37F78)` — all generic query+store in
F_3D952; values are descriptor-fed → runtime (watchlist W1).

`+0x20` CLOSED: word writers FW-wide are exactly `0x64010` (F_64004 zero-init,
sole caller `0x6755d`), `0x64039` (F_64039 arg-init `+0x20=+0x0=r3`, 2 callers:
`0x68147` with `r3=r28,r4=0x800`; `0x68BC9` with `r4=r13`), `0x6417c`
(F_64149 copy). No writer in F_45FD1/F_410A3/setup; ring helpers never store
`+0x20` (VERIFIED across `0x64000..0x64520`). Sub-word sweep (`sb/sbz/sh`,
trap #7 applied): all `sbz` (= zero, consistent), all `sh_1` value-stores live
in far regions (`0x07xxxx–0x1Dxxxx`), NONE in any function touching our objects
— aliasing disfavored, closer is runtime (W2). `+0x20` values per codec:
runtime. Residual: C1/C2 destination-object ≡ descSrc linkage.

So: "single word-state difference, not a missing code path" SURVIVES, corrected:
the words are `[r10+0x28]` / B-latch / `[r10+0x38]==0x400` (three silent-return
gates in series, Q3), and the remaining-gate operands are `+0x14/+0xC`, not the
latch. `r3 > 0x0D` for DTS vs AC-3 is runtime-decidable; statics deliver the
reduced formula + complete writer map.

### Q2. `+0x0` writers + the 4-gate latch chain (single-site narrowed, not picked)

Every `+0x0` writer on the object F_643DC reads (F_45FD1.r10 ≡ desc object):

- (a) Latch `0x3DE7C` (`[r10+0x0]=1`), conditional on FOUR gates in series:
  1. `0x3DA55/0x3DA59`: `[r12+0xC]∈{1,2}` → `0x3DDD2`/`0x3DE18` arms, which write
     `[r10+0x18]` (`0x3DDE8` via F_37EC2; `0x3DE2C` via F_37ED6). Fallthrough
     (∉{1,2}) leaves `[r10+0x18]` stale-zero (init zeroes it: `sbz` at `0x64016`
     / `0x6404e`) → latch unreachable. r12 = entry r6 ← F_3E961 `@0x3F6B6`
     (sole caller of F_3D952, VERIFIED; r6 computed just above — chain residual).
  2. `0x3DA6B`: `[r10+0x18]==1 → 0x3DE6B`.
  3. `0x3DE71`: `F_37EDC(r13)==1`, where F_37EDC (`037edc..037ee2`, 3 insns) =
     `[[r3+0x54]+0x68]` double-deref; r13 = entry r4 (`0x3D988 mov r13,r4`).
  4. `0x3DE78`: `[r10+0x4]==1` (`+0x4 ← 0x3DA68`, F_37F25 query).
- (b) Derivation `0x642B8..0x642BE` (above) — overwrites latch before queries.
- (c) Producer advances (`0x64408/0x64426/0x64444/0x64498/0x644B6/0x6451D`,
  bufptr) and merge-helper recomputes
  (`0x641CF/0x6422F/0x6409B/0x640AE/0x64085+`).
- (d) Inits: SETUP_B `0x294A6` (`+0x0` = `F_37535` query return), F_64004-zero,
  F_64039-arg, F_64149-copy.
- EXCLUDED with proof: F_410A3-tail `sw 0x0(r4)` sites (`0412e4..041354` —
  status codes, each followed by `movi r3,1 + jr`, different struct) and the
  `0x46238+` burst stores (output buffers, base regs r23/r24/r5).

No `beqi type,4/5`-style immediate exists anywhere in this chain (VERIFIED
absence across all five windows) — the latch is data-driven. The single site
where AC-3 gets it and DTS does not is ONE of gates 1–4; which one fails for
DTS is runtime (all four operands are descriptor-fed). Watchlist W3 =
`[r12+0xC]@0x3DA55`, `[r10+0x18]@0x3DA6B`, `F_37EDC-ret@0x3DE71`,
`[r10+0x4]@0x3DE78`. A forcing patch would be 1–2 bytes at the failing gate —
NOT proposed (3 nulls + unknown gate; observation first).

### Q3. Full arm map; burst needs `[0x38]==0x400`; H1 doubly null

- Remaining-table arms (`0x4610c..0x46144`, targets VERIFIED last session) all
  `ori r23,IMM; j 0x46069`, joining the common store
  (`0x46069..0x46070`: `[r10+0x80/0x78/0x7C]=r23`; rem13 → `0x46065` = `0xBB80`).
- `0x46073` filter: `r12 = remaining−5` (`0x46036..0x46038`); `sfgtui r12,0x7A;
  bf → 0x46000` (RETURN). Only rem ∈ {6,7,8,11,12,13} proceed; rem {0–4} return
  here, rem {5,9,10} return at the table. L89104–89108 = `0x4608E..0x46097`
  (line↔addr identity VERIFIED): `[r10+0x38] = (rem+1)<<5` — reachable via
  `0x46079/0x4607C → 0x45FFC → 0x4608E` (jump skips the `movi r11,0`, so
  r11 = remaining, hence `{0x20..0x1C0}` ✓).
- Fork: `0x460AA jal` + `[r10+0x30]` store (`0x460AE`) + H1-site `0x460B1`.
  `0x4614C` = clear + `j 0x460F3` (refill → `0x46101` remaining →
  `[r10+0x34]` store → `j 0x46000` RETURN). Refill returns, never emits.
- Size gate `0x460B5..0x460C2` → `0x460F3` (refill) or flag tests:
  `==0x200 → 0x46156` (arm A + fallthrough `(r4=0,r6=0x800)` builder),
  `==0x400 → F_46172` (own frame `0x46172`, no `jal` entry — sole entry is the
  `0x460D1` branch, VERIFIED), `==0x800 → refill`, else `(r4=2,r6=0x2000)`
  builder (`0x460DB..0x460EF`) → refill.
- Builder `F_45D07`: `(r6>>2)`-class × r4-mode machine writing control words
  `[r3+0x16C]/[0x170]`; tags `r4=1 → 0x10C` (`0x45D7F`), `r4=0 → 0x0B`
  (`0x45DA0`), `r4=2 → 0x20D` (`0x45DBF`). Programs control, emits nothing.
  (4th caller `0x45FC9`, r4=0, in the previous function — noted, notDEC-output.)
- Burst cluster (`0x4618E`, sole entry `F_410A3:0x41289 jal`, VERIFIED —
  space-form grep; `0x461A1` has no entries, i.e. fallthrough-only): link gate
  `0x461A1/0x461A8` (`[r3+0x28]==1 → 0x461C4`, else `[r10+0xF0]=0` + RETURN);
  buf clears via `0x130D25` ×2; FORK RE-TEST `0x461E2..0x461F7`
  (`[r10+0x30]==1` → param select); halfword `[0x16C]` → `0x462A0`; `ble →
  0x46259`; burst loop `0x46201..0x46257`; `0x46259` (`[0x2C]==1 → 0x461AB`);
  `0x46263` fork → `0x462DE` markers (`0x1FFF/0xE800`); fallthrough markers
  (`0x7FFE/0x8001`) → `0x461AB`; `0x462A0` halfword fills; `0x462FB` returns
  `0xF1`; `0x46300+` bit-dispatch.
- Linkage (call-VERIFIED, not inferred): `F_410A3:0x41281 mov r3,r10; 0x41283
  jal F_45FD1` (sole caller of F_45FD1 ✓) so F_45FD1.r10 ≡ F_410A3.r10; then
  `0x41287 mov r3,r10; 0x41289 jal 0x4618E`. The fork VALUE stored at `0x460AE`
  is therefore visible to the `0x461E2` re-test.

H1 coverage, exact: forced ONLY `0x4614C`. Did NOT cover table arms,
`0x46156`/builder arms, F_46172/burst, refill-store, entry/arm paths — and
(doubly null) left `[r10+0x30]` intact, since the store at `0x460AE` precedes
the branch. Forcing the VALUE at `[r10+0x30]` was never tried — noted as a
future distinct experiment, not proposed.

DTS route after the size gate: post-`0x4608E` `[0x38]` is small
(`0x20..0x1C0`), so `0x200/0x400` never match → else-builder (`0x20D`) →
refill → return, NO burst. Burst requires `[work+0x38]==0x400` = carried copy
(B==1 path) or the `0x46080`-arm copy surviving to `0x460D1`.

### Q4. F_295BC chain + `+0x24` setter — both closed

- F_295BC (`0295bc..029660+`): reads `+0xC/+0x10/+0x8`, AND `+0x0` (`0x295DC`)
  / `+0x4` (`0x295DF`) as F_378A5 args; `0x295E2 jal F_378A5` (sole caller ✓);
  `[r10+0x14]` store (`0x295E6`); zeroes `+0x2C…`/`+0x80…` via `0x130D25` ×2;
  re-derives `+0x44/+0x2C/+0x28` (`0x2960E/0x29611/0x29632`, the last gated on
  `[r11+0x264/0x268]` state). Single `jal 0x295BC` at `0x29E49`, inside a
  straight-line region (only 2 prologues in `0x29000..0x29F00`: `0x29391`,
  `0x29427`; NO `jr` in `0x29391..0x29E49`, VERIFIED) with a self-`jal 0x29427`
  at `0x29EE1` (sole entry — re-entrant block, outer head above `0x29391`
  residual). Path `0x29D60..0x29E00`: ~20 word-state branches, zero codec
  immediates. Chain to leaf: init-region → F_295BC → F_378A5 → `+0x14`.
- `+0x24` setter: CLOSED as DEAD. Suffix-tolerant word sweep (trap #7):
  `0x64004`-zero, `0x64039`-zero, `0x64182`-copy, `0x6434F`-clear — no nonzero
  writer FW-wide; sub-word sweep adds only `sbz` (= zero) and far-region
  `sh_1` with no call-path to our objects. descSrc `+0x24` is therefore always
  0 → `0x460B1` never takes naturally → H1 forced a dead arm (triply null).
  The fork VALUE (`[r10+0x30]`) is downstream state, not a branch input worth
  forcing blind.

### SND side

`snd32_clean.txt` format identical (6-hex addr, `bg.jal` 8-digit); 326
`sw_0x24`-family sites, 9639 `jal`s — inventoried only. All Q1–Q4 close on the
DEC side; SND matters only if the DEC word-gates pass yet the wire stays
silent (TX stage).

### Runtime watchlist (ordered, cheapest first)

- W3 (latch gates): `[r12+0xC]@0x3DA55`, `[r10+0x18]@0x3DA6B`,
  `F_37EDC-ret@0x3DE71`, `[r10+0x4]@0x3DE78` — one of four fails for DTS.
- W1 (remaining operands): descSrc `+0x14/+0xC` AC-3 vs DTS (decides `0x4604f`).
- W2 (`+0x20/+0x24` liveness): `[work+0x24]@0x460AA` (expect 0),
  `[work+0x0/+0x20]@0x4604B`.
- Output gates: `[r4+0x0]@0x45FEB` (B), `[r4+0x10]@0x45FF1` (arm),
  `[r10+0x28]@0x46019/0x46079/0x461A1`, `[r10+0x38]@0x460D1` (==0x400?),
  `[r10+0x30]@0x461E2` (fork value).

### Residuals (all bounded, none blocking the watchlist)

- R-a: entry r4/r6/r5 (→r13/r12/r11) origins above `0x3F6A0` in F_3E961 —
  whether `[r12+0xC]` is codec-typed.
- R-b: per-codec VALUES at every watchlist point (host cannot read them: Q3
  negative stands, corroborated by your 4099-cell sweep — needs JTAG/printk/DSP
  trace).
- R-c: F_29427-region outer head above `0x29391`; F_64039-C1/C2 destination ≡
  descSrc linkage; `0x45FC9`-caller context; `[r4+0x10]` per-codec value
  (`0x3DB62` = F_37F54 return, generic).
- R-d: `0x46259`-vs-`0x461AB` loop detail (routing VERIFIED, iteration semantic
  residual); `0x130D25`/`0x130CEA` bodies assumed zero-fill/log (2 sites each
  consistent; not decoded).

Scripts: `d_entry_arms.py` (F_45FD1 entry + Q3 gaps), `d_sweep.py`
(suffix-tolerant sweeps + `jal` lists), `d_gaps.py` (arm/rout/new-cluster/tail/
F_295BC-caller), `d_setup.py` (F_3D952/latch/SETUP_B/F_295BC + callers),
`d_heads.py` (prologues, F_37EDC, F_3D952-head, callers), `d_arm1/2/3.py`
(arms, builder, burst, r11-arms), `d_final.py` + `d_micro*.py` (branch-into,
sub-word stores, F_410A3→burst) — all in
`C:\Users\k0994\AppData\Local\Temp\opencode\`.