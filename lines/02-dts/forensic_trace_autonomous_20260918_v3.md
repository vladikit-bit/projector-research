# Autonomous v3 — table allocation, A/B downstream, submission slots, gates (followed past next-3)
Date: 2026-09-18. New file; history untouched. Primary-evidence first. No writes/patches/reboot/settings/Ghidra changes. No TEE/pkg. No playback (gated, not met).

Bounds: `F_0375BE 0x375BE`, `F_0378A5 0x378A5`, `F_03E0DD [0x3E0DD,0x3E392)`, `F_03E961 [0x3E961,0x3F44F)`, `F_03F44F 0x3F44F`, `F_037C94 [0x37C94,0x37FC6)` (`0x37D2D`), `F_040269 [0x40269,0x40320)`, `F_045DE0`, `F_45E52`, `F_0411A6`, `F_04618E`, `F_046F37`, helpers `0x465F0/0x465F6/0x46618` (inside `F_04618E`).

## 1. F_03E0DD callers → table (followed, not just listed)
Direct xrefs `jal 0x3E0DD`: `037810 | e400d19a` (in `F_0375BE`), `037986 | e400ceae` + `037a61 | e400ccf8` (both in `F_0378A5`), `03f1ae | e7ffde5e` (in `F_03E961`), `03f7d2 | e7ffd216` (in `F_03F44F`).

- `037810`: pre `037804 mov r3,r10; 037806 mov r4,r15; 037808 mov r5,r10` → `F_03E0DD(r10, r15, r10)`. So `r11=r15`, `r14=r10` at callee. `r10/r15` are `F_0375BE` locals (outer root family; entry above window). Post: `037814 addi r23,0,-0x3EB; 03781e bnei r3,1→037692` — return-gated.
- `037986`: pre `03797c addi r16,r15,0xBA0; 037980 mov r3,r15; 037982 mov r4,r16; 037984 mov r5,r15` → `F_03E0DD(r15, r15+0xBA0, r15)`. Crucially `037978 jal 0x40269` with `r3=r15` just before: `F_040269` = `movi r4,0; ori r5,0xBA0; jal 0x130D25` + ~50 flag stores (`0xA18..0xB20`) — i.e. zeroes `0xBA0`-byte workspace at `r15`, then `F_03E0DD` uses `r15` and `r15+0xBA0` as its two bases. Post: `03798a beqi r3,1→037A86`.
- `037a61`: same `r3=r15, r4=r16, r5=r15` second call after `037a46/037a51 jal 0x130D25` fills of `*r11` (L69793 `037a3b lwz r3,0x0(r11)`). Same workspace, re-entered. Post: `sfeqi/cMov` return shaping (not named).
- `03f1ae` (in `F_03E961`): `03f1aa mov r3,r14; 03f1ac mov r4,r11` → `F_03E0DD(r14, r11)` — pass-through of own inputs (`r14=orig_r3, r11=orig_r4` saved at `03e9b8/03e9b6`). No new allocation; recursive init. Post: `03f1b2 mov r13,r3; 03f1b4 beqi→03F7FE; 03f1b8 beqi r3,0→03EA89`.
- `03f7d2` (in `F_03F44F`): `03f7ce mov r3,r14; 03f7d0 mov r4,r11` — same pass-through pattern. Post: return-gated `03f7d6/03f7d8`.

Table construction in `F_03E0DD` (L78680-78689): `r4 = r11+(r14|0x4E40)`, `r3 = r11+r14(0x4CCC)`, `jal F_45DE0`; `F_45DE0` then `F8=*(r4+0)`, `FC=*(r4+4)`.

Followed one more level: `table+0/+4` live inside the `r15`-workspace family zeroed by `F_040269` and touched by `0x3D600/0x3D62C/0x3D25C/0x3851B/0x385EC` fills/getters between the two `F_03E0DD` calls in `F_0378A5` — but no isolated `sw` creating the two pointer VALUES was found in-window. Whole-image `0x4E40(rX)` stores (e.g. `03e337 sw_0 0x4E40(r10)`) use a different base (`r10=r4+0x10000`), not the table base, so not equated.

FIRST OPAQUE (flow order): writer/initializer of `table+0` / `table+4` pointer values (backing-buffer allocation: heap/static/ring/SHM/decoder-workspace identity + size/index/flags). Evidence ends at `F_0375BE/F_0378A5` input boundary with a zeroed `0xBA0` workspace and helper-filled fields; the exact pointer-creating store is BLOCKED above. Deliberately not named.

## 2. A/B downstream via instance (not global hits)
- Instance rule enforced: only `root+0x14CCC` chain (`3×F_45E52 + parser + output` in `F_0411A6`) counts; global `F8=400/FC=364` hits ignored for identity.
- In `F_0411A6` after `041289 jal 0x4618E`: only `04128d bnei r12,1→0411C9 (F0-alias clear)` else `041291 j 0x41228 (return-1)`. No `F8/FC` load, no `A/B` register use, no store of `A/B` values elsewhere in `[0x411A6,0x4139A)`. VERIFIED absence.
- In `F_03E961` tail `[0x3EF64,0x3F040)`: `0x37D2D` arg is `root+0x7C4` (stored at `03ea38 sw_0 -0x5EF8(r10),r28`, `r28=r11+0x7C4`), subsequent `0x37ED6` args are `0x58/0x428`-derefs of it; `0x40DE0/0x64354/0x6434F/0x64296` calls operate on `root+0x14CCC`-vs-`root+0x7C4` families without moving `A/B` pointers. Focused `0xF8/0xFC/0xF0(r` scan in window: no hit. VERIFIED absence.
- Return path `F_03E961→caller`: restores (`03eadd-03eb1b`) and `jr`; `A/B` not in `r3` (return is status `r12`-family). Caller-of-`F_03E961` (`037687 jal 0x3E961 | e400e5b4` in `F_0375BE`) on `ret==1` does four `sw 0x0(r22/r21/r20/r19)` of root-offsets (`-0x5340(r14)`, `r15=root+0xBA0`-family, `root+0xCE8`, `root+0x158A8`), none loaded from `ctx+0xF8/FC`. So submission slots exist but payload≠A/B. Link BLOCKED, explicitly not bridged.
- Conclusion: `A/B` are memory-resident staging after `0x4618E`, modified only by `0x130D25/0x64369`/header loops inside `F_45E52/output`; no verified register/descriptor/queue/DMA propagation outward. Opaque boundary stands, now with instance-proof rather than global-hit argument. No покровительственный `SPDIF` literal required — structural search also negative.

## 3. Submission slots r22/r21/r20/r19
- In caller frame (`F_0375BE` around `037687`): `r22/r21/r20/r19` are live inputs/saved regs (used at `0376d2-0376f7` stores). Entry provenance above `0x37600` window — BLOCKED one frame up. Stored values: `0x0(r22)= -0x5340(r14)`, `0x0(r21)= r15`, `0x0(r20)= r10+0xCE8`, `0x0(r19)= r10+0x158A8` (L69502-69512). Readers of those four `0x0()` slots not traced (would be broad sweep beyond instance rule + queue typing unproven). Not named queue/descriptor. Indirect A/B link: none (values are root-offsets, not heap `A/B`). Recorded, not claimed.

## 4. Gates 0x465F0/0x465F6/0x46618 (before any gated playback)
All inside `F_04618E`, all direct, no indirect edge:
- `0x465F0 [0x465F0,0x465F4): lwz r3,0x7E3C(r3); jr` — getter for limit field. Called at `045e73 jal 0x465F0`.
- `0x465F6 [0x465F6,0x46616): lwz r24,0x7E3C(r3); ble r4,r24→0x46607 (alloc: muli 0x80C, add 0x6E24, sw 0x0(r5)) else sw 0x0(r5),0 + r3=0`. Called at `045eae jal 0x465F6` with `(r3=r12, r4=r10, r5=stack)`; return gates `045eb2 beqi r3,1→045EE4` (continue) vs loop `045eb5`. VERIFIED loop-control predicate on `*(r12+0x7E3C)`, not named DTS/compressed.
- `0x46618: sw_0 0x7E3C(r3),r0; jr` — clears same field. Called at `045ec1 jal 0x46618` with `r3=r12`. VERIFIED clear after loop.
- None prevent/allow `F_45E52` invocation itself (they are inside it); they gate its internal `0x45EB5` loop vs `0x45EBF` F0-path. Context fields changed: `0x7E3C` (cleared), stack `(r5)` pointer slot, `F0` downstream. Not DTS gates.

## 5. Runtime (gated use only)
- Baseline re-preserved without new state change: `ps -A` shows `audioserver 30479`, `mstar.hardware.audio.service 30480`, `tee-supplicant 188`, `AUDIO/AOUT` kernel tasks, `PCM_OfflineDete`; `dumpsys media.audio_flinger | head -40` effect libs only (no active track claim).
- `cat /proc/30480/maps` → `Permission denied` (SELinux shell). Recorded as runtime BLOCKED (no privilege escalation attempted).
- No playback: gated hypothesis (A/B reach submittable queue) not met (§2 BLOCKED), so smoke test not justified. Baseline unchanged.

## 6. Std vs MS12V22 (new)
- 64B heads: `F_03E0DD`, `F_03E961`, `0x37D2D`, `F_45E52` identical at `+0xADF7`, differ same-offset (global divergence, expected).
- `F_046F37` 269B: `+0xADF7` single jal-encoding byte diff (`+0x0D 0x1D vs 0x29`, same `jal 0x130D25`-family) — LIKELY same semantics, relocation-only. No functional identity assumed.

## NEW VERIFIED (this pass)
- Five `F_03E0DD` call sites with exact arg patterns + containing funcs + return gates.
- `F_040269` 0xBA0-workspace zeroing immediately before `037986` call (allocation-adjacent, not allocation itself).
- `F_45E52` internal gates `0x465F0/6/8` semantics (limit-field loop control + clear).
- `0x37D2D/0x40B68/0x46F37` exclusion as A/B downstream (status/fill/upstream) with call-site proof.
- Instance-scoped A/B absence in `F_0411A6` + `F_03E961` tail + submission payload mismatch.

## NEW CORRECTIONS
- v2 `NEXT 0x40B68/0x46F37 as downstream-adjacent` → corrected: both upstream of `0x3E344` (`03e303/03e31b` precede it).
- v2 `submission slots maybe queue` → narrowed: four stores VERIFIED, queue/descriptor typing explicitly withheld; payload≠A/B VERIFIED.

## Complete data-flow discovered (per-arrow status)
```text
outer roots (F_0375BE/78A5/3E961/3F44F inputs) [BLOCKED alloc]
 → F_03E0DD arg build (r11/r14) [VERIFIED]
 → 0x3E344 jal F_45DE0 [VERIFIED]
 → F8=*(table+0), FC=*(table+4) [VERIFIED load, BLOCKED table+0/+4 writer]
 → F_45E52 x3 (same root+0x14CCC) fill + builder 0x45FC9 [VERIFIED]
 → parser 0x45FD1 [VERIFIED same ctx]
 → builder dispatch [VERIFIED jal, UNKNOWN names]
 → output 0x4618E → A/B memory [VERIFIED, no escape]
 → caller r12/F0 gate [VERIFIED]
 → F_03E961 tail non-A/B [VERIFIED absence]
 → caller submission stores non-A/B [VERIFIED stores, BLOCKED link]
FIRST OPAQUE (flow order): table+0/+4 pointer-creating store above F_0375BE/78A5 inputs (workspace zeroed, filler helpers known, exact store unisolated).
DOWNSTREAM OPAQUE: A/B memory-resident; no verified handoff to submission/DMA/queue/HW.
```

## Runtime-confirmed facts
- Process/service/module baseline + idle PCM (prior pass) still holds; `audioserver/mstar-audio/tee` live; `/proc/PID/maps` unreadable as shell (permission boundary, not bypassed).

## Remaining blockers
1. `table+0/+4` writer (one frame above `F_0375BE/78A5` roots; candidates `0x3D600/0x3D62C/0x3851B` fills + `0xBA0` workspace).
2. Submission-slot reader graph (`0x0(r22/r21/r20/r19)` consumers) without assuming queue typing.
3. `*(root+0xA124)` (r12) writer — unchanged, now with 12-reader / 1-distant-writer evidence.
