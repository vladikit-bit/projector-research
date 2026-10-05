# REPORT_runtime_phase8 — Tracing SND SHM +0xf8 into the R2 DSP firmware

**Phase:** R8 (static, read-only)
**Target:** Thundeal TD98 Pro / C50A — MStar MT5889 (AEON "R2" audio DSP)
**Module under analysis:** `kmods/utpa2k_expB_0x4d8_3_dts_license.ko`
(MD5 `4c5e6fbb10e24abc3b8d2b0490507983` — the currently deployed Exp-B image)
**Working dir:** `C:\firmware_temp\spdif_audio_investigation`
**Safety statement:** **Nothing was patched.** No change was made to `mik.ko`, `utpa2k.ko`,
SHM contents, device state, EDID, Kodi or any vendor setting. All work is static analysis of
files already on disk. In line with the R8 brief, **no patch is prepared or proposed here.**

---

## 0. Executive summary

| # | Question | Verdict |
|---|----------|---------|
| 1 | Where is the DSP firmware and how is it mapped? | **PROVEN** — full inventory + exact load map (§1–§3) |
| 2 | Find all reads of SND SHM +0xf8 in the firmware | **PARTIAL** — host-side producer fully mapped (§4); DSP-side consumer **NOT LOCATED** (§5, blocker documented) |
| 3 | Reconstructed meaning of +0xf8 | **STRONGLY SUPPORTED** — it is an **AC3/Dolby special-case flag**, *not* a DTS identity field (§6, §10) |
| 4 | Trace forward to IEC61937/SDO/burst selection | **PARTIAL** — SDO packer architecture recovered from firmware strings; the burst preamble is **not a stored constant anywhere** (§7) |
| 5 | Does DTS identity arrive by another channel? | **YES — 4 independent channels, only one of which is SHM +0xf8** (§8) |
| 6 | `mst_snd_r2` vs `mst_snd_r2_MS12V22` | **PROVEN** — different builds sharing only a 0x206-byte boot stub; MS12V22 is the active image (§9) |
| 7 | No patching | **COMPLIANT** |

**Headline result.** The DSP firmware *does* contain both an AC3 (Dolby Digital / DD+)
encoder path and a DTS (DTS-HD M6 / DTS:X core2 + SDO packer) path, and both have explicit
SPDIF variants. SND SHM **+0xf8** is the **only** codec-conditional field the host writes into
the SND SHM, and it is written **1 for AC3 and 0 for everything else** — including DTS.
Therefore:

> **+0xf8 is an "AC3 / Dolby-encoder active" special-case flag. It is not a DTS identity
> field. DTS identity is asserted by the *absence* of the AC3 flag plus three other channels
> (DEC SHM, the per-codec enable register, and the SDO/NonPCM transport mode).**

**Consequence for the "missing DTS-specific encoder configuration" hypothesis:**
R8 **supports but does not fully confirm** it. The asymmetry is real and now proven at the
*host→SHM* boundary with a complete map; what remains unproven is the exact DSP-side branch
that consumes it (§5, §12).

---

## 1. Sub-task 1 — Locating the DSP firmware images

The four images are **not separate files**. They are **ELF symbols inside `.data` of
`utpa2k.ko`** (which is why earlier greps for them as strings only ever hit `.strtab`).

`.data` of `utpa2k_expB…ko`: file offset **0x6592a0**, size 0x9c3e84.

| Symbol | `st_value` | Size | File offset in `.ko` |
|---|---|---|---|
| `mst_codec_r2` | `0x0016be18` | `0x0029c0f4` (2 736 372) | `0x007c50b8` – `0x00a611ac` |
| `mst_codec_r2_MS12V22` | `0x00407f0c` | `0x001e401c` (1 982 492) | `0x00a611ac` – `0x00c451c8` |
| `mst_snd_r2` | `0x005fb81c` | `0x0017e948` (1 567 048) | `0x00c54abc` – `0x00dd3404` |
| `mst_snd_r2_MS12V22` | `0x0077a164` | `0x001c1330` (1 839 920) | `0x00dd3404` – `0x00f94734` |

All four were extracted to `r8_out/*.bin` (`tools/r8_extract.py`).
Extraction was **verified against the raw `.ko`**: e.g. `dtsx-sdo/src/sdo_spdif_packer_api.c`
resolves to `.ko` offset `0x00da9599`, which falls inside the `mst_snd_r2_MS12V22` range —
confirming the SDO-packer strings below genuinely belong to the SND image and not to a
neighbouring symbol.

---

## 2. Sub-task 1 — Firmware selection: which image is live?

`HAL_AUDSP_DspLoadCode` @ VA `0x46f5a8`, size `0xb94`
(disassembly: `r8_out/HAL_AUDSP_DspLoadCode.asm`).

Decisive block at `0x46fb14`–`0x46fb68`:

```
0x0046fb14: movw  r2, #0        <<< REL sym=mst_snd_r2_MS12V22
0x0046fb18: movw  fp, #0        <<< REL sym=mst_snd_r2
0x0046fb24: movw  r7, #0        <<< REL sym=mst_codec_r2
0x0046fb28: ldr   r1, [r0, #0x4d0]        ; r1 = g_AudioVars2->field_0x4d0
0x0046fb38: cmp   r1, #4
0x0046fb40: moveq fp, r2                  ; ==4  ->  fp = mst_snd_r2_MS12V22
0x0046fb50: moveq r7, r2                  ; ==4  ->  r7 = mst_codec_r2_MS12V22
0x0046fb5c: movweq r8, #0x6930            ;       snd   len = 0x1b6930
0x0046fb60: movweq r5, #0x401c
0x0046fb68: movteq r5, #0x1e              ;       codec len = 0x1e401c
```

The copy lengths match the ELF symbol sizes exactly
(`0x1c1330 − 0xaa00 = 0x1b6930`, `0x29c0f4/0x1e401c` for the codec).

**The device currently runs `0x4d0 == 4` ⇒ `mst_snd_r2_MS12V22` + `mst_codec_r2_MS12V22`
are the active images.** All DSP-side conclusions in this report were therefore taken from
the MS12V22 image first, as instructed.

---

## 3. Sub-task 1 — Exact load / address mapping

### 3.1 DEC (decoder) DSP
```
0x0046fc54: mov  r0,#2 ; bl HAL_AUDIO_GetDspMadBaseAddr   -> MAD(2)
0x0046fc64: bl   MsOS_PA2KSEG1
0x0046fc6c: mov  r2,#0x700000
0x0046fc70: bl   memset                    ; zero MAD(2)[0 .. 0x700000]
0x0046fc80: mov  r1, r7                    ; mst_codec_r2[_MS12V22]
0x0046fc84: mov  r2, r5                    ; 0x29c0f4 / 0x1e401c
0x0046fc88: bl   memcpy                    ; MAD(2) <- whole codec image
0x0046fc8c: bl   HAL_DEC_R2_init_SHM_param
```

### 3.2 SND (post-processing / encoder) DSP
```
0x0046fd2c: mov  r0,#2 ; bl HAL_AUDIO_GetDspMadBaseAddr
0x0046fd34: adds r7, r0, #0x700000         ; SND_BASE = MAD(2) + 0x700000
0x0046fd44: bl   MsOS_PA2KSEG1
0x0046fd48: mov  r1, fp                    ; mst_snd_r2[_MS12V22]
0x0046fd4c: mov  r2, #0xa000
0x0046fd50: bl   memcpy                    ; SND_BASE[0x0000:0xa000] <- blob[0:0xa000]
0x0046fd60: movw r2, #0x5600 ; movt r2,#0x6f
0x0046fd64: add  r0, r0, #0xaa00
0x0046fd74: bl   memset                    ; zero SND_BASE+0xaa00 .. +0x6f5600
0x0046fd84: add  r0, r0, #0xaa00
0x0046fd88: add  r1, fp, #0xaa00
0x0046fd8c: mov  r2, r8                    ; 0x1b6930 (MS12V22)
0x0046fd90: bl   memcpy                    ; SND_BASE+0xaa00 <- blob[0xaa00:]
0x0046fd94: bl   HAL_SND_R2_init_SHM_param
0x0046fda8: bl   HAL_SND_R2_EnableR2(1)
0x0046fdb0: bl   HAL_DEC_R2_EnableR2(1)
```

### 3.3 Resulting SND memory map

| DSP-region offset | Size | Source | Content |
|---|---|---|---|
| `+0x0000` | 0xa000 (40 KB) | `blob[0x0000:0xa000]` | Lower / PM region. **Only the first 0x206 bytes are meaningful** — a boot/vector stub identical across all four images; `0x0200`–`0xa000` is `0xFF` fill. |
| **`+0xa000`** | 0xa00 (2 560 B) | *not copied by the loader* | **SND SHM.** Deliberately skipped between the two `memcpy`s. Zeroed by `HAL_SND_R2_init_SHM_param` (0x8b8 bytes). |
| `+0xaa00` | 0x1b6930 | `blob[0xaa00:]` | Main firmware image: code + rodata + data. |

### 3.4 SHM base pointers (host side)

```
HAL_SND_R2_init_SHM_param @0x4582a8  and  HAL_SND_R2_Set_SHM_PARAM @0x459dfc:
    movw r2, #0xa000
    movt r2, #0x70            ; 0x0070a000
    adds r0, r0, r2           ; MAD(2) + 0x0070a000
    bl   MsOS_PA2KSEG1        ; -> g_virSndR2shm
```

> ### ⚠ Correction to REPORT_runtime_phase7A
> R7-A states `g_virSndR2shm = DSP MAD base + 0x70a0000`. That is a **typo**.
> The code emits `movw r2,#0xa000 / movt r2,#0x70`, i.e. **`0x0070a000`**, and the loader
> confirms it (`MAD(2) + 0x700000` for the SND base, `+0xa000` for the SHM).
> **Correct value: `g_virSndR2shm = PA2KSEG1( MAD(2) + 0x0070a000 )` = `SND_BASE + 0xa000`.**
> All R7-A offsets *within* the SHM (including `+0xf8`) are unaffected.

`g_virDecR2shm = MAD(2) + 0xe0001000` is carried over from R7-A; it was **not** re-verified in R8.

### 3.5 What `HAL_SND_R2_init_SHM_param` establishes about +0xf8

```
0x004582f4: movw r2, #0x8b8
0x004582fc: bl   memset          ; shm[0 .. 0x8b8) = 0
   ...then explicit initialisers (see §4.2)
```

**+0xf8 lies inside the memset range and is NOT in the explicit initialiser list.**
⇒ **After firmware load, SND SHM +0xf8 == 0.** It only becomes non-zero if a host call to
`HAL_SND_R2_Set_SHM_PARAM(0x6e, …)` sets it.

---

## 4. Sub-task 2/3/5 — The complete host→SND SHM write map

`HAL_SND_R2_Set_SHM_PARAM` @ `0x459dfc`, size `0x638`.
Signature recovered from the prologue:
`r0 = param id`, `r1 = instance (must be < 2)`, `r2 = value (held in r7)`, `r3 = aux (r8)`.

```
0x00459e54: cmp  r4, #2        ; instance
0x00459e58: bhs  #0x45a05c     ; >=2 -> log & bail (no SHM write)
0x00459e5c: sub  r0, sb, #0x59 ; index = param - 0x59
0x00459e60: cmp  r0, #0x76
0x00459e64: bhi  #0x45a3cc     ; default
0x00459e68: add  r1, pc, #0
0x00459e6c: ldr  pc, [r1, r0, lsl #2]   ; jump table @0x459e70
```

**The table has 119 entries, not 30** — the bound is `index <= 0x76`, so the valid param
range is **0x59 … 0xCF**. (R7-A only exercised 0x6e / 0x9d / 0x9a / 0x4c; the range matters
because it defines the *complete* set of things the DSP can be told.)

Every handler has the form `str r7, [r6, #off] ; b <exit>` (a few are multi-field, noted).

### 4.1 Complete param → SND SHM offset table

| param | SHM offset | notes |
|---|---|---|
| 0x59 | (special — writes `[r0+0x70]`, then 0x88) | non-SHM-relative first store |
| 0x5a | `0x88` | |
| 0x5b–0x64 | — | default (no write) |
| 0x65 | `0x90` | |
| **0x66** | `0xe4` | |
| 0x67 | — | default |
| 0x68 | `0xe0` | |
| 0x69 | `0x94` | |
| 0x6a | `0x134` | |
| 0x6b | `0xd8` | |
| 0x6c | `0xdc` | |
| 0x6d | `0xf4` | |
| **0x6e** | **`0xf8`** | **← is-AC3 (the R8 subject)** |
| 0x6f | `0xf0` | |
| 0x70 | `0x118` | |
| 0x71 | `0x11c` | clamped `<= 0xfe` |
| 0x72 | `0x120` | clamped `<= 0xfe` |
| 0x73 | `0x124` | |
| 0x74 | `0x128` | |
| 0x75 | `0x12c` | |
| 0x76–0x84 | — | default |
| 0x85 | `0x144` | read-modify-write: keeps bit 15, `bfc r7,#0xf,#0x11` |
| 0x86 | `0x148` | same RMW form |
| 0x87 | `0x14c` | same RMW form |
| 0x88 | — | default |
| 0x89 / 0x8a / 0x8b | `0x144` / `0x148` / `0x14c` | second bank of the same RMW |
| 0x8c | — | default |
| 0x8d | `0x154` | |
| 0x8e | `0x158` | |
| 0x8f | `0x15c` | |
| 0x90–0x9a | — | default |
| 0x9b | `0x13c` | |
| 0x9c | `0x140` | |
| **0x9d** | **`0x8c`** | R7-A: computed via `clz;lsr`, **always 0** — degenerate |
| 0x9e–0x9f | — | default |
| 0xa0 | `0x104` | |
| 0xa1–0xb1 | — | default |
| 0xb2 / 0xb3 / 0xb4 / 0xb5 / 0xb6 | `0x164` / `0x168` / `0x16c` / `0x170` / `0x174` | |
| 0xb7 | `0x194` | |
| 0xb8–0xb9 | — | default |
| 0xba | `0x10c` | clamped: `if (v-1) >= 0x7800 then v = 0x7800` |
| 0xbb | `0x110` | |
| 0xbc | `0x114` | clamped `<= 0x1e4` |
| 0xbd–0xc1 | — | default |
| 0xc2 | `0x328` **+** `0x17c = 1` **+** `0x180` | flush between each |
| 0xc3 | `0x32c` **+** `0x184` | |
| 0xc4 | `0x330` **+** `0x188` | |
| 0xc5 | `0x334` **+** `0x18c` | |
| 0xc6 | `0x338` **+** `0x190` | |
| 0xc7 | `0x198` | `mvn` — toggles/inverts |
| 0xc8–0xcf | — | default |

### 4.2 Fields initialised at load time (`HAL_SND_R2_init_SHM_param`)

| SHM offset | init value | inference |
|---|---|---|
| `0x858` (byte) | `1` | flag |
| `0xec` | `1` | |
| `0x118` | `0x20` | |
| `0x140` | `0x1111` | |
| `0x144` / `0x148` / `0x14c` | `0xc00` | |
| `0x10c` | `0x7800` | delay/latency budget (param 0xba clamps to the same 0x7800) |
| `0x328`–`0x338` | `0x800000` | **5 pending output gains** |
| `0x180`–`0x190` | `0x800000` | **5 applied output gains** (0x800000 = unity) |
| `0x104`, `0x110`, `0x114`, `0x11c`, `0x120`, `0x124`, `0x13c`, `0x154`, `0x158`, `0x15c`, `0x164`–`0x174`, `0x198` | `0` | |
| **`0xf8`** | **not touched → 0** | **the R8 subject** |

**Structural inference (new in R8).** Params 0xc2–0xc6 write a *pending* value into
`0x328+4k`, then set `0x17c = 1`, then write the *applied* value into `0x180+4k`, with
`MsOS_FlushMemory` between each step. That is a 5-slot gain/volume bank with a commit flag.
The DSP-side debug format string

```
0x127efc: [%s] spk[%06x,%06x,%06x,%06x,%06x,%06x,%06x,%06x] [%06x,%06x] hp[%06x,%06x] spdif[%06x,%06x] ch6-10[...]
```

shows the DSP tracks **spk / hp / spdif / ch6-10** as separate output paths — i.e.
**SPDIF has its own slot in this 5-way output bank.** This is the first concrete link between
a SND SHM field and the SPDIF output.

### 4.3 Which of these are codec-conditional?

R7-A established — and the completed map above does not contradict — that **param 0x6e
(→ SHM +0xf8) is the only codec-conditional SND SHM write**. Its value is computed as

```
sub r1, [r1, #0x4f4], #5     ; codec_id - 5
clz r1, r1                   ; 32 if (codec_id-5)==0  else small
lsr r8, r1, #5               ; 1 iff codec_id == 5 (AC3), else 0
```

⇒ **+0xf8 == 1 ⟺ codec is AC3 (5); +0xf8 == 0 for DTS (9) and for every other codec.**
There is no positive DTS value and no second codec-id field in the SND SHM.

---

## 5. Sub-task 2 — Searching the DSP firmware for reads of +0xf8

### 5.1 What was established about the core and the ISA

* The core is **AEON** (confirmed by the firmware strings
  `AEON: call hal_init() in exception.S` and `AEON: call main()`).
* `objdump` in this environment has **no** `or1k` / `openrisc` / `or32` / `aeon` target.
* Capstone (5.0.7, in the venv) has **no** OR1K/AEON module.
* AEON is OR1K-*shaped* but **is not stock OR1K** (see below).

### 5.2 Partial ISA recovery achieved

Because all four images share an identical boot stub, and only three words differ between
them, those three words are the per-image link constants. One of them is:

```
snd   : c8 63 aa 00        codec : c8 63 d0 00
```

`0xaa00` is **exactly** the SND data-base offset used by the loader (`add r1, fp, #0xaa00`),
and `0xd000` is the codec's. This pins the instruction format:

```
| opcode(6) | rD(5) | rA(5) | imm16(16) |     (big-endian, 32-bit fixed length)
op = 0x32  ->  load/OR immediate      (c8 e0 48 00 -> r7 = 0x4800 ; c8 e0 48 02 -> r7 = 0x4802)
op = 0x30  ->  high / second immediate (c0 60 00 01 -> r3  = 1, preceding the 0x32 form)
```

Note `movhi`/`ori` are **0x30 / 0x32 here**, whereas stock OR1K uses **0x06 / 0x2a** — so
AEON is a derived core with its own opcode numbering and a stock OR1K disassembler would
mis-decode it.

### 5.3 Targeted searches and their results

| Search | Result |
|---|---|
| literal `0xf8` as an imm16 with the recovered load-immediate opcode | **1 hit** @ blob `0x0f31c3` (`c8 d1 00 f8` → `op 0x32, r6, r17, 0x00f8`). Random expectation for this pattern in 1.84 MB ≈ 0.44, so **1 hit is not statistically significant**. |
| imm16 `0xa000` (SHM base) with the same opcode | **4 hits**: `0x0a3a37` (rA=15), `0x0dea40` (**rA=0 — a clean `li r25, 0xa000`**), `0x0e1e75` (rA=23), `0x0ecaca` (rA=23). Random expectation ≈ 0.44 ⇒ ~9× enrichment. Context inspection shows `0x0e1e75` and `0x0ecaca` sit inside a repeating **bit-mask table** (`0x0180, 0x0600, 0x1800, 0xa000, 0x4000, 0x1000, 0x0400, 0x2000 …`) and are **false positives**. |
| **Best surviving candidate** | `0x0dea40 : cb 20 a0 00` = `li r25, 0xa000` (rA = r0). Nearby: `c9 a0 c0 00` = `li r13, 0xc000`, `c9 a0 e0 00` = `li r13, 0xe000`. These look like DSP-side region bases. |
| opcode-concentration scan to isolate code regions | Inconclusive — the region `0xb000…` is a single flat 1.7 MB code+rodata+data image; no dominant function-epilogue word (max 289/283 648 = 0.1 %, far below the 1–3 % typical of real code). |
| embedded symbol / function-name table | **Absent.** 7 488 printable runs, 1 060 identifier-like, but gaps are irregular (top gap 24 occurs only 25×) — no name table. |

**Conclusion (honest):** the instruction(s) that consume SND SHM +0xf8 could **not** be
located. The blocker is the absence of an AEON disassembler, and AEON's opcode numbering
differs from stock OR1K so a stock decoder cannot be substituted safely.

**What would unblock it** (no patching required): obtain the AEON opcode map (MStar
toolchain / `aeon-elf-objdump`), or write a decoder validated against the shared 0x206-byte
boot stub, then disassemble `blob[0xaa00:]`. The single best entry point is
`blob 0x0dea40` (`li r25, 0xa000`).

---

## 6. Sub-task 3 — Reconstructed semantics of +0xf8

Combining §3.5, §4.1 and §4.3:

| Property | Value |
|---|---|
| Address | `SND_BASE + 0xf8`, i.e. `MAD(2) + 0x70a0f8` (host view) / `0xa0f8` (DSP view) |
| Value after firmware load | `0` (inside the 0x8b8 memset, never initialised) |
| Written by | `HAL_SND_R2_Set_SHM_PARAM(param=0x6e, instance<2, value)` → `str r7,[r6,#0xf8]` @ `0x45a1a0` |
| Value for AC3 (codec 5) | **1** |
| Value for DTS (codec 9) | **0** |
| Value for all other codecs | **0** |
| Read-back path | `HAL_SND_R2_Get_SHM_PARAM` @ `0x45b3d8` |
| Neighbouring fields | `0xf0` (0x6f), `0xf4` (0x6d), `0xd8` (0x6b), `0xdc` (0x6c), `0x134` (0x6a), `0x118`–`0x12c` (0x70–0x75) — all generic, none codec-conditional |
| Is a codec-ID read in the same host function? | Yes — `ldr r1,[r1,#0x4f4]` feeds `sub #5; clz; lsr #5`, but the ID is *consumed* into a single boolean; **only the boolean crosses into the SHM** |

**Semantic verdict.** +0xf8 is a **one-bit "AC3 / Dolby encoder active" flag**. It carries
**no DTS information**. Setting it to 0 does not "select DTS"; it says "this is not AC3".

---

## 7. Sub-task 4 — Forward trace to IEC61937 / SDO / burst selection

### 7.1 Negative result: there is no IEC61937 sync constant anywhere

Exhaustive search of the **entire** `mst_snd_r2_MS12V22` image (all rotations and both
byte orders) using `tools/r8_scan2.py`:

| Pattern | Hits |
|---|---|
| IEC 61937 Pa `F8 72 4E 1F` (BE) | **0** |
| IEC 61937 Pa `72 F8 1F 4E` (LE) | **0** |
| IEC 61937 Pa `1F 4E 72 F8` (halfword-swapped BE) | **0** |
| IEC 61937 Pa `4E 1F F8 72` (halfword-swapped LE) | **0** |
| DTS 16-bit BE sync `7F FE 80 01` | **0** |
| DTS 16-bit LE sync `FE 7F 01 80` | **0** |
| DTS 14-bit BE sync `1F FF E8 00` | **0** |
| DTS 14-bit LE sync `FF 1F 00 E8` | **0** |

Two-byte half-patterns (`F872`, `4E1F`, `1F4E`, `72F8`) occur 5–32 times each — i.e. at
exactly the rate expected by chance in a 1.84 MB image (≈28 expected). **They are noise.**

Combined with R7-A's identical negative result for the LKM
(`r6_out/scan_iec_packer.txt`: "NO IEC61937 sync constant found in data/rodata"):

> **The IEC61937 burst preamble is never present as a stored constant in either the host
> driver or the DSP firmware image.** It is constructed at run time (computed/parameterised),
> or emitted by dedicated hardware/packer logic. Exact burst-info / data-type codes therefore
> cannot be recovered by constant search — exactly as the R8 brief anticipated.

### 7.2 Positive result: the DSP's SPDIF/SDO architecture is fully visible in its strings

`mst_snd_r2_MS12V22` contains a complete DTS:X SDO packer library and its assertion strings:

**DTS output packers**
```
0x183319  DTSX_CORE2_API_SDO_Packer
0x183516  DTSDecSDOPacker_API_Initialize
0x1834dc  DTSDecSDOPacker_API_GetParam
0x1834f9  DTSDecSDOPacker_API_SetParam
0x183535  dtsx-sdo/src/sdo_spdif_packer_api.c
0x1836e4  SDO_SpdifPacker_SAPI_GetSampleRate
0x183707  SDO_SpdifPacker_SAPI_GetFrameSize
0x183729  SDO_SpdifPacker_SAPI_Transmit
0x183747  SDO_SpdifPacker_SAPI_GetIEC      <-- IEC header accessor
0x183763  SDO_SpdifPacker_SAPI_SetIEC      <-- IEC header accessor
0x18377f  SDO_SpdifPacker_SAPI_StartFrame
0x1877cc  dtsx-sdo/src/sdo_hdmi_packer_api.c
0x187a25..0x187aa1  SDO_HdmiPacker_SAPI_{GetSampleRate,GetFrameSize,Transmit,SetMode,StartFrame}
0x187c6c  DTSSPDIFPackFrame
0x18362b  DTSSPDIFPackFrame( p_spdif_packer, nBytesToPack, nPCMformat, nStride )
```

**The two output selectors are explicit**
```
DTS_SDO_SPDIF_OUT   (DTSDecSDOPacker_API_StartFrame / _GetFrameSize / _Process)
DTS_SDO_HDMI_OUT    (same trio)
```
called from two distinct sites — a `p_file_player` site (`0x182564`–`0x18293a`) and a
`pstCommon` multi-frame site (`0x187213`–`0x187629`).

**The IEC header is a *parameter*, not a literal** (this is the key finding)
```
0x1827e0: DTSDecSDOPacker_API_SetParam(p_file_player->SDOPacker,
              DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32, &IEC_Header)
0x1874db: DTSDecSDOPacker_API_SetParam(pstCommon->SDOPacker,
              DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32, &IEC_Header)
```
⇒ the SPDIF IEC header is supplied by the caller through `…_ADD_IEC_HEADER_I32`.
**This is why no sync constant exists in the image** (§7.1) and it is the most likely place
where the AC3-vs-DTS data-type value is decided.

**Both AC3 and DTS have explicit SPDIF encoder paths, and both reset SPDIF**
```
0x1296a6  MS12V2 ddpe : DD mode : reset spdif 48K !![1]        <- Dolby (AC3) encoder
0x1295ce  MS12V2 ddpe : DDP mode : reset hdmi 192K !![1]       <- Dolby+ encoder
0x1296d5  MS12V2 ddpe : Spdif dd encoder skip frame !! Spdif level:0x%x(%dms)
0x12ad38  ms11 ddenc: reset spdif !![1]
0x12ad06  ddenc_ret:%d, ac3e_bytesWritten:%d
0x12ae9e  dts m6 enc: reset spdif !![1]                        <- DTS-HD M6 encoder
0x12aebd  dts m6 enc: reset hdmi !![1]
0x12c66c  dtsX spdif output reset !!                           <- DTS:X
0x12c63c  [dtsX enc] hdmi output reset !!(sr=%d, hbr=%d)
0x12c693  dtsX_core2   0x12c6e8  dtsX_core2_init   0x12c6f8  dtsX_core2_hook
```

**The DSP maintains an explicit non-PCM selector and an SPDIF owner**
```
0x126ea0  [SPDIF] spdif ISR: fs:%d(%d), nonpcm_sel:%d, state:%d, owner:%d
0x126ee1  [HDMI]  hdmi  ISR: fs:%d(%d), nonpcm_sel:%d, state:%d, owner:%d, decimation:%d, hbr:%d
0x126862  Spdif start , level = 0x%x, pcmIn = 0x%x(%x)
0x126b76  [Settings] sound_mode:%d, post_mixing:%d, spdif_npcm_delay_flag:%d
0x130a33  SPDIF      0x130a64  <ac3>     0x130a6c  <ac3p>     0x130a7e  <dts>     0x130af1  <dtsX>
```

So the DSP-side SPDIF path is a **state machine with `nonpcm_sel`, `state` and `owner`**,
and the codec family (`<ac3>`, `<ac3p>`, `<dts>`, `<dtsX>`) is a first-class concept inside
the firmware. That is where the AC3-vs-DTS decision ultimately lands — but the branch that
sets `nonpcm_sel` from SHM +0xf8 could not be located (§5).

---

## 8. Sub-task 5 — Other channels by which DTS identity reaches the DSP

**This is the most important corrective finding of R8.** Codec identity is **not** carried
only by SND SHM +0xf8. Four independent channels exist:

| # | Channel | Carrier | AC3 | DTS | Where |
|---|---|---|---|---|---|
| 1 | **SND SHM +0xf8** | shared memory | `1` | `0` | `HAL_SND_R2_Set_SHM_PARAM(0x6e)` @ `0x45a1a0` |
| 2 | **DEC SHM** (decoder side) | shared memory | codec-dependent | codec-dependent | `HAL_DEC_R2_Set_SHM_PARAM`, table `0x45a4c0`; R7-A: param `0x9a`→DEC `0x1158`, `0x4c`→DEC `0xe60` |
| 3 | **Per-codec enable register** | **hardware register, not SHM** | bit **2** | bit **6** | `HAL_AUDIO_AbsWriteMaskByte(idx = codec_id − 3, 0x10)` |
| 4 | **SDO / NonPCM transport mode** | driver→DSP mode command | set via `utpa2k` SDO/NonPCM/SetMode path | same | R5/R7 chain; `nonpcm_sel` visible in DSP ISR string |

Key consequences:

* Channel 3 is a **positive DTS identity signal** — `HAL_AUDIO_AbsWriteMaskByte` is called
  with `idx = codec_id − 3`, so DTS (9) → index 6 → **bit 6**, AC3 (5) → index 2 → **bit 2**.
  It does **not** go through the SND SHM, so it is invisible to a SHM-level comparison.
* Channel 4 (`nonpcm_sel`) is the transport-level gate; it is what actually decides whether
  SPDIF carries PCM or a compressed burst.
* Channel 1 (+0xf8) is the **encoder-side** selector, and it is *negative* with respect to DTS.

**Answer to the key question of sub-task 5:** `+0xf8` is **not** the codec selector. It is an
**AC3 special-case flag on the encoder side**. DTS identity is carried positively by
channels 2, 3 and 4, and is only *implied* by the zero value of +0xf8.

---

## 9. Sub-task 6 — `mst_snd_r2` vs `mst_snd_r2_MS12V22`

| | `mst_snd_r2` | `mst_snd_r2_MS12V22` |
|---|---|---|
| size | 0x17e948 (1 567 048) | 0x1c1330 (1 839 920) |
| data copy len | 0x173f48 | 0x1b6930 |
| first differing byte | — | **0x206** |
| 4 KB blocks identical / differing | — | **8 identical / 374 differing** (of 382 comparable) |

**They are not "the same image with a patch".** They share only the first **0x206 bytes** —
the AEON boot/vector stub — and then diverge completely, because they are two separately
linked builds with different layouts.

**The boot stub is common to all four images** (`mst_codec_r2`, `mst_codec_r2_MS12V22`,
`mst_snd_r2`, `mst_snd_r2_MS12V22`), differing in exactly three words:

| offset | snd | snd_MS12V22 | codec | codec_MS12V22 | meaning |
|---|---|---|---|---|---|
| `0x404` | `c0 20 09 a1` | `c0 20 0a 81` | `c0 20 0a e1` | `c0 20 09 61` | per-image link constant |
| `0x408` | `c8 21 f8 b0` | `c8 21 a8 80` | `c8 21 05 50` | `c8 21 84 50` | per-image link constant |
| `0x17c` | `c8 63 aa 00` | `c8 63 aa 00` | `c8 63 d0 00` | `c8 63 d0 00` | **data-base: 0xaa00 (SND) vs 0xd000 (codec)** — matches the loader exactly |

**Implication for the investigation:** every conclusion drawn from `mst_snd_r2_MS12V22`
cannot be assumed to hold for `mst_snd_r2`, and vice-versa. Since the device runs
`0x4d0 = 4`, **only the MS12V22 findings are load-bearing** — which is why all DSP-side
analysis above was done on that image.

Structural comparison of both SND images (0x1000 granularity):

```
0x0000-0x1000  boot/vector stub + tables   (32 % zeros, no 0xff)
0x1000-0x2000  per-image tables            (36 % zeros)
0x2000-0xa000  0xFF fill                   (unused IMEM)
0xa000-0xaa00  SHM slot (skipped by loader)
0xaa00-...     main image: 99 % non-zero halfwords, ~900 unique 32-bit words per 4 KB
```

---

## 10. Sub-task "final question" — What the DSP does with +0xf8 = 1 vs 0

> **Q: What does the R2 DSP actually do when SND SHM +0xf8 is 1 versus 0, and what
> information does it use to decide whether to emit AC3 or DTS IEC61937 bursts?**

### What is proven

1. **+0xf8 = 1 ⟺ the host has selected AC3 (codec 5).** Proven at the host boundary
   (`sub #5; clz; lsr #5` → boolean; SHM +0xf8 ← boolean; `str r7,[r6,#0xf8]` @ `0x45a1a0`).
2. **+0xf8 = 0 for DTS (codec 9) and for every other codec.** There is no third value and no
   positive DTS encoding in the SND SHM.
3. **The DSP firmware contains both output families, each with an explicit SPDIF path** —
   Dolby/MS12 (`MS12V2 ddpe : DD mode : reset spdif 48K`, `ms11 ddenc: reset spdif`) and DTS
   (`dts m6 enc: reset spdif`, `dtsX spdif output reset`, `SDO_SpdifPacker_SAPI_*`,
   `DTSSPDIFPackFrame`).
4. **The IEC header is passed as a parameter** (`DTS_PARAM_SDO_PACKER_SPDIF_ADD_IEC_HEADER_I32`),
   and **no sync-word constant exists in either the driver or the firmware** — so the AC3/DTS
   burst type is decided by *which packer is invoked and with what parameter*, not by a
   stored constant.
5. **The DSP keeps a `nonpcm_sel` state** in its SPDIF ISR — this is the transport-level
   PCM-vs-compressed decision.

### What is therefore the defensible model (STRONGLY SUPPORTED, not proven)

* **+0xf8 = 1** → the SND DSP takes the **Dolby / MS12 DD(+) encoder** branch for the SPDIF
  output; the IEC61937 stream is produced by the DD/DDP encoder with AC3 burst-info.
* **+0xf8 = 0** → the SND DSP does **not** take that branch. It falls through to the
  alternative encoder: **DTS-HD M6 / DTS:X core2 + `SDO_SpdifPacker`** when the DTS path is
  enabled, or plain MS12 PCM otherwise.
* The selection between "DTS" and "PCM" in the `= 0` case is **not** made by +0xf8. It is
  made by the other three channels in §8 — in particular the **per-codec enable register bit
  (DTS → bit 6)** and the **SDO/NonPCM transport mode**.

### The precise limitation

The individual AEON instruction(s) that branch on +0xf8, and the code that sets
`nonpcm_sel` / selects `DTS_SDO_SPDIF_OUT`, **were not located** (§5). The model above is
inferred from (a) the complete host→SHM map, (b) the firmware's own debug/assertion strings,
and (c) the proven absence of any other codec-conditional SHM field. It is consistent with
every piece of evidence gathered, but it is **not** a disassembly-level proof.

**One-line answer:**
> **+0xf8 is an "is-AC3" flag, not a codec selector. 1 = take the Dolby/MS12 SPDIF encoder
> path; 0 = not-AC3, and the choice between DTS and PCM is then made by the per-codec enable
> register (DTS → bit 6) and the SDO/NonPCM mode, not by +0xf8. The DSP therefore never
> receives a positive "this is DTS" marker in the SND SHM at all.**

---

## 11. Verdict on the "missing DTS-specific encoder configuration" hypothesis

| Claim | Status |
|---|---|
| AC3 receives a codec-specific SHM configuration that DTS does not | **CONFIRMED** — and now bounded: exactly one field, +0xf8, out of the 46 writable SND SHM offsets |
| The complete set of host→SND SHM fields is known | **CONFIRMED** — 119-entry jump table fully enumerated (§4.1) |
| DTS has no positive identity in the SND SHM | **CONFIRMED** — it is implied by `+0xf8 == 0` only |
| DTS identity exists elsewhere (non-SHM) | **CONFIRMED** — per-codec enable register bit 6 (§8 ch. 3) |
| The DSP branches on +0xf8 to choose AC3-vs-DTS SPDIF output | **NOT ESTABLISHED** — no AEON disassembler available (§5) |
| The missing-DTS-config hypothesis is the cause of the failure | **SUPPORTED, NOT CONFIRMED** at the DSP level |

R8 has **narrowed but not closed** the gap: the boundary has moved from "the host writes
something AC3-specific" (R7-A) to "the host writes exactly one AC3-specific bit, out of a now
completely enumerated field set, and the DSP has both code paths available."

---

## 12. If a patch ever becomes justified — the exact surfaces

Recorded for completeness only. **No patch was prepared, and none should be applied on the
strength of R8 alone**, because the DSP-side consumer is still unproven.

| Surface | Location | Current | Would need |
|---|---|---|---|
| SND SHM +0xf8 | `utpa2k.ko` VA `0x45a1a0` (`str r7,[r6,#0xf8]`, param `0x6e`) | `1` iff codec == AC3 | a positive DTS branch — but the *meaning* of any other value is unknown, so this is unsafe without the DSP decode |
| Per-codec enable register | `HAL_AUDIO_AbsWriteMaskByte(idx = codec_id − 3, 0x10)` | DTS → bit 6, AC3 → bit 2 | verify DTS bit 6 is actually asserted at runtime (runtime probe, not a patch) |
| SDO / NonPCM transport mode | `utpa2k` SDO + NonPCM + SetMode path | set by R5 Exp-B | already handled by the deployed Exp-B image |
| DEC SHM codec config | `HAL_DEC_R2_Set_SHM_PARAM`, table `0x45a4c0` | params `0x9a`→`0x1158`, `0x4c`→`0xe60` | unverified in R8 |
| Firmware image itself | `mst_snd_r2_MS12V22` @ `.ko` file `0x00dd3404`, 0x1c1330 bytes | shipped binary | **must not be patched** — no toolchain, no way to verify |

**Recommended next step (read-only):** resolve the AEON opcode map and disassemble
`mst_snd_r2_MS12V22` from `blob 0x0dea40` (`li r25, 0xa000`). That single step converts
§10's STRONGLY SUPPORTED model into a proof, and is the only remaining blocker.

---

## 13. Confidence classification

| Finding | Confidence |
|---|---|
| Four firmware images: identity, file offsets, sizes | **PROVEN** |
| Firmware selection by `0x4d0 == 4`; MS12V22 active | **PROVEN** |
| SND load map: code `+0x0000`, SHM `+0xa000`, data `+0xaa00` | **PROVEN** |
| `g_virSndR2shm = MAD(2) + 0x70a000` (R7-A typo corrected) | **PROVEN** |
| `+0xf8` is zero after load; never initialised | **PROVEN** |
| Complete param→SND SHM offset table (119 entries) | **PROVEN** |
| `0x6e` is the only codec-conditional SND SHM write | **PROVEN** |
| No IEC61937 / DTS sync constant in the DSP image or the LKM | **PROVEN** |
| DSP contains AC3 *and* DTS SPDIF encoder/packer paths | **PROVEN** (strings) |
| IEC header supplied via `…_ADD_IEC_HEADER_I32` parameter | **PROVEN** (assertion strings) |
| `mst_snd_r2` vs `mst_snd_r2_MS12V22` differ beyond 0x206 | **PROVEN** |
| AEON instruction format `|op(6)|rD(5)|rA(5)|imm16|`, op 0x32 = load-imm | **STRONGLY SUPPORTED** |
| DSP-side SHM base candidate `blob 0x0dea40` = `li r25, 0xa000` | **WEAK / CANDIDATE** |
| `+0xf8` semantics = "is-AC3", not a DTS selector | **STRONGLY SUPPORTED** |
| Exact DSP branch on +0xf8, and `nonpcm_sel` assignment | **NOT ESTABLISHED** |

---

## 14. Artifacts produced in R8 (all read-only)

| Path | Content |
|---|---|
| `r8_out/mst_codec_r2.bin` | extracted image, 2 736 372 B |
| `r8_out/mst_codec_r2_MS12V22.bin` | extracted image, 1 982 492 B |
| `r8_out/mst_snd_r2.bin` | extracted image, 1 567 048 B |
| `r8_out/mst_snd_r2_MS12V22.bin` | extracted image, 1 839 920 B |
| `r8_out/HAL_AUDSP_DspLoadCode.asm` | 742-line relocation-aware disassembly |
| `r8_out/HAL_SND_R2_Set_SHM_PARAM.asm` | 399-line relocation-aware disassembly |
| `tools/r8_findfw.py` | locates firmware symbols / sections |
| `tools/r8_symfw.py` | resolves firmware blob symbols |
| `tools/r8_extract.py` | extracts the four blobs |
| `tools/r8_loader.py` | finds relocation sites referencing firmware symbols |
| `tools/r8_diswin.py` | relocation-aware disassembly of any VA window (reusable) |
| `tools/r8_strings.py` | string mining in a blob |
| `tools/r8_syms.py` | embedded symbol-table hunt |
| `tools/r8_map.py` | per-block structural map |
| `tools/r8_codeloc.py` | opcode-concentration code-region locator |
| `tools/r8_freq.py` | frequent-word (prologue/epilogue) analysis |
| `tools/r8_immscan.py` | imm16 ↔ known-SHM-offset correlation |
| `tools/r8_sndshmmap.py` | **resolves the SND SHM jump table → full param map** |
| `tools/r8_cmp.py` | two-blob comparison + sync-constant scan |
| `tools/r8_scan2.py` | exhaustive sync-word + SHM-immediate scan |

---

## 15. Corrections and open items

**Correction to R7-A:** `g_virSndR2shm` is `MAD(2) + 0x0070a000`, **not** `0x70a0000`
(§3.4). SHM-internal offsets are unaffected.

**Carried over, not re-verified in R8:**
* `g_virDecR2shm = MAD(2) + 0xe0001000`
* DEC SHM params `0x9a` → `0x1158`, `0x4c` → `0xe60`
* `HAL_AUDIO_AbsWriteMaskByte(idx = codec_id − 3, 0x10)`

**Open items, in priority order:**
1. Obtain / derive the AEON opcode map; disassemble `mst_snd_r2_MS12V22` from `blob 0x0dea40`.
2. Locate the code that sets `nonpcm_sel` and selects `DTS_SDO_SPDIF_OUT`.
3. Re-derive the DEC SHM map with the same jump-table technique used in §4
   (`HAL_DEC_R2_Set_SHM_PARAM`, table `0x45a4c0`) — it was out of scope for R8.
4. Runtime-verify (read-only) that DTS enable bit 6 is actually asserted during DTS playback.
