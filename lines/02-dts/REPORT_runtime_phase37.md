# R37 — Lifecycle / Context of SND R2 SHM `+0x868` / `+0x86C`

**Prior:** R33-B (write-only) · R34-C (window = SND R2 SHM) · R35-D (no reader) · R36-C (region identified, semantics unknown)  
**R36 baseline taken as established:** SHM base `DspMadBase+0x7000A000`, size `0x8B8`; `+0x868`/`+0x86C` are 32-bit  
fields in the trailing region; 6 stores / 0 loads:  
`A868: 0xC8C2 (0), 0xD15E (value)` · `A86C: 0xCAAC (0), 0xCE26 / 0xCE72 / 0xCF48 (value)`.  
**This phase:** pure static. No patch, no device change, no runtime experiment, no EDID/ARC/topology/settings/AUTH change.  
**Not repeated:** R35/R36 exhaustive reader search; `0x25EE1`, `0x0C06`, `divu`, `0x109595`, codec-R2, `mik.ko`.  
**Deliverable date:** 2026-09-10

---

## Headline classification

> ### R37 = **R37-B** — **OUTPUT-RELATED LIFECYCLE; field semantics remain UNKNOWN.**
>
> Both clear sites sit inside **output / format-handling code**, and the `A86C` clear is demonstrably part of a  
> **DTS descriptor reset sequence**. This raises confidence that the fields belong to the DTS output state machine —  
> but it still does **not** establish what the fields *mean*.

---

# 1. Clear-site table

| Field    | Clear address | Function / region                                                                                                                       | Context immediately before                                                                                                                                                                        | Trigger                                    | Following action                                                                  | Confidence                                                                         |
| -------- | ------------: | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `+0x86C` |      `0xCAAC` | function containing `0xCA60…0xCAB0` (prologue not in dumped range; `0xC8CA` begins the *next* function)                                 | **descriptor reset**: `+0x0=r3, +0x4=r23, +0x8=r11, +0xc=r3, +0x10=r3, +0x14=0, +0x18=0, +0x1c=0xbb80, +0x20=0, +0x24=0, +0x28=0` (struct @`r10`), then `jal 0x10C5B9`, then `jal 0xD4C8` (flush) | not statically known (callers unavailable) | `bg.sw_0` → `[0xA86C]=0`, then `bg.j 0xD4C8` (flush+invalidate+`syncwritebuffer`) | **STRONG** (reset context verified; trigger unknown)                               |
| `+0x868` |      `0xC8C2` | block `0xC8B4…0xC8C6`; neighbouring function `0xC800…0xC830` does format dispatch and writes `0xB000_0854` (SPDIF control) then returns | `0xC8B1 bn.sw 0x34(r3),r0` (write 0 to struct `+0x34`); surrounding region compares format codes `0xee00 / 0x5888 / 0xf400 / 0xb110 / 0x7d00 / 0xac44 / 0xfa00` and writes `0xB000_0854`          | not statically known                       | `bg.sw_0` → `[0xA868]=0`, then `bg.j 0xD4C8` (flush)                              | **MODERATE** (output neighbourhood verified; exact function boundary not resolved) |

### Notes on function boundaries

- `0xC830` is `bt.jr r9` (return) ⇒ the format-dispatch function that writes `0xB000_0854` **ends** there.
- `0xC8CA` is `bt.addi r1,-0x10` (prologue) ⇒ a **new function begins** at `0xC8CA`; this new function contains `0xCAAC`.
- Therefore `0xC8C2` (A868 clear) and `0xCAAC` (A86C clear) are in **different functions**, both inside the  
  output/format-handling area (below `0xCC9B`).

# 2. SHM tail map (only confirmed fields)

```
SND R2 SHM base = g_virSndR2shm ; size 0x8B8
+0x104 … +0x194   known host-configured parameter block
… further known host fields to ~+0x338 …
+0x800 …          (no confirmed access with verified base)
+0x854            STORE  (0x25A4D, via base 0xA000)          confirmed
+0x856            STORE  (0x2587D, 0x25E70)                  confirmed
+0x868            STORE set 0xD15E (0xCF86) / clear 0xC8C2   confirmed
+0x86C            STORE set 0xCE26/0xCE72/0xCF48 (0xCC9B) / clear 0xCAAC   confirmed
+0x870            STORE  (0xB3DD)                            confirmed
+0x87C            STORE  (0x266C7)                           confirmed
+0x87E            STORE  (0x266B5)                           confirmed
+0x884            STORE  (0x265E4)                           confirmed
+0x886            STORE  (0x265D2)                           confirmed
+0x88A            STORE  (0xB3B1)                            confirmed
+0x88C            STORE  (0xD903)                            confirmed
+0x890            STORE  (0xD90D)                            confirmed
+0x894            STORE  (0xD8F9)                            confirmed
+0x895 … +0x8B7   (no confirmed access with verified base)
```

*(Every entry above required a **verified `0xA000` base** within 64 bytes; offset-only matches were rejected.)*

⇒ `+0x868`/`+0x86C` are **not isolated**: they sit in a **local cluster of 32-bit runtime-published fields**  
spanning roughly `+0x854 … +0x894`.

# 3. Lifecycle

```text
+0x86C:
  [init]      SHM memset 0x8B8 by host (HAL_SND_R2_init_SHM_param, movw r2,#0x8b8 @0x4582F4)
     ↓
  [reset]     descriptor struct @r10 zeroed:
              +0x14=0 +0x18=0 +0x20=0 +0x24=0 +0x28=0, +0x1c=0xbb80,
              +0x0/+0x4/+0x8/+0xc/+0x10 = r3/r23/r11
              → jal 0x10C5B9 → jal 0xD4C8 (flush)
     ↓
  [clear]     0xCAAC: [0xA86C] = 0   → 0xD4C8 flush
     ↓
  [set]       0xCC9B computes value → 0xCE26 / 0xCE72 / 0xCF48: [0xA86C] = value → 0xD4C8 flush
     ↓
  … (clear again on next reset; trigger not statically known)
```

```text
+0x868:
  [init]      SHM memset 0x8B8
     ↓
  [clear]     0xC8B1: struct+0x34 = 0 ; then 0xC8C2: [0xA868] = 0 → 0xD4C8 flush
     ↓
  [set]       0xCF86 computes value → 0xD15E: [0xA868] = value → 0xD4C8 flush
```

**The set/clear pairing is symmetrical (store → flush) for both fields.** The sequencing  
`init → clear → set → … → clear` is **consistent** with the static evidence but the *trigger* of the clear is  
**not proven** (callers unavailable).

# 4. Semantic assessment

## VERIFIED

- Both clear sites store **0** and are immediately followed by the same cache-maintenance helper `0xD4C8`  
  (flush+invalidate+`syncwritebuffer`), invoked with `r3 =` the slot address and `r4 = 4`.
- The `A86C` clear (`0xCAAC`) is preceded by writes to a struct at `r10` using the field offsets  
  `+0x0, +0x4, +0x8, +0xc, +0x10, +0x14, +0x18, +0x1c, +0x20, +0x24, +0x28` — **the same offset set that  
  `0x13292` writes to the DTS descriptor `0x252AD0`** — with the arithmetic inputs (`+0x14`, `+0x18`, `+0x20`)  
  **zeroed**. This is a reset of the descriptor, not a population.
- The `A868` clear (`0xC8C2`) lies in a code region that performs **format-code dispatch**  
  (`0xee00 / 0x5888 / 0xf400 / 0xb110 / 0x7d00 / 0xac44 / 0xfa00`) and writes **`0xB000_0854`** (the SPDIF  
  control register) — i.e. an output/format-handling neighbourhood.
- `+0x868`/`+0x86C` belong to a **verified cluster** of 32-bit fields at SHM `+0x854 … +0x894`.

## STRONG INFERENCE

- **The two fields are published runtime state of the DTS output path.** They are (a) computed from the DTS  
  descriptor fields `+0x14/+0x18/+0x20` plus globals, (b) published with cache maintenance, and (c) zeroed during  
  a descriptor reset / output-handling sequence. That is a *publish-and-invalidate* lifecycle tied to DTS output.
- **The clear is not merely a boot-time init**: it lives in runtime output/format code, and it is paired with a  
  set, so the field is re-published and re-cleared during operation.

## UNRESOLVED

- **What the fields mean.** Still only “32-bit computed runtime value”.
- **The trigger/condition** that leads to each clear (caller graph not recoverable from the sparse dumps).
- **Whether `+0x868` and `+0x86C` are two instances of one thing** (e.g. two output legs) or two different values.
- **Exact function boundaries / names** for both clear functions (no symbols in the R2 image).
- **Any direct link to SDO / IEC61937 / packer / SPDIF-HDMI TX** — none established.
- The role of `+0x1c = 0xbb80` in the reset, and of `struct+0x34 = 0` before the `A868` clear.

# 5. Architectural implication (change vs R36)

|                 | R36                                            | R37 (now)                                                                                 |
| --------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Placement       | inside SHM, trailing ~80 B, outside host block | unchanged, **plus** shown to be in a verified local cluster `+0x854…+0x894`               |
| Lifecycle       | set + clear + flush                            | **clear context decoded**: descriptor-reset (`A86C`) and output/format region (`A868`)    |
| Output relation | unknown                                        | **output-related lifecycle established** (descriptor reset + SPDIF-control neighbourhood) |
| Semantics       | unknown                                        | **still unknown**                                                                         |

R37 therefore **upgrades confidence** that these fields belong to the DTS output state machine, but it  
**does not** convert that into a semantic identification, and it **does not** identify the consumer (R35-D stands).  
The honest reading remains: *published DTS-output-path runtime state, purpose unknown, read by an agent not present  
in this workspace.*

# 6. Stopping condition

Per the R37 rule, no new exhaustive static scan is warranted without a new artifact. The single most valuable  
missing artifact is unchanged from R36:

1. **SND R2 SHM layout header covering `+0x854 … +0x894`** (would name the tail cluster, including `+0x868`/`+0x86C`).
2. (secondary) **R2 linker map / symbols** for `snd_full.bin`, to give real function/symbol names for the two clear  
   functions instead of raw addresses.

---

## Appendix — artifacts

- `r36_work/dump_C8.txt` — dump `0xC800:0x400` (both clear sites).
- `r37_work/dump_C4.txt` — dump `0xC400:0x420` (code leading to the `A868` clear; format dispatch; `0xB000_0854` write).
- Tail-cluster scan: base-`0xA000`-verified stores in `+0x800…+0x8B7` (both address encodings searched).
