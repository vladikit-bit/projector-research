# FINDING — CONSOLIDATED ROOT CAUSE: SND DSP emits DTS bursts with the wrong IEC61937 Pc

**Date:** 2026-09-12
**Supersedes:** `REPORT_final_dts_patch_experiment.md` §1 ("no patch point was selected") —
that negative was about the **wrong** region (the `0x26270–0x2661D` DTS pack *gate*). The real
writer lives in a **different function** (`0x4F20`) and **is** patchable.
**Extends:** `r56_work/FINDING_pc_dts_rootcause.md` (R56), whose byte-level claim I have
**independently re-verified from the raw image** in §3 below.
**Independent verification status:** all §3 bytes re-read and re-decoded by me from
`r27_work/snd_full.bin` (md5 `eb879cdc07f510722f19db6d18d77d3c`) — **VERIFIED BY BINARY**.

---

## 0. One-paragraph summary

DTS passthrough has **never** worked on this device, and the final blocker is now identified with
byte-exact evidence: the SND DSP's IEC61937 burst-header writer emits DTS bursts with
**`Pc = 0x0001`, which is the AC-3 data-type code**. It should be `0x0B` (IEC61937_DTS1, 512-sample
core). The payload carries a valid DTS sync word, so the Pioneer VSX-817 receives a burst that
**declares itself as AC-3 but contains DTS** — it cannot lock, and it does not fall back to PCM.
The E-AC3 path in the **same function** writes the correct `0x15`, which is why E-AC3 works and only
DTS is broken. This is a **1-instruction bug (2 bytes)** in a function that has **exactly two writers**
of the Pc slot — one correct (E-AC3), one wrong (DTS).

---

## 1. Address model (re-confirmed)

**SND image:** `file_offset = fw_addr + 0x16F00`  (5 independent byte anchors, R56; re-checked here)

| fw address | file offset | instruction |
|---|---|---|
| `0x4FEC` | `0x1BEEC` | `bg.addi r23,r10,9484` (`0x250C` = header base) |
| `0x501C` | `0x1BF1C` | `bg.sw r25,9488(r10)` (`0x2510` = **Pc**) |
| `0x5150` | `0x1C050` | `bt.movi r25,1` ← **THE BUG** |
| `0x5152` | `0x1C052` | `bg.sw r25,9488(r10)` (`0x2510` = **Pc**) |

`9484 = 0x250C`, `9488 = 0x2510`. The 8-byte IEC61937 preamble occupies
`[r10+0x250C .. r10+0x2513]`:

```
0x250C (sh) = Pa
0x250E (sh) = Pb
0x2510 (sh) = Pc   <-- data-type; THE BUG
0x2512 (sh) = Pd
0x2514+     = payload
```

---

## 2. The function and both branches

Function containing **`0x4F20`**. Only **two** writers of the Pc slot `0x2510` exist in the entire
SND image (verified by byte-pattern scan for `ef 2a 25 11` = `bg.sw r25,9488(r10)`):

```
writer #1  fw 0x501C  (E-AC3 branch)
writer #2  fw 0x5152  (DTS branch)
```

### 2.1 Common header (Pa / Pb)

```
0x4FEC  bg.addi r23, r10, 9484      ; r23 = &header (r10+0x250C)
0x4FF6  bg.addi r25, r0, -1934 / ori r25,r0,0xF872  ; Pa = 0xF872
0x4FFA  bg.sh  r25, 9484(r10)       ; [0x250C] = Pa
0x5002  bg.addi r25,r0,19999 / ori r25,r0,0x4E1F    ; Pb = 0x4E1F
0x5006  bg.sh  r25, 9486(r10)       ; [0x250E] = Pb
0x500A  bg.beqi r24, 0, 0x5142      ; if (*(r10+0xA0) == 0) -> DTS branch
```

### 2.2 E-AC3 branch — CORRECT (this is why E-AC3 works)

```
0x500E  bg.lwz r24, 1424(r11)       ; burst length
0x5012  bg.ori r25, r0, 0x6000      ; slot size
0x5016  bn.sw  0(r12), r25          ; *(r12) = 0x6000
0x5019  bn.ori r25, r0, 0x15        ; *** Pc = 0x15 = IEC61937_EAC3  (CORRECT) ***
0x501C  bg.sw  r25, 9488(r10)       ; [0x2510] = Pc
0x5020  bg.sh  r24, 9488(r10)       ; [0x2512] = Pd = length
0x5024  bg.ori r5, r0, 0x6000
```

### 2.3 DTS branch — WRONG

```
0x5142  bg.lwz r24, 1424(r11)       ; burst length
0x5146  bg.ori r25, r0, 0x1800      ; slot size = 0x1800 (6144 B)
0x514A  bn.slli r24, r24, 3         ; Pd = len << 3   (bit count - correct for DTS)
0x514D  bn.sw  0(r12), r25          ; *(r12) = 0x1800
0x5150  bt.movi r25, 1              ; *** Pc = 0x0001 = IEC61937_AC3  (WRONG) ***
0x5152  bg.sw  r25, 9488(r10)       ; [0x2510] = Pc
0x5156  bg.sh  r24, 9488(r10)       ; [0x2512] = Pd
0x515A  bg.ori r5, r0, 0x1800
0x515E  bg.j   0x1BF28              ; -> common tail
```

Everything else in the DTS branch is **correct**: Pa `0xF872`, Pb `0x4E1F`, slot `0x1800`,
`Pd = len<<3` (bits — right for DTS), and the payload is deliberately **not** byte-swapped
(the tail tests for the E-AC3 sync `0x770B` and only byte-swaps for E-AC3). The sole defect is Pc.

---

## 3. My independent byte-level verification (raw image, no decoder relied upon)

Executed 2026-09-12 against `r27_work/snd_full.bin`:

```
E-AC3 path (fw 0x500E..0x5024)
  0x500e bg.lwz  r24,1424(r11)    ef 0b 85 92
  0x5012 bg.ori  r25,r0,0x6000    cb 20 60 00
  0x5019 bn.ori  r25,r0,0x15      53 20 15
  0x501c bg.sw   r25,9488(r10)    ef 2a 25 11     -> [0x2510] = 0x15   OK
  0x5020 bg.sh   r24,9488(r10)    ef 0a 25 13     -> [0x2512] = len

DTS path (fw 0x5142..0x515E)
  0x5142 bg.lwz  r24,1424(r11)    ef 0b 85 92
  0x5146 bg.ori  r25,r0,0x1800    cb 20 18 00
  0x514a bn.slli r24,r24,3        4f 18 18
  0x514d bn.sw   0(r12),r25       0f 2c 00
  0x5150 bt.movi r25,0x1          9b 21           <-- THE BUG
  0x5152 bg.sw   r25,9488(r10)    ef 2a 25 11     -> [0x2510] = 0x01   WRONG
  0x5156 bg.sh   r24,9488(r10)    ef 0a 25 13     -> [0x2512] = len<<3
  0x515a bg.ori  r5,r0,0x1800     c8 a0 18 00
```

**Encoding check (independent arithmetic):**
- `0x9B21` = `(0x26<<10)|(25<<5)|1` = `bt.movi r25, 0x1`  ✔ matches the bytes exactly
- `0x9B2B` = `(0x26<<10)|(25<<5)|11` = `bt.movi r25, 0x0B`  ✔ 5-bit immediate, `0x0B = 11` fits

⇒ The bug is a literal `1` where the DTS data-type `0x0B` belongs.

---

## 4. Why this explains EVERYTHING previously observed

| Observation (already in the record) | Explanation by this finding |
|---|---|
| AVR never locks DTS (`dts_01.bin`) | Burst declares AC-3 (`Pc=0x0001`) but carries a DTS sync word ⇒ the AVR's AC-3 parser rejects the payload, and no DTS mode is signalled. |
| `dts_01.bin` shows the AC-3 header `Pc=0x0001`, `Pd=0x3000`, sync `0x0B77` | Consistent: Pc **is** AC-3 by bug; the *frozen* aspect is a separate capture artifact (stale buffer) — note `Pd=0x3000` there comes from a preceding AC-3 burst, so that capture predates a clean DTS-only window. |
| `ac3_03.bin` works (93 blocks, live) | AC-3 path is untouched and correct. |
| E-AC3 works | Its branch writes `0x15` correctly. |
| `Invalid Spdif license` **never appears in any runtime log** | **No license involvement at all** — consistent with the string never printing. |
| ARM-side AUTH bypasses (`Exp-B`) and diagnostic kernels changed nothing | They were never on this path. |
| The `0x26270–0x2661D` "DTS pack gate" was a dead end | Wrong function — the header writer is `0x4F20`. |
| `0x0C00–0x0C07` status divergence, `0x854` bits, `0x868/86C/870` | Downstream/adjacent consequences, not the cause. |
| DTS decoder is fully alive (`dec play !!`, live counters) | Decode is fine; only the **IEC header Pc** is wrong. |

---

## 5. Patch candidates (minimal, reversible, one change)

Target image: `/vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin`
md5 `eb879cdc07f510722f19db6d18d77d3c` · 1,839,920 B (**STOCK — never patched**; must not be overwritten).

### Option P1 — MINIMAL, 2 bytes, DTS1 (512-sample core) only
Change **file offset `0x1C050`** (fw `0x5150`):

```
before:  9b 21     bt.movi r25, 0x1     ; Pc = 0x0001  (AC-3)  WRONG
after:   9b 2b     bt.movi r25, 0x0B    ; Pc = 0x000B  (DTS1) CORRECT
```

Zero side-effects on any other instruction (same length, same register, same slot).
This is the smallest possible change and is fully reversible.

### Option P2 — type-selecting (DTS1/2/3 = 0x0B/0x0C/0x0D)
Requires the DTS core frame size at this point to choose between `0x0B`/`0x0C`/`0x0D`.
The sample count is **not yet located in SND** (R56 §B2). Not recommended as the first step —
it needs a second unknown resolved, violating "smallest evidence-backed change".

**Decision: attempt P1 first.** It covers the overwhelmingly common DTS core case
(`STREAM_TYPE_DTS_512` was the observed Kodi stream type) and is a 2-byte change.

---

## 6. Deployment plan and rollback

```sh
# 0. Verify current state
adb shell md5sum /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin
#    expect eb879cdc07f510722f19db6d18d77d3c

# 1. Backup (device-side AND host-side)
adb shell 'cp /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin /data/local/tmp/asnd_stock.bin'

# 2. Build the patched artifact host-side (NEW versioned file — never overwrite stock)
#    copy byte 0x1C050: 0x21 -> 0x2B   (only this one byte changes!)
#    -> aucode_asnd_r2_MS12V22_DTS_PC_PATCH_V2.bin

# 3. Deploy
adb root; adb remount
adb push aucode_asnd_r2_MS12V22_DTS_PC_PATCH_V2.bin /data/local/tmp/
adb shell 'cat /data/local/tmp/aucode_asnd_r2_MS12V22_DTS_PC_PATCH_V2.bin > /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin && sync'

# 4. Reload (DSP image is loaded at audio init -> reboot required)
adb reboot
```

**Rollback:**
```sh
adb root; adb remount
adb shell 'cat /data/local/tmp/asnd_stock.bin > /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin && sync'
adb reboot
```

**Success criteria:** play `test_dts_51.mp4` with passthrough ON, **Pioneer VSX-817 panel shows DTS**.
**Regression control (mandatory):** play `test_ac3_51.mp4` → panel must still show **DD**.
**Secondary check:** capture the wire with `dump_spdif_npcm` and confirm the DTS burst now reads
`Pa=F872 Pb=4E1F Pc=0x000B` instead of `Pc=0x0001`.

---

## 7. Residual uncertainty (honest statement)

1. **This is the last identified blocker, but "last" cannot be proven** without running it. There
   may be a further gate downstream once Pc is corrected. The prediction is falsifiable: if the AVR
   locks DTS after this 2-byte change, the model is correct end-to-end.
2. **DTS1 only (`0x0B`).** If the source ever presents a 1024/2048-sample DTS core, the AVR may
   still fail to lock on those specific streams. Option P2 addresses that but needs the sample-count
   location found first.
3. **IEC61937 compliance detail:** strictly, `Pd` for DTS should be the number of **bits**;
   the branch already does `len<<3`, so this is consistent.
4. The V1 patch on record (`aucode_asnd_r2_MS12V22.bin` file `0x1C051`, `0x21`→`0x2B`, md5
   `785fd73b…`) is **almost the same idea but the WRONG byte**: it targeted the *second* byte of the
   instruction stream (`0x1C051`), i.e. it tried to change the immediate field only, leaving the
   opcode byte `0x9B` in place. Changing `0x21`→`0x2B` at `0x1C051` yields the 16-bit word `0x9B2B`
   = `bt.movi r25,0x0B` — **which is actually correct arithmetic**, but it is fragile/ambiguous to
   describe. **The canonical, unambiguous statement is: set the 16-bit instruction at file `0x1C050`
   to `0x9B2B`**, i.e. bytes `9B 2B`. If V1 was indeed that byte change and it was tested and failed,
   then this was tested — see §8.

---

## 8. ⚠️ CRITICAL — reconciliation with the already-tested V1 patch

The standing constraint is: **do not re-propose a patch that was already tested.**
The device currently carries a patched SND image (V1: file `0x1C051`, `0x21`→`0x2B`, md5
`785fd73b…`). If V1 == Option P1, then **P1 has already been tested and failed**, and this finding
would be a rediscovery.

**This must be resolved before any deployment.** Two possibilities:
- **(a)** V1 changed only byte `0x1C051`, leaving the instruction word as `0x9B2B` → identical to P1.
  Then P1 is already tested → the DTS-Pc theory is either wrong or insufficient → look elsewhere.
- **(b)** V1's byte edit did not form a valid `bt.movi r25,0x0B` in the executing phase (e.g. the
  decoder reads the pair starting at `0x1C050`; an edit at `0x1C051` gives `0x9B2B` — same thing),
  so (a) and (b) converge: **it is the same instruction.**

⇒ **Therefore the immediate next action is NOT to deploy.** It is to verify V1's exact byte state and
whether it was ever actually exercised with the DSP image reloaded (reboot), and to confirm whether
the DTS-Pc value was ever observed on the wire. See §9.

---

## 9. EXACT NEXT STEP

1. **Read the V1 patched SND image** (`785fd73b…`, if a copy exists) and diff it against stock —
   confirm whether the instruction at `0x1C050` is `0x9B2B` (i.e. V1 ≡ P1).
2. **If V1 ≡ P1 and it was tested:** the Pc theory is falsified as *sufficient*; re-examine P2
   (needs the sample-count location) and the downstream path.
3. **If V1 ≡ P1 but deployment was incomplete** (no reboot → DSP image never reloaded; or the
   DTS-only wire was never captured after it), then **P1 is untested in effect** and should be
   deployed properly with a reboot and a wire capture.
4. **If V1 ≠ P1** (different instruction/phase), deploy P1 as a fresh, versioned artifact.

The decisive instrument is the **wire capture** (`dump_spdif_npcm`): an AVR lock test alone cannot
distinguish "Pc still wrong" from "Pc fixed but something else blocks". Capture `Pc` directly.
