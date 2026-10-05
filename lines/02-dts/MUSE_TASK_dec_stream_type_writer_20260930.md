# MUSE TASK — DEC firmware: who writes the stream-type field, and can DTS id be made to reach the `==5` sites?

**Date:** 2026-09-30 · **Priority:** highest — this decides whether DTS passthrough is fixable at all
**Files (all local, no device needed):**
- Authoritative listing: `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (637 122 lines)
- DEC image: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (1 982 492 B, md5 `4b7e9509…`; **file offset == fw address**)
- Read first, it is the direct parent: `C:\firmware_temp\MUSE_TASK_dec_channel_bitmap_20260930.md` → `## RESULTS`
- Also relevant: `C:\firmware_temp\aeon_validate\r57_npcm\FINDING_dec_dts_output_rootcause.md`

---

## Where the previous task left off — and what I have already verified myself

I re-verified the load-bearing claim of your last report from the raw listing: all **13**
id-gated sites are `beqi r23,0x5` on `[r12+0xC]` and **none compares 4 or 9**. Confirmed by
reading the raw lines at 0x312A5, 0x31483, 0x31D19, 0x31E96, 0x32020, 0x32171, 0x322E5,
0x32490, 0x32635, 0x3280F, 0x329C2, 0x32B72, 0x32D10. Your conclusion stands: only id 5
populates `[P+0xB0]`, the 29-iteration loop at `0x3FD25` skips every bit for any other id,
`[X+0x64C]` is never written, field `A = 0`, and the 9 KB output body at `0x44193`–`0x46508`
is unreachable for DTS.

That is also why three flashed patches returned nothing: H1 at `0x460B1`, H3′ at `0x21AD3`,
and the SND `Pc` patch together — all acted on code that never executes. **Do not re-propose
a DEC output-stage patch.**

New fact I added: the value `5` is **systemic** — 295 comparisons against it image-wide,
including a cluster inside the DTS/AC-3 codec-init `0x13C8B`–`0x140B5`. It is the firmware's
stream-type constant, not a local switch.

---

## The open question

**Who writes `+0xC` on the stream descriptor (`r12` at `0x312A5`)?**

A search for `sw 0xC(base)` across the whole image returns **zero hits**, so the field is not
stored by that instruction form or with that base register. Find it anyway.

**Q1. Locate every writer of that field, however it is written** — `sb`/`sh`/`str`, a
different base register, an `ldm`/`stm` block copy, a `memcpy`-style loop, or a struct
constructor that writes a run of fields. Report the address of each and the value written.

**Q2. Is the stream descriptor in the host-writable window?**

The 58 offsets reachable through `HAL_DEC_R2_Set_SHM_PARAM` (0xD1 params in `utpa2k.ko`
`+0x45A434`, absolute jump table at `+0x45A4C0`) are:

```
0x1014 0x1034 0x1044 0x1048 0x104c 0x1050 0x1054 0x1062 0x1074 0x1078 0x1084 0x1088
0x108c 0x1090 0x1094 0x10c4 0x10c8 0x10cc 0x10d0 0x10d4 0x10e0 0x10e4 0x10f0 0x10f4
0x10f8 0x10fc 0x1104 0x1110 0x1114 0x1118 0x112c 0x1134 0x1138 0x113c 0x1144 0x1148
0x114c 0x1150 0x1154 0x1158 0x115c 0x1164 0x1168 0x116c 0x1170 0x1188 0x1190 0x1194
0x1198 0x119c 0x11a0 0x11a4 0x11a8 0x11ac 0x11b0 0x11b8 0x11c8
```

Note `0x1014` — the window starts there, and the stream descriptor is plausibly near the front.
**Is the stream-type field among these, or above `0x1014`, or in the 0xE01000-based block that
is not a 0xD1 parameter at all?** (Recall `shm+0x118C` and `shm+0x196C` are both **absent** from
that list, so "not a parameter" is a real possibility for this one too.)

**Q3. What is the identity of the descriptor, and what else lives in it?** Trace how `r12` is
derived at the `0x312A5` reader and report the struct's base — i.e. is it
`g_virDecR2shm + k` (`MAD_base + 0xE01000 + k`) or a private heap object? If it is a SHM
struct, the offset `k` tells us whether the host can write it. State `k` explicitly.

**Q4. The decisive question — is there a single mappable point?**

If the id is written in exactly one place, then the DTS gap is either a host protocol gap
(the host sends 9 and the table has no 9) or a one-point firmware fix (remap 9 → the slot id
that the `==5` sites accept). **Report whether such a single point exists.** If the id is
computed in the SE from a capability table instead, say so and name the table — a capability
table that omits DTS would be the same build-time gap one level up.

Note the risk of the remap: if the `==5` sites select a **codec-specific** path rather than a
generic channel snapshot, mapping 9 → 5 would run AC-3 handling on DTS data. **Check what the
`==5` gate actually protects** (what `F_15F67` snapshots into `[P+0xB0]` is a channel bit-map,
but is the *selection* codec-specific?) and say whether the remap is safe in principle.

## Already solved — do not re-derive

* **Branch encoding, 100 % on all 23 273 branches.** 4-byte: signed 13-bit offset bits 3..15,
  condition bits 0..2 (`3` = `011i` unconditional). 3-byte `bn.*`: signed 8-bit offset at
  bits 10..17; no unconditional 3-byte form. 3-byte `bn.nop` = `00 00 00`.
* `shm+0x118C = *(r3+0x196C) + 0x4F1`, one writer at `0x87D49`; `0x118C` and `0x196C` are
  **absent** from the 0xD1 list.
* Host side is **fully exonerated**: the HAL knows DTS and defaults SPDIF to BYPASS+DTS, the
  decoder runs the whole track at `cmd:84`/`DecStatus:1`, the DTS bitstream reaches the decoder
  as valid IEC61937 (692 sync words, subframe `0x77`), and `HAL_AUDIO_SPDIF_BypassMode`'s
  96-entry switch sends AC-3 (format 5), DTS (9) and DTS-HD (17) to the **same** arm — the
  host does not select the codec in bypass mode. `AudioVars2+0x4F8` is only `printk`ed.

## Method notes — this file has bitten us four times

1. **Cross-check every listing scan with `grep` first.** Four separate Python regexes over
   `dec32_clean.txt` silently returned zero while `grep` found thousands. Causes: mnemonic
   field needs `.strip()`; branch targets are **eight** hex digits; the address field is exactly
   six with no assumed leading zero; and in this listing the immediate is written
   `beqi r23,0x5` with **no `#`**, so `#`-anchored patterns miss it.
2. `r3`/`r10`/`r12` are different structs in different functions. Never treat an offset as one
   global field; scope to the function's own struct.
3. Distinguish *read from the listing*, *inferred*, and *not found*. A clean negative is a
   result here.

## Rules

* Every claim cites a `dec32_clean.txt` line or a file offset in `dec_full.bin`. Quote, do not
  paraphrase.
* Q4 is the one that matters. If there is no single mappable point, say so plainly — that is
  the most valuable answer, and it closes the project honestly.
* Do not flash anything. Static analysis only.

---

## RESULTS (2026-09-30, static, listing-verified)

Refs are `dec32_clean.txt` file lines. All Python scans grep-verified first.
Fifth trip-wire for the method notes (yours `sw 0xC(base)` hit zero because the
listing writes the offset **lowercase** (`0xc`) and the mnemonic is always
prefixed (`bn.sw`/`bg.sw_0`/`bn.sh_1`/`bn.sbz`…) — `sw …0xC(` matches nothing).

### Q1. Every writer of the `+0xC` stream-type field

The field lives on **0xE01BC8-based SHM table entries, stride 0x78**
(VERIFIED below). Writers with codec-id-like constants (movi-traced, all forms):

- `F_1822B` (`0x1822B–0x1885B`), all VERIFIED same-family (explicit
  `0xE01BC8+idx*0x78` composition at each site):
  - `L28714–28717: 01874c lwz r19,0x0(r10) / muli 0x78 / add 0xE01BC8 /
    018764 sw 0xc(r19),r24 (movi r24,5 @L28714)` + `0x10(r19)` + `e0CF` flushes
    (`L28718–28722`), entered from `L28581: 0185c0 bnei r24,0 → 0x1874C`;
  - `L28724–28730`: same shape, `01878e sw 0xc(r19),r23 (movi 5)`,
    entered from `L28462: 01842e bnei [r10+0x38],0 → 0x1877B`;
  - `L28769–28786: 018800 lwz r12,0x0(r3) / muli 0x78 / add 0xE01BC8 /
    018837 sw 0xc(r12),r23 (movi r23,2 @L28785)` + flushes,
    entered from `L28747: 0187bd bnei [r3+0x38],0 → 0x18800`.
- `F_18CB0`: `L30044–30053: 019860 movi r23,4 / 019862 sw 0xc(r12),r23 /
  019865 movi 1 / 01986c sw 0x10(r12) / 01986f+e0CF flushes ×2 / j 0x1940E`,
  entered only from `L29700: 01940a bnei r24,0 → 0x19860`, where
  `L29694–29698: r24 = ([r12+0x10]≠0) ? [r12+0x10] : 1` and the
  `L29699: 019406 beqi [r16],4 → 0x196E6` divert runs first
  (the `0x196E6` arm is generic buffer/level code with an SE read
  `0xB000001E&0x100`, `L29810–29814` — not a DTS arm; the `==4` is a
  count/index compare, NOT a codec id — method note #2 applied).
  Base linkage (`r12` stack-restored `L30001`, computed `0xE01BC8+r23/r24`
  candidates `L29508–29510/29593–29601`) is residual, not VERIFIED same-family.
- `F_33516`: `L64707: 033ddd sw 0xc(r12),r26 (movi 1)`,
  `L64933: 0340a8 sw 0xc(r12),r26 (movi 1)`, `r12 = entry r3` (`L63999`) —
  same linkage residual.
- `F_11D96`: `L20251: 011f02 sw 0xc(r14),r23 (movi 5)` — residual likewise.
- `sh/sbz` 1-byte forms (`0x783C0`-family, `0xA49AF`-family, `0x8C822`-family…)
  write small ints on unrelated structs (no `0xE01BC8` composition in their
  windows) — excluded, listed for completeness.
- Clean negatives: **no `+0xC = 9` writer anywhere** (full movi-traced sweep;
  values seen `{0,1,2,3,4,5,6,7,8,0xC,0xE,0xF,−1}`); no `ldm/stm` or
  `memcpy`-loop writer of this field (no such idiom targets `0xE01BC8`
  entries — the copies observed are explicit `lwz/sw` field pairs).

Readers (the 13 id-gated sites, re-confirmed): `beqi [r12+0xC],5` at
`L61043/61202/61922/62050/62182/62297/62422/62565/62705/62860/63005/63148/63287`,
each `r23=[r12+0xC]` (`L61042`-class) with `r12=[r3+0]` (`L61018`-class);
`r3=r12` into `F_15F67`, which slots SHM by `[r3]*0x4F0+0xE01080`
(`L25575–25581`). The other 4 `F_15F67` sites are status-gated, id-free.

### Q2. Is it in the host-writable window?

No — not via any documented host mechanism. DSP-absolute address of the field:
`0xE01BC8 + idx*0x78 + 0xC` (file offset == fw address, brief). Under the
brief's own mapping (`g_virDecR2shm = MAD_base + 0xE01000`) the host offset is
`k ≈ 0x1BC8 + idx*0x78 + 0xC` (idx0: `0x1BD4`). The 58-entry 0xD1 list tops out
at `0x11C8` — `0x1BD4` is **absent** (VERIFIED by list membership), so
`HAL_DEC_R2_Set_SHM_PARAM` cannot write it: the "near the front / above
`0x1014`" hypothesis is DISPROVEN for this field; it sits in the `0xE01000`
block but outside the parameter window (same class of negative as
`shm+0x118C/0x196C`). A direct host SHM write through the mapped window is
possible in principle but no host code was examined (residual — needs
`utpa2k.ko` SHM-store analysis, out of static-DSP scope). What the DSP side
proves instead: every `+0xC` write is followed by `e0CF` write-back+barrier
calls (`L28718–28722, L28731–28737, L28788–28799, L30047–30053`), i.e. the DSP
**publishes** this field for other bus masters (host/SND/DMA can READ it;
writing needs its own path).

### Q3. Descriptor identity — SHM entry, with `k`

`r12` at the `==5` readers **is** the SHM type-entry pointer
(`0xE01BC8 + idx*0x78`), STRONGLY SUPPORTED by three independent compositions
of exactly that formula: writer side `F_1822B` (`L28709–28713`,
`L28724–28728`, `L28769–28782`), reader side (`F_3124E`: `L61018 lwz r12,[r3]` →
`L61020 lwz r10,[r12]` → `L61022–61027 muli/add 0xE01BC8`, same `*0x78`), plus
`F_18CB0` (`L29508–29510`, `L29593–29601`), all with `e0CF` cross-master
flushes. It is **not** heap-private (no heap allocation idiom feeds it; the
bases are absolute `0xE0…` compositions). Stated `k` (host view, modulo the
brief's MAD↔DSP correspondence, INFERRED): **`k = 0x1BC8 + idx*0x78 + 0xC`**.
`idx` = the stream index (`[entry]` is consumed as index elsewhere in the same
functions). Residual: absolute proof that reader-`r12` and writer-`r19/r12`
are the same entry instance (same formula + same flush discipline + same
stride, but no single pointer-flow trace end to end).

### Q4. Single mappable point? Remap safety? — the decisive answers

**No single mappable point exists — stated plainly.** The write side is ≥3
paths in `F_1822B` alone (two `=5` state gates `[r10+0x38]≠0` / `r24≠0`, one
`=2` gate `[r3+0x38]≠0`), plus the `F_18CB0` (`=4`) and `F_33516` (`=1`)
writers on linkage-residual bases — selected per-instance by stream state, not
by one table. The read side is 13 distributed `==5` gates plus 4 status gates.
There is no one address where "DTS id" is converted, stored, or omitted; the
gap is the *absence* of any DTS-membered path through both sides. No SE
capability table holding the id was found either (the nearby SE reads —
`0xB000000A&0x200` at `L61044–61048`, `&0x10` at `L25635–25639`,
`0xB000001E&0x100` at `L29810–29814` — gate block *reachability*, never the id
value; clean negative on value-flow). That closes the project honestly on the
static side: the fix is not a point but a population problem.

**Remap `9 → 5`: safe in principle, not a two-byte patch.** What `==5`
protects was verified by full-body read of `F_15F67` (`L25567–25673`): no
codec-id switch anywhere — it entry-checks (`+0x278/+0xE0/+0xDE`), does
counter arithmetic, stores `+0xA8/+0xB0/+0xD4/+0xAC`, flushes. It snapshots
whatever struct it is handed, generically. So running it on DTS data would
execute no AC-3-specific handling (nothing codec-specific exists in it).
But effecting the remap means writing `5` into DTS's SHM entry
(`k ≈ 0x1BC8+idx*0x78+0xC`) at runtime through a SHM-write primitive — i.e. a
runtime/host-side action, *not* a DEC branch patch (the three flashed patches
all died on unreached code for exactly this reason: the gap is upstream of
every patchable point). Static analysis cannot authorize or validate that
write; it names the word.
