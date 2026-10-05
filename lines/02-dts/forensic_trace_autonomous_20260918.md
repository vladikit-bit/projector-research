# Autonomous forensic trace — F8/FC, F0/r12, downstream consumer (static + non-destructive runtime)
Date: 2026-09-18. New separate note. Does NOT modify prior artifacts. Prior reports are not authority; contradictions below are corrections with primary evidence.

Primary sources: `dec32_clean.txt` (authoritative listing, byte-matched `dec_full.bin`), `aeon_ORBIS32.sinc`, `utpa2k_stock.ko` / `mik_stock.ko` (.data embedded `mst_codec_r2` + `mst_codec_r2_MS12V22`), live device `C50A/mt5889` read-only (`adb`, `logcat -d`, `dmesg`, `/proc/asound`, `/vendor/lib/modules` hashes).

No patches, no firmware/binary writes, no Ghidra project modification, no settings change, no reboot, no destructive tests. No TEE/pkg investigation. No address-translation guesses.

## Executive summary
- F8/FC writers are in `F_45DE0 [0x45DE0,0x45E52)`: `0x45E17 sw_0 0xF8(r10),r3 | ec6a00f8` (L88880), `0x45E2F sw_0 0xFC(r10),r3 | ec6a00fc` (L88887), where values are `*(input+0x00)` / `*(input+0x04)` (L88875 `lwz r3,0x0(r13)`, L88884 `lwz r3,0x4(r13)`). Single direct caller `0x3E344 jal 0x45DE0 | e400f538` (L78689).
- Same-descriptor instance for the 0x411A6 chain is VERIFIED: all three `F_45E52` calls (`0x411F5 | e40098ba`, `0x41259 | e40097f2`, `0x4126D | e40097ca`) + `0x41283 jal 0x45FD1 | e4009a9c` + `0x41289 jal 0x4618E | e4009e0a`) use `r3 = root+0x14CCC`. `F_45E52` also reads F8/FC (`0x45EFB | ef0b00fa`, `0x45EFF | eeeb00fe`) and writes F0 (`0x45EC5 sw_0 0xF0(r11),r16 | ee0b00f0`), and calls builder itself at `0x45FC9 jal 0x45D07 | e7fffa7c`.
- Downstream consumer after `0x4618E` return is NOT proven. Caller `0x411A6` only does `r12`-gate + F0-alias clear, then returns 1. Upstream `0x3E961:0x3EF64 jal 0x411A6 | e4004484` stores return status (`0x3EF6E sw_0 -0x5EE4(r10),r0`) and calls `0x37D2D`; no F8/FC/F0 load. First opaque boundary = caller return with memory-resident A/B.
- F0: only `F0=0` on disabled path in output (`0x461AB sw_0 0xF0(r10),r14 | edca00f0`, L89194, r14=0). No `F0=N` store in `[0x4618E,0x4661E)`. Caller alias `*(r3+0x14DBC)` (`0x411C9/0x41221 sw_0 0x4DBC(r11),r0 | ec0b4dbc`) = `descriptor+0xF0`. r12 = `*(r3+0xA124)` (`0x41213 lwz r12,-0x5EDC(r11) | ed8ba126`), checked at `0x41225/0x4128D`. Writers outside chain BLOCKED.
- Runtime (non-destructive): `/vendor/lib/modules/utpa2k.ko MD5 2fc6e9fc...` = stock, `mik.ko c1421040...` = stock; `optee`, `audioserver`, `vendor.audio-hal`, `cpu-audio` running; all `/proc/asound/card0/pcm*p/sub0/status` = `closed`; `logcat -d` (22543 lines) shows only VDEC/MediaCodec, no DTS/SPDIF/IEC61937/DigitalTx/TEE-audio; `dmesg` (1288 lines) no MAD/DEC_R2/DspLoad/CheckHash/SHM matches in tail. Device idle — static idle-path analysis applies; live DTS handoff not observable without playback (not started; would change runtime state).
- SHM33: NO PROVEN RELATION (structural only, no numeric equivalence used).
- Std vs MS12: builder/gate/prep/copy identical; `F_45E52`/`F_4618E` heads identical at +0xADF7; `F_45DE0` near-match with jal-encoding-only diffs (same `jal 0x130D25` semantics, relative encoding differs).

## Verified findings
1. `F_45DE0` F8/FC init: L88875 `045e08 lwz r3,0x0(r13) | 0c6d02` → L88880 `045e17 sw_0 0xF8(r10),r3`; L88884 `045e25 lwz r3,0x4(r13) | 0c6d06` → L88887 `045e2f sw_0 0xFC(r10),r3`. `r10` = ctx (`045df3 mov r10,r3`), `r13` = input (`045de9 mov r13,r4`). Pre-fills via `0x130D25` (L88870 `045df7`, L88874 `045e04`, L88883 `045e21`, L88890 `045e39`). Ends `045e50 jr r9`.
2. `F_45DE0` sole direct xref: L78689 `03e344 jal 0x45DE0`. Caller `F_3E0DD [0x3E0DD,?)`: `03e341 add r3,r11,r14` (r14 contains `0x4CCC`), `03e33f add r4,r11` — descriptor-family construction, exact outer allocation BLOCKED above this.
3. `F_45E52 [0x45E52,0x45FD1)` (L88900-88995): entry `mov r11,r3; mov r3,r4; mov r12,r4`; F0 write L88943 `045ec5`; F8/FC reads L88964-88965; header staging copy loop L88966-88988 (`lhz 0x16C/16E/170/172`, `sw 0x0(r28)...`); second F8/FC load pair L88993-88994; bulk copy loop with `opcode_2B` (L89007) + `lh/sw` lane ops (L89008-89028); builder call L89036 `045fc9 jal 0x45D07` then `045fcd j 0x45EFB` (reuses same F8/FC readers). So `F_45E52` is a second builder→buffers path on the same descriptor, executed BEFORE parser/output in caller order.
4. Caller `0x411A6 [0x411A6,0x4139A)` order VERIFIED: `0x411F5 jal 0x45E52` → `0x41259 jal 0x45E52` → `0x4126D jal 0x45E52` → `0x41283 jal 0x45FD1` → `0x41289 jal 0x4618E` → `0x4128D bnei r12,1 → 0x411C9 (F0-alias clear)` else `0x41291 j 0x41228 (return-1 epilogue)`. All use `r10/r3 = root+0x14CCC` (L82513-82515 `041275 movhi, 041277 ori 0x4CCC, 04127B add r10,r23`; same pattern at 82463-82466, 82501-82503, 82507-82509).
5. Output `0x4618E [0x4618E,0x4661E)` F8/FC reads: L89206 `0461c7 lwz r3,0xF8(r3)`, L89211 `0461d6 lwz r3,0xFC(r10)`, L89225 `046201`, L89232 `046214`, L89237-89238 `046222/046226`, L89261/89263 `04626A/046271`, L89278-89279 `0462A0/0462A4`, L89297/89299 `0462DE/0462E6`. Utilities: `0x130D25` (L89210/89214) + `0x64369` (L89231/89236 `046210 jal 0x64369 | e403c2b2`, `04621E | e403c296`). No pointer escape (no store of A/B values elsewhere in range).
6. `0x64369` lives in `F_063B4F`; `0x130D25` in `F_130CEA`; both have 100+ xrefs image-wide (e.g. `0x130D25` 150+ `jal` hits) — generic utilities, not DTS-specific sinks.
7. Containing funcs: `0x3EF64` in `F_03E961` (L79190); `0x3E344` in `F_03E0DD` (L78497); `0x64369` in `F_063B4F`; `0x130D25` in `F_130CEA`. No direct `jal F_03E961` found with 8-digit search anomaly — upstream call graph above `F_03E961` BLOCKED (not pursued to avoid broad sweep).

## Runtime-confirmed findings (separate)
- Stock modules live: `utpa2k 2fc6e9fc...`, `mik c1421040...` match `patch_baseline/*_stock.ko`, not `*_installed.ko`.
- `Linux 4.19.116`, `C50A/mt5889`, `MStar-MAD-No00` card0 present, `optee` + audio services running.
- Idle: `pcm0p/1p/2p/3p status=closed`; no audio DTS/SPDIF evidence in `logcat -d` / `dmesg` tail. Confirms static analysis was on the active code, but live compressed-audio handoff unobserved.

## Corrections (original → corrected → evidence)
- `F0=N at end of 0x4618E` → NO such store in `[0x4618E,0x4661E)`; only `0x461AB F0=0` disabled-path. Evidence: exhaustive `0xF0(r` scan in range (auto dumps).
- `builder requires (N<<5)<=...` → requires `>`. Evidence: `sinc:514-515 bg.ble taken on <= to 0x460F3`, fall-through to builder checks.
- `F8/FC created in output` → created in `F_45DE0`, consumed in `F_45E52` + output. Evidence: L88880/L88887 vs L88964/L89206+.
- `0x45E52 unrelated` → `0x45E52` is same-instance pre-parser path with own builder call `0x45FC9`. Evidence: same `root+0x14CCC` construction + F8/FC/F0 accesses.

## DSP data-flow
```text
F_45DE0 init (F8=*(in+0), FC=*(in+4)) [VERIFIED, caller 0x3E344]
 ↓ VERIFIED? NO — link 0x3E344-instance → 0x411A6-instance NOT proven (different roots). LIKELY same struct family, BLOCKED same-instance.
F_45E52 (x3) header/buffer prep + builder 0x45FC9 [VERIFIED same-instance as parser/output]
 ↓ VERIFIED order
parser 0x45FD1 (0x28/0x30/0x34/0x38/N/L state) [VERIFIED]
 ↓ VERIFIED (0x41283→0x41289, same ctx)
builder dispatch (N==0x200/0x400/0x800 + geometry) [VERIFIED conditions, UNKNOWN semantic names]
 ↓ VERIFIED direct jal
output 0x4618E (fill/header-copy/0x64369 loop/sync restore) [VERIFIED to memory A/B]
 ↓ VERIFIED to memory, NO register escape
caller r12-gate + F0-alias clear / return-1 [VERIFIED]
 ↓ BLOCKED downstream (no F8/FC/F0 load in F_03E961 tail: 0x3EF6E/0x3EF72/0x3EF76)
opaque boundary: memory-resident A/B, no verified consumer
```

## F8/FC provenance
Writers: `F_45DE0:0x45E17/0x45E2F` from outer table (`r13`). No other writer in `[0x411A6,0x4661E)` chain.
Readers: `F_45E52:0x45EFB/0x45EFF/0x45F53/0x45F57`, `output:0x461C7/.../0x462E6` (list §Verified-5). Helpers `0x130D25` (fill) + `0x64369` (bit-reader) + direct `lh/lhz/sw` header + payload loops.
Instance: `0x411A6`-chain (3×45E52 + parser + output) VERIFIED same `root+0x14CCC`. `F_45DE0`-instance → `0x411A6`-instance link BLOCKED (different caller roots `F_03E0DD` vs `F_03E961`; no allocation/forward proof). Whole-image F8=400/FC=364 hits → no global claim.
External upstream: `F_45DE0` input table (`r13` = caller `r4`); `F_03E0DD` builds `r3/r4` from `r11` + `0x4CCC/0x4E40/0x4EE8` immediates + `0x4CC8/0x4E40/0x4E44` stores — allocation BLOCKED above.

## F0 provenance
Writers: `output:0x461AB (F0=0 disabled)`, `F_45E52:0x45EC5 (F0=r16)`, `caller:0x411C9/0x41221 (alias clear)`. No success-path `F0=N` in output range.
Readers: caller does NOT read F0, only writes alias; `F_03E961` tail does not read F0. Whole-image F0=327 hits → downstream reader BLOCKED.
Chain `output → F0 → caller → cleanup/return` VERIFIED as: output leaves F0 (success) or zeroes (disabled) → caller conditionally zeroes alias on `r12!=1` → returns `r3=1` (two paths: `0x411CD return-1` vs `0x41228 return-1` vs fallthrough `0x41295+` size-dispatch). F0 NOT named length/status.

## r12 provenance
`0x41213: r12 = *(r3+0xA124)` (via `r11=r3+0x10000`, `-0x5EDC`). No writer in caller. Checked `0x41225` (early `→0x411C9`) and `0x4128D` (post-output `→0x411C9`). Origin = outer struct field from caller input `r3` (which is `F_03E961` internal `r11`, ultimately from `0x3EEFD lwz r23,0x290(r11)` chain — see L79613). Nearest init / writers / acceptance semantics BLOCKED. No name given. No proven link to output acceptance beyond gating the alias-clear.

## SHM33 relation
NO PROVEN RELATION. No shared struct/field/index/state-block found between `HAL_DEC_R2_Set_SHM_PARAM(0x33)` host path and DSP ctx/output path. Small ctx offsets vs host `B+0xE02044/04` never equated.

## Standard vs MS12V22
Byte compare from `utpa2k_stock.ko` `.data` (`std .data+0x16BE18`, `ms12 .data+0x407F0C`):
- `F_45E52 head 64B`, `F_4618E head 64B` identical at `+0xADF7`.
- `F_45DE0 114B` near-match: only `jal 0x130D25` relative encodings differ (`e41d5e5c` vs `e42a0a84` etc.), semantics same callee. LIKELY identical behavior; NOT claimed without std listing decode.
- Direct same-offset compares at `0x411A6/0x45FD1/0x4618E` differ — expected (images diverge globally).

## Runtime observations (read-only)
`adb -P 5038 -s 192.168.0.183:5555`: `shell id` shell-only; `lsmod` shows `mik 6565888`, `utpa2k 23715840`; hashes stock; `tinymix` unusable (`Failed to open mixer` / 0 lines); `dumpsys` not needed; `logcat -d` 22543 lines, `dmesg` 1288 lines samples above. No state changed.

## Remaining blockers (ranked)
1. Same-instance link `F_45DE0 → 0x411A6` descriptor (allocation/forward proof).
2. r12 writer(s) at `*(root+0xA124)` + outer `0x290/0xA30/0xB94` gate semantics.
3. Any post-`F_03E961` consumer of A/B (requires caller-of-`F_03E961` graph + possible HW/DMA sink; speculative without playback).

## Highest-value next targets (static only, no state change)
1. `F_03E0DD` full + `F_03E961` full data-flow (r11/r13/r14 construction, descriptor allocation, return-status use).
2. `F_45E52` loop bounds `0x45EBB bgt` + `opcode_2B` regions (determines A/B valid length without runtime).
3. Caller-of-`F_03E961` + `0x37D2D`/`0x40B68`/`0x46F37` targets (closes or confirms HW boundary for A/B).
