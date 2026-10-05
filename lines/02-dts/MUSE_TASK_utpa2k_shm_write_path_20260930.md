# MUSE TASK — utpa2k.ko: find the host-side code that can write the decoder SHM window

**Date:** 2026-09-30 · **Priority:** highest — this is the last open door
**Files:**
- `C:\firmware_temp\patch_baseline\utpa2k_stock.ko` (25 381 336 B, ARM 32-bit relocatable)
- Parent task, read its `## RESULTS` first: `C:\firmware_temp\MUSE_TASK_dec_stream_type_writer_20260930.md`
- DEC listing for cross-reference: `C:\firmware_temp\aeon_validate\dec_work\dec32_clean.txt`

---

## Where the previous task left off

Your Q1–Q4 are accepted; I verified the Q1 field formula and the `==5` reader set
independently from the raw listing. The established picture:

* The DTS stream-type field lives at **DSP `0xE01BC8 + idx*0x78 + 0xC`**, i.e. inside
  `g_virDecR2shm = MAD_base + 0xE01000`, so the **host offset is `k = 0x1BC8 + idx*0x78 + 0xC`
  (idx0 = `0x1BD4`)**.
* Writers set `5`, `2`, `4`, `1`, `5` — **never `9`**. Clean negative over a full movi sweep.
* It is **not** in the 58-entry `HAL_DEC_R2_Set_SHM_PARAM` window (that list tops out at
  `0x11C8`), same class of negative as `shm+0x118C` / `+0x196C`.
* **No single mappable point exists.** The gap is a population problem: DTS is simply not a
  member of any write path or read path.
* **Remap `9 → 5` is safe in principle** — you showed `F_15F67` has no codec-id switch and
  snapshots whatever struct it is handed, so running it on DTS data executes no AC-3-specific
  handling.
* Every `+0xC` write is followed by `e0CF` write-back + barrier, so the field is **published**
  for the host/SND/DMA to read.

**You flagged the residual yourself:** *"A direct host SHM write through the mapped window is
possible in principle but no host code was examined — needs `utpa2k.ko` SHM-store analysis,
out of static-DSP scope."* That is this task.

---

## Q1. Find every host-side store into the decoder SHM window

`g_virDecR2shm` is mapped by the host as `MsOS_PA2KSEG1(GetDspMadBaseAddr() + 0xE01000)`.
In `utpa2k_stock.ko` the mapping is at `+0x45A468` region (`movw r2,#0x1000 / movt r2,#0xe0`).
Find **every** instruction that stores through a pointer derived from that mapping, i.e. any
`str` whose base traces back to `MAD_base + 0xE01000`. Report each site with the offset it
writes.

**For each, state the offset range it can reach.** Specifically answer: **can any of them reach
`0x1BD4` (or `0x1BC8 + idx*0x78 + 0xC` for any `idx`)?**

## Q2. If a path exists, is it usable from userspace?

Trace it back to a UTOPIA dispatch entry. We already have a working vehicle —
`e1_ipauth/pwoff` sends an arbitrary `cmd` with a 64-byte payload to `/proc/utopia`
(`ioctl 0xC0085503`, `{cmd, arg_ptr}`), and `pwoff run 144 …` is proven to reach
`MApi_AUDIO_SetDTSCommonCtrl`. So the question is narrow and answerable:

* what **MApi command number** reaches the store path from Q1?
* what is the **argument layout** it expects?
* is there a command that writes a host-supplied value to a host-supplied SHM offset
  (a generic "write SHM" primitive), rather than a fixed internal one?

If a generic primitive exists, the DTS test becomes: send it with offset `0x1BD4` and value
`5` during verified DTS playback, and watch `dump_spdif_npcm`.

## Q3. If no path exists, prove it cleanly

Say so and state what you searched: the store sites, the dispatch table, and the
`ops`-table reach. A clean negative here is a **final, honest closure** of the project on the
static side, and it is worth as much as a positive. Note the anchor already established for
that table: `cmd 137 → ops+0x26C`, `cmd 145 → ops+0x28C`, hence **`ops_index = cmd + 18`**.

## Do not re-derive — already settled

* **The `0xBC` opcode class is named, by ReKo, in our own corpus.** The public Ghidra slaspec
  (`aeon_ghidra_public/aeon/data/languages/aeon.slaspec` + `aeon_ORBIS32.sinc`) has **empty
  bodies** for opcodes `0x28`–`0x2F`; only `0x2E` is defined and it is `__trap`. But the full
  SND decompilation `aeon_validate/r27_work/snd_full.reko/*.dis` names them:
  `__addcqq_s`, `__subcqq_s`, `__addcnqq_s`, `__subcnqq_s`, `__trap`, `__move_to_spr`,
  `__invalidate_line`, `__flush_invalidate`, `__restore_exception_state`, `__syncp`,
  `__syncwritebuffer`, `__enable_interrupts`, `__disable_interrupts`, `__sys`, `__flush_line`.
  So `0x28`–`0x2F` are the carry/borrow extended add-subtract group. To map opcode → mnemonic,
  align a `.dis` against the raw image with the length rule
  `len = 4 if ((w>>29)&7) in (5,6,7) else 2`; the blocker is that ReKo `.dis` addresses are
  **segment-relative** while `snd_full.bin` starts with zero bytes, so the segment base must be
  recovered first. **If you need `bg.opcode_2F` for the DTS case-4 arithmetic
  (`0x30F72`, six `opcode_2F` + two `opcode_2A` that AC-3's case 1 does not have), do this
  alignment first — but treat it as a side quest, not the main task.**
* Host side is exonerated: the HAL knows DTS and defaults SPDIF to `BYPASS`+`DTS`; bypass mode
  does not select the codec (AC-3 format 5, DTS 9 and DTS-HD 17 all take the **same** arm of
  `HAL_AUDIO_SPDIF_BypassMode`'s 96-entry switch); `HAL_AUDIO_SPDIF_SetOutputType` is `bx lr`
  and `MDrv_AUDIO_SPDIF_SetOutputType` is `b .`, which is harmless because nothing consumes
  that field in bypass; `AudioVars2+0x4F8` is only `printk`ed.
* `shm+0x118C` and `shm+0x196C` are likewise absent from the 0xD1 list.

## Method notes — six mines on this corpus now

1. Cross-check every scan with `grep` first. Six Python regexes over these listings have
   silently returned zero while `grep` found thousands.
2. Mnemonic field must be `.strip()`ed, and **immediates are written without `#`**
   (`beqi r23,0x5`), and some mnemonics carry a **trailing comma** — so never anchor with `$`.
3. **Offsets are lowercase in the DEC listing**: `sw …0xc(` matches nothing; `0xC` does. This
   one bit me and Muse independently on the same search.
4. The address field is exactly six hex digits, no assumed leading zero (`13fad7`, not `013fad7`).
5. `r3`/`r10`/`r12` are different structs in different functions.
6. State *read / inferred / not found*. Clean negatives are results — two earlier
   "narrowing" lists in this project had to be retracted.

## Rules

* Every claim cites a `utpa2k_stock.ko` address (or a symbol + offset) or a listing line.
* Q3 is a first-class answer. If the host cannot write `0x1BD4`, say so plainly and do not
  invent a workaround.
* Do not flash anything. Static analysis only.

---

## RESULTS (2026-09-30, static, `utpa2k_stock.ko` ARM disassembly — pyelftools + capstone + relocs)

All addresses are `.text` offsets (`ET_REL`, `e_machine=EM_ARM`, `.text` file
base `0xD654`, vaddr 0). `bl` targets resolved via `.rel.text` (`R_ARM_CALL`=28);
`movw/movt` pairs via `R_ARM_MOVW_ABS_NC`=43 / `MOVT_ABS`=44. Linear-sweep
desync was avoided for the mapping census by byte-pattern scan.

### Base facts (read, not inferred)

- `g_virDecR2shm`: sym `[20664]`, `.bss+0xEFBF4`, 4 bytes (the cached mapping).
  Only 9 use sites image-wide (movw/movt pairs): `0x457E34, 0x458070, 0x458440,
  0x458968, 0x4591D0, 0x4592AC, 0x45A43C, 0x45B7C8, 0x45C0B0`.
- Byte-scan for `movw Rd,#0x1000` + `movt Rd,#0xE0` (any Rd) finds **exactly**
  those 9 DEC-SHM mapping sites (in `HAL_AUR2_Backup/RestoreShareMemory`,
  `HAL_DEC_R2_init_SHM_param`, `HAL_DEC_R2_SetCommInfo2`,
  `HAL_DEC_R2_Get_SHM_INFO2/INFO/INFO_64bit`, `HAL_DEC_R2_Set_SHM_PARAM`,
  `HAL_DEC_R2_Get_SHM_PARAM`) — no hidden recomputation of `MAD+0xE01000`
  exists. (SND uses `MadBase+0x70A000`, a different window — out of scope for
  DEC `0x1BD4`.)
- `HAL_DEC_R2_Set_SHM_PARAM @0x45A434/0xFA4` head (`0x45A434–0x45A4BC`, read):
  `r7`=global slot, `r6`=[mapping]; null → `GetDspMadBaseAddr(2)` +
  `0xE01000` (`0x45A468 movw r2,#0x1000 / 0x45A46C movt r2,#0xE0` — the brief's
  anchor, confirmed) + `MsOS_PA2KSEG1`, cached. Args:
  `r5=r0=id, fp=r1, sl=r2, r4=r3`. Validation is a flag trick —
  `0x45A48C mov sb,0 / 0x45A490 cmp fp,1 / 0x45A494 movls sb,1 /
  0x45A498 cmpls r5,0xD0 / 0x45A49C bhi reject(0x45B354)`:
  `fp>1` leaves stale HI flags from `cmp fp,1` → `bhi` taken; `fp≤1` validates
  the id. **So `fp ∈ {0,1}` is enforced** (decoder index, matching the
  1st/2nd-decoder `audio_status` world), `id ≤ 0xD0`, else return `sb`.
  Dispatch: `r0 = shm+0x1590`, `ldr pc,[r2+r5,lsl#2]` with the **absolute jump
  table at `0x45A4C0`** (entries already linked: `id → 0x45A804…0x45B340`,
  unmapped ids → default `0x45B354`) — fully decoded, ids `0x00–0xD0`.
- Handlers run with `r0 = shm+fp*960` (recomputed per handler:
  `rsb r0,fp,fp,lsl#4 / add r0,r6,r0,lsl#6`; fp=decoder index, stride 960 =
  `0x3C0`, the same stride the DSP side uses) and write **fixed offsets only**:
  tail-form (`movw r1,#OFF / b 0x45B348 / str sl,[r0,r1]`, OFFs `0x1034`…
  `0x11C8`, exhaustive list extracted — max `0x11C8`) or direct
  (`str …,[r0,#C]`, C ≤ `0xFEC`; `str …,[r0,#8/#0xC/#0x10]` on the
  `shm+0x1590` base, i.e. `0x1598/0x159C/0x15A0`; one read-modify-write at
  `+0x10E0`, `0x45B150–0x45B170`). The only indexed-looking stores resolve to
  fixed: `0x45B140 str r1,[r0,r2]` (`r2` = movw-fixed `0x1134/0x1138/0x113C`,
  `r1` = clz-derived 0/2); `0x45B1A4/0x1B0 str sl,[r4,r0]` (`r4` recomputed =
  `shm+fp*960`, `r0` = movw-fixed `0x1148/0x114C`); `0x45B208` (`r2` =
  movw-fixed `0x1170`); `0x45B334` (`r1` = `0x11C0/0x11B8` fixed, values
  `r4/sl` caller-supplied but offsets not). Every handler ends in
  `MsOS_FlushMemory` (`0x45B1A8`, `0x45B350` — reloc-verified), so host writes
  are DSP-visible by construction.
- The remaining SHM users: `init_SHM_param` = `memset(shm,0,0x1C98)`
  (`0x458480–0x458490`, **covers `0x1BD4` — target init value is 0**) plus
  per-instance constants (`fp = shm + r7*960`, r7 ∈ {0,1}, OFFs ≤ `0x11A8`,
  values `0/1/0x10/0x60/0x4B00/0x2000/−0x1F/0x12345678/0x11940000…`,
  `0x4584D8–0x458664`, read in full); `SetCommInfo2` gated `sb==6` writes only
  `+0xD9C/+0xD98` (`0x4589E8–0x4589F4`); `Backup/RestoreShareMemory` full-range
  scan (`[0x457E04,0x458054)` / `[0x458054,0x4582A8)`, symtab-sized) shows only
  pointer/global/stack bookkeeping stores plus `malloc(0x1C98/0x8B8)` —
  **no SHM-content store** (they snapshot console buffers, take no
  userspace values); `Get_*` are reads.

### Q1. Every host-side store into the DEC SHM window — and `0x1BD4`

The exhaustive set above yields reachable offsets `{fp*960+OFF}` with
`fp ∈ {0,1}` (enforced) and `OFF ≤ 0x11C8`, plus `{0x1590…0x15A0}`:
maximum reachable `= 0x3C0+0x11C8 = 0x1588` (fixed handlers) and `0x15A0`
(`shm+0x1590` base). Exact-hit equation `fp*960+OFF = 0x1BD4` has **no
solution** in the observed OFF set (`fp=0 → OFF 0x1BD4` absent, max `0x11C8`;
`fp=1 → OFF 0x1814` absent; huge-`fp` wrap games excluded — `fp>1` is
rejected outright, so no modular aliasing is possible). Margins are wide
(`0x1BD4−0x1588 = 0x64C`, `0x1BD4−0x15A0 = 0x634` — not even adjacent).
**No host-side store instruction in this image can write `0x1BD4`
(or `0x1BC8+idx*0x78+0xC` for any idx — same arithmetic), for any param id,
any decoder index, any caller.** `init` additionally guarantees the word
reads 0 until DSP code touches it.

### Q2. Usability from userspace — moot, with the vehicle documented

- The proven vehicle (`pwoff run 144 → MApi_AUDIO_SetDTSCommonCtrl`)
  terminates, statically verified end to end, at exactly one SHM store:
  `MApi_AUDIO_SetDTSCommonCtrl @0x3F9298` (userspace/UTOPIA shim:
  `UtopiaOpen`/`UtopiaIoctl`) → `_MApi_… @0x3EAD04` →
  `MDrv_AUDIO_SetDTSCommonCtrl @0x4276BC` (4-byte trampoline) →
  `HAL_MAD_SetDTSCommonCtrl @0x464888`, whose sub-command table
  (`0x4648A0`, 9 words, read out: `6→id 0x30`, `8→id 0x31`, rest default or
  the no-store subs 13/14 paths) calls `Set_SHM_PARAM`
  (`0x4648E4`: `id = 0x30/0x31`, `fp = 0`, `sl = (arg1≠0)`, `r3 = 0`).
  Handlers `0x45ACC8/0x45ACD4` write `shm+fp*960+{0x1034,0x1040} = sl`
  (0/1 only). So cmd144 moves exactly two fixed bits near `0x1034`,
  consistent with the brief's "fired subs 6..14, changed nothing".
- Argument layout for that vehicle: `(sub-cmd ∈ 6..14, value-bit)`; no
  offset operand exists anywhere in the chain.
- The only other userspace-adjacent surface,
  `MDrv_AUDIO_Debug_Cmd_Write @0x408B30/0x8C28` (giant debug switch, reached
  from `_MApi_AUDIO_Monitor`/`_MApi_AUDIO_Debug_Cmd_Write`), fans out to
  fixed ids only (`0x34/0x35/0xBD/0xC1` observed at its `Set_SHM_PARAM` sites
  → OFFs `0xF58/0xF5C/0x1198/0x11A8`-class, same fp∈{0,1} envelope).
- **No generic "write SHM offset,value" primitive exists**: every store offset
  in every handler is an immediate (`movw`/`[r0,#C]`); the three
  register-indexed stores resolve to immediates on inspection (above). Hence
  the Q2-proposed test ("send offset `0x1BD4`, value `5`") is **not
  expressible** through any MApi/MApi-debug path in this image — stated
  plainly as required.

### Q3. Clean-negative proof (first-class answer)

(a) **Store sites**: complete disassembly of the handler range
`0x45A804–0x45B360` (every `str`, enumerated above by offset class) plus the
other SHM-touching functions (`init`, `SetCommInfo2`, `Backup/Restore`,
`Get_*`); SND-window functions excluded by base (`MadBase+0x70A000` ≠ DEC
window) — stated with reason. (b) **Dispatch table**: `0x45A4C0`, ids
`0x00–0xD0` fully decoded (linked targets + shared default `0x45B354`);
`id>0xD0` rejected at entry. (c) **Ops-table reach**: attempted via
`.rel.data` runs (only peripheral tables: panel/PCM/CPU_Sound) and
`.rel.rodata` (zero `MApi/MDrv_AUDIO_*` refs) — **the UTOPIA AUDIO ops table
is not in this `.ko`** (no `R_ARM_ABS32` ref to any `*SetDTSCommonCtrl*` /
`*SetDecCmd*` symbol from data/rodata), so the brief's `cmd+18` anchor
cannot be verified here; dispatch past `UtopiaIoctl` lives in `utopia.ko`
(out of scope, and unnecessary — the negative rests on handler
exhaustiveness over all ids, which no caller chain can escape, not on ops
enumeration). Net: the host **cannot** place `5` (or anything) at DSP
`0xE01BC8+idx*0x78+0xC` through any examined mechanism; combined with the
parent task (DSP side never writes `9`, never selects DTS), the word is
unwritable-from-host and unset-by-firmware — the honest static closure.
