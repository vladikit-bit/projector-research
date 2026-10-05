# R38 — Runtime Validation of SND R2 SHM `+0x868` / `+0x86C`

**Prior:** R33-B (write-only) · R34-C (SND R2 SHM) · R35-D (no reader) · R36-C (region identified) · R37-B (output lifecycle)
**This phase:** a **read-only runtime observational pass** — no patching, no device-config change, no firmware change, no reboot.
**Method (pre-approved R30 mechanism):** `/proc/utopia_mdb/audio` via `read_dsp_sram_type=N addr=<CELL> len=<CELL>`, then `dmesg | grep 'DM\['`.
**Deliverable date:** 2026-09-10

> **Device access note:** the device was offline; `adb connect 192.168.0.183:5555` re-established the bridge, and
> the proc entry `/proc/utopia_mdb/audio` is `-rw------- root root`, so captures required `adb root` (which this
> userdebug device permits). No reboot, no config change, no patch.

---

## Headline classification

> ### R38 = **R38-D** — **Observation mapping cannot be validated.**
> The `read_dsp_sram_type` mirror (values 0–5 all tried) reads a live DSP/MMIO-mirror SRAM, **not** the SND R2
> shared memory where `0xCF86`/`0xCC9B` publish `A868`/`A86C`. Therefore the captured `A868`/`A86C` values **cannot be
> attributed to the SHM fields**, and no conclusion about their DTS-vs-AC3 behaviour is possible.

---

## Observation-method validation

### Step 1 — expected ground truth
`HAL_SND_R2_init_SHM_param` (`utpa2k.ko` @`0x4582A8`, `movw r2,#0x8b8` @`0x4582F4`) zeroes `+0x000…0x8B8` and sets
known values including `+0x118 = 0x20`, `+0x10c = 0x7800`, `+0x140 = 0x1111`, `+0x144/8/c = 0xc00`
(verified static in R36/R37). If `DM[0xAxxx]` mapped to the SHM, these would be readable at
`DM[0xa118]`, `DM[0xa10c]`, `DM[0xa140]`, `DM[0xa144]`.

### Step 2 — what `read_dsp_sram_type=1` actually returns over `0xA000…0xA8FF`
| Region | Result |
|--------|--------|
| `0xA100…0xA15F` | **all cells `0x000000`** (expected `0xa118=0x20`, `0xa10c=0x7800`, `0xa140=0x1111`, `0xa144=0xc00`) |
| `0xA000…0xA8FF` (2305 cells, 206 non-zero) | **0 cells** carry any of the init signatures `0x20 / 0x7800 / 0x1111 / 0xc00` |
| `0xA850…0xA8AF` (tail cluster) | non-zero **stable** values: `0xa854=0x0018a0`, `0xa870=0x0089e0`, `0xa87c=0x0103b4`, `0xa884=0x01176a`, `0xa890=0x000008`; `0xa868=0x000000`, `0xa86c=0x000000` |

### Step 3 — try other `read_dsp_sram_type` values
Type ∈ {0, 1, 2, 3, 4, 5} × read `0xA100…0xA160`: **none** yields any init-signature value.

### Conclusion of validation
`read_dsp_sram_type` (every available type) reads **a live DSP SRAM / MMIO-mirror region** (e.g. the `0xB000_xxxx`
mirror that R30 correlated with MMIO registers) — **not** the SND R2 SHM. The SHM (host-mapped at
`DspMadBase + 0x7000A000`, R2-side base `0xA000`) is **not covered** by this mirror. ⇒ **Phase A validation
FAILED**; the DM index is *not* a validated SHM offset.

> **Consequence:** the values measured below are observations of the *wrong* memory. They carry **no evidence** about
> the producers' writes. In particular, "A868/A86C read 0 during DTS" is **NOT** a valid conclusion.

---

## Captured table (observations of the *mirror* memory — NOT the SHM)

| Phase | A868 (mirror) | A86C (mirror) | Neighboring mirror fields | Notes |
|-------|--------------:|--------------:|---------------------------|-------|
| Idle_1 | 0x000000 | 0x000000 | 0xa854=0x18a0, 0xa870=0x89e0, 0xa87c=0x103b4, 0xa884=0x1176a, 0xa890=0x8 | values **static** |
| Idle_2 | 0x000000 | 0x000000 | same | same |
| AC3_T1 | 0x000000 | 0x000000 | same | same |
| AC3_T2 | 0x000000 | 0x000000 | same | same |
| AC3_T3 | 0x000000 | 0x000000 | same | same |
| AC3_STOP | 0x000000 | 0x000000 | same | same |
| DTS_T1 | 0x000000 | 0x000000 | same | same |
| DTS_T2 | 0x000000 | 0x000000 | same | same |
| DTS_T3 | 0x000000 | 0x000000 | same | same |
| DTS_STOP | 0x000000 | 0x000000 | same | same |

*(Every `A868`/`A86C` line reads `0x000000`; the neighboring cluster is non-zero and **identical across all ten
captures, idle→AC3→DTS→stop*. Because the mirror ≠ SHM, none of this is attributed to `0xCF86`/`0xCC9B`.)*

---

## VERIFIED

- The runtime observation mechanism **works** and reads a **live, populated** memory region (206 non-zero cells in
  `0xA000…0xA8FF`; the tail cluster `0xA850…0xA8AF` shows specific, stable values). It is a valid DSP-memory probe.
- **That region is NOT the SND R2 SHM.** Definitive proof: the SHM init signatures (`0x118=0x20`, `0x10c=0x7800`,
  `0x140=0x1111`, `0x144=0xc00`) are **absent** from the entire `0xA000…0xA8FF` mirror, for every
  `read_dsp_sram_type ∈ {0,1,2,3,4,5}`.
- A full idle→AC3→DTS→stop capture was executed (launch Kodi `test_ac3_51.mp4` / `test_dts_51.mp4`, sample ×3 each,
  then `am force-stop`); the device audio pipeline ran (Kodi launched, playback started).
- `0xb000_xxxx` MMIO-mirror interpretation (R30) remains consistent: `type=1` mirrors the DSP's MMIO/register SRAM,
  separate from the SHM data window.

## STRONG INFERENCE

- The producers' writes to `A868`/`A86C` (R33/R37) live in memory that the available `read_dsp_sram_type` mirror
  does **not** expose. The mirror likely targets the **DEC/codec R2 SRAM or an MMIO-mirror**, whereas the SND R2
  SHM is a distinct shared-memory window mapped at `DspMadBase + 0x7000A000`.
- The prior R34-era captures (`dm3_dts.txt`/`dm3_ac3.txt` showing `DM[0xa868]=0`) were **also** obtained via this
  same `type=1` mirror, so they too do **not** observe the SHM — consistent with the R38 failure, not contradicting it.

## UNRESOLVED

- **The runtime values of `A868`/`A86C` during AC3 vs DTS playback.**
- **Whether `A868`/`A86C` ever become non-zero** (the producers' computed expression, R32/R33, may evaluate to a
  value, or the write path may be bypassed — both are consistent with the static evidence, but neither is runtime-proven).
- **The correct read mechanism for the SND R2 SHM** — none of the available `read_dsp_sram_type` values targets it.

---

## Decision — R38-D

The required Phase A validation (`DM[index] == SHM_offset`) **cannot be satisfied** with the available mechanism:
the mirror reads a different memory than the SHM. Per the R38 rules, the result is **R38-D — observation mapping
cannot be validated**, and no inference about `A868`/`A86C` is drawn from the (wrong-target) readings.

## What artifact is still missing (specific)

The **correct runtime handle to the SND R2 SHM**, one of:
1. **A valid `read_dsp_sram_type` value (or equivalent proc/sys entry) that maps to the SND R2 SHM** — none of
   {0,1,2,3,4,5} does; vendor documentation for the `utopia_mdb` audio proc handler is required to identify it.
2. **A host-side read of `g_virSndR2shm`** (the kernel-virtual mapping `MsOS_PA2KSEG1(DspMadBase+0x7000A000)`),
   e.g. via `/dev/mem`/`devmem` on the physical address `DspMadBase+0x7000A868/0x7000A86C`, or a debugfs/sysfs
   export — none present in this workspace.
3. **Runtime instrumentation** that observes the SHM at the correct address (would also resolve R35's reader question).

> Without one of these, `A868`/`A86C` runtime behaviour **cannot be observed**, and the investigation of their
> DTS-vs-AC3 difference remains a static-only result (R33–R37).

---

## Appendix — artifacts
- `aeon_validate/r38_out/all.txt` — full 10-phase capture (idle/AC3/DTS ×3 + stops).
- `tools/r38_capture.sh` — the read-only capture script (push to `/data/local/tmp`, run as root).
- Validation reads: `type=1` `0xA100…0xA15F`, `0xA850…0xA8AF`, `0xA000…0xA8FF`; types 0–5 over `0xA100…0xA160`.
- Device: `adb connect 192.168.0.183:5555` + `adb root` (userdebug, permitted); no reboot, no config/firmware change.
