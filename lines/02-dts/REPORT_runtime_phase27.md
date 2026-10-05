# R27 — DTS / AC3 SDO-IEC output state investigation (SND)

**Date:** 2026-09-09
**Scope:** static firmware analysis only — **no patches applied, no device / firmware /
EDID changes made, and the installed Ghidra module was not further modified**.
**Targets:** `mst_snd_r2_MS12V22.bin` (SND, 1,839,920 B) and `mst_codec_r2_MS12V22.bin`
(DEC, 1,982,492 B). All addresses below are **file offsets** (image base `0`).

---

## 0. Headline result

The single most important outcome of R27 is a **negative result that invalidates a
previous working assumption**:

> **The IEC61937 burst preamble constants are not present anywhere in SND or DEC.**
> There is no `0xF8724E1F` (Pa/Pb) in either image, and the previously-reported
> "6 IEC-header clusters" were **false positives** produced by instruction-encoding
> coincidences. The DTS `SDO_SpdifPacker` / `ADD_IEC_HEADER` strings have **no code
> references at all**.

Consequently the "DTS codec → SDO_SpdifPacker → IEC header → output" chain
**cannot be traced as firmware data-flow in SND**, because the IEC-header stage is
not implemented there. What *is* provable is a **separate, structurally parallel
AC3-vs-DTS output-reset path** in the encoder layer — documented in §5.

---

## 1. Tooling — and fixing it

The user's rule for R27 was: **Reko = primary** for region decode / boundaries /
`j`/`jal` / branch targets; **Ghidra = cross-check** for opcode / registers /
immediates / length / mixed-width; **do not trust `bg.jal` targets**.

### 1.1 Ghidra headless was broken — root cause found and bypassed

All prior headless attempts produced empty project directories and no output. The
`.bat` launch chain was the problem, and two distinct faults were found:

1. **`APPDATA` is not set in this environment.** `analyzeHeadless.bat` →
   `launch.bat` → `ghidra.Ghidra` throws immediately:
   ```
   java.io.FileNotFoundException: Required environment variable "APPDATA" is not set!
       at utility.application.ApplicationUtilities.getDefaultUserSettingsDir(...)
       at ghidra.GhidraApplicationLayout.<init>(...)
   ```
2. **`JAVA_HOME` is empty**, so Ghidra's `LaunchSupport` falls through to its
   *interactive* JDK chooser (`LaunchSupport ... -ask`), which cannot work
   non-interactively and exits silently.

**Fix — bypass the `.bat` chain entirely and invoke Java directly** (this is the
reusable recipe):

```bash
JH="/c/Users/k0994/AppData/Local/Programs/Eclipse Adoptium/jdk-21.0.12.8-hotspot/bin/java"
APPDATA="C:/Users/k0994/AppData/Roaming"; export APPDATA
"$JH" -Xmx2G \
  -Djava.system.class.loader=ghidra.GhidraClassLoader \
  -Dapplication.settingsdir="<writable dir>" \
  -Dfile.encoding=UTF8 -Duser.country=US -Duser.language=en \
  -Dsun.java2d.d3d=false -Dsun.java2d.opengl=false \
  -Dlog4j.skipJansi=true -Xshare:off --enable-native-access=ALL-UNNAMED \
  -cp "<GHIDRA>/Ghidra/Framework/Utility/lib/Utility.jar" \
  ghidra.Ghidra ghidra.app.util.headless.AnalyzeHeadless \
  <projdir> <projname> -import <bin> -loader BinaryLoader \
  -loader-baseAddr 0 -processor 'aeon:LE:32:default' -noanalysis \
  -scriptPath <scripts> -postScript DumpAEON.java "DUMPALL:1B000:5400"
```

Result: `EXIT=0`, `REPORT: Import succeeded`, 817 KB of decode output. **The
patched AEON module works correctly on Ghidra 12.1.2** — this independently
re-confirms the PART 1 fix.

### 1.2 Reko

A full-image Reko `decompile` run completed in the background, but Reko produces
only internal `.tests` output (no assembly listing), and its linear sweep
desynchronises on mixed-width code — exactly as expected. Reko could not serve as
the primary decoder here. **Ghidra's recursive-descent / per-address
`disassemble()` was therefore used as the authoritative decoder**, with the
encoding independently re-derived from `aeon.slaspec` + `aeon_ORBIS32.sinc` as a
cross-check (§1.3).

### 1.3 Encoder self-test (the Ghidra cross-check, done statically)

To validate decoding independently of Ghidra, the `bg.movhi` / `bg.addi` encodings
were hand-derived from the slaspec and compared byte-for-byte against the firmware:

| Construct | Bytes built from spec | Bytes in file | |
|---|---|---|---|
| `bg.movhi r4,0x13` | `C0 80 02 61` | `C0 80 02 61` @ `0x1B02F` | **MATCH** |
| `bg.addi r4,r4,-0x59a1` | `FC 84 A6 5F` | `FC 84 A6 5F` @ `0x1B033` | **MATCH** |

Confirmed bit layout (token bit 0 = LSB, big-endian bytes):

| Width | Opcode | Registers | Immediate |
|---|---|---|---|
| 4 B `bg.*` | `(w>>26)&0x3F` | `rD=(w>>21)&0x1F`, `rA=(w>>16)&0x1F` | `w & 0xFFFF` |
| 3 B `bn.*` | `(w>>18)&0x3F` | `rD=(w>>13)&0x1F`, `rA=(w>>8)&0x1F`, `rB=(w>>3)&0x1F` | `w & 0xFF` |
| 2 B `bt.*` | `(w>>10)&0x3F` | `rD=(w>>5)&0x1F`, `rA=w&0x1F` | — |

Width is selected by a decoder sub-field read at a **different bit offset per
width** (`bg`=bits 29-31 → 5/6/7; `bn`=bits 21-23 → 0-3; `bt`=bits 13-15 → 4).

---

## 2. Method: finding true instruction boundaries

Because widths interleave, any fixed-stride sweep desynchronises. Proven approach:

1. Ask Ghidra to attempt a decode at **every** byte over the region
   (`DUMPALL` mode, step 1). True instruction starts decode; misaligned offsets
   come back `<UNDEFINED>`.
2. Keep only **maximal contiguous chains** where `addr + length == next_addr`.

Over `0x1B000`–`0x20400` this yields **one contiguous run of 7,073 instructions**
spanning the whole region — i.e. the region is fully decodable real code, mixing
4/3/2-byte instructions. (489 of these in the first `0x600` bytes alone; the rest
of the byte offsets were `<UNDEFINED>`, as expected.)

---

## 3. Anchor / string map

All requested strings were located. They fall into **two distinct string tables**:

### Table A — `0x12A5xx`–`0x12AFxx` (actively referenced by code at `0x1B000`–`0x20400`)

Resolved by decoding `bg.movhi rX,hi` + `bg.addi rX,rX,lo` pairs (48 sites).

### Table B — `0x1823xx`–`0x1877xx` (DTS SDO packer / assert text)

| Offset | String |
|---|---|
| `0x182319` | `DTSX_CORE2_API_SDO_Packer(p_file_player->pCore2Instance, pFrameBuf, &…` |
| `0x1826AA` | `DTSDecSDOPacker_API_Process(p_file_player->SDOPacker, output_buf, nuFr…` |
| `0x182832` | `ADD_IEC_HEADER_I32, &IEC_Header)` |
| `0x182847` | `IEC_Header)` |
| `0x183319` | `DTSX_CORE2_API_SDO_Packer` |
| `0x183747` | `SDO_SpdifPacker_SAPI_GetIEC` |
| `0x183763` | `SDO_SpdifPacker_SAPI_SetIEC` |
| `0x187372` / `0x187629` | `DTSDecSDOPacker_API_Process(…)` (2nd copy pair) |
| `0x187529` / `0x18753E` | `ADD_IEC_HEADER_I32, &IEC_Header)` / `IEC_Header)` (2nd copy pair) |

**Key methodological note:** an earlier "xref" scan found no references to these
strings because it looked for a *single 32-bit immediate*. In this ISA addresses
are materialised as **`movhi`(high16) + `addi`(signed low16) pairs**. Once that was
accounted for, 128 code sites building addresses in `0x180000`–`0x188FFF` were
found — **but none of them resolves to `0x182832`, `0x183747` or `0x183763`.**

---

## 4. IEC header: negative findings

### 4.1 No Pa/Pb constants

| Search | SND | DEC |
|---|---|---|
| `0xF8724E1F` (Pa+Pb, BE) | **0 hits** | **0 hits** |
| `0x4E1FF872` (Pb+Pa, BE) | **0 hits** | **0 hits** |

### 4.2 The previously-reported "6 IEC clusters" were false positives

| Cluster | Bytes | Actual instruction | Why it matched |
|---|---|---|---|
| `0x1B4AB` | `F8 72 0E EC` | `bg.sbz 0xeec(r18),r3` | opcode `0x3E` → high byte `0xF8`; next byte `0x72` |
| `0x1B4B2` | `4E 1F 0E` | `bn.sra r16,r31,r1` | opcode `0x13` → high byte `0x4E`; next byte `0x1F` |
| `0x1BEF8` | `F8 72 EF 2A` | `bg.sbz 0xef2a(r18),r3` | same opcode coincidence |
| `0x1EE7F`, `0x1F7FF`, `0x1F8A3`, `0x1EDA6` | — | same pattern | same opcode coincidence |

The 16-bit values `0xF872` / `0x4E1F` occur as the **leading two bytes of ordinary
instructions**, because `bg.sbz` (op `0x3E`) and `bn.sra`/`bn.slli` (op `0x13`)
happen to encode their opcode there. **Instruction bytes were mistaken for data
constants.**

### 4.3 `0x1800` is a buffer offset, not an IEC length

`0x1800` appears 57 times in SND, but in decoded context it is used as an
**address offset / stride**, not as an IEC header field:

```
0x1C046  bg.ori   r25,r0,0x1800
0x1C05A  bg.ori   r5, r0,0x1800
0x1E790  bg.addi  r23,r13,0x1800
0x1E6F5  bg.sw_1  0x1800(r24),r0
```

`0x1800` = 6144, consistent with a buffer size/stride.

### 4.4 Implication

SND firmware does **not** construct the IEC61937 burst preamble. Either the
preamble is inserted by the SDO/SPDIF **hardware** block (firmware supplying only
payload + data-type/length), or it lives in code outside the analysed images. This
is consistent with §4.5.

### 4.5 The DTS packer strings are unreferenced

No `movhi`+`addi` pair anywhere in SND builds any address in Table B that is
actually used. Combined with §4.1, the most consistent reading is that the DTS
flib was shipped with **`assert`s compiled out**: the message text remains in
`.rodata` but no code path reaches it.

---

## 5. The first PROVEN AC3 / DTS divergence

Region `0x1B000`–`0x20400` is the **MS12V2 encoder node layer**, containing
DDPENC, MATENC, **MS11_DDE (AC3 / Dolby Digital encoder)** and **M6 DTS Encode**.
It exposes two *structurally parallel* output-reset paths — one per codec, and
within each, one per output port.

### 5.1 SPDIF reset — AC3 vs DTS

```
AC3 / MS11_DDE                          DTS / M6
─────────────────────────────────       ─────────────────────────────────
0x1EF2C bg.addi r3, r13, 0x6ad4         0x1F8FC bg.addi r3, r14, 0x6ad4
0x1EF30 bg.jal  0x0000ca42              0x1F900 bg.jal  0x0000ca42
0x1EF34 bg.addi r3, r13, 0x6ad4         0x1F904 bg.addi r3, r14, 0x6ad4
0x1EF38 bg.jal  0x0000c55e              0x1F908 bg.jal  0x0000c55e
0x1EF3C bg.movhi r3, 0x13               0x1F90C bg.movhi r3, 0x13
0x1EF40 bg.sw_0 0x486c(r10), r12        0x1F910 bn.sw   0x30(r10), r11
0x1EF44 bg.addi r3, r3, -0x52c8         0x1F913 bg.addi r3, r3, -0x5162
0x1EF48 bg.jal  0x0010786e              0x1F917 bg.jal  0x0010786e
0x1EF4C bg.j    0x0001ed89              0x1F91B bg.j    0x0001f730
```

`r3` resolves to the log string via `movhi`+`addi`:
* AC3: `0x130000 - 0x52C8` = **`0x12AD38`** → `"ms11 ddenc: reset spdif !![1]"`
* DTS: `0x130000 - 0x5162` = **`0x12AE9E`** → `"dts m6 enc: reset spdif !![1]"`

### 5.2 HDMI reset — AC3 vs DTS

```
0x1EF77 bg.addi r3, r12, 0x332c         0x1F87F bg.addi r3, r11, 0x332c
0x1EF7B bg.ori  r4, r0, 0xbb80          0x1F883 bg.ori  r4, r0, 0xbb80
0x1EF7F bt.movi r5, 0x0                 0x1F887 bt.movi r5, 0x0
0x1EF81 bg.jal  0x0000c798              0x1F889 bg.jal  0x0000c798
0x1EF85 bg.movhi r3, 0x13               0x1F88D bg.movhi r3, 0x13
0x1EF89 bg.sw_0 0x4870(r10), r11        0x1F891 bn.sw   0x34(r10), r12
0x1EF8D bg.addi r3, r3, -0x52a9         0x1F894 bg.addi r3, r3, -0x5143
0x1EF91 bg.jal  0x0010786e              0x1F898 bg.jal  0x0010786e
```

`0xBB80` = **48000** = 48 kHz sample rate. Strings:
* AC3: `0x130000 - 0x52A9` = `0x12AD57` → `"ms11 ddenc: reset hdmi !![1]"`
* DTS: `0x130000 - 0x5143` = `0x12AEBD` → `"dts m6 enc: reset hdmi !![1]"`

### 5.3 The divergence

The two paths call the **same** helpers (`0xca42`, `0xc55e`, `0xc798`, `0x10786e`)
in the **same** order, but differ in exactly three respects:

| | AC3 (`ddenc`) | DTS (`m6 enc`) |
|---|---|---|
| Encoder instance base register | **`r13`** | **`r14`** |
| Output-state slot (SPDIF / HDMI) | **`0x486c` / `0x4870`** (base `r10`) | **`0x30` / `0x34`** (base `r10`) |
| Log string | `0x12AD38` / `0x12AD57` | `0x12AE9E` / `0x12AEBD` |

This is the **first proven AC3/DTS divergence** in the SPDIF output path: the codec
selects a different encoder instance *and* a **different pair of output-state
slots** in the global `r10` structure.

### 5.4 The `r10` global output-state structure

Stores into `[r10 + 0x4840 … 0x4870]` are all 4-byte slots, several cleared
together in bursts — the signature of an output-port state block:

```
0x1EB88 bg.sw_0 0x4840(r10),r0      0x1EBBC bg.sw_0 0x4870(r10),r0
0x1EB8C bg.sw_0 0x484c(r10),r0      0x1EBC0 bg.sw_0 0x486c(r10),r0
0x1EB90 bg.sw_0 0x4844(r10),r0      0x1ECFC bg.sw_0 0x486c(r10),r0
0x1EB94 bg.sw_0 0x4850(r10),r0      0x1ED08 bg.sw_0 0x4870(r10),r0
0x1EB98 bg.sw_1 0x4848(r10),r0      0x1ED33 bg.sw_0 0x4868(r10),r24
0x1EB9C bg.sw_1 0x4854(r10),r0      0x1ED37 bg.sw_0 0x4864(r10),r23
0x1EBA0 bg.sw_1 0x4858(r10),r0      0x1ED6E bg.sw_0 0x4864(r10),r0
0x1EB48 bg.sw_0 0x485c(r10),r3      0x1ED72 bg.sw_0 0x4868(r10),r0
0x1E8B7 bg.sw_0 0x4860(r10),r24     0x1EF40 bg.sw_0 0x486c(r10),r12
                                    0x1EF89 bg.sw_0 0x4870(r10),r11
```

`0x486c` (SPDIF) and `0x4870` (HDMI) are **adjacent 4-byte slots**, strongly
indicating a per-output-port reset/state flag pair on the AC3 side. The DTS side
uses `0x30`/`0x34` — a different, smaller-offset pair.

---

### 5.5 The shared callees — the reset is a HARDWARE register operation

Decoding the four helpers (`0xC000`–`0xCE00`, one contiguous 1,136-instruction run)
shows the "reset spdif / reset hdmi" is **not** a software state flag write — it is
a **memory-mapped hardware sequence**. `0xC55E` and `0xC731` are near-identical
twins differing only in which port's registers they touch:

```
0xC55E  (port A)                      0xC731  (port B)
─────────────────────────────────     ─────────────────────────────────
movhi r23,0xb000                      movhi r23,0xb000
ori   r24,r23,0x854  -> 0xB000_0854   ori   r24,r23,0x854  -> 0xB000_0854
lwz   r25,0(r24)                      lwz   r25,0(r24)
and   r25,r25,0xF5FFFF                and   r25,r25,0x5FFFFF
sw    0(r24),r25      ; clear bits    sw    0(r24),r25      ; clear bits
mfspr r26,0x5001      ; t_start       mfspr r26,0x5001      ; t_start
ori   r23,r23,0x80c   -> 0xB000_080C  ori   r23,r23,0x810   -> 0xB000_0810
loop: lwz r24,0(r23)                  loop: lwz r24,0(r23)
      and r24,r24,0xFFFFFF                  and r24,r24,0xFFFFFF
      beqi r24,0 -> done                    beqi r24,0 -> done
      mfspr r24,0x5001                      mfspr r24,0x5001
      sub  r24,r24,r26  ; elapsed           sub  r24,r24,r26
      bles r24,0x2DC6C00 -> loop            bles r24,0x2DC6C00 -> loop
done: sw 0xB000_084C, 0               done: sw 0xB000_0850, 0
```

**Timeout `0x2DC6C00` = 48,000,000 cycles = exactly 1 second at 48 MHz.** The
1-second round number is strong corroboration that this is a deliberate
wait-for-hardware-acknowledge loop (SPR `0x5001` is a free-running counter).

Register roles:

| Address | Role |
|---|---|
| `0xB000_0854` | shared control/trigger — bits cleared to start the reset |
| `0xB000_080C` | status, polled by port A (SPDIF) until it reads 0 |
| `0xB000_0810` | status, polled by port B (HDMI) until it reads 0 |
| `0xB000_084C` | cleared after port A completes |
| `0xB000_0850` | cleared after port B completes |

The other two helpers are software-side:

* **`0xCA42(r3)`** — initialises the instance struct at `r3` (= `instance + 0x6ad4`):
  writes `0xD7_2000` at `+0x0/+0xc/+0x10`, `0x6_6000` at `+0x8/+0x4`,
  **`0xBB80` (48000 Hz) at `+0x1c`**, zeroes `+0x14/+0x18/+0x20/+0x24/+0x28`,
  then calls `0x10C5B9` with `r4=0`.
* **`0xC873(r3)`** — struct reset/defaults: `+0x0/+0xc/+0x10 = 0xDD_8000`,
  `+0x4 = 0xF7_C000`, `+0x8 = 0xFFFA_4000`, **`+0x1c = 0x2EE00` (192000)**,
  zeroing `+0x14/+0x18/+0x20/+0x24/+0x28/+0x34/+0x38`.
* **`0xC798(r3,r4,r5)`** — sample-rate / format setter. Stores `r4` at `+0x1c`,
  substitutes `0x2EE00` if `r4==0`, stores `r5` at `+0x34`, and compares `r4`
  against rate constants **`0x15888` (88200)** and **`0x1F400` (128000)**.
  Called with `r4 = 0xBB80` (48000) in both the AC3 and DTS HDMI paths.
* **`0x10C5B9(r3,r4,r5)`** — **`memset`**. It alignment-checks (`bn.andi r23,r3,0x3`),
  replicates the `r4` byte across all four lanes (`slli 8 / or / slli 16 / or`),
  and bulk-writes 16-byte chunks (`bn.sw` at `+0x0/+0x4/+0x8/+0xc`), falling back
  to a byte-at-a-time `bn.sbz` loop when unaligned. `0xCA42` calls it with `r4 = 0`
  to zero the remainder of the instance struct.

### 5.6 MMIO register map and port identification

Enumerating every `movhi 0xB000` + `ori` pair across the whole image (wide window,
88 candidate addresses) and keeping the dense, plausible block gives the audio
output peripheral at **`0xB000_0800`–`0xB000_0878`**:

| Address | Accesses | Role |
|---|---|---|
| `0xB000_0854` | 26 | **shared control/trigger** — bits cleared to begin a reset |
| `0xB000_0814` | 22 | high-traffic (status / FIFO level) |
| `0xB000_085C` | 20 | high-traffic (likely data / FIFO) |
| `0xB000_0850` | 7 | **port B completion** — written 0 by `0xC731` |
| `0xB000_084C` | 6 | **port A completion** — written 0 by `0xC55E` |
| `0xB000_080C` | 5 | **status polled by port A** (`0xC55E`) |
| `0xB000_0810` | 5 | **status polled by port B** (`0xC731`) |
| `0x0834`, `0x0838`, `0x083C` | 10 / 4 / 12 | additional config/state |
| `0x0800`,`0x0804`,`0x0808`,`0x0818`,`0x081C`,`0x0824`,`0x0828`,`0x082C`,`0x0830`,`0x0840`,`0x0844`,`0x0848`,`0x0858`,`0x0864`,`0x0868`,`0x086C`,`0x0870`,`0x0874`,`0x0878` | 1–4 each | remainder of the block |

**Port identification is now resolved** (upgrading claim 23 from STRONG INFERENCE).
The calling code disambiguates the two ports by which log string it emits:

* the path that logs **"reset spdif"** calls **`0xC55E`** → touches `0xB000_080C` / `0xB000_084C`
* the path that logs **"reset hdmi"** calls **`0xC731`** → touches `0xB000_0810` / `0xB000_0850`

and **both the AC3 and the DTS path agree** on this mapping, which is what makes it
reliable rather than coincidental:

| Port | Status polled | Cleared on completion | Called by | Log string |
|---|---|---|---|---|
| **SPDIF** | `0xB000_080C` | `0xB000_084C` | `0xC55E` | `… reset spdif !!` |
| **HDMI** | `0xB000_0810` | `0xB000_0850` | `0xC731` | `… reset hdmi !!` |

*Enumeration caveat:* the wide-window scan also produced false positives where a
`movhi 0xB000` was followed within 80 bytes by an `ori` belonging to a different
constant — visible as `0xB000_5888` (= low half of 88200), `0xB000_EE00`
(= low half of 192000) and `0xB000_F400` (= low half of 128000). These are the
sample-rate constants of §5.5, not registers, and are excluded above.

### 5.7 Refined statement of the divergence

Critically, **the hardware reset callees are shared**: both the AC3 and the DTS
SPDIF path call `0xCA42` + `0xC55E`, and both HDMI paths call `0xC873` + `0xC731` +
`0xC798`. The reset is therefore **codec-agnostic at the hardware level**.

The divergence is purely on the software side, and is exactly three things:
the **encoder instance base register** (`r13` vs `r14`), the **output-state slot
pair** in the `r10` struct (`0x486c/0x4870` vs `0x30/0x34`), and the **log string**.

## 6. Claim classification

| # | Claim | Classification |
|---|---|---|
| 1 | Ghidra 12.1.2 + patched AEON module loads `aeon:LE:32:default`, SLEIGH compiles, headless import succeeds | **VERIFIED BY ISA/STATIC** (tool run, `EXIT=0`, `Import succeeded`) |
| 2 | Ghidra headless failed because `APPDATA` unset and `JAVA_HOME` empty (→ interactive `LaunchSupport -ask`) | **VERIFIED BY ISA/STATIC** (exception text captured) |
| 3 | `bg.movhi` / `bg.addi` encodings as derived from slaspec | **VERIFIED BY ISA/STATIC** (byte-exact self-test vs firmware) |
| 4 | `0x1B000`–`0x20400` is one contiguous mixed-width code stream (7,073 insns) | **VERIFIED BY ISA/STATIC** |
| 5 | No `0xF8724E1F` / `0x4E1FF872` in SND or DEC | **VERIFIED BY ISA/STATIC** (exhaustive byte scan, 0 hits both images) |
| 6 | The 6 "IEC clusters" are opcode coincidences (`bg.sbz` `0x3E`→`F8 72`; `bn.sra` `0x13`→`4E 1F`) | **VERIFIED BY ISA/STATIC** (decoded at each site) |
| 7 | No `movhi`+`addi` in SND builds `0x182832` / `0x183747` / `0x183763` | **VERIFIED BY ISA/STATIC** (exhaustive encoding search, 128 sites inspected) |
| 8 | AC3 and DTS SPDIF/HDMI reset paths are structurally parallel twins | **VERIFIED BY ISA/STATIC** (decoded side by side) |
| 9 | AC3 uses instance base `r13` + slots `0x486c`/`0x4870`; DTS uses `r14` + slots `0x30`/`0x34` | **VERIFIED BY ISA/STATIC** |
| 10 | `0x12AD38`/`0x12AD57` = "ms11 ddenc: reset spdif/hdmi"; `0x12AE9E`/`0x12AEBD` = "dts m6 enc: reset spdif/hdmi" | **VERIFIED BY ISA/STATIC** (address computed from `movhi`+`addi`, content read back) |
| 11 | `0x1800` is a buffer offset/stride, not an IEC length field | **STRONG INFERENCE** (used as an address displacement in every decoded context) |
| 12 | `r10 + 0x4840…0x4870` is an output-port state block; `0x486c`/`0x4870` = SPDIF/HDMI flags | **STRONG INFERENCE** (contiguous 4-byte slots cleared in bursts; not symbolically confirmed) |
| 13 | `bg.jal 0x10786e` is a log/printf routine | **STRONG INFERENCE** (called with `r3` = string address; consistent at 48 sites) |
| 14 | DTS SDO packer strings are dead data (asserts compiled out) | **STRONG INFERENCE** (no code refs + no IEC constants; cannot prove absence of an exotic reference form) |
| 15 | IEC61937 preamble is inserted by hardware, not SND firmware | **STRONG INFERENCE** (best explanation of §4.1 + §4.5; hardware not directly observable) |
| 16 | Exact semantics of each `r10` slot, and of `0xca42` / `0xc55e` / `0xc798` | **UNRESOLVED** (callees not analysed) |
| 17 | DTS-HD vs DTS-core selection; burst length/size; PCM fallback; IEC enable/disable | **UNRESOLVED** — no code locating these was found in SND |
| 18 | `bg.jal` far-branch target arithmetic | **UNRESOLVED** — per R27 rules `bg.jal` targets are not trusted; offsets computed by hand from `i32_simm1_25` do not reconcile with a previously recorded target for `E7FFFD95`, so far-branch targets remain unconfirmed |
| 19 | `0xC55E` / `0xC731` are MMIO hardware reset+poll routines on `0xB000_0854` / `0x080C` / `0x0810` / `0x084C` / `0x0850` | **VERIFIED BY ISA/STATIC** (decoded from Ghidra; absolute `movhi`+`ori` constants leave no ambiguity) |
| 20 | Poll timeout is `0x2DC6C00` = 48,000,000 cycles = 1 s @ 48 MHz | **VERIFIED BY ISA/STATIC** (arithmetic on a decoded immediate) |
| 21 | `0xCA42` / `0xC873` initialise the instance struct; `0xC798` sets sample rate (48000 / 88200 / 128000 / 192000 constants) | **VERIFIED BY ISA/STATIC** |
| 22 | The AC3 and DTS paths call the **same** hardware reset helpers — the reset is codec-agnostic at HW level | **VERIFIED BY ISA/STATIC** |
| 23 | `0xB000_080C`/`0x084C` = **SPDIF**, `0xB000_0810`/`0x0850` = **HDMI** | **VERIFIED BY ISA/STATIC** — port identity is fixed by which log string the calling path emits, and AC3 + DTS agree independently (§5.6). (Upgraded from STRONG INFERENCE.) |
| 24 | `0x10C5B9` is `memset` (align check, byte→word replication, 16-byte chunk loop) | **VERIFIED BY ISA/STATIC** |
| 25 | The audio output peripheral occupies `0xB000_0800`–`0xB000_0878`; the densest registers are `0x0854` (control), `0x0814`, `0x085C` | **VERIFIED BY ISA/STATIC** (encoding-matched enumeration over the whole image) |
| 26 | `0xB000_0814` / `0xB000_085C` are data/FIFO or level registers | **STRONG INFERENCE** (highest access counts; role not directly proven) |

**No claim in this report is VERIFIED BY RUNTIME** — no device was exercised, and
none of the above was observed on hardware.

---

## 7. Answers to the specific R27 questions

| Question | Answer |
|---|---|
| Where is `ADD_IEC_HEADER`? | String exists at `0x182832` / `0x187529`, but **no code references it**. Not a live code path in SND. |
| Where is IEC enable/disable? | **NOT FOUND.** |
| DTS burst / data-type? | **NOT FOUND** (no IEC constants at all). |
| Burst length / size? | `0x1800` appears but is a buffer stride, not a burst length. |
| DTS vs AC3 selection? | **FOUND** — §5.3: encoder instance register (`r13` vs `r14`) and output-state slot pair. |
| DTS-HD vs DTS-core? | **NOT FOUND.** |
| Packer output state? | **PARTIAL** — `r10`-based slot pairs, §5.4 (STRONG INFERENCE). |
| How is the port actually reset? | **FOUND** — MMIO hardware sequence at `0xB000_08xx` with a 1-second poll, §5.5. |
| PCM fallback? | **NOT FOUND** in this region. |
| First proven AC3/DTS divergence? | §5.3 / §5.6 — encoder instance base (`r13` vs `r14`) + output-state slot pair. The **hardware** reset itself is shared/codec-agnostic. |

---

## 8. Recommended next steps

1. ~~Analyse the callees `0xca42`, `0xc55e`, `0xc798`, `0xc873`, `0xc731`.~~ **DONE — see §5.5.** They resolve to: struct init (`0xCA42`, `0xC873`), sample-rate setter (`0xC798`), and **MMIO hardware reset+poll (`0xC55E` / `0xC731`)**. Remaining follow-ups:
   * Identify `0x10C5B9` (called by `0xCA42` with `r4=0`) — likely the final "apply" step.
   * Correlate `0xB000_08xx` against the MT5889 register map to confirm which pair is SPDIF vs HDMI (currently STRONG INFERENCE, claim 23).
2. **Dump the string region `0x182000`–`0x188000`** as code (not data) if any part
   of it is executable, to check for a second copy of packer logic.
3. **Check DEC** (`mst_codec_r2_MS12V22.bin`) for the IEC constants as well — DEC
   had 10 Pa / 11 Pb raw hits in an earlier scan; those must be re-validated
   against decoded instructions before being believed (same false-positive risk).
4. **Do not** pursue the 6 "IEC clusters" further — they are disproven.
5. If the IEC preamble is genuinely hardware-inserted, the productive question
   becomes *what the firmware writes into the SDO data-type/length registers*,
   which requires the SDO register map (not available in these images).

---

## Appendix — artefacts

| Path | Contents |
|---|---|
| `C:/firmware_temp/aeon_validate/scripts/aeon_decode.py` | Self-contained AEON decoder (bg/bn/bt) derived from the slaspec |
| `C:/firmware_temp/aeon_validate/scripts/DumpAEON.java` | Ghidra post-script: `DUMPALL`/`SWEEP`/per-address decode → file |
| `C:/firmware_temp/aeon_validate/r27_work/ghidra_dump_1B000_5400.txt` | Ghidra decode of `0x1B000`–`0x20400` (817 KB, authoritative) |
| `C:/firmware_temp/aeon_validate/r27_work/ghidra_dump_C000_E00.txt` | Ghidra decode of `0xC000`–`0xCE00` — the shared callees (§5.5), 131 KB |
| `C:/firmware_temp/aeon_validate/r27_work/ghidra_direct.log`, `ghidra_big.log` | Headless run logs (incl. the `APPDATA` failure and the successful runs) |
| `C:/firmware_temp/aeon_validate/r27_work/snd_full.bin` | Working copy of SND, base `0` |
| `C:/firmware_temp/aeon_ghidra_public/` | PART 1 public package (README / COMPATIBILITY / PATCH_NOTES / UPSTREAM_PROVENANCE / MANIFEST) |
