# P-A control-flow verification record (pre-deployment, read-only)

**Scope:** prove the exact control-flow semantics of `0x45FF5: bn.sw 0x28(r3),r0 → bn.j 0x4600C` from raw bytes.
**Method:** raw bytes of `aucode_adec_r2_MS12V22_STOCK.bin` (md5 `4b7e9509…`), instruction boundaries cross-checked against `dec_work/dec32_clean.txt` (Ghidra-authoritative), branch semantics checked against `aeon_ORBIS32.sinc`. Whole-image scans (both 24-bit and 32-bit branch families) for callers/predecessors. No device access, no new patch.

---

## 1. Instructions executed between entry `0x45FD1` and `0x45FF5` (raw bytes)

```
045FD1  9c 30        bt.addi r1,-0x10    (2B)  frame push
045FD3  81 44        bt.trap 0x2         (2B)  frame pseudo-op
045FD5  81 26        bt.swst 0x6         (2B)
045FD7  81 62        bt.swst 0x2         (2B)
045FD9  81 80        bt.trap 0x0         (2B)
045FDB  9a e1        bt.movi r23,0x1     (2B)  r23 = 1
045FDD  0e e3 28     bn.sw 0x28(r3),r23  (3B)  desc->0x28 = 1        ← prepared flag, ALL paths
045FE0  0e e4 02     bn.lwz r23,0x0(r4)  (3B)  r23 = params->0x0
045FE3  0c 03 34     bn.sw 0x34(r3),r0   (3B)  desc->0x34 = 0
045FE6  0c 03 38     bn.sw 0x38(r3),r0   (3B)  desc->0x38 = 0        ← selector = 0, ALL paths
045FE9  89 43        bt.mov r10,r3       (2B)  r10 = desc            ← descriptor ptr, ALL paths
045FEB  22 e4 84     bn.beqi r23,1,+0x21 (3B)  row 1 → 0x4600C
045FEE  0d 64 12     bn.lwz r11,0x10(r4) (3B)  r11 = params->0x10
045FF1  d1 61 04 7a  bg.beqi r11,1,0x46080 (4B) row 2 → 0x46080
045FF5  0c 03 28     bn.sw 0x28(r3),r0   (3B)  ← PATCH SITE (row-3 fall-through)
```
Byte stream (contiguous, `0x45FD1–0x46010`): `9c30 8144 8126 8162 8180 9ae1 0ee328 0ee402 0c0334 0c0338 8943 22e484 0d6412 d161047a 0c0328 9ae0 9960 d2e10492 8127 8181 8163 8145 8434 8529 0c842a e4…` — boundaries confirmed.

## 2. Descriptor state initialized before `0x45FF5`

| field | site | value |
|---|---|---|
| `desc->0x28` | 0x45FDD | **1** (prepared) |
| `desc->0x34` | 0x45FE3 | 0 |
| `desc->0x38` | 0x45FE6 | 0 |
| `r10` | 0x45FE9 | desc |
| `r3` | function arg | desc (never clobbered 0x45FD1→0x4600C on any path) |
| `r4` | function arg | params |

## 3. What the jump skips (and what it does NOT)

**Skipped (0x45FF5 → 0x4600C):**
- `0x45FF5 bn.sw 0x28(r3),r0` (stock `desc->0x28 = 0`) — *skipping this is the fix*.
- `0x45FF8 9a e0 bt.movi r23,0` — register only; `r23` is clobbered by the `0x64149` copy callee before any read, and reloaded from `desc->0x28` at `0x46016`. Dead.
- `0x45FFA 99 60 bt.movi r11,0` — register only; `r11` is reassigned at `0x46036` (`bt.mov r11,r3`) before any read on this path. Note: the stock **row-1** path arrives at `0x4600C` with `r11` equally uninitialized in this frame — no new risk class is introduced.
- `0x45FFC d2 e1 04 92 bg.beqi r23,1,0x4608E` — stock-untaken on this path anyway (r23 would be 0); stock-reached via `0x4607C` with `r23 = desc->0x28`.
- `0x46000–0x46008` (`81 27 81 81 81 63 81 45 84 34`) — the bail-out **landing pad** (return sequence), only exercised at return time; the patched path returns later through the same pad (`0x46108 bg.j 0x46000 → 0x4600A bt.jr r9`), so stack discipline is preserved.
- `0x4600A 85 29 bt.jr r9` — the return; intentionally not taken.

**Not skipped:** every row-1 instruction. `0x4600C` *is* the stock row-1 target (`0x45FEB` branches exactly there); the continuation begins with the row-1 setup itself:
```
04600C  0c 84 2a   bn.lwz r4,0x28(r4)   r4 = params->0x28 (sub-struct ptr)
04600F  e4 03 c2 74 bg.jal 0x64149      copy 10 words: (r4) → desc+0x00..0x24
046013  0c 0a 2c   bn.sw 0x2c(r10),r0   desc->0x2c = 0
046016  0e ea 2a   bn.lwz r23,0x28(r10) r23 = desc->0x28
046019  22 e7 86   bn.bnei r23,1,+0x86  if desc->0x28 != 1 → 0x45FFA → (movi r11,0) → 0x45FFC(untaken) → return
```
**No memory write exists anywhere in the skipped region** (raw bytes are movi/movi/beqi/no-ops/jr only).

## 4. Register/flag delta at the common entry `0x4600C`

| register | stock row-1 entry | patched row-3 entry | read before reassignment? |
|---|---|---|---|
| r3 | desc | desc | — identical |
| r4 | params | params | — identical (reloaded at 0x4600C) |
| r10 | desc (0x45FE9) | desc (0x45FE9) | — identical |
| r23 | 1 (params->0x0) | params->0x0 (≠1) | **no** — clobbered by 0x64149 @0x4600F, reloaded @0x46016 |
| r11 | caller's value (uninit in frame) | params->0x10 (0x45FEE) | **no** — reassigned @0x46036 |
| r12 | caller's value | caller's value | **no** — reassigned @0x46038 |
| flags F | stale | stale | **no** — first flag op is `sfgtui` @0x46073; all intervening branches are register-compare (`bnei` 0x46019/0x4603B, `bgtui` 0x4604F) |

## 5. What the continuation requires from `desc+0x00..0x38`

| requirement | site | satisfied because |
|---|---|---|
| `desc->0x28 == 1` | 0x46016/0x46019, 0x46033, 0x46079 | set at 0x45FDD on ALL paths; the zeroing is the skipped instruction. If it were ≠1 the continuation bails **safely** (0x46019 → 0x45FFA → return), no crash path |
| `desc+0x00..0x24` (bit-reader: ptr/0x4/0x8 + saved 0x0c/0x10/0x14/0x18/0x20/0x24) | read by re-seek 0x64296 @0x4601E and getter 0x64354 @0x460AA | fully overwritten by the copy @0x4600F **before** any read |
| `desc->0x2c` | read in emit @0x46259 | written 0 @0x46013 before use |
| `desc->0x34` | written @0x46105 | zeroed @0x45FE3 before use |
| `desc->0x38` | gate @0x460B8, emit @0x461C4 | zeroed @0x45FE6, written with selector @0x46097 before any read |
| `desc+0xF8/0xFC` (emit buffer pointers) | emit @0x461C7/0x461D6 | set at **stream open** by `0x45DE0` (`sw_0 0xf8(r10),r3` @0x45E17 from `*(arg2+0)`, `sw_0 0xfc(r10),r3` @0x45E2F from `*(arg2+4)`; arg2 = `&control[0x4E40]`, holding buffer A/B = `control+0x4EE8+r11` / `+0x800`); each zeroed 0x200 B at open. Not touched by prepare at all |

## 6. Necessity of each skipped instruction

- **descriptor pointer/state:** `r10`/`desc->0x28=1` are *before* the patch site (not skipped). The only skipped descriptor write is the `0x28 = 0` zeroing — whose absence is the intended change.
- **saved bit-reader state:** supplied entirely by the copy @0x4600F (not skipped).
- **selector calculation:** reached through the continuation's own path: rate-table gate @0x46073 (`bn.bf` polarity sinc-verified: continue iff `field−5 ≤ 122`, i.e. field ≥ 5), `0x46079` reloads `desc->0x28`, `bg.j 0x45FFC` (@0x4607C), stock `bg.beqi r23,1` **taken** (r23 = 1) → `0x4608E` → `desc->0x38 = (field+1)<<5`. For the DTS-classified field 15 → selector `0x200`. Nothing skipped contributes.
- **buffer setup:** `desc->0xF8/0xFC` are open-time state (see §5); the skipped region writes no memory. Emit's memsets are len≤0-safe (`0x130D40 bg.blesi r5,0x0 → 0x130E07` exit), so a bail-out (selector 0) degenerates to the stock no-output behavior.
- **later DTS header emission:** header slots `desc+0x16C..0x173` are memset (0x4609E, 8 B) then built by `0x45D07` (DTS arm → data-type 0x0B) inside the continuation; emit's preamble copy keys on `desc->0x16C != 0`. All in-continuation.

## 7. `bn.j` encoding and target — byte-for-byte

- Field map (`aeon.slaspec` instr24): opcode (18,23), k18 displacement (0,17) signed.
- Patch word `0x2C0017` = op `0x0B` (`bn.j`) | k18 = `0x17` (23) → bytes big-endian `2c 00 17`.
- **Displacement base = PC, byte units** — proven on three stock `bn.j` instances: `0xD0FB+0x3F9=0xD4F4`; `0xD19F+0x355=0xD4F4`; `0x13FE13+0x2045=0x141E58`.
- Target = `0x45FF5 + 0x17 = 0x4600C` — exactly the stock row-1 branch target (`0x45FEB` rel 0x21 → `0x45FEB+0x21 = 0x4600C`, same base convention).
- Artifact decode-back: `2c 00 17 → op 0x0B, rel 23, target 0x4600c` ✓; stock `0c 03 28 → op 0x03 = bn.sw 0x28(r3),r0` ✓.

## 8. Diff extent

`cmp`-equivalent whole-file scan: differing offsets = **exactly [0x45FF5, 0x45FF6, 0x45FF7]**; all other 1,982,489 bytes identical to stock.
Artifact `aucode_adec_r2_MS12V22_PA_45FF5.bin` md5 `afde18ec92ffe0a960390a423d7f63c2`, sha256 `98a9a63124df3343a5bc60c66c97fa79d38f277ee658988aa293c25c03fb529c`.

## Additional hardening established during this verification

- `0x45FD1` (prepare) has **exactly one caller** in the whole image: `0x41283` (`bg.jal`) — full-image scan over both the 32-bit (`bg.jal/bg.j`, op 0x39) and 24-bit (`bn.jal`, op 0x0A) families.
- `memset` helper `0x130D25` returns immediately for len ≤ 0.
- Open-time init `0x45DE0` (called from the open sequence @0x3E344) memsets `desc[0x0..0x173]` and `desc[0x3C..0x16B]`, sets `desc->0x7C/0x80 = 48000`, `desc->0xEC = 6`, and installs the buffer pointers — **codec-independent**, so emit's prepared-path dereferences are valid for DTS streams exactly as for any other stream.
- The caller (`0x411B0…`) has two modes selected by `control[0xB94]`: a pool-drain consumer (`0x45E52`, which shares the same buffers and even calls the same header builder `0x45D07` when `entry->0x808 == 1`) and the prepare+emit mode gated by `control[0x14EE4] == 1`. P-A modifies only the prepare+emit mode's row-3 arm; in consumer mode it is inert. Rows 1/2 (the AC3 route) are untouched in both modes.
- Worst-case failure of the patched path (bitstream parse bails early) leaves `desc->0x28 = 1`, `desc->0x38 = 0` → emit memsets 0 bytes and publishes `desc->0xF0 = 0` → observable behavior identical to stock row 3 (no output). The patch can only *add* output when the stream genuinely parses with the DTS field (15 → selector 0x200 → DTS arm → header 0x0B → `desc->0xF0 = 0x200`).

## VERDICT

**A) P-A jumps into a genuinely valid common continuation; all required state (descriptor pointer, prepared flag, bit-reader state via the in-continuation copy, selector, open-time buffer pointers) is initialized before any use; the skipped region contains no memory writes and no required setup; encoding and 3-byte diff are proven byte-for-byte.**

**READY TO DEPLOY** (deployment itself remains gated on explicit user authorization).
