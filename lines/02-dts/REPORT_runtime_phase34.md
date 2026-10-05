# R34 — Search for the External Consumer of `0xA868` / `0xA86C`

**Module:** `snd_full.bin` (= `mst_snd_r2_MS12V22.bin`) — MStar/MedTek AEON **SND R2** core
**Prior:** R29–R33 (R33-B: `0xA868`/`0xA86C` are write-only, no firmware reader inside `snd_full.bin`)
**This phase:** R34 — locate the **other end of the handoff**.
**Constraints honoured:** no patch, no device change, no runtime experiment, no EDID/ARC/topology/settings/AUTH change.
**Not repeated:** `0x25EE1`, `r11`, `0x0C06`, `divu`, `0x109595` (R29–R33). The "six-slot array" hypothesis is **not** revisited (disproven in R33).
**Deliverable date:** 2026-09-09

---

## Headline classification

> ### R34 = **R34-C** — `0xA868`/`0xA86C` are **confirmed as DSP shared-memory (SHM) state**, but the
> ### **consumer identity is NOT recovered.**

The window *class* was established from host-side source: the SND R2 shared memory is mapped by
`HAL_SND_R2_Set_SHM_PARAM` as **`HAL_AUDIO_GetDspMadBaseAddr() + 0x7000A000`** via `MsOS_PA2KSEG1`, cached in
`g_virSndR2shm`. No image in the workspace (SND R2, codec R2, ARM fw, Utopia HAL asm) was found to **read**
offset `0x868` / `0x86C` of that window.

---

# VERIFIED

Facts directly demonstrated by disassembly / artifacts present in the workspace.

### V1 — The `0xA000` window is the SND R2 **shared memory** (host-side proof)
From `spdif_audio_investigation/r6_out/HAL_SND_R2_Set_SHM_PARAM.asm` (`libutopia.so`, real exported symbol,
audit records it at `0x003A342C`):
```
0x00459e04  movw  r5, #0          <<< REL sym=g_virSndR2shm
0x00459e0c  movt  r5, #0
0x00459e14  ldr   r6, [r5]                       ; r6 = g_virSndR2shm
0x00459e20  cmp   r6, #0
0x00459e24  bne   #0x459e54                      ; already mapped -> skip
0x00459e28  mov   r0, #2
0x00459e2c  bl    HAL_AUDIO_GetDspMadBaseAddr    ; physical DSP MAD base
0x00459e30  movw  r2, #0xa000                    ; <<< 0xA000
0x00459e34  movt  r2, #0x70                      ; r2 = 0x7000A000
0x00459e38  adds  r0, r0, r2                     ; base + 0x7000A000
0x00459e40  bl    MsOS_PA2KSEG1                  ; phys -> KSEG1 (uncached) mapping
0x00459e4c  str   r0, [r5]                       ; g_virSndR2shm = mapped SHM
```
> **`g_virSndR2shm = MsOS_PA2KSEG1( HAL_AUDIO_GetDspMadBaseAddr() + 0x7000A000 )`**
> The low 16 bits of `0x7000A000` are exactly `0xA000`, matching the R2-side base used by `snd_full.bin`.
> **VERIFIED.**

### V2 — A separate DEC-R2 SHM exists
`g_virDecR2shm` is referenced in the same HAL asm set — i.e. the DEC (codec) R2 has its **own** shared-memory
window, distinct from `g_virSndR2shm`. **VERIFIED.**

### V3 — Host `HAL_SND_R2_Set_SHM_PARAM` does **not** touch `0x868`/`0x86C`
Its SHM accesses (via `r6 = g_virSndR2shm`) use offsets:
`0x104, 0x10c, 0x110, 0x114, 0x118, 0x11c, 0x120, 0x124, 0x128, 0x12c, 0x134, 0x13c, 0x140, 0x144, 0x148,
0x14c, 0x154, 0x158, 0x15c, 0x164, 0x168, 0x16c, 0x170, 0x174, 0x17c, 0x180, 0x184, 0x188, 0x18c, 0x190, 0x194`
— a **param block at 0x104–0x194**, driven by a jump table over param IDs `0x59…0xCF`.
**None is `0x868` or `0x86C`.** **VERIFIED.**

### V4 — The codec R2 accesses **MMIO** `0xB000_0868`/`0xB000_086C`, not SHM `0xA868`/`0xA86C`
`mst_codec_r2*.bin` contain AEON-identical sequences:
```
c2f60001  bg.movhi r23,0xb000
cb170868  bg.ori  r?,r23,0x868      -> 0xB000_0868
cb17086c  bg.ori  r?,r23,0x86c      -> 0xB000_086C
cb170864  bg.ori  r?,r23,0x864      -> 0xB000_0864
```
Scan result: for **both** codec images, the number of `0xA000`-immediate sites linked (within 52 B) to a
`0x868`/`0x86C` access is **0**. **VERIFIED** — the codec R2 is not a reader of the SHM slots.

### V5 — The cache-flush helper `0xD4C8` is generic cache maintenance, called on the exact slot
`0xD4C8(r3 = addr, r4 = len)`:
```
0xD4C8  beqi r4,0 -> return
0xD4CC  mfspr r27,r0,0x11        ; save cache-control SPR
0xD4D2  and  r23,r27,-0x7 ; mtspr r0,r23,0x11
        … loop: bg.flush_invalidate 0x0/0x40/0x80/0xc0(r3|r24|r25|r26) …
0xD549  bg.syncwritebuffer
0xD54D  mtspr r0,r27,0x11        ; restore
0xD551  jr   r9
```
Call sites on the DTS path:
- `0xCF86`: `0xD15E` store → `0xD16C addi r3,r3,-0x5798` (= `0x10000-0x5798` = **`0xA868`**), `r4 = 4`, `j 0xD4C8`.
- `0xCC9B`: `0xCE26` store → `0xCE32 addi r3,r3,-0x5794` (= **`0xA86C`**), `r4 = 4`, `j 0xD4C8`.

So the flush is issued **specifically over the slot just written**. It performs
**flush+invalidate + `syncwritebuffer`** — i.e. it makes the store visible outside the local cache.
It is **generic** (also called from `0x115693`, `0x13292`-adjacent paths). **VERIFIED.**

### V6 — Only the two DTS helpers produce these slots
Whole-image result from R33 (re-confirmed): the only sites building base `0xA000` and then touching
`0x868`/`0x86C` are `0xCE26`, `0xCE72`, `0xCF48` (→ `0xA86C`, in `0xCC9B`) and `0xD15E` (→ `0xA868`, in `0xCF86`).
**No AC3-path function writes them.** **VERIFIED.**

### V7 — Artifact observation (prior capture files, not a new experiment)
`spdif_audio_investigation/r11_out/dm3_dts.txt` / `dm3_ac3.txt` contain:
```
DM[0xa865] = 0x000000 ; DM[0xa866] = 0x000080 ; DM[0xa867] = 0x000000
DM[0xa868] = 0x000000     <- our slot
DM[0xa869] = 0x000000 ; DM[0xa86a] = 0x000080 ; DM[0xa86b] = 0x000000
```
Both AC3 and DTS captures read `0`. **Reported with caveats**: capture timing/phase unknown, and whether
`DM[cell]` index == the R2 data address for this window is not independently proven. Not used as proof of
anything on its own.

---

# STRONG INFERENCE

Conclusions supported by several evidence points, not directly proven.

### S1 — `0xA868`/`0xA86C` are **DSP shared-memory slots** (class E2), not MMIO registers
Evidence: (V1) the host maps `DspMadBase + 0x7000A000` as **SND R2 SHM** via a *memory* mapping
(`MsOS_PA2KSEG1`, uncached KSEG1) — not an `ioremap` of a register block; (V2) a sibling `g_virDecR2shm`
exists; (V5) the R2 publishes with **flush+invalidate + syncwritebuffer**, the canonical way to expose a
store to another agent; (V6) the slots live at offset `0x868` inside that mapped window.
⇒ They are **shared memory visible to another agent**, consumed outside `snd_full.bin`.

### S2 — The publication is a **cross-agent handoff**, but its direction/partner is unproven
The flush covers exactly the written slot, so the intent is clearly "make this visible". However nothing in the
available artifacts shows *who* reads it. Candidate partners: host CPU (via `g_virSndR2shm`), the DEC/codec R2,
or a hardware engine that shares the memory. (V4) rules out the codec R2 for *these offsets*; (V3) rules out
`HAL_SND_R2_Set_SHM_PARAM`. Other host functions remain possible but were not found.

### S3 — The values are DTS-path-specific
Only `0xCC9B`/`0xCF86` (the DTS private helpers) write them; the AC3 path does not. Combined with R32/R33
(the value is `*(0x253364) + *(0x251194+0x114) + *(0x251194+0x118) + (+0x14)/(((+0x18)*(+0x20))*0x30)`),
the two slots carry a **DTS-computed parameter pair**, one per helper (`0xCF86 → 0xA868`, `0xCC9B → 0xA86C`).

---

# UNRESOLVED

- **Consumer identity.** No reader of `g_virSndR2shm + 0x868` / `+ 0x86C` (or the R2-side equivalent) was found in
  any available image or disassembly.
- **Whether the consumer is host CPU, another DSP core, or hardware.**
- **The SHM layout** — the offset `0x868` region's field definition (the HAL only documents the `0x104–0x194`
  param block).
- **Whether `0x868` and `0x86C` together form a descriptor** or are two independent parameters.
- **Any link to SDO / IEC61937 / DTS packer / SPDIF-HDMI TX** — none established.
- **The exact semantics of the flush length `r4 = 4`** (unit not proven).
- **DMA (E3) is NOT established.** No DMA descriptor, physical-address base, or engine programming tied to
  `0x868`/`0x86C` was found. Per instruction, this is **not** called DMA.

---

# Phase results summary

| Phase | Result |
|-------|--------|
| **A — Inventory** | `snd_full.bin` == `mst_snd_r2_MS12V22.bin` (1,839,920). Other candidates: `mst_snd_r2.bin` (1,567,048), `mst_codec_r2.bin` (2,736,372), `mst_codec_r2_MS12V22.bin` (1,982,492), `armfw.bin` (1,048,576), `utpa2k.ko` (several backups), `libutopia.so` (×2 projects), `libmi3.so`, `audio.primary.mt5889.so`, plus HAL asm (`HAL_SND_R2_Set_SHM_PARAM`, `HAL_DEC_R2_Set_SHM_PARAM`, `HAL_AUDIO_SPDIF_ApplySetting`, `HAL_AUDIO_DigitalTx_ApplySetting`, `HAL_AUDIO_SPDIF_SetMode/SetOutputType/Tx_SetNonPCM`). |
| **B — Cross-image A868/A86C** | Codec R2 hits are **MMIO `0xB000_0868/0x86C`** (V4) — **0** links to a `0xA000` base. `armfw.bin`: 1 loose `0x868` candidate, 0 `0x86C`, 0 `0xA000`. No literal `0x0000A868/0xA86C` dword in any image. **No external reader found.** |
| **C — Flush protocol** | `0xD4C8` = generic flush+invalidate+`syncwritebuffer` over `[r3, len)`; called with the exact slot address (`0xA868` / `0xA86C`). Confirms publication intent; does **not** by itself prove DMA. |
| **D — Kernel/Utopia** | SHM base proven (V1/V2). `HAL_SND_R2_Set_SHM_PARAM` uses offsets `0x104–0x194` only (V3). `HAL_DEC_R2_Set/Get_SHM_PARAM` is called from `HAL_AUDIO_SPDIF_ApplySetting` and `HAL_AUDIO_DigitalTx_ApplySetting` — but for the **DEC** R2 SHM / param range, not `0x868`. |
| **E — Classify window** | **E2 (shared memory / other core) supported.** E1 (MMIO) not supported for these offsets. E3 (DMA) not established. E4 not needed. |
| **F — Trace to DTS output** | **Not possible** — no consumer. No SDO/IEC61937/TX link established; no DTS-specific constants found on any consumer side. |
| **G — Cross-check producers** | Both producers are the DTS helpers; AC3 does not write them (V6). Publish sequence identical (store → flush same address, len 4). Purpose: publish a DTS-computed parameter pair into SND R2 SHM. |

---

# Critical chain

```text
0xCF86 ──┐
         ├─ compute: *(0x253364) + *(0x251194+0x114) + *(0x251194+0x118)
         │           + [ (+0x14) / (((+0x18)*(+0x20))*0x30) ]
         │
0xCC9B ──┘
         ↓
0xCF86 → STORE 0xA868      (SND R2 shared memory, base = DspMadBase + 0x7000A000)
0xCC9B → STORE 0xA86C
         ↓
0xD4C8  flush+invalidate + syncwritebuffer over that exact slot
         ↓
[ FIRST VERIFIED EXTERNAL CONSUMER ]  ← *** NOT IDENTIFIED ***
         ↓
[ next stage ]                        ← unknown
         ↓
[ SDO / IEC61937 / TX ? ]             ← unknown
```
**The unknown links are deliberately left empty.** They are not filled with assumptions.

---

# Final classification — R34-C

- **R34-A — NO.** No consumer found; no path to DTS SDO/IEC61937/TX proven.
- **R34-B — NO.** No *specific* external consumer was identified (only the window class).
- **R34-C — YES.** `0xA868`/`0xA86C` are confirmed as **shared / hardware-facing state** (specifically: SND R2
  **shared memory**, published with cache maintenance), but **consumer identity is not recovered**.
- **R34-D — NO.** The window *was* classified (DSP shared memory) from host-side source.

### Exactly what artifact is needed next
1. **Full `libutopia.so` analysis** (complete ARM disassembly or source) searching for
   `ldr … [g_virSndR2shm, #0x868]` / `#0x86C` — the current workspace only has ~14 HAL asm fragments.
   (`ghidra_proj_new/libutopia.so` and `ghidra_proj_sdo/libutopia.so` exist and could be queried.)
2. **The SND R2 SHM layout/header** (the struct definition covering offset `0x868`) — would name the field and
   its consumer.
3. **`utpa2k.ko` / `dtv_driver.ko` audio paths** — to see whether the kernel driver reads the SND R2 SHM at
   those offsets.
4. Optionally, the **R2-side linker map / symbol table** for `snd_full.bin` (would name `0xA868`/`0xA86C`).

Until one of these is available, the correct statement is:

> **`UNRESOLVED: external consumer not available in current evidence set`** — with `0xA868`/`0xA86C` established
> as SND-R2 shared-memory slots published via cache-flush handoff.

---

## Appendix — Commands / artifacts
- Inventory: `find . -iname "*mst_*" -o -iname "*_r2*" -o -iname "*.bin"` (+ `*.ko`, `*.so`).
- Codec scan: 4-byte window scan for `… A0 00` (base) linked to `… 08 68 / 08 6C` (offsets).
- Host mapping: `spdif_audio_investigation/r6_out/HAL_SND_R2_Set_SHM_PARAM.asm` lines 3–22.
- Host SHM offsets: `[r6, #0x…]` extract from the same file (V3).
- Flush helper: `aeon_validate/r29_work/decoded_CF86.txt` (`0xD4C8` body; call sites `0xD16C/0xD174`, `0xCE32/0xCE3A`).
- Capture observation: `spdif_audio_investigation/r11_out/dm3_dts.txt` / `dm3_ac3.txt` (~line 43197).
