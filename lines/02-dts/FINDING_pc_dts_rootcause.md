# R56 — SND DSP writes wrong IEC61937 Pc for DTS

> # ⚠️ REFUTED BY LIVE EXPERIMENT (2026-09-13) — READ THIS FIRST
>
> The **causal claim below is REFUTED.** The suggested fix (Pc `0x01` → `0x0B`) was **already
> deployed** on the device (SND image md5 `785fd73b…`, one byte at file `0x1C051`), and DTS still
> produced **0 bytes** on the SPDIF TX (measured twice with `dump_spdif_npcm`; AC-3 control produced
> 5,695,488 bytes in the same session). The DTS decoder ran at full rate (`format: DTS`, `PLAY`,
> 12,569 frames) while the TX received **nothing**.
>
> ⇒ The DTS output path **never reaches this header writer**, so `Pc` is irrelevant. The `Pc=0x0001`
> read statically here is the slot's **stale value left from the last AC-3 burst** — which is also the
> correct reading of the old `dts_01.bin` "frozen AC-3 header".
>
> **Do NOT re-propose the Pc patch.** Verdict: **PATCH HAD NO EFFECT.**
> The static byte analysis below remains valid *as a description of the code*; only the root-cause
> attribution is withdrawn.
>
> Full refutation + the corrected next target (the DEC capability gate at file `0x21F66`) and the
> corrected evidence: `r57_npcm/FINDING_dts_pc_REFUTED_by_experiment.md`.

Date: 2026-09-11

---

Image: aucode_asnd_r2_MS12V22.bin  (1839920 B, MD5 eb879cdc07f510722f19db6d18d77d3c) — STOCK, never patched
Region analysed: [0x16F00:0x1C100], Reko --arch aeon, listing snd_region.reko/snd_region_code.asm

## Address mapping (PROVEN by 5 independent byte anchors)
  file_offset = reko_addr + 0x16F00
  anchors: 0x45A9->0x1B4A9, 0x45B0->0x1B4B0, 0x4F20->0x1BE20,
           0x5058->0x1BF58, 0x41FE->0x1B0FE
  => Reko printed addresses ARE firmware addresses (matches toolchain-validation report)

## The 8-byte IEC61937 preamble slot
  [r10+0x250C..0x2513], byte-swap/payload starts at [r10+0x2514]
    0x250C (sh) = Pa
    0x250E (sh) = Pb
    0x2510 (sh) = Pc   <<<< THE BUG
    0x2512 (sh) = Pd
  Single-writer each; exactly one of two branches runs per codec.

## Function: fn00004F20  (region 0x4F20..0x5140+)
  0x4FEC  r23 = r10 + 0x250C         ; header base
  0x4FF6  r25 = 0xF872               ; Pa
  0x4FFA  bg.sh 0x250C(r10),r25
  0x5002  r25 = 0x4E1F               ; Pb
  0x5006  bg.sh 0x250E(r10),r25
  0x500A  bg.beqi r24,0x0, 0x5142    ; *(r10+0xA0)==0  -> DTS branch

### E-AC3 branch (0x500E..0x5024)
  0x500E  r24 = *(r11-0x7A70)        ; burst length
  0x5012  r25 = 0x6000
  0x5016  bn.sw (r12),r25            ; *(r12) = 0x6000  (slot size)
  0x5019  bn.ori r25,r0,0x15         ; *** Pc = 0x15 = IEC61937_EAC3  (CORRECT) ***
  0x501C  bg.sh 0x2510(r10),r25      ; store Pc
  0x5020  bg.sh 0x2512(r10),r24      ; Pd = length
  0x5024  bg.ori r5,r0,0x6000

### DTS branch (0x5142..0x5161)
  0x5142  r24 = *(r11-0x7A70)
  0x5146  bg.ori r25,r0,0x1800
  0x514A  bn.slli r24,r24,3          ; len <<= 3
  0x514D  bn.sw (r12),r25            ; *(r12) = 0x1800  (slot size, bytes)
  0x5150  bt.movi r25, 0x1           ; *** Pc = 0x0001 = IEC61937_AC3  (WRONG) ***
         bytes: 9B 21  slaspec i16_opcode=0x26 i16_rD=25 i16_simm0_5=1 -> bt.movi r25,1
  0x5152  bg.sh 0x2510(r10),r25      ; store Pc
  0x5156  bg.sh 0x2512(r10),r24      ; Pd = (len<<3)  [bit count]
  0x515A  bg.ori r5,r0,0x1800
  0x515E  bg.j 0x1BF28

### Common tail (0x5028..0x5038)
  r5 -= len ; memset(r23+len+8, 0, r5-8)   ; zero-pad burst to slot
  0x503C  lhz r23,[r10+0x2514]
  0x5040  sfnei r23,0x770B          ; EAC3 sync?
  0x5044  bnf -> return             ; byte-swap payload ONLY for EAC3
  (DTS payload is NOT byte-swapped — correct, DTS core is BE on the wire)

## Reference (authoritative): FFmpeg libavformat/spdif.h
  SYNCWORD1 0xF872, SYNCWORD2 0x4E1F, BURST_HEADER_SIZE 0x8
  IEC61937_AC3   = 0x01
  IEC61937_DTS1  = 0x0B   (512 samples)
  IEC61937_DTS2  = 0x0C   (1024 samples)
  IEC61937_DTS3  = 0x0D   (2048 samples)
  IEC61937_DTSHD = 0x11
  IEC61937_EAC3  = 0x15
  IEC61937_TRUEHD= 0x16
  Pc layout: bits0-6 datatype, bit7 error, bits8-12 datatype-dependent
  Pd: length code (bits or bytes per data type)
  FFmpeg selects DTSn by samples/32: 512>>5=DTS1, 1024>>5=DTS2, 2048>>5=DTS3

## Verdict
  DTS bursts are emitted with Pa=F872 Pb=4E1F **Pc=0x0001 (AC-3)** and a DTS sync word
  inside the payload. The AVR sees an AC-3-declared burst carrying DTS => cannot lock.
  E-AC3 writes the correct 0x15; only the DTS branch is wrong. No license involvement.
  Grep over the whole 21 KB SND region: NO write of 0x0B/0x0C/0x0D as a data type exists.

## Patch candidates (not yet applied)
  B1 MINIMAL, TYPE I ONLY (512-sample core):
     0x5150 (2 bytes) 9B 21  ->  set r25 = 0x0B
     need a 2-byte insn; bt.movi holds only 5 bits (max 31) so 0x0B fits:
       bt.movi r25,0x0B  -> opcode 0x26, rD=25, imm5=0x0B=11
       word = (0x26<<10)|(25<<5)|11 = 0x9B2B   bytes 9B 2B
  B2 TYPE-SELECTING (correct for all DTS variants): replace the branch region with
     a 3-way select on core frame size (512/1024/2048 -> 0x0B/0x0C/0x0D).
     Requires knowing where the sample count lives — NOT yet located in SND.
