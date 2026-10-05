# SPACE BUNNY DTS/SPDIF — STATIC FOLLOW-UP

**Date:** 2026-09-24  
**Mode:** static/corpus-only, read-only  
**Parent report:** `forensic_space_bunny_full_corpus_audit_20260924_independent.md`  
**Scope:** non-TCL Space Bunny corpus  
**New runtime/patch/flash actions:** none  
**TCL/T615:** not searched, extracted, compared, or used

---

## 1. Purpose and result of this follow-up

This follow-up continues the static audit at the three boundaries that were least closed in the parent report:

1. the exact host-side `_MApi_AUDIO_SetDecodeSystem` dispatch and the `+0x10` value;
2. the symbolic base relationship between `F_03E0DD`, `F_0411A6`, `F_45DE0`, and F8/FC;
3. possible static producers or bridges for `0xB000001E` and the related DSP key/status words.

The parent report’s negative conclusion remains intact. This pass adds three material refinements:

- The host `+0x10 == 0xFF` claim is not universally “pass” or universally “kill”; the stock `_MApi_AUDIO_SetDecodeSystem` has a path-specific rejection branch, and `MI_AUDIO_Start` statically supplies `0xFF` to the decode-system structure.
- The F8/FC instance relation is not safely describable as `B+...` without resolving one context pointer. The direct relation is conditional on a pointer loaded from `[B+0x1414C]`.
- The `0xB000001E` producer is still not found, but the corpus now gives a sharper fork: SND uses the word in an E3/F3 request/ack protocol, while DEC also reads `0xB0000838/0xB000081C` as separate key/status fields. No direct edge between the ARM SE-IDMA stream and either destination is proven.

No remediation is proposed in this follow-up.

---

## 2. Host `MI_AUDIO_Open` → `MI_AUDIO_Start` → `_MApi_AUDIO_SetDecodeSystem`

### 2.1 What `MI_AUDIO_Start` passes as the first argument

The stock `mik.ko` symbols and raw ARM disassembly give this chain:

```text
MI_AUDIO_Open       @ 0xA256C
MI_AUDIO_Start      @ 0xA437C
MApi_AUDIO_OpenDecodeSystem  @ utpa2k 0x3F32EC
MApi_AUDIO_SetDecodeSystem   @ utpa2k 0x3F2FD4
_MApi_AUDIO_SetDecodeSystem  @ utpa2k 0x3E52A8
```

In `MI_AUDIO_Open` raw code:

```text
0xA29F4  call MApi_AUDIO_OpenDecodeSystem
0xA2A04  mov   r4,r0                 ; return value
...
0xA2AC0  str   r4,[r0,#0xADC]         ; store return in MI instance
```

Thus `MI_AUDIO_Start` later loads:

```text
0xA4850  ldr r0,[sb,#0xADC]
0xA4858  bl  MApi_AUDIO_SetDecodeSystem
```

The first API argument is structurally the stored return of `MApi_AUDIO_OpenDecodeSystem`, not the codec number `5` or `9`. The return may represent an audio-device/decoder handle or slot; the static corpus does not justify a stronger semantic name. The value is passed unchanged into the host API.

### 2.2 What `MI_AUDIO_Start` puts at decode-system `+0x10`

The `MI_AUDIO_Start` body writes:

```text
0xA44B4  mov  r0,#0xFF
0xA44BC  str  r0,[sp,#0x24]
```

The decode-system structure is later copied/initialized at `sp+0x60`, and the value from `sp+0x18` is stored to the structure’s `+0x10` slot at:

```text
0xA47E0  str  r1,[sp,#0x70]          ; sp+0x60 + 0x10
```

The call at `0xA4858` therefore receives a structure whose `+0x10` is statically `0xFF` on the shown path.

### 2.3 Exact `_MApi` branch behavior

Raw stock code at `_MApi_AUDIO_SetDecodeSystem`:

```text
0x3E534C  cmp   r4,#5
0x3E5350  cmnne r4,#1
0x3E5354  bne   0x3E53B8

; r4 == 5 or -1:
0x3E5358  ... alternate early path ...
0x3E5410  ... calls another path; does not take the direct 0x3E546C tail-call ...

; r4 != 5 and r4 != -1:
0x3E53B8  ldr   r0,[r5,#0x10]
0x3E53BC  cmp   r0,#0xFF
0x3E53C0  cmpne r0,#0
0x3E53C4  bne   0x3E5448

; 0 or 0xFF reaches the rejection/no-dispatch path at 0x3E53C8
; another value reaches the AV byte check at 0x3E5448
0x3E546C  b     0x3E546C              ; direct MDrv path
```

Therefore the correct static statement is:

```text
On the non-(5,-1) r4 path, +0x10 values 0 and 0xFF reject;
on the r4==5/-1 path, the +0x10 test is bypassed and a different early path is used.
```

The old universal claims “`+0x10=0xFF` always kills the gate” and “`+0x10=0xFF` always passes” are both `ARTIFACT/ERROR` as universal claims. The current evidence proves path structure, not the live value taken during a particular DTS playback.

**Status:** host branch structure `VERIFIED`; live branch selection `RUNTIME REQUIRED`.

---

## 3. `F_03E0DD` symbolic base analysis

### 3.1 Entry values

At `F_03E0DD @ 0x3E0DD`, the entry arguments from the `F_03E961` call are symbolically:

```text
O = F_03E961 entry r3
B = F_03E961 entry r4
Q = F_03E961 entry r5
```

The callee sets:

```text
0x3E0E4  r14 = O
0x3E0EE  r10 = B + 0x10000
0x3E0F1  r11 = B
0x3E0F7  r12 = Q
```

It then calls `0x3DF03` at `0x3E101`.

### 3.2 The pointer returned by `0x3DF03`

`0x3DF03` explicitly loads a context pointer into `r11`:

```text
0x3DF0E  r10 = r3 + 0x10000
0x3DF1B  r14 = [r10-0x5ED8]
0x3DF25  r13 = [r10-0x5ED4]
0x3DF29  r12 = [r10+0x4148]
0x3DF2D  r11 = [r10+0x414C]
...
0x3DF79  [r10+0x414C] = r11
```

With the `F_03E0DD` entry values, this is:

```text
r10_before_helper = B + 0x10000
r11_after_helper  = V = [B + 0x1414C]
```

The corpus contains no bounded direct static writer that proves `V == B`; the field is a runtime/context pointer. This is the missing value that invalidated an unconditional `B+0x7C4` claim in the parent report.

### 3.3 A108 and F8/FC-related construction

On the path reaching the later construction:

```text
0x3E19E  r23 = r11 + 0x7C4
0x3EA38/equivalent store path:
         [r10-0x5EF8] = r23
```

Therefore the exact static relation is:

```text
[B + 0xA108] = V + 0x7C4
r4_at_F_0411A6 = *(B+0xA108) = V+0x7C4
```

The later construction path sets `r14=0x10000` at `0x3E29F` and builds:

```text
P0 = V + 0x14EE8
P1 = P0 + 0x800 = V + 0x156E8
Dopen = V + 0x14CCC
T = V + 0x14E40
```

It also stores the two table values at addresses based on `r10`:

```text
[r10+0x4E40] = P0
[r10+0x4E44] = P1
```

But `r10` is still `B+0x10000` on the intended register flow, so those store addresses are:

```text
[B+0x14E40] = P0
[B+0x14E44] = P1
```

The table argument passed to `F_45DE0` is `T=V+0x14E40`, while the just-populated table address is `B+0x14E40`. They are identical only if:

```text
V == B
```

This is a concrete static identity condition, not a license semantic.

### 3.4 Relation to `F_0411A6`

`F_03E961` calls `F_0411A6` with `r3=r11=B` at `0x3EF62/0x3EF64`. In `F_0411A6`:

```text
0x41210  r11 = C + 0x10000
0x41275  r23 = 0x10000 | 0x4CCC
0x4127B  r10 = C + 0x14CCC
0x4127D  r4 = *(C+0xA108)
```

Thus, for the direct call path:

```text
D411 = B + 0x14CCC
r4   = *(B+0xA108) = V+0x7C4
```

`F_45DE0` initializes descriptor `Dopen=V+0x14CCC`; `F_0411A6` consumes descriptor `D411=B+0x14CCC`. The direct same-instance relation is therefore conditional on `V==B`.

**Static conclusion:**

```text
The F8/FC instance link is not proven universally.
It is algebraically reduced to the unresolved condition
[B+0x1414C] == B.
```

If the runtime context guarantees that pointer equality, the same-instance relation follows. That runtime/initialization fact is not present in the static corpus.

---

## 4. `0xB000001E` and related key/status words

### 4.1 DEC reader inventory

The authoritative DEC listing has eight `==5` comparison sites:

```text
0x138AE
0x13904
0x13D97
0x13DE5
0x21AC3
0x21B19
0x21F0E
0x21F5C
```

There are fourteen direct address constructions to `0xB000001E`; six are non-`==5` status reads. The four later sites are duplicated by earlier decoder/state-machine paths.

### 4.2 Direct writer search

Instruction-aware/raw scans of the checked stock artifacts found no coherent direct construction-and-store of `0xB000001E` in:

```text
patch_baseline/utpa2k_stock.ko
patch_baseline/mik_stock.ko
libutopia.so
audio.primary.mt5889.so
libmi3.so
dtv_driver.ko
MS12V22 DEC/SND images
armfw.bin
```

The raw `utpa2k` hits are in `.ARM.exidx`, `.data`, or symbol/relocation material, not direct code writers. The DEC raw hit at `0x178D41` is in a data-like table region. This does not prove that no indirect, hardware, table-mediated, or external-SE writer exists.

### 4.3 Data-driven SE co-writers do not close the bridge

`AudioInitTbl_0` contains mask/value records for `0x112A82..85`, and `HAL_AUDIO_WriteInitTable` dispatches them through the AbsWrite helpers. This is a real class of writers that a literal-only scan misses.

However, the checked init-table records do not contain a direct `0xB000001E` writer. The ARM/SE stream and the DSP `0xB000001E` word remain disconnected statically.

### 4.4 SND request/ack evidence

The SND dump around `ghidra_dump_C000_E00.txt:606–644` shows:

```text
0xC249  construct 0xB0000000
0xC24D  construct 0xB000002E
0xC253  write 0xE3 to 0xB000002E
0xC256  construct 0xB000001E
0xC259  read byte from 0xB000001E
0xC260–0xC264  loop until byte == 0xF3
0xC26E  clear 0xB000002E
```

This makes `0xB000001E` look like a status/handshake word in at least one SND path. It does not identify the producer of `0xF3`, nor prove that the ARM mask stream produces it.

### 4.5 Separate DEC key fields

DEC `r2_decoder_select` reads:

```text
0xB0000838 -> key1
0xB000081C -> key2
```

These are distinct from `0xB000001E`. No direct writer for either key field was found in the bounded static corpus. The ARM event stream from `0x112A82..85` is compatible with a possible external key delivery mechanism, but compatibility is not a data-flow proof.

### 4.6 Updated fork

```text
ARM/SE 0x112A82..85
       |
       +--> possible external delivery to 0xB0000838/0xB000081C [OPAQUE]
       |
       +--> independent status/handshake word 0xB000001E [producer NOT FOUND]
```

The strongest surviving statement is a fork, not a closed chain.

---

## 5. Corrections to the parent report

| Parent statement | Follow-up correction |
|---|---|
| `+0x10=0xFF` universally passes or kills the host gate | Both universal forms are wrong. `MI_AUDIO_Start` supplies `0xFF`; `_MApi_AUDIO_SetDecodeSystem` rejects it only on the non-(5,-1) `r4` path, while 5/-1 takes a different early path. |
| `r4` is an opaque AU slot/codec value | The static first argument is the stored return of `MApi_AUDIO_OpenDecodeSystem`; exact live handle/slot semantics remain unproven. |
| `r4=B+0x7C4` on traced DEC paths | More precise: `r4=V+0x7C4`, where `V=[B+0x1414C]` after `0x3DF03`; equality to B is unresolved. |
| F8/FC same-instance link is fully proven or fully withdrawn | The relation is reduced to `V==B`; the table is stored at `B+0x14E40` but passed as `V+0x14E40`. |
| Eight B000001E comparisons | Confirmed. Four additional earlier comparisons were missing from the four-site inventory. |
| No direct writer found | Still true only as a bounded result. Data-driven writers and external/hardware mechanisms must remain open. |

---

## 6. Independent static graph after this pass

```text
MI_AUDIO_Open
  └─ MApi_AUDIO_OpenDecodeSystem return
       └─ [MI instance + 0xADC]
            └─ MI_AUDIO_Start passes it as first API argument
                 └─ decode-system struct +0x10 = 0xFF
                      └─ _MApi_AUDIO_SetDecodeSystem
                           ├─ r4 == 5/-1: alternate early path
                           └─ other r4: +0x10 == 0/0xFF rejects;
                                       other value continues to AV-byte check
                           [live branch still RUNTIME REQUIRED]

DEC initialization:
  B = outer context passed through F_03E961
  V = [B+0x1414C] loaded by F_03DF03
  [B+0xA108] = V+0x7C4
  F_0411A6 descriptor = B+0x14CCC
  F_03E0DD descriptor/table = V+0x14CCC / V+0x14E40
  table values stored at B+0x14E40
  same-instance condition: V == B

ARM SE-IDMA:
  0x112A82..85 software event stream [VERIFIED]
  host mapped stores [VERIFIED]
  SE sampling / external key delivery [OPAQUE]

DSP:
  0xB0000838/081C key readers [VERIFIED consumers]
  0xB000001E eight ==5 gates + SND E3/F3 handshake [VERIFIED consumers]
  writers/producers [NOT FOUND in bounded corpus]
```

---

## 7. Remaining static questions

1. Where is `[B+0x1414C]` initialized, and can static initialization prove `V==B`?
2. Does the valid runtime path guarantee the table store at `B+0x14E40` and the `F_45DE0` argument `V+0x14E40` are the same object?
3. What exact return values does `MApi_AUDIO_OpenDecodeSystem` produce for the relevant MI instances?
4. Which live `r4` branch does DTS START take?
5. Is `0xB000001E` supplied by an external SE core, a hardware handshake, or an indirect mailbox helper?
6. Is there a data-driven table or indirect helper that writes `0xB0000838/081C`?
7. Does the F path hand its A/B buffers to any queue/DMA/SND function after `F_4618E`?

These are static/runtime boundaries, not patch instructions.

---

## 8. Integrity

- This is a new follow-up report.
- No runtime, ADB, device, playback, patch, flash, module reload, settings, or reboot action was performed.
- No TCL/T615 evidence was used.
- The parent report and all historical reports remain unchanged.
- The earlier 20260924 report remains the authoritative full audit; this document records only the incremental static closure and corrections above.
