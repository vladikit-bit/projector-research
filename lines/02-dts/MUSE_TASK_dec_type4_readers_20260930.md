# MUSE TASK — DEC firmware: does ANY code path handle stream-type 4, or is only 5 wired up?

**Date:** 2026-09-30 · **Priority:** highest — this decides the next patch
**Files:**
- `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (authoritative listing, 637 122 lines)
- `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (md5 `4b7e9509…`; **file offset == fw address**)
- Read first: `C:\firmware_temp\MUSE_TASK_utpa2k_shm_write_path_20260930.md` → `## RESULTS`
  (your exhaustive negative: no host store can reach `0x1BD4`; `init` memsets it to 0)

---

## What I found since your report — and why it changes the question

Your negative is accepted and it is sound: the host cannot write that word, so the
userspace route is closed. But the word is not left at 0 — **the DSP writes it**, at exactly
one place I verified from the raw listing:

```
019854  bg.jal 0x00130cea
019858  bg.sw_0 -0x3a2c(r11),r0,
01985c  bg.j   0x00019083
019860  bt.movi r23,0x4        | 9a e4      <<<  r23 = 4
019862  bn.sw   0xc(r12),r23    | 0e ec 0c   <<<  [r12+0xC] = 4     <-- the stream-type write
019865  bt.movi r23,0x1        | 9a e1
019867  bn.addi r3,r12,0xc
01986a  bt.movi r4,0x4         | 98 84
01986c  bn.sw   0x10(r12),r23   | 0e ec 10
01986f  bg.jal  0x0000e0cf               <<<  the publish barrier
```

This is your `F_18CB0 = 4` candidate from the previous task's Q1, now confirmed to sit on the
same field composition, immediately followed by the `e0CF` publish.

**So: DTS instances get the word `4`, and all thirteen readers require `== 5`.** `4` is the
firmware's own id for DTS — the same firmware's `decType` legend reads `4:dts`, and
`audio_status` prints `type:04` for DTS and `type:01` for AC-3 in live playback. AC-3 works,
so `5` is AC-3's word and its readers are the AC-3 path.

**That reframes your "population problem"**: it is not that no DTS value is ever written —
it is written, as 4 — but **no reader accepts 4**.

## Q1 (decisive). Does any code in the DEC image branch on the stream-type being 4?

Sweep the whole image for every comparison of a type-like field against `4`, in the manner
that established the thirteen `==5` readers. Report:

* every site that tests a value equal to 4 **on the same field family** as the `==5` readers
  (the SHM type entry `0xE01BC8 + idx*0x78 + 0xC`, and the other per-stream fields the 13
  sites read — note the sites also read `[r3+0]`, `+0x4C`-class members);
* whether any of them leads into the output path (`0x46040` decision function, builder
  `0x45D07`, the `0x44193`–`0x46508` body) or anywhere near it;
* or a **clean negative**.

This is the one question that decides the patch. If a `==4` path exists, the right fix is to
route DTS into it. If it does not, DTS has no reader at all and the only cheap option is to
make DTS instances carry the value the existing readers accept.

## Q2. If there is no `==4` reader, confirm the writer is DTS-specific

I want the "is 4 really DTS" identification nailed to the image rather than to the runtime
legend. Check the writer at `0x19860` (`F_18CB0`): what selects it, what `[r12+0x10] = 1` is
used for, and what the neighbouring `F_1822B` paths that write `5` have as their entry
condition. Specifically: **are `F_18CB0` (writes 4) and `F_1822B` (writes 5) the two arms of
one per-instance setup, keyed on the same discriminator?** If so, name the discriminator and
say whether it is the incoming host stream format.

## Q3. What would routing DTS into the `==5` path actually do?

If Q1 is negative, I am going to test a **one-byte** patch: `dec_full.bin[0x19861]` `0xE4` →
`0xE5`, so the DTS instance's type word becomes `5` and the existing AC-3 readers accept it
instead of adding thirteen `==4` sites. Before I flash it:

* trace where the `5`-only readers' downstream data ends up — does anything on that path
  **hard-code AC-3 framing**, or does the actual output format come from the stream itself?
  I need to know whether this could emit a *mislabeled* AC-3 burst (which would be worse than
  silence and would also break the AC-3 control), or whether it plausibly lets the existing
  DTS burst builder run.
* state plainly whether you think this is likely to work, and what the failure modes are.

## Context you can rely on

* DTS bitstream is valid and complete: 2.1–5.0 MB per run, 692 × `7F FE 80 01` sync words,
  subframe `0x77` (DTS), measured with `dump_es` during verified playback.
* The DTS hardware decoder runs the whole track: `cmd:84`, `DecStatus:1`, identical to AC-3.
* The optical TX emits 0 B for DTS and ~2.1 MB for AC-3 in the same state.
* Three earlier two-byte patches (forcing the `0x460B1` fork, NOPing the licence clobber at
  `0x21AD3`, and the SND `Pc` patch) all returned nothing — because they act on code inside
  the `0x44193`–`0x46508` body, which this gate never lets DTS reach. Patched DSP images load
  and roll back cleanly; that is a safe patch class.
* Host side is exonerated: the HAL knows DTS and defaults SPDIF to `BYPASS`+`DTS`; bypass
  mode does not select the codec (AC-3 format 5, DTS 9, DTS-HD 17 share one arm of a 96-entry
  switch); `HAL_AUDIO_SPDIF_SetOutputType` is `bx lr`; `AudioVars2+0x4F8` is only `printk`ed.

## Tooling note that will save you time

`reko.exe` is at `C:\Program Files\jklSoft\Reko\reko.exe` and decodes the AEON MAC group
that the public Ghidra slaspec leaves blank:

```
reko.exe disassemble --arch aeon --data <HEX>
```

`<HEX>` is a slice of `dec_full.bin` as uppercase hex; output lands in `image.reko\image_code.dis`.
It names `__amul`, `__amac`, `__amsb`, `__amacr_ex`, `__loop?(n, count)`, `__trap`,
`__move_to_spr`. **`--entry` and `--base` are rejected outright for raw images; a whole-image
run decodes only the 110-line entry stub.** Use `--data` on small slices.

## Method notes

Six regexes over these listings have silently returned zero while `grep` found thousands.
Immediates are written **without `#`** (`beqi r23,0x5`); some mnemonics carry a **trailing
comma**, so never anchor with `$`; DEC listing offsets are **lowercase** (`sw …0xc(`, not
`0xC`); the address field is exactly six hex digits. State *read / inferred / not found*.

## Rules

* Every claim cites a `dec32_clean.txt` line or a `dec_full.bin` offset.
* Q1's negative and Q3's blunt assessment are both first-class answers. I would rather be
  told "routing DTS into the 5-path will mislabel it as AC-3" than left guessing.
* Do not flash anything. Static analysis only.

---

## RESULTS (2026-09-30, static, listing-verified)

Line refs are `dec32_clean.txt` file lines. Sweep method for Q1: every
`beqi/bnei rX,0x4` (bg 4-byte; `bn.*` 3-byte forms carry no `0x4` target in the
sweep window — checked), then 10-insn lookback for `rX = +0xC(rY)`, then
double-deref shape (`rX = [rZ]` first), then per-function `jal`-target scan
against output ranges. One self-correction below (prior turn overstated an
`F_18CB0 → F_1822B` call edge — the sole `jal 0x1822B` is at `0x19993`, outside
`F_18CB0`).

### Q1. `==4` readers — exist, but none feeds the output chain

Same-field-family hits (loaded-pointer `+0xC`, the `==5`-reader shape) — **two**:

- `L48690–48697 @0x27DD8–0x27DE7` (in `F_2780D [0x2780D,0x27DF9)`):
  `lwz r23,0x0(r3) / mov r24,r3 / lwz r23,0xc(r23)` then an explicit
  three-way classifier: `L48693 beqi r23,4 → 0x27DF5 (j 0x27BD7)`,
  `L48694/48695 beqi r23,0 / beqi r23,5 → 0x27DEF (join)`, else
  `L48696 movi r3,1 + jr` (valid-type predicate returning 1).
- `L48885–48888 @0x2802C-region` (in `F_27F6F [0x27F6F,0x28248)`):
  `lwz r23,0x0(r11) / lwz r23,0xc(r23) / beqi r23,4 → 0x2823C`,
  and `0x2823C` is `mov r3,r11 + jal 0x27BD7` (`L49044–49045`) — both `==4`
  arms converge on the `0x27BD7` header-staging block.

What the `0x27BD7` arm does (`L48537–48589` read in full): SHM table walk
(`r23 = 0xE01E10 + [[r3]]*0x3C0`, `lbz 0x13C`), then burst-header staging
(`0x2D8/0x310/0x2E4/0x2E8/0x2DC/0x2E0` stores) — same code shape as the
`==5/==0` join at `0x2797C` (`L48352–48389`). Its `jal` set
(`0x132CD/0x14B3F/0x130CEA/0x130D25` only) contains **no** bitmap, gate,
builder, or decision call; both arms return into their callers. `F_2780D`
itself has no direct `jal` caller (invoked via `jalr`/table — residual which
table), same for the exact `r3` provenance (residual).

Other `==4` sites (single-deref, decoder-core/state families — listed so the
sweep is auditable): `0x1365D→0x13823`, `0x13B5F→0x13D17` (the known
`decType` init dispatch, `F_13B08`), `0x1B440→0x1B465`, `0x239E2→0x23E01`,
`0x23F75→0x23D3C`, `0x27F8A→0x281B0` (same `F_27F6F`, `+0xC(r13)` variant),
`0x3CE0C→0x3CE49`. Per-function `jal`-target scan for all nine containing
functions shows **zero calls** into bitmap (`F_15F67`), gate (`0x440F9`),
body (`0x44193–0x46508`), builder (`0x45D07`), or decision (`0x46040`-family);
all nine `==4` branch targets are intra-function. Two 3-byte-form
`bn.beqi r23,0 / bn.bnei` neighbours exist but no 3-byte `==4` compare was
found in the sweep range (stated, not assumed).

**Q1 verdict: a `==4` path that reaches the output chain does not exist
(clean negative on reach); `==4` handling exists only as decoder-core init
dispatch and `F_2780D/F_27F6F` header staging, both stranded upstream of the
gate.** The staged DTS headers go nowhere while the gate stays shut —
consistent with "decoder runs, headers staged, wire silent".

### Q2. The `0x19860` writer, nailed to the image

- Selection (all read): single predecessor `L29700: 01940a bnei r24,0 →
  0x19860`, with `L29694–29698: r24 = ([r12+0x10]≠0) ? [r12+0x10] : 1`
  (`lwz/sfnei/cmovsi_2/extbz`) and the prior divert `L29699: 019406 beqi
  [r16],4 → 0x196E6` (`L29695: lwz r23,0x0(r16)`; the `0x196E6` arm is generic
  buffer/level code with an SE read `0xB000001E&0x100`, `L29810–29814` — the
  `==4` there is a count/index compare, not a codec id; method note #2).
- `[r12+0x10] = 1` is a **one-shot trigger**: gated nonzero (via the `r24`
  normalization above), consumed once, then overwritten with `1` and
  `e0CF`-flushed (`L30045–30053: movi / sw 0x10 / jal e0CF ×2 / j 0x1940E`).
- `F_18CB0` vs `F_1822B` are **not** two arms of one discriminator
  (correction of my prior-turn phrasing): `F_1822B`'s only caller is
  `0x19993` (`L30142`, in `F_198BD`, args `r3=r10, r4=[[r13]]-byte`), reached
  after the `F_1760D` init call at `0x19981`; `F_18CB0` is called from
  `F_198BD` at `0x19ABE` (gates: `[r10+0xC]≠5` survives `L30224: beqi →0x19E78`
  divert, size/state gates, `[r10+0x4]==4` survives `L30234: bnei →0x19A3C`;
  `r3=r10` at `L30239`) and at `0x1A014` (sole gate mapped:
  `L30516: bnei [r10+0x248],0 → 0x1A012`; deeper provenance residual).
  All three call sites (`0x19ABE/0x1A014/0x19993`) sit in **one** function,
  `F_198BD [0x198BD,0x1A1BE)` (VERIFIED bounds) — i.e. one per-stream
  dispatcher routes AC-3 (`[C]==5` divert) around `F_18CB0`, routes DTS-ish
  (`[C]≠5`, `[+0x4]==4`) into it, and separately sequences init +
  `F_1822B` (whose internal `[r10+0x38]/r24/[r3+0x38]` gates then decide the
  `=5` overwrites).
- Host-format link: the ultimate writers of this dispatcher's `[r10+0xC]` /
  `[r10+0x4]` selector words are unmapped (residual); the chain is quoted
  exactly so a runtime read (`[r10+0xC]`, `[r10+0x4]` at `0x19A86/0x19AA0`
  during DTS playback) can close it. Triangulation that `4` is DTS (image
  side): the writer exists and fires on the non-5 path; `5` flows through the
  working AC-3 path; `9` is written nowhere (prior movi sweep).

### Q3. Forging `5` — blunt assessment

- **Patch bytes VERIFIED**: `dec_full.bin[0x19860:0x19862] = 9A E4`
  (`movi r23,4`; cross-checked `movi r23,1 = 9A E1`, `md5 = 4b7e9509…`);
  `0xE4 → 0xE5` is exactly `movi r23,5`; 2-byte insn, no overlap; file
  offset == fw address per brief.
- **`r4` does not derive from the forged word** (the core safety result):
  `r4` is selected inside the body from `[structB+0x38]` exact-matches —
  `L89122–89127: ==0x200 → r4=0 (@0x4616A), ==0x400 → r4=1 (@0x46186),
  ==0x800 → skip`, else fall-through `r4=2 (@0x460EF)` after the size gate
  `L89117–89121` — and the fork/builder region contains **no SHM read**
  (sole `movhi` in `[0x45FD1,0x4618E)` is the `0x15` jump table,
  `L89086`; builder window `[0x45D07,0x45DDE)` read in full, no `0xE0`
  composition). The `==5` path itself carries no codec-specific writes
  (`F_15F67` body: entry checks, counter arithmetic, generic stores).
  **Forging `5` therefore cannot force AC-3 framing via `r4`.**
- **What `r4` will actually be:** `[structB+0x38]` is entry-zeroed
  (`L89047: 045fe6 sw 0x38(r3),r0`) and its only writer in `F_45FD1` is the
  `L89104–89108` loop (`r11=(r11+1)<<5`, `r11` = `F_643DC` return ≤ `0xD` by
  the `L89085 bgtui` guard) — i.e. small values `{0x20…0x1C0}`, never
  `0x200/0x400/0x800`. Consequence (INFERRED, stated): at runtime `r4`
  resolves identically for both codecs — `2` when the size gate passes
  (`0x20D` at `0x170`), tail otherwise. AC-3 provably ships with whatever
  this yields, so a forged DTS takes **the same framing path AC-3 takes** —
  no AC-3-only hard-coding exists downstream of the forged word. (Side
  correction to prior shorthand: burst data-type goes to `0x170`
  (`0x10C/0x0B/0x20D`), while `0x16C` carries the `-0x78E/0x4E1F`
  (`0xF872/0x4E1F` Pa/Pb) constants — `L88833–88858` read in full; `r4=0`
  fires at *init* via `F_45E52` (`L89034–89035 movi r4,0`), not at runtime.)
- **Failure modes, ranked:** (i) *works* — if the size gate
  (`ble [0x38]<<5, [0x8]+0x80`) and the fork (`s->0x24==0`) pass for DTS
  with decoder-fed counters (decoder runs, `DecStatus:1` — plausible), the
  wire gets `0x20D`-labeled DTS-payload bursts; whether a receiver accepts
  them is a wire/spec question statics cannot close (residual — compare
  against a known-good DTS `Pc` or watch an analyzer); (ii) *silence
  persists* — if fork/size still divert on DTS-private state (the two
  named words are the next verification targets, both readable at runtime);
  `B`-latch state is irrelevant on the `A≠0` entry; (iii) *mislabeled burst*
  via `r4` is **excluded by construction** (above); payload garbage is
  unlikely (decoder output verified valid `dump_es`).
  **AC-3 control cannot regress from this patch**: the mapped `F_18CB0`
  caller diverts `[C]==5` around it (`L30224`); the unmapped caller
  (`0x1A014`, gate `[r10+0x248]≠0`) is the residual — but any working stream
  traversing `0x19860` in stock would already be clobbered `→4`, so either
  AC-3 avoids it or `F_1822B` re-establishes `5` downstream; the patch only
  ever turns `4→5`, never the reverse. Single-active-stream operation
  removes cross-contamination between test runs.
- **Likelihood: moderate, and diagnostic either way.** The patch is a probe
  in the `H1/H3′` class (safe: loads/rolls back cleanly): nonzero
  `dump_spdif_npcm` proves the chain opens through the wire (then analyze
  `Pc`/payload); continued zero isolates the block to fork/size-gate,
  downstream of everything patched so far.
