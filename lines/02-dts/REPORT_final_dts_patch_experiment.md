# REPORT — Final DTS patch experiment

**Device:** Thundeal TD98 Pro (MStar MT5889, AEON DSP)
**DSP firmware target:** `/vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin`
**Local analysed copy:** `aeon_validate/r27_work/snd_full.bin`
**MD5:** `eb879cdc07f510722f19db6d18d77d3c` · **Size:** 1,839,920 B
**Date:** 2026-09-11
**Scope:** STEP 1 (patch-point selection) → STEP 7 (honest stop). Steps 3–6 not reached.

---

## 1. Selected patch point

**None. No patch point was selected.**

STEP 1 was executed to completion and returned a **negative**: there is no single
firmware instruction on the DTS output path whose modification can restore DTS passthrough,
because **no firmware-writable node on the live DTS output path reaches an output/SDO/TX
arm action.**

This is not an "I could not find one" result. It is a *proved* negative, established three
independent ways in §2: (a) the candidate gate's predicate is the wrong polarity and is
never taken at runtime, (b) the candidate gate's TRUE path is a **function return**, not an
output arm, and (c) the block's only consequential input is a hardware status register with
**zero firmware writers**.

Every previously-leading candidate was re-examined and eliminated; see §1.1.

### 1.1 Candidate elimination table

| Candidate | Source phase | Why rejected |
|---|---|---|
| `0x854` bits 17\|19 (AC3-set / DTS-clear) | R46d/R47e | R49-B: the `CC9B→P2→CB50` triplet is a **shared clear/re-read/conditionally-set latch re-arm**, not a codec selector. R46d §D4.3 explicitly: *"Do not patch `0x854` yet."* Runtime: reads `0x000000` in 20/20 samples for **both** codecs. |
| `0x25EE1` gate | R46/R47 | All 4 callers branch into a **telemetry status write** (`0x252AD0`/`0x255CA8`); callee `0x13292` is a generic arithmetic helper. R47e: **downstream consequence-validator**, not a TX arm. |
| `0x1F6DA` / `mem[0x8(r10)]` | R50/R51/R52 | R51-C: `0x1F575` has **no static caller** (single ref = pointer-table entry `0x12AF20`). R52: dispatcher **not statically locatable** ⇒ `struct+0x8` is framework/runtime state. **Provenance additionally retracted by the user.** |
| `0x868` / `0x86C` / `0x870` | R46b §21.3 | **No functional firmware reader** — only a straight-line telemetry dump `0xE60C–0xE702`. A change here cannot be observed. |
| The four `0xCB50`-gating branches (`0x19494`, `0x1EDFE`, `0x1F760`, `0x26601`) | R48-A | R49-B shows each gates the **same state reconciler**, not the codec decision. Newly traced in R54: `0x26601` is the DTS one and is **never taken** (§2.1). |
| `0x814` bit21 guard | R29/R46 | **0 firmware writers**; runtime reads `0x000000` in 20/20 samples for both codecs. |
| `0x0C00–0x0C07` status block | R47/R53 | **0 firmware stores** (Ghidra-authoritative). Hardware-produced. |

---

## 2. Evidence

### 2.1 The DTS output-pack path is `0x26270–0x2661D`, gate at `0x265F3`

New this phase (**R54**): Ghidra flow-following decode of
`0x26270–0x26620`, script `scripts/R54_TracePacker.java` + `scripts/R54b_FollowPack.java`,
raw dumps `r54_work/r54_full.txt`, `r54_work/r54b_full.txt`, synthesis
`r54_work/REPORT_r54_decisive_trace.md`.

The function entry is `0x26270` (`bn.addi r1,r1,-0x48`), return `0x2646B` (`bt.jr r9`).
(`getFunctionContaining(0x265F3)` returns nothing in the `-noanalysis` project — R49 §2.4
recorded the same, recovering the entry from the prologue.)

**The DTS sub-block, entered from `0x2644A  bg.beqi r14,0x1,0x26574`:**

```
0x26574  bn.lwz r23,0x3c(r10)          ; slot A
0x26577  bg.bnei r23,0x0,0x2644e       ; slot A != 0 -> exit
0x2657B  bn.lwz r23,0x44(r10)          ; slot B
0x2657E  bg.ori r25,r0,0xbb80          ; 48000
0x26582  bg.bgts r23,r25,0x2644e       ; slot B > 48000 -> exit
0x26586  bn.lwz r23,0x60(r10)          ; slot C
0x2658D  bn.bnei r23,0x0,0x265af       ; slot C != 0 -> other-helper path
0x265AF  bg.addi r3,r12,0x6ad4         ; r3 = ctx
0x265B3  bt.mov r11,r3
0x265B5  bg.jal 0xc49d                 ; PREDICATE A #1 (pre-CC9B guard)
0x265BC  bg.bnei r3,0x0,0x26773        ; if bits17|19 SET -> skip whole block
...
0x265F3  bg.jal 0xc49d                 ; PREDICATE A #2 (the gate in question)
0x265F7  bt.mov r14,r3
0x265FD  bg.jal 0xcc9b                 ; DTS-clear helper
0x26601  bg.bnei r14,0x0,0x26451       ; if predicate TRUE -> SKIP CB50  <-- the branch
0x26605  bn.lwz r23,0x14(r11)          ; size
0x26608  bg.sflesi r23,0x2fff
0x2660C  bn.bf 0x26451                 ; size > 0x2FFF -> SKIP CB50
0x26615  bg.jal 0xcb50                 ; AC3-set helper
0x2661D  bg.j 0x2648e
```

This is the **only** site in the image where the `CC9B→CB50` pair sits on a format-20
(DTS) path guarded by the `0xB000_0814` byte test at `0x2644A`, and it is exactly the site
whose gate the user named.

### 2.2 The gate's TRUE path is a function RETURN, not an output arm

```
0x26451  bn.bnei r13,0x0,0x2648e       ; join target of the DTS sub-block
0x26454  bt.movi r3,0x0                ; <-- COMMON EPILOGUE
0x26456  bn.lwz  r9,0x44(r1)
...
0x2646B  bt.jr   r9                    ; RETURN
```

`0x26451` is the join point. On the DTS path `r13 == 0`, so control falls into the **common
epilogue and returns**. The predicate's TRUE result therefore does not merely skip a
register write — **it exits the entire packer function**, bypassing the frame-builder at
`0x262A3`–`0x2632C` (reached only via `0x26480`/`0x2648A`, i.e. only when `r13 != 0`).

### 2.3 The predicate is already FALSE at runtime — the branch is never taken

`0x265F3` reads `0xB000_0854` bits 17|19. Here the ordering is **P2 first (`0x265F3`),
then CC9B (`0x265FD`)** — the "read the entry latch" group of R49 §3's cross-comparison
table, same as the AC3 caller `0x1EDF0`.

Runtime (`r47_work/r47_capture_{ac3,dts}.log`, 20 samples each):

```
DM[0x0854] = 0x000000     AC3: 20/20 zeros    DTS: 20/20 zeros
DM[0x0814] = 0x000000     AC3: 20/20 zeros    DTS: 20/20 zeros
DM[0x0858] = 0x000000     AC3: 20/20 zeros    DTS: 20/20 zeros
DM[0x086c] = 0x000000     AC3: 20/20 zeros    DTS: 20/20 zeros
DM[0x0870] = 0x000000     AC3: 20/20 zeros    DTS: 20/20 zeros
```

The latch is never set, so the predicate is **already FALSE** and `0x26601` is **never
taken**. Replacing it with a NOP, or with an unconditional branch, changes **nothing**.

### 2.4 The real divergence is upstream, in hardware status

Runtime `r47_work/r47_dense_{ac3,dts}.log`, 20 samples each, on the hardware output-status
block:

| Register | AC3 | DTS |
|---|---|---|
| `0x0C00` | **33 distinct non-zero values** (`0xdb6f00`, `0xad6b00`, `0x9f3500`, `0x24f900`, …) | **0 non-zero** |
| `0x0C01`–`0x0C05` | non-zero in 16 of 20 samples | **0** |
| `0x0C06` | non-zero in 4 samples; **byte0 bit7 SET in 3** (`0xad`, `0xdb`, `0xe3`) | **0** |
| `0x0C07` | 4 non-zero samples (`0xf3e700`, `0xb20000`, `0x8f1e00`, `0x3e7c00`) | **0** |
| `0x0FE8` (DEC liveness) | `0x014`, `0x020`, `0x004`, `0x002` | `0x018`, `0x020`, `0x01e`, `0x014`, `0x012` |

Two facts fall out:

1. **The `0x25EE1` gate's first condition — `bit7(byte@0x0C06)` — is satisfied during AC3
   (3/20 samples) and never during DTS (0/20).** The gate is not the cause; it is a
   downstream consequence-validator reading a hardware register that the DTS path never
   populates. This **quantitatively confirms R47e's "symptom, not cause" verdict.**
2. **The DTS decoder is alive** — `0x0FE8` is non-zero throughout DTS playback — while
   the hardware output status stays at zero. The audio is decoded but the transmit-side
   status never populates.

And `0x0C00–0x0C07` has **0 firmware stores** (Ghidra-authoritative; `r53d_status.txt`,
0 stores / 10 loads). Firmware cannot write it.

### 2.5 Conclusion of the evidence chain

> The AC3/DTS divergence originates in **hardware state** (`0x0C00–0x0C07`, and `0x0814`
> which also has 0 firmware writers). The only firmware-writable correlating field (`0x0854`
> bits 17|19) is a **shared state-normalization latch** (R49-B), invisible to the host
> (20/20 zeros for both codecs). The DTS packer sub-block that reads it is guarded by a
> predicate that is **already FALSE**, and its TRUE path is a **function return**, not a
> TX arm.

---

## 3. Exact binary diff

**No bytes were changed.**

| Item | Value |
|---|---|
| Source path | `aeon_validate/r27_work/snd_full.bin` |
| Source MD5 | `eb879cdc07f510722f19db6d18d77d3c` |
| Source size | 1,839,920 B |
| Output path | *(none — no patched artifact created)* |
| Output MD5 | *(n/a)* |
| VA / file offset | *(n/a)* |
| Old instruction bytes | *(n/a)* |
| New instruction bytes | *(n/a)* |
| Total changed bytes | **0** |

`r27_work/snd_full.bin` was re-hashed after this phase and is **byte-identical** to its
pre-phase state (`eb879cdc…`). No versioned copy was needed because no modification was
made; the original was never at risk.

---

## 4. Deployment verification

**Not performed — no patch existed to deploy.**

- Device untouched this session.
- No `/vendor` remount, no file transfer, no reboot.
- No backup was required (nothing to back up *from*); the active on-device firmware
  `/vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin` remains byte-identical to
  `r27_work/snd_full.bin` as established in the baseline step.

This is the correct outcome: deploying an unverified or pointless binary would have
violated the user's explicit *"Do not deploy unverified binary"* and
*"Prefer enabling an already-existing DTS output path over implementing new logic"* rules.

---

## 5. Test results

**Not performed — no patch to test.**

No AC3 regression run, no DTS test run. The R47 runtime captures from the prior phase
(quoted in §2.3/§2.4) were re-analysed as **existing evidence**, not re-collected. The
device was not disturbed.

---

## 6. Final verdict

# **PATCH TARGET INVALID**

The attempted patch was abandoned at STEP 1 because no firmware-writable node on the DTS
output path reaches an output/SDO/TX arm action. Specifically:

1. The named gate `0x265F3` on the DTS packer path tests `0x854` bits 17|19, which are
   **already FALSE** at runtime in both codecs. Its guarded branch `0x26601` is **never
   taken** — patching it is a no-op.
2. Even if forced, its TRUE path (`0x26601` → `0x26451` → `0x26454`) is a **function
   return**, not an output arm. It skips the frame builder rather than enabling anything.
3. The block's only consequential input — the hardware output-status block
   `0x0C00–0x0C07` — has **0 firmware stores** and is **all-zero during DTS** while
   populated during AC3. Firmware cannot write it.
4. Every other candidate (`0x854` bits 17|19, `0x25EE1`, `0x1F6DA`, `0x868/0x86C/0x870`,
   `0x814` bit21, the four `0xCB50`-gating branches) was independently eliminated.

**The DTS-passthrough failure is not a firmware control-flow defect in the output/pack
path. It is a hardware-side or upstream-configuration condition that the DSP firmware
assembled here neither decides nor can influence.**

### 6.1 Exact next blocker (no new broad research phase)

The single concrete blocker revealed by this experiment:

> **The transmit-side hardware status block `0xB000_0C00–0x0C07` never populates during DTS
> playback (0/20 non-zero) while it populates during AC3 (33 distinct values on `0x0C00`
> alone), even though the DTS decoder is demonstrably alive (`0x0FE8` non-zero throughout).
> The DSP firmware has zero stores into that block, so the condition that arms the hardware
> digital transmitter for DTS is set by something the DSP firmware does not control.**

To progress, the next investigation must leave the "patch the DSP firmware `.bin`" strategy.
The two remaining concrete options — neither of which is authorized or started here:

- **(A) Trace the hardware-adjacent producer.** Determine which hardware block (the
  demodulator/DEC-side or the SDO/TX engine) drives `0xB000_0C00–0x0C07`, and why DTS
  decoding does not trigger it. This requires the DEC image or the register-level hardware
  contract, not the SND image.
- **(B) Re-examine the currently-patched components.** Per the baseline step, `mik.ko`,
  `utpa2k.ko`, and `libmi3.so` are **already patched** (non-stock MD5s) while the DSP
  firmware and `libutopia.so` are stock. Since the DSP firmware is now excluded as the
  fault locus, the active experimental state in those three components is the
  higher-probability cause of the current DTS failure — and reverting them to their
  `.orig` backups is a cheap, fully reversible experiment worth running before any new
  static work.

### 6.2 What was NOT done, and why

- No binary was modified. No patch artifact was created. No backup was needed.
- No deployment, no reboot, no device change.
- No new broad research phase was launched, per the instruction
  *"If not fixed, do not launch another broad research phase; state the exact next blocker
  revealed by the experiment."* §6.1 states it.

---

## 7. FOLLOW-UP (2026-09-12) — corrected after user testimony

### 7.0 ⚠️ RETRACTION FIRST

An earlier draft of this section claimed a "regression": that `libmi3.so` md5 `2e34d0c9…` had made
DTS work and a later `4c199a5c…` broke it. **That claim is WITHDRAWN.**

It rested entirely on `REPORT_PASSTHROUGH_FIX.md` §3 ("DTS mp4 → pass-through; HAL parser streaming
DTS frames (size 2012) continuously"). The user has stated that **that report is WRONG — the sound
was only transcoding**. Therefore:

- There is **no evidence that DTS passthrough ever worked** on this device, with any `libmi3` patch.
- The "streaming DTS frames (size 2012)" line was, on the user's authority, **not real DTS
  bitstream** — consistent with the transcoded/AC-3 or PCM outcome observed every time since.
- **No regression can be attributed** to `4c199a5c` (or to `2e34d0c9`). The comparison is void.
- This matches the user's earlier statement: *"А от чистого дтс ще не було"* — clean DTS has never
  been achieved.

Anything below that depended on that report is **struck**. What survives is listed in §7.1.

### 7.1 What still stands (independent of the discredited report)

| fact | source | status |
|---|---|---|
| Under DTS the SPDIF TX carries a **frozen AC-3 header repeated 26×** (only 3 unique blocks) | fresh wire capture `dts_01.bin` | VALID |
| Under AC-3 the SPDIF TX carries a **live stream**, 93 blocks | wire capture `ac3_03.bin` | VALID |
| The DEC image contains a **complete, correct DTS IEC61937 writer** that is **never invoked** | Ghidra disassembly | VALID |
| Capability probe `*(u16*)0xB000001E == 5` else error if `*(s8*)0xB000001D <= -1`, at **4 sites** (`0x21AC3`, `0x21B19` [DTS branch], `0x21F0E`, `0x21F5C`) | Ghidra ground truth | VALID |
| `0xB000000A` / `0xB000001D` / `0xB000001E` are the only MMIO registers in the gate window; the only firmware **writers** in the image are of the internal state field `0xec(r10)` (`0x013DD1–0x013F85`) | `dec04/dec05_out.txt` | VALID |
| The active `/vendor/lib/libmi3.so` (`4c199a5c…`) differs from both stock (`c2b5c57d…`) and the other build (`2e34d0c9…`) at file `0x06163E` in `MI_AUDIO_GetCaps`; its mask is `0x720002E1` and it zeroes `r4` | byte-exact diff | VALID **as fact** — see §7.2 |

### 7.2 Corrected interpretation of the `libmi3.so` state

The active patch has two real defects (wrong capability mask `0x720002E1` instead of `0x2E0`;
`r4` clobbered at `00 24`). But with the "it used to work" premise removed, the honest statement is:

> The active `libmi3.so` is a **suspect, unverified patch** whose capability word is malformed.
> It has **never been shown** to deliver clean DTS. It should be replaced by a correct patch (or by
> stock) and re-tested — **not** described as a regression that broke a working state.

A correct mask for the codecs of interest is `0x2E0` (bits 5,6,7,9 = AC3/E-AC3, TrueHD,
AAC family, DTS/DTS-HD), which is what the `2e34d0c9` build used. Whether that is *sufficient* is
**unknown and untested**.

### 7.3 Device inventory (valid)

```
/vendor/lib/libmi3.so                    4c199a5c…  <- active, malformed caps mask
/vendor/lib/libmi3.so.orig               c2b5c57d…  <- stock
/vendor/lib/hw/audio.primary.mt5889.so   8c11c348…  <- STOCK (HAL patches reverted)
/vendor/lib/modules/mik.ko               03fc2c0d…  <- R5 (r6=1), inert for DTS
/vendor/lib/modules/utpa2k.ko            f44ad0a4…  <- cmd04+isac3
No DEC/SND firmware image is patched (both stock: 4b7e9509… / eb879cdc…).
```

### 7.4 Verdict for the DSP-firmware patch attempt (unchanged)

# **PATCH TARGET INVALID**

No firmware-writable node on the DTS output path reaches an output/SDO/TX arm; the
transmit-side status block `0xB000_0C00–0x0C07` has 0 firmware stores and is all-zero under DTS
while populated under AC-3. The DSP firmware neither decides nor can influence that condition.

### 7.5 Exact next step

Since the "working baseline" premise is void, the correct order is to **establish a real baseline
before patching firmware**:

1. **First, make the userspace capability path correct** (replace the malformed `libmi3.so` caps
   mask with `0x2E0`, or restore stock and confirm the HAL/format decision), then play
   `test_dts_51.mp4` and read the AVR panel. This answers the open question: *does DTS bitstream
   even leave userspace as DTS on this device?*
   ```sh
   adb root; adb remount
   # candidate mask fix (see FINDING_last_blocker_libmi3_regression.md §3 for the byte edit)
   adb shell 'stop vendor.audio-hal; start vendor.audio-hal'
   ```
2. **Only if DTS bitstream is confirmed to reach the DSP/SPDIF and the AVR still does not lock**,
   apply the DEC capability-gate patch (force the `*(0xB000001E)==5` branch at `0x21F5C` and/or the
   DTS-branch site `0x21B19`) — a separate, single change, on a NEW versioned artifact, never
   overwriting the stock image.

Full write-up: `aeon_validate/r57_npcm/FINDING_last_blocker_libmi3_regression.md` (corrected).

---

## 8. FOLLOW-UP B (2026-09-13) — the DTS-Pc patch was already deployed, and it fails

### 8.0 The single most important fact established this session

While checking device state I found that the SND DSP image **already carries the "DTS Pc fix"**:

```
/vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin
   md5 785fd73bcae39a88e6b6029445190e98        <- DEPLOYED
   stock eb879cdc07f510722f19db6d18d77d3c (1,839,920 B)
   byte diff: EXACTLY 1 byte — file 0x1C051, 0x21 -> 0x2B
   instruction at file 0x1C050:  9B 21 (bt.movi r25,0x1)  ->  9B 2B (bt.movi r25,0x0B)
```

That is precisely the R56 "Option P1 / fix the DTS IEC61937 Pc to 0x0B" patch. It is **not** a
proposal — it is **live on the device**. So the question "is the wrong Pc the blocker?" has a
definitive answer, and it is **no**.

### 8.1 The experiment that proves it (never run before)

I used `dump_spdif_npcm` — a capture of the **actual bytes handed to the SPDIF TX** — with playback
confirmed live via Kodi JSON-RPC (`Player.GetActivePlayers` → `playerid 1, video`).

| run | source | decoder state | wire capture |
|---|---|---|---|
| A | `test_ac3_51.mp4` | AC3 playing | **5,695,488 bytes**, 927 bursts, `Pc=0x0001` (all) |
| B | `test_dts_51.mp4` | **`format: DTS`, `state: PLAY`, `frame count: 12569`** | **0 bytes** |
| B′ | `test_dts_51.mp4` (repeat) | DTS playing | **0 bytes** |

⇒ The capture instrument is **proven working** (run A). Under DTS the SPDIF TX receives
**nothing at all** — not a malformed burst, not a wrong-Pc burst, nothing — while the DTS decoder
runs at full rate.

### 8.2 Consequence — R56 is refuted, and the old wire evidence is re-read

If the DTS branch of the header writer had executed, the TX FIFO would receive at least the 8-byte
preamble plus a zero-padded payload → a **non-zero** capture. It receives 0 bytes, twice.

Therefore:
- **The DTS output path never reaches the burst-header writer.** Changing `Pc` cannot have any effect.
- The `Pc = 0x01` read statically in R56 is the **slot's stale/default value** left from the last AC-3
  burst — exactly what `dts_01.bin` ("frozen AC-3 header ×26") already showed. That capture is now
  explained as **stale buffer content with no DTS write at all**, not as a DTS burst mislabelled AC-3.

This is consistent with §1's own negative (the `0x26270–0x2661D` gate never arms). Both routes agree:
**the DTS output stage is never armed.**

### 8.3 Verdict for the DTS-Pc patch

# **PATCH HAD NO EFFECT**

(`Pc` is already correct in the running image; the output it would modify is never produced.)

### 8.4 What is now the only untried, evidence-backed target

I audited the record: **no artifact for a DEC-image patch exists anywhere, and no report describes one
being tested.** The DEC capability gate in `FINDING_rootcause_license_gate.md` is therefore genuinely
untried. It is also the only component that can **suppress output entirely** (a 0-byte capture is
exactly the signature of suppression): its error path at fw `0x22031` explicitly **zeroes the output**
(`sw r0,236(r11)` / `sh r0,236(r11)`), and its diagnostic string `"Invalid Spdif license"` **never
appears in any captured log**.

### 8.5 Exact next steps (one change at a time, each reversible)

**Step 0 — remove the confounding variable.** Restore stock SND (the Pc patch is inert and has been
distorting recent tests):
```sh
adb root; adb remount
adb shell 'cat /data/local/tmp/asnd_stock.bin > /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin && sync'
# verify md5 eb879cdc07f510722f19db6d18d77d3c
adb reboot
```

**Step 1 — the single untried change: neutralise the DEC suppress branch (Option L2).**
New versioned artifact built from stock DEC (`4b7e9509…`; **DEC image has NO file-offset shift**):
```
file offset 0x21F66 :  d3 1f 06 58   (bg.blesi r24,-0x1,->0x22031)
                       ->  NOP-equivalent so the <=-1 test no longer escapes to the zeroing error path
```
Deploy → reboot → **measure with `dump_spdif_npcm`**.

**Success criterion for this experiment is NOT an AVR lock — it is a non-zero DTS capture.**
Any bytes at all prove the suppress path was the blocker. AVR lock is evaluated only afterwards.

**AC-3 regression control after every step is mandatory** (`Pc=0x0001` × many, live stream).

Full write-up: `aeon_validate/r57_npcm/FINDING_dts_pc_REFUTED_by_experiment.md`.

---

## 9. FOLLOW-UP C (2026-09-13) — the DEC capability gate was patched and tested. It is NOT the blocker.

This section executes §8.5 Step 0 and Step 1, and reports the result. **Both steps were carried out
on the device, with a full baseline restore and a mandatory AC-3 regression control.**

### 9.0 A critical instrumentation error found and fixed first

The first measurement attempt in this phase produced a **false "0 bytes"** for *both* codecs and
**no capture file at all**. Root cause:

> **`adb shell` was running as `uid=2000(shell)`, not root.** `/proc/utopia_mdb/audio` is
> `-rw------- root root`; the write silently failed (`rc=1`), and no `Start Dump Spdif Tx Npcm`
> marker ever appeared in dmesg. There is no `su` binary on this device.

`adb root` was therefore re-issued (`restarting adbd as root`), after which the write returned
`rc=0` and the `----- Start/Stop Dump Spdif Tx Npcm -----` markers appeared normally.

**Any "0 bytes" result obtained without `adb root` is invalid.** This retroactively explains part of
the confusion in §8 and is the reason the measurements below were repeated from a clean state.

### 9.1 Step 0 — baseline restored (executed)

```sh
adb root; mount -o remount,rw /vendor
cat /data/local/tmp/asnd_stock.bin > .../aucode_asnd_r2_MS12V22.bin   # stock SND
```
| image | before | after |
|---|---|---|
| SND (`aucode_asnd_r2_MS12V22.bin`) | `785fd73b…` (V1 Pc patch) | **`eb879cdc07f510722f19db6d18d77d3c`** (stock) |

A stock DEC backup (`/data/local/tmp/adec_stock.bin`, `4b7e9509…`) and a stock SND backup
(`/data/local/tmp/asnd_stock.bin`, `eb879cdc…`) were staged on the device **before** any write.

### 9.2 Step 1 — the L2 artifact (built, deployed, rebooted)

**New versioned artifact:** `r57_npcm/artifacts/aucode_adec_r2_MS12V22_GATE_L2.bin`
**md5 `cba5f7e795818462b55736543329c9ea`** · 1,982,492 B · built from **stock** DEC `4b7e9509…`

**Diff = exactly 2 bytes** (verified by whole-file `cmp`):

| offset | stock | patched |
|---|---|---|
| `0x21F68` | `06` | `00` |
| `0x21F69` | `58` | `23` |

i.e. the 4-byte instruction at `0x21F66`:

```
stock    : d3 1f 06 58   bg.blesi r24, -0x1, ->0x22031
patched  : d3 1f 00 23   bg.011i r24, 0x1F, ->0x21F6A   (unconditional goto to next instruction)
```

Encoding was derived from the AEON slaspec (`i32_opcode=0x34`, `i32_uimm0_3` 0=błęsi/2=beqi/3=GOTO,
`i32_rel_simm3_13=(3,15) signed`, **branch base = PC**). Both the stock words were re-encoded and
matched byte-for-byte before the patch was applied; the patched word was decoded back and confirmed.

Deployment verified on-device:
```
cba5f7e795818462b55736543329c9ea  .../aucode_adec_r2_MS12V22.bin   (patched, loaded)
eb879cdc07f510722f19db6d18d77d3c  .../aucode_asnd_r2_MS12V22.bin   (stock)
xxd -s 0x21F64 -l 8  ->  1800 d31f 0023 eaeb                       (patch visible)
```
A real reboot was issued and confirmed by uptime reset (30 s), so the DSP reloaded the patched image.
(`adb reboot` had silently failed once — it must be verified via `/proc/uptime`, not by exit code.)

### 9.3 The measurement (instrument proven in the same session)

Playback was driven via Kodi JSON-RPC on `::1:8080`; decoder liveness was confirmed from
`show_all_decoder_status` **before** trusting any capture size.

| source | decoder status | wire capture |
|---|---|---|
| `test_dts_51.mp4` | `format: DTS`, `state: PLAY`, `sample rate 48000`, `channels 10`, **`frame count 1863`** | **0 bytes** (`DUMP_audio_spdifNpcm_01.bin`) |
| `test_ac3_51.mp4` | `format: AC3P`, `state: PLAY`, **`frame count 719`** | **5,431,296 bytes** (`DUMP_audio_spdifNpcm_02.bin`) |

The capture file **was created** in both cases, so the instrument was armed and working.

### 9.4 Verdict for the L2 (DEC capability gate) patch

# **PATCH HAD NO EFFECT**

* The DTS decoder runs at full rate (`format: DTS`, `PLAY`, frame count climbing).
* The SPDIF TX still receives **nothing** (0 bytes) under DTS.
* The **exact same instrument**, in the **exact same session**, captured **5.4 MB** of live AC-3.
* The patch was byte-verified as loaded and the device was truly rebooted.

⇒ **The DEC capability gate at `0x21F66` is NOT the last blocker.** Either the gate is not
traversed on this path at all, or its suppress branch is not what stops the stream. The 0-byte
capture under DTS is **not** explained by this gate.

### 9.5 Regression status

**No regression.** AC-3 is intact: `AC3P` / `PLAY` / frame count 719, with a 5,431,296-byte live
capture. This is expected — the patched branch is only reachable via the `0x21F66` test, which the
AC-3 path does not take. The legitimate zeroing paths Z1–Z3 (§4 of
`r57_npcm/FINDING_gate_L2_pre_experiment.md`) were deliberately left untouched.

### 9.6 What this eliminates, and where the search now stands

**Eliminated (proven, with the wire as the criterion):**

| target | status |
|---|---|
| SND IEC61937 `Pc` value (`0x1C050`, V1) | **PATCH HAD NO EFFECT** (§8) |
| DEC capability gate `0x21F66` (L2) | **PATCH HAD NO EFFECT** (§9) |
| DTS output-pack path `0x26270–0x2661D`, gate `0x265F3` | **PATCH TARGET INVALID** (§1–§6) |
| `0x854`, `0x25EE1`, `0x1F6DA`/`mem[0x8(r10)]`, `0x868/86C/870`, `0x814` bit21, `0x0C00–0x0C07` | eliminated by R46–R54 (see §1.1) |

**The load-bearing observation that now governs the search:** under DTS the decoder is fully alive
and the SPDIF TX receives **zero** bytes — while AC-3 in the same conditions produces megabytes.
The divergence is therefore **not** in any value the IEC61937 writer would set; it is in whether the
DTS audio path reaches the pre-TX output stage **at all**.

`FINDING_wire_capture.md` §4 records that the DEC image contains a **complete, plausibly correct DTS
IEC61937 mechanism** (data-type selector `0x45D07`, Pa/Pb/Pd writers, DTS syncword writers
`0x46259`/`0x462DE`, syncword **detector** `0x64624`). The next investigative step should be to
determine **which condition fails to arm that existing mechanism** — i.e. trace the DEC-side DTS
output entry (`0x45xxx`/`0x46xxx`) and find the guard that returns before the burst is built. That is
static work on the DEC image, and it has **not** been done yet.

### 9.7 Device state at the end of this phase

```
cba5f7e795818462b55736543329c9ea  /vendor/lib/.../aucode_adec_r2_MS12V22.bin   <- L2 patch STILL DEPLOYED
eb879cdc07f510722f19db6d18d77d3c  /vendor/lib/.../aucode_asnd_r2_MS12V22.bin   <- stock (restored)
/data/local/tmp/adec_stock.bin = 4b7e9509…   (rollback)
/data/local/tmp/asnd_stock.bin = eb879cdc…   (rollback)
/data/local/tmp/adec_L2.bin    = cba5f7e7…   (the patch, re-deployable)
```
`dump_spdif_npcm` was disarmed (`dump_spdif_npcm=0 path=1`). Capture is **off**.
To roll back: `cat /data/local/tmp/adec_stock.bin > /vendor/lib/.../aucode_adec_r2_MS12V22.bin && sync && reboot`.

---

## 10. FOLLOW-UP D (2026-09-13) — the DEC-side DTS output path: ROOT CAUSE LOCATED (static, no patch)

**Mode: READ-ONLY static analysis.** Nothing patched, nothing deployed. Verdict category for this
section: **none of the five patch verdicts apply** — this is location/derivation work, not an experiment.
The relevant verdict will apply to the H1/H2 patches in §10.5 once they are built and measured.

### 10.0 Why this section exists

§9.6 ended with "the next investigative step should be to trace the DEC-side DTS output entry
(`0x45xxx`/`0x46xxx`) and find the guard that returns before the burst is built." **That has now been
done**, and it produced a concrete, byte-verified answer. The full derivation is in
`r57_npcm/FINDING_dec_dts_output_rootcause.md`; this is the summary.

### 10.1 Correction of a false alarm (important)

A raw byte search for the DTS syncword constants (`7FFE8001`, `1FFFE800`, `58642520`, `80017FFE`,
`FE7F0180`, `00E8FF1F`) returns **0 hits in both `dec_full.bin` and `snd_full.bin`**. I initially read
this as "the DTS mechanism does not exist." **That was wrong.** AEON composes these constants from
**split instruction immediates**, so the 32-bit value never appears contiguously. At `0x4627B`:

```
04627B  bg.ori r25,r0,0x7ffe
04627F  bn.sw  0x0(r24),r25
046285  bg.ori r25,r0,0x8001
046289  bn.sw  0x0(r24),r25     ==> 0x7FFE8001 = DTS core sync, written correctly
```

**Rule reinforced (SKILL §13):** a byte scan is not evidence of absence for immediate-built constants.
Use the disassembly.

### 10.2 The DTS mechanism is COMPLETE and CORRECT

`FINDING_wire_capture.md` §4's address table is **confirmed correct** by independent Ghidra disassembly:

| address | role | status |
|---|---|---|
| `0x45D07` | header builder; `r4==0` → data-type **`0x0B` = DTS** (`0x45DA4`), `r4==1` → `0x10C` | ✓ intact |
| `0x45D4C/8A/CA` | Pa=`0xF872` (`0x45D3F`/`0x45DA0`), Pb=`0x4E1F` (`0x45D50`/`0x45D8E`), slots `+0x16D/+0x171/+0x173` | ✓ intact |
| `0x46259` | reads `s->0x2c`; `!=1` → leave DTS branch (`0x4625C`) | ✓ intact |
| `0x46266` | reads data-type `s->0x16c` | ✓ intact |
| `0x4627B`/`0x46285` | **DTS core sync `0x7FFE8001`** | ✓ intact |
| `0x462DE`/`0x462ED` | **DTS-HD sync `0x1FFFE800`** | ✓ intact |

### 10.3 THE FORK — a single field `s->0x24`

The decision function is at `0x46040`–`0x46160`. Load-bearing lines (DEC27, byte-verified):

```
0460A2  bt.mov  r3,r10
0460A4  bg.jal  0x00064296      ; r3 returned UNCHANGED (struct ptr) — a red herring
0460A8  bt.mov  r3,r10
0460AA  bg.jal  0x00064354      ; r3 = s->0x24      <-- SUB_00064354 is just  "lwz r3,0x24(r3); jr r9"
0460AE  bn.sw   0x30(r10),r3    ; guard-B = s->0x24
0460B1  bg.bnei r3,0x0,0x0004614c   ; *** s->0x24 != 0  ->  0x4614C ***
```

The `0x4614C` arm (raw `88 6a e4 03 c4 02 e7 ff ff 43`):
```
04614C  bt.mov  r3,r10
04614E  bg.jal  0x0006434f      ; SUB_0006434F:  sw 0x24(r3),r0 ; jr r9   <- CLEARS s->0x24
046152  bg.j     0x000460f3     ; -> the "no syncword written" tail
```

**Truth table for the DTS core syncword:**

| `s->0x24` | path | result |
|---|---|---|
| **`== 0`** | fall through `0x460B5` → size select → `0x460E9`+ (`r4=2`,`r6=0x2000`) / `0x46172` / `0x46156` | **DTS sync `0x7FFE8001` written** |
| `!= 0` | `0x4614C` → clear field → `0x460F3` | **no syncword at all** |

And `0x4625C` adds a second, independent condition: `s->0x2c` **must equal 1**, else the whole DTS
branch is left for the generic path.

### 10.4 Guard-field map (all verified)

| field | read at | written at |
|---|---|---|
| `s->0x2c` (must `== 1` for DTS) | `0x46259` | `0x46087` (from `SUB_00064149`), `0x21D68`, many others |
| `s->0x30` (must `== 0` for DTS core) | `0x46260` | `0x460AE` (from `s->0x24`), `0x14048`, `0x1413E`, … |
| **`s->0x24` (the fork input)** | `0x64354` | `0x6434F` (clears), plus 13 upstream writers incl. `0x23A7E`, `0x28F3A`, `0x2D3C4`, `0x3C49C` |
| `s->0x38` (frame-size class `0x200`/`0x400`/`0x800`) | `0x460B8`, `0x46080` | `0x46097` |
| `s->0xf8`,`s->0xfc` (output cursors) | `0x46222/6A/A0/DE` | many |

### 10.5 Two NEW patch hypotheses (neither has been tried)

* **H1 — neutralise the `s->0x24` fork** at `0x460B1`:
  `d0 60 04 de` (`bg.bnei r3,0,→0x4614C`) → **`d0 60 00 23`** (`bg.011i` unconditional goto to
  `0x460B5`; rD=3, imm=0, `u3=3`, `rel = 0x460B5−0x460B1 = 4`). Diff = 2 bytes at `0x460B2`/`0x460B3`.
  *Caveat:* only effectual if the DTS path is entered at all; if `s->0x2c != 1` the syncwriter is still
  skipped at `0x4625C`.
* **H2 — relax the `s->0x2c == 1` requirement** at `0x4625C`:
  `d2 e1 fa 7e` (`bg.bnei r23,0x1,→0x461AB`) → **`d2 e0 00 23`** (`bg.011i` unconditional goto to
  `0x46260`; rD=23, **imm=0**, `rel=4`).
  *Caveat:* same *class* as the already-proven-inert L2 gate patch (§9) — must be verified **on the wire**.

**Both patch words were verified by round-trip:** re-encoding the stock operands reproduces the stock
bytes (`0xd06004de`, `0xd2e1fa7e`) exactly. Note the H2 subtlety — the stock `bnei` has `imm=1`, and an
unconditional `bg.011i` must zero it.

Both are small, reversible, and preserve **AC-3** (they only alter the DTS-specific branch). Neither
touches the header builder, so the AC-3 `Pc=0x0001` path is untouched.

**Mandatory test protocol (SKILL §14):** `adb root` → verify `uid=0(root)` → real reboot verified via
`/proc/uptime` → **AC-3 positive control in the same session** → DTS capture. A 0-byte result without
root is invalid.

### 10.6 Where the search stands now

The blocker is no longer "some unknown guard in a huge region." It is **one of two named fields**
(`s->0x24` for the fork, `s->0x2c` for the branch entry), each with an identified read site, an
identified writer set, and a ready 2-byte patch. The remaining unknown is **which upstream writer sets
`s->0x24` non-zero during DTS playback** — 14 candidate writers are enumerated in the finding, and
`0x460AE` (inside the decision function) is confirmed while the other 13 are upstream state writes.

**Eliminated so far (wire is the criterion):** SND `Pc` (`0x1C050`) — **PATCH HAD NO EFFECT**;
DEC capability gate `0x21F66` (L2) — **PATCH HAD NO EFFECT**; `0x26270–0x2661D` packer gate —
**PATCH TARGET INVALID**; plus `0x854`, `0x25EE1`, `0x1F6DA`, `0x868/86C/870`, `0x814` bit21,
`0x0C00–0x0C07`.

---

## Appendix — artifacts produced this phase

| Artifact | Description |
|---|---|
| `scripts/R54_TracePacker.java` | Ghidra flow-following decoder for `0x26270–0x26420` + `0xC49D`/`0xCB50` xref enumeration |
| `scripts/R54b_FollowPack.java` | Ghidra decoder for `0x26320–0x26620` + targeted xref probes |
| `r54_work/r54_full.txt` | Raw R54 output (17,658 B) |
| `r54_work/r54b_full.txt` | Raw R54b output (27,402 B) |
| `r54_work/REPORT_r54_decisive_trace.md` | Byte-exact trace + verdict (this phase's analysis) |
| `r57_npcm/FINDING_dts_pc_REFUTED_by_experiment.md` | **§8 source** — live refutation of the DTS-Pc theory + corrected next target |
| `r57_npcm/ac3_PATCHED_0022.bin` | 5,695,488 B AC-3 wire capture (`Pc=0x0001` ×927) — proves `dump_spdif_npcm` works |
| `r57_npcm/FINDING_dts_pc_rootcause_CONSOLIDATED.md` | the R56 Pc finding, byte-verified by me (kept for provenance; its causal claim is refuted in §8) |
| `r57_npcm/FINDING_rootcause_license_gate.md` | the DEC capability gate — the **only untried** target (§8.4) |
| `r57_npcm/FINDING_gate_L2_pre_experiment.md` | pre-experiment derivation of the L2 change: encoding solved, control flow, reachability, zeroing-path map (§9 source) |
| `r57_npcm/artifacts/aucode_adec_r2_MS12V22_GATE_L2.bin` | **the L2 patch** — stock DEC + 2 bytes at `0x21F68/9`; md5 `cba5f7e7…` (§9) |
| `r57_npcm/artifacts/aucode_adec_r2_MS12V22_STOCK.bin` | stock DEC reference (`4b7e9509…`) |
| `r57_npcm/artifacts/aucode_asnd_r2_MS12V22_STOCK.bin` | stock SND reference (`eb879cdc…`) |
| `r57_npcm/ac3_L2_002.bin` | 5,431,296 B AC-3 wire capture under the L2 patch — regression control, proves the instrument (§9.3) |
| `r57_npcm/scratch/build_l2.py` / `verify_l2.py` | artifact builder + independent decode-back verifier (§9.2) |
| `r57_npcm/FINDING_dec_dts_output_rootcause.md` | **§10 source** — DEC-side DTS output root cause: the `s->0x24` fork at `0x460B1`, the guard-field map, H1/H2 patches |
| `scripts/DEC22_DtsOutTrace.java` | linear-dissassembly tracer, `0x45CD0`–`0x46380` (header builder + syncword writers) |
| `scripts/DEC23_BuilderCallers.java` | finds every call into the builder cluster + r4/r6 setup context |
| `scripts/DEC24_GuardWriters.java` | full-image scan of stores to `0x2c`/`0x30` (the two guard fields) |
| `scripts/DEC25_GuardCtx.java` / `DEC26_StructMap.java` | guard-writer context + per-field access map on base `r10` |
| `scripts/DEC27_DtsDecision.java` | the decision function `0x46040`–`0x46160` (where the fork is) |
| `scripts/DEC28_DecisionFn.java` | decode of `0x64296` / `0x64354` / `0x6434F` (proves `0x64296` is a red herring) |
| `dec_work/dec22_out.txt` … `dec28_out.txt` | the authoritative listings cited in §10 |

---

## 11. FOLLOW-UP E (2026-09-13) — §10 is SUPERSEDED: the real blocker is `entry+0x808` / `0x460C2`

**Mode: READ-ONLY static analysis.** Nothing patched, nothing deployed. No verdict from the fixed set
applies yet — this is location work with two artifacts built and *not* flashed.

Full derivation: **`r57_npcm/FINDING_dec_dts_output_SUPERSEDES_s24.md`**.

### 11.0 What was wrong in §10

§10 claimed the DTS blocker was `0x460B1 bg.bnei r3,0,→0x4614C` reading `s->0x24`, a "DTS-enable /
licence gate". **That was a misattribution.** Decoding the helpers with `dec32_clean.txt` (and the
`aeon.slaspec` field layout) shows:

| helper | what it actually is |
|---|---|
| `0x64296` | **a bitstream-reader re-seek** — computes `0x8/0x0/0x4` of the reader context from `0x14/0x18/0xc/0x10` |
| `0x6434F` | `sw 0x24(r3),r0` / `jr r9` — the reader-context **setter** |
| `0x64354` | `lwz r3,0x24(r3)` / `jr r9` — the reader-context **getter** |
| `0x4614C` arm | **a sample-rate-table dispatcher** — `0x7D00`=32000, `0x3E80`=16000, `0x1F40`=8000, `0x5622`=22050, `0x2B11`=11025, each `ori r23,.. ; j 0x46069` which stores to `0x80/0x78/0x7C(r10)` |

So `0x24(r10)` is a **bitstream-reader field**, not a DTS flag; and the `0x4614C` arm is a legitimate
"compute the frame rate because it is not known yet" fallback. `0x24(r10)` and `0x24(r15)` are
**different structs** (`r10` = the local descriptor set at `0x45FE9 mov r10,r3`; `r15` = the DEC
stream-manager context).

### 11.1 The real mechanism, end to end

```
0x41283   jal 0x45FD1          <-- the ONLY caller of 0x45FD1 in the whole image
             r4 = *(r11 - 0x5EF8)      (the mode/params struct)
             r3 = base + 0x14CCC        (the output descriptor)
0x45FD1   prepare(desc, params):
            0x45FDB  desc->0x28 = 1
            0x45FE6  desc->0x38 = 0
            0x45FE9  r10 = desc
            0x45FEB  beqi params->0x0, 1, →0x4600C      <-- DISCRIMINATOR #1
            0x45FF5  desc->0x28 = 0                     <-- the "unprepared" arm
0x4600C   ... sets desc->0x2c from params->0x28 ...
0x4608E   desc->0x38 = (r3 + 1) << 5                    <-- the selector, set ONLY via params->0x0==1
0x460B5   lwz r24, 0x8(r10)    ; payload_len
0x460B8   lwz r23, 0x38(r10)   ; selector
0x460BB   addi r26, r24, 0x80
0x460BF   slli r25, r23, 0x5
0x460C2   ble r25, r26, →0x460F3     *** GATE: if (selector<<5) <= (payload+0x80) -> SKIP ***
0x460C6..0x460D8   selector dispatch: 0x200 -> r4=0 (DTS), 0x400 -> r4=1, 0x800 -> skip
0x460EF   jal 0x45D07          <-- the IEC61937 header builder (DTS data-type 0x0B at 0x45DA4)
```

**If `params->0x0 != 1`, then `desc->0x38` is left at 0** (it was zeroed at `0x45FE6`), so
`r25 = 0` and `r26 = payload+0x80 > 0`, and **`0x460C2` is taken every single time → `0x45D07` is
never called → no IEC61937 header → no DTS on the wire.**

### 11.2 Cross-verified second chain (the per-stream entry pool)

```
0x465F6   lwz  r24,0x7e3c(r3)   ; entry count
0x46607   muli r4,r4,0x80C      ; ENTRY STRIDE = 0x80C
0x4660F   addi r23,r23,0x6e24   ; TABLE BASE  = +0x6E24
0x46618   sw   0x7e3c(r3),r0    ; reset count (free-all)
0x46D28   lwz  r23,0x7e3c(r10)
0x46D2C   bgtui r23,0x1,→0x46cbc ; *** CAPACITY: refuses if count > 1  => only 2 entries ***
0x46D81   sw   0x762c(r25),1     ; *** entry[idx].0x808 = 1 ***
```
`0x762C - 0x6E24 = 0x808`, and `(0x7E3C - 0x6E24)/0x80C = 2` — **a fixed pool of exactly 2 entries.**
The header builder tests that very flag: `0x45EE9 lwz r23,0x808(r23)` / `0x45EED beqi r23,1,→0x45FBF`.

### 11.3 All five IEC61937 Pa/Pb emitter clusters (the complete set)

`0xF872` is **always** materialised as `bg.addi rN,r0,-0x78e`, and `0x4E1F` as `bg.addi rN,r0,0x4e1f`.
That is why a text search for `f872` finds nothing useful.

| # | Pa | Pb |
|---|---|---|
| A | `0x24C0C` | `0x24C19` |
| B | `0x24C85` | `0x24C8E` |
| C | `0x2645E` | `0x2646E` |
| D | `0x2ABAC` | `0x2ABB3` |
| E | `0x2B5FC` | `0x2B603` |
| **F** | **`0x45D3F/45D7F/45DA0/45DBF`** | **`0x45D50/45D8E/45DAD/45DCE`** |

**Only F is reachable from the DTS frame-geometry code**, and it is gated by `0x460C2`.

### 11.4 Artifacts built (NOT flashed)

```
r57_npcm/artifacts/aucode_adec_r2_MS12V22_H1_460B1.bin   md5 e64b76e8b3ce05809ae8d50409e2b0ad
    0x460B1: d0 60 04 de -> d0 60 00 23   (bg.bnei -> bg.011i, unconditional goto to 0x460B5)
r57_npcm/artifacts/aucode_adec_r2_MS12V22_H2_4625C.bin   md5 28c4ea8588935894f67203e038b0eb43
    0x4625C: d2 e1 fa 7e -> d2 e0 00 23   (bg.bnei -> bg.011i, unconditional goto to 0x46260)
```
Both verified by `scripts/aeon_verify_patch_words.py`, which re-encodes from the `aeon.slaspec` field
definitions and round-trips two real stock words (one with a **negative** displacement).

### 11.5 A hard-won encoding lesson (read before editing any AEON branch)

The AEON 32-bit word is **big-endian in the file**: `d0 60 04 de` = `0xD06004DE`.
**`i32_rel_simm3_13` = bits 3..15 *straddles the 3rd and 4th file bytes*.** Therefore a branch
**cannot** be retargeted by editing the low byte alone: `d0600423` decodes to `rel=132` →
target `0x46135` (an arbitrary wrong instruction) rather than the intended `0x460B5`. Both field
bytes must be written together. I got this wrong twice this session before verifying numerically.

Also: **the `bn.*` 24-bit family has no unconditional-goto sub-op** — `uimm0_2` gives only
`beqi/bf/bnei/bnf` (op `0x08`) and `blesi/bleui/bgtsi/bgtui` (op `0x09`). The unconditional form is a
separate instruction, `bn.j` (`i24_opcode = 0x0B`), which is a **different length** and therefore
changes all downstream alignment. Contrast with the 32-bit `bg.*` group `0x34`, where
`uimm0_3 = 3` **is** `bg.011i`, an in-place unconditional goto — which is why H1/H2 are viable.

### 11.6 The unresolved question, stated precisely

**Which caller value makes `params->0x0 == 1` for AC-3 but not for DTS?**
`params->0x0` = `*(r11 - 0x5EF8)` where `r11` indexes the per-stream control block, and the whole
block is guarded by `0x4121E beqi r23,1,→0x41275` with `r23 = 0x4EE4(r11)`.
Until that is answered, **H1 is the correct discriminating probe**: if H1 makes DTS appear on the
wire, the defect is the fork's interpretation of state; if it does not, the defect is upstream
(before `0x41283`) and no patch in `0x45xxx`/`0x46xxx` can help.

**Mandatory protocol for the H1 test** (unchanged, from §9): `adb root` → confirm `uid=0(root)` →
real reboot verified via `/proc/uptime` reset → **AC-3 positive control in the same session** →
DTS capture. A 0-byte DTS result **without** a same-session non-zero AC-3 control is **INVALID**.
Success criterion: **a NON-ZERO DTS wire capture.**


### 11.7 Appendix additions for §11

| Artifact | Description |
|---|---|
| `r57_npcm/FINDING_dec_dts_output_SUPERSEDES_s24.md` | **§11 source** — the corrected root cause: `0x460C2` gate, the `entry+0x808` pool, the complete prepare chain from `0x41283` |
| `dec_work/dec32_clean.txt` | **authoritative** instruction listing — 637,122 instructions, `<addr> <len> <mnemonic> \| <bytes-big-endian>` |
| `dec_work/dec32_out.txt` | raw Ghidra log for the above |
| `scripts/DEC32_DumpAll.java` | produced the authoritative listing |
| `scripts/dec33_exact_attrib.py` | exact base/disp attribution using Ghidra's real instruction boundaries |
| `scripts/aeon_verify_patch_words.py` | re-encodes branch words from `aeon.slaspec`; round-trips stock words; **caught the 1-byte-vs-2-byte straddle bug** |
| `scripts/aeon_branch_encode.py` | stock-word decode consistency check (all 6 stock words OK) |
| `r57_npcm/artifacts/aucode_adec_r2_MS12V22_H1_460B1.bin` | H1 patch artifact, md5 `e64b76e8b3ce05809ae8d50409e2b0ad` — **built, not flashed** |
| `r57_npcm/artifacts/aucode_adec_r2_MS12V22_H2_4625C.bin` | H2 patch artifact, md5 `28c4ea8588935894f67203e038b0eb43` — **built, not flashed** |

---

# §12 — EXECUTED EXPERIMENT (2026-09-13): D1 + instrument-verified wire baseline

This section records the **authorized execute cycle**: baseline → patch → verify →
deploy → reboot → AC-3 regression → DTS test. It is the reproducible record.

## 12.1 Baseline identification + fresh backup (BASELINE/SAFETY rule)

| item | value |
|---|---|
| active DEC before any change | `/vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin` |
| its md5 | `cba5f7e795818462b55736543329c9ea` (the L2 image) |
| its sha256 | `7cf1c397f1ed2f6627d8860ea17c30b7ac4d54779c70045914c2622986ba34b8` |
| active SND | md5 `eb879cdc07f510722f19db6d18d77d3c` |
| fresh backup (non-overwriting) | `/data/local/tmp/adec_ACTIVE_20260913_013540.bin` |
| backup md5 verified | `cba5f7e795818462b55736543329c9ea` ✓ |
| SND backup | `/data/local/tmp/asnd_ACTIVE_20260913_013540.bin` |
| `/data` free | 18 GB |

## 12.2 THE INSTRUMENT — and why the earlier reasoning had to change

`echo 'dump_spdif_npcm=1 path=1' > /proc/utopia_mdb/audio`
dumps the **SPDIF Tx Non-PCM** stream to `/data/DUMP_audio_spdifNpcm_NN.bin`.
The implementation lives in **`utpa2k.ko`** (`Dump_SpdifNpcm_Monitor`), and the
write pointer it samples is `SpdifNonPcmWritePtr`, surfaced by **`mik.ko`**.
Because AC-3 (also Non-PCM passthrough) is captured in full, the instrument is
provably on the correct path for compressed passthrough.

## 12.3 Wire baseline — A/B in ONE session

| measurement | AC-3 `test_ac3_51.ac3` | DTS `test_dts_51.dts` |
|---|---|---|
| dump size (15–20 s) | **2,918,400 / 5,148,672 B** | **0 B** |
| IEC61937 preambles | **838** | **0** |
| data-type | **0x01 (AC-3) 100 %** | — |
| Pa / Pb | `0xF872` / `0x4E1F` at 0x1800 stride, **16-bit LE** | — |
| `MI_AUDIO_Start Codec:` | **5** | **9** |
| `eDrvOut` | 4 | 4 |
| `Profile[pre,cur]` | [5,5] | [5,5] |
| `MI_PCM` reader | **opened** (`hPcm:0x81000000`) | not opened (baseline) |
| Kodi sink | `AE_FMT_RAW / RAW (PT) / STREAM_TYPE_AC3` | `AE_FMT_RAW / RAW (PT) / STREAM_TYPE_DTS_512` |

Kodi's own log (authoritative, userspace):
```
CAEStreamParser::SyncDTS - dts stream detected
  (6 channels, 48000Hz, 16bit BE, period: 512, syncword: 0x7ffe8001,
   target rate: 0x18, framesize 2012)
CAESinkAUDIOTRACK::Initializing with: m_sampleRate: 48000
  format: AE_FMT_RAW (AE) method: RAW (PT) stream-type: STREAM_TYPE_DTS_512
```
and the HAL confirms the buffer:
```
<MI3_INFO>reallocate memory: _u32DefaultWriteBufferSize:0 -> stWriteParams.u32BufSize:2012
```

**⇒ Kodi and the HAL are correct. 2012 bytes/frame of raw DTS are handed down.**
**⇒ SPDIF Tx emits exactly 0 bytes. AC-3 emits MBs.**

## 12.4 Static refinement that preceded the patch

While resolving the user's steer ("do not blindly force 0x460C2; restore the
intended preparation"), the following was proven from `dec32_clean.txt`:

- `0x45FD1` (prepare) zeroes `desc->0x38` at `0x45FE6`, then dispatches on
  `params->0x0` (`0x45FEB`) and `params->0x10` (`0x45FF1`).
- The third arm at `0x45FF5` does `movi r23,0x0` (`0x45FF8`) and is immediately
  followed by `0x45FFC: bg.beqi r23,0x1,→0x4608E`. **`r23` is provably 0 there and
  is not modified in between**, so that branch can never be taken.
- Therefore the arm at **`0x4608E`** (which sets `desc->0x38 = (0+1)<<5 = 0x20`
  and `memset(desc+0x16C, 0, 8)`, then rejoins the common path at `0x460A2`) is
  **unreachable dead code**, reachable only via that impossible test.
- `0x46080` (the `params->0x10 == 1` arm) reads `params->0x38`, sets
  `desc->0x2c = 1`, and leaves `desc->0x38 == 0`.
- The gate is `0x460B8 lwz r23,0x38(r10)` / `0x460C2 bg.ble r25,r26,→0x460F3`
  with `r25 = 0x38<<5`, `r26 = 0x8(r10)+0x80`.
- `0x410A3` is a mode/eligibility decider; `params->0x0` is written by the
  setter family at `0x4129E`, which assigns **class 9 to selector 0x200 (DTS)**
  and **class 10 to selector 0x400 (AC-3)** — never 1 — when the capability bit
  is present, and returns 0 **without writing** when the capability bit is absent.
- `control->0x7C4` (= the `params->0x0` the prepare function reads) is a
  **latched copy of `control->0x40`**, made at `0x41C7A`, and `control->0x40` is
  built as a **bitfield** (`ori r24,r24,0x20` / `exthz`) — so `== 1` is a
  bit-0 test, not a codec id. A change of `0x40` triggers a large downstream
  state reset (`0x42221`–`0x4225D`).

## 12.5 The patch — D1 @ `0x45FFC` (single byte)

Restores the dead arm (priority #1: *restore existing intended DTS preparation*).

| field | value |
|---|---|
| source binary | `r57_npcm/artifacts/aucode_adec_r2_MS12V22_STOCK.bin` |
| source md5 | `4b7e9509b4358fd3a130bd4d3b9cbe0a` |
| patched artifact | `r57_npcm/artifacts/aucode_adec_r2_MS12V22_D1_45FFC.bin` |
| patched md5 | **`149f4275e9f1ef88ba079f0f32652cee`** |
| patched sha256 | `58bf22223fe92ec1272a9dc0e820027b169aae9a069065ae1ded20f8107e1115` |
| virtual address | `0x45FFC` |
| file offset | `0x45FFC` (DEC image: offset == fw addr, no shift) |
| old bytes | `d2 e1 04 92` |
| new bytes | `d2 e1 04 93` |
| bytes changed | **1** (only `0x45FFF`) |
| old instruction | `bg.beqi r23, 0x1, 0x4608E` |
| new instruction | `bg.011i 0x4608E` (unconditional) |
| opcode group | unchanged, `i32_opcode = 0x34` |
| length | unchanged, 4 bytes |
| displacement | unchanged, `rel = 146` — target stays exactly `0x4608E` |
| boundary/CFG validity | preserved: same group, same length, same field placement; only `uimm0_3` 2→3 |

Verifier: `scripts/aeon_verify_D1.py` (asserts group/length/displacement/target
unchanged and that the target is `0x4608E`). Builder: `scripts/build_D1.py`
(asserts exactly one byte differs). Both PASS.

## 12.6 Deployment

```sh
mount -o remount,rw /vendor
cat /data/local/tmp/D1_45FFC.bin > /vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin
sync
```
- installed md5 verified in place: `149f4275e9f1ef88ba079f0f32652cee` ✓
- reboot issued; **reboot confirmed** by `/proc/uptime` reset **4507.87 s → 32.43 s**
- after reboot, `adb root` re-established (`uid=0(root)`), active DEC re-verified
  as `149f4275e9f1ef88ba079f0f32652cee` ✓

## 12.7 Runtime results (patched D1)

### AC-3 regression — **PASS**
`Codec:5`, capture **2,918,400 B** with IEC61937 preambles. Dolby Digital
unchanged. No regression.

### DTS — **0 bytes** (3 consecutive runs, reproducible)
```
Codec:9
MI_PCM opened: mi_pcm_GetAttr Channel:2 / MI_PCM_Close hPcm:0x81000000
DUMP_audio_spdifNpcm_NN.bin = 0 bytes   (runs at 01:54, 01:54, 01:55)
```
**D1 changed the DTS path** (the `MI_PCM` open/close now occurs, which it did
not at baseline) **but produced 0 bytes on the wire.** The AVR therefore has
nothing to lock onto.

## 12.8 Verdict

**DTS PATH CHANGED BUT NOT FIXED.**

- AC-3 → **PASS** (regression clean).
- DTS → **FAIL**: still 0 bytes at SPDIF Tx.
- D1 is a real, single-byte, verified change that alters DTS behaviour, but the
  IEC61937 header/gate layer it touches is *not* the binding constraint.

## 12.9 What the live result establishes (and why it supersedes the gate model)

The 0-byte capture, together with a full-scale AC-3 capture through the *same*
instrument, establishes that:

1. **The failure is not the IEC61937 header.** A wrong header would still put
   payload bytes on the wire. Zero bytes means nothing is handed to SPDIF Tx.
2. **The failure is not Kodi and not the HAL.** Kodi emits `AE_FMT_RAW`,
   `STREAM_TYPE_DTS_512`, `syncword 0x7ffe8001`, `framesize 2012`; the HAL
   allocates exactly `u32BufSize:2012`. Data reaches the HAL.
3. **The failure is not the output-route selection.** `eDrvOut:4` and
   `Profile[5,5]` are identical for AC-3 and DTS.
4. **The failure is between the DEC/SND output stage and the SPDIF Tx write
   pointer** — i.e. the DEC does not deliver a DTS output frame to the SPDIF Tx
   ring, or the SND side discards it.
5. ∴ any further DEC-side work must target the **output-frame production /
   write-pointer path**, not `0x45D07`, not `0x460C2`, and not `desc->0x38`.

## 12.10 Configuration state observed (not the blocker, but relevant)

```
persist.vendor.audio.spdif.mode     = BYPASS
persist.vendor.audio.spdif.type     = DTS
persist.vendor.audio.hdmi_tx.mode   = BYPASS
persist.vendor.audio.hdmi_tx.type   = DTS
persist.vendor.audio.hdmi_arc.mode  = BYPASS
persist.vendor.audio.hdmi_arc.type  = DTS
```
The user-visible configuration already asks for DTS passthrough. AC-3 works
*while* `type = DTS`, so this property is not a DTS-only restriction.

## 12.11 Exact final modification (reproducible)

| | |
|---|---|
| file | `/vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin` |
| md5 currently active | `149f4275e9f1ef88ba079f0f32652cee` |
| single difference vs stock | `0x45FFF` : `0x92` → `0x93` |
| instruction | `bg.beqi r23,0x1,0x4608E` → `bg.011i 0x4608E` |
| rollback | `cat /data/local/tmp/adec_ACTIVE_20260913_013540.bin > /vendor/lib/utopia/audio_bin/aucode_adec_r2_MS12V22.bin && sync && reboot` (restores `cba5f7e795818462b55736543329c9ea`) |
| artifact | `r57_npcm/artifacts/aucode_adec_r2_MS12V22_D1_45FFC.bin` |
| verifier | `scripts/aeon_verify_D1.py` |
| builder | `scripts/build_D1.py` |
| capture script | `cap.sh` |
| wire analyzer | `scripts/analyze_wire.py` |
| baseline finding | `r57_npcm/FINDING_wire_baseline_20260913.md` |

## 12.12 Next concrete blocker (for the next iteration)

The evidence says the next target is the **output-frame production / SPDIF Tx
write-pointer path**, i.e.:

- who advances `SpdifNonPcmWritePtr` (in `mik.ko`) and whether the DTS path
  ever calls it;
- the DEC/SND output-frame handoff for codec class 9 vs class 10;
- the `(sndR2)` vs `(decR2)` latency paths referenced by `utpa2k.ko`, which
  imply DTS and AC-3 may use different internal routes.

This is a *new* direction indicated by live data, not a return to historical
research.
