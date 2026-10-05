# REPORT_runtime_phase16.md
## Fresh DEC R2 runtime log: concrete condition between DTS decoder execution and absence of DM 0x0900–0x0C00 writes

> Note on filename: the user's instruction wrote `REPORT_runtime_phase6.md`; given the
> running phase numbering (R12→R15 already delivered as `REPORT_runtime_phaseNN.md`),
> this is a typo for **R16**. The deliverable is `REPORT_runtime_phase16.md`.

---

### 0. TL;DR — answer to the core R16 question

**Yes.** Between *normal DTS decoder execution* and the *absence of writes to DM 0x0900–0x0C00*,
a concrete, logged condition appears in the real DEC R2 runtime log:

The DTS decoder **falls back** from the passthrough path (`decType 0x81`, producing compressed
IEC61937 bursts `es:1536 cB:600`) to the **PCM-decode path** (`decType 0x4`, `es:0 cB:0`), and
that fallback is logged *exactly* at:

```
r2_decoder_houseKeeping: dec_id:0, decType change !! type[81] -> type[4] !!
r2_decoder_select: dec:0, decType:0x4, key1=0xede227, key2=0xfffefd
dts m6 hook ok
dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0
dec_id:0, dec play !!          (now type 0x4 -> PCM only)
```

After this point the DEC emits `es:0` (no compressed burst), so the compressed-output DM window
`0x0900–0x0C00` is never written in steady state — which is exactly the R15 observation
(`DM 0x0900–0x0C00 = 0`, `Pa=F872 = 0` for DTS).

AC3 is the mirror image and **never falls back**: `decType change type[4] -> type[81]` +
`CPU MS12V2 ddp init ok`, with `es:1536 cB:600` from frame 0 to end.

**Evidence levels (R16 instruction #9):**
- **Observed:** the `decType 81→4` fallback, `dts m6 init ok … licensee=0`, and `es:0` for DTS;
  `es:1536` throughout for AC3.
- **Strongly correlated:** `transcoder_licensee=0` (and `dts/lbr/xll=0`) coincides with the loss of
  passthrough. IEC61937 packing/transcode of a DTS bitstream requires a valid transcoder license;
  on this device it reads `0`.
- **Not causally demonstrated by this log:** there is **no** literal `Invalid Spdif license` /
  `output_spdifSz` / `spdif` / `nonpcm` / `Packer` / `SDO` / `Hdmi` string in *either* DEC log
  (0 hits; only `pcm output stop`). The SPDIF/license-deny phrasing lives on the **SND/output**
  stage, which could not be observed (SND DM unreachable via `read_dsp_sram` 16-bit address
  truncation — R15). So `licensee=0` is a *reported status* strongly correlated with the fallback,
  not a self-proving causal sentence.

---

### 1. Method

- **Device:** Thundeal TD98 Pro / C50A (MStar MT5889), Android 11, kernel 4.19.116.
  ADB over TCP `192.168.0.183:5555`, root (userdebug). ADB daemon state (connect+root) does
  **not** persist across separate shell calls, so every call re-connects via `tools/r16_adb.sh`.
- **FS prep (instruction #2):** `rootfs ro → mount -o remount,rw / → mkdir /tmp →
  mount -t tmpfs tmpfs /tmp → mount -o remount,ro /`. Keeps rootfs RO while giving `/tmp` a
  writable tmpfs (the DEC R2 logger's legacy default path is `/tmp`; we actually logged to
  `/data` via `PATH=1`).
- **DEC R2 log trigger (corrected in R16):** `echo 'dump_r2_log_start=0 9E=0x16 PATH=1 8A=0x1' >
  /proc/utopia_mdb/audio` → DEC log to `/data/AudioDECR2_9E0x16_8A0x1_880x0_01.log`.
  (`param 0` = DEC; `PATH=1` = `/data`. `param 1` = SND; `PATH=0` = `/tmp`.)
- **Playback injection:** `app_process` `Probe7` on raw files —
  AC3: `/data/local/tmp/ac3_51.raw ac3raw ac3 2`;
  DTS: `/data/local/tmp/dts_51.raw dtsraw dts 1`. 12 s each.
- **Fresh file guaranteed before playback (instruction #3):** capture script does
  `rm -f /data/AudioDECR2_*`, sets `dump_r2_log_stop=0`, then `dump_r2_log_start=0 …`,
  waits 2 s, verifies the file exists, **then** starts playback.
- **Pulled locally:**
  - `r16_out/ac3/AudioDECR2_9E0x16_8A0x1_880x0_01.log` (129830 B, 2174 lines)
  - `r16_out/dts/AudioDECR2_9E0x16_8A0x1_880x0_02.log` (181026 B, 2869 lines)
- **FS restore (instruction #10):** `umount -l /tmp` (lazy; the tmpfs was busy-held by the
  audio R2-logger's stale `/tmp` handle from the earlier wrong SND-log attempt). rootfs confirmed
  `ro`. A harmless empty `/tmp` directory remains on the ro rootfs (cannot `rmdir` without an rw
  remount — forbidden). This is the maximum-achievable restoration and matches the instructed
  procedure.

---

### 2. Deliverables (raw logs are primary)

| File | Size | Notes |
|---|---|---|
| `r16_out/ac3/AudioDECR2_9E0x16_8A0x1_880x0_01.log` | 129830 B | AC3 DEC R2 log (fresh, frame 0→end) |
| `r16_out/dts/AudioDECR2_9E0x16_8A0x1_880x0_02.log` | 181026 B | DTS DEC R2 log (ring-buffered at top) |
| `r16_out/ac3_msgs.txt` | 20 lines | cleaned event lines (no status/perf noise) |
| `r16_out/dts_msgs.txt` | 27 lines | cleaned event lines |
| `r16_out/differential.txt` | — | chronological AC3-vs-DTS differential (this phase) |
| `r16_out/pre_mount.txt` | — | mount state before any change (baseline) |
| `REPORT_runtime_phase16.md` | — | this report |

---

### 3. Chronological differential (event order — not just grep)

Full table in `r16_out/differential.txt`. Summary:

**AC3 — single session, passthrough works, no fallback**
1. `decType change type[4] -> type[81]`
2. `r2_decoder_select decType:0x81`
3. `CPU MS12V2 ddp hook ok` / `CPU MS12V2 ddp init ok`
4. `cpuDecType(0x81) state req1->run1` → `playCmd 0x0->0x84` → `dec play !!` → `req4->run4`
5. frames `pcm:30720 es:1536 cB:600` (compressed burst) **from frame 0 to end**
6. no `decType 0x4`, no `dts m6`, no `licensee=0`
7. capture end: `dec stop ; pcm output stop`

**DTS — two sessions: passthrough attempt (A), then PCM fallback (B)**
- **Session A** (`type 0x81`): frames `pcm:30720 es:1536 cB:600` (compressed burst present,
  visible from frame 363 onward) → `dec stop ; pcm output stop`
- **FALLBACK:** `decType change type[81] -> type[4]`
  → `r2_decoder_select decType:0x4`
  → `dts m6 hook ok`
  → `dts m6 init ok, dts_licensee=0, lbr_licensee=0, xll_licensee=0, transcoder_licensee=0`
  → `cpuDecType(0x4) …` → `playCmd 0x0->0x84` → `dec play !!` (Session B, `type 0x4`)
- **Session B** (`type 0x4`): frames `pcm:5120 es:0 cB:0` (**PCM only — no burst**) → capture end

---

### 4. When does `dts m6 init ok … licensee=0` fire? (instruction #7)

It fires at the **decoder (re)select / second playback-start boundary** — i.e. at the
**fallback to the PCM-decode path** (`type 0x4`), **not** at initial module load and **not**
during the first (passthrough) playback. In-log order:

```
decType change 81->4  →  r2_decoder_select decType:0x4  →  dts m6 hook ok
  →  dts m6 init ok licensee=0  →  dec play (type 0x4)  →  es:0
```

So `licensee=0` is the decoder's *status report for the fallback path*, logged just before the
output decision that yields `es:0` (no burst). It is not a standalone "license check at module
init". Do **not** over-conclude from the string alone: it reports the license state of the
`dts m6` (DTS Master Suite 6) decoder as it is opened for PCM decode; it does not by itself prove
the passthrough was blocked *because* of the license — that causal link is inferred (see §6).

---

### 5. Reconciliation with R15

- R15: AC3 rewrites `DM 0x0900–0x0C00` continuously (~220–320 words/interval), `Pa=F872` live;
  DTS = 0, `Pa=F872 = 0`.
- R16 supplies the **mechanism**: AC3 DEC produces `es:1536` (IEC61937 burst) continuously →
  packer writes `Pa=F872` → AVR locks DD. DTS DEC produces `es:1536` only transiently (Session A),
  then **falls back** to `es:0` (Session B, the steady state R15 sampled) → no burst → `Pa=F872 = 0`.
- **Important nuance:** DEC `es:1536` is the DEC-side *intent* to pack passthrough. The actual
  write to `DM 0x0A00` (`Pa=F872`) is performed by the **SND/output IEC61937 packer**, which R15
  observed as `0` even during the transient `es:1536` phase. Therefore the effective gate (why
  `DM 0x0900–0x0C00` stays `0`) is **upstream of the DEC burst writer**, on the SND/packer/license
  stage. The DEC `decType 81→4` fallback is a **secondary reaction**; the primary failure is the
  packer/license gate that this DEC log cannot see.

---

### 6. What the instruction #8 hypothesized sequence got right / wrong

Hypothesized: `license check → Invalid Spdif license → output_spdifSz → dec play → no burst`.

- **Present (observed):** a license check/report (`dts m6 init ok … licensee=0`), a `dec play`,
  and `no burst` (`es:0`).
- **Absent (NOT in the DEC R2 log):** any literal `Invalid Spdif license`, `output_spdifSz`,
  `spdif`, `nonpcm`, `Packer`, `SDO`, `Hdmi` string. Both logs searched case-insensitively →
  **0 hits** (only `pcm output stop`). 
- **Conclusion:** the SPDIF / license-deny phrasing lives on the **SND side** (or a different log
  level), not in the DEC R2 log we captured. The DEC-level reality is the `decType 81→4` fallback
  with `licensee=0` reported.

---

### 7. Limitations / honesty

- The DTS raw file is **ring-buffered at the top** (starts at frame 108, not 0); raw line order is
  not time order. The differential uses the decoder state-machine messages, which are reliable.
- Capture used direct `app_process Probe7` raw-file injection, **bypassing** the Android audio
  policy / HDMI-ARC / SPDIF routing of the real Kodi path. DEC-level behavior is shown; end-to-end
  AVR lock/silence is governed by additional layers (SND output, HDMI/SPDIF driver, audio policy)
  not reached here.
- SND-side DM is unreachable (`read_dsp_sram` 16-bit address truncation), so the actual `Pa=F872`
  write gate could not be directly observed from the DEC side.
- **Forbidden actions NOT taken** (R16 constraints): no DM/PM writes, no binary patches, no
  EDID/routing/settings changes, no module replacement, no reboot, no full 64K dump, no full PM ISA
  decode.

---

### 8. Next steps (not executed)

1. **Capture the SND R2 log (`param=1`)** to observe the IEC61937 packer / `output_spdifSz` /
   license-deny path — the actual `DM 0x0900–0x0C00` writer. Note: SND DM is unreachable via
   `read_dsp_sram`, but the SND R2 *log string* (`param=1`) may still be capturable to `/data` if
   the logger supports `PATH=1` for SND as well.
2. Confirm whether `transcoder_licensee` is the gate by comparing the `dts m6` license query
   against a known-good device (or the license blob on this unit).
3. Inspect the Android audio HAL / MStar audio service decType-selection policy — why DTS is routed
   to the `0x4` fallback while AC3 stays `0x81`.
