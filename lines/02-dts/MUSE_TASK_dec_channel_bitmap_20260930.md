# MUSE TASK — DEC firmware: why is DTS missing from the descriptor's channel bit-map?

**Date:** 2026-09-30 · **Priority:** highest — this is the root cause the runtime has confirmed
**Files (all local, no device needed):**
- Authoritative listing: `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (637 122 lines)
- DEC image: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (1 982 492 B, md5 `4b7e9509…`; **file offset == fw address**)
- Your own previous report, whose Q1/Q2 this task follows up on: `C:\firmware_temp\MUSE_TASK_dec_output_entry_gate_20260930.md` → the `## RESULTS` section there.

---

## What is already established (do not re-derive)

Runtime, on the real device, same state, both runs verified playing:

```
DTS:  dump_es 2.0–5.0 MB, valid IEC61937, 692x 7F FE 80 01, subframe 0x77
      decoder cmd:84  DecStatus:1 for the whole ~22 s track
      dump_spdif_npcm = 0 B          <-- nothing reaches the optical transmitter
AC3:  dump_spdif_npcm = 2 125 824 B
```

So the bitstream is valid, the decoder runs, the host issues freeRun, and the break is on the
way out. Your Q2 established the entry gate at `0x4413E` / `0x44144` fails because `A = 0` and
the `B` latch is never set, and that **there is no licence anywhere in that chain** — a clean
scoped negative.

## The question

You identified field A as `[[desc+0x8D4] + 0x64C]`, whose effective writer is a
**29-iteration bit loop at `0x3FE6D` over a channel bit-map at `[r20+0xB0]`**, with
`[desc+0x648]` having a single writer at `0x6A63D` (a byte store, conditional on
`[r10+0x42C] == 1`), and `Xa` nil on the DTS path.

**Q1. What builds the bit-map at `[r20+0xB0]`?**

Find every writer. Report the full table of channel-slot → bit assignments the loop consumes,
and — this is the point — **which slot is DTS and whether that slot is ever set.** If DTS has
no slot, say so and say which codec ids do have slots (you noted `0x6A61D` and `0x3FD25` map
flags to codecs, and pre-gate `r7` conditions at `0x3DD8E`).

**Q2. What sets `[r10+0x42C]`?** It is the single gate on the only writer of `[desc+0x648]`,
and if it is never 1 then `Xa` is nil and field A is structurally zero for every codec. Find
every writer of `0x42C`, and state whether it is codec-dependent. If the value differs
between the AC-3 and DTS paths, that is the gate we need.

**Q3. The `+0x9EC` flags.** You listed writers `0x3DFD4` (=0), `0x3E056`/`0x3E088`
(=`[X+0x3C]`) under flags `0xA18/A24/B94/A2C/A30`. **Which codec makes each of those flags
true, and is DTS represented in that set at all?** If the flag set has no DTS member, the
`B` latch can never be set and the whole entry gate is closed for DTS by construction — state
that explicitly if it holds.

**Q4. Cross-check against the codec-init arms.** `0x13FBB` selects slot 0 = DTS, slot 1 = AC-3.
Both arms set `desc[0x4C] ∈ {1, 0x100}` and `desc[0x00] ∈ {0x911A, 0x810B}` on a struct whose
largest offset is `0x74`, while the gate's struct reaches `0x8D4`. **Is the `0x13FBB` struct
reachable from the gate's struct, or are they genuinely different objects?** (You already
suspected different instances — this asks you to settle it.) If they are different, say which
one feeds the DEC→SND handoff.

## Already solved — do not re-derive

* **Branch encoding, 100 % match on all 23 273 branches.** 4-byte: signed 13-bit offset at
  bits 3..15, condition in bits 0..2 (`0` bles/blesi, `1` ble/bleui, `2` beq/beqi,
  **`3` 011i = unconditional**, `4` bgts/bgtsi, `5` bgt/bgtui, `6` bne/bnei, `7` b111/b111i).
  3-byte `bn.*`: signed 8-bit offset at bits 10..17, condition in the top 6 bits of byte 1;
  **no unconditional 3-byte form**. 3-byte `bn.nop` = `00 00 00` (all 25 814 occurrences).
  13 337 `jal` instructions, 3 138 distinct targets; **none lands in `0x44200`–`0x45300`**.
* **Three two-byte DEC patches were flashed and all returned nothing**, with AC-3 controls
  passing each time: H1 (`0x460B1` forced to the fall-through), H3′ (`0x21AD3` NOPed), and
  H1 + the SND `Pc` patch together. Patched DSP images load fine and roll back cleanly.
  **Do not re-propose these.** They are diagnostics, not fixes.
* `dump_spdif_npcm` taps **before** IEC61937 framing, so 0 B proves nothing reached the
  transmitter — it does **not** prove the burst builder did not run.

## Method notes that have bitten three times on this project

1. **Cross-check every listing scan with `grep` first.** Three separate Python regexes over
   `dec32_clean.txt` silently returned zero while `grep` found thousands. Causes seen: the
   mnemonic field needs `.strip()`; branch targets are **eight** hex digits, not seven; and
   the address field is exactly six hex digits with no assumed leading zero (`13fad7`, not
   `013fad7`).
2. `r3`/`r10` are different structs in different functions. Never treat `0x24(r10)` as one
   global field; scope every search to the function's own struct.
3. Distinguish *read from the listing*, *inferred*, and *not found*. Report the negative —
   two earlier "narrowing" lists in this project were mis-labelled and had to be retracted.

## Rules

* Every claim cites a `dec32_clean.txt` line or a file offset in `dec_full.bin`.
* If the answer is "DTS has no entry in the bit-map / flag set", say so plainly — that is the
  most valuable possible answer here, and it points at a build-time configuration gap rather
  than a runtime one.
* Do not flash anything. Static analysis only.

---

## RESULTS (2026-09-30, static, listing-verified)

Line refs are `dec32_clean.txt` file lines. Every Python scan below was verified
with `grep` first (per method notes); the address field is 6 hex digits, no assumed
leading zero. Status words follow the project vocabulary
(VERIFIED / INFERRED / residual).

### Q1. The bit-map at `[r20+0xB0]` — writers, slot→bit table, DTS slot

**Consumer (the loop).** `0x3FD25`-loop (`L80712–80749`): setup
`L80712 muli r10,r12,0x6`, `L80713–80715 movi r23/r25/r24,0`,
`L80716 add r26,r17,r13`; head `L80717 lwz r28,0xb0(r20)`,
`L80718 sll r27,r16,r24`, `L80719 and + L80720 beqi→0x3FD81` (skip when clear),
`L80721–80722` second mask vs `r11` (→`0x3FDE9` when clear); per-set-bit body
`L80723–80744`; `L80745–80749` counters + `sfnei r24,0x1D / bf→loop`
(29 iterations, `r24 = 0..28`). `r16 = 1` on-path (`L80635: 03fc2d movi r16,1`,
fall-through into the `0x3FC37` micro-loop and `L80643: bleui r25,4 → 0x3FCD7`,
the loop's sole entry — VERIFIED single predecessor).
Slot→bit assignment: **identity, bit k ↔ slot k, k = 0..28**, dual-masked by
the bitmap word and the `0x130E0D`-produced codec mask (`r11`, `L80702`).
No codec→bit translation exists in the loop.

**`r20` identity.** Two ways in, one coherent: (a) `L79939: 03f371 lwz r20,-0x5EE0(r10)`
(pointer `P`), preserved along `0x3F3B4→0x3FC15→0x3FC1D→0x3FC25→0x3FCD7`
(no `r20` write in that span — VERIFIED full-range scan; only writes image-wide
in `[0x3F3BE,0x3FD25)` are `L79962 movi r20,0` and `L80366 addi` counter, both
off this path); (b) the counter-exit value via `0x3F8E7` (same word then reads
near-absolute low memory — INFERRED not taken for the bitmap use, since the
body dereferences it as a struct elsewhere). `P` itself is stored at
`L79831: 03f1f2 sw -0x5ee0(r10),r4` / `L79852: 03f241 …` (`r4` = `F_38E5E` /
`F_682D4` return), null-checked `L79832: beqi r4,0 → 0x3EADD` (bail).
Same-source corroboration inside `F_3E961`: `r13 = [r10−0x5EE0]`
(`L79419/79563`-class loads) with bitmap reads `L79565/79569: 03ee61/6f lwz …0xb0(r13)`,
`L80306: 03f823 …`, `L80666: 03fc8a …`.

**Writers of `+0xB0` words.** `F_15F67` (prologue `L25569: 0x15f67`), two stores:
`L25605: 015fdd sw 0xb0(r13),r23` (`r23 = [ptr+0x10]−[ptr+0x00]`,
`L25601–25604`, `ptr = [r10+0x1D4]`, addend base `[r10+0x1E4]`) and
`L25666: 0160a4 sw 0xb0(r13),r26` (`r26 = [r24+0xC]−[r24+0x0]`,
`L25653–25665`), where `r13 = [r3]*0x4F0 + 0xE01080` — SHM table entries,
0x4F0 stride (`L25575–25581`, `L25652–25658`). Entry gates: `[r3+0x278]≠0`
(`L25585`), `[r3+0xE0]≠0` (`L25587`), `[r3+0xDE]` vs `r12` (`L25588–25590`),
and an SE gate `L25635–25639: movhi …,0xb000 / ori 0xa / lh / andi 0x10 /
bnei → 0x16195` (`0xB000000A` bit 4 must be 0). Values written are
level/counter differences (small ints), and the same function stamps
`+0xA8/+0xD4/+0xAC` plus `0xE0CF` cache ops (`L25600–25624`).
Aliasing `r13 ≡ P` is UNPROVEN (different derivations: SHM-slot computation vs
helper return) — residual; the word is consumed as a bitmap in `F_3E961`
either way.

**The 17 call sites are NOT a codec ladder.** Gate scan (45-line windows,
branch-into-entry, VERIFIED): 13 sites gated by `beqi r23,5 → mov/call`
(`L61043, L61202, L61922, L62050, L62182, L62297, L62422, L62565, L62705,
L62860, L63005, L63148, L63287`), with `r23 = [r12+0xC]` (`L61042`,
`r12 = [arg]`, `L61018`, in `F_3124E`, prologue `L61012`) and a second SE gate
`L61044–61048: 0xB000000A & 0x200 must be 0`. The other 4 sites
(`0x19664` — buffer-level gates `L29878–29885`; `0x31513/0x31744/0x31A9F/0x31BD8`
— status-bit combos `andi 0x1001/0x282` + fall-through entries) test no codec
id in their windows. No site tests id 4 or 9 (scoped negative over all 17
entry windows). `r3` passed differs per site (`r10/r11/r12/r13`).

**DTS verdict: DTS has no slot — stated plainly.** Under mdb numbering
(`1=AC-3, 4=DTS`) none of the 13 id-gated sites can fire for DTS (`4≠5`), and
under userspace numbering (`5=AC-3, 9=DTS`) the same holds (`9≠5`); the
remaining sites are status-gated, not id-gated. AC-3 works through the `==5`
sites, which additionally fixes this struct's numbering to the userspace one
(`5=AC-3`) — INFERRED (best explanation; the `5=AAC` reading would leave
*both* working codecs siteless). Consequence, VERIFIED chain: with no site
firing for DTS, `[P+0xB0]` keeps its init value (0) → the `0x3FD25` loop skips
all 29 bits → `[X+0x64C]` is never written → field `A = 0`.

### Q2. Writers of `0x42C` — full table, codec-dependence

13 writers image-wide (grep-verified): `L44695, L45686, L46412, L46951, L83197,
L137836, L138267, L138364, L138393, L138395, L149556, L312368, L379408`.
The one that matters lives in `F_69C8A` (prologue `0x69C8A`, single caller
`L135138: 067bab jal`, args `r3=r18`, `r5=r12+r29+0x81C` per
`L135132–135137`; inside: `r12=r3`, `r10=r5`, `L137853–137854`):
- `L138378–138395`: `mov r3,r12 / movi r4,1 / jal 0x643DC`;
  `06a380 sw 0x42c(r10),r0`; `06a384 beqi r3,0 → skip`;
  `06a387 sw 0x42c(r10),r11`. `r11` reaching-def on this path is
  `L138262: 06a1ef cmovsi_2 r11,r3,0x1` filtered by `L138264: bnei r11,1 →
  0x6A324` (the zero-writer, `L138364`) — so on arrival `r11 == 1`.
  Hence **`[r10+0x42C] = (F_643DC(r12,1) ≠ 0) ? 1 : 0`**.
- Reaching `0x6A378` additionally requires `L138264` fall-through
  (`r11==1`, i.e. getter value ∈ {0,1}) and `L138265–138266:
  06a1fa lhz r23,0x424(r10) ([r10+0x424] = getter(r12,8)+1, `L138253–138257`) /
  06a1fe bgtui r23,2 → 0x6A378`, else `L138267: 0x42C = 0`.
- The reader that matters: `L138610–138611: 06a61d lwz r23,0x42c(r10) /
  06a621 bnei r23,1 → 0x6A36C`, itself reached only when
  `L138385: 06a368 beqi [r10+0x63C],1 → 0x6A61D`.
- `F_643DC` ignores its `r4` argument (dec28 `L192–198`: pure
  `[r3+0x14] − (([r3+0x00]−[r3+0x20])>>2)` ring-remaining computation —
  prior-session dump, cited as such), so the `r4 ∈ {1,2,3,5,8}` immediates at
  its call sites are tags, not selectors.

**Codec-dependence verdict:** structurally the flag is **state-dependent, not
id-dependent** — a per-instance buffer-remaining predicate
(`r12` = `F_69C8A`'s `r3` = caller `r18`, per-iteration stream struct in the
`0x67BAB` loop, `L135122–135126`). AC-3 vs DTS differ here only through runtime
stream state (AC-3's getter is nonzero → `0x42C=1` → `+0x648` written;
DTS's is zero → `0x42C=0` → `Xa` nil → `A=0`). The exact runtime values are not
statically decidable — INFERRED mapping, stated as such; what would decide it
is the `r12`-struct's live words, not firmware bytes.

### Q3. The `+0x9EC` flags — codec mapping, DTS representation

The flags are **not a codec set; they are per-invocation arg latches + SE
shadows**, so "which codec makes each true" has no static id answer:
- In `F_378A5` (`L69844–69929` dispatcher on `r23 = entry r3`,
  `L69863: mov r23,r3`): each `[r23+off]==1` arm copies an incoming arg into
  the struct and stamps SE-shadow `0x7AF0` — `L69853: 037af1 sw 0xa18,r24`,
  `L69897: 037b73 sw 0xb94,r4`, `L69917: 037bb0 sw 0xa30,r4`
  (`r4` = the caller's arg). Codec enters only via whoever calls with what.
- In the `0x3DFxx` state machine the same offsets are sustained/cleared on its
  own `r3`: `L78452: 03e045 sw 0xa24,r23`, `L78477: 03e098 sw 0xb94,r23`,
  `L78543: 03e16b sw 0xa18,r0` (= clear), gated by the current
  `0xA18/A24/B94/A2C/A30` values themselves (`L78416–78494`).
- A third site refreshes them conditionally from live fields:
  `L210606–210612: [r10+0xA18]=[r10+0x9E8], [r10+0xA24]=[r10+0x9F0]…`,
  gated by `L210590: bgtsi [r3+0x70],0xb → skip` and
  `L210598: bnei [r3+0xA3C],0 → skip` — **if DTS's `[r3+0x70] > 11`, its flags
  freeze at whatever init left (0)**. Other writers: `L81205–81209`,
  `L196028–196106`, `L210607–210611` (all-zero inits), `L294399/294996/334652`
  (negative-offset, different structs).
- **DTS representation: none found.** No `+0x9EC`-family writer is gated on a
  DTS codec id, and the sustain paths require prior nonzero state that the DTS
  flow never bootstraps (its `[X+0x3C]` source arms at `L78457/78473` need
  `[X+0xA18]==1`-class preconditions). Hence the `B` latch
  (`[Xa+0x9EC]==1` + `F_37EDC==1`, prior report) can never arm for DTS —
  the entry gate is closed for DTS **by construction**, as the brief suspected.
  (Residual: live-flag semantics of `0x70/0xA3C` per codec — runtime data.)

### Q4. `0x13FBB` struct vs gate struct — settled: different objects

- **Layout proof.** Init struct (`F_13FBB r4`): every store in the full arm
  region falls in `0x00–0x74` (max `L23004/L23082: sw 0x74`). Gate struct
  (`F_3D952/F_440F9 r10/r3`): same-register accesses reach `0x648`
  (`L78069`), `0x8D4` (`L77989/78274`), `0x9EC`-family, `0x16C`
  (`L89216: 0461e5 lhz r24,0x16c(r10)`, `L89260: 046266 …`, body-scan
  VERIFIED). One object cannot span both layouts.
- **Allocation proof.** Init `r4` = static-table computation
  (`L27534: muli r24,r11,0xd8` + `L27536: addi r18,…,-0x5CD0`, `0x46`-based);
  gate `r3` = heap chain (`L80178: r7=[r11+0x290]`, `L80195: r7+=0xA4`,
  `L80201: jal`, `L77905: mov r10,r7`).
- **Pointer-flow negative.** `F_13FBB` stores into its `r10`-struct only slot
  indices/immediates (`sw 0x30,r5 / 0x34,r6 / 0x20,0x24,r0`, `movi r4,0/1`);
  the `r4` pointer itself never escapes there (full-region read, VERIFIED).
  (`F_1760D`-level escape not exhaustively swept — residual, but unnecessary
  given layout+allocation.)
- **Handoff.** The output pipeline — 9 KB body plus the single builder
  `0x45D07` (4 sites `0x45FC9/0x460EF/0x4616A/0x46186`, body-scan VERIFIED) —
  consumes the **gate-side** struct (`r10`: `+0x48/0x08/0x44/0x40/0x0C/0x2C/
  0x30/0x16C`, re-read of `+0x4C` inside the body at `L86819: 0445f3`).
  The init-side struct feeds decoder-core setup only (slot loop, builder
  `r4`-select). The DEC→SND-visible surface is the SHM table
  (`0xE01080`-based, `F_15F67` writes) alongside those in-DEC buffers —
  SHM-as-handoff INFERRED from the host `MAD+0xE01000` mapping, stated as such.
- +0x4C readers image-wide are dominated by stack spills (`0x4C(r1)`);
  non-spill readers (`0x15671→F_1565C`, `0x1FCC4→F_1F709`,
  `0x26918→F_268F6`, `0x2A7CD→F_2A7B0`, `0x2DE20→F_2DDEB`, gate `0x44135`,
  body `0x445F3→F_440F9`) touch small-struct fields of their own functions —
  none aliases the gate word (per-function scoping, no shared base found).

### Net verdict (all four questions together)

DTS is missing by construction at three independent layers, all pointing at a
**build-time configuration gap** (no DTS member in any of the three sets),
not a runtime race: (1) no `F_15F67` call site fires for a DTS id
(`13× ==5`, rest status-gated; no `==4/==9` anywhere in the entry windows);
(2) the `0x42C` gate needs nonzero buffer-remaining state the DTS stream never
shows; (3) the `+0x9EC` flag set has no DTS-sourced member, so the `B` latch
cannot arm. Nothing here is patchable in DEC firmware with a two-byte edit —
the missing piece is populating these sets for the DTS codec id upstream of
all three (host/SHM configuration or aablement tables), or proving at runtime
which single link differs and driving exactly that word.
