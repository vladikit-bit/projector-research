# MUSE TASK — the DTS-only rejected DSP command: `cpuDec change state cmd (param=0x2000, val=2)`

**Date:** 2026-09-30 (night) · **Priority:** highest — first codec-specific refusal seen from the DSP
**Files:** `dec_work/dec33_realcode.txt`, `snd_work/snd33_realcode.txt`, `FORENSIC_dts_final_state_20260930.md`
**Do not delete any Ghidra project. Static only, no flashing.**

---

## THE NEW FACT (live, clean harness, `debug_level=4`)

A previously untried mdb knob unlocked a richer R2 log:

```
echo 'r2_log_dbg_option=0x20' > /proc/utopia_mdb/audio     # range 0x10..0x3e, never used before
echo 'dump_r2_log_start=0 9E=0x16 PATH=1 8A(88)=0x1'  > /proc/utopia_mdb/audio
# play DTS 14 s  →  echo 'dump_r2_log_stop=0' > /proc/utopia_mdb/audio
```

With the option set, the DEC R2 log gains a periodic status line and **one error that AC-3
never produces**:

```
DTS:  Err!! receive wrong cpuDec change state cmd!! (type=97, decInCpudec=1,
      runState_req=0, dec_id:0, param=2000, val=2)
AC3:  (no such line — 0 errors in an otherwise comparable capture)
```

`type=97` is the engine/licence byte (0x97, also printed as `type<97>` in the periodic line).
`param=0x2000` is bit 13 of the host capability mask `0x1A002000` seen in
`MDrv_AUDIO_Get_Decoder_Support`. The playCmd oscillation `0x84↔0xc4` occurs under **both**
codecs (DTS 12/11, AC-3 7/7 transitions) and is **not** the difference — do not chase it.

Captures: `C:\firmware_temp\runs\R2_dbg20_dts.log` (525 lines) and `R2_dbg20_ac3.log` (387).

## Q1 (decisive). Who produces and who validates this message

`param=0x2000, val=2` does **not** come from the host: a full `movw #0x2000` sweep of
`utpa2k_stock.ko` finds it only in video/ADAT functions (MHal_PQ_Check_UI_VR,
MDrv_MFD_CPU_ConfigFrameReg, Hal_EARC_TX_SetATOPSetting_AUPLL, ADC calibration …) — **never in
the audio path**. `HAL_AUDIO_SetDecCmd` @0x44C144 writes the play byte to R2 register
`0x112E99 + dec_id` (+2 → `0x112E9B`, the engine byte from the licence chain) and compares
`[g_AudioVars2+0x2510]` with `0xE18D`. So the rejected message is **DSP-internal**:
"cpuDec" = a CPU-side module of the audio DSP talking to the decoder core.

Find, in DEC and SND, the producer and the validator of that command: the string/error text
will be in the firmware (search `.rodata`-equivalent data for the message, then the code that
formats it), and the `param==0x2000` / `val==2` comparison that rejects it. Then answer:
**what makes the validator accept it for AC-3 and reject it for DTS?** If the discriminator
is a field the codec tag writes, that is the block — and it is the first one that is
*observable in a log*, which makes it the best patch candidate in the project.

## Q2. `type=0x97` — what the validator sees

We know `type<97>` is printed in the periodic line and the R2 init prints
`dts m6 init ok, dts_licensee=0, …` **and still decodes**. So the licence is not a gate for
decode — but the validator here *does* branch on `type`. Establish the type byte's provenance
and its values per codec (AC-3 vs DTS), because the rejection may be a type-value mismatch
rather than a licence failure.

## Q3. The other R2 debug options (cheap, do it)

`r2_log_dbg_option` accepts `0x10..0x3E`. Try the neighbours of `0x20` (0x10, 0x18, 0x28, 0x30,
0x38) under both codecs and report which extra fields each reveals. Any field that differs
between the codecs is a candidate; a field that shows the pump latch or the B-latch would close
the "not observable live" gap in `FINDING_latch_verification_20260930.md` §5.

## Rules

* Every claim cites a realcode address or a binary offset; a clean negative is a result — say
  what you searched. Listing format is `addr DIGIT mnemonic` (the digit matters for regex).
* Do not re-derive the verified chain in `FINDING_latch_verification_20260930.md` (16-bit DM
  wrap, single latch object, paradox word with no static writer) or
  `FINDING_host_exonerated_20260930.md` (host exonerated: bypass armed for DTS, SHM params and
  register banks identical).
* Deliverable: the producer/validator addresses for the rejected command, the exact
  discriminator, and whether it is a single compare (patchable) or a chain.
