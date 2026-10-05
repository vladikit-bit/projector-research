# FINDING — 2026-09-30 (night): verification of the latch report + two proofs beyond it

**Verification of `MUSE_TASK_b_latch_20260930.md ## RESULTS` against `dec33_realcode.txt`,
plus new static facts. Companion to `FINDING_host_tx_chain_20260930.md` (host side).**

## 1. Muse's report — verified, with two upgrades

* **Installers**: `0x3E19E addi r23,r11,0x7C4` → `0x3E1A6 sw_0 -0x5EF8(r10),r23`;
  `0x3EA04 addi r28,r11,0x7C4` → `0x3EA38 sw_0 -0x5EF8(r10),r28`. Value-unanimous. CONFIRMED.
* **Content writers**: `0x3DE7C sw 0x0(r10),r23` gated by `[r10+0x4]==1` (0x3DE78);
  `0x3DB5B jal 0x37F54` → `0x3DB62 sw 0x10(r10),r3`. Blind query+store. CONFIRMED.
* **Return-0 arms**: `0x4110C`, `0x4116D`, `0x4118A` (refill exit: calls `F_64296`/`F_64359`
  then `movi r23,0` — ALWAYS 0), `0x412C8`. CONFIRMED.
* **No codec-conditional write** — confirmed; the decision lives in compares and query
  returns. Agreed: no one-byte writer candidate.

## 2. PROOF 1 — `movhi` = `imm << 16` (from the slaspec), so K = 0x10000 and the DM space is 16-bit wrapping

`aeon.slaspec`/`aeon_ORBIS32.sinc`: `:bt.movhi … { i16_rD = i16_simm0_5 << 16; }`,
`:bn.movhi … { i24_rD = i24_iK13 << 16; }`. Therefore `movhi rX,0x1` → `rX = 0x10000`.

Consequences (resolves Muse's Q1 K-residual and R1):
* `[r10-0x5EF8]` (r10 = r3+0x10000) ≡ `[r11-0x5EF8]` (r11 = r3) — **the head read, the body
  reload (`0x41111`) and the call site (`0x4127D`) are the SAME address by 16-bit wraparound,
  not by coincidence. One latch object is now PROVEN, not "strongly supported".**
* `[r10-0x5EDC]` ≡ `[r11-0x5EDC]` — the "paradox word" is one and the same field in the pump
  head (`0x410EA`) and the body gate (`0x41213`).
* The caller's `+0x5EF8` stores (`0x3E7B7`) are a genuinely different field — Muse's
  exclusion was right.

## 3. PROOF 2 — the paradox word has NO writer in any static form

Independent sweep, both images (`dec33` + `snd33`):
* negative-form stores `sw -0x5EDC(...)`: **0 hits**;
* positive-form `0xA124` via `ori rX,r0,0xA124` (the ≥0x8000 idiom, verified present for the
  neighbour `0xA118` at `0x3E1C3/0x3E6F2/0x3E9F6/0x3F68C`): **0 hits**;
* `movhi 0x1` + `addi -0x5EDC` computed-address idiom: **0 hits**.

The value at `[base-0x5EDC]` (1 for AC-3 — it plays; unknown for DTS) comes from
boot-bulk-init / DMA / host-fed memory. **Live-watch is the only closer.**

## 4. NEW — a codec-conditional DISPATCH exists in the latch chain (a compare, not a write)

`F_3D952` (head `0x3D952 addi r1,r1,-0x48`; `r12 = r6` = its descriptor argument):

```
0x3DA4C lwz  r24,0xC(r12)        ; a type field of the DESCRIPTOR
0x3DA55 beqi r24,0x2,0x3DE18     ; type 2 -> dedicated prep arm
0x3DA59 beqi r24,0x1,0x3DDD2     ; type 1 -> dedicated prep arm
0x3DA5D lwz  r3,0x8D4(r10)       ; any other type (3,4,5…) -> query path:
0x3DA68 sw   0x4(r10),r3         ;   [+0x4] <- F_37F25()
0x3DA6B beqi r23,0x1,0x3DE6B     ;   continue only if [+0x18]==1
```

Both arms prepare the same fields the query path consumes (`[+0x18]` via `F_37EC2`/`F_37ED6`,
`[+0x8D4]`, `[+0x210]`); the type-1 arm even rejoins the query path (`0x3DE14 j 0x3DA5D`).
DTS's tag **4** (`0x19860 movi r23,0x4; 0x19862 sw 0xC(r12),r23`) misses both arms; a sibling
writer `0x18835 movi r23,0x2; 0x18837 sw 0xC(r12),r23` writes 2 (consistent with AC-3=2 /
DTS=4 / AAC=5, unproven).

**CAUTION — not yet established:** whether the `[r12+0xC]` of `F_3D952`'s descriptor is the
same field as `0x19862`'s `[r12+0xC]`. The descriptor is built by the `F_3E961` loop (Muse's
residual R2). If it IS the codec tag, this dispatch is the codec gate the whole project has
been converging on — and the old patch site `0x19861` returns with a *justified* target
value (2, not 5). If it is not, the dispatch is still a 3-way fork whose DTS arm must fail
somewhere in the 4-gate chain. Either way the next measurement is the same.

## 6. Late-session additions (static continuation)

* **Regex bug found and fixed (mine):** the listing format is `addr DIGIT mnemonic` and my
  caller-sweep pattern omitted the digit — several "0 callers" results this session were
  false negatives. Re-sweeps with the fixed pattern: `0x5381C` and `F_536F7` negatives
  **stand**; `0x4618E` has exactly one entry (`0x41289 jal`); **`F_3E961`'s sole caller is
  `0x37687`** (corrects Muse's R2 pointer, which led through `0x3F6B6` — that site calls
  `F_3D952`).
* **New paradox-word reader:** `0x3F60B lwz r23,-0x5EDC(r10); bnei r23,1,0x3EE8F` — the word
  also gates the whole codec-setup block in the pump-caller.
* **Descriptor array located:** the `F_3D952` loop walks `r7 = [r11+0x290]` (the same +0x290
  sub-object the pump head reads) with **stride 0xA4**; `F_3E961` runs on the same instance
  base (r10 = r4+0x10000 ≡ r4) fed by a template built at `r10+0x7AA0..0x7AC0` (`0x37644`).
* **RETRACTED (my own hypothesis, same day):** `0x18835 movi 0x2 / 0x18837 sw 0xC(r12)` is
  NOT an "AC-3 writes type 2" sibling. Its struct is the `0xE1BC8 + idx*0x78` table (and the
  neighbouring `0xE26B0 + idx*0x290` array) — a different object family entirely. Whether the
  `[r12+0xC]` of `F_3D952`'s descriptor carries the codec tag (4 vs 2 vs 1) remains
  **statically unresolved**; the live full-DM dump-diff (§5) decides it in one shot by
  showing the type cell's actual value under both codecs.

Because the DM space is 16-bit, **the entire addressable DM equals the entire mdb window
(0x0000–0xFFFF)** — every latch/stall word is readable, the base value irrelevant:

1. **Full-window dump + diff**: 16 × `read_dsp_sram_type=1 addr=0x0000/0x1000/…/0xF000
   len=0x1000` under AC-3, then under DTS; diff → every differing state word pops out
   (the B-latch, the pump latch `+0x4EE4`, the paradox word, the type field). Sample only
   while the decoder format is live (`Decoder format : DTS` / `AC3P`).
2. **`spdif_mode` Driver line per codec** — decides the host chain
   (`FINDING_host_tx_chain_20260930.md`).
3. **Live writes** via the untried `write_dsp_sram_type=1 addr=… value=…` — flip a diffed
   word (e.g. the paradox word → 1, or a B-latch candidate → 1) under DTS and watch the R2
   log (`play/state`) and the Pioneer. Reversible by reboot; nothing flashed.
