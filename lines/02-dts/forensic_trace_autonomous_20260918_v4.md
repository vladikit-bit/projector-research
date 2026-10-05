# Forensic Trace v4 — DTS Compressed Audio Data-Flow & Output Path
Date: 2026-09-18. New report. Does NOT modify prior artifacts. Corrections from v3 are evidence-based.

Primary sources: `dec32_clean.txt` (authoritative listing, byte-matched `dec_full.bin`), `aeon_ORBIS32.sinc`, `utpa2k_stock.ko` / `mik_stock.ko` (.data embedded `mst_codec_r2` + `mst_codec_r2_MS12V22`), live device `C50A/mt5889` read-only.

No patches, no firmware/binary writes, no Ghidra project modification, no settings change, no reboot, no destructive tests.

## Executive summary

### Complete call chain (5 levels)

```
F_0295BC [0x295BC] — top-level audio output orchestrator
  ├── F_0378A5 — HAL audio init
  ├── F_0375BE [0x375BE] — DTS decode dispatch
  │     └── F_03E961 [0x3E961] — DTS decode entry
  │           ├── F_0411A6 [0x411A6] — frame parser
  │           │     ├── F_45E52 (×3) — header/buffer prep
  │           │     ├── F_45FD1 — parser state init
  │           │     └── F_04618E [0x4618E] — bitstream field extractor
  │           │           ├── 0x130D25 — memset (F8/FC zeroing)
  │           │           └── 0x64369 — bitstream reader (ctx+0x0/0x4/0x8)
  │           ├── F_037C94 — post-decode validation
  │           └── F_037ED6 — sub-context check
  ├── F_028AFE [0x28AFE] — per-channel DMA write
  │     └── F_014F63 — DMA transfer to HW
  ├── F_028BC9 — DMA scatter-gather buffer mgmt
  └── F_029391 — DMA trigger
```

### Key correction from v3

v3 stated: "A/B memory-resident after 0x4618E, no verified consumer."

**Corrected**: F_04618E IS a consumer — it reads bits from the bitstream via 0x64369 into F8/FC arrays, sign-extends them, and writes DTS sync/header values (0x7FFE/0x8001). However, the PROCESSED F8/FC content is NOT propagated downstream — F_0411A6 returns status 1 without reading F8/FC, and F_03E961/F_0375BE also do not read F8/FC. The output data path goes through a completely separate buffer chain: r10+0xD4/D8/DC/E0 → F_028AFE → F_014F63 DMA transfer → 0xB000000A+ MMIO.

### The gap: F8/FC are consumed but not forwarded

F_04618E reads DTS frame header fields from the compressed bitstream into F8/FC arrays. After it returns:
- F_0411A6 does NOT read F8/FC — returns status 1 via r3
- F_03E961 does NOT read F8/FC — stores return status, calls F_037C94
- F_0375BE does NOT read F8/FC — writes output buffers at r10+0xD4/D8/DC/E0
- F_0295BC/F_029867 reads r10+0xD4/D8/DC/E0 — these are the actual output buffers

**The parsed DTS header data (in F8/FC) appears to be consumed locally within F_04618E's bitstream processing and is NOT forwarded to the output DMA path.** The actual decoded audio output goes through r10+0xCE8, r10+0x158A8, and the output pointer chain.

## Detailed findings

### 1. F_04618E — DTS frame header parser [VERIFIED]

**Entry**: `0x4618E` (L89181)
**Prologue**: `addi r1,r1,-0x20` (32-byte stack frame)

**Flow**:
1. `0x461a1 lwz r23,0x28(r3)` — load ctx+0x28 (enable flag)
2. `0x461a4 mov r10,r3` — r10 = ctx
3. `0x461a8 beqi r23,0x1,0x461c4` — if disabled, goto exit
4. **Exit (disabled)**: `0x461ab sw_0 0xf0(r10),r14` (r14=0 → F0=0) → return
5. **Processing (enabled)**:
   a. `0x461c4 lwz r14,0x38(r3)` — r14 = N (element count)
   b. `0x461c7 lwz r3,0xf8(r3)` — r3 = F8 pointer
   c. `0x461d2 jal 0x130D25` — memset(F8, 0, N×4)
   d. `0x461de jal 0x130D25` — memset(FC, 0, N×4)
   e. `0x461e2 lwz r25,0x30(r10)` — r25 = ctx+0x30
   f. `0x461e5 lhz r24,0x16c(r10)` — r24 = ctx+0x16C (header field)
   g. `0x461eb ori r12,r0,0x12` — r12 = 18 (sign-extension shift)
   h. **Main loop** (0x461fb–0x46251):
      - For each i in 0..N-1:
        - `0x46210 jal 0x64369` — read r15 bits → F8[i]
        - `0x4621e jal 0x64369` — read r15 bits → FC[i]
        - Sign-extend F8[i] and FC[i] via sll/sra by r12=18
   i. **Post-loop** (0x46259–0x46304):
      - If ctx+0x2C == 1: write DTS sync words to F8/FC
        - F8[0] = 0x7FFE, FC[0] = 0x8001 (DTS IEC61937 sync)
        - Or F8[0] = 0x1FFF, FC[0] = 0xE800 (alternate header)
      - If ctx+0x16C != 0: write header values from ctx+0x16C/16E/170/172
   j. `0x461ab sw_0 0xf0(r10),r14` — F0=0 (always)
   k. Return

**Key observation**: F_04618E writes DTS sync word 0x7FFE into F8[0]. This is the IEC 61937 sync word for DTS compressed audio. Combined with the 0x1FFF/0xE800 values (DTS frame size/type fields), this function appears to be building DTS IEC 61937 headers for pass-through output.

### 2. 0x64369 — Bitstream reader [VERIFIED]

**Entry**: `0x64369` (L130312)
**Structure at r3** (ctx-relative):
- `+0x0`: current byte pointer in bitstream
- `+0x4`: bit offset within current word (0-31)
- `+0x8`: bits remaining in stream

**Function**: Reads N bits (r4) from compressed bitstream, sign-extends result, stores to *r5. Returns 1 on success, 0 on insufficient bits.

**Algorithm**:
```
if bits_requested > bits_remaining:
    *r5 = 0; return 0
bits_remaining -= bits_requested
load word from *current_byte_ptr
shift left by bit_offset, OR with next word shift if spanning word boundary
arithmetic right shift to sign-extend
store result to *r5; return 1
```

### 3. F_03E0DD — ctx initialization (prologue fully resolved)

**Prologue** (L78470-78479):
```
0x3E0DD: addi r1,r1,-0x50     ; 80-byte stack frame
0x3E0E0: sw 0x48(r1),r9       ; save LR
0x3E0E3: sw 0x44(r1),r10
0x3E0E6: sw 0x40(r1),r11
0x3E0E9: sw 0x3c(r1),r12
...
0x3E106: mov r10,r3            ; r10 = r3_in (first param = ctx base)
0x3E108: mov r11,r4            ; r11 = r4_in (second param = table ptr)
0x3E10A: mov r12,r5            ; r12 = r5_in (third param)
```

**Register assignments** (CORRECTED from v3):
- `r14 = r3_in` (first param) — ctx base pointer
- `r11 = r4_in` (second param) — table pointer
- `r12 = r5_in` (third param)
- `r10 = r4_in + 0x10000` (base + 0x10000)
- `r23 = 0x10000` (via `movhi r23,0x1`)

**F_45DE0 call** (L78689):
```
0x3E341: add r3,r11,r14       ; r3 = r11 + r14 (table base + offset)
0x3E343: add r4,r11           ; r4 = r11 (table ptr)
0x3E344: jal 0x45DE0          ; F_45DE0(r3=table+r14, r4=table)
```

### 4. F_45DE0 — F8/FC initialization [VERIFIED CORRECTED]

**Register assignments** (CORRECTED):
- `r3_in` = first param = `r11 + (r14|0x4CCC)` = ctx+0x4CCC (target ctx region)
- `r4_in` = second param = `r11` = table pointer (ctx+0x4E40/0x4E44)

**Prologue** (L88862-88863):
```
0x45DE0: addi r1,r1,-0x10    ; 16-byte stack frame
0x45DE3: sw 0x8(r1),r9       ; save LR
0x45DE6: sw 0xC(r1),r10      ; save r10
```

**Body** (L88865-88896):
```
0x45DE9: mov r13,r4           ; r13 = r4_in (table pointer)
0x45DEB: mov r12,r4           ; r12 = r4_in
0x45DED: mov r11,r0           ; r11 = 0
0x45DEF: mov r14,r0           ; r14 = 0
0x45DF1: mov r9,r0            ; r9 = 0
0x45DF3: mov r10,r3           ; r10 = r3_in (ctx base)

; Pre-fill F8 via 0x130D25 (memset/zero)
0x45DF7: jal 0x130D25         ; fill r3=r10(r3_in), r4=0, r5=0
0x45E00: lwz r3,0(r13)        ; r3 = *(r13+0) = *(table+0)
0x45E04: jal 0x130D25         ; fill r3, r4, r5
0x45E08: lwz r3,0(r13)        ; r3 = *(r13+0) again
0x45E0B: lwz r24,4(r13)       ; r24 = *(r13+4)
0x45E0E: mov r12,r13          ; r12 = r13

; Store F8 = *(table+0)
0x45E17: sw_0 0xF8(r10),r3    ; ctx+0xF8 = *(r13+0) ✓
0x45E1B: lwz r3,0(r12)        ; reload
0x45E1E: sw_0 0xF8(r10),r3    ; redundant store

; Pre-fill FC
0x45E21: jal 0x130D25
0x45E25: lwz r3,4(r12)        ; r3 = *(r12+4) = *(r13+4)
0x45E28: lwz r24,0(r12)       ; r24 = *(r12+0) = *(r13+0)

; Store FC = *(table+4)
0x45E2F: sw_0 0xFC(r10),r3    ; ctx+0xFC = *(r13+4) ✓

; Continue filling
0x45E32: lwz r3,0(r12)
0x45E35: jal 0x130D25
0x45E39: lwz r3,4(r12)
0x45E3C: jal 0x130D25
0x45E40: lwz r3,0(r12)
0x45E43: jal 0x130D25
0x45E47: lwz r3,0(r13)
0x45E4A: jal 0x130D25
0x45E4E: lwz r9,0x8(r1)
0x45E51: jr r9
```

### 5. F8/FC provenance (fully traced)

**Writers** (single chain):
1. `F_03E0DD:0x3E341-0x3E344` builds args → `jal F_45DE0`
2. `F_45DE0:0x45E17` stores `*(r13+0)` → `ctx+0xF8`
3. `F_45DE0:0x45E2F` stores `*(r13+4)` → `ctx+0xFC`

Where r13 = r4 = `r11 + (r14|0x4E40)` in F_03E0DD context.

**Source of table+0/+4** (L78685-78686):
```
0x3E338: movhi r23,0x1
0x3E33B: ori r23,r23,0x4EE8
0x3E33F: add r23,r11,r23      ; r23 = r11+0x4EE8
0x3E341: add r3,r11,r14       ; r3 = r11+r14 = r11+(r14|0x4CCC)
0x3E343: add r4,r11           ; r4 = r11
0x3E344: jal F_45DE0           ; F_45DE0(r3=ctx+0x4CCC, r4=r11)
```

The table base (`r13` in F_45DE0 = `r4` = `r11`) points to ctx+0x4E40/0x4E44 (r11+(r14|0x4E40) and r11+(r14|0x4E44) where r14 = r3_in|0x4CCC). The offsets 0x4E40/0x4E44 store pointers to the actual data regions within the ctx structure.

**Readers**:
- `F_45E52:0x45EFB/0x45EFF/0x45F53/0x45F57` — reads F8/FC as buffer pointers
- `F_04618E:0x461C7/0x461D6/0x46201/0x46214` — memset + bitstream read into F8/FC
- `F_04618E:0x46222-0x46296` — sign-extension and DTS sync word write

### 6. Hardware output path [NEW]

**F_0295BC** (top-level audio output, starts at 0x295BC):
- Reads DMA config from r10+0xEC/0xF0
- Initializes DMA scatter-gather buffers
- Sets DMA buffer size 0x3138, block size 0xC00, period 0x1000
- Writes to hardware table at 0xE0001E10 + channelID×960
- Reads hardware IRQ status registers:
  - `0xB000081C` — IRQ status A
  - `0xB0000838` — IRQ status B
  - `0xB000083C` — IRQ status C (masked with 0x2080)
- Reads audio DMA status from `0xB000000A` (halfword)

**F_0375BE** (DTS decode dispatch, starts at 0x375BE):
- Calls F_03E961 with r3=ctx, r4=r10+0xBA0 (per-stream sub-struct)
- On success, writes 4 output buffer pointers:
  - `r10+0xD4` ← r10+0xACC0 (buffer 0)
  - `r10+0xD8` ← r10+0xBA0 (buffer 1)
  - `r10+0xDC` ← r10+0xCE8 (buffer 2)
  - `r10+0xE0` ← r10+0x158A8 (buffer 3)
- Returns 0 on success

**F_028AFE** (per-channel DMA write, starts at 0x28AFE):
- Reads channel status from `0xB0(r5)` (bitmask, bits 0-5 = channels 1-6)
- For each active channel, reads buffer pointer from:
  - `0xB8` (ch1), `0xBC` (ch2), `0xC0` (ch3), `0xC4` (ch4), `0xC8` (ch5), `0xCC` (ch6)
- Calls trap instructions for channel signaling
- At `0x28B57`: `lhz r7,0xb4(r6)` — load buffer length
- At `0x28B5B`: `lwz r4,0xbc(r6)` — load buffer address
- At `0x28B65`: `jal F_014F63` — DMA transfer (r4=addr, r5=offset, r6=stack_desc, r7=length, r8=6 channels)

**Channel count determination** (F_029867 post-decode, L51131-51167):
- Reads `r14+0xB0` (channel status bitmask)
- Tests bits 0x1,0x2,0x4,0x8,0x10,0x20 for channels 1-6
- Routes to mono/stereo/multi-channel output paths

### 7. Data-flow diagram

```
                    ┌──────────────────────────────┐
                    │   DTS Compressed Bitstream     │
                    │   (ctx+0xF8/FC = buffer ptrs)  │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  F_45E52 (×3)                 │
                    │  Header/buffer prep            │
                    │  F_45FC9 builder               │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  F_45FD1                      │
                    │  Parser state init             │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼───────────────────┐
                    │  F_04618E                     │
                    │  Bitstream field extractor     │
                    │  ┌────────────────────────┐   │
                    │  │ 0x64369 reads N bits    │   │
                    │  │ → F8[i], FC[i]          │   │
                    │  │ Sign-extend (sll/sra 18)│   │
                    │  │ Write DTS sync 0x7FFE   │   │
                    │  └────────────────────────┘   │
                    └──────────┬───────────────────┘
                               │ (F8/FC data NOT forwarded)
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼────────┐     │       ┌────────▼────────┐
    │ r10+0xD4/D8/DC/E0│     │       │ F_037C94        │
    │ Output buffers   │     │       │ Validation       │
    │ (separate path)  │     │       └─────────────────┘
    └─────────┬────────┘
              │
    ┌─────────▼────────┐
    │  F_028AFE        │
    │  Channel dispatch │
    │  (6 channels)    │
    └─────────┬────────┘
              │
    ┌─────────▼────────┐
    │  F_014F63        │
    │  DMA transfer     │
    └─────────┬────────┘
              │
    ┌─────────▼────────┐
    │  0xB000000A      │
    │  HW DMA registers │
    │  0xB000081C/838  │
    │  /83C IRQ status  │
    └──────────────────┘
```

### 8. DTS sync word evidence

The values written into F8/FC at `0x4627B`/`0x46289`:
- F8[0] = 0x7FFE — IEC 61937 DTS sync word (preamble)
- FC[0] = 0x8001 — IEC 61937 DTS sync word (complement)

The values at `0x462EA`/`0x462F1`:
- F8[0] = 0x1FFF — DTS frame size/type field
- FC[0] = 0xE800 — DTS frame type field

These are consistent with IEC 61937 framing for DTS pass-through output. The DSP appears to be constructing DTS compressed audio frames for digital output, not decoding to PCM.

### 9. Sample rate dispatch in F_04618E

The sample-rate lookup table at `0x4610C-0x46148`:
```
0x4610C: ori r23,r0,0x5dc0  ; 24000 Hz
0x46114: ori r23,r0,0x2ee0  ; 12000 Hz
0x4611C: ori r23,r0,0xac44  ; 44100 Hz
0x46124: ori r23,r0,0x5622  ; 22050 Hz
0x4612C: ori r23,r0,0x2b11  ; 11025 Hz
0x46134: ori r23,r0,0x7d00  ; 32000 Hz
0x4613C: ori r23,r0,0x3e80  ; 16000 Hz
0x46144: ori r23,r0,0x1f40  ; 8000 Hz
```

These are DTS standard sample rates, supporting the hypothesis that F_04618E is part of a DTS pass-through pipeline.

### 10. Hardware registers

| Address | Type | Purpose | Access |
|---------|------|---------|--------|
| `0xB000000A` | MMIO R | Audio DMA status (halfword) | `lh` at L50919 |
| `0xB000081C` | MMIO R | IRQ status register A | `lwz` at L50810,51092 |
| `0xB0000838` | MMIO R | IRQ status register B | `lwz` at L50809,51091 |
| `0xB000083C` | MMIO R | IRQ status register C | `lwz` at L50808,51090 |
| `0xE0001E10` | Memory | HW config table (960 bytes/channel) | R at L50732-50741 |

### 11. Runtime status

- Device idle: all `pcm*p` closed, no audio in logcat/dmesg
- Stock modules: `utpa2k 2fc6e9fc...`, `mik c1421040...`
- Live DTS handoff not observable without playback

## Remaining blockers (ranked by priority)

1. **Who fills ctx+0x4E40/0x4E44 (the pointers read by F_45DE0)?** — The exact writer store was found at `0x3E338-0x3E344` in F_03E0DD, but the DATA being pointed to (at the offsets) and who initially populates 0x4E40/0x4E44 is still in the init chain above F_03E0DD.

2. **The F8/FC-to-output bridge** — F_04618E writes DTS sync/header into F8/FC, but this data is NOT forwarded to the output DMA path. The output buffers (r10+0xD4/D8/DC/E0) are populated separately. The connection between F8/FC parsed data and the actual output must go through some intermediate step not yet traced.

3. **F_029867 full function body** — The function at 0x29867 contains the main decode loop and DMA setup but its full body was only partially traced. The complete channel management and DMA descriptor chain needs further analysis.

4. **Playback-gated verification** — All static analysis; live DTS handoff unobserved.

5. **ADB /proc/maps unreadable** — SELinux blocks shell access.

## Next steps

1. Trace ctx+0x4E40/0x4E44 initialization upstream of F_03E0DD (who allocates and fills the deep ctx offsets)
2. Find the bridge between F8/FC parsed DTS data and the output DMA path — likely through the builder functions (F_45D07, F_45FC9) or through ctx fields that F_029867 reads
3. Complete analysis of F_029867 (0x29867) body — the main decode loop that connects F_0411A6 output to DMA
4. Identify whether DTS pass-through or PCM decode is happening (0x7FFE sync word suggests pass-through)
5. Write v5 report once bridge between F8/FC and output is established
