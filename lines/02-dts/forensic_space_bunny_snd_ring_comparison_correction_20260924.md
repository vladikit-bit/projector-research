# SPACE BUNNY — SND RING COMPARISON CORRECTION

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no device/ADB, patch, flash, or module changes  
**TCL/T615:** not used  
**Preservation:** new correction artifact only

## Scope

A deep standard/MS12 SND comparison produced a valid ring-aliasing result but an incorrect host-loader length result. This document separates the two and preserves the corrected version.

## 1. Rejected claim: MS12 copies only `0x176930` bytes

The current `utpa2k_stock.ko` bytes at the selector branch are:

```text
46fb2c  movw    r8, #0x3f48
46fb3c  movt    r8, #0x17
46fb54  cmp     r1, #4
46fb5c  movweq  r8, #0x6930
46fb64  movteq  r8, #0x1b
```

The `ldrb` between the compare and the conditional moves does not change ARM condition flags. Therefore, when selector `4` is active:

```text
r8 = 0x001B6930
```

not `0x00176930`.

The correct relation is:

```text
mst_snd_r2             0x17E948 - 0x173F48 = 0xAA00
mst_snd_r2_MS12V22    0x1C1330 - 0x1B6930 = 0xAA00
```

Both loader paths copy the complete symbol after the reserved `0xA000..0xAA00` host-SHM window:

```text
0xAA00 + r8 = symbol size
```

The claimed MS12 `0x40000` shortfall and the claimed `0x37748` non-zero omitted tail are rejected as consequences of the missed `movteq`.

## 2. Confirmed ring-manager geometry

The SND image raw bytes independently confirm:

```text
base        = 0x00D72000
limit       = 0x00DD8000
span        = 0x00066000
read cursor = 0x00D72000 on reset
write cursor= 0x00D72000 on reset
fill        = 0
```

The relation is exact:

```text
0xDD8000 - 0xD72000 = 0x66000
```

The previous `0x60000` span was a decode/read error. The initializer also clears `0x66000` bytes at `0xD72000`.

## 3. Confirmed MS12 AC3/DTS ring aliasing

The MS12 SND image constructs the same manager root for both paths:

```text
AC3:
0x1EB59  movhi   r3,0x4E0000
0x1EB74  addi    r3,r3,0x6AD4
0x1EBA4  jal     0xCA42

DTS:
0x1F4A5  movhi   r11,0x4E0000
0x1F4AC  addi    r3,r11,0x6AD4
0x1F4B0  jal     0xCA42
```

The copy paths re-enter the same `0x4E6AD4` manager. Thus, in MS12, AC3 and DTS records share one cursor-managed ring and compete for the same `0x66000` window.

## 4. Confirmed standard-image separation

The standard SND image has different manager roots at the corresponding construction sites:

```text
standard AC3 manager = 0x4C6380
standard DTS manager = 0x4D561C
```

The standard DTS manager is not passed to the MS12-style `0xCA42` initialization path in the traced construction, so its base/limit/span remain unresolved.

This standard/MS12 difference is a real structural behavior change, independent of the rejected host copy-length claim.

## 5. Consumer boundary

The ring subsystem still shows:

- write-cursor append/accounting paths;
- no read-cursor payload load in the ring subsystem;
- reset/handshake logic through `0xB00008xx` mailbox/status addresses.

The external consumer identity remains unresolved. The corrected result is therefore:

```text
MS12:
    AC3 + DTS -> one ring manager at 0x4E6AD4

standard:
    AC3 and DTS -> separate manager roots

host loader:
    both image symbols are fully copied after the reserved SHM window
```

## 6. Status correction

| Claim | Corrected status |
|---|---|
| MS12 host `r8 = 0x176930` | **Rejected** |
| MS12 host `r8 = 0x1B6930` | **Verified from raw bytes** |
| MS12 0x40000 image shortfall | **Rejected** |
| MS12 AC3/DTS shared ring | **Verified** |
| Standard AC3/DTS separate manager roots | **Verified at construction sites** |
| Ring span `0x66000` | **Verified** |
| External ring consumer | **Unresolved** |

This correction supersedes any statement in the SND comparison result that treated `0x176930` as the MS12 loader length. No existing artifact was modified.
