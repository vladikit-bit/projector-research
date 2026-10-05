# SPACE BUNNY — ARCHIVE/RAW CORRECTIONS

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device, ADB, playback, breakpoint, memory read, patch, or flash  
**TCL/T615:** not used  
**Preservation:** this is a new correction artifact; earlier reports were not edited

## Purpose

A deeper archive-mining pass found two places where the preceding continuation report was too strong or arithmetically wrong. This document records the correction against the current raw `utpa2k_stock.ko` and authoritative DEC listing.

## 1. Correction: MS12 SND copy length

The previous continuation report stated that the MS12 loader length was `0x176930`. That value was caused by overlooking the conditional high-half write.

The exact current instructions are:

```text
46fb2c  movw    r8, #0x3f48
46fb3c  movt    r8, #0x17       -> r8 = 0x00173F48
46fb54  cmp     r1, #4
46fb5c  movweq  r8, #0x6930     -> low half = 0x6930
46fb64  movteq  r8, #0x1b       -> high half = 0x001B
```

Therefore:

```text
standard selector path: r8 = 0x00173F48
MS12 selector path:     r8 = 0x001B6930
```

The embedded symbol sizes independently confirm the relationship:

```text
mst_snd_r2             0x17E948
mst_snd_r2_MS12V22    0x1C1330

0x17E948 - 0x173F48 = 0xAA00
0x1C1330 - 0x1B6930 = 0xAA00
```

The loader copy sequence is therefore consistent for both profiles:

```text
first copy:  [0x00000, 0x0A000)
clear:       [0x0AA00, 0x700000)
second copy: [0x0AA00, 0x0AA00 + r8)
```

For both profiles, `0x0AA00 + r8` equals the corresponding full SND symbol size. The earlier “MS12 length discrepancy” was an analysis error, not a firmware discrepancy.

The skipped window `[0xA000,0xAA00)` is structurally aligned with:

```text
g_virSndR2shm = MsOS_PA2KSEG1(B + 0x70A000)
```

because the SND image begins at `B+0x700000`. This is a strong host/SND-layout correspondence; the archive/code still does not state the design intent, so it remains an inference rather than a documented contract.

## 2. Correction: F8/FC have additional first-word writers

The earlier statement “F8/FC have exactly two writers” was accurate only for the **pointer stores** in `F_45DE0`. It was too broad for the pointed-to buffers.

`F_45DE0` still performs:

```text
045e17  [D+0xF8] = *(arg4+0)
045e21  memset(F8, 0, 0x200)
045e2f  [D+0xFC] = *(arg4+4)
045e39  memset(FC, 0, 0x200)
```

However, the following block, identified in the archive as the `F_45E52` path, directly writes the first header words through those pointers:

```text
045efb  lwz     r24,0xf8(r11)
045eff  lwz     r23,0xfc(r11)
045f14  sw      [F8 + index*4], h[0x16c]
045f20  sw      [FC + index*4], h[0x16e]
045f2c  sw      [F8+4], h[0x170]
045f34  sw      [FC+4], h[0x172]
```

The same block then reloads F8/FC and performs the byte/word transformation loop at `0x45F53..0x45FBB`, including the `0x80C`/`0x6E2C` arithmetic and halfword stores.

Corrected statement:

```text
F_45DE0:
    establishes the two 0x200-byte buffer pointers

F_45E52 path:
    writes header/transformed words through those pointers

F_4618E:
    performs the later local fill/transform path
```

The earlier absence result for `F8/FC -> R+0x174` is unchanged: no direct pointer/payload copy into `R+0x174` was found.

## 3. Archive evidence boundary

The archive-mining pass searched the rendered transcript chunks, raw DB extracts, raw JSONL copies, and subagent trajectories. It found:

- no `V_in`, `U`, `[B0+0x1414C]`, or full `A108` semantics;
- no `R+0x174` evidence beyond later user/task text;
- no proof of `F8/FC -> F_028AFE/F_014F63` or physical transport;
- raw A108 load bytes at `0x4127D`, later independently verified against `dec_full.bin`;
- raw F8/FC and SND-loader bytes that were present but had not been fully interpreted in the historical summary.

Thus the archive is useful primary evidence for raw bytes and host loader structure, but it does not replace the later DEC/AEON analysis.

## 4. Other archive corrections carried forward

The same pass confirmed or corrected:

- `F_411A6` has six direct calls in its raw block, not only the parser/output pair: `0x45E52` at `0x411F5`, `0x41259`, `0x4126D`, plus `0x410A3` at `0x41238`, `0x45FD1` at `0x41283`, and `0x4618E` at `0x41289`.
- `0x130D25` is memset-like across the inspected `(ptr, 0, length)` call sites; this is stronger than the old “zero-fill-shaped” wording.
- The four direct `0x45D07` call sites are `0x45FC9`, `0x460EF`, `0x4616A`, and `0x46186`; the historical “exactly one caller” statement is rejected.
- `0xDD3404` is the file offset of `mst_snd_r2_MS12V22`, not a DEC runtime address.

## 5. Corrected matrix entries

| Item | Corrected status |
|---|---|
| MS12 SND `r8` | `0x1B6930`; equals symbol size minus `0xAA00` |
| SND copy-length discrepancy | **Rejected as an analysis error** |
| SND skipped window | `[0xA000,0xAA00)` aligns structurally with `g_virSndR2shm` mapping |
| F8/FC pointer writers | Two stores in `F_45DE0` |
| F8/FC first-word writers | Additional writes in the `F_45E52` block |
| F8/FC → `R+0x174` | Still not found |
| Archive proof of V_in/U/R+0x174 | None |

## Final status

The corrected static graph still ends at:

```text
host configuration
 -> mapped B
 -> DEC/SND image placement
 -> host shared-memory/control programming
 -> local DEC parser/builder/output
 -> SND ring production
 -> hardware-facing mailbox boundary
```

The MS12 length anomaly is removed from the unresolved list. The remaining high-value unknowns are the AEON epilogue semantics, V_in/U provenance, F8/FC-to-R+0x174 identity, ring-instance aliasing, external hardware consumer identity, and physical SPDIF transport.

No previous artifact was modified.
