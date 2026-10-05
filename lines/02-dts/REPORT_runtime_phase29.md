# R29 — Differential trace of codec-private output helpers (MT5889 / MStar AEON SND, `snd_full.bin`)

**Static binary analysis only. No patches, no device changes, no EDID/AUTH/settings/root-cause claims.**
Processor: `aeon:LE:32:default` (Ghidra 12.1.2 custom module). Decode authority: Ghidra-resolved `flow[0]` (my own far-jump arithmetic is *not* authoritative).
Companion report: `REPORT_runtime_phase28.md` (established symmetric output-state slots + equivalent HW-reset/IEC-prep, first divergence in private format-setter callees).

---

## Classification discipline used in this report

- **VERIFIED** — established directly from the decoded ISA / locally-resolved control flow (`flow[0]`).
- **STRONG INFERENCE** — high-confidence reading of *what the code does*, but the runtime truth or MMIO/struct semantics are not independently confirmed (no datasheet, data globals unreadable).
- **UNRESOLVED** — genuinely un-decodable from the extracted binary (address beyond `snd_full.bin` end `0x1C132F`, or data-content we cannot statically read).

Binary bound: `snd_full.bin` = `0x1C1330` bytes → max valid code address `0x1C132F`.
- **In range (decoded this session / prior):** `0xCB50 0xCAB4 0xC49D 0xC5C5 0xCC9B 0xCF86 0x10786E 0x107860 0x107379 0x115693 0x13292 0x25EE1 0xD4C8` and the `0x25_xxxx`/`0x1B_xxxx`/`0x18_xxx`/`0x12_8xxx` globals *as addresses* (their struct *contents* are data, not readable here).
- **Out of range (data pointers, NOT code):** `0x4D17C0` (arg passed to `0x25EE1`), `0x23_xxxx` (arg passed to `0x10786E`). These are pointer values; the funnels are still fully decodable because the *code* lives in range.

> **Correction to prior working assumption:** `0x25EE1` was previously (wrongly) assessed as "beyond the extracted binary." `0x25EE1` = 155 361 bytes — well inside `0x1C1330`. It is **decoded in full this session** (74 instrs). Only the *argument global* `0x4D17C0` it is called with is out of range.

---

## A. Routine map

| Routine | Codec | Size | Role | Callees (VERIFIED `flow[0]`) | In/Out of range |
|---|---|---|---|---|---|
| `0xCB50` | AC3 | 104 | AC3 "set format" private helper | `0x10786E` | IN |
| `0xCAB4` | AC3 | 47 | AC3 "enable/set" private helper | tail→`0x10786E` | IN |
| `0xC49D` | shared | 54 | SPDIF-status / buffer-pos checker (disable path writes `0x854`) | — | IN |
| `0xC5C5` | shared | 109 | Format-code→index status checker (disable path writes `0x854`) | — | IN |
| `0xCC9B` | DTS | 230 | DTS "set format" private helper | `0x25EE1`; tail→`0xD4C8` | IN |
| `0xCF86` | DTS | 299 | DTS "format/status update" private helper | `0x25EE1`; tail→`0xD4C8` | IN |
| `0x10786E` | AC3 funnel | 20 | Thin wrapper: null-guard clear of a global byte, then `0x115693` | `0x107860`, `0x115693` | IN |
| `0x107860` | AC3 leaf | 3 | Null-guarded byte-store clear (`*(r3)=0`) | tail→`0x107379` | IN |
| `0x115693` | AC3 leaf (shared finalizer) | 108 | Buffer/threshold math + **MMIO writes `0xB000_0020`,`0xB000_003E`** + cache flush | `0x10C8EA`,`0x1078A9`,`0xD4C8` | IN |
| `0x25EE1` | **DTS funnel** | 74 | **State-sync / enable-gate**: reads `0xB0000C06`,`0xB0000015`, global`[0x130]`; conditionally calls `0x13292` | `0x13292` | IN |
| `0x13292` | DTS funnel callee | 106 | DMA/buffer-descriptor setup (struct @ `0x252AD0`, size `0x8000`) + converges on shared `0x10786E` | `0x10786E`, `0x10C5B9` | IN |
| `0xD4C8` | shared | — | Cache/DMA flush (`flush_invalidate`+`syncwritebuffer`) | — | IN |

**Funnel topology (the key structural fact):**
- AC3 path: `0xCB50`/`0xCAB4` → **`0x10786E` (direct, unconditional)** → `0x107860` → `0x115693`.
- DTS path: `0xCC9B`/`0xCF86` → **`0x25EE1`** (state-gate) → `0x13292` (DMA/buffer setup) → `0x10786E` → `0x115693`.

Both codecs *eventually* reach the same shared finalizer `0x10786E`→`0x115693`. The DTS path reaches it **transitively and conditionally**; the AC3 path reaches it **directly and unconditionally**.

---

## B. DTS helper analysis (`0xCC9B`, `0xCF86`, `0x25EE1`, `0x13292`)

### `0xCC9B` (DTS "set format", 230 instrs) — VERIFIED
1. Prologue; reads `*(0xB000_0814)` (status), `slli` by 0xa; if bit31 set → **disable path**: `*(0xB000_0854) &= 0xf5ffff`, `*(0xB000_084C)=0`, `+0x2c` cnt++, `+0x10/+0xc=0x2000`, `+0x14=0` (lines ~`0xCCD9`–`0xCD37`). *Note: the DTS disable path writes the SPDIF control reg `0x854`, consistent with the shared disable convention.*
2. Fixed-point sample-rate calc (lines `0xCD39`–`0xCD5C`): `r10=0x25_1194`; `r25 = r23 * 0xa57eb503`; `r26 = mfspr1 0x2808`; `r25 = (r23+r26) >> 0xb` — **identical const/ops to AC3 `0xCB50`**.
3. **Primary DTS output-register write (VERIFIED):** `0xCD6C` `r26 = 0xB000_086C`; `0xCD76` `*(0xB000_086C) = r25 & 0xffff`.
4. Reads `*(0xB000_0814)` (status), `*(0xB000_0818)`; `opcode_2E` MMIO-touch sequences.
5. **DTS funnel call (VERIFIED):** `0xCEC4` `r3 = 0x4D17C0` (out-of-range data global); `0xCECC` `bg.jal 0x25EE1`. If `0x25EE1` returns ≠0, an extra block at `0xCF60` updates global `0x25_2AD0` (`+0x18=0xa` or `+0x20=0x2`, rate config).
6. Tail `0xCE3A` `bg.j 0xD4C8` (cache flush).

### `0xCF86` (DTS "format/status update", 299 instrs) — VERIFIED
1. Reads `*(0xB000_0814) & 0xffff`; reads `r26 = *(r3+0x34)` (DTS state slot); `opcode_2E`.
2. **Format-code → index switch (VERIFIED):** on `r24 = +0x1c(r3)`:
   `0x5888→1`, `0xf400→0xe`, `0xb110→0xd`, `0xee00→0xc`, `0xac44→4`, `0xfa00→2`, `0x7d00→5`, else `3` (lines `0xCFB5`–`0xCFF3`). This is the **same mapping as shared `0xC5C5`**.
3. `0xD097` `*(r3+0x38) = r24` (DTS state field).
4. **Primary DTS output-register write (VERIFIED) — DIFFERENT register from `0xCC9B`:** `0xD09A` `r26 = 0xB000_0870`; `0xD0A1` `*(0xB000_0870) = r24 & 0xffff`. (So `0xCC9B`→`0x86C`, `0xCF86`→`0x870`: **two distinct DTS control registers** within the DTS path.)
5. Reads `*(0xB000_0814)`, `*(0xB000_081C)`, `*(0xB000_0818)`; `opcode_2E` sequences.
6. **DTS funnel call (VERIFIED):** `0xD2C0` `r3 = 0x4D17C0`; `0xD2C8` `bg.jal 0x25EE1`. If returns ≠0, global `0x25_5CA8`/`0x25_2AD0` rate-config block at `0xD2CF`–`0xD340`.
7. Tail `0xD174` `bg.j 0xD4C8`.

### `0x25EE1` (DTS funnel / state-gate, 74 instrs) — VERIFIED (decoded this session)
- `0x25EE4` `r23 = 0xB000_0000`.
- `0x25EF2` `r24 = 0xB000_0C06`; `0x25EF6` `r12 = *(0xB000_0C06)` (**reads a DIFFERENT MMIO page/block than the `0x08xx` output regs** — INFERENCE: link/clock/enable-status block).
- `0x25EF9` `r23 = 0xB000_0015`; `0x25EFC` `r23 = *(0xB000_0015)` (reads low-offset status byte).
- `0x25F10` `r24 = *(0x251194 + 0x130)` = **module-global enable flag `global[0x130]`** (VERIFIED read; flag *meaning* UNRESOLVED).
- Computes `r11 = (bit7 of `*(0xB000_0C06)`) AND (`*(0xB000_0015)` ≥ 0) AND (`global[0x130]` ≠ 0)` — a status/enable predicate (VERIFIED arithmetic; semantics INFERENCE).
- Branches:
  - `0x25F26` if `*(r3+0xbc) == r11` → skip (already synced) `0x25F36`.
  - `0x25F2A` if `r11 != 0` → **`0x25F91`** sub-path (enable-transition): `*(r3+0xb0)=0`, sets `r3=0x252AD0`,`r23=0x1B6098`,`r4=0x10F`,`r5=0x96000`,`r6=8`,`r7=0x100`,`r8=4`, then `0x25FBB` `bg.jal 0x13292`, then `0x25FBF` `bg.j 0x25F2D`.
  - else → `0x25F2D` (writes `*(r3+0x58)=1`, `*(r3+0xbc)=r11`; state sync only).
- `0x25F36`+ : maintains `*(r3+0x54)` (bit6 mask of `*(0xB000_0C06)`), `*(r3+0xb8)=r11`, and returns `r3` = 0, or **1 if a table entry at `0x18CCC + (*(r3+0x94)+0x12)*4 == 1` AND `global[0x130] != 0`** (VERIFIED control flow; table/data UNRESOLVED).

### `0x13292` (DTS funnel callee / DMA-buffer setup, 106 instrs) — VERIFIED
- Takes the globals/constants from `0x25EE1` (`r3=0x252AD0`,`r4=0x10F`,`r5=0x96000`,`r6=8`,`r7=0x100`,`r8=4`).
- Fixed-point/dma math (`muls`/`divu`/`sub`); writes a **buffer descriptor** into struct @ `r10=r3=0x252AD0`: fields `+0x0..+0x40`, including **`+0x30 = 0x8000`** (buffer size) and `+0x34 = 0x100` (VERIFIED writes — purpose INFERENCE: DMA/scatter-gather descriptor for the DTS output path).
- Calls the **shared finalizer** `0x10786E` at `0x13338` and `0x133A7` (with `r3 = 0x128xxx` in-range globals), and `0x10C5B9` at `0x13376`.

**Takeaway for B:** the DTS private helpers each (1) write a *distinct* DTS control register (`0x86C` / `0x870`), (2) route through the `0x25EE1` state-gate that consults status MMIO `0xB0000C06`/`0xB0000015` + `global[0x130]`, and (3) only through that gate reach `0x13292` (DMA/buffer setup) → shared `0x10786E` finalizer.

---

## C. AC3 comparison (`0xCB50`, `0xCAB4`, `0x10786E`, `0x107860`, `0x115693`)

### `0xCB50` (AC3 "set format", 104 instrs) — VERIFIED
1. `r5 = *(0x25_1194 + 0x11a8)` (global sampling-rate-ish field). If `r5 ≤ 0x317f` → **disable**: `*(0xB000_0854) &= 0xf5ffff`, `*(0xB000_084C)=0`, `+0x2c`, `+0x10/+0xc=0xd72000`, `+0x14=0` (`0xCC08`+).
2. **Format-code → SPDIF control-word switch (VERIFIED):** on `r4` (=`+0x1c(r3)`, stored at `0xCB6F`):
   `0x5888→0x1000`, `0xac44→0x4000`, `0xfa00→0x2000`, `0x7d00→0x5000`, else `0x3000`; **extended** (for `r4>0x7d00`): `0xf400→0xe000`, `0x7700→0x0`, `0xb110→0xd000`, `0xee00→0xc000`, else `0x3000` (`0xCC49`+).
   *Note the AC3 switch also has DTS-format cases, but it treats them as **SPDIF control words**, whereas the DTS helper treats the same codes as **status indices** — a semantic divergence (see §D).*
3. Fixed-point sample-rate calc (identical consts/ops to DTS: `0xa57eb503`, `mfspr1 0x2808`, `>>0xb`).
4. `0xCBEC` `bg.jal 0x10786E` (**unconditional** funnel call).
5. **Primary AC3 output-register write (VERIFIED):** `0xCBF4` `r10 = r11 | r10` where `r11 = *(0xB000_0854)&0xfff | 0xa44`; `0xCBF7` `r23 = 0xB000_0854`; `0xCBFB` `*(0xB000_0854) = r10`. → **writes SPDIF CTRL `0x854`** (INFERENCE: `0x854`=SPDIF vs `0x86C/0x870`=DTS/HDMI from address layout + which codec writes which).

### `0xCAB4` (AC3 "enable/set", 47 instrs) — VERIFIED
- `r5 = *(0x25_1194 + 0x14)`. If `r5 ≤ 0x317f` → disable: `*(0xB000_0854) &= 0x5fffff`, `*(0xB000_0850)=0`, `+0x30`, `+0x10/+0xc=0xdd8000`, `+0x14=0`.
- else (`0xCAFF`): `*(0xB000_0854) &= 0x5fffff`, `| 0xa090`, write; `*(0x25_1194+0x11c)=1`; fixed-point calc; `0xCB4C` `bg.j 0x10786E` (**unconditional tail-call** funnel).

### `0x10786E` (AC3 funnel, 20 instrs) — VERIFIED
- `r5=r3`; `r6=r1+0xc`; `r3 = 0x23_xxxx` (out-of-range global arg); `r4 = 0x400`; `0x107860` (null-guard clear `*(r3)`); `0x115693`; `r1+=0x20`; `jr r9`. → **trivial wrapper**; its only real effect (when `r3` in range) is a byte-clear, then it dispatches to the shared finalizer `0x115693`.

### `0x107860` (AC3 leaf, 3 instrs) — VERIFIED
- `if (r3==0) goto 0x107866; *(0x0(r3)) = 0` (null-guarded byte clear); `goto 0x107379`.

### `0x115693` (shared finalizer, 108 instrs) — VERIFIED
- `mfspr/mtspr r0,rX,0x11` (cache/DC control, same SPR class as `0xD4C8`); buffer/threshold arithmetic; calls `0x10C8EA`, `0x1078A9`, `0xD4C8`.
- **Finalizer MMIO writes (VERIFIED):** `0x11579F` `r23=0xB000_0000`; `0x1157A3` `r24=0xB000_003E`; `0x1157A9` `*(0xB000_003E) = r10` (halfword); `0x1157AC` `r23=0xB000_0020`; `0x1157B1` `*(0xB000_0020) = 0x2` (halfword). → writes `0xB000_0020 = 0x2` (INFERENCE: enable) and `0xB000_003E = <computed>` (INFERENCE: a value/threshold). These are in the `0xB000_00xx` block, **distinct from the `0x08xx` output regs**.

**Takeaway for C:** the AC3 private helpers write **SPDIF `0x854`** and call the finalizer `0x10786E`→`0x115693` **unconditionally** on every invocation.

---

## D. Differential table (functional op → AC3 vs DTS)

| Functional op | AC3 (`0xCB50`/`0xCAB4`) | DTS (`0xCC9B`/`0xCF86`) | Classification |
|---|---|---|---|
| Status MMIO read | `0xB000_0814` (via shared `0xC49D`/`0xC5C5`) | `0xB000_0814` (same, via shared) | **VERIFIED identical** (shared helpers) |
| Format-code usage | `0xCB50` maps code → **SPDIF control word** (`0x5888→0x1000`, `0xac44→0x4000`, `0xfa00→0x2000`, `0x7d00→0x5000`, else `0x3000`; +`0xf400/0xb110/0xee00→0xe000/0xd000/0xc000`) | `0xCF86` maps code → **status index** (`0x5888→1`, `0xf400→0xe`, `0xb110→0xd`, `0xee00→0xc`, `0xac44→4`, `0xfa00→2`, `0x7d00→5`, else `3`); `0xCC9B` uses fixed-point calc, no format switch | **VERIFIED constants overlap; SEMANTICS DIVERGE** (control word vs status index) |
| Sample-rate (fixed-point) | `r10=0x25_1194`; `muls 0xa57eb503`; `mfspr1 0x2808`; `srai 0xb` | identical | **VERIFIED identical** |
| **Physical output register written** | **`0xB000_0854` (SPDIF CTRL)** | **`0xB000_086C` (`0xCC9B`) / `0xB000_0870` (`0xCF86`)** — two *different* DTS regs | **VERIFIED: DIFFERENT registers** |
| Output-state struct writes | `+0x30` cnt (`0xCAB4`), `+0x10/+0xc/+0x14`, `+0x2c` (`0xCB50`) | `+0x30` (DTS slot, `0xCC9B`), `+0x38` (`0xCF86`), `+0x10/+0xc/+0x14` | **VERIFIED equivalent fields** (R28: `+0x30/0x34` are the DTS slots) |
| **Funnel (post-write helper)** | `0x10786E` — **called UNCONDITIONALLY** (`0xCBEC` jal / `0xCB4C` tail) | `0x25EE1` — state-gate, called unconditionally but **forwards to `0x10786E` only on enable-transition** (`0x25F91`→`0x13292`→`0x10786E`) | **VERIFIED structural divergence** (AC3 unconditional vs DTS gated) |
| **Finalizer `0x115693` MMIO writes** (`0xB000_0020=0x2`, `0xB000_003E=val`) | **ALWAYS executed** (AC3 always reaches `0x10786E`) | **ONLY if `0x25EE1` gate passes** (`r11≠0` AND `*(r3+0xbc)≠r11`, with `r11` depending on `0xB0000C06`/`0xB0000015`/`global[0x130]`) | **VERIFIED control-flow divergence** (runtime truth UNRESOLVED) |
| DMA/buffer descriptor setup | none in AC3 funnel path | `0x13292` writes buffer desc @ `0x252AD0` (`+0x30=0x8000`, `+0x34=0x100`) before finalizer | **VERIFIED DTS-only** (purpose INFERENCE) |
| Extra status MMIO read | none | `0x25EE1` reads `0xB000_0C06`, `0xB000_0015` | **VERIFIED DTS-only** (semantics UNRESOLVED) |
| Cache flush | reached via `0x115693`→`0xD4C8` | tail `0xD4C8` + via `0x115693`→`0xD4C8` | **VERIFIED equivalent** |
| Disable/error path | mask `0x854 &= 0xf5ffff` (`0xCB50`) / `0x5fffff` (`0xCAB4`); clear `0x84C`/`0x850` | mask `0x854 &= 0xf5ffff`; clear `0x84C` (shared-style) | **VERIFIED similar convention** |

---

## E. Return / error analysis

- **Shared helpers `0xC49D`/`0xC5C5`:** return `r3 = 1` if SPDIF status (`0xB000_0814` low bits) matches the expected pattern / buffer pos valid, else `0` (disable path). They gate on SPDIF *status* registers regardless of codec — used by both paths.
- **AC3 `0xCB50`/`0xCAB4`:** no meaningful return value used by caller for gating the finalizer — `0x10786E` is invoked **unconditionally**. The AC3 path does not condition its finalizer on any runtime status.
- **DTS `0xCC9B`/`0xCF86`:** call `0x25EE1` **unconditionally**, then branch on `0x25EE1`'s return: `0xCED0` `if (r3≠0) goto 0xCF60` (extra global rate-config); `0xD2CC` `if (r3≠0) goto 0xD31F`. So the DTS helpers *do* act on `0x25EE1`'s return, but the **finalizer `0x10786E`→`0x115693` (the `0xB000_0020`/`0x003E` writes) is reached only inside `0x25EE1`'s enable-transition sub-path** (`0x25F91`), i.e. only when `r11≠0` AND `*(r3+0xbc)≠r11`, where `r11` is the predicate over `0xB000_0C06`/`0xB000_0015`/`global[0x130]`.
- **Error/disable convention:** both codecs share the "mask `0xB000_0854` + clear `0x84C`/`0x850` + bump a frame counter + reset buffer fields" pattern — VERIFIED symmetric, consistent with R28's negative result (DTS not disabled by its slot).

**Net:** the *error/disable* machinery is symmetric, but the *enable/finalize* path is asymmetric: AC3 finalizes unconditionally; DTS finalizes only through a gate whose inputs are data-dependent (status MMIO + a global enable flag) and therefore **cannot be confirmed to pass at runtime from static analysis**.

---

## F. First output-relevant divergence

The first point where AC3 and DTS configure **different physical output state** is the **primary control-register write**:

- AC3 `0xCB50` @ `0xCBF7`/`0xCBFB` → **`*(0xB000_0854) = …`** (SPDIF CTRL).
- DTS `0xCC9B` @ `0xCD6C`/`0xCD76` → **`*(0xB000_086C) = …`**; DTS `0xCF86` @ `0xD09A`/`0xD0A1` → **`*(0xB000_0870) = …`**.

From that point the paths also diverge in *how the finalizer is reached*:

- AC3: `0xCBEC` `jal 0x10786E` → `0x10786E` → `0x115693` → writes `0xB000_0020=0x2` and `0xB000_003E` — **every enable-path call** (the `r5≤0x317f` disable path returns early without the funnel, same convention as DTS).
- DTS: `0xCECC`/`0xD2C8` `jal 0x25EE1` → inside `0x25EE1`, the finalizer is reached only via the `0x25F91` enable-transition branch (→ `0x13292` → `0x10786E` → `0x115693`). If the gate's predicate (`0xB000_0C06` bit7 ∧ `0xB000_0015 ≥ 0` ∧ `global[0x130]≠0`) is false, the DTS helper **skips the `0xB000_0020`/`0x003E` finalizer entirely**, while AC3 never skips it.

This is the **first and most output-relevant divergence**: different target registers *and* an unconditional-vs-gated finalizer. Everything before it (status read, format constants, fixed-point timing) is either shared or arithmetically identical.

---

## G. Final classification

### → **R29-B** (STRONG INFERENCE of a codec-specific divergence; not fully verified; specific items UNRESOLVED)

**Why not R29-A (equivalent state):** The DTS private helpers do **not** configure the same functional state as their AC3 counterparts. VERIFIED divergences:
1. Different physical output registers: AC3→`0xB000_0854` (SPDIF); DTS→`0xB000_086C` *and* `0xB000_0870` (two distinct DTS control regs).
2. Format-code semantics differ: AC3 maps format→SPDIF control word; DTS maps format→status index (same constants, different use).
3. Funnel structure diverges: AC3→`0x10786E` (trivial, **unconditional**); DTS→`0x25EE1` (state-gate) → `0x13292` (DMA/buffer setup) → `0x10786E` (**gated**).
4. DTS adds a state-gate that reads extra status MMIO (`0xB000_0C06`, `0xB000_0015`) and a module-global enable flag (`global[0x130]`), and a DMA/buffer descriptor setup (`0x13292`, struct @ `0x252AD0`) that the AC3 path lacks.

**Why not R29-C (divergence is cosmetic):** The divergences are functional (different MMIO targets, gated vs unconditional finalizer, extra enable predicate), not cosmetic renaming.

**Why not R29-D (insufficient evidence):** `0x25EE1` and `0x13292` *are* decoded (correcting the earlier out-of-range error); the code-level divergence is established by ISA/control flow. R29-D would only hold if the DTS funnel were genuinely unreachable in the binary — it is not.

**UNRESOLVED items preventing a stronger (proven root-cause) classification:**
- Data globals `0x4D17C0` (arg to `0x25EE1`) and `0x23_xxxx` (arg to `0x10786E`) are **out of range** — their struct contents are not readable.
- MMIO semantics: the roles of `0xB000_0854` (SPDIF) vs `0xB000_086C`/`0xB000_0870` (DTS/HDMI), and of `0xB000_0C06`/`0xB000_0015`/`0xB000_0020`/`0xB000_003E`, are **INFERENCE** (address-layout + which codec writes which), not datasheet-confirmed.
- The `0x25EE1` predicate's runtime truth is **data-dependent** (status MMIO + `global[0x130]`); static analysis cannot confirm whether the gate passes on real DTS hardware.
- Leaves `0x10C5B9`, `0x1078A9`, `0x10C8EA`, `0x107379` are in range but not decoded this session.

**Bottom line (answers the R29 key question):** The DTS-private helpers do **not** configure the same functional state as the AC3 helpers. There is a verified, codec-specific divergence — different output-control registers, a gated (vs unconditional) finalizer, and an extra enable/state-gate + DMA-buffer-setup layer on the DTS side. This is a **strong inference** for *why AC3 may produce a valid physical digital stream while DTS may not*, but it is **not a proven root cause**: the gating inputs and MMIO semantics remain unresolved, and no device was modified or tested.

---

*Generated by static AEON disassembly (Ghidra `flow[0]`-authoritative) of `snd_full.bin`. No firmware was patched, flashed, or executed; no EDID/AUTH/settings were altered. See `REPORT_runtime_phase28.md` for the preceding symmetric-slot / HW-reset result.*
