# FINDING — DTS-Pc theory REFUTED by live experiment; the block is UPSTREAM of the burst header

**Date:** 2026-09-13 (00:22 EEST)
**Device:** Thundeal C50A / MTStar MT5889, Android 11, load avg ~60
**Method:** live runtime experiment on the device (read-only: no new patch written, no file on
`/vendor` modified). The SND image currently deployed **already carries the DTS-Pc patch**.
**Image:** `/vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin` md5 **`785fd73b…`**
(= stock `eb879cdc…` with exactly **one byte** changed: file `0x1C051`, `0x21`→`0x2B`).

---

## 0. VERDICT IN ONE LINE

**The "wrong IEC61937 Pc for DTS" theory (R56 / V1) is REFUTED.** The device already emits
`Pc = 0x0B` for DTS, and DTS still produces **zero bytes** on the SPDIF TX path while the decoder is
decoding at full rate. The blocker is **upstream of the burst-header writer**, so changing Pc cannot
help.

---

## 1. What was already deployed (established this session, byte-exact)

| item | value |
|---|---|
| Stock SND image | `eb879cdc07f510722f19db6d18d77d3c`, 1,839,920 B |
| **Deployed** SND image | **`785fd73bcae39a88e6b6029445190e98`** |
| Byte diff vs stock | **exactly 1 byte**: file `0x1C051`, stock `0x21` → deployed `0x2B` |
| Instruction at file `0x1C050` | stock `9B 21` = `bt.movi r25,0x1` → deployed **`9B 2B` = `bt.movi r25,0x0B`** |
| Meaning | DTS branch now writes **`Pc = 0x0B` (IEC61937_DTS1)** — the "correct" value |

I verified this by reading the deployed artifact directly
(`r56_work/artifacts/aucode_asnd_r2_MS12V22_DTS_PC_PATCH_V1.bin`) — the immediate field decodes to
`11 (0x0B)`. **V1 ≡ the R56 "Option P1" fix.** It was not a pending idea; it is live.

---

## 2. The decisive experiment (never performed before)

The `dump_spdif_npcm` instrument captures the **actual bytes handed to the SPDIF TX**. I ran it twice,
with playback confirmed live via Kodi JSON-RPC (`Player.GetActivePlayers` → `playerid 1, video`).

### Run A — AC-3 (`test_ac3_51.mp4`), same image, same session
```
/data/DUMP_audio_spdifNpcm_06.bin   =  5,695,488 bytes
Pa(0xF872) occurrences: 927
Pc histogram: {0x0001: 927}
first bursts: @0x0/@0x1800/@0x3000/... Pa=F872 Pb=4E1F Pc=0001 Pd=3000
```
⇒ The capture mechanism works; AC-3 streams a healthy, live burst train.

### Run B — DTS (`test_dts_51.mp4`), Pc=0x0B patch LIVE
Decoder status while capturing (from dmesg, `show_all_decoder_status`):
```
Decoder ID : 0
Decoder format : DTS
Decoder play state : PLAY
Decode frame count : 12569          <-- DTS IS DECODING, at ~full rate
```
Wire capture:
```
/data/DUMP_audio_spdifNpcm_05.bin   =  0 bytes
/data/DUMP_audio_spdifNpcm_07.bin   =  0 bytes    (independent repeat)
```

**Result: DTS decodes at full rate, and the SPDIF TX receives NOTHING — not even a malformed burst.**

---

## 3. Why this refutes R56

R56's model required the DTS branch of the header writer (`fw 0x5142–0x515E`) to **execute** and emit a
burst with the wrong Pc. The evidence now shows:

1. If that branch executed, the TX FIFO would receive **at minimum** Pa/Pb/Pc/Pd + zero-padded payload
   — `dump_spdif_npcm` would return non-zero bytes (as it does for AC-3, and as the old `dts_01.bin`
   capture implicitly showed by containing stale AC-3 headers).
2. It returns **0 bytes**, twice.
3. Therefore **the DTS path never reaches the header writer at all.** The `Pc = 0x01` value that R56
   read statically is simply the **reset/default value** of that slot, retained from the last AC-3 burst
   — which is exactly what the old `dts_01.bin` (frozen AC-3 header ×26) showed.

**Corrected reading of the old `dts_01.bin`:** it was never "DTS emitted with wrong Pc". It was
**stale AC-3 buffer content**, because the DTS path writes nothing and the peripheral retransmits what
was left behind. That was already the claim in `FINDING_rootcause_license_gate.md` §7.3 — and this
experiment confirms it, while R56's Pc attribution does not survive.

---

## 4. Where the blocker actually is

Given:
- DTS **decode** is alive and streaming (`format: DTS`, `PLAY`, frame count climbing),
- ARM/userspace/HAL/kernel path is proven **format-symmetric** (R4, R6–R8, R10, independent audit),
- the SPDIF TX buffer receives **nothing** for DTS,

the block is in the **DSP-side output/arm stage that feeds the SPDIF TX FIFO** — i.e. between the
decoder output and the header writer. This matches the *earlier*, pre-R56 localisation:

- `FINDING_rootcause_license_gate.md` — the DEC-image SPDIF capability gate
  (`*(u16*)0xB000001E == 5`, else error path `0x22031` that **zeroes the output**), whose error print
  `"Invalid Spdif license:%d, output_spdifSz:%d"` **never appears in any runtime log** — so either the
  gate is not taken, or its print is debug-suppressed.
- R26-B / R26-D — "burst malformed / never emitted".
- R46-FINAL-C — the DTS-path-exclusive MMIO register set and the `0x25EE1` gate (semantics unresolved).

The **next single discriminator** is not another static re-read. It is: **does the DTS arm ever write
the SPDIF TX at all, and if not, which condition suppresses it?** The concrete, already-identified
candidate is the DEC-image gate's error path at `0x22031` (which explicitly **clears the output** —
`sw r0,236(r11)` / `sh r0,236(r11)`), because a suppressed output is exactly what a 0-byte capture means.

---

## 5. What to do next (one change, reversible)

Because the deployed image is already the Pc patch and it is **not** the fix, the honest baseline must
be restored first, then the DEC-side gate tested:

**Step 1 — restore stock SND (remove the ineffective V1 patch).**
```sh
adb root; adb remount
adb shell 'cat /data/local/tmp/asnd_stock.bin > /vendor/lib/utopia/audio_bin/aucode_asnd_r2_MS12V22.bin && sync'
# expect md5 eb879cdc07f510722f19db6d18d77d3c
adb reboot
```
This removes a variable that has been confounding every recent test.

**Step 2 — instrument the DEC-side suppress path.** The DEC image
`aucode_adec_r2_MS12V22.bin` (md5 `4b7e9509…`, currently stock) is the only component that runs the
SPDIF capability check. The candidate is Option L2 from `FINDING_rootcause_license_gate.md`:
neutralise `bg.blesi r24,-0x1,0x22031` at fw `0x21F66` (file `0x21F66`, DEC image has **no** offset
shift) so a negative capability read no longer escapes to the output-zeroing error path — a **4-byte**
change on a **new versioned artifact**, one change at a time.

**Step 3 — measure with the same instrument.** Re-run `dump_spdif_npcm` for DTS. Success criterion for
this experiment is **non-zero bytes** (any bytes) in the DTS capture — that alone proves the suppress
path was the blocker. AVR lock is the *second* criterion, only after bytes appear.

AC-3 regression control after every step is mandatory.

---

## 6. Artifacts produced this session

| Path | Content |
|---|---|
| `r57_npcm/ac3_PATCHED_0022.bin` | 5,695,488 B AC-3 wire capture (Pc=0x0001 ×927) — proves the instrument |
| `r57_npcm/r57_ghidra_out.txt` | Ghidra `-noanalysis` dump attempt (region not analysed → raw-byte method used) |
| `scripts/R57_DtsPc.java` | the Ghidra script (kept for reproduction) |
| `r57_npcm/run_ghidra_r57.sh` | headless runner |
| this file | the refutation + correction |

Wire captures for DTS are **0 bytes** and therefore produce no artifact.

---

## 7. Corrections this document makes to the record

1. **R56 / `FINDING_pc_dts_rootcause.md`**: the "wrong Pc" root cause is **REFUTED** — the correct Pc
   is already deployed and changes nothing. Its static byte analysis remains valid *as a description of
   the code*, but it is **not** the blocker and its patch must not be re-proposed.
2. **`REPORT_PASSTHROUGH_FIX.md`**: unchanged — still discredited (user-confirmed).
3. **`REPORT_final_dts_patch_experiment.md` §1** ("no patch point exists")**: its negative was about the
   `0x26270–0x2661D` gate, which is the wrong region, but its *conclusion* — that the DTS output path
   never arms — is now **independently confirmed** by the 0-byte capture.
4. **The old `dts_01.bin` "frozen AC-3 header"** is now definitively explained as **stale buffer content**
   with **no DTS write at all**, not as a DTS burst carrying AC-3 Pc.
