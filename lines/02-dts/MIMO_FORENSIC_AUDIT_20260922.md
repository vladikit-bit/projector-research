# MIMO FORENSIC AUDIT 20260922 — Thundeal TD98 Pro / C50A (MT5889)

**Classification:** Static + prior-runtime forensic continuation. STRICT READ-ONLY.  
**No patching, flashing, module loads, `/proc` writes, sysfs/settings/props/mixer writes, ftrace, logcat -c, or debugfs mount. No projector changes of any kind.**  
**Corpus preservation:** no files deleted, moved, or overwritten in prior report/binary/script trees; TEMP analysis scripts only appended/replaced in `C:\Users\k0994\AppData\Local\Temp\opencode\`.  
**Device:** `192.168.0.183:5555` (ADB root previously succeeded; SELinux Permissive). This report is produced from already-pulled binaries + prior probes; no new device mutation.  
**Reference:** TCL T615 V440 `update.img` = static architectural reference only — **never** a working-DTS reference.

---

## 1. Executive summary

The DTS-passthrough failure model on TD98 is a **host AUTH gate + SHM latch chain**, not a missing SoC codec path. AC3 works because it bypasses AUTH; DTS fails when `MDrv_AUTH_IPCheck` cannot authorize the DTS feature bits and/or SHM38 is out of range.

**H1 (does `gIpAuthVars` ever get provisioned?) — LIKELY NO invocation on this firmware as shipped; live value remains BLOCKED.**

Provisioning machinery is fully present:

| Layer | Artifact | Role |
|--------|-----------|------|
| Kernel | `utopia_proc_ioctl` cmd6 `0xC08C5506` | sole writer of `gIpAuthVars` (0x8C alloc) |
| Kernel | cmd7 `0xC0125507` | `gCusID` (2 B) + `gCusHash` (16 B) |
| Userspace | `UtopiaSetIPAUTH` ×3 | `open("/proc/utopia")` → ioctl6 → ioctl7 |
| Bridge | `libdrvIPAUTH` `MApi_AUTH_Process` | **only identified caller** of `UtopiaSetIPAUTH` (PLT @0x2454 → `0xb1c`) |

But across **every ELF pulled from the midaemon/cpu_audio process map** (midaemon, cpu_audio, libMsOS, liblinux, libdrvIPAUTH, libdrvIPAUTH_dev, libapiAUDIO, libhashkey_ca, libutopia_td98):

- **No binary imports `MApi_AUTH_Process`.**
- **No BL / MOVW-MOVT / `.rel.dyn` function pointer materializes `MApi_AUTH_Process`.**
- **libutopia_td98 exports `MApi_AUTH_Process` but has 0 local BLs to it.**
- **midaemon** calls only `HASHKEY_Verify` (hashkey.sign path); does **not** call `UtopiaSetIPAUTH`.
- **cpu_audio** `NEEDED libdrvIPAUTH.so` but **zero AUTH-related PLT imports**; constructor is ctor-guard/`blx` thunk, not `MApi_AUTH_Process`.
- **`/vendor/tvconfig/config/vir_hashkey.txt` MISSING** (midaemon string target); no IPAUTH dmesg/logcat lines; no `_IOR` read-back.

Therefore: **H1 = LIKELY never provisioned (no reachable caller found); live `gIpAuthVars` = BLOCKED.**  
Combined causal model: NULL/unprovisioned `gIpAuthVars` → `MDrv_AUTH_IPCheck` returns 0 → `AV+0x4D8` stays 0 → AUTH arm `0x45F6BC` **FAIL (r1==0)** → DTS blocked while AC3 (latch 3 → SHM33, no AUTH) still works.

**Corrections applied this session** (see §7): prior “only string-grep” / “midaemon maps libutopia” / buggy MOVW scanner claims are superseded by dynsym + ARM PLT + PC-relative decode evidence.

---

## 2. Independently verified (static)

### 2.1 Kernel AUTH chain (unchanged; reconfirmed against stock `utpa2k`)

- `MDrv_AUTH_IPCheck(arg)` @ `0x1390C`: `*gIpAuthVars` NULL → log + **return 0**. Non-null: bit-test `struct[0xA+N-1-(arg>>3)]`, bit `arg&7`.
- **Sole `gIpAuthVars` writer:** utopia ioctl nr6 cmd `0xC08C5506` (`utopia_proc_ioctl` @ `0x13D00`, alloc 0x8C @ `0x13D88`).
- **Hash key writer:** cmd `0xC0125507` @ `0x13E38`.
- `MDrv_AUDIO_CheckHashkey`: sole production store `AV+0x4D8=r8` @ `0x424878`. Ladder: IPCheck(0xF)→1 / IPCheck(0x12)→2 / IPCheck(7)→3 if >0; `AV+0x503==1` skips stores; after stores `HAL_AUDIO_GET_INIT_FLAG()==1` freezes `AV+0x503` (init-race → r8=0 frozen).
- `MDrv_AUDIO_SET_IPAUTH_GROUP` @ **`0x424BD8`**: group0→0x83, group1→0x84; IDMA base `0x112A82` (mask `0x112AC0`); callers ApplyHashkey / SetAudioParam2 / SetSystem / SetDecodeSystem.
- AUTH arm @ `0x45F6BC`: SHM38=`Get_SHM_INFO(0x38)`; `AV+0x4D8`∈{1,2}→subtable `0x45F6F4`; ==3→`0x45F89C`; **==0 or ≥4 → FAIL**; SHM38∉[1..7]→FAIL. Second gate @ `0x45EAE8`: IPCheck(0x7D), IPCheck(0x0A), `AV+0x4D0==4 && r8!=0 && SHM4D!=0`.
- No `_IOR` utopia read ioctls; `gIpAuthVars` not readable from userspace via utopia.

### 2.2 Userspace `UtopiaSetIPAUTH` — three full ARM implementations

All three are **ARM (A32) state**, not Thumb (`thumb_bit=0` on dynsym). Identical control flow:

1. `open("/proc/utopia", O_RDWR=2)` — fail → `"open /proc/utopia fail"`, return 1  
2. `malloc(0x8C)` + memcpy arg  
3. **`ioctl(fd, 0xC08C5506, buf)`** (`movw #0x5506` / `movt #0xc08c`)  
4. success: `malloc(0x12)` (2 B cusID + 16 B hash from arg)  
5. **`ioctl(fd, 0xC0125507, buf)`** (`movw #0x5507` / `movt #0xc012`)  
6. close/free, return 0  

| Binary | VA | Size | File off |
|--------|-----|------|----------|
| `libMsOS.so` | `0x260B4` | 424 | `0x260B4` |
| `liblinux.so` | `0x3E3B4` | 424 | `0x3E3B4` |
| `libutopia_td98.so` | `0x50BA28` | 376 | `0x50AA28` |

ELF: `machine=40` (ARM), flags `0x5000202` / `0x5000200`. Decoded with Capstone `CS_MODE_ARM` (Thumb decode of these VAs was garbage — earlier Thumb-only scans were wrong).

### 2.3 Cross-library import/export map (dynsym-proven)

| Symbol | Exports | Imports (UND / `.rel.plt`) |
|--------|---------|----------------------------|
| `UtopiaSetIPAUTH` | libMsOS, liblinux, libutopia_td98, libdrvIPAUTH(+dev) as UND | libdrvIPAUTH, libdrvIPAUTH_dev, libutopia_td98 |
| `MApi_AUTH_Process` | libdrvIPAUTH, libdrvIPAUTH_dev, libutopia_td98 | **none in any pulled ELF** |
| `MDrv_AUTH_IPCheck` | libdrvIPAUTH(+dev), libutopia_td98 | libapiAUDIO, libdrvIPAUTH, libutopia_td98 |
| `MDrv_AUDIO_CheckHashkey` | libutopia_td98 (local) | libapiAUDIO, libutopia_td98 (`.rel.plt`) |
| `HASHKEY_Verify` | libhashkey_ca | **midaemon only** |

`libdrvIPAUTH`: `FLAGS_1=NOW` (BIND_NOW); `UtopiaSetIPAUTH` GOT `0x15F84`, PLT `0xB1C`.  
**`MApi_AUTH_Process+0x2454` → `bl 0xB1C` = `UtopiaSetIPAUTH@plt`** (PLT layout start=20, stride=12 — validated).

### 2.4 midaemon (pid 193) — full call graph facts

- **Mode:** ARM A32 (not Thumb). `main` @ `0x1D3C` size **6340** (dynsym).  
- **`.rel.plt` 77 imports; PLT layout: 20-byte header + 12-byte stubs** (e.g. `HASHKEY_Verify` @ `0x1BC8`, `system` @ `0x1B50`, `MI_FS_Fopen` @ `0x1AC0`).  
- **`main` ARM BL count = 184**; interesting PLT uses: MI_FS_*, MI_SYS_*, iniparser_*, `access`, `system` (1× @ `0x21C4`), `strstr`, mutex/sem — **no `UtopiaSetIPAUTH`, no `MApi_AUTH_Process`, no `MDrv_AUTH_*`.**  
- **`HASHKEY_Verify`: exactly one call** @ `0x3164` (`bl 0x1bc8`). Format string: **`HASHKEY_Verify=(%u)(%d)\n`**. Non-zero return → log @ `0x35D8` (line 0x1CF) then continue.  
- Paths in binary: **`/vendor/tvcertificate/hashkey.sign`** (`0x4A50`), **`/vendor/tvconfig/config/vir_hashkey.txt`** (`0x4C9C`) — **device: file MISSING**.  
- BSS data (not functions): `IP_Cntrol_Mapping_1..4` (8 B), **`Customer_hash` (16 B)**, `CID_Buf` (32 B) @ `0x1600C..0x160B4`.  
- **NEEDED includes libMsOS, liblinux, libdrvIPAUTH, libapiAUDIO, libhashkey_ca** — so `UtopiaSetIPAUTH` **resolves** into midaemon’s address space (BIND_NOW satisfied ⇒ export exists), but **nothing in midaemon invokes it**.

### 2.5 cpu_audio (pid 479)

- Size 7371528; entry `0x2A5F4`; opens `/proc/utopia` (fd14/15) per prior probe.  
- **`NEEDED libdrvIPAUTH.so`** — but **full `.rel.plt` (128 names) contains no AUTH / UtopiaSetIPAUTH / HASHKEY / MDrv_AUTH symbols** (only `fopen`, `mi_audio_*`, MI_SYS/OS, TEEC_*, etc.).  
- `.init_array`: `0x2A778`, `0x294C0`, `0x2985C` (local). Strings: **no** `UtopiaSetIPAUTH`, `MApi_AUTH`, `/proc/utopia`, `IPCheckArg`, `HASHKEY`.  
- Conclusion: cpu_audio loads libdrvIPAUTH as a **dependency without direct AUTH PLT calls**; no evidence it runs `MApi_AUTH_Process`.

### 2.6 `UTOPIA_AUTH_IPCHECK_ARG` string sites (PC-relative, not MOVW)

ARM uses `ldr rN,[pc,#imm]` + `add rN,pc,rN`. Fixed scanner found near-string `add pc` sites in libMsOS/liblinux/libutopia_td98 (error-log paths around UtopiaOpen/ResourceObtain/UtopiaSetIPAUTH). **Absolute literal-pool / MOVW hits were 0** (correct — these libs are A32 PIC). Full body of a dedicated `UTOPIA_AUTH_IPCHECK_ARG` wrapper still not isolated as a named dynsym; strings remain proof the wrapper type exists in all three libs.

### 2.7 mik `mi_audio_GetCodecType` (prior session, still valid)

`@0xB0720`: runtime origin of **`[2185] Fail to get DTS codec type!`** via `SetDecodeSystem_b` xrefs `MI_AUDIO_Start+0x4c0` / `MI_AUDIO_GetAttr+0xd20`. Distinct from host AUTH gate; remains a downstream symptom when AUTH/latch path leaves decoder type unset/invalid.

---

## 3. Runtime observations (prior sessions; not re-executed for this report)

- `adb root` OK; read surface: no `/proc/kcore`, `/dev/mem`, `/dev/kmem`; `kptr_restrict=2`; tracefs off; debugfs unmounted; `/proc/utopia_mdb/audio` cat empty (write forbidden); `/sys/kernel/mik/MI_AUDIO` denied.  
- **Holders of `/proc/utopia`:** midaemon 193 fds 15/16; cpu_audio 479 fds 14/15; factory 1551 (prior).  
- midaemon maps: midaemon, ld-2.21, binder, /dev/malloc, /dev/miomap (truncated list); **does not map libutopia**.  
- **`vir_hashkey.txt` MISSING**; `hashkey.sign` 306 B MHKD present.  
- No dmesg/logcat IPAUTH / `UtopiaSetIPAUTH` / `open /proc/utopia fail` lines observed.  
- AC3 playback via `org.xbmc.kodi/.Main` OK; DTS blocked (historical invariants: mid-play fail, Codec:9, −1, NONE — unchanged).

**Live values still unread:** `AV+0x4D8`, latches, `AV+0x503`, `AV+0x440/0x444`, `AV+0x4C8`, SHM38, `gIpAuthVars`.

---

## 4. Static analysis method (this session)

- **ARM vs Thumb:** ELF `e_machine=40`; dynsym `thumb_bit=0` → **A32**. Capstone `CS_MODE_ARM` for utopia libs and midaemon.  
- **PLT:** classic ARM **20-byte PLT0 + 12-byte stubs**; order matches `.rel.plt`. Wrong start=16/Thumb main caused false “0 calls” earlier.  
- **BL scan:** A32 `imm24<<2 + src+8`.  
- **String xref:** `add rD, pc, rN` + literal `W = S - (A+8)` within ±0x600.  
- **MOVW/MOVT A32:** `0xE30xxxxx` / `0xE34xxxxx`, imm16 = `imm4:imm12`. Thumb decoder (i-bit) used only where `thumb_bit=1` (kernel/mik).  
- **Relocs:** `.rel.plt` JUMP_SLOT type 22; **zero `.rel.dyn` ABS32/GLOB_DAT to `MApi_AUTH_Process`**.  
- Scripts: `h1_arm_xref.py`, `h1_callers.py`, `h1_plt_callers.py`, `h1_plt_fix.py`, `h1_reloc_fnptr.py` under `Temp\opencode\`.  
- **Stability:** avoided unbounded glob loops; Python at `...\WindowsApps` → real 3.14 + capstone 5.0.7; no 19 MB full-text Thumb loops.

---

## 5. Current causal model

```
                    [USERSPACE - NOT REACHED on evidence]
  hashkey.sign ──► midaemon HASHKEY_Verify (only)
  vir_hashkey.txt  MISSING ──► fopen fails
  MApi_AUTH_Process ──► UtopiaSetIPAUTH ──► ioctl 0xC08C5506
         ▲                    │                    │
         │ no importer        │ 3 providers exist  ▼
         └────────────────────┴──────────► gIpAuthVars (0x8C)
                                            gCusID/gCusHash (ioctl7)
                            │
                            ▼
              MDrv_AUTH_IPCheck ── NULL? return 0
                            │
                            ▼
              CheckHashkey: AV+0x4D8 = r8 ∈ {0,1,2,3}
                            │
              ┌─────────────┴──────────────┐
              ▼                            ▼
     AUTH arm 0x45F6BC              AC3 path (no AUTH)
     r1==0 or SHM38∉[1..7]         latch 3 → SHM33
     → FAIL (DTS)                   → WORKS (AC3)

  Secondary: mik GetCodecType fail → [2185] (downstream)
  Distinct: R2 DSP AEON MMIO 0xB000001E==5 (DSP license gate)
```

- **H1 (strengthened):** provisioner **present in process** (libMsOS/liblinux export + libdrvIPAUTH NEEDED) but **no invocation path** in any pulled ELF → **LIKELY never runs**; **gIpAuthVars live = BLOCKED**.  
- **H2:** `AV+0x4D8∈{1,2,3}` but SHM38∉[1..7] — still open if some path sets r8 without gIpAuthVars (does not match IPCheck-null→0).  
- **Init-race freeze:** `AV+0x503` — unchanged.  
- **R2 DSP gate** — parallel DSP-side condition; not substituted for host AUTH.

---

## 6. New findings (this session)

1. **Three real `UtopiaSetIPAUTH` bodies (ARM)** — ioctl constants `0xC08C5506` / `0xC0125507` confirmed in libMsOS/liblinux/libutopia_td98 disassembly.  
2. **`MApi_AUTH_Process` is the sole `UtopiaSetIPAUTH` caller** in libdrvIPAUTH (`0x2454` → PLT `0xB1C`); **zero importers** of `MApi_AUTH_Process` corpus-wide.  
3. **No function-pointer / MOVW-MOVT / `.rel.dyn` registration** of `MApi_AUTH_Process` (0 hits). libutopia_td98 exports it with **0 internal BLs**.  
4. **libdrvIPAUTH `.init_array` @ `0x15E64` → ctor `0xD4C`:** standard `frame_dummy` (load guard → optional `blx r3`), **not** AUTH provisioning.  
5. **midaemon HASHKEY path decoded:** one `HASHKEY_Verify` @ `0x3164`; format `HASHKEY_Verify=(%u)(%d)\n`; fail log line `0x1CF`. **This is signature verification, not `gIpAuthVars` ioctl provisioning.**  
6. **cpu_audio:** `NEEDED libdrvIPAUTH` with **empty AUTH PLT** — dependency without direct AUTH calls.  
7. **midaemon BSS `Customer_hash` (16 B)** mirrors `gCusHash` size — data slot only; no evidence of write via `UtopiaSetIPAUTH` (that path never called).  
8. **PIC string refs** are PC-relative `add pc`; absolute/MOVW scans on these libs are false negatives if used alone.  
9. **midaemon `main` is A32 size 6340** — prior Thumb/`plt_map=0` runs discarded.  
10. **H1 verdict written:** see §5 and §11 (LIKELY not provisioned + BLOCKED live).

---

## 7. Corrections (carry into any future patch doc)

| Wrong / superseded | Correct |
|--------------------|---------|
| “libMsOS/liblinux only string-grep `UtopiaSetIPAUTH`” | Both **EXPORT** full 424 B ARM implementations |
| “midaemon maps libutopia → provider” | midaemon does **not** map libutopia; provider = libMsOS/liblinux |
| “libutopia_td98 sole provider” | **Three** providers: libutopia_td98, libMsOS, liblinux |
| “Thumb MOVW scanner found 0 xrefs ⇒ no code refs” | Scanner missed **i-bit**; these libs are **A32** — use ARM MOVW + PC-rel |
| “midaemon main has 0 BL” | **184** A32 BLs; PLT start=**20** stride=**12** |
| “`+0x4D8` four writers” | Sole writer CheckHashkey `0x424878` (+alloc zeroing) |
| “SET_IPAUTH_GROUP not located” | **`0x424BD8`** |
| “`.L.str.77` = copy_from_user” | `[AUDIO][INFO]` |
| “`.L.str.105` / `0x45F6B0` = AUTH arm” | Dolby AAC error |
| “raw ioctl bytes in binaries” | movw/movt constructed |
| Disproven still | `+0x10=0xFF` kill; query-time DTS latch 0xB; 0xB→SHM38 arm; MAD(0x4D); status−1 fail; V440 working-DTS; `[2172]` fail; NODTS gate; subtable returns stored; V440 as DTS donor |

---

## 8. TCL T615 comparison (static only)

- T615 `lib_libutopia.so` / `tclaudio/lib_libutopia.so`: `UtopiaSetIPAUTH @ 0x8BACE8` **same ioctl body** as TD98.  
- Extracted `UtopiaSetIPAUTH` matches TD98 control flow (open → 0xC08C5506 → 0xC0125507).  
- **Not** used as proof TD98 DTS works; V440 remains architectural reference only (`AUDIT_UPDATE_20260919.md`).  
- TCL `lib_modules_utpa2k.ko` stripped (shoff=0) — strings/jump-table diff only; not required for H1.

---

## 9. Stealth / Union Alpha cross-checks

- No new Stealth/Union Alpha-specific binaries ingested this session.  
- Cross-check stands: vendor software decides DTS availability on identical silicon (community/T615 phenomenology) — consistent with **software AUTH gate**, not missing DSP codec.  
- R2 AEON MMIO gate (`FINDING_rootcause_license_gate.md`) remains **separate** DSP-side condition; host `AV+0x4D8`/SHM38 chain is independent failure layer.

---

## 10. DSP / auth analysis

- Host AUTH (this report): `gIpAuthVars` bit vector + `AV+0x4D8` + SHM38 + SET_IPAUTH_GROUP IDMA `0x112A82`.  
- DSP AUTH (AEON): `MMIO 0xB000001E==5` license gate — **distinct**, must not be conflated.  
- mik `GetCodecType` `[2185]`: decoder-type query after host path — **downstream**.  
- `HAL_AUDIO_CheckHashkeyDone` stub 0; `UtopiaSetIPAUTH` kernel symbol `0x6224` stub (real work is userspace→ioctl).  
- AUTH feature bits 7 / 0xF / 0x12 / 0x7D / 0x0A: still **unnamed** (LIKELY decode vs libdrvIPAUTH/libutopia strings pending).

---

## 11. Evidence states

### VERIFIED (static + prior runtime)

- Full kernel IPCheck / CheckHashkey / SET_IPAUTH_GROUP / AUTH arm / second gate / Get_SHM_INFO.  
- `gIpAuthVars` sole writer = ioctl `0xC08C5506`; hash = `0xC0125507`.  
- Three ARM `UtopiaSetIPAUTH` implementations with those ioctls.  
- `MApi_AUTH_Process` → `UtopiaSetIPAUTH@plt` @ libdrvIPAUTH `0x2454`.  
- **No pulled ELF imports `MApi_AUTH_Process`; no fn ptr / MOVW / rel.dyn to it.**  
- midaemon: A32 `main` 6340 B; one `HASHKEY_Verify`; PLT layout 20/12; no AUTH PLT.  
- cpu_audio: `NEEDED libdrvIPAUTH`, zero AUTH PLT imports.  
- `vir_hashkey.txt` missing; `/proc/utopia` held by midaemon+cpu_audio.  
- R2 DSP gate distinct; AC3 bypasses AUTH.  
- Corrections in §7.

### LIKELY

- **`gIpAuthVars` never provisioned on this firmware** (no reachable caller) → IPCheck→0 → `AV+0x4D8=0` → AUTH arm FAIL → DTS blocked, AC3 OK.  
- midaemon `HASHKEY_Verify` only validates `hashkey.sign`; does not issue utopia AUTH ioctls.  
- cpu_audio’s libdrvIPAUTH dependency is not a direct provisioning caller.  
- `Customer_hash` BSS is intended hash slot fed only if `UtopiaSetIPAUTH` ran (it does not).  
- Downstream `[2185]` is consequence of host latch/decoder-type failure.  
- PIC string xrefs require PC-rel decode (not absolute VAs).

### BLOCKED (live / missing)

- Actual `gIpAuthVars`, `AV+0x4D8`, `AV+0x503`, SHM38, `AV+0x440/0x444`, `AV+0x4C8` at runtime.  
- mik GET ioctl command inventory / GET_CODEC_TYPE path (rel-xref `NameError` unfinished).  
- AUTH bit feature names; SET_IPAUTH_GROUP byte semantics vs DSP block.  
- Whether an **unpulled** binary (factory tool, vendor player) dlsyms `MApi_AUTH_Process`.  
- SHM38 producer on R2.  
- TCL stripped utpa2k jump-table full diff.

### DISPROVEN

- See correction table §7 (kill-byte, latch 0xB arm, V440 working-DTS donor, raw ioctl bytes, four writers of +0x4D8, etc.).

---

## 12. Blockers

1. **No read path** for kernel AUTH state (no kcore/devmem; kptr_restrict=2; forbidden writes; no `_IOR`).  
2. **No invocation path found** for `MApi_AUTH_Process` — need dlsym audit of unpulled binaries or accept LIKELY-not-run.  
3. SHM38 runtime + DSP producer unknown.  
4. Feature-bit names unknown.  
5. Host memory pressure — keep scans bounded; **do not** repeat runaway glob loops.  
6. No patch proposed at this evidence level (explicitly out of scope).

---

## 13. Next steps (read-only)

1. **Close H1 live gap if a safe read appears:** only a future read-only GET ioctl or authorized core dump — **not** attempted here.  
2. **midaemon dlsym/string scan** for `MApi_AUTH_Process` / `UtopiaSetIPAUTH` as *strings* used with `dlsym` (dynamic) — cheap static pass on midaemon + any newly pulled tvservice bins.  
3. **Pull remaining NEEDED of cpu_audio** (`libmi.so`, `libMsFS.so`) only if read-only justified — check whether *they* import `MApi_AUTH_Process`.  
4. **mik GET-ioctl:** fix `.rel.text` addend scan for `.rodata.str1.1+0x8D49D`; inventory `movt #0x80xx` in `DoIoctl`.  
5. **Name AUTH bits** 7/0xF/0x12/0x7D/0x0A from libutopia/libdrvIPAUTH rodata.  
6. Optional TCL strings diff on stripped utpa2k — low priority vs H1.  
7. **Do not patch** until: live `AV+0x4D8` or equivalent + SHM38 + one authorized write surface exist.

---

## 14. What must be known before the first patch attempt

1. **Live value of `AV+0x4D8` (and whether `AV+0x503` freeze occurred)** — distinguishes H1-null from H2-SHM.  
2. **Live SHM38** (and SHM4D if second gate relevant).  
3. **Whether any process ever succeeds `ioctl 0xC08C5506`** (strace-equivalent forbidden here → static LIKELY-not is the stand-in; residual doubt = unpulled dlsym).  
4. **Which AUTH bits gate DTS** by name/semantics (0xF / 0x12 / 7).  
5. **Independent confirmation DSP AEON gate state** — host-only patch may be insufficient if DSP also fails (parallel, not substitute).  
6. **A single authorized, reversible host write surface** (none proposed; utopia/mik writes remain forbidden under standing RO rules).  
7. **Regression plan for AC3** (AC3 must remain green if any AUTH/latch change is ever contemplated).

---

*End of Phase A report body. Nothing modified on the projector. No patches, no hex diffs proposed. Corpus and prior reports untouched.*

---
---

# PHASE B ADDENDUM — 2026-09-22 (static-only; no device contact)

**Scope (user directive):** NO runtime of any kind (no ADB, no playback/probe, no `/proc` reads, no writes). Corpus preserved; new extractions only into **new** directories. Goal: close or weaken H1 via dynamic-caller audit, missing NEEDED deps, `MApi_AUTH_Process` data-flow, midaemon hashkey flow, gIpAuthVars writers, CheckHashkey ordering, TCL lineage, final verdict table.

**Method:** single scripted Python scans only (no glob tool); targeted `os.walk`; Capstone A32 for `e_flags` bit0-clear PLT/code; Thumb BL decoder used where symbols had thumb bit.

---

## B1. Corpus expansion (read-only extract)

Source: `uni_out\tvservice_a.bin` (ext4 @1024, magic EF53). Extractor: Python `ext4` package → **new** dir only:

`C:\Users\k0994\AppData\Local\Temp\opencode\phaseb_extract\`

| File | Size | Note |
|------|------|------|
| `bin__midaemon` | **195072** | Full production midaemon (Phase A used 26092 B stub — **different binary**) |
| `glibc__libmi.so` | **3785112** | **Previously missing NEEDED** — now present |
| `glibc__libMsFS.so` | 20768 | Previously missing NEEDED |
| `glibc__libdrvIPAUTH.so` | 33048 | vs Phase A 26148 (same logic, more rodata) |
| `glibc__libhashkey_ca.so` | 18348 | vs Phase A 5452 (adds OP-TEE dlopen path) |
| `bin__cpu_audio` | 7371528 | Re-extract for NEEDED completeness |
| `glibc__liblinux.so`, `glibc__libMsOS.so`, `glibc__libapiAUDIO.so`, … | various | Full glibc set |

Squashfs partitions (`linux_rootfs`, `3rd`) not needed after ext4 hit. `libmi3.so` already in corpus under `ghidra_proj_new` / `spdif_audio_investigation\libs`.

---

## B2. Dynamic-caller audit (dlsym / dlopen / RTLD / fnptr / ctors / relocs)

**60+ ELFs** scanned (midaemon, cpu_audio, libmi*, libdrv*, libhashkey, libutopia*, libapiAUDIO, libMsOS, liblinux, libmi3*, audio HAL, hwcomposer, TCL libs, utpa2k/mik .ko, phaseb_extract set).

| Pattern | Result |
|---------|--------|
| String `dlsym`/`dlopen` as AUTH resolution | **Only** graphics/OMX/audio-parser HALs + `libhashkey_ca` (OP-TEE `libteec.so`) — **not** AUTH symbol lookup |
| `libhashkey_ca` (tvservice) `dlopen` | Loads **`libteec.so`** then `dlsym` `TZCTL_TEE_HASHKEY_VERIFY` / `…_INFO_CLEAR` — TEE hash verify, **not** `MApi_AUTH_Process` |
| `.init_array` ctors on AUTH libs | `libdrvIPAUTH` ctor `0xD4C`/`0xCC8` = frame_dummy guard (reconfirmed); **no** AUTH provisioning ctors |
| `.rel.dyn` / MOVW-MOVT fn-ptr to `MApi_AUTH_Process` | **None** across corpus |
| **`.rel.plt` UNDEF import of `MApi_AUTH_Process`** | **YES — two families** (Phase A miss): see B3 |

**dlsym residual doubt for H1 is closed for AUTH:** no binary materializes `MApi_AUTH_Process` or `UtopiaSetIPAUTH` via `dlsym`.

---

## B3. `MApi_AUTH_Process` — importers, data-flow, prototype

### B3.1 Importers (`.rel.plt` / UNDEF dynsym)

| Binary | GOT | PLT | BL sites found |
|--------|-----|-----|----------------|
| **`libmi.so` (tvservice)** | `0x110B5C` | `0x16754` | **1× A32 `BL @0xAB32C` inside `MI_SYS_Init` (`0xA9AC8`, sz 6768)** |
| **`libmi3.so`** (+ `libmi3_dts_v3` / `_patched` / `_patched_v2`) | `0xB2BD0` | `0xB0670` | **0 BL (A32+Thumb)** — import present, **no call site** (dead binding or unused path) |
| libdrvIPAUTH / libutopia* | *export* | — | see B3.2 |

### B3.2 `MApi_AUTH_Process` body (tvservice `libdrvIPAUTH` @ `0x1BB4`, sz 3244, A32)

Key compares: `r0==0x30` (**Decoder_Type 0x30**), `r3==0x37`, many `==0x60` (ASCII `'`'), group `==6`, `r3==1`, `ip==0x48`.

Resolved BLs (PLT map `0xA80–0xB94`):

| Site | Target | Role |
|------|--------|------|
| `0x1BE0` | `MDrv_AUTH_InitialVars` | seed auth vars |
| `0x1BFC` | `MDrv_SYS_Init` | chip init |
| `0x1CB8/0x1CC0/0x2378/0x257C` | `MDrv_SYS_GetChipType` / `GetChipID` | identity |
| `0x22EC`, `0x2300` | **`MDrv_AUTH_IPCheck`** ×2 | read feature bits (needs `gIpAuthVars` live) |
| **`0x2358`** | **`UtopiaSetIPAUTH`** | **provision path** (Phase A cited old VA `0x2454`→`0xb1c`; tvservice PLT base shifted) |
| `0x26DC` | `MDrv_SYS_Query` | query |

**Prototype (inferred):** `int MApi_AUTH_Process(void *cfg_or_path, void *buf)` — r0/r1 live across; builds/validates 0x8C-class structure; on success branch calls `UtopiaSetIPAUTH`.

### B3.3 Call chain into `MApi_AUTH_Process` (new — Phase B)

```
midaemon main
  └─ BL MI_SYS_Init @0x2184  (PLT 0x1A90)          [tvservice bin__midaemon]
        └─ libmi.so MI_SYS_Init @0xA9AC8
              ├─ [sp+0x30] flag ← 0 @0xA9C5C (init); later strb r5 @0xAA57C
              ├─ ldrb r0,[sp,#0x30]; cmp r0,#1; beq 0xAB530  ← GATE: skip AUTH if flag==1
              └─ BL MApi_AUTH_Process @0xAB32C
                    └─ … IPCheck … UtopiaSetIPAUTH @0x2358
                          └─ open("/proc/utopia") → ioctl 0xC08C5506 → gIpAuthVars
                                                     → ioctl 0xC0125507 → gCusID/gCusHash

cpu_audio main
  └─ BL MI_SYS_Init @0x2B46C (PLT 0x25140)  r0=0   → same libmi path (if linked libmi)
```

**Both `midaemon` and `cpu_audio` import `MI_SYS_Init` (UND + `.rel.plt`).** Phase A's "cpu_audio has zero AUTH PLT imports" remains true for direct AUTH symbols — the path is **indirect via `MI_SYS_Init` → libmi → MApi**.

### B3.4 Gate on AUTH inside `MI_SYS_Init`

- Early: `mov r2,#0; str r2,[sp,#0x30]` @ `0xA9C5C` then used as out-buf for `bl 0x173c0`.
- Later: `strb r5,[sp,#0x30]` @ `0xAA57C` (r5 = prior call result).
- At AUTH: `ldrb r0,[sp,#0x30]; cmp #1; beq 0xAB530` → **skips `MApi_AUTH_Process` when flag==1**.
- String xref at call (`ldr ip,[pc,#-0x558]; add ip,pc,ip`) did not resolve cleanly to a named rodata string (PC-rel offset landed in `.bss` in this decode) — non-blocking; r0=ip, r1=`[r4+0x4c]`.

**Implication:** provisioning is **conditionally reachable** from midaemon `main`, not from a ctor. Flag polarity unknown without runtime (minimal fact §B8).

---

## B4. midaemon hashkey control/data flow

### B4.1 tvservice full `bin__midaemon` (195072 B)

| Item | Value |
|------|-------|
| `main` | `0x1C88`, sz **7164** |
| `MI_SYS_Init` | BL **`@0x2184`** (early in main; **before** hashkey verify) |
| `HASHKEY_Verify` | PLT `0x1B08`, **1 BL @`0x3418`** in `main` |
| Call site | `mov r1,r7; mov r0,r5; bl 0x1b08` → `r6=r0`; `cmp r6,#0; bne 0x385c` |
| Strings | `vir_hashkey` `0x51A8`, `HASHKEY_Verify` fmt `0x50E8`/`0x228DB`/`0x2F127`, `Customer_hash` BSS **`0x16070` (16 B)** |
| NEEDED | includes `libmi.so`, `libMsFS.so`, `libdrvIPAUTH.so`, `libhashkey_ca.so`, `libMsOS.so`, `liblinux.so` |

**Order in `main`:** `MI_SYS_Init` (`0x2184`) **first**, then later `HASHKEY_Verify` (`0x3418`). Hashkey verify is **not** a precondition for `MI_SYS_Init`.

### B4.2 Phase A stub `midaemon` (26092 B)

Still valid for: `HASHKEY_Verify` only @ `0x3164`, no `MI_SYS_Init` import in that build. **Do not treat as production binary.**

### B4.3 `libhashkey_ca` (tvservice)

`HASHKEY_Verify @0x8F8` sz 280: builds OP-TEE invoke struct (`movw #0x6121` UUID/command), `dlopen("libteec.so")` → `dlsym(TZCTL_TEE_HASHKEY_VERIFY)` → on fail logs `"Call TZCTL_TEE_HASHKEY_VERIFY failed"`. **Independent of utopia AUTH ioctls.**

---

## B5. gIpAuthVars alternative writers (re-scan)

| Writer | Found? |
|--------|--------|
| Kernel `utopia_proc_ioctl` cmd6 `0xC08C5506` | **only** (utpa2k*.ko) |
| Userspace `UtopiaSetIPAUTH` | libMsOS, liblinux, libutopia_td98, lib_libutopia, libdrvIPAUTH(+dev) — same three families + TCL |
| Direct MOVW of `gIpAuthVars` VA in userspace | **none** (symbol is kernel bss) |
| New writers in phaseb_extract | **none** |

Raw pattern `06 55 8C C0` / string `0xC08C5506` only inside `UtopiaSetIPAUTH` bodies + ioctl construction.

**No alternative userspace writer.** Provisioning still exclusively via `UtopiaSetIPAUTH` ← `MApi_AUTH_Process` ← `MI_SYS_Init`.

---

## B6. CheckHashkey ordering vs SetDecodeSystem / Decoder_Type(0x30)

### B6.1 `MDrv_AUDIO_SetDecodeSystem` (`_glibc_libapiAUDIO` @ `0x31F74`, sz 356)

```
push …
load table ptr → [r5]
if !table: goto 0x3208C (init path)
if r6==0: goto 0x32004          // zero-arg path: check AV+0x4C8, byte>3 → log, return 0
BL CheckHashkey @0x31FA4         // *** CheckHashkey FIRST ***
BL 0x15D20                       // helper
load fn; blx r3                  // actual set-decode
if ret==1: copy config into AV table
clear byte [AV+0x2000+0x5D9]
```

**Ordering verified:** `MDrv_AUDIO_CheckHashkey` runs **before** the vendor set-decode `blx`, and **before** Decoder_Type-related payload is applied. Same pattern:

| Caller | Site | Pre-note |
|--------|------|----------|
| `MDrv_AUDIO_SetDecodeSystem` | `0x31FA4` | gate then blx |
| `MDrv_AUDIO_SetSystem` | `0x31F5C` | CheckHashkey then set |
| `MDrv_AUDIO_SetAudioParam2` | `0x31EB8` | when `r1∈{0x12,3}` |
| `MDrv_AUDIO_DspReboot` | `0x1760C` | |
| `HAL_MAD_SetCommInfo` | `0x724DC` | **`strb r2=0, [r3,#0x503]` immediately before CheckHashkey** (clears freeze/skip byte) |

### B6.2 Decoder_Type `0x30` inside `MApi_AUTH_Process`

`cmp r0,#0x30` at `0x1CAC` (tvservice) / Phase A `0x1DA8` (old VA) — confirms host auth path **explicitly branches on Decoder_Type 0x30**. Linkage: CheckHashkey (userspace/API) → kernel `MDrv_AUDIO_CheckHashkey` → `AV+0x4D8` ← `IPCheck` ← `gIpAuthVars`.

**Static order (DTS set path):**  
`SetDecodeSystem` → **CheckHashkey** → (vendor set-decode) → R2 AUTH arm reads `AV+0x4D8` / SHM38.

---

## B7. TCL / T615 lineage

| Symbol | TD98 `libutopia_td98` | TCL `lib_libutopia` |
|--------|----------------------|---------------------|
| `MApi_AUTH_Process` | `0x4C6768` sz 3160, **0 internal BL** | `0x5047A8` sz 3160, **0 internal BL** |
| `UtopiaSetIPAUTH` | `0x50BA28` | `0x8BACE8` |
| `MDrv_AUTH_IPCheck` | present | `0x505440` sz 136 |
| `MDrv_AUDIO_CheckHashkey` | present | `0x4AE598` sz 5652 |
| NEEDED | libcutils, log, c++, c, m, dl | same |

**Lineage confirmed:** same mega-lib architecture, same export set, same zero-internal-caller pattern for `MApi_AUTH_Process`. TCL is **not** a working-DTS donor; it only corroborates that MApi is an **external** entry (called from `libmi`-class `MI_SYS_Init`, not from utopia mega-lib).

---

## B8. H1 verdict table (Phase B)

| # | Claim | Status | Evidence |
|---|--------|--------|----------|
| 1 | `gIpAuthVars` sole kernel writer = ioctl6 `0xC08C5506` | **VERIFIED** | utpa2k re-scan; no other writers |
| 2 | Sole userspace bridge = `UtopiaSetIPAUTH` | **VERIFIED** | 3 providers + extraction set; no dlsym of it |
| 3 | Sole high-level caller of `UtopiaSetIPAUTH` = `MApi_AUTH_Process` | **VERIFIED** | BL `0x2358` (tvservice) / `0x2454` (Phase A VA); PLT map |
| 4 | **`MApi_AUTH_Process` has zero importers** (Phase A) | **DISPROVEN** | `libmi.so` UNDEF+PLT+**BL `0xAB32C`**; `libmi3*` UNDEF (0 BL) |
| 5 | **Reachable static chain:** midaemon → `MI_SYS_Init` → `MApi_AUTH_Process` → `UtopiaSetIPAUTH` → ioctl6 | **VERIFIED (static)** | midaemon BL `0x2184`; libmi BL `0xAB32C`; libdrv BL `0x2358`; cpu_audio BL `0x2B46C` |
| 6 | Chain **executes** on device (provisioning actually runs) | **BLOCKED** | Gate `ldrb [sp+0x30]==1` may skip; flag polarity + ioctl success need **one live read** (B9) |
| 7 | midaemon `HASHKEY_Verify` provisions `gIpAuthVars` | **DISPROVEN** | BL `0x3418` only → TEE hash; **before** order is MI_SYS_Init not hashkey |
| 8 | `libhashkey` `dlsym` resolves AUTH symbols | **DISPROVEN** | `libteec.so` / `TZCTL_TEE_*` only |
| 9 | cpu_audio directly imports AUTH symbols | **VERIFIED false / indirect only** | no AUTH PLT; imports `MI_SYS_Init` |
| 10 | CheckHashkey ordered **before** SetDecodeSystem apply | **VERIFIED** | `0x31FA4` before `blx r3` |
| 11 | `MApi_AUTH_Process` branches on Decoder_Type `0x30` | **VERIFIED** | `cmp r0,#0x30` @ `0x1CAC` |
| 12 | Alternative gIpAuthVars writers in extracted libs | **DISPROVEN** | B5 scan |
| 13 | TCL lineage shares MApi-external-call pattern | **VERIFIED** | B7 |
| 14 | Live `gIpAuthVars` / `AV+0x4D8` / SHM38 | **BLOCKED** | no RO read path under standing rules |

### H1 closing statement (revised)

**Phase A: “LIKELY never provisioned.”**  
**Phase B: static reachability of the provisioning path is now VERIFIED** end-to-end on the **production** tvservice midaemon + libmi + libdrvIPAUTH (previously missing pieces). H1 narrows from “is there a caller?” to **“does `MI_SYS_Init` take the AUTH branch (flag≠1) and does `ioctl 0xC08C5506` succeed at least once?”** — a **runtime gate**, not a static dead-end.

---

## B9. Single minimal future runtime read-only fact (if H1 still must be closed)

**Preferred one-liner:** live value of **`AV+0x4D8`** (audio vars latch byte) — distinguishes provisioned+checked (`1/2/3`) vs never (`0`).

**Equivalent alternatives (any one):**

1. Live pointer **`gIpAuthVars`** (NULL vs non-NULL) in `utpa2k` bss — direct H1 answer.  
2. Whether midaemon `main` past `0x2184` took `beq 0xAB530` (flag `[sp+0x30]`) — gate polarity.  
3. Single observation: any successful `ioctl(fd, 0xC08C5506, …)` on `/proc/utopia` (audit/strace — **forbidden under current RO**; only if rules change).

**Still required for DTS work (unchanged):** live SHM38 + AUTH bit names 7/0xF/0x12.

---

## B10. Methods × corpus (audit completeness)

| Method | Coverage |
|--------|----------|
| ELF32 parse, dynsym, `.rel.plt`, `.init_array` | all named ELFs in roots below |
| A32 BL/BLX + Thumb-2 BL/BLX scan to PLT/defined VAs | libmi, libmi3, midaemon, cpu_audio, libdrvIPAUTH, libapiAUDIO, libutopia*, libhashkey |
| String scan (`dlsym`, AUTH names, ioctl hex, paths) | full file bytes of each ELF |
| gIpAuthVars / `0xC08C5506` byte scan | phaseb_extract + temp opencode + patch_baseline + tclaudio |
| ext4 walk extract | `tvservice_a.bin` only (read-only source) |

**Roots:**  
`Temp\opencode\` (top + tvsvc_out2 + tclaudio + tvcfg_out + phaseb_extract),  
`firmware_temp\ghidra_proj_new`, `spdif_audio_investigation\libs`, `patch_baseline`, `display_edid_investigation\extracted\hw`.

**Scripts (TEMP, disposable):** `phaseb_recon.py`, `phaseb_fstype.py`, `phaseb_strscan.py`, `phaseb_extract_tvservice.py`, `phaseb_extract_libmi.py`, `phaseb_audit_all.py`, `phaseb_callers_full.py`, `phaseb_libmi3_mapicall.py`, `phaseb_deep.py`, `phaseb_chain.py`, `phaseb_verify_chain.py`, `phaseb_final_verify.py`.

---

## B11. Phase A statements amended

| Phase A | Phase B |
|---------|---------|
| “No binary imports `MApi_AUTH_Process`” | **False** — `libmi.so` (+ unused import in `libmi3*`) |
| “midaemon calls only HASHKEY_Verify; does not call UtopiaSetIPAUTH” | **Incomplete** — production midaemon also calls **`MI_SYS_Init` → MApi → UtopiaSetIPAUTH** (conditional gate) |
| “cpu_audio zero AUTH PLT; ctor not provisioning” | Still true for **direct** AUTH; **indirect** via `MI_SYS_Init` import |
| H1 “LIKELY never provisioned” | **Weakened → chain VERIFIED static; execution BLOCKED on one live flag/ioctl fact** |
| Missing `libmi.so` / `libMsFS.so` | **Extracted** into `phaseb_extract\` |

---

*End of Phase B addendum. No device contact. No patches. Prior report body (§1–14) and all prior corpus files preserved. New artifacts only under `Temp\opencode\phaseb_extract\` and TEMP scripts. Deliverable remains `C:\firmware_temp\aeon_validate\dec_work\MIMO_FORENSIC_AUDIT_20260922.md`.*
