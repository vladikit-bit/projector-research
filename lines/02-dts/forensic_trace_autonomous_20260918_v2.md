# Autonomous v2 — upstream F_03E0DD/0x3E344, F_45E52 dissect, downstream F_03E961, r12 (static + non-destructive runtime)
Date: 2026-09-18. New file; prior artifacts untouched; prior reports not authority. Binary/listing-first; contradictions are corrections here, not edits there.

Sources: `dec32_clean.txt` + `dec_full.bin`, `aeon_ORBIS32.sinc`, `utpa2k_stock.ko` `.data` (`std .data+0x16BE18`, `ms12 .data+0x407F0C`), live `C50A` read-only (`adb -P 5038 -s 192.168.0.183:5555`, `logcat -d`, `dmesg`, `/proc/asound`, `dumpsys media.audio_flinger`, module hashes). No writes/patches/reboot/settings/Ghidra changes. No TEE/pkg. No playback (would change runtime state; noted as future gated test only).

Function bounds (via `addi r1` prologues): `F_03E0DD [0x3E0DD,0x3E392) L78497`, `F_03E961 [0x3E961,0x3F44F) L79190`, `F_037C94 [0x37C94,0x37FC6) L69991` (contains `0x37D2D`), `F_040607 [0x40607,?)` (contains `0x40B68` trampoline), `F_046F37 [0x46F37,0x47044) L90314`, `F_045DE0 [0x45DE0,0x45E52)`, `F_45E52 [0x45E52,0x45FD1)`, `F_0411A6 [0x411A6,0x4139A)`, `F_04618E [0x4618E,0x4661E)`.

## 1. F_03E0DD / 0x3E344 → F_45DE0 (FIRST PRIORITY)
- Entry `F_03E0DD L78497-78514`: `movhi r23,1; mov r14,r3; mov r3,r4; add r10,r4,r23 (r10=r4+0x10000); mov r11,r4; mov r12,r5`. So two-pointer input: `r14=orig_r3`, `r11=orig_r4`.
- Init fills via `0x130D25` (L78523 `03e123`, L78527 `03e12f`), state stores `0x290/0x294/0x411C/0x4120/0x4124/0x4128/0x4130` etc., then `0x67366`, `0x38D49` calls; return-status gates `0x3E1D7/0x3E208` (early `jr` on `r3!=1` / `==1` paths).
- Pre-call construction (L78680-78688): `03e325 ori r23,r14,0x4EE8; 03e329 add r23,r11; 03e32b addi r24,r23,0x800; 03e32f ori r4,r14,0x4E40; 03e333 ori r14,r14,0x4CCC; 03e337 sw_0 0x4E40(r10),r23; 03e33b sw_0 0x4E44(r10),r24; 03e33f add r4,r11 (r4=r4+r11); 03e341 add r3,r11,r14 (r3=r11+r14)`; then L78689 `03e344 jal 0x45DE0 | e400f538`.
- `F_45DE0` entry: `045df3 mov r10,r3 (ctx); 045de9 mov r13,r4 (table)`; L88875 `045e08 lwz r3,0x0(r13)` → L88880 `045e17 sw_0 0xF8(r10),r3`; L88884 `045e25 lwz r3,0x4(r13)` → L88887 `045e2f sw_0 0xFC(r10),r3`.
- VERIFIED: `F8=*(table+0)`, `FC=*(table+4)`, table = `F_03E0DD`-built `r4 = r11+(r14|0x4E40...)`. Post-call `F_03E0DD` stores return `r13` to `-0x5EE8/-0x5EE4/0x4138/0x413C(r10)` (L78693-78696) then `j 0x3E1ED`.
- BLOCKED above: allocation/owner/size/index of that outer table (i.e. who fills `table+0/+4` buffers, heap/ring/SHM/decoder-buf identity). `F_03E0DD` has 5 direct callers (`037810 | e400d19a`, `037986 | e400ceae`, `037a61 | e400ccf8`, `03f1ae | e7ffde5e`, `03f7d2 | e7ffd216`) — generic init, not pursued further this pass to avoid broad sweep. No semantic names given.
- Correction to v1 wording: `0x3E344` is not a function, it is the `jal` site inside `F_03E0DD`; `F_45DE0` is the callee function.

## 2. F_45E52 dissect (SECOND PRIORITY)
- A (reads): `045efb lwz r24,0xF8(r11) | ef0b00fa`, `045eff lwz r23,0xFC(r11) | eeeb00fe` (L88964-65); second pair `045f53/045f57` (L88993-94) same bases. Header halves `045f06 lhz r29,0x16C(r11)`, `045f10 lhz r27,0x16E`, `045f1c lhz r28,0x170`, `045f28 lhz r27,0x172`.
- B (src/dst): F8/FC are DESTINATIONS. Header path stores `0x16C...` halves to `A/B+index` (`045f14 sw 0x0(r28),r29`, `045f20 sw 0x0(r25),r27`, `045f2c sw 0x0(r25),r28`, `045f34 sw 0x0(r26),r27` + `lh/sw` lane copies). Bulk path copies stream `r27=r12+0x6E2C...` (`045f6d add r27,r12; 045f73 addi 0x6E2C; 045f7f lh r3,0x0(r27)`) → `045f85 sw 0x0(r29),r3` (A) / `045f88 sw 0x0(r30),r8` (B) + `slli/sw` word lanes. Sources: (i) same-ctx header, (ii) `r12`-derived stream (`r12` = entry `r4`, second arg). VERIFIED direction.
- C (returns): multiple `jr r9` exits (`045ee2`, plus loop `045fbb j 0x45EB5`, `045fcd j 0x45EFB`). `r3` at exit is helper result (`0x465F6`/`0x46618`/bulk counters), provenance unclear due to helper clobber + `inst16_4` gaps. Return-value discipline BLOCKED; effects (A/B fills + `045ec5 F0=r16`) VERIFIED.
- D (descriptor for 0x4618E?): NO — F8/FC pre-exist (from `F_45DE0`); `F_45E52` pre-fills contents + `F0` + header, same `root+0x14CCC` instance as parser/output (three calls `0411f5/041259/04126d` all build `r3=root+0x14CCC`). VERIFIED pre-fill, not creator.
- E (compressed/non-compressed branch?): gates on `r3` (`045e96 beqi r3,0→045EBF`, `045eb2 beqi r3,1→045EE4`), `r18` (`045ee4 beqi r18,0→045F47`), `0x808`-field (`045eed beqi r23,1→045FBF` builder-prep vs header-copy). No evidence to name compressed/non-compressed. No indirect jump/call in `F_45E52` (all direct `jal/j`, returns `jr r9`) — indirect-target analysis N/A here.

## 3. Downstream F_03E961 → 0x37D2D / 0x40B68 / 0x46F37 (THIRD PRIORITY)
- `F_03E961` tail after `03ef64 jal 0x411A6 | e4004484`: `03ef68 mov r17,r3; 03ef6a bnei r3,1→03EABD (early restore+jr)`; else `03ef6e sw_0 -0x5EE4(r10),r0; 03ef72 lwz r3,-0x5EF8(r10) (=r11+0x7C4, stored at 03ea38)`; L79645 `03ef76 jal 0x37D2D | e7ff1b6e`. So `0x37D2D` gets `root+0x7C4`, NOT A/B. VERIFIED non-A/B arg.
- `0x37D2D [0x37D2D,0x37D36)`: `037d2d lwz r23,0x24(r3); sfnei; cmovsi_2 r3; jr` — 0/1 status from `*(arg+0x24)`. No F8/FC/F0 touch. NO RELATION to A/B (different base/offset family; same numeric offset `0x24` is coincidence across structs — explicitly not equated).
- `0x40B68 [in F_040607)`: `040b68 movi r4,0; 040b6a ori r5,0x2E8; 040b6e j 0x130D25` — fill trampoline, called once from `F_03E0DD:03e303 jal 0x40B68 | e40050ca` (upstream init). NO RELATION to post-output downstream.
- `F_046F37`: fill + `jal 0x6585B` + `jal 0x46FA9` + stores to `0x61C8/0x7E44/0x61B4...`, called once from `F_03E0DD:03e31b jal 0x46F37 | e4011838` (upstream init, before `F_45DE0`). NOT downstream. Correction of v1 hypothesis listing it as downstream candidate.
- After `0x37D2D`, `F_03E961` continues: `03ef7a bnei→03F136`, `03ef7e lwz -0x5EF8`, `0x10/0x20` checks, `03ef93 jal 0x37ED6` (`lwz r3,0x428(r3)` getter), etc. — all status/pointer-chasing on `root+0x7C4` family, no `F8/FC/F0` load. VERIFIED absence in `[0x3EF64,0x3F040)` window for those three offsets (focused scan).

## 4. Caller of F_03E961 + return-gated submission
- Sole direct caller: L69472 `037687 jal 0x3E961 | e400e5b4` (in outer `F_0375xx` region). Args: `037683 mov r3,r10; 037685 mov r4,r15` where `037618 addi r15,r10,0xBA0` → `r4=root+0xBA0`.
- Return use (L69473-69512): `03768b addi r23,0,-0x3EC; 03768f beqi r3,1→0376B9 (success)` else restore+`jr` (failure path `037692-0376B7`). Success stores: `0376d9 sw 0x0(r22),r23 (r23=-0x5340(r14))`, `0376df sw 0x0(r21),r15 (root+0xBA0)`, `0376e9 sw 0x0(r20),r23 (root+0xCE8)`, `0376f7 sw 0x0(r19),r23 (root+0x158A8)`.
- This IS a return-gated downstream submission structure (four `sw 0x0(rXx)`), but stored payloads are root-offsets/table values, none loaded from `ctx+0xF8/FC`. Link `A/B → submission` BLOCKED (explicitly not claimed). `r22/r21/r20/r19` are caller inputs (queue/ring/descriptor slots LIKELY, not proven — no names given).

## 5. r12 / root state
- `041213 lwz r12,-0x5EDC(r11) | ed8ba126`, `r11=r3+0x10000` → `r12=*(root+0xA124)`. Image-wide `-0x5EDC(r10/r11)` readers: 12 hits (`03DEB9,03E87A,03EBB8,03F0E2,03F0F4,03F17C,03F60B,0400F7,040138,040EE2,0410EA,041213`), all `0x3D-0x41` family. Sole writer candidate: `0e7c33 sw_0 0x5EDC(r10),r26 | ef4a5edc` — distant region, base identity unproven. Nearest init / writers / `==1` semantics BLOCKED. No success/ready/DTS/bypass name. Runtime DSP-memory read not attempted (would need `/dev` probing; left BLOCKED rather than risk state change).

## 6. Runtime (non-destructive only)
- Baseline preserved: `pcm0p/1p/2p/3p status=closed`; `dumpsys media.audio_flinger | head -40` shows effect libs loaded, no active DTS track asserted (full dump not needed); module hashes still stock (rechecked this pass via earlier `md5sum` outputs, not re-run to avoid redundant daemon calls).
- No playback/smoke test started: would open PCM and change baseline; gated behind explicit static hypothesis that A/B reach a submittable queue — not met (see §4 BLOCKED). Recorded as future gated test, not executed.

## 7. Std vs MS12V22 (new funcs, byte compare from stock `.data`)
- Heads (64B): `F_03E0DD`, `F_03E961`, `0x37D2D`, `F_45E52` identical at `+0xADF7`, differ at same-offset (expected global divergence). `F_046F37` 269B: needle `ms12[0x46F37:+64]` NOT found verbatim in std, but `+0xADF7` 269B diff is single jal-encoding byte (`+0x0D: 0x1D vs 0x29`, `e41d3bb8` vs `e429e7e0`, both `jal 0x130D25`-family) — LIKELY same semantics with relative-branch relocation, not functional divergence. No identity assumed beyond bytes.

## 8. Decoder discipline
`bt.inst16_4` (e.g. `045ee4`-region gaps, `03e1ed` epilogues), `sw_1/sh/lhz` lanes, `opcode_2B/2E/3D`, `muls/divu`, `cmovsi_2/sfnei` flag semantics, `jr r9` returns — limitations marked BLOCKED where they gate exact packing/return-value proof. No gap filled by inference.

## NEW VERIFIED
- `F_03E0DD` bounds/inputs and `0x3E344→F_45DE0` arg construction (`F8/FC` table provenance one level up).
- `F_45E52` src/dst direction (F8/FC = destinations), pre-fill role, no indirect edge, branch conditions listed without names.
- `0x37D2D/0x40B68/0x46F37` are NOT A/B downstream (status/fill/upstream-init respectively) with exact evidence.
- `F_03E961` tail passes non-A/B pointer (`root+0x7C4`) onward; A/B absence VERIFIED in window.
- Return-gated submission at caller-of-`F_03E961` exists but payload≠A/B (BLOCKED link).
- Std/MS12 homology extends to new funcs (bytes above).

## NEW LIKELY
- Outer table (`table+0/+4` → F8/FC) is a buffer-descriptor pair (two adjacent pointers + `0x130/0x200`-length fills + `0xEC` flag at `045e3f`), but heap/ring/SHM/DMA identity unproven.
- Caller submission slots (`0x0(r22/r21/r20/r19)`) look like queue/ring hand-off, but descriptor-vs-pointer typing unproven.

## CORRECTIONS TO PREVIOUS
- v1 chain arrow `F_45DE0 → F8/FC → F_45E52` left instance link implicit → now explicit: `F_45DE0`-instance → `F_03E961/F_03E0DD`-family-instance BLOCKED same-instance; `F_45E52/parser/output`-instance VERIFIED same.
- `0x46F37`/`0x40B68` as downstream candidates → corrected to upstream-init/fill (call sites `03e31b/03e303` precede `03e344`).
- `0x37D2D` as possible A/B consumer → corrected to `*(root+0x7C4+0x24)` status predicate, no buffer touch.

## DATA FLOW (status per arrow)
```text
outer table (+0/+4) [BLOCKED alloc]
 ↓ VERIFIED load (0x45E08/0x45E25)
F_45DE0: F8/FC init [VERIFIED writer]
 ↓ LIKELY same family, BLOCKED same-instance to 0x411A6-root
F_45E52 (x3, same root+0x14CCC) header+bulk fill + builder 0x45FC9 [VERIFIED]
 ↓ VERIFIED order (0411F5/1259/126D → 041283)
parser 0x45FD1 [VERIFIED state]
 ↓ VERIFIED (041283→041289 same ctx)
builder dispatch 0x45D07 [VERIFIED direct jal, UNKNOWN names]
 ↓ VERIFIED
output 0x4618E → A/B memory [VERIFIED, no register escape]
 ↓ VERIFIED gate/clear (04128D→0411C9 alias), BLOCKED payload forward
F_03E961 tail (root+0x7C4 → 0x37D2D/0x37ED6...) [VERIFIED non-A/B]
 ↓ VERIFIED absence of F8/FC/F0
caller-of-F_03E961 return-gated stores [VERIFIED stores, BLOCKED A/B link]
 ↓ BLOCKED
FIRST OPAQUE BOUNDARY: A/B memory-resident after 0x411A6 return; submission slots carry root-offsets, not proven A/B
```

## RUNTIME CONFIRMATION
Stock-code applicability + idle baseline only (see header). No live DTS handoff observed; no state changed.

## REMAINING QUESTIONS
1. Who fills `table+0/+4` (F8/FC backing buffers) and with what lengths/flags?
2. What are submission slots `0x0(r22/r21/r20/r19)` at caller-of-`F_03E961`, and do they ever alias A/B?
3. What writes `*(root+0xA124)` (r12 gate)?

## NEXT HIGH-VALUE TARGETS (max 3)
1. Callers `037810/037986/037A61/03F1AE/03F7D2` of `F_03E0DD` — one level further up to buffer allocation.
2. `F_0375xx` caller frame (`r22/r21/r20/r19` provenance + `0x0()` store targets) — closes submission typing.
3. `0x465F0/0x465F6/0x46618` helpers (gate `F_45E52` loops/F0) — last static gate before any future gated playback test.
