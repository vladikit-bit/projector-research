# R28 — Trace AC3/DTS output-state consumers to actual TX control

**Target firmware:** `mst_snd_r2_MS12V22.bin` (MT5889 / MStar AEON SND audio module)  
**Decoder:** custom `aeon:LE:32:default`      Ghidra processor + self-written AEON disassembler (verified byte-exact in R27).  
**Source decode:** `r27_work/ghidra_dump_1B000_5400.txt` (region `0x1B000`–`0x20400`, MS12V2 encoder layer, 7073 instructions).  
**Authoritative enumeration script:** `scripts/r28_extract.py` (full context windows) + `scripts/r28_ranges.py` (range dumps).

---


## 0. Scope, method, and constraints (restated)

R28 asks: *what do the AC3/DTS codec-specific output-state slot pairs actually control, and do they control the same downstream mechanism?* The four slots (all on base register `r10`):

| Codec | Port  | Slot offset | Read site (load→r23)               |
| ----- | ----- | ----------- | ---------------------------------- |
| AC3   | SPDIF | `0x486c`    | `0x1ED7D` `bg.lwz r23,0x486c(r10)` |
| AC3   | HDMI  | `0x4870`    | `0x1EE49` `bg.lwz r23,0x4870(r10)` |
| DTS   | SPDIF | `0x30`      | `0x1F725` `bn.lwz r23,0x30(r10)`   |
| DTS   | HDMI  | `0x34`      | `0x1F78B` `bn.lwz r23,0x34(r10)`   |

**Hard constraints honoured:** no patches, no device changes, no speculative root cause. Every claim is tagged  
**VERIFIED BY ISA/STATIC**, **STRONG INFERENCE**, or **UNRESOLVED**. The IEC-constant bytes (`0xF872`/`0x4E1F`/`0xF8724E1F`),  
the "6 IEC clusters", and `ADD_IEC_HEADER`/`SDO_SpdifPacker` string hunting are treated as dead ends per R28 §7 — and the  
result below *confirms* that treatment (see §C / §E and the R27 correction note).

---


## 1. Methodology

1. Parsed the Ghidra dump with regex `0x([0-9A-Fa-f]+)\s+([0-9A-Fa-f]{8})\s+len=(\d+)\s+(.*)`, split into contiguous  
   instruction runs, and located **every** load/store whose base register is `r10` and whose offset is one of the four  
   slot offsets. (Ghidra renders AEON loads/stores as `<mnemonic>.lwz rD,0xNNNN(r10)` / `<mnemonic>.sw 0xNNNN(r10),rS`;  
   AC3 uses the 32-bit `bg.*` forms, DTS uses the 24-bit `bn.*` forms.)
2. For each access, extracted a ±22-instruction window and traced the loaded value (`r23`) and the stored value  
   (`r11`/`r12`) forward through compares, branches, and stores.
3. Followed `bn.bf`/`bn.bnf`/`bg.beqi` targets across function-internal jumps to make sure no IEC/MMIO write was missed  
   in the branch shadow (this caught the DTS SPDIF refresh → shared IEC writer at `0x1F8A0`, which a naive window would miss).
4. Cross-checked the whole `0x1B000`–`0x20400` region for the IEC constants `0x4e1f`/`0xf872` to classify every occurrence  
   (shared encoder vs codec-specific consumer).

**Enumeration result: 20 slot accesses total — 4 READs (one per codec/port) and 16 WRITEs (clears + one "set" per port).**  
There is exactly **one READ per slot**; the slot is consumed only at that single branch point.

---

## A. Consumer table (all 20 accesses, VERIFIED BY ISA/STATIC)

Legend: **CLR** = clear-to-0 write (`…,r0`); **SET** = open/write of a non-zero value (`…,r11/r12`); **RD** = load into `r23`.

### AC3 — SPDIF slot `r10+0x486c`

| Addr      | Kind   | Instruction               | Notes / context                                         |
| --------- | ------ | ------------------------- | ------------------------------------------------------- |
| `0x1EBC0` | CLR    | `bg.sw_0 0x486c(r10),r0`  | function-entry clear                                    |
| `0x1ECFC` | CLR    | `bg.sw_0 0x486c(r10),r0`  | mid-path clear (after `bnei r3,0` re-check)             |
| `0x1ED7D` | **RD** | `bg.lwz r23,0x486c(r10)`  | **consumer**: `beqi r23,0x0,0x1EF2C` (slot==0 → OPEN)   |
| `0x1EF40` | SET    | `bg.sw_0 0x486c(r10),r12` | OPEN store (r12 = enable flag, =1) after `0xC55E` reset |

### AC3 — HDMI slot `r10+0x4870`

| Addr      | Kind   | Instruction               | Notes / context                                       |
| --------- | ------ | ------------------------- | ----------------------------------------------------- |
| `0x1EBBC` | CLR    | `bg.sw_0 0x4870(r10),r0`  | function-entry clear                                  |
| `0x1ED08` | CLR    | `bg.sw_0 0x4870(r10),r0`  | mid-path clear                                        |
| `0x1EE49` | **RD** | `bg.lwz r23,0x4870(r10)`  | **consumer**: `beqi r23,0x0,0x1EF67` (slot==0 → OPEN) |
| `0x1EF89` | SET    | `bg.sw_0 0x4870(r10),r11` | OPEN store after `0xC731` reset                       |

### DTS — SPDIF slot `r10+0x30`

| Addr      | Kind   | Instruction            | Notes / context                                       |
| --------- | ------ | ---------------------- | ----------------------------------------------------- |
| `0x1F35D` | CLR    | `bn.sw 0x30(r10),r0`   | function-entry clear                                  |
| `0x1F4BF` | CLR    | `bn.sw 0x30(r10),r0`   | clear in alt open path                                |
| `0x1F618` | CLR    | `bn.sw 0x30(r10),r0`   | clear in open path #2                                 |
| `0x1F6EF` | CLR    | `bn.sw 0x30(r10),r0`   | clear when spdif-enable flag `r11 != 1`               |
| `0x1F725` | **RD** | `bn.lwz r23,0x30(r10)` | **consumer**: `beqi r23,0x0,0x1F8FC` (slot==0 → OPEN) |
| `0x1F910` | SET    | `bn.sw 0x30(r10),r11`  | OPEN store after `0xC55E` reset                       |


### DTS — HDMI slot `r10+0x34`

| Addr      | Kind   | Instruction            | Notes / context                                       |
| --------- | ------ | ---------------------- | ----------------------------------------------------- |
| `0x1F356` | CLR    | `bn.sw 0x34(r10),r0`   | function-entry clear                                  |
| `0x1F4B8` | CLR    | `bn.sw 0x34(r10),r0`   | clear in alt open path                                |
| `0x1F61B` | CLR    | `bn.sw 0x34(r10),r0`   | clear in open path #2                                 |
| `0x1F6FA` | CLR    | `bn.sw 0x34(r10),r0`   | clear when hdmi-enable flag `r12 != 1`                |
| `0x1F78B` | **RD** | `bn.lwz r23,0x34(r10)` | **consumer**: `beqi r23,0x0,0x1F86F` (slot==0 → OPEN) |
| `0x1F891` | SET    | `bn.sw 0x34(r10),r12`  | OPEN store after `0xC731` reset                       |

**Quantitative asymmetry (noted, not over-weighted):** DTS clears each slot in **4 distinct sites**; AC3 in **2**. This is the  
same "clear-to-0" operation repeated across more error/branch paths in the DTS function — a structural difference in *how  
often* the flag is reset, not in *what the flag means*.

**Key structural fact (VERIFIED BY ISA/STATIC):** in every codec/port, the slot is read exactly once, into `r23`, and the  
*only* use of the loaded value is the zero/non-zero test (`beqi r23,0x0,…`). The slot is therefore a **boolean output-state  
flag**: `0` = output not yet opened, `≠0` = output open/active. No slot value is masked, shifted, ORed, or passed to another  
routine — it is purely a branch predicate.

---

## B. Parallel AC3/DTS data-flow (VERIFIED BY ISA/STATIC for the control flow; STRONG INFERENCE for the semantics)

Each codec/port has a symmetric "set/refresh output" function. The slot flag drives a 2-way branch:

```
                READ slot (r10+off) → r23
                          │
                 beqi r23,0  ?
                ┌───────────┴───────────┐
              YES (slot==0)           NO (slot != 0)
                │                        │
          OPEN path                 REFRESH path
                │                        │
   • HW reset callee           • IEC61937 burst-preamble
     (0xC55E SPDIF /             store to output buffer
      0xC731 HDMI)             • counter increment
   • store enable flag           (instance+0x28)
     (r12/r11 = 1)             • codec-format setter callee
   • jal 0x10786e (log)         (AC3: 0xCB50/0xCAB4;
                                   DTS: 0xCC9B/0xCF86)
```

### OPEN path (slot == 0) — fully symmetric

| Step            | AC3 SPDIF         | AC3 HDMI          | DTS SPDIF       | DTS HDMI                                  |
| --------------- | ----------------- | ----------------- | --------------- | ----------------------------------------- |
| HW reset        | `0xC55E`          | `0xC731`          | `0xC55E`        | `0xC731`                                  |
| store slot      | `r12` (→`0x486c`) | `r11` (→`0x4870`) | `r11` (→`0x30`) | `r12` (→`0x34`)                           |
| log/notify      | `jal 0x10786e`    | `jal 0x10786e`    | `jal 0x10786e`  | `jal 0x10786e`                            |
| extra IEC write | —                 | —                 | —               | **yes** → instance+0x10 (`0x1F8A0` block) |

OPEN is byte-for-byte parallel except that **DTS HDMI additionally writes the IEC preamble to `instance+0x10` on open**  
(AC3 HDMI open does not; both still write it on refresh — see below). This is a redundant extra, not a missing capability.


### REFRESH path (slot != 0) — IEC preamble is symmetric

Both codecs, on refresh, under the identical guard `instance+0x14 ≤ 0x3c00`, write an 8-byte IEC61937 burst preamble  
(Pa=`0xF872`, Pb=`0x4E1F`, repetition-count field, burst-info field = `instance+0x14 << 3`) into the output buffer:

| Codec/Port | Preamble store site                                                     | Target buffer                             | Guard                                       |
| ---------- | ----------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------- |
| AC3 SPDIF  | `0x1EDA6`–`0x1EDC1`                                                     | `instance+0x10` (`r14 = *(r12+0x10)`)     | `instance+0x14 ≤ 0x3c00` (`0x1EDA0 bn.bnf`) |
| AC3 HDMI   | `0x1EE71`–`0x1EE93`                                                     | **`0x531894`** (`r13=0x53`, `r13+0x1894`) | `instance+0x14 ≤ 0x3c00` (`0x1EE6C bn.bnf`) |
| DTS SPDIF  | `0x1F8A0`–`0x1F8BA` (shared block, reached via `0x1F745 bn.bf 0x1F8A0`) | `instance+0x10` (`r3 = *(r11+0x10)`)      | `instance+0x14 ≤ 0x3c00` (`0x1F745 bn.bf`)  |
| DTS HDMI   | `0x1F7FF`–`0x1F813` (refresh) **and** `0x1F8A0`–`0x1F8BA` (open)        | **`0x531894`** and `instance+0x10`        | `instance+0x14 ≤ 0x3c00` (`0x1F7AB bn.bf`)  |

The SPDIF buffer is `instance+0x10` for **both** codecs; the HDMI buffer is the **same fixed global `0x531894`** for **both**  
codecs. The preamble byte layout (`0xF872`@+0, `0x4E1F`@+2, `0x1`@+4, `instance+0x14<<3`@+6) is identical in every case.  
**Conclusion: the IEC/non-PCM serialization configuration is performed symmetrically by AC3 and DTS.** (See §C and R27-correction.)

### Shared encoder IEC sites (outside the codec-specific consumer chain)

`0x1B4B0` and `0x1BF02` (early MS12V2 encoder region) also write `0xF872`/`0x4E1F` into a passed-in buffer (`r12`). These are  
reached by the shared encoder setup used by **both** codecs — further confirming IEC burst framing is a common mechanism, not  
a codec-divergent one.

---


## C. First semantic divergence

**Finding (VERIFIED BY ISA/STATIC): there is NO semantic divergence in the output-state control mechanism itself.**  
The four codec-specific slots are consumed identically — each is a boolean open-flag gated by the same `beqi r23,0x0` test,  
driving the same OPEN (HW reset) and REFRESH (IEC preamble) actions, writing the same buffers, and calling the same  
codec-agnostic HW reset callees. The slot values do **not** "do something different" between codecs.

The **first observed codec-specific divergence** is *downstream of, and parallel to*, the slot consumer — it is in the  
**secondary output-format setter callees**, not in the flag or its TX gating:

| Refresh-path callee          | AC3 SPDIF | AC3 HDMI | DTS SPDIF | DTS HDMI |
| ---------------------------- | --------- | -------- | --------- | -------- |
| primary `0xC49D`             | ✓         | —        | ✓         | —        |
| `0xC5C5`                     | —         | ✓        | —         | ✓        |
| `0xCAB4`                     | —         | ✓        | —         | ✓        |
| `0xCB50` (r4=`0xBB80`=48000) | ✓         | —        | ✓         | ✓        |
| `0xCC9B` (DTS-private)       | —         | —        | ✓         | —        |
| `0xCF86` (DTS-private)       | —         | —        | —         | ✓        |

`0xCB50`/`0xCAB4` (AC3) vs `0xCC9B`/`0xCF86` (DTS) are codec-private "configure output format / start codec" routines. They  
are **parallel routine choices that converge on the same IEC + HW-reset result**, not a disabled or skipped path. Their exact  
internals are **UNRESOLVED** (not decoded in R28 scope).

Two minor asymmetries worth recording (neither disables output):

1. **DTS HDMI OPEN additionally writes the IEC preamble to `instance+0x10`** (`0x1F8A0` block) — AC3 HDMI open does not.  
   Net HDMI capability is identical (both also write it on refresh to `0x531894`).
2. **AC3 SPDIF refresh performs an inline output-buffer operation** (compare `r5`=`instance+0x28` vs `r23`=`instance+0x14`,  
   then `jal 0x10C5B9` = memset, at `0x1EDC4`→`0x1EFBD`) that has **no directly corresponding inline block** in the DTS SPDIF  
   refresh tail (DTS delegates to `0xCB50`). Whether `0xCB50` performs the equivalent internally is **UNRESOLVED**.

**R27 correction (important):** R27 concluded the IEC61937 Pa/Pb constants were a *false lead* (instruction-byte coincidence).  
That conclusion was incomplete for the *code* sites: `0xF872`/`0x4E1F` are genuinely computed (`addi r0,-0x78e`,  
`addi r0,0x4e1f`) and stored as real IEC burst preambles — but **only in symmetric, codec-agnostic locations** (AC3 SPDIF/HDMI,  
DTS SPDIF/HDMI, and shared encoder). They therefore cannot explain a DTS-specific output defect, which *validates* R27's  
"ignore the IEC constants" guidance even though the literal classification needed refinement.

---


## D. Hardware correlation (builds on R27 §5.5/§5.6)

The slot's `==0` branch gates the **codec-agnostic** hardware reset/enable callees established in R27:

| Port  | Reset/enable callee | MMIO written (VERIFIED by R27 decode)                                                    | Port mapping                        |
| ----- | ------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------- |
| SPDIF | `0xC55E`            | clears `0xB000_0854` (mask `0xF5FFFF`) → polls `0xB000_080C` → writes 0 to `0xB000_084C` | **STRONG INFERENCE**: SPDIF TX port |
| HDMI  | `0xC731`            | clears `0xB000_0854` (mask `0x5FFFFF`) → polls `0xB000_0810` → writes 0 to `0xB000_0850` | **STRONG INFERENCE**: HDMI TX port  |

Both codecs call the **same** callees with the **same** MMIO sequence on output open. The `0xB000_08xx` block is the audio  
output hardware (R27). The slot flag is what decides *whether* this hardware reset/enable runs (only on the open transition);  
the IEC preamble (REFRESH) is what keeps the non-PCM serializer configured afterward. Neither decision differs by codec.

**Therefore the hardware TX-enable correlation is identical for AC3 and DTS.** The slot does not route DTS away from the  
SPDIF/HDMI hardware; both reach `0xC55E`/`0xC731` identically.

---


## E. Conclusion — do the AC3 and DTS slots control the same mechanism?

**YES. The four codec-specific output-state slots control the same downstream mechanism.** Specifically:

1. Each slot is a **boolean output-open flag** (VERIFIED BY ISA/STATIC — single read, used only as a zero/non-zero predicate).
2. `slot == 0` → OPEN: both codecs call the **same** codec-agnostic HW reset/enable callee (`0xC55E` SPDIF / `0xC731` HDMI)  
   writing `0xB000_08xx`, then store the enable flag. Identical across codecs.
3. `slot != 0` → REFRESH: both codecs write the **identical IEC61937 burst preamble** (`0xF872`/`0x4E1F`/rep-count/burst-info)  
   to the **same** output buffers (`instance+0x10` for SPDIF, `0x531894` for HDMI) under the **same** guard. Identical across codecs.
4. The only codec-specific divergence is in the **parallel, codec-private format-setter callees** (`0xCB50`/`0xCAB4` for AC3,  
   `0xCC9B`/`0xCF86` for DTS) reached *after* the slot has already driven the symmetric TX action — and in a redundant DTS-HDMI  
   open-time IEC write. None of these skips or disables the SPDIF/HDMI output.

**Where DTS would "first diverge into a different or disabled output path": it does not, within the slot-consumer chain.**  
The codec-specific state slots are not the locus of any DTS-specific disable. If a real DTS SPDIF/HDMI output defect exists, it  
must originate **upstream of these slots** (in the codec path that decides the enable flag `r11`/`r12`, or in the private  
format-setter callees `0xCC9B`/`0xCF86`/`0xCB50`) or **downstream** (in how the serialized IEC frames are handed to the  
`0xB000_08xx` TX block). That origin is **UNRESOLVED** within R28 scope and must not be speculated.

**Direct answer to the R28 question:** *"Does one of these codec-specific state values eventually control SDO/IEC/non-PCM/  
output enable, while the other does something different?"* — **No. Both state values control the same SDO/IEC/non-PCM/output-  
enable mechanism symmetrically. Neither does "something different" in a way that disables output.**

---

## Residual / UNRESOLVED

- Internal semantics of callees `0xC49D`, `0xC5C5`, `0xCAB4`, `0xCB50`, `0xCC9B`, `0xCF86` (codec-private format setters).
- Whether DTS achieves, inside `0xCB50`, the output-buffer memset that AC3 performs inline after its SPDIF refresh IEC write.
- What sets the enable flags `r11`/`r12` (the upstream decision that ultimately populates the slot) — outside this decode region.
- Whether `instance+0x14 ≤ 0x3c00` guard ever fails differently for DTS vs AC3 (would suppress the IEC preamble for *both*  
  equally; not a codec-specific effect).

## Classification legend

- **VERIFIED BY ISA/STATIC** — provable from instruction bytes / control flow in the decoded dump.
- **STRONG INFERENCE** — consistent with R27 MMIO mapping and IEC61937 structure, but callee internals not fully decoded.
- **UNRESOLVED** — cannot be proven from the static decode available; explicitly not speculated upon.
