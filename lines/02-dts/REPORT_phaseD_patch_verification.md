# REPORT: R5 Phase D — Experimental `mik.ko` Patch Verification Package

**Device:** Thundeal TD98 Pro / C50A (MStar MT5889)
**Phase:** D only. **NOTHING DEPLOYED.** Device untouched.
**Date:** 2026-09-05

## What this patch is

> **minimal experimental forcing of the compressed-output branch when the DTS capability bit is absent**

It is **not** a confirmed final fix. Its purpose is to **test the EDID/output-mode hypothesis**
established in Phase C. Whether it restores a physical DTS lock on the AVR is **unknown** and can
only be answered by the Phase E playback test, which has **not** been run.

---

# A. PATCH VERIFICATION REPORT

## 1. Source (base = currently deployed `mik_dts_hdmienable.ko`)

| field | value |
|---|---|
| filename | `mik_dts_hdmienable.ko` |
| path | `C:\firmware_temp\spdif_audio_investigation\libs\mik_dts_hdmienable.ko` |
| size | 8 084 248 bytes (0x7b5b18) |
| **MD5** | `647b08db19dcbc20b7d5c6a4a4e36e9b` |
| **SHA-256** | `fe1ff648d882ab0758cdd64fc2603fbff7e903a9499603e423b45661f9fbe07f` |

This is the module currently loaded on the device (verified by on-device `md5sum` in R5 Phase C/D).

## 2. Output (experimental)

| field | value |
|---|---|
| filename | `mik_r5_patched.ko` |
| path | `C:\firmware_temp\spdif_audio_investigation\runtime_phase5\mik_r5_patched.ko` |
| size | 8 084 248 bytes (0x7b5b18) — **identical to source** |
| **MD5** | `03fc2c0d22b231bc478e574557b85473` |
| **SHA-256** | `95a4876b4af4d5eab9621b8c404cc460c9e468c0931c236240e1a2fcefa35769` |

Build is deterministic — repeated rebuilds from the same base reproduce this md5 exactly.

## 3. Exact file offset and VA

| field | value |
|---|---|
| file offset | **`0x9b54c`** |
| virtual address | **`0x98a58`** |
| section | `.text` (addr 0x0, size 0x2ca2d8, file off 0x2af4 — unlinked `.ko`, so `VA == file_off - 0x2af4 + 0` mapping confirmed: `va2off(0x98a58) == 0x9b54c`) |
| enclosing function | `_MI_AOUT_SetHdmiAutoMode` @ `0x989cc` |
| enclosing basic block | `0x98a54` |

## 4. Old 4-byte instruction

```
offset 0x9b54c:  00 60 a0 e3
assembly:        mov  r6, #0
```

## 5. New 4-byte instruction

```
offset 0x9b54c:  01 60 a0 e3
assembly:        mov  r6, #1
```

Only the **immediate rotate/value field** changes: `0x00` → `0x01` (bit 0 of byte 0).
The opcode, condition, destination register and everything else are untouched.

## 6. Number of changed bytes

# **1**

Single byte at file offset `0x9b54c`: `0x00` → `0x01`.

## 7. Complete binary diff — proof that only one byte changed

Full 8 084 248-byte comparison of source vs output:

```
Total differing bytes : 1

offset       old    new    bit        byte-in-instr
--------------------------------------------------------------
0x0009b54c  0x00   0x01    bit 1      byte 0 of 0060a0e3 -> 0160a0e3
```

`cmp -l` equivalent (1-based offset, octal values):

```
  636237  0  1
```

Hex context (±16 bytes):

```
  -0x0010   SRC: 00 70 40 e3 00 10 97 e5 6b 00 00 1a 01 50 a0 e3
            OUT: 00 70 40 e3 00 10 97 e5 6b 00 00 1a 01 50 a0 e3
  +0x0000   SRC: 00 60 a0 e3 40 00 51 e3 6b 00 00 3a fe ff ff eb
            OUT: 01 60 a0 e3 40 00 51 e3 6b 00 00 3a fe ff ff eb   <== PATCH BYTE
  +0x0010   SRC: 0c 00 90 e5 00 20 00 e3 00 20 40 e3 99 3d 01 e3
            OUT: 0c 00 90 e5 00 20 00 e3 00 20 40 e3 99 3d 01 e3
```

Corroborating instruction-level check across the whole gate function
(`_MI_AOUT_SetHdmiAutoMode`, 365 instructions):

```
instructions whose ENCODING differs : 1
   0x00098a58 :  OLD  0060a0e3 mov  r6, #0
              NEW  0160a0e3 mov  r6, #1
```

## 8. Existing `mik_dts_hdmienable` MonitorTask patch at `0x9a334` — PRESERVED

| variant | bytes @0x9a334 | state |
|---|---|---|
| pristine `kmods/mik.ko` | `03 01 00 1a` | `bne` (original branch) |
| base = deployed | `00 00 a0 e1` | `nop` (mov r0,r0) — **patched** |
| **output (patched)** | `00 00 a0 e1` | `nop` (mov r0,r0) — **patched** |

```
output == base at 0x9a334      : True
output != pristine at 0x9a334  : True
=> MonitorTask patch PRESERVED : True
```

## 9. No existing Exp-B / active-stack changes modified

**Within `mik.ko`** — every pre-existing (pristine → deployed) delta is intact:

```
pristine -> deployed base : 4 differing byte(s)
   0x0009a334: 03 -> 00
   0x0009a335: 01 -> 00
   0x0009a336: 00 -> a0
   0x0009a337: 1a -> e1

deltas lost in output     : 0
=> all pre-existing deltas intact : True

pristine -> output total diffs     : 5
expected (pre-existing + 1 new)    : 5
sets identical                     : True
=> output = deployed base + exactly the 1 intended new byte.
```

So the output is a **strict superset** of everything already deployed: it adds the one new byte and
loses nothing.

**Exp-B (`utpa2k.ko`)** — untouched. Exp-B lives in a *different* module; this patch reads and writes
only `mik.ko`. The build script does not open `utpa2k.ko` at all.

```
on-device utpa2k.ko (R5 verified) md5 4c5e6fbb...   [UNCHANGED - not touched]
```

Nothing in the AUTH/licence path, EDID data, parser, decoder, or HDMI/ARC routing is modified.

## 10. ELF / module structural validity — PASS

Both source and output were checked:

```
  [OK ] magic \x7fELF                              7f454c46
  [OK ] EI_CLASS = ELFCLASS32                      1
  [OK ] EI_DATA  = ELFDATA2LSB                     1
  [OK ] EI_VERSION = 1                             1
  [OK ] e_type = ET_REL (1) = relocatable .ko      1
  [OK ] e_machine = EM_ARM (40)                    40
  [OK ] e_shoff sane (< filesize)                  0x7b52d0
  [OK ] e_shnum > 0                                53
  [OK ] e_shentsize = 40                           40
  [OK ] e_shstrndx < e_shnum                       36
  [OK ] all non-NOBITS sections in-bounds          []
  [OK ] file size unchanged                        8084248
  [OK ] .text present                              addr=0x0 size=0x2ca2d8 off=0x2af4
  [OK ] patch VA 0x98a58 inside .text              delta +0x98a58
  [OK ] patch file offset inside .text             delta +0x98a58
  [OK ] VA<->offset mapping consistent             va2off(0x98a58)=0x9b54c
  [OK ] .modinfo present (module metadata intact)  size=0x265
  [OK ] .modinfo contains "vermagic="
  [OK ] .modinfo blob byte-identical to source
  [OK ] .rel.text intact                           size=0x1c25e0
  [OK ] .rel.data intact                           size=0x548
  [OK ] .rel.rodata intact                         size=0x2500
  [OK ] .rel.gnu.linkonce.this_module intact       size=0x10

  OVERALL ELF/MODULE SANITY: PASS
```

Why this matters for loadability: the file size, section table, `.modinfo` (incl. `vermagic=`) and
**all relocation sections are byte-identical**. The patch alters one immediate field inside an
existing instruction — it does not shift a single byte, so no relocation offset, no symbol value and
no section header is invalidated. The kernel's `insmod`/`module_alloc` + `apply_relocations` path
sees a structurally identical module.

## 11. Re-disassembled OUTPUT — resulting control flow

Disassembled from the **output** binary (`mik_r5_patched.ko`):

```
--- r6 written (patched block, 0x98a54) ---
   0x00098a54: 0150a0e3 mov       r5, #1
   0x00098a58: 0160a0e3 mov       r6, #1          <<<< PATCHED  (r6 = 1)
   0x00098a5c: 400051e3 cmp       r1, #0x40
   0x00098a60: 6b00003a blo       #0x98c14

--- selector (0x98c5c) ---
   0x00098c5c: 030056e3 cmp       r6, #3
   0x00098c60: 5900000a beq       #0x98dcc        ; r6==3 -> HBR path (r4=3)
   0x00098c64: 010056e3 cmp       r6, #1
   0x00098c68: 0100001a bne       #0x98c74        ; r6!=1 -> PCM
   0x00098c6c: 0240a0e3 mov       r4, #2          ; r6==1 -> COMPRESSED
   0x00098c70: 560000ea b         #0x98dd0
   0x00098c74: 0000a0e3 mov       r0, #0
   0x00098c78: 0040a0e3 mov       r4, #0          ; r4 = 0 (PCM)
   0x00098c7c: 540000ea b         #0x98dd4

--- call ---
   0x00098dd8: 0400a0e1 mov       r0, r4
   0x00098ddc: feffffeb bl        MApi_AUDIO_SPDIF_SetMode
```

Resulting flow, exactly as intended:

```text
DTS capability absent
        ↓
r6 = 1                      (0x98a58, patched)
        ↓
selector: cmp r6,#3  -> not taken   (0x98c5c)
          cmp r6,#1  -> bne NOT taken (0x98c64/0x98c68)
        ↓
r4 = 2                      (0x98c6c)
        ↓
MApi_AUDIO_SPDIF_SetMode(2) (0x98ddc)
```

**Forward-path proof.** Enumerating every acyclic path from the patched block to the `SetMode` call:

```
acyclic paths found: 48
paths where patched r6=1 survives to the selector: 48 / 48
```

On **all 48** paths there is no further write to `r6`, so the patched value reaches the selector
intact and `r4 = 2` (compressed) is selected.

## 12. Adjacent control-flow paths (AC3 / E-AC3 / DTS-HD) — why they are unchanged

### 12.1 The decisive structural fact: this is a codec `switch`

The gate function begins with a jump table:

```
   0x000989cc: f0482de9 push      {r4, r5, r6, r7, fp, lr}
   0x000989d4: 041041e2 sub       r1, r1, #4              ; index = arg - 4
   0x000989d8: 130051e3 cmp       r1, #0x13               ; 20 cases
   0x000989dc: 6a00008a bhi       #0x98b8c                ; default
   0x000989e0: 00208fe2 add       r2, pc, #0
   0x000989e4: 01f192e7 ldr       pc, [r2, r1, lsl #2]    ; switch(arg)
```

Decoding the 20-entry table (`jump table identical in source and output: True`):

| arg | target | chain |
|---|---|---|
| 4 | `0x98a88` | AC3 / E-AC3 / MLP chain (tests `0x1404`) |
| 5 | `0x98b3c` | other |
| 6 | `0x98b8c` | default / other |
| 7 | `0x98a88` | AC3 / E-AC3 / MLP chain (tests `0x1404`) |
| 8 | `0x98ba8` | other |
| **9** | **`0x98a38`** | **DTS chain — contains `0x98a58`** |
| **10** | **`0x98a38`** | **DTS chain — contains `0x98a58`** |
| **11** | **`0x98a38`** | **DTS chain — contains `0x98a58`** |
| 12–22 | `0x98b8c` | default / other |
| **23** | **`0x98a38`** | **DTS chain — contains `0x98a58`** |

**Only codec cases 9, 10, 11 and 23 enter the chain that contains the patched instruction.**
Every other case is dispatched to a different chain and never executes `0x98a58`.

### 12.2 Per-codec analysis

| codec | where it is handled | passes through `0x98a58`? | affected? |
|---|---|---|---|
| **AC3** | cases 4 and 7 → `0x98a88`, which tests `#0x1404` (AC3\|E-AC3\|MLP) and sets its own `r5=2, r6=1` (`0x98aa0`/`0x98aa4`); further AC3 sites at `0x98d10`, `0x98d28`, `0x98d54` under `tst r0,#4` (`0x98cf0`) | **No** | **No** |
| **E-AC3** | own site `0x98b48`–`0x98b50` (`r5=0xa`, `r6=3`) | **No** | **No** |
| **DTS-HD** | tested **first** at `0x98a38` (`tst r0,#0x800`); if the bit is set it branches to `0x98ad4`, i.e. it **leaves the chain before** `0x98a58` (own sites `0x98b2c`, `0x98e98`: `r5=0xb`, `r6=3`) | **No** | **No** |
| **MLP / TrueHD** | own sites `0x98bcc` (`r6=1`) and `0x98ca8`–`0x98cb0` (`r5=0xc`, `r6=3`) | **No** | **No** |
| **DTS-core present** | `0x98a44 tst r0,#0x80` → `bne 0x98c04` (`r5=7`, `r6=1`) — already compressed | **No** (branches away) | **No** |
| **DTS-core ABSENT** | falls through to `0x98a54`/`0x98a58` | **Yes** | **Yes — this is the intended target** |

### 12.3 Reachability proof (automated)

Forward reachability from the patched block to every other `r6`-write site:

```
VA         codec path                             r6 value     reachable from patched blk?
----------------------------------------------------------------------------------------------
0x00098aa4  AC3|E-AC3|MLP present (tst 0x1404)     r6=1          NO  (disjoint)
0x00098b2c  DTS-HD (r5=0xb)                        r6=3          NO  (disjoint)
0x00098b50  E-AC3 (r5=0xa)                        r6=3          NO  (disjoint)
0x00098ba0  fallback (r5=1)                        r6=0          NO  (disjoint)
0x00098bcc  MLP/TrueHD (r5=2)                      r6=1          NO  (disjoint)
0x00098c08  DTS-core PRESENT (r5=7)                r6=1          NO  (disjoint)
0x00098c84  no-caps default (r5=1)                 r6=0          NO  (disjoint)
0x00098cb0  MLP (r5=0xc)                           r6=3          NO  (disjoint)
0x00098d10  AC3 (r5=1)                             r6=0          NO  (disjoint)
0x00098d28  AC3 (r5=1)                             r6=0          NO  (disjoint)
0x00098d54  AC3 (r5=2)                             r6=1          NO  (disjoint)
0x00098e98  DTS-HD (r5=0xb)                        r6=3          NO  (disjoint)
```

All 12 are **disjoint** — the patched block cannot reach any of them.

### 12.4 The airtight argument

1. The output differs from the source in **exactly one byte** (item 7).
2. Therefore **365 of 365 − 1 = 364** instructions in the gate function are bit-identical, and every
   instruction outside the function is bit-identical.
3. Therefore **every branch condition, every branch target and the entire CFG are identical** between
   source and output. The jump table is byte-identical too.
4. Therefore the **only** behavioural difference in the entire module is the value `r6` receives at
   `0x98a58`, and only on executions that reach that instruction.
5. Executions that reach `0x98a58` are exactly: codec cases 9/10/11/23 **and** DTS-HD bit clear
   **and** DTS-core bit clear.
6. AC3 (cases 4/7), E-AC3, DTS-HD and MLP never satisfy that conjunction, so they execute
   **bit-identical instruction sequences** and are **provably unaffected**.

### 12.5 Residual risk (honest)

- The one thing static analysis cannot prove is that the sink's actual runtime capability mask
  produces the assumed branch — that was Phase A, and it is **blocked** on this device (no kprobe /
  ftrace / kallsyms / devmem; debug CLIs silent or handle-gated).
- Whether `MApi_AUDIO_SPDIF_SetMode(2)` then results in a real DTS lock downstream (non-PCM framing,
  SDO packing, AVR decode) is **not established** — it is precisely what Phase E tests.
- This patch does **not** claim to restore DTS. It forces the compressed-output branch so the
  hypothesis can be tested.

---

# B. BINARY DIFF

```
--- mik_dts_hdmienable.ko    (source / currently deployed)
+++ mik_r5_patched.ko        (experimental output)
@@ file offset 0x9b54c (VA 0x98a58), .text @@
-  00 60 a0 e3      mov  r6, #0
+  01 60 a0 e3      mov  r6, #1
```

```
cmp -l mik_dts_hdmienable.ko mik_r5_patched.ko
  636237  0  1          # 1-based byte 636237 == file offset 0x9b54c

total differing bytes: 1
```

Both files: 8 084 248 bytes. All section headers, `.modinfo`, and all `.rel*` sections identical.

---

# C. DEPLOYMENT COMMANDS (for later use — **NOT EXECUTED**)

> These are the exact commands to be run **only after explicit authorization**. None of them has
> been run. They are recorded here so the deployment step is reproducible and reviewable.

```bash
# ---------------------------------------------------------------------------
# 0. PRE-FLIGHT — confirm the device is still on the expected baseline
# ---------------------------------------------------------------------------
adb root                                   # MUST re-root; adbd can drop back to shell (uid 2000)
adb shell md5sum /vendor/lib/modules/mik.ko
    # expect: 647b08db19dcbc20b7d5c6a4a4e36e9b   <-- abort if different
adb shell md5sum /vendor/lib/modules/utpa2k.ko
    # expect: 4c5e6fbb...                        <-- Exp-B must be unchanged

# ---------------------------------------------------------------------------
# 1. BACK UP the currently deployed module (revert path)
# ---------------------------------------------------------------------------
mkdir -p backup
adb pull /vendor/lib/modules/mik.ko backup/mik.ko.baseline_647b08db
adb shell md5sum /vendor/lib/modules/mik.ko

# ---------------------------------------------------------------------------
# 2. PUSH the experimental module to a scratch location (does NOT activate it)
# ---------------------------------------------------------------------------
adb push runtime_phase5/mik_r5_patched.ko /data/local/tmp/mik_r5_patched.ko
adb shell md5sum /data/local/tmp/mik_r5_patched.ko
    # expect: 03fc2c0d22b231bc478e574557b85473    <-- abort if different

# ---------------------------------------------------------------------------
# 3. ACTIVATE — replace the module in place
# ---------------------------------------------------------------------------
adb shell "mount -o rw,remount /vendor"
adb shell "cp /data/local/tmp/mik_r5_patched.ko /vendor/lib/modules/mik.ko"
adb shell "chmod 644 /vendor/lib/modules/mik.ko"
adb shell "sync"
adb shell md5sum /vendor/lib/modules/mik.ko
    # expect: 03fc2c0d22b231bc478e574557b85473

# ---------------------------------------------------------------------------
# 4. REBOOT so the new module is loaded
# ---------------------------------------------------------------------------
adb reboot
# wait for boot, then re-root
adb root
adb shell md5sum /vendor/lib/modules/mik.ko      # confirm 03fc2c0d... is in place
adb shell lsmod | grep mik                        # confirm mik loaded

# ---------------------------------------------------------------------------
# 5. PHASE E REGRESSION TEST (requires separate authorization)
#    AC3 control FIRST, then DTS. No dumpsys inside playback windows.
# ---------------------------------------------------------------------------
# 5a. AC3 CONTROL — must still lock (regression check)
adb logcat -c
adb shell am start -a android.intent.action.VIEW \
    -d 'file:///sdcard/Movies/test_ac3_51.mp4' -t 'video/mp4' \
    -n net.kodinerds.maven.kodi22/.Splash
sleep 14
adb shell dmesg      > runtime_phase5/R5e_ac3.dmesg
adb logcat -d        > runtime_phase5/R5e_ac3.logcat
#    -> confirm AVR still reports Dolby Digital (AC3); if not, REVERT immediately
adb shell am force-stop net.kodinerds.maven.kodi22

# 5b. DTS MAIN TEST
adb logcat -c
adb shell am start -a android.intent.action.VIEW \
    -d 'file:///sdcard/Movies/test_dts_51.mp4' -t 'video/mp4' \
    -n net.kodinerds.maven.kodi22/.Splash
sleep 14
adb shell dmesg      > runtime_phase5/R5e_dts.dmesg
adb logcat -d        > runtime_phase5/R5e_dts.logcat
#    -> confirm whether the AVR identifies DTS (not PCM)
adb shell am force-stop net.kodinerds.maven.kodi22

# ---------------------------------------------------------------------------
# 6. REVERT (if AC3 regresses, or DTS still fails)
# ---------------------------------------------------------------------------
adb root
adb shell "mount -o rw,remount /vendor"
adb shell "cp /sdcard/backup/mik.ko.baseline /vendor/lib/modules/mik.ko"   # or re-push backup
adb shell "sync"
adb reboot
```

**Note:** `adb` is not on the PATH of the analysis shell used to produce this report; the commands
above are published for the deployment session, not executed here.

---

# D. DEPLOYMENT STATUS

## **DEPLOYMENT WAS NOT PERFORMED.**

Explicitly confirmed:

- ❌ No file was pushed to the device.
- ❌ `mik.ko` on the device was **not** modified — still `647b08db19dcbc20b7d5c6a4a4e36e9b`.
- ❌ `utpa2k.ko` on the device was **not** modified — still `4c5e6fbb…` (Exp-B).
- ❌ No `insmod` / `rmmod` / module reload.
- ❌ No reboot.
- ❌ No settings or property changes.
- ❌ No EDID changes.
- ❌ No playback test (Phase E not run).
- ❌ No `adb` command was issued at all during the production of this package.

The experimental module exists **only** at:

```
C:\firmware_temp\spdif_audio_investigation\runtime_phase5\mik_r5_patched.ko
md5 03fc2c0d22b231bc478e574557b85473
```

The device remains exactly as it was before this phase.

---

## Summary verdict

| check | result |
|---|---|
| 1. Source MD5/SHA-256 | `647b08db…` / `fe1ff648…` |
| 2. Output MD5/SHA-256 | `03fc2c0d…` / `95a4876b…` |
| 3. Offset / VA | `0x9b54c` / `0x98a58` |
| 4. Old instruction | `00 60 a0 e3` = `mov r6,#0` |
| 5. New instruction | `01 60 a0 e3` = `mov r6,#1` |
| 6. Changed bytes | **1** |
| 7. Binary diff | only `0x9b54c: 00 → 01` |
| 8. MonitorTask `0x9a334` patch | **preserved** |
| 9. Exp-B / prior changes | **all preserved; `utpa2k.ko` untouched** |
| 10. ELF / module sanity | **PASS** (size, sections, `.modinfo`, all `.rel*` identical) |
| 11. Output control flow | `r6=1 → r4=2 → SetMode(2)` on **48/48** paths |
| 12. AC3 / E-AC3 / DTS-HD | **provably unaffected** (disjoint switch cases + identical CFG) |

**Classification:** *minimal experimental forcing of the compressed-output branch when the DTS
capability bit is absent.* Ready for the authorized Phase E test. **Not a confirmed fix.**
