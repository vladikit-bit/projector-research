# R36 — Semantic Identification of SND R2 SHM `+0x868` / `+0x86C`

**Prior:** R33-B (write-only in `snd_full.bin`) · R34-C (window = SND R2 SHM) · R35-D (no reader in available artifacts)
**This phase:** pure static. No patch, no device change, no runtime experiment, no EDID/ARC/topology/settings/AUTH change.
**Not repeated:** `0x25EE1`, `0x0C06`, `divu`, `0x109595`, producer internals, prior false positives, codec-R2
`B000_0868/086C`, `mik.ko`.
**Deliverable date:** 2026-09-10

---

## Headline classification

> ### R36 = **R36-C** — **REGION IDENTIFIED, FIELD SEMANTICS UNKNOWN**
> `+0x868`/`+0x86C` are proven to lie **inside the SND R2 shared memory** (size `0x8B8`), in the **trailing
> ~80 bytes**, **outside** the host-configured parameter block. **No artifact names them or states their purpose.**

No vendor header, struct, `#define`, linker map or symbol gives them a semantic name. The only "names" that exist
are **decompiler-generated from the address itself** (`A868`, `A86C`), which carry no meaning.

---

# 1. VERIFIED

### V1 — No semantic definition exists in any workspace artifact
Searched `*.h *.hpp *.c *.cpp *.inc *.map *.lst *.sym *.asm *.s` for `0x868`, `0x86C`, `A868`, `A86C`,
`7000A000`, `SND_R2_SHM`, `R2SHM`, `SHM_PARAM`, `g_virSndR2shm`.
- The only hits for `A868`/`A86C` are in the **Reko decompilation output**
  `aeon_validate/r27_work/snd_full.reko/snd_full.h`:
  ```
  1558: (A868 Eq_3563 tA868)
  1559: (A86C Eq_3938 tA86C)
  ```
  These are **auto-generated, address-derived** names (Reko names unknown globals after their address).
  **They are not vendor/semantic names.**
- No `struct { … field @ +0x868 … }` and no `#define …_XXX 0x868` exists anywhere in the workspace.
- `7000A000` appears only in the R34/R35 reports (derived by us), not in any vendor artifact.
- **No `.map`, `.sym`, `.lst`, `.axf`, `.out` artifacts exist** for the R2 images. The only `.elf` in the
  workspace is an unrelated `hwcomposer.debugdata.elf`.

### V2 — Complete access enumeration (and correction to R33): **6 stores, 0 loads**
R33 enumerated sites using the `base 0xA000 + offset 0x868/0x86C` form and found 4. R36 additionally searched the
**direct-immediate** form and found **2 more**, which R33 had missed:

| Addr | Instruction | Form | Kind |
|------|-------------|------|------|
| `0xCE26` | `bg.sw_0 0x86c(r23),r24` | `0xA000 + 0x86C` | **STORE (set)** — `0xCC9B` |
| `0xCE72` | `bg.sw_0 0x86c(r23),r24` | `0xA000 + 0x86C` | **STORE (set)** — `0xCC9B` |
| `0xCF48` | `bg.sw_0 0x86c(r23),r24` | `0xA000 + 0x86C` | **STORE (set)** — `0xCC9B` |
| `0xD15E` | `bg.sw_0 0x868(r23),r24` | `0xA000 + 0x868` | **STORE (set)** — `0xCF86` |
| **`0xC8C2`** | **`bg.sw_0 -0x5798(r23),r0`** | **`0x10000 − 0x5798 = 0xA868`** | **STORE (clear → 0)** ← NEW |
| **`0xCAAC`** | **`bg.sw_0 -0x5794(r23),r0`** | **`0x10000 − 0x5794 = 0xA86C`** | **STORE (clear → 0)** ← NEW |

> **Why R33 missed them:** R33 scanned for a base built as `0xA000` (`movhi 0x1; addi −0x6000`) followed by offset
> `0x868/0x86C`. These two build the **full address directly** (`movhi r23,0x1; addi/sw −0x5798/−0x5794`), so an
> offset-only or base-only scan cannot see them. **Any future search must cover both encodings.**

Independently confirmed by the Reko SSA dump, which lists exactly these six memory operations:
`0xA868 @ 0xC8C2, 0xD15E` · `0xA86C @ 0xCAAC, 0xCE26, 0xCE72, 0xCF48`.
**⇒ Zero load operations exist anywhere in `snd_full.bin` for these addresses (R33 write-only conclusion held).**

### V3 — Width is 32-bit
Reko types both as `word32` (`Mem[0xA868<32>:word32]`), and all six accesses are `bg.sw_0` (store word).
**VERIFIED: 32-bit.**

### V4 — SHM size is `0x8B8` (2232 bytes)
In `utpa2k.ko`, `HAL_SND_R2_init_SHM_param` (`@0x4582A8`) contains:
```
0x4582F4  0xE30028B8   movw  r2, #0x8b8
```
i.e. the SHM is zeroed over **`0x8B8` bytes**. **VERIFIED** (independently confirms the earlier R7/R8 note
`memset(shm,0,0x8b8)`).

### V5 — `+0x868`/`+0x86C` lie inside the SHM, in its trailing region
```
SHM span      : [ g_virSndR2shm , g_virSndR2shm + 0x8B8 )
+0x868        : offset 2152 of 2232   → 0x50 (80) bytes before the end
+0x86C        : offset 2156 of 2232   → 0x4C (76) bytes before the end
```
Both are **inside** the SHM, in the **last ~80 bytes**, i.e. **beyond every known host-configured field**
(the documented parameter block is `+0x104…+0x194`, with further known fields up to `~+0x338`).

### V6 — Set/clear publication pattern with cache maintenance
Each of the six stores is followed by the same cache-maintenance helper `0xD4C8`
(`flush+invalidate` + `bg.syncwritebuffer`), invoked with `r3 =` the slot address and `r4 = 4`:
- set path: `0xD16C addi r3,r3,−0x5798` (=`0xA868`) → `j 0xD4C8`; `0xCE32/0xCE7E/0xCF54 addi r3,r3,−0x5794` (=`0xA86C`) → `j 0xD4C8`.
- clear path: `0xC8C2` store 0 → `0xC8C6 j 0xD4C8`; `0xCAAC` store 0 → `0xCAB0 j 0xD4C8`.

⇒ The firmware **publishes a 32-bit value** (set by the DTS helpers, cleared by a separate path) and makes it
visible outside its own cache. **This implies an intended external observer**, but does not identify it.

---

# 2. STRONG INFERENCE

- **The two slots are runtime-published R2 state, not host-configured parameters.** They are zeroed at SHM init
  (the whole `0x8B8` is memset), never written by the host accessor layer (R35), set only by `0xCC9B`/`0xCF86`
  (DTS private helpers), and cleared by a distinct path. The set → flush / clear → flush symmetry is a
  publication lifecycle.
- **They are not part of the host parameter block.** Their offset (`0x868+`) is far beyond the documented
  `0x104–0x194` block and the other known fields, consistent with R35's finding that no host function touches them.
- **The value is a computed additive 32-bit quantity**, not a flag: it is formed as a sum of globals plus a
  quotient, and is stored/cleared as a full word. Beyond "32-bit computed value", its *kind* (see Phase D) is
  **not** established.

# 3. UNRESOLVED

- **Field semantics / purpose** of `+0x868` and `+0x86C` — no artifact names or describes them.
- **The value's kind**: whether it is timing-like, length-like, rate-like, pointer-like or flag-like.
  Per instruction, it is **not** being called a buffer length, DMA size, sample count, rate, or descriptor.
- **Whether `0x868` and `0x86C` are a pair** (two instances of one thing) or two independent parameters.
- **Intended reader / consumer class** (host / other DSP / hardware) — still unknown (R35-D).
- **Connection (if any) to DTS SDO / IEC61937 / packer / SPDIF-HDMI TX** — not established.
- **Identity of the function containing the clear sites** `0xC8C2` / `0xCAAC` (they sit below `0xCC9B`, outside
  the previously dumped `0xCC00–0xDDFF` window) — the clear path's owning function was not decoded.

# 4. SHM map

```
SND R2 SHM base = g_virSndR2shm = MsOS_PA2KSEG1(DspMadBase + 0x7000A000)
total size      = 0x8B8 (2232 B), zeroed by HAL_SND_R2_init_SHM_param (movw r2,#0x8b8 @0x4582F4)

+0x000  …                                   (SHM header / unclassified)
+0x104 … +0x194    KNOWN host-configured parameter block
                   (HAL_SND_R2_Set_SHM_PARAM jump table, params 0x59…0xCF)
+0x198 …           further host fields (prior work: ~+0x17c commit, +0x180…+0x190,
                   +0x328…+0x338, …) — up to ~+0x338
…
+0x868             ?  32-bit, SET by 0xCF86  / CLEARED (→0) at 0xC8C2   [outside known block]
+0x86C             ?  32-bit, SET by 0xCC9B  / CLEARED (→0) at 0xCAAC   [outside known block]
+0x870 … +0x8B8    (trailing ~0x48 bytes, no identified accesses)
```

# 5. Evidence table

| Offset | Artifact | Evidence | Proposed meaning | Confidence |
|--------|----------|----------|------------------|------------|
| `+0x868` | `snd_full.reko/snd_full.h:1558` | `(A868 Eq_3563 tA868)` — auto name from address | **none** (name is address-derived) | — |
| `+0x86C` | `snd_full.reko/snd_full.h:1559` | `(A86C Eq_3938 tA86C)` — auto name from address | **none** (name is address-derived) | — |
| `+0x868` | `snd_full.bin` | 2 stores (`0xD15E` set, `0xC8C2` clear), 0 loads, `word32` | 32-bit published value | VERIFIED (existence/width) |
| `+0x86C` | `snd_full.bin` | 4 stores (`0xCE26/0xCE72/0xCF48` set, `0xCAAC` clear), 0 loads, `word32` | 32-bit published value | VERIFIED (existence/width) |
| SHM size | `utpa2k.ko` `0x4582F4` | `movw r2,#0x8b8` in `HAL_SND_R2_init_SHM_param` | SHM = 0x8B8 bytes | VERIFIED |
| region | derived | `0x868/0x86C < 0x8B8`, beyond `0x194`/`~0x338` | trailing region of SHM | VERIFIED (position) |
| semantics | — | no header/struct/define/map exists | **left blank** | UNRESOLVED |

*(Per instruction, "proposed meaning" is left empty where evidence does not support it.)*

# 6. Producer values

| | `0xCF86 → +0x868` | `0xCC9B → +0x86C` |
|---|---|---|
| value source | computed, stored at `0xD15E` | computed, stored at `0xCE26` / `0xCE72` / `0xCF48` |
| formula (R32/R33, not re-derived here) | `*(0x253364) + *(0x251194+0x114) + *(0x251194+0x118) + [ (+0x14) / (((+0x18)*(+0x20))*0x30) ]` | **same formula** |
| same formula? | **yes** — both helpers use the identical expression | |
| inputs | globals `0x253364`, `0x251194+0x114/+0x118`, and struct fields `+0x14/+0x18/+0x20` (of `0x252AD0` or `0x255CA8` per the R32 return branch) | |
| width | **32 bit** (`word32`) | **32 bit** (`word32`) |
| publication | store → `0xD4C8` flush+invalidate+`syncwritebuffer` (`r3 =` slot, `r4 = 4`) | identical |
| clear path | `0xC8C2` store 0 → `0xD4C8` | `0xCAAC` store 0 → `0xD4C8` |
| value kind | **not determined** — a 32-bit additive computed quantity; not provable as length/timing/rate/flag/descriptor | |

# 7. Architectural implication

R36 changes the *classification* of the two slots without changing the *consumer* outcome:
- **Positive:** they are now placed — real, 32-bit, mutable fields **inside** the SND R2 SHM, with a full
  set/clear publication lifecycle and cache maintenance. This strongly implies they are **meant to be observed
  externally**; a value that is written, flushed, and later cleared is not scratch state.
- **Negative for root-cause:** because they lie in the **trailing region beyond every host-configured field**, and
  because no host/kernel/DSP artifact reads them (R35), the intended observer is **not present in this workspace**.
  Their semantics therefore cannot be used to infer the observer, and **no link to DTS SDO / IEC61937 / SPDIF-HDMI
  TX can be drawn**.
- **Explicitly avoided:** "looks like a timing value ⇒ TX timing register", "written only by DTS ⇒ DTS enable",
  "published with flush ⇒ DMA descriptor". None of these is supported.

# 8. Next missing artifact (specific, ranked)

1. **The SND R2 SHM layout header** covering the **trailing region `0x860–0x8B8`** — i.e. the vendor struct /
   `#define` that names the fields after `+0x338`. This single artifact would name `+0x868`/`+0x86C` and identify
   their intended reader. (Not present: no `.h/.c/.map/.sym` for the R2 exists in the workspace.)
2. **The R2 linker map / symbol table for `snd_full.bin`** (or a non-address-named symbol dump), which would give
   real names instead of Reko's `A868`/`A86C`.

*(Runner-up, lower value: decoding the function that owns the clear sites `0xC8C2`/`0xCAAC` — it would show the
teardown/disable context but still not name the consumer.)*

---

## Appendix — Method notes (correction worth keeping)
- Two address encodings exist for the same location: **`base 0xA000 + 0x868`** and **direct `0x10000 − 0x5798`**.
  A scan that covers only one form yields false negatives. Both were required to reach 6/6 sites.
- Decompiler/SSA output (`*.reko/*.h`) is a useful **cross-check for complete access enumeration**, but its symbol
  names are address-derived and must not be treated as semantic.
- `0x8B8` SHM size came from an A32 `movw` immediate (`imm12 = 0x8B8`), not from a literal dword; searching raw
  bytes for the constant would fail.
