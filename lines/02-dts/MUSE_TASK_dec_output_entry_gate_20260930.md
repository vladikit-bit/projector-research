# MUSE TASK — DEC firmware: the entry gate that skips the whole DTS output block

**Date:** 2026-09-30 · **Priority:** highest — this is the only live lead
**Files (all local, no device needed):**
- Authoritative listing: `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt` (637 122 lines)
- DEC image: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (1 982 492 B, md5 `4b7e9509…`; **file offset == fw address**)
- Prior analysis, re-verify everything: `C:\firmware_temp\aeon_validate\r57_npcm\FINDING_dec_dts_output_rootcause.md`
- Branch-encoding reference (see "already solved" below)

---

## Why this task exists

Two on-device patches were applied and **both returned nothing** on the optical path
(DTS `dump_spdif_npcm` = 0 B, AC-3 control = 2 107 392 B, in both cases):

* **H1** — `0x460B1` forced to the fall-through so the DTS burst builder runs regardless of
  `s->0x24`. `d0 60 04 de → d0 60 00 23`.
* **H3′** — `0x21AD3` `lbz r27,(r24)` NOPed so the licence gate cannot clobber the block
  count `r27`. `13 78 00 → 00 00 00`.

**Neither helped, and the reason is now known: the code both patches act upon is never
reached.** The whole DTS output region is one ~9 KB function body entered by a *branch*
from a small wrapper, and that branch is not taken. Patched DSP images load fine and the
rollback works, so this is a safe and cheap test bed — but patching inside a block that never
executes is a null operation.

## The gate (established, please confirm and then work past it)

`0x44193`–`0x46508` is **one 9 078-byte function body with no `jr r9` inside it**. It is
entered from the wrapper at `0x440F9`, which has one caller (`0x3DDB6`):

```
0440F9  bg.addi r1,r1,-0x1d8        ; prologue, 0x1D8 frame, saves r9-r22
        …
044135  bn.lwz  r23,0x4c(r3)       ; <-- field A
04413e  bn.bnei r23,0x0,0x00044194  ; A != 0  -> ENTER the output block
044141  bn.lwz  r23,0x0(r3)        ; <-- field B
044144  bg.beqi r23,0x1,0x00044769  ; B == 1  -> ENTER the output block
044148  … register restore …
044192  bt.jr   r9                  ; NEITHER -> return, output path never runs
```

**Q1. What are field A (`r3+0x4C`) and field B (`r3+0x00`)?**

Find every writer of each in `dec_full.bin` (note: `r3` is the output-descriptor struct, and
`0x24(r3)`/`0x2C(r3)`/`0x30(r3)`/`0x16C(r3)` are already known in this struct — the prior
report calls `0x16C` the "data-type" field with `0x0B` = DTS, `0x10C` = AC-3). Report for
**both** fields: every writer, the values written, and the value chain that produces them.
Then state plainly which field discriminates DTS from AC-3 and where that decision is made.

**Q2. Why does DTS not satisfy the gate?**

We know the DTS path *is* entered elsewhere — the codec init arm for slot 0 runs
(`0x13FCA bnei r3,0x0,0x140B7`; slot 0 = DTS, `r3==0` falls through to `0x1404E movi r4,0`),
and both the DTS and AC-3 arms clear `s->0x24` (`0x14053`, `0x14149`). So find what runs
**after** those arms and sets (or fails to set) field A and field B. If a licence or
capability read feeds either field, name the exact register, shift and mask.

**Q3. What is at `0x44769`?** That is the second entry, taken when `B == 1`. Determine
whether it is the DTS burst path or a different one (AC-3? a passthrough-only path?), and
whether A and B select between two different output paths or two stages of one.

**Q4. What is at `0x44194`?** The first entry. Same question. Note both entries are inside
the same 9 KB body, so the two conditions are more likely *stages* or *codec classes* than
alternative entry points — prove which.

## Already solved — do not re-derive (this cost real time)

**Branch encoding, verified against all 23 273 branches, 100 % match:**

* 4-byte: offset = **signed 13-bit at bits 3..15**; condition = **bits 0..2**:
  `0` bles/blesi · `1` ble/bleui · `2` beq/beqi · `3` **011i (unconditional)** · `4` bgts/bgtsi
  · `5` bgt/bgtui · `6` bne/bnei · `7` b111/b111i.
* 3-byte `bn.*`: offset = **signed 8-bit at bits 10..17** (straddles byte 2 into byte 1);
  condition in the top 6 bits of byte 1. **There is no unconditional 3-byte branch form.**
* 3-byte `bn.nop` is unambiguously `00 00 00` (all 25 814 occurrences).
* 13 337 `jal` instructions, 3 138 distinct targets. **No `jal` target lies in
  `0x44200`–`0x45300`.**

**Method warning that has bitten twice already:** scanning the listing with a Python regex
has silently returned zero matches while `grep` found thousands. Two causes: the mnemonic
field needs `.strip()`, and the target is **eight** hex digits (`0x00048836`), not seven.
**Verify any search with `grep` first, then do the analysis in Python.**

## Also relevant, already established

* The DTS IEC61937 builder is complete and byte-correct; AC-3 works through the same path.
* The block is **not** at `0x460B1` (H1 falsified) and **not** at the licence clobber
  `0x21AD3` (H3′ falsified).
* `dump_spdif_npcm` taps **before** IEC61937 framing, so 0 B proves nothing reached the
  transmitter — it does **not** prove the burst builder did not run.
* No host lever exists: `shm+0x118C` and `shm+0x196C` are absent from the 58 reachable
  0xD1 `HAL_DEC_R2_Set_SHM_PARAM` offsets; `cmd 144` `SetDTSCommonCtrl` sub-commands 6..14
  were fired during verified DTS playback with a verified payload and changed nothing.
* The audio HAL already knows DTS (`SetSpdifOutputMode=BYPASS`, `SetSpdifOutputType=DTS`
  are its SPDIF defaults) and the DTS decoder really runs: `cmd:84`, `DecStatus:1` for the
  whole track. So the host layer is exonerated.

## Rules

* Every claim cites a `dec32_clean.txt` line or a file offset in `dec_full.bin`. Quote, do
  not paraphrase.
* Separate *read from the listing*, *inferred*, and *not found*.
* A clean negative is a result. Report what you searched and the negative.
* If a proposed patch targets code that is unreachable, **say so explicitly** — that is
  worth more than a patch.
* Do not flash anything. This is static analysis only.

---

## RESULTS (2026-09-30, static, listing-verified)

Line refs below are `dec32_clean.txt` file lines (`Read`-tool numbering = file lines;
verified: the file has **no** line numbers, format is `<6-hex-addr> <len> <mnem ops> | <bytes>`).
Search discipline per the brief: every Python search was verified with `grep` first;
plus one new trip-wire found this session (see Method notes).

### Gate confirm (read from the listing)

- Wrapper `F_440F9`: prologue `L86409: 0440f9 addi r1,-0x1d8`, saves r9–r22
  (`L86410–86423`), then `L86424: 044135 lwz r23,0x4c(r3)` [A],
  `L86428: 04413e bnei r23,0 → 0x44194` [A≠0 → entry A],
  `L86429: 044141 lwz r23,0x0(r3)` [B],
  `L86430: 044144 beqi r23,1 → 0x44769` [B==1 → entry B],
  else restore (`L86431–86450`) and `L86451: 044192 jr r9`.
  Entry passes `r10=r3, r11=r5` (`L86426–86427`). Single caller
  `L78243: 03ddb6 jal 0x440f9` with `r3=r10` (`L78242`).

### Q1. Fields A and B — writers, values, chains

**Field A = `[desc+0x4C]`. The only writer that can affect the gate is**
`L77989: 03da77 sw 0x4c(r10),r3` in caller `F_3D952` (prologue `L77880: 0x3d952`),
with `r3` = return of `L77988: jal 0x37ee4` fed by `L77987: lwz r3,0x8d4(r10)`.
Leaf `F_37EE4` (`L70186–70190`):
`0x37ee4 movi r23,0 / beqi r3,0→0x37eed / lwz r23,0x64c(r3) / mov r3,r23 / jr`.
So **`A = [[desc+0x8D4]+0x64C]`, or 0 when `[desc+0x8D4]` is null.**
Upstream links (all read): `[desc+0x8D4] ← [r13+0x58]`
(`L78273–78274: 03de18 lwz r23,0x58(r13) / 03de1b sw 0x8d4(r10),r23`),
with `r13 = [desc+0x648]` (`L78069: 03db8e lwz r13,0x648(r10)`; entry value
`r13=r4`, `L77903`); `[desc+0x648]` has exactly **one** writer in the image,
`L138620: 06a63d sbz 0x648(r10),r3`, itself gated by
`L138611: 06a61d lwz r23,0x42c(r10) / bnei r23,1→0x6a36c`, value =
`F_643DC(r12,5)` getter (`L138617–138620`).
`[Xa+0x64C]` writer: `L80828: 03fe6d sw 0x64c(r30),r23`
(`r30=[r17+0x8D4]+…`, `L80814–80817`), inside the `0x3FD25` 29-iteration
bit-loop (`L80717–80749`) over channel bitmap `[r20+0xB0]`
(`L80717/80719: lwz r28,0xb0(r20) / sll+and`).

**Field B = `[desc+0x00]`. Writers:** (i) init arms on the *small* slot struct
(see instance note): DTS `L22996: 014027 sw 0x0(r4),r27` with
`r27=0x11A|0x9000=0x911A` (`L22975/22977`), AC-3 `L23077: 014126 sw 0x0(r4),r27`
with `r27=0x10B|0x8000=0x810B` (`L23055/23058`); (ii) on the **gate** struct:
`L78301–78303: 03de75 lwz r23,0x4(r10) / 03de78 bnei r23,1→0x3da6f /
03de7c sw 0x0(r10),r23`, i.e. **B := 1 iff `F_37EDC(…)==1` and
`[desc+0x04]==1`**, where `F_37EDC` is the leaf `L70177–70180`
(`lwz r23,0x24(r4) / beqi… / movi r3,1 / jr`) and `[desc+0x04]` was set at
`L77985: 03da68 sw 0x4(r10),r3` = `F_37F25([desc+0x8D4])`, leaf `L70209–70213`
(`movi r23,0 / beqi r3,0 / lwz r23,0x9ec(r3)`), i.e. **`[desc+0x04] =
[[desc+0x8D4]+0x9EC]`**. `[X+0x9EC]` writers: `L78417: 03dfd4 …r0`,
`L78457: 03e056 …[r3+0x3C]`, `L78473: 03e088 …[r3+0x3C]`, gated by the
`[X+0xA18]/[X+0xA24]/[X+0xB94]/[X+0xA2C]/[X+0xA30]` flag state machine
(`L78416–78494`). Sibling callees with `r3=desc` (`0x41635, 0x414AE→0x4139A,
0x415CB, 0x430C2, 0x43510, 0x41786`) write **no** `0x00/0x4C` on the descriptor
(scoped scan, clean negative); `F_3D952` itself never writes `0x0(r10)`
(verified over full caller dump `L77880–78269`).

**Instance note (which field discriminates, and where).** The init arms write
`A∈{1,0x100}` (`L22974/22982` DTS `movi r25,1`; `L23059/23081` AC-3
`ori r28,0x100`) and `B∈{0x911A,0x810B}` — but on the **small slot descriptor**
(`F_13FBB r4` = `r18` = static-table computation `L27534: muli r24,r11,0xd8` +
`L27536: addi r18,…,-0x5cd0`; max store offset in the whole arm region is
`0x74`, `L23004/L23082`). The gate struct spans to `0x8D4`/`0x648`/`0x9EC`
writes on the same `r10`, and arrives via a heap chain
(`L80178/80195/80201: r7=[r11+0x290]+0xa4` → `F_3D952` arg → `L77905 mov r10,r7`).
Different layouts ⇒ different instances ⇒ **the init-arm A/B values are dead
with respect to this gate** (layout proof; allocation proof supporting).
The effective decision is made in `F_3D952`: **field A discriminates**
(recomputed every iteration at `0x3DA77` from live struct data);
field B is a latch (see ordering below). No `jal` targets `0x44200–0x45300`
(re-confirmed: body `jal` set = `0x3D091/0x4129E/0x41A4C/0x45D07/0x465F0…/
0x64149-family/0x130D25/0x131E33/0x132001`).

**Ordering (frame-latch model).** Within `F_3D952`, all verified in program order:
`0x3DA77` (A recompute) < `0x3DDB6` (gate call) < `0x3DE1B` (Xa update) <
`0x3DE7C` (B latch); and post-gate code jumps back to `0x3DA5D/0x3DAA5/0x3DA6F`
(`L78291/78294/78304`), all **before** the gate call — i.e. `F_3D952` iterates
and iteration *k+1* reads the A/B latched by iteration *k* (loop verified;
per-iteration varying index INFERRED, likely channel/instance).

### Q2. Why DTS does not satisfy the gate

1. Entry A needs `A≠0`, i.e. `Xa≠0` and `[Xa+0x64C]≠0`. `Xa=[desc+0x8D4]` is null
   unless the single-byte writer `0x6A63D` fired (`[r10+0x42C]==1` gate) and the
   `0x3FE6D` writer fired (channel-bit loop). Both are codec-state-gated and
   neither fires on the DTS path in the traced data (exact flag→codec mapping
   at `0x6A61D`/`0x3FD25` is the residual — see below). First iteration therefore
   reads alloc-zero A → skips entry A. The init arm's `A=1` never reaches this
   instance (Q1 instance proof).
2. Entry B needs the latch `B==1`, i.e. `[Xa+0x9EC]==1` via `desc+0x04` plus
   `F_37EDC==1`. The `+0x9EC` writers zero the field unless the
   `0xA18/0xA24/0xB94/0xA2C/0xA30` flags select the `[X+0x3C]` arms
   (`L78416–78494`); on the DTS path they select the zero arm
   (`L78417: sw 0x9ec,r0`) or never run → `desc+0x04=0` → `0x3DE78` diverts to
   `0x3DA6F` → B stays 0. (Init's `B=0x911A` is on the wrong instance.)
3. Net: for DTS both branches fall through to restore + `jr r9`
   (`L86431–86451`) — the ~9 KB body never executes, exactly matching the
   device result (H1/H3′ patched dead code; `dump_spdif_npcm` taps an
   unreached transmitter).
4. **Licence/capability: clean scoped negative.** The A-chain
   (`0x37EE4`/`0x37F25` leaves — call-free; `F_378A5` full-body read
   `L69652–70266`, all `movhi` are `0x1/0x2/0x4/0x15`, no `0xB000` composition)
   and the B-chain leaves (`0x37EDC`, `0x37EBA`), plus every dumped window of
   the outlying writers (`0x3DFD4–0x3E0D2`, `0x3FD25–0x3FE84`, `0x6A5FA–0x6A673`),
   contain **no SE/licence read** — no register/shift/mask to name. Residual:
   un-dumped callees behind those windows (`0x39A64/0x40408/0x4650A/0x643DC…`)
   were not SE-swept.
5. Additional suspects (not fully traced, stated as residual): pre-gate bails
   `L78230–78238` (`r7=[[[r14+0x174]+1]−0x619C]` must be 0, else `→0x3DBF5`;
   then the `r23+r24≤1` test) can divert DTS before the gate is even evaluated.

### Q3. What is at `0x44769` (entry B)

A validation preamble with tag `r14=1` (`L86951`), not a codec path:
`trap; jal 0x41A4c; beqi r3,0→0x44154` (abort straight into the wrapper's
restore/return, `L86435`), `lwz 0x48(r10); beqi→0x44154`,
6-bit scatter of `[r10+0x40]` to a stack table with channel-count validation
(`L86953–86973: srl/andi, `sw -0x108(r26)`, `beq r24,r25→0x447bc`),
then `L86974–86979: [r10+0xC]≠0 → 0x45644` (far checks on
`0x28/0x14/0x1C` with magics `0x3F/0x7A7`, looping back to `0x447D1/0x447D6`,
`L88206–88225`) and `[r10+0x40]&0x618==0x600` test → `0x45AC5`
(`L88591–88593: movi r23,1 / sw 0xc(r10),r23 / j 0x45648`), else fallthrough
`0x447D1` (`0x28==0x21?`, `0x1C&7==6?`, memsets…). It validates "one live
output, sane channel map", arms the shared downstream, and can set
`[r10+0xC]=1` itself.

### Q4. What is at `0x44194` (entry A)

The twin preamble with tag `r14=0` (`L86454`): same null-check shape
(`L86456/86458: r3==0→0x44451`, `0x48(r10)==0→0x4444F`), three zeroed tables
via `0x130D25` (`L86459–86475`), then channel-count check
(`L86476–86489: r23=[r10+8]; r23+=r10; lbz 0x44; ble→0x443A1`) into the body.
Both abort arms (`0x44451`, `0x4444F: movi r14,1`, memsets) converge to
`L86704: j 0x44154` = wrapper return.

**Stages, proved:** both entries set only the `r14` tag (0 vs 1), share the
same downstream (single burst builder `0x45D07` from 4 sites
`0x45FC9/0x460EF/0x4616A/0x46186`; single data-type select in the builder,
`L88834: ori r23,0x10C` vs `L88843: movi r23,0xb`; `0x16C` reads at
`0x45F06/0x461E5/0x46266`), and the tag is consumed downstream
(`0x461FD ble r14,r16→0x46259`, dec22_out `L182`). Codec discrimination happens
upstream (A/B production, this report) and at the builder — never at
`0x44194/0x44769`.

### Method notes (for the next session)

- The brief's two regex warnings were honored (grep-first; `.strip()` on the
  mnemonic; 8-digit `jal` targets) — plus a third trip-wire found here: the
  address field is exactly 6 hex digits with **no** leading-zero convention to
  assume (`13fad7`, clean-file `L431125: bn.addi r1,r1,-0x20`, is a real
  prologue; the pattern `013fad[0-9a-f]` silently misses it, which briefly
  produced a phantom "func 0x13fad7" confusion).
- `F_13FAD7` (memset-like, called from `0x130CE6/0x130D14`) is unrelated to
  `F_13FBB` despite the address proximity — do not conflate.
- Clean negatives delivered: no `jal` into `0x44200–0x45300`; no `0x648`
  writer besides `0x6A63D`; no `0x0(r10)` writer in `F_3D952`; no `0x00/0x4C`
  writer on the descriptor in any `r3=desc` callee; no SE read in either
  A/B value chain as scoped above.
