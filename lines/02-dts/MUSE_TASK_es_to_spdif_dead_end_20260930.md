# MUSE TASK — the ES→SPDIF dead end: who decides the decoder output is not sent to the transmitter

**Date:** 2026-09-30 · **Priority:** highest — this is now the *only* unexplained live fact
**Files:**
- **Use `dec33_realcode.txt`** (504 216 real code instructions, no data). Never `dec32_clean.txt`.
- DEC image `dec_work/dec_full.bin` (file offset == fw addr)
- SND realcode `snd_work/snd33_realcode.txt`
- Read first: `FORENSIC_dts_final_state_20260930.md`

**Do not delete any Ghidra project. Create new folders/projects only. Do not flash anything.**

---

## THE NEW FACT — measured live today, on a working harness

The mdb `dump_es` instrument was run for the first time. On a **clean** system:

| | decoder ES (`dump_es`) | optical TX (`dump_spdif_npcm`) |
|---|---|---|
| **DTS** | **2 883 196 B**, 1586 × `7FFE 8001`, subframe `0x77` | **0 B** |
| **AC-3** | 714 240 B | 2 727 936 B |

DTS head: `7f fe 80 01 fc 3c 7d b2 77 00 05 38 00 09 ef 7b`

**So the DTS decoder receives and emits a fully valid IEC61937 bitstream, and the optical
transmitter emits nothing.** The block is *downstream of the decode*, in the handoff. Both
files are on the projector at `/data/` and the DTS one was pulled to
`C:\firmware_temp\aeon_validate\r57_npcm\DTS_clean_DUMP_ES1_from_DDR_00.bin`.

Kodi requests passthrough identically for both codecs:
`AE_FMT_RAW (AE) method: RAW (PT) stream-type: STREAM_TYPE_DTS_512` vs `STREAM_TYPE_AC3`.

**This is a different question from every one asked so far.** All previous work assumed the
block was at the decoder *entry* or in the DEC output *gate* arithmetic. The evidence now says
the ES side is complete and the loss is after it.

## HARNESS NOTES — three traps that cost this project real time. Do not repeat them.

1. **Kodi's `Player.Stop` requires `playerid`.** Without it the call errors and the player
   never releases the audio device. Every "DTS = 0 bytes" measurement in the old archive was
   taken in this broken state. Always `{"method":"Player.Stop","params":{"playerid":1}}`.
2. **`[Audio Mon Task]` can wedge in state `D`.** Symptom: Kodi logs
   `CActiveAE::InitSink - failed to init` and `CDVDAudio::AddPacketsRenderer - timeout adding
   data to renderer` for *both* codecs, and every dump reads 0. Check with
   `ps -A -o PID,STAT,NAME | grep -aE 'MsOS_WaitEvent|Audio Mon Task'` — `D` is bad, `S` is
   good. **Only a reboot clears it.** Nothing playing reads 0 because the monitor is dead, not
   because the codec is blocked.
3. `Decode frame count` and `play state` in `show_all_decoder_status` are **frozen constants
   (2560, STOP)** — not oracles. `Decoder format` IS reliable (it reads DTS / AC3P correctly).
   The mdb DM window `read_dsp_sram_type=1` does **not** discriminate codecs: cells
   `0x100C`/`0x1038`/`0x103F` are free-running counters that move with nothing playing.

## Q1 (decisive). The ES→output handoff in DEC

`dump_es` is the decoder's elementary-stream output. Find the code that:
* takes the decoded/packed ES and hands it to the output stage, and
* can **drop, divert or refuse** it.

Start from the known output-stage sites (these are already verified):
* `0x4604B jal 0x643DC` / `0x4604F bgtui r3,0xd,0x46000` — the size divert
* `F_643DC` = `[0x14] − ([0x00]−[0x20])/4`; `F_64296` writes `[0x00] = ([0x0C]<<2)+[0x20]`
* burst arm needs `[0x38] == 0x400`

**Report the full call chain from "ES buffer ready" to "byte written toward the transmitter",
and name every conditional on it.** Specifically: is there any branch on the *codec* (stream
type 4 vs 5, or the `0x77` subframe) anywhere between decode and output?

## Q2. The IEC61937 packer

`7FFE 8001` framing and the `0x77` subframe are written by *someone*. The DTS ES dump already
contains correct framing, so the packer ran. **Find the packer** and determine whether the
DTS and AC-3 paths use the same one, and whether anything downstream of the packer re-checks
the data-type byte (`0x0B` DTS / `0x10C` AC-3) before passing the frame on.

If the packer is common and unconditional, say so — that is a useful negative and it moves the
block to Q1/Q3.

## Q3. The transmitter side, and what "0 bytes" means

`dump_spdif_npcm` reads 0 for DTS and 2.7 MB for AC-3 in the same session, same player.
**Find the code that arms/enables the SPDIF transmitter** and determine what condition must be
true for it to carry a non-PCM stream. Then answer: given that a valid DTS IEC61937 stream
exists in the decoder, what single condition could still keep the transmitter silent?

Check for a **codec-identity field** that must be set before the TX is armed (a format
enumeration, a channel-status/data-type register, a "non-PCM mode" flag). `AudioVars2+0x4F8` is
the codec word the SPDIF TX is told to emit (`HAL_AUDIO_SPDIF_ApplySetting` @0x4467CC) — the
host-side value of that word is kernel-only and was never read. Its **DSP-side** setting code
is fair game and has never been examined.

## Q4. One thing only you can settle cheaply

`reg_bank=0x112A`-style dumps and the `0x7FFE8001`/`0x36363636`/`0x5C5C5C5C` constants in
`snd_full.bin` suggest channel-status/data-type words exist per core. **Enumerate every
immediate constant in DEC and SND that looks like an IEC61937 data type or a channel-status
word**, and report which function writes each and whether the writer is codec-conditional.

## Method notes (each of these has silently produced a wrong number here)

* Branch targets are **8** hex digits; mnemonics carry suffixes and **trailing commas**;
  offsets are **lowercase** in the DEC listing; strip the mnemonic field.
* `jr r9` can end a fragment — verify with a branch into the block before claiming a function head.
* A counter read once will be re-derived wrongly. Sample it more than once.
* **Every claim cites a `realcode` line (6-hex address) or a binary offset. A clean negative is
  a result — say what you searched.**

## Deliverable

A ranked list of the **specific words/conditions** that could keep a valid DTS IEC61937 stream
from reaching the transmitter, each with the DEC or SND address that implements it and whether
it is codec-conditional. If one of them is a single word the host can reach, say so loudly —
A ranked list of the **specific words/conditions** that could keep a valid DTS IEC61937 stream
from reaching the transmitter, each with the DEC or SND address that implements it and whether
it is codec-conditional. If one of them is a single word the host can reach, say so loudly —
that is the patch target this whole project has been looking for.

---

## RESULTS (2026-09-30, static only, `dec33_realcode.txt` / `snd33_realcode.txt`)

All prior DEC addresses re-verified in `dec33` (same image; `0x4604B jal`,
`0x4604F bgtui`, `F_643DC @0x642DC`, fork `0x460B1`, builder tags
`0x45D7F/0x45DA0/0x45DBF`, markers `0x4627B/0x46285` — all present).
Harness-note-3 correction carried: `0x100C/0x1038/0x103F` are activity
counters (move idle); only `0x100E/F` (`0x202/0x1FF`) and `0x104C` (`0x400`)
are stable. Prior ptr/count labels softened accordingly — structure stands.

### Q1. Chain ES-ready → TX (verified segments + one junction residual)

`ES fill (helpers 0x111D7/0x10BB6/0x10D0E/0x10F88, 27/40/2/6 callers;
0x10BB6 itself works the ring-word family [0x28]/[0x30]/[0x14]/[0x20])`
→ `classifier F_24CB4 (prologue 0x24CB4; branch-entered, no jal)`
→ `…junction residual R-J (classifier→F_410A3 caller link unmapped;
F_410A3 callers: 0x3E829 + self-recursion 0x41238)…`
→ `F_410A3: copy pump; 0x41275-block gated by [r11+0x4EE4] (0x4121E)`
→ `F_45FD1 (entry/arm/remaining/fork/size/builder/refill)`
→ `0x4618E-cluster, called from F_410A3 @0x41289 (call-verified, shared r10):
link-gate 0x461A8 → buf-clear (2× jal 0x130D25) → fork re-test 0x461E2 →
burst loop / 0x462A0 / 0x462DE markers`
→ `SND drain (whitelist + 0x0B-scan + state struct; both verified)`
→ `TX (arming unmapped, R-TX)`.

Conditionals per segment are enumerated below. **Codec branches between
decode and output: NONE test codec-type directly** — every test is on data
bytes (`0x0B`/`0x77`) or state words. One value-gate brushes DTS: `0x253C7`
(see #7).

The classifier (new this session): sync scanner `0x2417B` (sole caller
`0x25026`; returns 0/−1/−2 scanning for the `0x77/0x0B` adjacency) plus the
`0x24CB4` dispatcher whose lookahead bytes `[r1+0x5E]/[r1+0x5F]` (helper-fed
via `0x111D7`, cleared in-situ `0x24FD7/0x2540E`) route:
`(0x77,0x0B)→process (0x24FCD+/0x25054)`, `[0x5E]==0x0B→0x24EDE-handler`,
else reset/resync (`0x25000`: `jal 0x10BB6`, scanner, `0x1F/0x1E` dispatch to
`0x25426/0x253F1`, both refill `[r1+0x5E]` and rejoin). Sync-found goes to
processing — the classifier is not a DTS drop on its face; a misroute would
have to come from lookahead fill geometry (frame-size-dependent, runtime).

### Q2. Packer: common, type-blind; downstream Pc re-checks enumerated

- `0x7FFE/0x8001` FW-wide: DEC 31+19, SND 15+14. Of DEC's, 29 are
  **comparisons** (parser sync detection, incl. the shared-idiom clusters
  `0x3496E+`/`0x358F7+` mirrored in SND `0xED93D+`/`0xFA3F0+` — same source
  shape, both FWs); exactly two **construct**, one being the burst packer
  `0x46259..0x4629C` (FINDING-verified), the other the `0x64624` swapped
  pair (`0x80017FFE` variant — mask/other-header, residual).
- **No Pc/`0x0B`/`0x77` test within ±0x100 of any `0x7FFE` site, either
  image** (complete scan) → the packer is type-blind and common. Useful
  negative, as briefed.
- Downstream Pc re-checks (all survive DTS on static reading): the DEC
  classifier above; the `0x253C7` range gate (`r24−0x0B ≤ 5` unsigned —
  `{0x0B..0x10}` pass, DTS at the exact floor; enclosing function entered
  by fallthrough, callers residual R-253); the SND whitelist
  `{3,0x0B,0x0C,0x0D,0x0E,0x0F,0x10}` (`0x8A160+`, DTS included,
  consume-and-continue arms `0x8A00B/0x8A18F`); the SND `0x0B`-scan
  (`0x820D0`: found → full processing `0x82122+` incl. helper chain
  `0x3DA6E/0x1157DE/0x3DB15`, not-found → bare return).

### Ranked deliverable (words/conditions, codec-conditionality, address)

1. **`[r11+0x4EE4] @0x4121E` (F_410A3)** — gates the whole
   F_45FD1+packer block (`beqi 1 → 0x41275`, else `0x411C9/0x411CD` arms).
   Narrowest upstream single word. Writer unmapped (residual) →
   codec-conditionality unknown. Loudest unknown.
2. **B-latch `[r4+0x0] @0x45FEB` + `[r4+0x10] @0x45FF1`** — `B==0 &
   [0x10]≠1` returns immediately (packer, `+0x2C=1`, everything downstream
   never runs). DTS (`B==0`) hinges entirely on `[r4+0x10]==1`.
   Codec-conditional via the 4-gate latch chain (prior work). Prime suspect,
   unchanged.
3. **Remaining gate `0x4604F` (`[0x14]−[0x0C] ≷ 13`)** — diverts to return.
   Runtime words, prior derivation stands.
4. **`[r10+0x28]==1` at `0x46019/0x46079/0x461A8`** — `≠1` bails at three
   points (`0x45FFA`-return, skip-`0x4608E`, `0x461AB` size-zero return).
   AC-3 passes via the copy-path guarantee (`0x46019` screens it);
   DTS-arm carries `[r4+0x38]-obj+0x28` (runtime). FINDING #1, qualified.
5. **`[r10+0x38]==0x400 @0x460D1`** — else builder/refill returns, no
   `F_46172` burst. Unchanged.
6. **Packer-enable `[r10+0x2C] @0x4625C` — POLARITY CORRECTED.** Complete
   `sw 0x2C` sweep: in-output-path writers are only `0x46013` (=0, copy
   path) and `0x46087` (=r11=1, `0x46080`-arm only); no store between the
   `0x41283/0x41289` jals; r10 identity call-verified across all three
   functions. So `+0x2C==1` ⟺ last pass took the B==0&`[0x10]`==1 arm —
   **the packer runs on the "DTS-arm", not the copy path.** FINDING #2 as a
   lever is inverted: forcing it without the arm's other outputs risks a
   malformed burst. Open sub-question (stated, not guessed): how AC-3 packs
   — same arm, SND-side constructors, init-persist, or recursion pass.
7. **Data-type range gate `0x253C7`** (`r24−0x0B ≤ 5` unsigned → divert
   `0x2549D`) — DTS `Pc=0x0B` sits at the exact passing floor; an off-by-one
   here kills DTS only. NEW. Residual: enclosing head/callers + `r24`
   origin (mid-function straight-line block, no `jal`/head in
   `0x25380..0x253C0`).
8. **SND whitelist + `0x0B`-scan — EXONERATED** (useful negatives): whitelist
   includes `0x0B` with consume-and-continue arms; scan finds DTS and runs
   the full found-arm. Not suspects.
9. **TX hardware arming — unmapped (R-TX).** No SND SE/TX-enable idiom
   identified; host word `AudioVars2+0x4F8` kernel-only (userspace dead);
   prior module-wide scan: zero host refs to any gate word. If 1–7 pass
   yet TX is silent, the remainder is here (SND SE programming).
10. **Host-reachable word: NONE — stated loudly.** Every gate word above is
    FW-internal; SHM tables top out far below; no host primitive reaches
    them. There is no host-side patch target; any patch is a DSP-image
    change at sites 1–7 (auth-gated, reversible, Pioneer-oracle per
    FINDING §7 caveats).

### Q4 enumeration (data-type / channel-status immediates)

- Preamble: `0x7FFE/0x8001` sites above (all listed; 29+2 split DEC).
- Pc tags (builder, VERIFIED): `0x45DA4: 0x0B` (r4=0 arm = DTS path),
  `0x45D7F: 0x10C`, `0x45DBF: 0x20D`. The ~600 other `0x0B` hits are
  shifts/offsets/registers (searched, triaged); `0x10C`'s 146/71 likewise
  mostly struct offsets (triaged); `0x20D` 7/1 are the tag + few others.
- `0x77`: classifier tests (above); remainder is op-imm noise and
  `movhi_2` halves (triaged).
- `0x36363636/0x5C5C5C5C`: **zero in realcode** (both images) — they were
  data, removed by the reachability filter. Closes the brief's suggestion.
- Channel-status / non-PCM-mode bit as a distinct word: NOT FOUND (bounded
  negative). The non-PCM identity travels in Pc/burst header per the code;
  no separate flag word and no TX-mode-set function pair were found —
  falls into R-TX.

### Residuals (each one address-named)

- R-J: classifier→pump junction (caller of `F_24CB4`; source of
  `F_410A3.r23`); `F_410A3` callers `0x3E829` + self `0x41238` verified.
- R-253: `0x25399`-block head/callers, `r24` origin, `0x2549D/0x2544D/
  0x24E61` divert semantics.
- R-AC3PACK: how AC-3 packs given the `+0x2C` polarity (four named
  alternatives above) — constrains DTS identically once answered.
- R-TX: SPDIF TX-enable programming (SND SE side); `[r11+0x4EE4]` writer;
  SND reader entries (`0xD1203/0xD1223/0xD1241`, branch-reached).
- R-TAP (unchanged): `dump_es` tap point is "Decoder ES" per mdb help
  (pre-gate side); host handler statically unlinked (strings in `.data`
  registry, no `.text` xrefs, no fn-ptrs nearby) — runtime-traceable,
  not statically closable here.

Scripts: `e_recon.py` (sites+consts), `e_pack.py` (all `0x7FFE/0x8001/
0x0B/0x77`), `e_disc.py` (classifier/scanner/SND discs + Pc-near-packer
scan), `e_handoff.py` (arms/callers/stack-fillers), `e_chain.py`
(class-head/helpers/entries), `e_final.py` (`+0x2C`/`+0x28` sweeps,
`0x2549D`, SND sinks), `e_410a3.py` (recursion + `0x41275`-block),
`u_dump*.py` (tap hunt: strings→relocs→fn-ptrs, all negative).
