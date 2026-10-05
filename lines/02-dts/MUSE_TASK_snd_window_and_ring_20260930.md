# MUSE TASK — SND side: what is the readable DSP window, and where is the DEC→SND ring?

**Date:** 2026-09-30 · **Priority:** highest — this is the only remaining live lead
**Files:**
- **New listing, use it:** `C:\firmware_temp\aeon_validate\snd_work\snd33_realcode.txt`
  (SND, 253 507 real code instructions — produced with the reachability filter, so unlike
  `snd32_clean.txt` it contains **no data**)
- SND image: `C:\firmware_temp\aeon_validate\r27_work\snd_full.bin` (1 839 920 B; **file offset
  = fw addr − 0x16F00** per the 2026-09-12 note)
- DEC listing: `C:\firmware_temp\aeon_validate\dec_work\dec33_realcode.txt` (504 216 real instructions)
- DEC image: `C:\firmware_temp\aeon_validate\dec_work\dec_full.bin` (md5 `4b7e9509…`; file offset == fw addr)
- Read first: `C:\firmware_temp\FORENSIC_dts_final_state_20260930.md` — it has the whole closed
  chain on the DEC side, so you do not re-derive any of it.

---

## What is closed, so you do not re-open it

The DEC side is machine-checked and final:

* DTS bitstream valid: 2.1–5.0 MB, 692 × `7F FE 80 01`, IEC61937 subframe `0x77`; decoder runs the
  whole track at `cmd:0x84` / `DecStatus:1`; optical TX emits 0 B for DTS and ~2.1 MB for AC-3.
* The output stage is **two data conditions, no code branch**:
  `[0x14] − [0x0C] > 0x0D` diverts (`0x4604F`), and the burst arm needs `[0x38] == 0x400`.
  I verified `F_643DC` = `[0x14] − ([0x00]−[0x20])/4` and `F_64296` writes
  `[0x00] = ([0x0C]<<2) + [0x20]`, so the `[0x20]` terms cancel and the gate is exactly
  `[0x14] − [0x0C]`.
* Four patches flashed, all null, all reverted, AC-3 control passing each time. **They all acted
  on code; the block was never in code.**
* Host exonerated: HAL knows DTS:X/DTS-HD/DRC, SPDIF defaults `BYPASS`+`DTS`, bypass does not
  select the codec, `HAL_AUDIO_SPDIF_SetOutputType` is `bx lr`, `AudioVars2+0x4F8` is only printed.

## The new fact that opens this task

The mdb window `read_dsp_sram_type=1 addr=N` (DM, cells 0x0000–0xFFFF) is **live, audio-shaped
and codec-dependent**:

```
cell      AC-3       DTS
0x100C    0x6570     0x65A0
0x1038    0x001F87   0x001FAE
0x103F    free-running counter (moves on its own)
0x104C    0x000400   0x000400
0x100E    0x000202   0x000202
0x100F    0x0001FF   0x0001FF
```

**But it is NOT the decoder SHM.** `cmd 144` sub 6 writes `shm+0x1034 = 1` (rc = 0) and **no cell
in the window changes** — checked against both `PM` (param1 = 0) and `DM` (param1 = 1).
Known bases: DEC `MAD_base + 0xE01000`, SND `MAD_base + 0x70A000`.

## Q1 (decisive). Identify the window's absolute address

In `snd_full.bin`, find every construction of an absolute DSP address and report the **distinct
base constants**. Muse's earlier note says the SND uses `MadBase + 0x70A000`. Confirm or correct
it. Then:

* locate the structure that would sit at **`MadBase + 0x70A000 + 0x1000`** — because the window's
  live cells cluster around `0x1000`–`0x1048`;
* **match the observed values.** `0x400` at `+0x104C`, `0x202` at `+0x100E`, `0x1FF` at `+0x100F`,
  a counter at `+0x103F`, and `0x1FAE` / `0x65A0` under DTS. Any field whose *purpose* explains
  those values confirms the mapping. **Report the mapping explicitly** (base + the offset of each
  observed cell).

## Q2. The DEC→SND ring

Find the ring the decoder fills and the SND drains: its base, the head/tail/occupancy fields,
and who writes each side. Then state:

* does the ring live inside the readable window? If so, which cells are the pointers?
* is there any **codec-conditional** code on the SND side that would keep DTS from draining it
  — including anything that tests a word the DEC sets (the stream type, a `0x400`-class flag, a
  count)? This is the mirror of the DEC question and it has never been asked.

## Q3. What consumes `[0x38] == 0x400` on the SND side

The DEC builds the burst only when that word reads `0x400`. **Find where `0x400` originates on
the SND side** — is it also the value of an SE/host-writable parameter? On the DEC it sits at
SHM offset `0x104C`, which is in the 0xD1 host-writable list and reads `0x400` for both codecs.

If the SND has an equivalent parameter, that is the second lever, and it is one the host may be
able to reach.

## Tooling now available

* **AEON assembler** — not available. But the **encoder is derived and verified**:
  `enc(rD,rA,imm,sub) = (7<<29)|(0x3B<<26)|(rD<<21)|(rA<<16)|(imm & ~3)|sub`, checked against
  24 859 DEC instructions with 0 mismatches. `sub` 0 = `sw_0`, 2 = `lwz`. From
  `aeon_ORBIS32.sinc`, block `i32_opcode = 0x3B`.
* `reko.exe disassemble --arch aeon --data <HEX>` for the `0x28`–`0x2F` MAC group the Ghidra
  slaspec leaves blank.
* Ghidra headless works; the recipe (and three fixes it needed) is in
  `td98-dec-output-entry-gate.md`. **Do not delete any existing project** — make new folders.

## Method notes — these have cost this project real time

* Use `snd33_realcode.txt`, **not** `snd32_clean.txt`. The latter was produced by sweeping every
  offset and emits data as instructions; ~14 % of it is not code. Counts from it are upper
  bounds only.
* Branch/offset targets are **eight** hex digits; mnemonics carry **suffixes and trailing
  commas**; offsets are **lowercase**. Six separate regex traps, each of which silently
  produced a wrong number.
* `jr r9` can be the *end* of a fragment, not a prologue — `0x190F0` turned out to be a return
  and the code after it belonged to a tail-callee. Three false conclusions came from this.
* A counter read as a constant will be re-derived wrongly; sample it more than once.

## Rules

* Every claim cites a `realcode` line or a binary offset.
* A clean negative is a result — say what you searched.
* If the window is neither DEC nor SND, say so plainly; that is a useful answer.
* Every claim cites a `realcode` line or a binary offset.
* A clean negative is a result — say what you searched.
* If the window is neither DEC nor SND, say so plainly; that is a useful answer.
* Do not flash anything. Static only.

---

## RESULTS (2026-09-30, static only)

Listings used as cited: `snd33_realcode.txt` (253 507 insns, no line numbers —
cite the 6-hex address), `dec33_realcode.txt`, `utpa2k_stock.ko` (ARM,
`.text` off `0xd654`). Method traps respected: 8-digit `jal` targets,
suffix/trailing-comma mnemonics (`sw_0 …,` matched), lowercase offsets,
`jr r9` fragment discipline, branch-into verification for every entry claim.

### Q1. The window is DEC DSP DM — the `MadBase+0x70A000` premise is corrected

**Base constants in `snd_full.bin`:** top composed `movhi+addi/ori`
constants are DSP-local (`0x0014xxxx–0x001Cxxxx`, 8–40 uses each) plus audio
constants (`0x7FFE8001` ×12, `0x7FFFFFFF` ×56, `0x36363636`, `0x5C5C5C5C`).
Full-listing greps for `0x70a000`/MadBase in **both** FW images return **zero**.
`B+0x70A000` exists only on the host: `HAL_SND_R2_Set_SHM_PARAM @0x459DFC`
builds it inline (`0x459E30 movw r2,0xA000; 0x459E34 movt r2,0x70` → CPU-mapped
`g_virSndR2shm`, VERIFIED in `.ko`). The IDMA `read_dsp_sram` path never
touches it. Two different mechanisms: CPU-bus SHM alias (host) vs DSP-DM
cells (mdb). The brief conflated them — corrected plainly.

**Which core:** DEC. (1) Prior decode stands (`_HAL_MAD2_DBG_CMD_Read_DSP_sram`
`@0x467788`: `param1` 1 = DM / 0 = PM with addr ≤ `0x2800`; single SE-IDMA
port `0x112DDC`, no core select). (2) Corroborated in-session: the SND image
has **exactly one** `+0x104C` writer and it writes 0 (`045d08`), while the
live cell reads `0x400` — a live computing writer exists **only on DEC**
(three sites, below). (3) Cells are 24-bit DM words (`DM[0xNNNN] = 0x......`).

**Struct at `+0x1000` — VERIFIED as layout in both FWs** (relative mapping;
absolute origin INFERRED, see residual R1):

- SND init `F_45BE4` (prologue `045be4`, sole caller `03e38d` with base
  `r11+0x940`): `jal 0x10C5B9` memset `0xFC0`, twelve `-0x3000` slots stride
  `0x120` (`+0x12C…+0xD8C`), then the field run `+0xFCC..+0x10C8` —
  `+0x100C=r23`, `+0x1038/0x104C=0`, per-slot config `+0x1084=0x200`,
  `+0x1088=0x300`, `+0x108C=0x400` (×2 slots), `+0x109C=0xFF00`.
- SND readers (all branchless leaves, codec-blind by construction):
  `0xD1203`-block (`+0x1680/8C/94`, `>>8`), `0xD1223`-block
  (`+0xFF4/8/C`, `>>8`), `0xD1241`-block (`+0x1000`, `+0x100C>>8`, `+0x1014`
  into `r5+0x8/0x10/0x14`).
- SND big writer `0xD70C2+` (per-slot blocks to `+0x2004`; `+0x100C`=runtime
  `r25`; gain-table tail `0xD7258+`, state-word branches only).
- DEC writers: `0x12A4E4` (`+0x1038=0x307880`, via `jal 0x12A483` in per-init
  `F_12A470`), `0x12A778`/`0x12ABB3` (`+0x1038=f(+0x924,frame)` runtime),
  `0x12AE1B` / `0x12B027` / `0x12B062` (`+0x104C` formula, Q3).

**Observed-cell mapping (base + offset → value → purpose-fit):**

| cell | AC-3 | DTS | static owner |
|---|---|---|---|
| `+0x100C` | `0x6570` | `0x65A0` | ptr-ish; SND init/writer regs, `>>8`-read |
| `+0x1038` | `0x1F87` | `0x1FAE` | count/gain; DEC init `0x307880` → runtime `f(+0x924)` |
| `+0x103F` | counter | counter | NO accessor either image (bounded negative; DMA/computed-addr) |
| `+0x104C` | `0x400` | `0x400` | quantum; DEC `[+0x918]<<7` ⇒ `[+0x918]=8` both |
| `+0x100E/F` | `0x202/1FF` | same | NO accessor either image; SND `r3=0x100E` is a RETURN STATUS (`0x6AA80→0x6AA43 pads+jr`), not a cell ref — trap avoided |

Mapping STRONGLY SUPPORTED (five offsets coincide in both FWs; writer
asymmetry picks DEC as the live core).

### Q2. No head/tail ring in the window; no SND codec gate — both closed

- Shape search (load→arith→store same-offset triples FW-wide): only stack
  save/restore false positives plus unrelated mixer scoreboards
  (`+0xE0/+0x548/+0x28` in `0x10xxxx–0x12xxxx` DSP code). No head/tail/
  occupancy triple on the `+0x1000` struct. Consistent with the archive:
  DEC hands over **pointers** (`desc+0xF8/0xFC`) + size (`+0xF0`), not a
  classic ring. The window carries **state both sides maintain** (DEC writes
  counts/quantum; SND `>>8`-reads and re-inits) — the payload lives behind
  the pointers, outside the window. Ring-inside-window: clean negative
  (bounded: full offset-shape search + 46K-cell forensic sweep found no
  burst-shaped payload).
- SND codec-conditional drain: NOT FOUND. Every SND accessor enumerated
  above — zero branches on codec or on DEC-set words. In particular SND
  **never reads** DEC's branch words `+0x1040/+0x1044` (zero accesses) and
  the gain-tail branches (`0xD7254`, `0xD72C0`) test struct words only.
  Mirror of the DEC question: CLOSED negative.

### Q3. `0x400` is computed FW state (`[+0x918]<<7`) — no host lever exists

- DEC, three sites: `0x12AE1B` (`[r3+0x104C]=[r3+0x918]<<7` + companions
  `0x1098=0x10000/0x10A0=2/0x10A4=0x14/0x1094=0x10000`; via `jal 0x12A489`
  in per-init `F_12A470`), `0x12B027`/`0x12B062` (same formula, internal to
  `F_12AFD0`, conditional on the `[0x90C]≠[0x109C]` change check).
  Observed `0x400` ⟹ `[+0x918]=8` under **both** codecs.
  `[+0x918] ← 0x12A3AB` (caller `r14`), `[+0x90C] ← caller r11` — per-instance
  init args; caller chain + per-codec values residual (R3).
- SND: the runtime `0x400` does **not** come from SND (sole `+0x104C` writer
  is the zeroer); SND's own `0x400`-configs sit at `+0x108C/+0x1074`
  (different cells).
- Host: SND Set table (`@0x459E70`, ids `sb−0x59 ≤ 0x76`) completely
  enumerated (`0x45A04C–0x45A3CC`) — max OFF **`0x348`**; DEC Set OFFs top
  out `≤0x11C8` (prior) with nothing at `0x918/0x104C/0x1000+`; full-module
  operand scan for `0x918/0x104C/0x1000–0x1100`: 77 hits, **zero at
  ≥0x400000** (all MsOS/QOS/AESDMA) — no audio/SHM function references them.
  → `0x400` is not an SE/host parameter and neither is its source
  `[+0x918]`. **Second lever: NONE — Q3 premise corrected, clean negative
  (module-wide).** Corollary: the `0x400`-quantum is healthy under both
  codecs, so it is not the block; the block stays the DEC word-gates.

### Residuals (bounded; none blocking the watchlist)

- R1: numeric DM origin of the struct base (cells-vs-base absolute proof
  needs a live base read — the publishing patch or JTAG per forensic close).
- R2: writers of `0x103F/0x100E/0x100F` (computed-address/DMA hypothesis
  untested; `r3=0x100E`-as-status closed, not a lead).
- R3: `+0x918/+0x90C` FW caller chains + per-codec values (`0x12A3AB`'s
  function head is above `0x12A380`; no prologue/`jal` inside — mid-function
  block).
- R4: ring payload (pointed buffers) location; SND reader entries
  (`0xD1203/0xD1223/0xD1241` — no `jal`, branch-reached; graph residual).
- R5: PM window (`type=0`) unmapped — out of scope.

Scripts (`C:\Users\k0994\AppData\Local\Temp\opencode\`): `s_recon.py`
(constants + hot imms), `s_q1.py` (struct accessors + zeroer), `s_q2.py`
(readers/counters/`0x1000`-uses), `d_q2.py` (DEC offset grep),
`s_q3.py`/`d_q3.py` (heads, writer-wide, DEC formula cluster),
`s_q4.py`/`d_q4.py` (callers, runtime `+0x1038`, `+0x918`),
`s_q5.py`/`d_q5.py` (join-`0x100E`, zeroer head, `+0x918` sweep),
`s_q6.py` (zeroer caller, writer tail), `u_snd*.py` (SND Set table+OFFs),
`u_dec3/4/5.py` (DEC Set table+OFFs), `u_scan.py` (module-wide operand
scan).