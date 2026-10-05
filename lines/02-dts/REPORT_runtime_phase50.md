# R50 — DTS state slots `r10+0x30` / `r10+0x34`: production & use (narrow trace)

**Scope (per user):** narrow trace of the DTS slots `r10+0x30`/`r10+0x34` from the format  
dispatch at `0x1F70C–0x1F718` (`r26 ∈ {9,8,5,2}`) to the `0x1F75C` DTS output-config block.  
R48/R49 and `0x854` are **not** re-opened. Ghidra `aeon:LE:32:default` slaspec is ground truth  
(`-noanalysis` project `r27_work/ghidra_r27f` / `sndr27`). No patch, no device, no full-image  
MMIO survey, no Python-decoder reliance.

**Critical correction (integrity note):** the pending `R50_CheckR14.java` (which asked whether  
callees preserve `r14=0x4E0000` across `0x1F601→0x1F6DA`) is **moot** — `r14` is reassigned at  
`0x1F683` (`bt.mov r14,r3`) from `mem[0x1c(r10)]` = a snapshot of `mem[0x8(r10)]` taken at `0x1F605`,  
*before* the intervening callees. The gate at `0x1F6DA` therefore tests **`mem[0x8(r10)]`**, not the  
transient `0x4E0000`. The script was not run.

---


## 1. Control-flow trace (Ghidra ground truth)

Function entry `0x1F575` (prologue `bn.addi r1,r1,-0x38`; `r10 = r3` arg @`0x1F5A1`).

```
0x1F595  bn.lh  r11,0x0(r25)        ; r25=0xB000_000A → r11 = half@0xB000_000A
0x1F598  bn.lwz r12,0x0(r23)        ; r23=0xB000_0814 → r12 = word@0xB000_0814
0x1F59B  bn.lbz r23,0x0(r24)        ; r24=0xB000_0C05 → r23 = byte@0xB000_0C05
0x1F59E  bn.lbz r24,0x0(r24)        ; r24 = byte@0xB000_0C05
0x1F5A1  bt.mov r10,r3              ; r10 = fn argument (DSP-side struct)
0x1F5A6  bn.extbz r26,r23           ; r26 = byte@0xB000_0C05  (CODEC/FORMAT dispatch value)
0x1F5A9  bg.beq r24,r23,0x0001f701  ; r24==r23 (both=byte@0xC05) → TAKEN → 0x1F701
...
0x1F701  bg.opcode_2E r25
0x1F705  bn.sw 0x20(r3),r25         ; mem[0x20(r3)] = op result
0x1F708  bg.opcode_2E r26
0x1F70C  bg.beqi r26,0x9,0x0001f7de ; *** Codec 9 (DTS) ***
0x1F710  bg.beqi r26,0x8,0x0001f7e9 ; Codec 8
0x1F714  bg.beqi r26,0x5,0x0001f7f4 ; Codec 5
0x1F718  bg.bnei r26,0x2,0x0001f91f ; r26==2 → fall-through; else 0x1F91F
0x1F71C  bn.sw 0x28(r3),r26         ; mem[0x28(r3)] = codec (stream field)
0x1F721  bg.j 0x0001f5b3
```

**Codec 9 path (`r26==9` ⇒ `0x1F7DE`):**

```
0x1F7DE  bt.movi r23,0x5
0x1F7E0  bn.sw 0x28(r3),r23         ; mem[0x28(r3)] = 5  (stream/format field for DTS)
0x1F7E5  bg.j 0x0001f5b3           ; → loop to 0x1F5B3
```

The Codec-9 dispatch writes **only the stream field `mem[0x28(r3)]` (=5)** and loops to `0x1F5B3`.  
It does **NOT** populate `r10+0x30`/`r10+0x34` directly. DTS-slot population happens downstream,  
behind the gate.

**Loop body `0x1F5B3` (reached by every codec incl. 9):**

```
0x1F5B3  bn.lwz r23,0x2c(r10)
0x1F5B6  bg.beq r23,r24,0x0001f647  ; if mem[0x2c]==r24(=5 for codec9) → 0x1F647 (skip re-init)
0x1F5BA  bn.lwz r23,0x28(r10)
0x1F5BD  bn.lwz r24,0x20(r10)
0x1F5C0  bn.sw 0x2c(r10),r23        ; mem[0x2c]=mem[0x28]  (state latch)
0x1F5D1  bn.lwz r14,0xc(r10)        ; decoder ctx fields
0x1F5D4  bn.lwz r15,0x8(r10)        ; r15 = mem[0x8(r10)]  ← GATE SOURCE
0x1F5D7  bn.lwz r13,0x10(r10)
0x1F5FB  bt.mov r3,r15              ; r3 = mem[0x8(r10)]
0x1F5FD  bg.jal 0x000b544a          ; 0xB544A(mem[0x8],mem[0xc],mem[0x10])
0x1F601  bg.movhi r14,0x4e          ; r14 = 0x4E0000 (transient)
0x1F605  bn.sw 0x1c(r10),r3         ; mem[0x1c]=mem[0x8(r10)]  (GATE SNAPSHOT)
0x1F608  bg.addi r3,r14,0x6ad4      ; r3 = 0x4E6AD4
0x1F60C  bg.jal 0x0000ca42          ; CA42(0x4E6AD4)
0x1F618  bn.sw 0x30(r10),r0         ; *** CLEAR DTS slot 0x30 *** (unconditional in init)
0x1F61B  bn.sw 0x34(r10),r0         ; *** CLEAR DTS slot 0x34 *** (unconditional in init)
0x1F61E  bg.jal 0x0010786e          ; log
0x1F622  bt.mov r3,r10
0x1F624  bg.jal 0x0001f0a0          ; 0x1F0A0(r10)
0x1F62E  bg.bles r23,r24,0x0001f65e ; if mem[0x18]<=mem[0x0] → 0x1F65E (else return 0)
```

**Gate prelude `0x1F65E` → `0x1F6DA`:**

```
0x1F65E  bn.sw 0x0(r10),r0
0x1F66B  bg.addi r13,r10,0x3050
0x1F66F  bn.lwz r3,0x1c(r10)        ; r3 = GATE SNAPSHOT = mem[0x8(r10)]
0x1F67B  bg.jal 0x000b6d8f          ; 0xB6D8F (runs AFTER snapshot — cannot change gate)
0x1F67F  bg.andi r11,r11,0x200      ; r11 = 0xB6D8F result & 0x200  (status bit9)
0x1F683  bt.mov r14,r3              ; r14 = mem[0x8(r10)]  (GATE PREDICATE)
0x1F685  bn.beqi r11,0x0,0x0001f6da ; if r11==0 → 0x1F6DA (skip telemetry)
0x1F6DA  bg.bnei r14,0x0,0x0001f77b ; *** GATE: if mem[0x8(r10)]!=0 → SKIP DTS block → 0x1F77B ***
```

**DTS block `0x1F6DE` (reached only if gate fell through, i.e. `mem[0x8(r10)]==0`):**

```
0x1F6DE  bn.movhi_2 r23,0xff
0x1F6E5  bn.and r12,r12,r23
0x1F6E8  bg.opcode_2E r11          ; r11 = op result (unmodelled DSP op)
0x1F6EC  bn.beqi r11,0x1,0x0001f725 ; if r11==1 → DTS slot block
0x1F6EF  bn.sw 0x30(r10),r0         ; else CLEAR 0x30
0x1F6F6  bg.beqi r12,0x1,0x0001f78b ; alt: if r12==1 → 0x34 path
0x1F6FA  bn.sw 0x34(r10),r0         ; else CLEAR 0x34
0x1F6FD  bg.j 0x0001f632           ; return
```

**DTS slot block `0x1F725` → populate → `0x1F75C` (R50 target):**

```
0x1F725  bn.lwz r23,0x30(r10)       ; read DTS slot 0x30
0x1F72C  bg.beqi r23,0x0,0x0001f8fc ; if 0x30==0 → populate path
0x1F730  bg.movhi r14,0x4e
0x1F736  bg.jal 0x0000c49d          ; 0xC49D (state normalizer — R49 finding)
0x1F754  bg.jal 0x0000cc9b          ; 0xCC9B (DTS helper → 0x25EE1 chain, R46)
0x1F75C  bg.jal 0x0000c49d          ; *** 0x1F75C DTS output-config block (state normalizer) ***
...
0x1F8FC  bg.addi r3,r14,0x6ad4
0x1F900  bg.jal 0x0000ca42
0x1F904  bg.jal 0x0000c55e
0x1F910  bn.sw 0x30(r10),r11        ; *** POPULATE 0x30 = r11 ***
0x1F91B  bg.j 0x0001f730           ; re-enter (now 0x30 != 0)
```

**Alt `0x34` path `0x1F78B` (if `r12==1`):**

```
0x1F78B  bn.lwz r23,0x34(r10)       ; read DTS slot 0x34
0x1F792  bg.beqi r23,0x0,0x0001f86f
0x1F7BA  bg.jal 0x0000cf86          ; 0xCF86 (DTS helper → 0x25EE1 chain, R46)
0x1F891  bn.sw 0x34(r10),r12        ; *** POPULATE 0x34 = r12 ***
```

---


## 2. Per-write table — DTS slots `r10+0x30` / `r10+0x34`

All writes **within the `0x1F575` dispatch function** (verified: no other `0x30`/`0x34` writers in  
`0x1F55C–0x1F920`; the `0x1F356`/`0x1F4BF` clears are in a *different* function).

| Slot | Addr      | Instruction           | Value              | Upstream condition                                                                                               |
| ---- | --------- | --------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| 0x30 | `0x1F618` | `bn.sw 0x30(r10),r0`  | **0 (CLEAR)**      | Codec-9 init path; unconditional after `0xB544A`+`CA42`. Resets slot every entry.                                |
| 0x30 | `0x1F6EF` | `bn.sw 0x30(r10),r0`  | **0 (CLEAR)**      | DTS block reached (gate fell through) **but** `opcode_2E r11 != 1`.                                              |
| 0x30 | `0x1F910` | `bn.sw 0x30(r10),r11` | **r11 (POPULATE)** | Populate path `0x1F8FC`, taken iff gate fell through **AND** `r11==1` (`0x1F6EC`) **AND** `0x30==0` (`0x1F72C`). |
| 0x34 | `0x1F61B` | `bn.sw 0x34(r10),r0`  | **0 (CLEAR)**      | Codec-9 init path; unconditional. Resets slot every entry.                                                       |
| 0x34 | `0x1F6FA` | `bn.sw 0x34(r10),r0`  | **0 (CLEAR)**      | DTS block reached **but** `opcode_2E r12 != 1`.                                                                  |
| 0x34 | `0x1F891` | `bn.sw 0x34(r10),r12` | **r12 (POPULATE)** | Alt path `0x1F78B`, taken iff gate fell through **AND** `r12==1` (`0x1F6F6`).                                    |

**Writers of `0x30`/`0x34` elsewhere in the image** (Section-2 full scan, informational only):  
`0x1F356`/`0x1F4BF` (clear, different fn), `0x1F618`/`0x1F61B`/`0x1F6EF`/`0x1F6FA`/`0x1F891`/`0x1F910`  
(this fn), plus many unrelated functions (e.g. `0x1334C`/`0x13356`, `0x135E0`, `0x13A87`, `0x1A89F`,  
`0x21E01`/`0x21E07`, `0x25D51`/`0x25D73`, …) — these are **different struct bases**; the  
`r10-relative` DTS slots in the codec-9 dispatch are uniquely `0x1F575`'s.

---


## 3. The gate — what it actually tests

| Element                              | Value                             | Source                                       | Statically determinable? |
| ------------------------------------ | --------------------------------- | -------------------------------------------- | ------------------------ |
| Gate register at `0x1F6DA`           | `r14`                             | `bt.mov r14,r3` @`0x1F683`                   | —                        |
| `r14` content                        | `mem[0x1c(r10)]`                  | snapshot @`0x1F605`                          | —                        |
| `mem[0x1c(r10)]` content             | `mem[0x8(r10)]` (as of `0x1F5D4`) | `bn.sw 0x1c(r10),r3` where `r3←r15←mem[0x8]` | **NO** — caller-provided |
| Secondary predicate `r11` @`0x1F6EC` | `opcode_2E r11` result            | `0x1F6E8` (unmodelled DSP op)                | **NO** — runtime         |
| Secondary predicate `r12` @`0x1F6F6` | `opcode_2E r12` result            | `0x1F6F2` (unmodelled DSP op)                | **NO** — runtime         |

`mem[0x8(r10)]` is a **decoder-context field** of the struct passed to `0x1F575`: it is read at  
`0x1F5D4`, passed as the argument to `0xB544A` (`0x1F5FD`), and copied to `mem[0x1c(r10)]`. **The  
function never writes `0x8(r10)` or `0x1c(r10)` except the snapshot at `0x1F605`.** Therefore its value  
is fixed by the caller and is **not derivable from this function alone**.

This is why the `r14=0x4E0000` callee-preservation question (`R50_CheckR14.java`) is moot: `r14` is  
overwritten from memory at `0x1F683`, so whatever `CA42`/`log`/`0x1F0A0` do to `r14` is discarded  
before the gate.

---

## 4. Comparison with AC3 slot `r10+0x486c` (only where necessary)

- **In `0x1F575`, `0x486c`/`0x4870` are NEVER written** (Section-2 scan: zero hits for those offsets  
  in `0x1F55C–0x1F920`). The AC3 state slot is populated in a **separate** AC3 helper (R28/R49:  
  `0xCB50`/`0xCAB4`/`0xC49D`), not in the codec-9 dispatch.
- **Asymmetry (structural):** AC3 slot population lives in AC3's funnel that is reached  
  **unconditionally** (`0xCB50/0xCAB4 → 0x10786E (unconditional) → 0x115693`, R46). The DTS slot in  
  `0x1F575` is **unconditionally CLEARED** at init (`0x1F618`/`0x1F61B`) and only **conditionally  
  repopulated** behind the `mem[0x8(r10)]` gate + `opcode_2E` predicates.
- The two codecs use **separate slots** (`0x30/0x34` vs `0x486c/0x4870`) and **separate functions**; the  
  Codec-9 dispatch is the DTS-slot owner.

---

## 5. Can we prove R50-A or R50-B? — **R50-C**

**R50-A** (Codec 9 reliably populates DTS slot & enters `0x1F75C`): **NOT provable.** The slot is  
cleared unconditionally at `0x1F618` and only repopulated (`0x1F910`) when:  
(a) gate `0x1F6DA` falls through → requires `mem[0x8(r10)] == 0`, and  
(b) `opcode_2E r11 == 1` at `0x1F6EC`.  
Both (a) and (b) are runtime/struct/op-dependent — not statically determinable.

**R50-B** (Codec 9 fails to populate / DTS path skipped): **NOT provable as a definite failure.**  
The gate SKIPS the DTS block iff `mem[0x8(r10)] != 0`; we cannot prove `mem[0x8(r10)]` is  
*always* non-zero (a caller could pass 0, e.g. on a first/unconfigured entry).

**R50-C — Slot production cannot be proven statically.** This is the honest classification.


### Structural finding (the durable result)

1. The `r14=0x4E0000` hypothesis is **refuted**; the real `0x1F6DA` gate tests `mem[0x8(r10)]`.
2. The DTS slots `0x30`/`0x34` are **unconditionally cleared** in the Codec-9 init (`0x1F618`/`0x1F61B`)  
   and **only conditionally repopulated** (`0x1F910`/`0x1F891`). Populating them *and* entering the  
   `0x1F75C` DTS output-config block requires the `mem[0x8(r10)]==0` gate to fall through **and** an  
   `opcode_2E` predicate (`r11`/`r12`) to be 1.
3. `0x1F75C` is reached **only** via that gated, repopulated path; it calls `0xC49D` (state normalizer,  
   R49-consistent) and `0xCC9B` (DTS helper → `0x25EE1` chain, R46). The path `0x30 != 0 → 0x1F75C →
   0x25EE1` holds **conditional** on the gate, exactly as the user's trace expected — but the gate value  
   is not statically fixed.
4. **Lean, not proof:** because the slot is cleared unconditionally and repopulated only under a gate,  
   the *default* state for Codec 9 is "DTS slot = 0 / DTS block skipped" (R50-B-shaped), with  
   population being the conditional exception (R50-A-shaped). Statically we cannot pick between them.

---

## 6. What would resolve C → A/B (runtime, read-only, no patch/device-mod)

- Capture `mem[0x8(r10)]` (the gate predicate) at the `0x1F6DA` test. `r10` is the DSP-side struct  
  pointer; if it resolves into the `/proc/utopia_mdb/audio` SRAM mirror (16-bit-truncated cells, per  
  SKILL §8.2), the cell is observable. If `mem[0x8(r10)] != 0` in all DTS captures, R50-B is confirmed  
  (gate skips → slot stays 0 → `0x1F75C` skipped). If it is 0 and `opcode_2E r11` yields 1, R50-A.
- Alternatively, trace the caller of `0x1F575` to see what it initializes at offset `0x8` of its struct  
  argument (targeted `jal 0x1F575` scan — allowed, not a full-image MMIO survey).
-

---

## 7. Status vs prior phases

- R48/R49: **not re-opened** (per user). R49 finding (`0xC49D` = state normalizer, not codec selector;  
  `0x854` bits 17|19 not the AC3/DTS divergence) is corroborated here: `0xC49D` appears at `0x1F736`/  
  `0x1F75C`/`0x1F8DD` as a normalizer inside the DTS path, never as a codec selector.
- R46: DTS funnel `0xCC9B/0xCF86 → 0x25EE1 → 0x13292` already characterised; this R50 stops at `0x1F75C`  
  (downstream `0x25EE1` hardware-arm is out of scope for R50, per user).
- **No patch written. No device modified. No EDID/AUTH/CheckHashkey re-opened.**
