
---

# PHASE LOG — P-A deployment + runtime findings (2026-09-13, GLM53 session)

## 1. P-A deployed and verified

| item | value |
|---|---|
| artifact | `r57_npcm/artifacts/aucode_adec_r2_MS12V22_PA_45FF5.bin` |
| md5 | `afde18ec92ffe0a960390a423d7f63c2` (sha256 `98a9a631…`) |
| source | stock DEC `4b7e9509…` (D1 implicitly reverted) |
| diff | exactly 3 bytes @ 0x45FF5: `0c 03 28` → `2c 00 17` (`bn.sw 0x28(r3),r0` → `bn.j 0x4600C`) |
| deployed | md5 verified in place, bytes `xxd`-verified, reboot uptime-verified (7219→62s) |
| backup | `/data/local/tmp/adec_ACTIVE_20260913_230801.bin` (D1 image `149f4275…`); stock also staged (`adec_stock.bin` `4b7e9509…`) |
| SND | stock `eb879cdc…` (untouched) |

## 2. AC3 regression: **PASS**
- `test_ac3_51.ac3` → 4,177,920 B wire capture, **680 IEC61937 preambles, 100 % data-type 0x01 (AC-3)**, 0x1800 stride. Dolby Digital passthrough fully intact under P-A.

## 3. DTS: **DTS PATH CHANGED — DTS BURSTS APPEARED ON THE WIRE FOR THE FIRST TIME**
- `dump_spdif_npcm` during DTS playback captured **1,253,376 B** (previous best: 0 bytes, always).
- Wire content: **valid IEC61937 DTS bursts** — Pa=0xF872, Pb=0x4E1F, **Pc=0x000B (DTS Type I)**, **Pd=0x3EE0 (=2012 B × 8 = exactly Kodi's framesize)**, payload = **real DTS core frames** (0x7FFE8001 sync, byte-swapped like AC3), burst stride **0x800 (2048 B = padded 2012-byte frame)** — precisely the selector-0x200 emission the P-A patch enables.
- Payloads verified live: full 2012-byte payloads differ burst-to-burst and match frames of the source file (found e.g. at file offset 0x7dc00 / 0xfbfdc).
- Cadence: 114 valid bursts per 446 slots — **runs of exactly 38 valid, then 166 zero slots, period 204 slots ≈ 2.18 s** (one non-PCM ring wrap).
- AVR did not lock (user heard nothing): the stream is silent ~82 % of the time — 0.4 s of DTS per ~2.2 s cycle cannot hold a DTS lock.

## 4. Root cause of the 0.4 s window — DEC R2 log (runtime, `dump_r2_log_start=0 9E=0x16/0x23/0x28`)
```
r2_decoder_houseKeeping: decType change !! type[0] -> type[81] !!      ← session starts on the passthrough engine
   (P-A's SDO path emits valid DTS bursts during this phase)
[tick 179168 ≈ 2.2 s] decType change !! type[81] -> type[4] !!         ← THE FALLBACK
   r2_decoder_select: decType:0x4, key1=0xede227, key2=0xfffefd
   dts m6 hook ok / dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0
steady state: pcm:5120 es:0 cB:0                                       ← decode continues, compressed output = 0
```
Security dump (9E=0x28): `HASH key = ede227 / HASH key2 = fffefd / OTP bounding = ff00 / DEC 0: unsupported otp / DEC 1: supported`.
Licensee struct fields (read by the m6 init from DM 0x1C012734+idx·0x290) verified **zero at runtime** from the host mirror.

## 5. Corrected causal chain (supersedes the audit's single-gate model)
```
Kodi/HAL OK → Codec:9 → DEC ES buffer OK → session starts on engine 0x81 (passthrough)
  → P-A-fixed prepare/emit SDO path emits valid DTS bursts (PROVEN on wire)
  → ~2 s in: houseKeeping switches session to the DTS-m6 decode engine (type 81 → 4)
     driven by the DTS license verdict: OTP bounding says "unsupported otp" for DEC 0
     → licensee fields stay 0
  → m6 engine decodes to PCM only (es:0) → SDO output starves → wire silent → no AVR lock
```
The DSP images contain the whole DTS-SDO emission machinery and it **works** (proven end-to-end to the physical wire). The single remaining blocker is the **DTS license verdict inside the DEC** (OTP-based), which (a) zeroes the licensee state and (b) keeps the m6 session in decode-only output.

## 6. Next-patch candidates (ranked; NOT built)
| # | target | idea | evidence |
|---|---|---|---|
| N-1 | DEC OTP-verdict function (prints "DEC %d: unsupported otp"; strings file 0x14395C–0x1439A4 → rodata VA 0x028109xx) | force the "supported" verdict for DEC 0 → licensee fields get written → m6 session emits es:2012 | strings located; print-site identification blocked by nonlinear rodata VA↔file mapping — needs a Ghidra pass with a proper rodata memory segment (VA 0x02810000 ↔ file 0x143000-ish), then the bit-test cascade patch |
| N-2 | licensee-field writer (DM 0x1C012734+idx·0x290, no direct store in the listing) | find the param-blob/loader writer; feed licensee≠0 | fields proven zero at runtime; no composed-address store found — writer is the IDMA/param loader (ARM-supplied blob?) |
| N-3 | ARM-side (utpa2k.ko): the decoder-select that sends decType=4 + hash keys (0xede227/0xfffefd) at the m6 switch | keep the session on engine 0x81 for DTS-passthrough (the emission there is proven working) | houseKeeping reacts to ARM config; ARM patchable; needs the codec-9 SetDecodeSystem/param trace |
| N-4 | the per-frame es:0 decision in the m6 session | force compressed output per frame | the "es:" counter is the output-size handoff; site not yet located |

## 7. Session verdict (interim)
**DTS PATH CHANGED BUT NOT FIXED** — P-A is proven correct and valuable (it opened the producer path and produced the first valid DTS wire output ever observed on this device), but the remaining blocker moved definitively to the DEC-side **DTS license verdict** (OTP-driven engine switch 81→4 + licensee=0). AC3 passthrough remains fully intact.

---

# PHASE LOG — N-1 static search (2026-09-14, GLM53 session, read-only)

## Achieved: the OTP verdict chain is now structurally mapped
1. **Verdict print function = DEC 0x1e3e0** ("DEC %d:" + verdict). String table SOLVED:
   rodata delta for the verdict cluster = **0x26D0000** (file 0x14395C-0x1439E2 ↔ VA 0x0281 395C-0x28139E2):
   `DEC %d:` 0x396F · `supported` 0x3977 · `unsupported format` 0x3981 · `unsupported otp` 0x3994 ·
   `unsupported hash` 0x39a4 · `, muted` 0x39b5 · `, force syncing STC` 0x39bd · `, disable output` 0x39d1.
   (Corpus discrepancy "0x26CD000 vs 0x26D0000" resolved: BOTH exist — different string sections.)
2. **Verdict selection = a config word per decoder**: `*(0x08C15840+0x120)` (DEC 0) / `+0x470` (DEC 1).
   Values: 1→"unsupported hash", 2→"unsupported otp", 3→"unsupported format", other→"supported".
   Our runtime: DEC0 word=2, DEC1 word=other — exactly the log.
3. The word is COPIED at DEC 0x22EC8 from the per-decoder DM param block
   (0x1C011000+0x26B0+idx·0x290+0x10C) — preceded by cache-invalidate calls (0xE0CF/0xDFB1)
   = the classic "ARM wrote it via DMA" pattern.
4. **The verdict word is DIAGNOSTIC-ONLY on the DSP** (its only reader is the print). Patching the
   print path (cosmetic verdict) would NOT write licensees → N-1-as-"patch the print" REJECTED.
5. **The licensee fields (DM 0x1C012734+idx·0x290) have NO writer in the entire DEC listing**
   (fixed the scanner bug: memory disp = bits2-15 ×4; still zero writers) ⇒ they arrive from the
   **ARM-side config blob** (IDMA/SHM), which also carries the verdict class.
6. ARM-side hunt: a candidate writer function in utpa2k.ko (~0x3DBA00, stores 0xDCC/0xDD4/0xDD8/
   0xDEC/0xDF0 with license-shaped values: 30000/1001/15/24/25 per DTS-rate case 1500/2500/3004/
   3005/3008) was REFUTED — it has a symbol: **`mfeM4VE_Init`** = the MPEG-4 video encoder init.
   False lead; the byte-offset coincidence was chance.

## Consequence for N-1
The functional patch surface is the **ARM-side builder of the DEC config blob** (utpa2k.ko):
the code that classifies the DTS license (verdict class = 2 "unsupported otp" written into the
blob) and/or fills the licensee fields as 0. Patching it to report "supported" should make the
m6-init take the supported path (licensee≠0 → es:2012), keeping the P-A producer path alive.

## Next concrete steps (fresh context recommended)
1. Ghidra-analyze utpa2k.ko fully (25 MB; import was started but needs an unattended run).
2. Find the DEC-SHM config-blob builder: anchor = g_virDecR2shm (.bss 0xEFBF4, R35 method);
   enumerate MOVW/MOVT+LDR sites; map per-decoder struct stores (the 0x290-stride blob with
   verdict class and licensee fields).
3. Locate the classification (verdict=2) decision; patch it to 0 ("supported") — smallest site.
4. Deploy per the established procedure; runtime comparison per the N-1 brief
   (81→4 switch, licensee≠0?, es>0?, wire continuity, AVR lock).

## Device state
P-A still deployed (DEC `afde18ec…`), SND stock — unchanged all phase. No new artifacts built.

---

# N-1 NARROW PASS RESULT (2026-09-14, session 2) — A–E with explicit missing link

## A) ARM config-builder function: **NOT LOCATED.**
Located instead: ALL 9 g_virDecR2shm accessor sites (MOVW/MOVT pairs @ 0x457E34, 0x458070,
0x458440, 0x458968, 0x4591D0, 0x4592AC, 0x45A43C, 0x45B7C8, 0x45C0B8 .text-rel) =
the param-block API family (init_SHM_param/Get_INFO/Set_COMMON/Set_PARAM/Get_PARAM/…,
addend=0 → plain base loads). Their param block = 0x8B8 B; the verdict class (SHM+0x27BC)
and licensee fields (SHM+0x2734+) are OUTSIDE that block ⇒ delivered by a different transport
(R2→R2 mailbox "SE R2->DEC R2 MBOX" or IDMA) — sender NOT identified.

## B) DEC0 verdict-field writer (DSP side): **0x22EC8** (`bg.sw_0 0x120(r10),r30` — base 0x08C15840
built at 0x22D7E; source = DM 0x1C0127BC+idx·0x290 field; DEC1 twin = 0x24B27).
But the verdict word is **diagnostic-only** (single reader = the print at 0x1e3f7) ⇒ B is a copy, not the cause.

## C) The condition producing value 2: **NOT FOUND.** It computes upstream of the DM param block,
inside the transport sender (mailbox/IDMA), outside both analyzed images.

## D) Does the same condition control licensee population: **UNKNOWN/UNPROVEN.**
Licensee fields (DM 0x1C012734+idx·0x290) have ZERO writers in the whole DEC listing
(verified with idx strides 0..3 and corrected disp encoding) ⇒ they arrive pre-zeroed through
the same transport; the coupling to the verdict class is plausible but unproven.

## E) Smallest causal patch candidate: **CANNOT BE RESPONSIBLY PROPOSED YET.**
Two hypotheses for the next session (no patch built, none deployed):
(i) the transport sender (ARM-side mailbox write "SE R2->DEC R2 MBOX" path or IDMA param loader)
    — anchor for the search: the DEC-side mbox print strings (file 0x14390C/0x143921) and the
    mbox-register writes in the DEC image (the receiver side) ⇒ reverse the mailbox payload
    format, then find the ARM sender;
(ii) verify empirically whether a "supported" verdict (0 in the blob class field) makes ANY
    DEC code write licensees — currently unsupported by static evidence.
Also open: a reliable host-side read of the DM param block (the mirror↔DSP-VA mapping is
unproven — cell 0x27BC read 0 while the DSP log proves the field = 2).

## Runtime anchor unchanged (P-A baseline)
81 → valid DTS bursts; ~2 s → type[4]; "DEC 0: unsupported otp"; licensee=0; es:0; wire silent.

---

# N-1 STEP 1-3 RESULT (2026-09-14, session 2, continued) — transport partially traced

## New VERIFIED facts
1. The "HASH key / HASH key2 / OTP bounding" values are **DATA FIELDS in the DEC's local data
   window** (DSP addresses 0x0001_0838 / 0x0001_081C / 0x0001_083C), not MMIO. The debug dump
   block (DEC 0x1E280-0x1E3DD) prints them; **ZERO writers exist in the whole DEC image**
   (55 read sites, 0 writes) ⇒ delivered by the loader/mailbox from outside.
2. Mailbox register protocol (DEC-side diagnostics 0x1E2B9-0x1E363): MB-out = 0x0001_0C20-0x0C2C,
   MB-in = 0x0001_0C00-0x0C0C (4+4 32-bit words), plus the state bit test at 0xB000_000A-analog
   (0x0001_000A, bit 4 of halfword — mailbox busy flag).
3. The per-decoder config block (0x1C0126B0+idx·0x290; verdict class at +0x10C) is read by the
   config-apply function (0x22DB8-0x22EC8) after cache-invalidate — still ZERO DSP writers of the
   class field ⇒ ARM/loader-delivered.
4. The hash keys (0xede227/0xfffefd) are **NOT constants anywhere in utpa2k.ko** (.text/.data
   word/movw-movt scans: 0 hits for key1; key2 only coincidental data patterns) ⇒ they are
   runtime-computed or read from the efuse controller on the ARM, then transported.

## Where the trace STOPS (the missing link, precisely)
The ARM-side sender that composes the per-decoder config message (carrying hash keys,
OTP bounding, verdict class=2, licensee=0) and delivers it into DEC DM 0x1C0126B0+ —
i.e. the writer behind the DEC's mailbox receiver (MB-in 0x0001_0C00-0x0C0C) or the IDMA
param loader — is NOT identified. Candidate mechanisms, in probability order:
(a) the MAD2/IDMA parameter loader driven by utpa2k's HAL_AUDSP_DspLoadCode family
    (the DEC image + attached param tables are loaded from utpa2k .data; the per-session
    keys/class may be patched into the table before the IDMA kick);
(b) a dedicated mailbox write path (the "SE R2" intermediate core relays ARM→DEC).

## Next concrete anchor (for the next session)
- Reverse the DEC mailbox receiver: the MB-in register reads at 0x11F14-0x11FBC and the
  dispatch around 0x16100-0x16210; find the message opcode table and the copy destination
  (proves receiver side fully).
- On the ARM: HAL_AUDSP_DspLoadCode (0x46f5a8) and the DEC-load path — locate the param-table
  base in .data and check whether the verdict class/licensee bytes are part of the static
  table (then the patch = table bytes) or runtime-written (then patch the writer).

---

# PRIORITY-1 RESULT (2026-09-14, session 2 end) — DspLoadCode / static-table search

## Verified
1. `HAL_AUDSP_DspLoadCode` (0x46f5a8) copies the WHOLE DEC image blob (length 0x1E401C,
   the selection at 0x46fb14-68: cmp r1,#4 → sizes 0x1B6930/0x1E401C — matches R8) to the DSP.
   ⇒ the DEC's DM initial content (if any) IS inside dec_full.bin.
2. The DEC boot stub (blob 0x0-0x206) contains the DM-init copy: fragmentary decode gives
   **r6 = 0xD0000** (blob data-region base — corrects R8's "0xd000") and **r3 = 0x0001_0000**
   (the local data window base). ⇒ hypothesis: local window addr 0x0001_0000+N ↔ blob 0xD0000+N.
3. TEST FAILED the naive mapping: blob[0xD0838/0x081C/0x083C] ≠ runtime hash/OTP values
   (0x34e1222a/0x70c0ea00/0x5c00e6ce vs 0xede227/0xfffefd/0xff00).
   Interpretations: (a) the stub copy table has different src/dst segmentation (needs full
   phase-aware stub decode), or (b) the key/OTP fields are runtime-written by the ARM per session
   (the keys ARE session values — "key1/key2" printed by r2_decoder_select).
4. Host mirror cells 0x0838/0x081C/0x083C/0x27BC read 0 (idle) — mirror↔DSP-VA mapping remains
   unproven; mirror unusable for these fields.
5. The verdict-class static-presence scan (word==2 at X+0x10C with the 0x290 stride, X in the
   data region) = 282 noisy candidates — inconclusive without the exact DM↔blob mapping.

## A–E status (unchanged conclusions, sharper targets)
A) ARM config-builder: still not located.
B) DSP verdict copy: 0x22EC8 (diagnostic-only).
C) verdict=2 producer: upstream — either runtime ARM-written into DM, or static in the blob.
D) licensee transport: same unknown.
E) patch: none.

## EXACT next step (bounded, one tool run)
Fully decode the DEC boot stub (0x0-0x206) with `scripts/aeon_decode.py` (phase-aware mixed
width) to extract the DM-init copy table (dst/src/len list). That yields the exact DM↔blob
mapping, which then locates the per-decoder config record (0x1C0126B0+idx·0x290, licensees
at +0x84.., verdict class at +0x10C) inside dec_full.bin. Then compare DEC0 vs DEC1 verdict
bytes: if DEC0=2/DEC1=supported are LITERAL in the blob → data-only patch candidate (change
the DEC0 verdict byte to 0); if not literal → the class is runtime-written and the sender hunt
(mailbox/IDMA) resumes.

---

# N-1 STEP 3 RESULT (2026-09-14, session 2 end) — SND-side classifier FOUND

## Decisive model correction
"SE R2" = **Sound Engine R2 = the SND core** (not a security engine). The mailbox peer of the
DEC is the SND. The config blob (verdict class + licensees) is composed on the SND and
delivered to the DEC through the 0xB000_0Cxx mailbox registers.

## The classifier located (SND image, aeon_decode.py phase-correct decode of 0x1F4A0-0x1F590)
```
1F4E7  r23 = 0xB000_0000                       (SND MMIO base)
1F4F3  r24 = *(0xB000_083C)                    ← OTP bounding (efuse mirror)
1F4FA  r25 = *(0xB000_0838)                    ← HASH key (efuse mirror)
1F4FE  andi r24, r24, 0x2080                   ← keep bits 13 (0x2000) and 7 (0x80)
1F505  sfeqi r24, 0x2000                       ← condition: bit13 SET && bit7 CLEAR
1F521  r24 = 0xB000_0C20 (MB-out)
1F525  lbz r25, 0(r24); 1F52F ori r25,r25,0x2; 1F532 sbz 0(r24),r25   ← mailbox notify bit1
1F535-1F53E  lbz r11, 0xB000_0C05; andi 1; compare vs *(r3+4)         ← config flag check
1F545-1F55D  r3 = 0x4E0000+27348 → jal 0xC55E / 0xCA42 (the reset/notify helpers)
1F56A  jal 0x1F44F                              ← the classification continuation
```
The same function block also touches MB 0xC05/0xC04/0xC0C. The OTP bit test
`(bounding & 0x2080) == 0x2000` is ONE of the license conditions (runtime bounding = 0xff00
passes THIS one — 0xff00&0x2080 = 0x2000 — so the failing condition is elsewhere in the
classifier chain: the DTS-core/DTS:X bits are tested separately).

## CRITICAL tooling correction (invalidates earlier raw-scan "0 hits")
**The AEON movhi immediate = bits (5,20) of the word, value = imm<<16.**
Bytes C2F60001 = movhi r23, 0xB000 (imm field (w>>5)&0xFFFF = 0xB000), NOT imm=0x0001.
All raw-byte composed scans that used bits (0,15) for movhi were blind. The DEC listing-based
conclusions remain valid (they used the Ghidra-rendered text, which is correct).

## Deliverable A–F status
A) DEC receiver: MB-in 0x0001_0C00-0C0C, diagnostics 0x1E280-0x1E3DD. B) protocol: 4-word
mailbox + busy bit 0x0001_000A.4. C) DEC-side destination: config-apply 0x22DB8-0x22EC8
(DM 0x1C0126B0+idx·0x290 → working 0x08C15960). D) **SENDER = the SND core; classifier
function ≈ 0x1F440-0x1F590 in snd_full.bin** (reads OTP 0x83C + HASH 0x838, tests 0x2080
mask, writes MB 0xC20/0xC05). E) verdict=2 producer: inside the SND classifier chain — the
exact failing sub-condition and the licensee-zero writes are in the remaining part of the
function 0x1F440-0x1F590 (phase-correct decode of the whole function is the next bounded step).
F) patch: not yet — next step is the complete decode of the SND classifier function, then the
smallest branch/data change (the OTP bit test at 0x1F4FE/0x1F505 is the first concrete candidate:
force the licensed path).

## Device state: unchanged (P-A active, AC3 PASS, SND stock).

---

# SND CLASSIFIER DECODE STATE (2026-09-14, checkpoint before context compaction)

## Decoded flow so far (aeon_decode.py, phase-anchored — VERIFIED fragments)
```
0x1F4E7  r23 = 0xB000_0000                    (SND MMIO base)
0x1F4F3  r24 = *(0xB000_083C)                 ← OTP bounding (efuse mirror)
0x1F4FA  r25 = *(0xB000_0838)                 ← HASH key (efuse mirror)
0x1F4FE  andi r24, r24, 0x2080                ← keep bits 13 + 7
0x1F505  sfeqi r24, 0x2000                    ← condition: bit13 SET && bit7 CLEAR
0x1F521  r24 = 0xB000_0C20 (MB-out reg)
0x1F525/0x1F52F/0x1F532  lbz/ori 0x2/sbz     ← MB notify bit1 SET (message pending)
0x1F535  lbz r11, 0xB000_0C05                 ← MB flag byte
0x1F538  lwz r23, 4(r3)                       ← r3 = config struct (0x4E6AD4-ish); *(r3+4)
0x1F53B  andi r11, 1
0x1F53E  beq r23, r11 → 0x1F55C               ; config in sync → join
0x1F542  beqi r11, 1 → 0x1F56A                ; MB flag==1 → message pending → jal 0x1F44F
0x1F545-0x1F559  r3 = 0x4E0000+0x6AD4=0x4E6AD4; jal 0xC55E; jal 0xCA42 (reset helpers); sw 4(r10),r11
0x1F55C  join
0x1F575  NEW FUNCTION prologue (bn.addi r1,-56)
```

## Message processor 0x1F44F (called when MB flag==1) — partially decoded, LINEAR PHASE DRIFTS
Verified fragments:
```
0x1F44F  bn.addi r1,r1,-48        (inner function prologue)
0x1F45F  lwz r23, 40(r3)
0x1F462  sw  36(r3), r24
0x1F465  sw  44(r3), r23          ; config field swaps at r3+36/40/44
0x1F484  ori r25, r0, 0xBB80      ; 48000
0x1F489  jal 0x2778D
0x1F491  jal 0x10786E
```
The middle drifts (undef) — needs flow-following decode (Ghidra SWEEP) instead of linear.

## Ghidra script blocker + fix status
- DumpAEON_R29/SND01/SND02 fail: "Failed to get OSGi bundle" — javac errors are HIDDEN by
  the headless logger. OSGi cache cleared (AppData/Roaming/ghidra/.../osgi/compiled-bundles)
  — did NOT fix. Manual javac inconclusive (classpath). NEXT: write the sweep script using the
  EXACT DEC35 skeleton (which compiled+ran on ghidra_dec project) — differences to check:
  unused imports are fine; suspect a missing jar in the sndr27 environment or script-arg parsing.
  Alternative: run SND02_Sweep on the ghidra_dec project?? no — wrong program. Or use
  aeon_decode.py with MORE anchors (each branch target re-syncs the phase) — cheapest.

## NEXT STEPS (in order)
1. Decode 0x1F44F body with per-branch anchors (0x1F452, 0x1F457, 0x1F45B, 0x1F468, ...) OR fix
   the Ghidra sweep script → find where the MB payload (verdict class + licensees) lands.
2. The classifier condition chain: the 0x2080-mask test at 0x1F4FE/0x1F505 PASSES at runtime
   (0xff00&0x2080=0x2000) — the FAILING condition (producing verdict=2/licensee=0) is a
   DIFFERENT test — find it in the 0x1F44F body or the 0x1F56A-continuation.
3. Then: minimal patch (branch/data), verify per protocol, deploy, AC3 control, DTS wire, AVR.

## Device state: P-A active (DEC afde18ec), SND stock, AC3 PASS. No changes this phase.

---

# CHECKPOINT (2026-09-14, session 2 — pre-compaction)

## Last negatives (do NOT revisit)
- SND 0x29FA6 "movhi 0x1c01" = phase false positive: at the correct phase it decodes
  `bg.sfnei r1, 7169` (a compare). The SND does NOT construct the 0x1C010000 shared-DM base
  via a simple movhi — if the SND writes the DEC config block, it goes through the mailbox
  (0xB000_0Cxx writes at SND 0x0F5AF/0x1006A/0x16A5C/0x16BF7/0x1E88F/0x1F38F/0x1F532) or IDMA.

## Established SND classifier facts (verified fragments)
- SND fn ≈0x1F380-0x1F590: reads OTP bounding (0xB000_083C) + HASH key (0x0838);
  test `(OTP & 0x2080) == 0x2000` at 0x1F4FE/0x1F505 (PASSES at runtime 0xff00 — the FAILING
  condition is a different test, elsewhere in the chain);
- MB-out 0xB000_0C20 notify bit1 set at 0x1F521-0x1F532 (message pending);
- config compare: 0xB000_0C05 bit0 vs *(r3+4), r3 = 0x4E0000+0x6AD4 = 0x4E6AD4;
- dispatch: config-sync → join; MB-flag==1 → jal 0x1F44F (message processor);
  else → reset helpers 0xC55E/0xCA42 with r3 = 0x4E6AD4;
- 0x1F44F (message processor, prologue r1-48): config field swaps at r3+36/40/44,
  rate 48000 setup, calls 0x2778D / 0x10786E / 0xCA42 / 0xC55E, tail 0x1F4E4 → jal 0x39114.
- Linear decode of 0x1F44F body DRIFTS (mixed-width + unmodelled ops op24-28/2A-2F) — needs
  flow-following (Ghidra) or per-branch anchored decode.

## THE OPEN QUESTION (precise)
Where does the SND compose the per-decoder config message that the DEC receives
(verdict class + licensee fields, landing in DEC DM 0x1C0126B0+idx·0x290)?
- The SND MB-out writes (7 sites above) are the transport candidates;
- The message COMPOSER (which fills verdict=2/licensee=0 from the OTP classification)
  is the remaining unknown. The classification INPUT = OTP bounding 0xB000_083C (0xff00)
  read at 0x1F4F3 and ~11 other SND sites (0xD0B2, 0xE7E5-0xEA08, 0x16A20, 0x1EFE1, 0x1F4F3,
  0x28328, 0x2C3DB, 0x2E1FE per R46b ghidra_writers.txt).

## Next session plan (in order)
1. Ghidra SWEEP dumps of the SND MB-writer functions (0x0F5AF/0x1006A/0x1E88F/0x1F38F/0x1F532
   contexts) — identify which one carries the per-decoder config payload (verdict class bytes).
2. Trace that payload's source: the classification of OTP bounding → verdict class 1/2/3/0.
3. Minimal patch: force verdict=0 ("supported") for the DEC-0 config — either at the SND
   classification branch or at the SND MB-payload composition.
4. Deploy per protocol (backup → remount → cat → md5 → uptime-verified reboot → adb root),
   AC3 wire control, DTS wire + R2-log (81→4? licensee? es?), AVR panel.

## Device state: P-A active (DEC afde18ec…), SND stock, AC3 PASS. Everything reversible.

---

# CHECKPOINT 2 (2026-09-14, session 2 — compaction imminent) — THE DECISIVE 12 BYTES

## The classification zone resolved to 0x1F505-0x1F525 (SND classifier)
```
1F4FE  andi r24, r24, 0x2080      ; OTP bounding keep bits 13+7
1F505  sfeqi r24, 0x2000          ; F = (OTP&0x2080 == 0x2000)   [runtime 0xff00 → F=TRUE]
1F509  <2-byte insn, bytes 89 43>  ; unresolved (unmodelled op 0x25?)
1F50B  bn.bf → 0x1F55C            ; branch-if-F-SET → JOIN (skip classification!)
1F50A+ FALL-THROUGH (F clear) → the classification/message path (drift: divu/ori 0x400/and/sw 24(r11))
1F521  r24 = 0xB000_0C20; 1F525 lbz; 1F52F ori 0x2; 1F532 sbz  → MB notify bit1 SET
1F535-1F53E  MB 0xC05 bit0 vs *(r3+4) compare → dispatch
```
**POLARITY PROBLEM TO CLOSE**: runtime OTP=0xff00 → (OTP&0x2080)==0x2000 → F TRUE → bf TAKEN →
join. But the runtime verdict IS 2 (unsupported). Two possibilities:
(p1) the bf-target 0x1F55C path LEADS to the verdict=2 composition later (0x1F55C → 0x1F55D
     blesi r28,6 → 0x1F575 new function!), or
(p2) the bytes 89 43 at 0x1F509 invert the flag (a flag-inverting 2-byte op between sfeqi and bf).
RESOLVE: decode bytes at 0x1F509 (2B: 89 43) via the slaspec fields, and dump 0x1F55C-0x1F575 +
0x1F575-0x1F620 (the new function = the suspected verdict composer).

## THE 12 BYTES to decode (from snd_full.bin)
file offsets 0x1F505-0x1F525:
05 42 32 C1 [1F505 sfeqi] · 89 43 [1F509 ?2B] · 20 01 ?? [1F50B bf→1F55C] ·
then the fall-through/join regions per the flow above.

## Full context recovered so far (do not re-derive)
- SND classifier fn ≈0x1F380-0x1F590; OTP 0xB000_083C, HASH 0xB000_0838 (efuse mirrors, 0 writers).
- 5 config slots (stride 0x800) at r10+0x838..0x2838; pointer table r10+0x3038-0x3060.
- r3 config struct = 0x4E0000+0x6AD4 = 0x4E6AD4; MB-out 0xB000_0C20-0C2C; MB notify bit1 at 0xC20.
- MB flag 0xC05 bit0: 0 → sync (join); 1 → message pending → jal 0x1F44F (processor:
  config swaps r3+36/40/44, rate 0xBB80, calls 0x2778D/0x10786E/0xCA42/0xC55E, tail → jal 0x39114).
- DEC side: config-apply 0x22DB8-0x22EC8 reads DM 0x1C0126B0+idx·0x290 (class at +0x10C,
  licensees at r10+0x1734 i.e. block+0x84) after cache-invalidate 0xE0CF/0xDFB1; verdict word
  0x08C15960 diagnostic-only; DEC MB-in 0x0001_0C00-0C0C; diagnostics 0x1E280-0x1E3DD.
- Tooling: aeon_decode.py + self-healing walker (works); Ghidra OSGi broken in this env
  (both caches wiped, still fails; manual javac with full jar classpath COMPILES the script fine).

## After resolving the 12 bytes → the patch
If verdict=2 composition is in the fall-through/join path → smallest branch/data change forcing
the DEC-0 "supported" verdict (either the 0x2080 test or the message-composition branch).
Then: verify (builder + decode-back + diff), deploy per protocol, AC3 control, DTS wire+R2-log, AVR.

---

# CHECKPOINT 2 ADDENDUM — final decode of the decisive zone (session 2 end)

## Fully decoded 0x1F505-0x1F52F (aeon_decode.py):
```
1F4FE  andi r24, r24, 0x2080     ; r24 = OTP & 0x2080
1F505  sfeqi r24, 0x2000         ; F = (OTP&0x2080 == 0x2000)   [runtime 0xff00 → F=TRUE]
1F509  <2B unmodelled op 0x22-family, bytes 89 43>   ; may modify F — UNKNOWN
1F50B  bn.bf → 0x1F55C           ; branch-if-F-SET → join       [runtime: TAKEN]
1F50E-1F51D (fall-through, F clear path):
       ori r24, r24, 0x400; bn.and r25, r25, r24 (HASH key & (OTP|0x400));
       bg.beq r25, r24 → 0x1F55C                       ; second condition!
1F521-1F52F  MB-out 0xC20: lbz/ori 0x2/sbz (notify bit1 SET)
```
Runtime flow: F TRUE → 1F50B TAKEN → 0x1F55C join. The fall-through second test
`(HASH & (OTP|0x400)) == (OTP|0x400)` never runs at runtime.
⇒ The verdict=2 composition is NOT in this fall-through; it is in the JOIN path
(0x1F55C → 0x1F55D blesi r28,6 → 0x1F575 new function) or was already in the DM block.

## NEXT SESSION — exact first actions
1. Decode 0x1F509's 2-byte op (bytes 89 43, unmodelled op 0x22 family) against the slaspec.
2. Decode 0x1F55C-0x1F620 (the join + new function 0x1F575, prologue r1-56) — find the
   verdict-class/licensee composition for the DEC message.
3. The 0x2080 test passed at runtime — so the "unsupported otp" verdict comes from a
   DIFFERENT check: candidates = the unmodelled 0x1F509 op, the join path, or the 0x1F44F
   message processor. Follow the join path first.
4. Then: minimal patch → verify (builder+decode-back+diff) → deploy per protocol →
   AC3 control → DTS wire+R2-log → AVR panel.

## Everything else remains as documented above (P-A active, AC3 PASS, SND stock, all reversible).

---

# CHECKPOINT 3 ADDENDUM (2026-09-14) — the DTS-license bit test FOUND in the SND config fn

## The SND function 0x1F575 (called from the join) contains the license-bit test:
```
1F67B  bg.jal 0xB6D8F        ← the license/capability query (same helper the DEC-side
                                0x1F67B-analog gate uses — R53: "gate = return of 0xB6D8F")
1F67F  bg.andi r11, r11, 0x200   ← *** isolate BIT 9 = the DTS license flag ***
```
Surrounding context (0x1F654-0x1F69C): config counter compares (r10+0/24), prints setup
(r10+12344/12368 = the config-pointer table from the fn head!), byte stores, sfnei r25,2826
(0xB0A = a config tag), mfspr 0x5001 (the timer), r10+14416/14420 loads/stores.

## Consolidated classification chain (all VERIFIED fragments)
```
SND fn 0x1F380-0x1F590:
  OTP bounding (0xB000_083C) + HASH key (0x0838) reads;
  test1 (OTP&0x2080)==0x2000 → join        [runtime 0xff00 → PASSES]
  test2 (HASH&(OTP|0x400))==(OTP|0x400)    [runtime: skipped]
  → join 0x1F55C → blesi r28,6 → fn 0x1F575
SND fn 0x1F575 (the config apply/compose):
  ... jal 0xB6D8F; andi r11,0x200          ← the DTS-license bit 9 TEST
  → verdict class + licensee fields → the per-decoder config → MB/mailbox → DEC
```
NEXT: decode 0xB6D8F (what returns bit 9: reads the OTP bounding 0xB000_083C? the 0x200 bit of
OTP bounding = 0xff00 → bit9 (0x200) = 0 ✗ NOT SET = "unsupported otp"!!) — 0xff00 has bits
8-15 set: bit 9 = 0x200 IS set (0xff00 & 0x200 = 0x200 ✓)... wait: 0xff00 bits 8-15: bit9=1.
Hmm — then bit9 SET → licensed?? The classification outcome (verdict=2) must come from the
CONSUMERS of bit9 elsewhere, or 0xB6D8F reads a different source. Decode 0xB6D8F next.

---

# CONSOLIDATION (2026-09-14, session 2 final) — the two-image model unified

## KEY RECOGNITION
The SND-image function 0x1F575 (with jal 0xB6D8F at 0x1F67B, the 0xCC9B/0xC49D calls at
0x1F754/0x1F75C, the gate 0x25EE1 chain) = **the SAME function family the R46-R54 era
analyzed as "the SND DTS output-config"** (mstsound handler, latch re-arm, Predicate A).
This session's "SND classifier" decode = re-entering that (declared-exhausted) envelope —
BUT with two NEW verified facts:
1. The SND→DEC transport: SND writes 0xB000_0Cxx mailbox (7 sites); DEC reads 0x0001_0Cxx
   (MB-in) and the config-apply pulls the DM param block after cache-invalidate.
2. The DEC's verdict class/licensee fields = SND-composed values (the DEC never writes them).

## The unified two-image DTS-passthrough chain (final model)
```
ARM (utpa2k): MI_AUDIO config (codec 5/9, keys, params)
  ↓ SND R2: mstsound handler 0x1F575 → config compose (0x1F44F processor;
    OTP bounding 0xB000_083C + HASH 0x0838 read; bit-9 test via 0xB6D8F;
    the classification state = the mstsound framework state)
  ↓ mailbox 0xB000_0C20-0C2C (notify bit1) + the shared config block
  ↓ DEC R2: config-apply 0x22DB8-0x22EC8 (DM 0x1C0126B0+idx·0x290 → 0x08C15960)
  ↓ m6-init 0x1e3e0: prints verdict + licensees (=0) → the type[81]→type[4] switch
  ↓ es:0 (no compressed output) → the DEC SDO path starves
  (P-A proved the DEC SDO emit machinery works when fed: 38-frame bursts on the wire)
```

## The remaining functional unknown (the SAME as the R46-era conclusion, now precise)
**What fills the SND's mstsound state with the unlicensed values** (verdict class=2,
licensees=0) — the mstsound framework state composition. The R46-era envelope-exhaustion
verdict stands for the SND static side; the runtime composition point remains unlocated.

## The two viable patch surfaces (final ranking)
1. **SND-side**: the config compose (0x1F575/0x1F44F) — force the licensed values into the
   DEC-bound message (verdict=0, licensees≠0). The exact composition sites still need the
   flow-following decode of 0x1F44F's body (linear drifts; Ghidra SWEEP blocked by the OSGi
   issue in this env — SND02_Sweep.java is ready and javac-valid).
2. **DEC-side**: the m6-init's licensee consumption (0x1e60c-0x1e652 reads; the consumer of
   the licensee fields for the es-decision) — force the es-path regardless of licensee.
   DEC-2 candidate: patch the DEC config-apply 0x22EC8 to store verdict=0 — REJECTED
   (the verdict word is diagnostic-only, proven).

## Device state: P-A active (DEC afde18ec), SND stock, AC3 PASS. No changes this phase.

---

# N-1-F PATCH CANDIDATE (2026-09-14, session 2 final) — 1-byte SND license-flag force

## The discovery chain (all byte-verified this session)
1. SND fn 0x1F575 (the config apply/compose — the R46/R50-era mstsound handler):
   `0x1F67B bg.jal 0xB6D8F` (the license/capability query)
   `0x1F67F bg.andi r11, r11, 0x200`  (bytes c5 6b 02 00; op 0x31; isolates BIT 9)
2. **Polarity (from the R50/R53 DEC-side analysis of the same function family):**
   the 0xB6D8F return = 0 → the DTS output-config block RUNS; non-zero → SKIPPED.
   Runtime (no DTS license): bit9 = 1 → r11 = 0x200 → the DTS config SKIPPED
   → the "unsupported" config composed (verdict=2, licensees=0) → DEC es:0.
3. **The patch: 0x1F681: 0x02 → 0x00** (andi mask 0x200 → 0x000) → r11 = 0 ALWAYS
   → the DTS config block ALWAYS RUNS → the licensed-path composition.

## Patch spec (NOT BUILT — pending the verification gate)
| field | value |
|---|---|
| image | snd_full.bin (stock eb879cdc07f510722f19db6d18d77d3c, 1,839,920 B) — NEVER patched before |
| file/fw offset | 0x1F681 (the SND address model: file = fw + 0x16F00 ⇒ fw 0x36 5881?? — file offset is authoritative) |
| old byte | 0x02 |
| new byte | 0x00 |
| changed bytes | 1 |
| instruction | `bg.andi r11,r11,0x200` → `bg.andi r11,r11,0x000` (r11 = 0) |
| op | unchanged (0x31); only the imm16 low byte |
| artifact | (to build) aucode_asnd_r2_MS12V22_N1F_1F681.bin |

## ⚠️ OPEN VERIFICATION ITEMS (per the standing rules — must close before deploy)
1. **Polarity proof**: the R50/R53 polarity came from the DEC-image analysis of the same
   function family. For the SND instance: confirm the downstream r11 consumption
   (0x1F684-0x1F7xx region) branches to the DTS-config compose on r11==0.
   The linear decode drifts — needs the flow-following decode (Ghidra OSGi blocked in this
   env; SND02_Sweep.java is javac-valid, the bundle env is the blocker) or more anchors.
2. **Requirement #4**: whether the r11==0 path actually writes non-zero licensees into the
   DEC-bound config, or only flips the verdict class. If the licensees come from the (absent)
   license material, the wire result may still be invalid bursts — the runtime test decides.
3. **AC3 safety**: the patched instruction is in the DTS-class branch (the R46/R50 disjointness:
   the DTS-family cases 9/10/11/23 → the 0xCC9B/0xC49D chain; AC3 cases 4/7 → the separate
   0x98A88 chain) — the andi site is inside the DTS-dispatch path, provably not the AC3 path.
4. Rollback: restore stock SND (eb879cdc) — the SAME procedure as the DEC rollbacks.

## If deployed, the success criterion (unchanged)
A NON-ZERO, continuously-populated DTS wire capture (not 38-frame windows) + the R2 log
showing es:2012-class output + the Pioneer VSX-817 showing DTS.

---

# N-1-G PATCH CANDIDATE (2026-09-14, session 2 final) — 1-BIT gate flip, byte-verified

## The gate instruction (byte-exact)
```
SND 0x1F6EC: bytes 21 64 E4 = bn.beqi r11, 1, → 0x1F725
  op 0x08 (bits 18-23), rD=r11 (13-17), imm=1 (10-12), rel=57 (2-9) → target 0x1F6EC+57=0x1F725, sub-op=00 (bits 0-1)
```
## The semantics chain (all verified fragments this session)
```
0x1F67B  jal 0xB6D8F                ← the framework license/capability query (r11 = return)
0x1F67F  andi r11, r11, 0x200       ← isolate BIT 9 (the DTS-enable framework flag)
0x1F684-0x1F6D6  the tag-2826 compare + the 14416/14420 config fields + the sflesi-11 test
0x1F6E8  op2E r11                   ← the unmodelled transform (R50-era: "opcode_2E r11==1")
0x1F6EC  beqi r11, 1 → 0x1F725      ← THE GATE: ==1 → the DTS output-config block
0x1F725+ = the DTS config block (R50-era verified: the DTS slot populate 0x1F910,
           the 0xCC9B/0xC49D calls at 0x1F754/0x1F75C, Predicate A, the latch re-arm)
```
Runtime: the DTS config block does NOT run ⇒ op2E(r11) != 1 ⇒ (with the andi feeding r11)
the framework flag denies the DTS output ⇒ the "unsupported" config → es:0.

## THE PATCH (N-1-G): 0x1F6EC byte2: 0xE4 → 0xE6 (ONE BIT: sub-op beqi→bnei)
Effect: the gate inverts — the DTS config block runs when op2E(r11) != 1.
At the current runtime state (op2E != 1, proven by the block not running) → the block RUNS.
On a hypothetical licensed stream (op2E == 1) → the block would skip — acceptable for this
experiment (no licensed streams exist on this device; AC3 path is provably disjoint).

## NOT BUILT. The verification gates before deploy:
1. The polarity risk: if the runtime op2E(r11) result is actually ==1 (and the block's
   non-execution has another cause), the bnei flip = NO CHANGE (harmless, falsifies fast).
2. The r11 consumption inside the 0x1F725 block: the R50-era verified the slot populate
   (0x1F910) requires 0x30==0 (the cleared slot) ✓ (the runtime state) — no conflict.
3. AC3 safety: the whole 0x1F575/0x1F6DA/0x1F6EC/0x1F725 chain = the DTS-dispatch branch
   (the R46/R50 disjointness: AC3 = the separate 0x98A88 chain) ✓ provably untouched.
4. Rollback: restore stock SND (eb879cdc) — the standard procedure.

## Alternative candidate (same gate, coarser): N-1-F — 0x1F681 0x02→0x00 (the andi mask→0,
   r11=0 before op2E) — held in reserve if N-1-G shows the flag (not the gate) is the blocker.

---

# PHASE N-1-G: DEPLOYED, TESTED — NO EFFECT ON THE DEC CHAIN (2026-09-14 02:27–02:35)

## Deploy record
- SND N-1-G: aucode_asnd_r2_MS12V22_N1G_1F6EC.bin (md5 ea7cd454e01d74e0a253963c46400023)
  active after reboot (verified via / file md5). Byte 0x1F6EE: 0xE4→0xE6 (beqi→bnei).
- DEC P-A: afde18ec92ffe0a960390a423d7f63c2 active (unchanged from the previous deploy).
- Uptime reset confirmed; both images verified post-boot.

## Test 1 — AC3 wire regression control: PASS
- `./cap.sh AC3_N1G test_ac3_51.ac3 20` → Codec:5 ×2, eRet:0x0.
- DUMP_audio_spdifNpcm_00.bin = 8,355,840 B → analyze_wire.py:
  **1360 preambles, 100% data-type 0x01 (AC-3), stride 0x1800** — AC3 passthrough intact.
  (Largest clean AC3 capture of the whole project.)

## Test 2–4 — DTS: playback ran, chain UNCHANGED
- test_dts_51.dts → Codec:9 ×2, eRet:0x0 (player accepted the stream).
- DEC R2 log (AudioDECR2_9E0x16_8A0x1_880x0_00.log, 427 KB):
  - `type[0]->type[81]` then **`type[81] -> type[4]`** (×2 start rounds) — unchanged.
  - **`dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0`** ×4 — unchanged.
  - Per-frame: **`pcm:5120 es:0` ×2914** — unchanged (decode-only, zero ES passthrough).
- Wire capture: dump opened _01.bin but wrote **0 bytes** (no DTS bursts this run;
  under P-A alone some runs gave intermittent 38-frame windows — single-run variance,
  cannot attribute to N-1-G; the DEC-side evidence below is deterministic anyway).

## Verdict: N-1-G FAILED — the gate at SND 0x1F6EC does not causally feed the DTS config
Transition-by-transition (BEFORE = P-A only vs AFTER = P-A + N-1-G):
| Transition | BEFORE | AFTER | Changed? |
|---|---|---|---|
| SND→DEC engine class | 81 → 4 (m6 decode-only) | 81 → 4 | **NO** |
| dts_licensee | 0 | 0 | **NO** |
| lbr/xll/transcoder licensee | 0/0/0 | 0/0/0 | **NO** |
| per-frame es: | 0 | 0 (×2914) | **NO** |
| DTS wire bursts | intermittent (38-frame windows) | 0 B this run | not improved |
| AC3 passthrough | OK | OK (1360/1360 0x01) | NO regression |

Conclusion: flipping the 0x1F6EC polarity did not change what SND writes into the
0xB000_0Cxx mailbox config for DTS. Either (a) that branch is not upstream of the DTS
config population, or (b) it is upstream but the licensee values it would gate are
sourced from the OTP/HASH verdict (the 0xB000_083C efuse-mirror chain, OTP bounding
0xff00 → "unsupported otp") — so the block, even when it runs, still writes licensee=0.
Under (b) the causal constraint is the **verdict value itself**, not this gate.

Per instruction: NO further patch stacked. Rollback available (stock SND eb879cdc).

## DEPLOYMENT STATUS DECISION (user-approved, 2026-09-14)
N-1-G REMAINS DEPLOYED (no rollback): it is provably harmless on AC3 (1360/1360 type-0x01
control capture) and produces no other observable change. Active image set:
- DEC: P-A afde18ec92ffe0a960390a423d7f63c2 (0x45FF5 bn.sw→bn.j)
- SND: N-1-G ea7cd454e01d74e0a253963c46400023 (0x1F6EE beqi→bnei)
Rollback images unchanged: stock SND eb879cdc, stock DEC 4b7e9509.
User approved next step: trace the licensee-bit SOURCE in SND (where the OTP verdict
becomes mailbox config bytes) to close the (a)/(b) dilemma from the N-1-G verdict.

---

# PHASE N-2: THE VERDICT CHAIN FULLY TRACED (2026-09-14, static + prior runtime)

## The engine-switch mechanism (all links closed)
1. **DEC monitor fn @0x198BD** (clean listing): compares DM struct 0x465840+0x4 (current
   type) against the BYTE at MMIO **0xB000_0018** (via rodata ptr table 0x141F40..54 =
   0xB000_0014/15/18/19/1A/1B). On mismatch: prints `decType change !! type[old] ->
   type[new] !!` (fmt 0x141A17, prefix 0x14207C, emitter 0x130CEA) and calls the applier
   **0x1760D(dec_id, struct, newtype)**, which writes struct+0x4 = newtype (at 0x17880)
   and reconfigures the engine. => **the 81→4 switch is EXTERNALLY DRIVEN by the byte
   at 0xB000_0018.**
2. **No DEC code writes 4 to 0xB000_0018** (listing-verified dataflow; only reset-0 at
   0x2378E, threshold 0x64 at 0x2071F, and monitor ack-clear 0x1C at 0x19C1C).
   The window 0xB000_0000 is SHARED between cores (both read 0xB000_0838). SND has
   word-store candidates to 0xB000_0018 (0xFECC/0x180DB/0x1E968/0x23278 — phase-
   unverified; no clean SND listing exists, Ghidra OSGi broken). ARM HAL
   audio.primary.mt5889.so contains 0xB0000018/15/1A/1B descriptor literals.
3. **The SND OTP gate (the decision input), fully decoded:**
   - sites: `0x16A44 bn.bf -> 0x16B5A` (site1) and `0x16BDF bn.bf -> 0x16CA3` (site2),
     plus one-shot init entry 0x1689F (cond ==0x158E) into the same print block.
   - `r23=0xB000_083C; r6=*(0x083C)` (the OTP bounding mirror), `r24=0xB000_0834`.
   - **check1: (B & 0x1001) == 0x1000 → bf → 'OTP fail : %x,%x,%x' (fmt 0x1287C3,
     emitter 0x10786E, args r4, 0x1000, B&0xFFFFFF) then `bg.j 0x16A68` — SKIPPING the
     mailbox poke.** Runtime match: B=0xff00 → 0xff00&0x1001=0x1000 → fail; print shows
     `OTP fail : 0x1, 0x1000, 0xff00` (×14628 per 15 s DTS).
   - **check2 (success path): (A & 0x282) == 0x282 → bf 0x16A68 (skip silently);
     else `0xB000_0C20 |= 0x4; 0xB000_0C20 |= 0x1` — the mailbox config-notify poke.**
   - => OTP fail ⇒ no config-notify ⇒ DEC never receives the DTS-capable config ⇒
     downgrade byte 4 at 0xB000_0018 ⇒ monitor applies ⇒ type[4] m6 decode-only ⇒ es:0.
   - This closes the N-1-G (a)/(b) dilemma: the gate N-1-G flipped was NOT upstream of
     THIS OTP gate; the OTP bounding value is the constraint.

## Patch N-2a (BUILT, NOT DEPLOYED)
- File: aucode_asnd_r2_MS12V22_N2A_16A44.bin
- md5 a1b2b5d57eea9840243eacd00819bcab, sha256 875d29f7...377626
- Change: 0x16A44 `bn.bf -> 0x16B5A` (20 04 59) → `bn.nop` (00 00 00), 3 bytes.
- Effect: check1 can never take the fail path; flow always reaches check2.
  If check2 passes at runtime → the 0xC20 poke fires → config-notify → no downgrade
  byte → DEC stays type[81] passthrough → P-A's SDO emit continues → DTS bursts.
- Flag-branch note: bn.bf cannot be polarity-flipped in one bit (unlike N-1-G's beqi);
  NOP is the minimal correct bypass. The OTP-fail print spam also disappears.
- Deliberately NOT touched: check2 (0x16A4F bf) — held as N-2b escalation if N-2a
  shows no runtime change (i.e., (0x0834 & 0x282)==0x282 also true); site2 (0x16BDF)
  — second instance, role not yet runtime-attributed; 0x1689F one-shot entry.
- Rollback: stock SND eb879cdc (and N-1-G ea7cd454 currently active — N-2a is built
  on STOCK, deploy procedure must re-apply N-1-G on top or build a combined image).

---

# PHASE N-3 → N-11: THE KEY/LICENSEE MARATHON (2026-09-14 04:20–05:30)

## Deployed image evolution (each = DEC on top of PA_N3V2 lineage, SND on N2COMBO)
| Ver | Change | md5 | Result (DTS playback) |
|---|---|---|---|
| N-3 | monitor remap r15:4→0x51 @0x19962 | e317d82e | 0→3→4; licensee=0; wire 0B |
| N-3v2 | remap→0x81 + window-pop restore | 631cded9 | 0→3→4; licensee=0; wire 0B |
| N-4 | 0x2972E ori r24,r0,0x488 (HASH bits) | c05a9367 | licensee=0 (bf@0x2975A skips parse) |
| N-5 | + NOP bf @0x2975A | 48148748 | licensee=0 — block not the source |
| N-6 | + twin block 0x29ADE/0x29AE6/0x29B0A same 2 fixes | 164b9fde | licensee=0 |
| N-7 | select 0x15938: ori r12,r0,0x488 + nop lwz | ba7f1b7a | BOOT select key1=0xefe7ff ✓ but type4 select still 0xede227 |
| N-8 | config-ctor 0x230DB: ori r25,r0,0x488 + nop | 5dfedd12 | key1 still 0xede227 |
| N-9 | blanket: all 11 remaining ori+lwz HASH pairs → bn.ori rN,r0,0x88 | afaa5566 | key1 still 0xede227 |
| N-10 | + 5 missed sites (0x22dc2 = the DM-config copy!, 0x2aee4, 0x2b93c, 0x2c6d6, 0x319ab) | d123f2f2 | key1 still 0xede227 |
| N-11 (SND) | all 7 SND HASH lwz sites → bn.ori 0x88 | b2a7e246 | **type3 select key1→0x000000 ✓✓ (SND no longer forwards HASH); type4 select STILL key1=0xede227** |

## Key discoveries this phase
1. **r2_decoder_select** (fmt 0x14126B, fn @0x15929, called from applier-chain 0x179E9):
   r10=key1(arg r4), r12=HASH-selfread(NOPed). type3 select now key1=0 — N-11 works.
2. **The type4 (m6/DTS) select key1=0xede227 survives ALL patches** — its source is NOT any
   ori/lwz 0x838 pair in either image (19 DEC + 7 SND sites all forced to 0x88/0x488).
   Remaining channels: DM mailbox config (SND→DEC IDMA), or the DTS m6 SDK reading the
   efuse HASH via an unmodelled mechanism (custom op / different base / relocated copy).
3. DM-read interface confirmed: `read_dsp_sram_type=1 addr=<16-bit cell> len=N` →
   `DM[0xNNNN]=...` in dmesg. 16-bit cell window only (0x119610→0x9610 truncation).
4. m6 init print (only print site: fn 0x295BC, print@0x297C9) reads struct+0x118..0x124 —
   the SAME fields written at 0x2976D..0x297A1 in the patched block — yet prints 0.
   => the playback-time m6 init instance runs via a DIFFERENT struct/context than the
   patched code path (likely the twin-execution or an arg-passed pre-filled struct).
5. Wire status through this phase: **DTS bursts appeared on the wire (114/152 valid,
   Pc=0x0B, Pd=0x3EE0, sync 0x7FFE8001 ✓) whenever Kodi's PLAYER pipeline streamed raw
   DTS** — that was Kodi's own resumed session (its player resuming the .dts at boot).
   With my Player.Open(.dts) via RPC after user's player-settings fix: Codec:9 correct,
   but wire 0B (the DEC SDO path still not fed by the m6 decode-only engine).

## Current active set
- DEC: N-10 (PA + N3v2 + N4 + N5 + N6 + N7 + N8 + N9 + N10) d123f2f207d0e65a969c14c69ec42639
- SND: N-11 (N2COMBO + 7 HASH→0x88) b2a7e2465ef01ff8a4b87b8d21794e1f
- Rollback: stock DEC 4b7e9509, stock SND eb879cdc

## NEXT (ranked)
1. Runtime DM dump during DTS playback of the m6 struct neighborhood to see actual
   licensee field values & key1 source (needs the 16-bit DM window map around 0x9600).
2. The type4-select key1 source: trace r10(arg4) at 0x179E9's caller chain upward —
   it originates from houseKeeping 0x198BD r15 (the type byte!) — verify with a marker
   patch (movi r4,0x77) to confirm arg flow.
3. If key1 flows from the m6 SDK itself (DTS lib), the last resort is patching the
   licensee CHECK inside the m6 module (strings 'licensee=%d' cluster 0x14508C..0x146C5D).
