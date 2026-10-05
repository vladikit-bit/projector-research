# MUSE TASK — DEC ring: why is DTS's ring "remaining" zero, and who fills it?

**Date:** 2026-09-30 · **Priority:** highest — this is now the only open lead
**Files:**
- `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (authoritative listing)
- `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (md5 `4b7e9509…`; file offset == fw address)
- `C:\firmware_temp\patch_baseline\utpa2k_stock.ko` (host side, for Q4)
- Read first: `C:\firmware_temp\MUSE_TASK_dec_type4_readers_20260930.md` → `## RESULTS`
  (its Q2 recorded, marked INFERRED: `[r10+0x42C] = (F_643DC(r12,1) ≠ 0) ? 1 : 0`, `F_643DC`
  being a pure ring-remaining query, and "AC-3 shows nonzero remaining, DTS zero")

---

## Where the project stands after the device test

The one-byte patch `dec_full.bin[0x19861] 0xE4→0xE5` (forging `5` for the DTS instance) was
flashed, tested and reverted. Result, with verified playback and an early capture window:

```
AC-3 control : dump_spdif_npcm = 2 119 680 B   (no regression)
DTS          : dump_spdif_npcm =       0 B
DTS          : dump_es            = 2 080 408 B   (decoder alive)
DTS          : 1 106 x [HAL_DEC_R2_SetCommInfo] in a 124 246-line debug_level=5 capture
```

Forging `5` did not open the wire, so **the block is excluded from the entry gate and the bit
map** and is isolated to the **fork (`s->0x24`) or the size gate**. The fork is already
negatively probed — forcing it to fall-through was a null result — so **the size gate is the
remaining candidate**:

```
045FD1 region : ble ([0x38]<<5), ([0x8]+0x80)     ; the size gate
[0x38]        : written by the loop at L89104–89108,  r11 = F_643DC() <= 0xD  ->  {0x20 … 0x1C0}
```

**So: if the DTS ring reads empty, the size gate can never pass and no burst is ever built.**
That is one level above anything patched so far, and it is consistent with every observation:
the R2 decoder runs, the elementary stream is valid, and every output-side patch is null.

---

## Q1 (highest value). How does `F_643DC` compute "remaining"?

Read `F_643DC` in full. Report exactly what it reads — which structure, which fields, what
arithmetic — and **what makes it return non-zero for AC-3 and zero for DTS**. If the answer
is "it depends on a value written elsewhere", name every writer of that value and say whether
that writer is codec-conditional. Follow the chain all the way to a leaf.

## Q2. Who fills the ring, and is the producer codec-conditional?

`F_643DC` counts something. Find the producer: the code that inserts entries into whatever
`F_643DC` counts, for both the DTS path and the AC-3 path. **State whether DTS and AC-3 reach
the same producer, or whether DTS is diverted earlier.** If diverted, name the branch and
what it tests — that branch is the block, and it would be the first thing to patch.

## Q3. Is "remaining" observable from outside the DSP?

Concretely: can the host, or any `/proc/utopia_mdb/audio` command, read the value `F_643DC`
returns, directly or through a counter? Check in this order and report what you tried:

1. the mdb command table (`help` lists ~93 commands; `reg_bank=0xNNNN` reads 256 bytes of
   hardware register bank NNNN) — is there a bank that exposes this word?
2. `HAL_DEC_R2_Get_SHM_PARAM` in `utpa2k.ko` — which offsets does it read, and is the ring
   count or its base among them? (You established its 0xD1 table tops out at `0x11C8`; the
   ring structures live at `0xE01E10`-class addresses, i.e. *above* that window — confirm
   whether any read path reaches them.)
3. `read_dsp_sram_type` / `write_dsp_sram_type` (cell-indexed DSP SRAM, range 0–0xFFFF on
   device) — can the ring base and count be expressed as a cell index?

**If the answer is "not observable from userspace", say so plainly.** I will then need a
different measurement strategy and your negative saves me building the wrong probe.

## Q4. The host side of the ring

In `utpa2k.ko`: is the ring opened, sized or armed differently per codec? Look for the code
that arms the DEC ring for a new stream — does it have an AC-3 branch and a DTS branch, and
if so what do they differ in? A codec-conditional arming here would explain an empty DTS ring
without any DTS-specific code existing deeper down.

## Already established — do not re-derive

* Entry gate, proven by control-flow trace: the type word is read at `0x190B5`
  (`lwz r23,0xc(r10)`); `beqi r23,0x5,0x1914A` at `0x190B8`. The `!=5` arm restores the
  frame and returns (`0x190F0 bt.jr r9`); the `==5` arm restores and **tail-calls
  `j 0x15F67`** at `0x19180` (the channel bit-map populator). That is the whole mechanism, and
  forging `5` did not change the outcome.
* The `=4` tag write is at `0x19862` (`sw 0xc(r12),r23`, `movi r23,4` at `0x19860`); inside
  that function `0xc(r12)` is accessed exactly once — the write — and the two `==4` tests at
  `0x19406` / `0x19434` read `[r16]` and `[r8+0x16B0]`, not the tag. Patch blast radius: one
  word. Proven, do not re-open.
* `r4` in the fork/builder is selected from `[structB+0x38]` and the fork/builder region reads
  no SHM word, so a forged type cannot force AC-3 framing there. Your own result.
* Three patches have now been flashed and all returned null: H1 (`0x460B1` fork forced),
  H3′ (`0x21AD3` licence clobber NOPed), and the tag patch (`0x19861` `4`→`5`). All reverted.
  **Stop proposing output-stage patches — that stage is downstream of the block.**

## Tooling note

`reko.exe disassemble --arch aeon --data <HEX>` (at
`C:\Program Files\jklSoft\Reko\reko.exe`) decodes the `0x28`–`0x2F` MAC group that the Ghidra
slaspec leaves blank, and writes `image.reko\image_code.dis`. `--entry` and `--base` are
rejected for raw images; feed small slices as hex.

## Method notes

Six regexes over these listings have silently returned zero while `grep` found thousands.
Immediates are written **without `#`** (`beqi r23,0x5`); some mnemonics carry a **trailing
comma** so never anchor with `$`; DEC listing offsets are **lowercase** (`sw …0xc(`, not
`0xC`); the address field is exactly six hex digits.

**Function-boundary discipline — I got this wrong three times and it produced two false
conclusions.** A `jr r9` can be the *end* of a fragment, not a prologue; `0x190F0` turned out
to be a return, and code after it belonged to a tail-callee. Derive entries from `jal` targets
and from the instruction after a `jr r9` only after confirming the `jr` is 2 bytes with no
padding, then verify with a walk that starts at a known `jr`. Model calls *and* returns.

## Rules

* Every claim cites a listing line or a binary offset.
* Q3's negative is a first-class answer and saves me from building a probe that cannot work.
* Do not flash anything. Static analysis only.

---

## CORRECTION INPUT added 2026-09-30 — the "empty ring" premise is now doubtful

Runtime A/B, same 12 s window, same state, `debug_level=5`:

```
                total lines   HAL_DEC_R2_SetCommInfo   AUR2   HAL_MAD   dump_es
DTS               124 246                1 106         1 371   3 465   2 080 408 B
AC-3              105 780                  340           604   1 368     529 920 B
```

**DTS drives the R2 ~3.3x harder than AC-3 and moves ~4x the elementary-stream bytes, while
emitting zero bytes on the optical TX.** An *empty* ring would mean DTS does **less** downstream
work, not more. So the premise in this brief — that `[r10+0x42C]` is zero for DTS because the
ring is empty — **needs re-testing before you work on it.** Two readings remain and Q1/Q2 must
separate them: the ring is fine and the block is purely the last stage, **or** DTS over-runs a
ring nobody drains (high upstream traffic, `remaining` still zero).

Treat these numbers as a constraint on your answer, and say which reading the static evidence
supports.

---

## RESULTS (2026-09-30, static only)

Conventions: `F_643DC @0x642DC` etc. are AEON addresses = file offsets in
`dec_full.bin`. `bg.jal 0x000643dc`-style caller counts are exact-match greps over
`dec32_clean.txt` (lowercase, 8-digit). Codec-blind = body contains no compare on a
codec-type value (absence VERIFIED by full-body reads, not by sampling).

### Q1. `F_643DC` — pure arithmetic leaf, codec-blind, data-fed

`F_643DC @0x642DC` (4 insns, no branches, returns in r3 — VERIFIED `0642dc..0642ee`):

```
r24=[r3+0x0]; r23=[r3+0x20]; r3=[r3+0x14]
r23=r24-r23; r23>>=2 (srai); r3=r3-r23; jr
```

`remaining = [0x14] − ([0x0]−[0x20])/4`. Entry r4 is ignored (body never reads r4).
343 `bg.jal 0x000643dc` callers, incl. `F_410A3:0x41152` (r4=7) and `F_45FD1:0x4602F`
(r4=7) / `0x4604B` (r4=4). Related leaves in the same family (all codec-blind):
`F_642F0 @0x642F0` (`[0x14]−[0xC]`, 1 caller `0x4112B`), `F_642FB` / `F_64325`
("remaining == capacity?" + flag compare; callers `0x47A10` / `0x47CC3` only).

Inputs arrive via `F_64149 @0x64149` — VERIFIED 10-word copy
`[r3+0x00..0x24] ← [r4+0x00..0x24]` (`064149..064185`, jr at `064185`), i.e.
structB ← descSrc **including +0x24**. 13 callers: `F_410A3:0x41191` (src `[r23+0x28]`)
/ `0x4119E` (src `[r23+0x38]`), `F_45FD1:0x4600F` (entry, src `[r4+0x28]` = descSrc)
/ `0x46083` (arm-path re-copy), `0x673A8` + 8 more. descSrc link is validity-gated:
`F_45FD1:0x46016/0x46033` `bnei [r10+0x28],1 → 0x45FFA/0x46073` (bail).

Per-word writers (structB ← descSrc copy; then descSrc-object writers):

- `+0x14` (capacity): only producer found is `F_295BC:[r10+0x14] = F_378A5(type,…)`
  return (`0x295E6` store). `F_378A5` is codec-AGNOSTIC: entry type is stored once
  (`sw 0x7B04(r14),r13`) and never compared in the body; the `0x37BDA+` compares
  test reassigned loop temps, not the type; return is a table-walk status
  (0 / 0x80 / 0x100-flags). Linkage `F_295BC.r10 ≡ descSrc` is RESIDUAL (R3), but
  no competing `+0x14` producer exists and the movi-sweep shows no large capacity
  constant anywhere (only 0/1/2/8/9/−1) — capacity is computed, not flashed.
- `+0x0` (base): B-latch writer `@0x3DE7C` (`F_3D952`): written only when
  `F_37EDC==1 && [desc+0x4]==1`; DTS never satisfies this → `+0x0` stays 0 for DTS.
  Pairing `F_45FD1.r4 ≡ F_3D952.r10` is STRONGLY SUPPORTED: `F_45FD1` consumes
  exactly `+0x0/+0x14/+0x28` = the three words `F_3D952` produces.
- `+0x28` (link/bitmap): writer `@0x3DB21` (same pairing).
- `+0x38` (size): written in the `F_45FD1` entry region (brief L89104–89108,
  values `{0x20…0x1C0}`); read at the size gate `0x460B8` and the `0x200/0x400`
  tests `0x460C6/0x460CD`.
- `+0x24` (fork flag): carried by the copy; cleared by `F_6434F @0x6434F`
  (`[r3+0x24]=0`, 2 call sites: `F_45FD1:0x4614E`, `0x4690D`); nonzero setter
  unscoped — RESIDUAL (R2).
- `+0x20`: no scoped writer — RESIDUAL (R1, 421 FW-wide `sw 0x20` sites).

So "what makes it non-zero for AC-3, zero for DTS": the arithmetic is fixed; codec
identity enters ONLY through stored words (`+0x0` = 0 for DTS via the B-latch,
`+0x14` capacity, `+0x28` link, `+0x8` avail balance). The zero/non-zero *values*
are runtime — the earlier `[r10+0x42C]` claim stays INFERRED. No `beqi type`
exists anywhere in this chain (VERIFIED absence across every body below).

### Q2. One shared producer/consumer/refill; no DTS divert branch exists

- Producer (put n bits): `F_644F6 @0x644F6` (`[0x8]−=r4, [0x4]+=r4`, word-ptr
  advance; `0644f6..064520`, jr `064520`). 54 callers: `F_410A3:0x4114A` (r4=0x27),
  `F_45FD1:0x46027` (r4=0x27) / `0x46043` (r4=0x14), `0x466D0/0x466E2`…. Codec-blind.
- Refill (avail +=): `F_64359 @0x64359` (`[0x4]+[0x8]→[0x8]`) + `F_64296 @0x64296`
  merge helper. 10 callers of `359`: `F_410A3:0x4113E/0x41177`,
  `F_45FD1:0x460FB`…. Codec-blind.
- Consumer (get): `F_64369 @0x64369` (`r4>[0x8]` → `[r5]=0,r3=0` EMPTY; else
  extract, `[r5]=bits, r3=1`). Exactly 2 callers, both `F_45FD1` tail
  `0x46210/0x4621E`. Codec-blind.
- Fork: `F_64354 @0x64354` is a pure load `r3=[r3+0x24]`. 3 callers, all branch on
  the *stored word*, never on type: `F_410A3:0x41123` (r11=`+0x24`; `0x41142`
  `bnei r11 → 0x410FD`), `F_45FD1:0x460AA` (`[r10+0x30]=fork`; `0x460B1` H1-site
  `bnei → 0x4614C`), `0x46744` (unexamined, R4).
- `F_45FD1` remaining-switch: `0x4604B` remaining → `bgtui r3,0xD → 0x46000`
  (over-full re-entry); else computed jump `0x46053–0x46063` via table `@0x14B7FC`
  (base VERIFIED: `movhi r23,0x15; addi −0x4804` at `0x46053–57`). Table: 14 words,
  on-disk bytes `00 60 04 00 …` (BE-read gives `0x00600400`; LE decode gives
  14/14 sane in-range targets — mapping VERIFIED by target sanity, mechanism
  residual R5): rem `{0,4,5,9,10} → 0x46000` (loop-head re-test = SPIN, not drop),
  rem `13 → 0x46065` (store `0xBB80 → [r10+0x78/0x7C/0x80]`), rem
  `{1,2,3,6,7,8,11,12} → 0x4610C–0x46144` arms (`0x46144`: `ori 0x1F40` arm seen).
- Size gate (fork==0 fallthrough, `0x460B5–0x460C2`):
  `ble ([r10+0x38]<<5),([r10+0x8]+0x80) → 0x460F3` (refill); else `0x200/0x400`
  tests → `0x46156`+ arms (UNPROBED, R4).
- H1-null explained: `0x4614C` = `jal F_6434F` (clear `+0x24`) → `j 0x460F3`
  (refill). Forcing fork-taken on a zero `+0x24` is a no-op — and since forcing it
  *unconditionally* still changed nothing, DTS either never reaches `0x460AA`
  (blocked at entry `[r4+0]/[r4+0x10]` tests or the `[r10+0x28]` link bail —
  favored, given B-latch = 0) or its refill state words are also bad. Either way
  the fork is excluded, and the block is NOT at/after `0x460B1`.

Q2 verdict: DTS and AC-3 reach the SAME producer, consumer, refill, remaining and
fork code — there is no divert branch to name or patch. Codec dependence in this
layer is data-only (object words, written upstream). "Diverted earlier" as a code
branch: DISPROVEN for the ring layer; as word state: the divert is the B-latch
writer `@0x3DE7C` + its preconditions (`F_37EDC==1`, `[desc+0x4]==1`).

### Q3. Not observable from userspace — three closed negatives

1. `reg_bank`: `Dump_RegBank @0x411758` (+0x320, full body read): mallocs `0xA00`,
   loops 16 banks (`r7`: 0→`0x100` step `0x10`), per-bank `0x28`-B reads via Abs
   helpers, `strh` results out. Bank→base mapping not decoded (R6). Bounded
   negative: banks are HW register banks; the ring structs live in DSP DRAM at
   runtime addresses — no evidence any bank maps them.
2. `Get_SHM_PARAM` (`@0x45B7C4`, dispatch `0x45B858 ldr pc,[r2+r7,lsl#2]`, table
   `@0x45B884`, `0xC0` ids, default `0x45B850`): ALL handlers `0x45BBFC–0x45C084`
   decoded. Read offsets (SHM `MAD+0xE01000 + fp*960 + OFF`): `0xE14/0xE1C/0xE18/`
   `0xE4C/0xE68/0xE74/0xE6C/0xE70/0xE7C/0xEC8/0xECC/0xE40/0xE44/0xE50/0xE54/0xD90/`
   `0xFB8/0xFBC/0xFC0/0xFC4/0xFD8/0xFEC`, bitfields `@0x10E0` (bits 2–8,10),
   tail-jump reads (via `0x45C080`) `0x10D4/0x1000/0x1014/0x1044/0x104C/0x1050/`
   `0x1054/0x10E8/0x10F0/0x1104/0x1144/0x116C/0x1090/0xEC4/0x1198/0x119C/0x11A0/`
   `0x11A4/0x11A8`. Maximum ≈ `0x11A8`. Ring words not in set — CLOSED negative
   (complete enumeration).
3. `read/write_dsp_sram_type`: backend `_HAL_MAD2_DBG_CMD_Read_DSP_sram @0x467788`
   (full body): type (`r1`) must be 0/1 else return 0; type 0 requires addr ≤
   `0x2800`; programs SE IDMA `@0x112DDC`, returns the 32-bit word. Cell reader
   `HAL_MAD2_Read_DSP_sram @0x467138`: 16-bit cell → SE-IDMA byte reads at the
   `0x112A??` port. Writer `MDrv_Write_DSP_sram @0x423298`: confined to SE
   `0x112A82+~0x40`. The ring (DSP DRAM, runtime address) is not expressible as a
   cell; these windows are SE SRAM — CLOSED negative.

Primitive verdict: NONE cleanly exists from the host (no SHM read ≥ `0x11A8` /
write ≥ `0x1588`; no `reg_bank`→DRAM; no `sram`→DRAM). Nearest non-clean options:
in-band driver verbs with side effects, or a DSP-side patch (auth-gated). Do not
build a userspace ring-usage probe — it cannot work.

### Q4. No codec-conditional ring arming on either side

- Host: AC-3 arms via `HAL_MAD_SetAC3PInfo` (broad: info ids `0x0E/0x0F`,
  `0x4D(u16)/0x52/0x53/0x56/0x06–0x09`, DRC `0x76/0xA2/0xA7/0xAF`, PCM
  `0xBB–0xBF`); DTS arms via `MDrv_AUDIO_SetDTSCommonCtrl` (2 bits:
  `Set_SHM_PARAM` id `0x30/0x31 → +0x1034/0x1040`, same `fp*960+OFF` math).
  NEITHER writes ring words (all OFF ≤ `0x11C8`; the `+0xC`/idx0 SHM slot `0x1BD4`
  has no `fp∈{0,1}` solution). `MDrv_AUDIO_SetDecCmd` is codec-agnostic
  (cmd `0x84 = 0x80|0x04` like every path). No AC-3/DTS ring-arming branch exists
  on the host (absence across all four bodies).
- DSP: `F_45FD1:0x46080` arm (structB build + `[structB+0x2C]=1`, satisfies H2) is
  gated on `[r4+0x10]==1`, not on codec; remaining-table arms + refill likewise.
- Q4 answer: an empty DTS ring via differential host arming is REFUTED as a
  mechanism — the host cannot touch ring words at all, and nothing arms them
  per-codec.

### CORRECTION verdict — which reading the statics support

Neither remaining *value* is statically knowable (runtime). What statics decide:

1. "A DTS-specific branch starves/drains the ring in this layer" — REFUTED
   (~15 decoded bodies + 3 orchestrator windows, zero codec tests; H1-site
   identified as clear-and-rejoin, explaining its null).
2. Bytes-per-`SetCommInfo` are codec-proportional (AC-3: 529 920 B / 340 ≈
   1.56 KB per call; DTS: 2 080 408 B / 1 106 ≈ 1.88 KB per call) — no excess a
   refill/loop-head spin would produce (both extremes, remaining 0 and >0xD,
   re-enter `0x46000`). The pipeline makes per-byte progress for DTS exactly as
   for AC-3. Reading 2 ("ring nobody drains") is disfavored — STRONGLY SUPPORTED
   by this ratio (caveat: per-frame call rates could legitimately differ;
   runtime word comparison is the closer).
3. Supported corrected premise: the ring layer is a shared, codec-blind byte
   pump; the block is either upstream word state (B-latch = 0 for DTS,
   `[r4+0x10]` arm R7, `[r10+0x28]` link) or the UNPROBED post-size-gate arms
   (`0x46156+`, `F_64369` drain outputs at `0x46210/0x4621E`, burst/SPDIF-TX).
   Runtime work must compare stored words (`+0x0/+0x14/+0x24/+0x28/+0x38/+0x8`)
   AC-3-vs-DTS — no host primitive reaches them (Q3), so this needs
   JTAG/printk/DSP-trace, not another mdb probe.

No patch proposed (per rules; also H1 analysis shows the next step is
observation, not patching). Cheapest unprobed static: `0x46156+` arms.

### Residuals (bounded, with closers)

- R1: descSrc `+0x20` writer unscoped (421 FW-wide `sw 0x20` sites). Closer: live
  dump of the word, or scope by callers of the descSrc allocator.
- R2: `+0x24` nonzero setter unscoped (378 FW-wide sites). Closer: same.
- R3: `F_295BC.r10 ≡ descSrc` linkage (call-tree). Closer: `jal 0x295BC` callers
  vs descSrc allocation (not walked this session).
- R4: `0x46000` loop-head content, `0x4610C–0x46144` arm bodies (only `0x46144`
  / `0x46065` decoded), `0x46156+` post-`0x200/0x400` arms, third fork caller
  `0x46744`. Closer: decode windows (all ≤ 0x60 lines each).
- R5: jump-table word order (LE decode validates 14/14; BE-`lwz` tension noted).
  Closer: break at `0x46060`, read r23.
- R6: `reg_bank` bank→base mapping; `Dump_RegBank` tail beyond `0x411990`.
- R7: `[r4+0x10]` writer (gates the `0x46080` arm DTS would need with B == 0).
- R8: `F_378A5` return value for the DTS instance (table-walk outcome) — runtime.

Scripts (all in `C:\Users\k0994\AppData\Local\Temp\opencode\`): `q_fork.py`
(`0x64280–0x64520` window + caller grep — fix: `bg.jal` + 8-digit targets),
`q_callers.py` (caller counts: `643dc`=343, `644f6`=54, `64359`=10, `64149`=13,
`64354`=3, `64369`=2, `6434f`=2, `642fb/64325/642f0`=1), `q_final2.py`
(`F_410A3`/`F_45FD1` windows), `q_last.py` (`F_64149` body + `sw 0x20` sweep),
`q_jtab.py` + `q_jtab2.py` (jump-table: LE decode at fileoff `0x14B7FC`),
`u_q3d.py` (DBG-CMD backend + Get table targets), `u_q3e.py` (all Get handlers),
`u_regbank.py` (`Dump_RegBank` loop).
