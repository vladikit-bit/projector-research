# SPACE BUNNY — STATIC CONTINUATION: HOST LOADER, DEC FIELDS, SND RING, AEON ABI

**Date:** 2026-09-24  
**Mode:** static/corpus-only  
**Runtime:** none; no ADB, device connection, playback, breakpoint, runtime memory read, module reload, reboot, settings change, patch, firmware write, or flash  
**TCL/T615:** not used  
**Preservation:** no previous report, raw JSONL, firmware binary, Ghidra project, or historical archive file was modified

## Executive result

This continuation materially advances the TD98 execution reconstruction in four areas:

1. **Host-side image placement is now substantially closed.** For the non-TEE loader path, the selected DEC image is copied into `MsOS_PA2KSEG1(B)`, and the selected SND image is copied into the mapped region beginning at `B+0x700000`. The host then initializes and enables the DEC/SND R2 paths. This proves a host mapping/copy chain, not a DSP PC.
2. **DEC descriptor provenance is substantially closed.** `F8/FC` are pointers to separately supplied `0x200`-byte buffers; their source is the fourth-argument structure, conditionally `B0+0x14E40/+0x14E44`. `A108` has two direct writers, with the direct `F_03E0DD -> F_411A6` path verified. No `F8/FC -> R+0x174` publication edge exists.
3. **The SND copy destination is a ring producer, not a standalone local frame buffer.** AC3 and DTS append opaque records into cursor-managed rings. The firmware image contains append/accounting/reset paths but no payload-draining firmware reader. The strongest remaining boundary is an out-of-image hardware/DSP agent synchronized through `0xB00008xx` mailbox/status addresses.
4. **AEON `0x3847E` definitely clobbers `r11` on one reachable path.** Its epilogue is a mirrored save/restore-looking block, strongly suggesting restoration, but the local SLEIGH does not define the `0x20`-family encoding. Therefore `r11` restoration remains a medium-confidence structural inference, not a proof.

A separate static discrepancy was also found: the MS12 SND copy length in the host loader is `0x176930`, while the embedded `mst_snd_r2_MS12V22` symbol is `0x1C1330` bytes. This may be intentional sectioned loading; no root-cause conclusion is made.

## 1. Evidence set and address domains

Primary artifacts used in this pass:

| Artifact | SHA-256 / size |
|---|---|
| `aeon_validate/dec_work/dec_full.bin` | `530bfa684bc363e8513951b731cd42ccae97c96accef3c1d51eb431b61a53f5b`, `0x1E401C` |
| `aeon_validate/r27_work/snd_full.bin` | `1da795fa47aec5aec2aa4cdf78d52028d62abe911f6993dac664206b9210c22d`, `0x1C1330` |
| `patch_baseline/utpa2k_stock.ko` | `8ad05b9688cdbf313c63aa1e5c9b69fa4ac1fdddf456c174f297cb6e716f60da` |
| `patch_baseline/mik_stock.ko` | `44b0b3b66b7255bd5c91c88fef7e9688737dee1c1a9c91667095e4a98d91e70d` |
| extracted `MMAP_MI.h` | `45d306cbf63efbda1181c4eccd0623cce49e3d491e7aa8ceeb741af9c2d4d756` |

Address domains must not be merged:

| Expression | Domain / status |
|---|---|
| `B = 0x22500000` | Physical/configuration base from `MI_MAD_ADV_BUF_ADR`; not a DSP PC |
| `MsOS_PA2KSEG1(x)` | CPU mapping result; the implementation is table-based, not proven identity |
| `g_virDecR2shm` | CPU pointer to mapped `B+0xE01000` region; numeric pointer not statically known |
| `g_virSndR2shm` | CPU pointer to mapped `B+0x70A000` region; numeric pointer not statically known |
| `0x23302044`, `0x23302404` | Arithmetic values before the `MsOS_PA2KSEG1` return translation; not proven live CPU/DSP addresses |
| `0xD72000` | SND image/runtime ring base used by SND firmware; not the host `g_virSndR2shm` pointer |
| `0xB00008xx` | SND hardware-facing mailbox/status/format/counter addresses; consumer identity is outside the firmware image |

## 2. Host loader: DEC/SND placement and enable boundary

### 2.1 Loader call order

In `utpa2k_stock.ko`, the initialization region calls `HAL_MAD2_SetMemInfo` before the image loader:

```text
467640  bl      HAL_MAD2_SetMemInfo
467644  mov     r0, r4
467648  bl      HAL_AUDSP_DspLoadCode
467650  bl      HAL_AUDSP_DspLoadCode
467658  bl      HAL_AUDSP_DspLoadCode
```

`HAL_MAD2_SetMemInfo` writes the low/high halves of the DSP MAD base into the programmed register family around `0x112982..0x112988` and writes the associated window constants. This proves host register programming, not the hardware translation equation.

### 2.2 Conditional standard/MS12 DEC selection

`HAL_AUDSP_DspLoadCode` selects the embedded DEC source and size from `AudioVars+0x4D0`:

```text
46fb14  movw    r2, mst_snd_r2_MS12V22
46fb18  movw    fp, mst_snd_r2
46fb24  movw    r7, mst_codec_r2
46fb28  ldr     r1, [AudioVars, #0x4d0]
...
46fb38  cmp     r1, #4
46fb40  moveq   fp, r2              ; MS12 SND source if selector == 4
46fb50  moveq   r7, r2              ; MS12 DEC source if selector == 4
46fb60  movweq  r8, #0x6930
```

The embedded DEC symbols are:

```text
mst_codec_r2             size 0x29C0F4
mst_codec_r2_MS12V22    size 0x1E401C
```

The standard and MS12 image selection is therefore statically verified as conditional. The live selector remains runtime-dependent.

### 2.3 Direct non-TEE copy path

The loader branches on `AudioVars+0x24F0`:

```text
46fc44  movw    r1, #0x24f0
46fc48  ldr     r0, [r0, r1]
46fc4c  cmp     r0, #1
46fc50  bhi     46fca0              ; TEE/descriptor path for values > 1
```

For values `0` or `1`, the host performs:

```text
46fc54  mov     r0, #2
46fc58  bl      HAL_AUDIO_GetDspMadBaseAddr
46fc5c  mov     r6, r0
46fc60  mov     sl, r1
46fc64  bl      MsOS_PA2KSEG1
46fc68  mov     r1, #0
46fc6c  mov     r2, #0x700000
46fc70  bl      memset
46fc74  mov     r0, r6
46fc78  mov     r1, sl
46fc7c  bl      MsOS_PA2KSEG1
46fc80  mov     r1, r7              ; selected DEC source
46fc84  mov     r2, r5              ; selected DEC size
46fc88  bl      memcpy
46fc8c  bl      HAL_DEC_R2_init_SHM_param
```

The first mapping result is used to clear `0x700000` bytes; the second is used as the destination for the selected embedded DEC image. This is the strongest current proof of host-side DEC image placement.

The `>1` branch builds the previously described command-3 TEE descriptor. The host TEE wrapper itself dispatches to OPTEE only when `AudioVars+0x24F0 == 2`; values above that return failure. The direct copy path is therefore distinct from the TEE path.

### 2.4 Common SND copy path

After either the direct DEC copy or the TEE branch, the common loader path maps `B+0x700000` and copies the selected SND source:

```text
46fd2c  mov     r0, #2
46fd30  bl      HAL_AUDIO_GetDspMadBaseAddr
46fd34  adds    r7, r0, #0x700000
46fd38  adc     r6, r1, #0
46fd3c  mov     r0, r7
46fd40  mov     r1, r6
46fd44  bl      MsOS_PA2KSEG1
46fd48  mov     r1, fp              ; selected SND source
46fd4c  mov     r2, #0xa000
46fd50  bl      memcpy
...
46fd5c  bl      MsOS_PA2KSEG1
46fd64  add     r0, r0, #0xaa00
46fd68  mov     r2, #0x6f5600
46fd70  bl      memset
46fd78  bl      MsOS_PA2KSEG1
46fd84  add     r0, r0, #0xaa00
46fd88  add     r1, fp, #0xaa00
46fd8c  mov     r2, r8
46fd90  bl      memcpy
```

The arithmetic `0xAA00 + 0x6F5600 = 0x700000` is exact. The second copy length is held in `r8` and is selected conditionally; it is not safe to identify it with the full symbol size in both profiles.

The loader then calls:

```text
46fd94  bl      HAL_SND_R2_init_SHM_param
46fda4  mov     r0, #1
46fda8  bl      HAL_SND_R2_EnableR2
46fdb0  bl      HAL_DEC_R2_EnableR2
```

This reaches a hardware-facing enable boundary, but does not identify a physical SPDIF transport.

### 2.5 `MsOS_PA2KSEG1` is not proven identity

`MsOS_MPool_PA2KSEG1` is a wrapper at `.text+0xFE18` that sets a mapping selector and enters the common PA-to-VA routine. The routine scans `mpool_info` entries and, for a selected range, loads a table-derived mapping base at `entry+0x20/+0x24`. The visible return path is:

```text
0ff84  ldr     r0, [sl, #0x20]
0ff88  ldr     r3, [sl, #0x24]
0ff94  addne   r5, r0, fp
0ffa4  mov     r0, r5
```

Thus the returned pointer is a table-derived mapping result plus the input offset. It is not safe to identify `MsOS_PA2KSEG1(B)` with numeric `B` without the runtime mapping table.

### 2.6 SND and DEC shared-memory initialization

`HAL_SND_R2_init_SHM_param` repeats the mapping construction:

```text
4582c4  bl      HAL_AUDIO_GetDspMadBaseAddr(2)
4582c8  movw    r2, #0xa000
4582cc  movt    r2, #0x70          ; r2 = 0x70a000
4582d8  bl      MsOS_PA2KSEG1
4582e4  str     r0, [g_virSndR2shm]
```

`HAL_DEC_R2_init_SHM_param` uses `B+0xE01000` for `g_virDecR2shm`.

`HAL_SND_R2_Set_SHM_PARAM` at `.text+0x459DFC` has the same lazy `B+0x70A000` mapping and a selector jump table at `0x459E70`. It writes control fields into `g_virSndR2shm` and flushes memory. Representative handlers include indexed fields around `+0x70`, flag fields around `+0x88/+0x90/+0xE0`, an array/count pair at `+0x94/+0x98`, and later fields through `+0x348`.

Relevant callers include `HAL_MAD_SetAudioParam2`, `HAL_MAD_SetCommInfo`, `HAL_MAD_SetAC3PInfo`, `HAL_AUDIO_CPU_Sound1_Write`, and `HAL_SND_SetParam`. This proves a host control plane into the mapped SND region. It does **not** prove that this CPU mapping is identical to the SND firmware ring address `0xD72000`.

### 2.7 MS12 SND copy-length discrepancy

For the standard path, the loader’s `r8` is initialized as:

```text
0x00173F48
```

For selector `4`, the conditional `movweq #0x6930` changes the low half while retaining the previously loaded high half, yielding:

```text
r8 = 0x00176930
```

The symbol sizes are:

```text
mst_snd_r2             0x17E948
mst_snd_r2_MS12V22    0x1C1330
```

For standard SND, `0x173F48` equals the symbol size minus `0xAA00`. For MS12 SND, the copied length is `0x176930`, leaving `0x4AA00` relative to the full symbol end. This is a verified static discrepancy. It may reflect sectioned loading or a deliberately copied subset; no failure, corruption, or DTS root cause is inferred.

## 3. DEC field closure

### 3.1 `[B0+0x1414C]`

The listing contains ten direct `0x414C(r10)` sites, with one direct writer:

```text
0x3DF2D  bg.lwz  r11,0x414c(r10)       ; V_in load
0x3DF79  bg.sw_0 0x414c(r10),r11       ; only direct writer
```

The other sites are readers or helper calls. The field is therefore not a collection of unrelated aliases; its address form is singular in this image.

The value stored at `0x3DF79` is the post-helper `U`, not necessarily the original `V_in`. The two reachable calls to `0x3847E` are at `0x3DF3B` and `0x3DF4D`.

### 3.2 `U` after `0x3847E`

The helper has a reachable loop that writes `r11`:

```text
0384c8  bn.blesi r12,0,0x384e4
0384cb  bt.mov   r11,r1
0384de  bt.addi  r11,0x4
0384e0  bg.bgts  r12,r10,0x384cf
```

When `r12 > 0`, `r11` becomes `r1 + 4*k`, a stack address. When `r12 <= 0`, that write block is skipped. The return-visible value is consequently conditional:

```text
U ∈ { V_in, stack address r1+4k }
```

The direct call/return structure and raw bytes are high confidence. The exact post-loop value remains medium confidence because the epilogue’s `0x20`-family instructions are not modeled by the local SLEIGH.

### 3.3 `[B0+0xA108]`

There are two direct writers using the `-0x5EF8(r10)` form:

```text
0x3E1A6  [B0+0xA108] = r23 = U+0x7C4
0x3EA38  [B0+0xA108] = r28
```

The first source is directly tied to the `U` result of `0x3847E`. The second is in `F_3E961`, where `r10` has multiple competing definitions and back edges; its identity as the same `B0` is conditional.

The direct consumer chain is verified:

```text
0x3E1A6  A108 = U+0x7C4
0x4127D  r4 = A108
0x41283  parser 0x45FD1(D_B, r4)
0x41289  output  0x4618E(D_B)
```

`F_45FD1`’s 14-entry dispatch table resolves to local scalar/control writers. No target touches `F8/FC` or an A/B transport buffer.

### 3.4 `D+0xF8` and `D+0xFC`

`F_45DE0` has exactly two relevant descriptor stores:

```text
0x45E17  [D+0xF8] = *(arg4+0)
0x45E2F  [D+0xFC] = *(arg4+4)
```

Each pointed-to buffer is cleared with length `0x200`:

```text
0x45E21  F_130D25(F8, 0, 0x200)
0x45E39  F_130D25(FC, 0, 0x200)
```

The descriptor clear itself is `F_130D25(D, 0, 0x174)`, so the two pointer fields lie inside the cleared descriptor range.

At the direct call site, the fourth argument is built from:

```text
0x3E32F/0x3E33F  r11 + (r14 | 0x4E40)
```

When `r14 == 0x10000`, this is conditionally:

```text
B0+0x14E40 / B0+0x14E44
```

Readers are confined to `F_45E52` and `F_4618E`. `F_4618E` uses `D+0x38` as an element count, zero-fills `4*count` bytes in both buffers, and processes indexed elements. No pointer or payload copy from `D+F8/FC` into `R+0x174` was found.

### 3.5 `R+0x174`

`F_029867` reads:

```text
0x29FE9  S = *(R+0x174)
0x29FF0  first dereference of S
```

It later passes `R+0x140` or `R+0xFC` to `0x28A33` with a length from `R+0x178`. This establishes a real `S` consumer path, but not an F8/FC source.

The image has multiple `R+0x174` writers, including in-family rotation/accumulation regions around `0x18E14`, `0x19D04`, `0x1ABF5/0x1ABF9`, and other listed state writers. None receives `D+F8/FC`. The edge remains blocked; the correct statement is “F8/FC publication into S was not found,” not “R+0x174 has no writers.”

## 4. SND ring producer and consumer boundary

### 4.1 Copy destinations are ring cursors

The AC3 path constructs:

```text
r13 = 0x4E0000
r12 = r13+0x6AD4
r14 = [r12+0x10]
r3  = r14+8
0x1EDD6: jal 0x1078A9
```

The DTS path constructs:

```text
r11 = 0x250000
r3  = [r11+0x10]
r3  = r3+8
0x1F8C5: jal 0x1078A9
```

The destination is therefore a cursor-derived location inside a ring manager, not a newly allocated standalone frame buffer.

### 4.2 Ring layout and record shape

The bounded SND sweep recovered the manager fields:

```text
[+0x00]  base
[+0x04]  limit
[+0x08]  span
[+0x0C]  read cursor
[+0x10]  write cursor
[+0x14]  fill = write-read
```

The AC3 manager at `0x4E6AD4` is initialized with the fixed SND-image base `0xD72000`, span `0x60000`, and reset/initial cursors at that base. The exact relation between the separately decoded `limit` value and `base+span` should not be assumed without the original initialization structure.

Each appended record has an 8-byte header followed by opaque payload bytes:

| Path | Header words | Payload copy length |
|---|---|---:|
| AC3 | `0xF872, 0x4E1F, 0x0001, count<<3` | `0xA00` |
| DTS | `0xF872, 0x4E1F, 0x000B, 0x3F80` | `0x7F0` |

The fourth DTS word is the decoded `0x7F0<<3` quantity. The header is a queue-element descriptor; no PCM/IEC payload interpretation is claimed.

### 4.3 Append/accounting behavior

`0xC8CA` reads the write cursor, performs a level/peak analysis call on the record, and advances the write cursor by the decoded length unit. `0xC49D` and `0xCC9B` update fill/accounting fields. `0xC544`, `0xCB50`, and `0xCAB4` handle flags, classification, and reset behavior.

The bounded sweep found no firmware function that:

- advances the read cursor as a payload consumer;
- dereferences the payload for transport;
- passes the record through a function-pointer/queue handoff;
- transfers the ring to a later firmware task.

The immediate post-copy calls to `0xC8CA` receive the ring object, not the copied destination. This reconfirms the earlier negative result at a stronger structural level.

### 4.4 Hardware-facing synchronization

The SND image accesses a hardware-facing mailbox/status family:

```text
0xB000080C   status polling / timeout
0xB0000814   dispatcher read
0xB000084C   counter/status write
0xB0000854   format/status write
```

`0xC55E` polls status and resets the ring after the hardware handshake. The strongest interpretation is that an external hardware/DSP agent drains the ring out of band. The exact consumer, DMA descriptor programming, and physical output path are not present in the SND firmware image.

### 4.5 Ring-instance aliasing remains open

The AC3 manager is explicitly initialized at `0x4E6AD4` with the `0xD72000` base. The DTS path uses a manager rooted at `0x250000`, but the traced path does not establish an equivalent initialization call for that instance. Therefore it is not proven that AC3 and DTS feed the same ring buffer.

A third appender exists in the common `fn00017AD4` region, but its exact clean call site was not pinned by the available listing. It does not change the primary result: all known append paths deposit records; no firmware reader is present.

## 5. AEON `0x3847E` and the epilogue

### 5.1 Reachable clobber

The helper extends over `0x3847E..0x38519`, with a `0x40` frame. The loop path at `0x384CB/0x384DE` writes `r11` and advances it by four bytes per iteration. The statement “`0x3847E` never clobbers r11” is disproven at the byte/control-flow level.

### 5.2 Epilogue boundary

The return sequence is:

```text
0x38516  bn.addi r1,r1,0x40
0x38519  bt.jr r9
```

The preceding `0x38500..0x38514` block consists of unmodeled `0x20`-family encodings. A 32-bit interpretation that swallowed the frame teardown and return is structurally impossible. A mixed 16/32-bit interpretation and an all-16-bit interpretation remain possible from the local bytes.

A structural comparison of prologue/epilogue blocks in four functions found mirrored operands: the corresponding words differ systematically in bit0 and preserve register/operand relationships. The most parsimonious reading is a save/restore pair, which would restore `r11`. That is an inference, not a decoded proof.

The standard-image counterpart at `0x43275` is byte-identical in the helper and epilogue structure, with shifted call displacements. It corroborates the code shape but supplies no independent decoding of the `0x20` family and does not transfer semantics automatically.

### 5.3 Consequence for `U` and `A108`

The static result is now:

```text
if the mirrored epilogue restores r11:
    U = V_in
    A108 = V_in+0x7C4
else:
    U may remain the stack-derived r1+4k value
```

The firmware’s use of the value across the helper calls makes restoration behaviorally likely, but the local decoder cannot certify it. `A108` is therefore still conditional on the epilogue interpretation, while the direct dataflow and writer/readers are verified.

## 6. Updated execution matrix

| Edge | New static status |
|---|---|
| `MI_MAD_ADV_BUF` config → `B` | Verified for extracted TD98 config |
| `B` → `HAL_AUDIO_SetDspBaseAddr` | Verified |
| Direct DEC image copy destination | Closed as `MsOS_PA2KSEG1(B)`; not DSP PC |
| Direct SND image copy destination | Closed as mapped `B+0x700000`; exact CPU pointer unresolved |
| `g_virSndR2shm` | Closed as `MsOS_PA2KSEG1(B+0x70A000)` |
| `g_virDecR2shm` | Closed as `MsOS_PA2KSEG1(B+0xE01000)` |
| DEC `A108` direct writer/reader chain | Verified conditionally on `U` |
| DEC `F8/FC` source and local consumers | Closed: two `0x200` buffers, two writers, local readers |
| `F8/FC → R+0x174` | Not found; remains blocked |
| `R+0x174 → F_28A33` | Reader path exists; object provenance still open |
| AC3/DTS SND copy destination | Closed as ring cursor destinations |
| SND firmware payload consumer | Not present in bounded firmware sweep |
| SND hardware synchronization | `0xB00008xx` boundary identified; consumer identity blocked |
| `0x3847E` r11 clobber | Proven on reachable loop path |
| `0x3847E` r11 restoration | Strong structural inference, not proof |
| TEE secure translation / DSP PC | Still blocked |
| Physical SPDIF output | Still blocked |

## 7. Remaining static work

The highest-value next static checks are now narrower:

1. Obtain or derive the MStar/AEON `0x20/0x21` instruction-family definition, or an independent R2 decoder, to decide the `0x3847E` epilogue width and register mapping.
2. Prove the constructor/lifetime of `[B0+0x1414C]` and determine whether the post-helper `U` is `V_in` on the active path.
3. Resolve the object identity behind `R+0x174` and determine whether any indirect state transition connects it to the F8/FC producer.
4. Establish whether the DTS manager at `0x250000` aliases the AC3 `0xD72000` ring.
5. Locate static hardware-loader/DMA metadata outside the SND firmware image that programs or drains the ring.
6. Compare the host `g_virSndR2shm` control fields with the SND firmware ring manager; do not merge them without a mapping proof.
7. Explain the MS12 SND copy-length discrepancy from image section metadata before assigning any behavioral meaning.

Until those edges are closed, the defensible static endpoint is:

```text
host configuration
  -> mapped B
  -> DEC/SND image placement
  -> host shared-memory/control programming
  -> local DEC parser/builder/output descriptors
  -> SND ring record production
  -> hardware-facing mailbox boundary
```

The exact hardware ring consumer, DMA/queue transport, secure TEE translation, and physical SPDIF endpoint remain unproven.

## Preservation statement

This pass created only this new report. It did not modify prior reports, raw JSONL, binaries, firmware images, Ghidra projects, ReKo projects, or historical archive files. No TCL/T615 evidence was used.
