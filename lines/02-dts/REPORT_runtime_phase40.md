# R40 — Reverse `/dev/miomap` and Recover the Audio-Service Mapping to SND R2 Memory

**Prior:** R34–R39 (SHM = `MsOS_PA2KSEG1(DspMadBase + 0x70A000)`, size `0x8B8`; `/dev/miomap` world-RW, used by audio service;
R39-C: mapping derivable but `DspMadBase` not obtainable). This phase: **reverse the existing mapping contract**
from real driver/service code — **no guessing of `DspMadBase`, no new mmap, no SHM write**.
**Deliverable date:** 2026-09-10

---

## Headline classification

> ### R40 = **R40-B** — **`/dev/miomap` semantics decoded; the SHM mapping is identifiable in form, but the live
> runtime read path is impractical** (the exact VA requires `DspMadBase`, which is a runtime kernel BSS global
> `g_DSPMadBaseBufferAdr` not readable from userspace).

---

## 1. `/dev/miomap` driver

- Major 176 / minor 0 → `/sys/dev/char/176:0 → ../../devices/virtual/miomap/miomap` (a **virtual** device).
- **Driver = `mstar_miu`** (built into the kernel; symbols in `kallsyms`):
  - `MDrv_MIOMAP_MMap`, `MDrv_MIOMAP_Ioctl`, `MDrv_MIOMAP_Open`, `MDrv_MIOMAP_Release`
  - `mstar_miu_drv_probe`, `_chip_flush_miu_pipe`, `miomap_cpu_page_fault_handler`
  - `MDrv_MIU_Kernel_*`, `dump_miu_part`, `dump_miu_records` (MIU partitioning/protect helpers)
- A **second char node** `/dev/malloc` (same underlying `mstar_miu` driver, different minor) is what the audio
  service actually uses for the DSP/MAD window (see §3). Both expose physical MIU memory to userspace.

## 2. `mmap` semantics (reconstructed from device evidence)

- The audio service `/proc/306/maps` shows `mmap` lines where the **file offset == the physical MIU address**
  (e.g. `… rw-s 24400000 … /dev/malloc` → offset `0x24400000`). So `mmap(fd, len, PROT, MAP_SHARED, offset=phys)`
  maps **physical MIU memory** to a userspace VA.
- Permissions observed: **`rw-s`** (shared, read+write) — a plain `memcpy`-style read is non-modifying.
- The driver binary (`mstar_miu.ko` / built-in) is **not present** in the workspace or on the device, so the
  exact internal arithmetic (MIU-bank encoding, cacheability, alignment rounding) is **not fully reversed from
  source**; the offset==physical translation above is inferred from the live `/proc/306/maps` layout and the
  `MDrv_MIOMAP_*` symbol set. *(This is the one inference in R40; it is consistent with all observed mappings.)*

## 3. Existing audio-service mappings (pid 306)

| VA range | mmap offset (physical) | device | candidate region |
|----------|----------------------:|--------|------------------|
| `a9993000–a9d93000` | `0x24400000` | `/dev/malloc` | **MAD window** |
| `a9d93000–aa193000` | `0x24000000` | `/dev/malloc` | **MAD window** |
| `aa193000–aa313000` | `0x25850000` | `/dev/malloc` | MAD/region |
| `aa313000–abe13000` | `0x22500000` | `/dev/malloc` | MAD window (part 1) |
| `abe13000–ad913000` | `0x22500000` | `/dev/malloc` | MAD window (part 2) |
| `ad913000–add53000` | `0x24830000` | `/dev/malloc` | MAD window |
| `add53000–adf53000` | `0x1f800000` | `/dev/miomap` | MIU bank region |
| `adf53000–ae153000` | `0x1f600000` | `/dev/miomap` | MIU bank region |
| `ae153000–af153000` | `0x14000000` | `/dev/miomap` | MIU bank region |
| `af153000–af353000` | `0x1f200000` | `/dev/miomap` | MIU bank region |
| `af353000–af753000` | `0x1f000000` | `/dev/miomap` | MIU bank region |

The `0x22500000 / 0x24000000 / 0x24400000 / 0x24830000 / 0x25850000` `/dev/malloc` ranges are the **DSP MAD memory
window** (the physical MIU region from which `DspMadBase` is derived). `libutopia.so` carries the strings
`/dev/malloc`, `/dev/miomap`, `miomap`, `DspMad` (×4), `PA2KSEG1` (×9) — i.e. the audio HAL (userspace) is the
code that opens these devices and maps the MAD/SHM window.

## 4. DSP MAD / SND R2 mapping chain (reversed)

```text
HAL_AUDIO_GetDspMadBaseAddr()        [utpa2k.ko @0x44CAE4]
   └─ wrapper: reads global ptr; else calls MDrv_AUDIO_GetDspMadBaseAddr (stub) → g_DSPMadBaseBufferAdr (BSS)
        ↓  returns DspMadBase  (runtime MIU-physical, stored in g_DSPMadBaseBufferAdr)
+ 0x70A000                          [movw #0xa000 ; movt #0x70]   ← SND R2 SHM offset
        ↓  (kernel)  MsOS_PA2KSEG1  → g_virSndR2shm (KSEG1, uncached)
        ↓  (userspace HAL / libutopia.so)  open("/dev/malloc") + mmap(offset = DspMadBase + 0x70A000)
        ↓  userspace VA inside one of the 0x22–0x25 MAD-window mappings above
SND R2 SHM  [base .. base+0x8B8]   including +0x868 / +0x86C
```
Verified kernel-side: `ut_r2shmget.asm` (kernel `HAL_DEC_R2_Get_SHM_INFO`) shows `g_virDecR2shm =
MsOS_PA2KSEG1(HAL_AUDIO_GetDspMadBaseAddr() + 0xE01000)` — same pattern, DEC offset `0xE01000`. This confirms the
kernel maps both SHMs into the KSEG1-mapped MAD window that the audio-service already mirrors via `/dev/malloc`.
**Therefore the SND R2 SHM at `DspMadBase + 0x70A000` DOES lie inside one of the already-mapped MAD-window regions.**

## 5. Runtime access feasibility

- The SHM **is** mapped into the audio service (the MAD window it maps via `/dev/malloc` contains `DspMadBase`,
  and `+0x70A000` is well within a 4 MB window such as `0x24400000`).
- A read could in principle be done by `open("/dev/malloc")`, `mmap(offset=DspMadBase+0x70A000)`, read — **but the
  exact offset needs `DspMadBase`**.
- `DspMadBase` resolves to **`g_DSPMadBaseBufferAdr`** (kernel BSS, confirmed via `kallsyms` and the decoded
  `HAL_AUDIO_GetDspMadBaseAddr` wrapper). It is **not** a fixed constant in any binary; it is written at module
  init. Reading kernel memory would require `/dev/mem` or `/dev/kmem` — **both absent**. `device-tree`, `dmesg`,
  `getprop`, `dumpsys` do **not** expose it.
- ⇒ The exact SHM VA (and thus a correct read) cannot be established without **guessing** `DspMadBase`, which R40
  forbids. So the live read path is **impractical** even though the mapping is structurally confirmed.

## 6. Validation plan / result

- Required Phase-H validation (known SHM fields `0x118=0x20`, `0x10c=0x7800`, `0x140=0x1111`, `0x144/8/c=0xc00`)
  was **not performed** — it needs the exact VA, which needs `DspMadBase`.
- A minimal read-only probe (Phase G) is **not executed**, because constructing the mmap offset would constitute
  guessing `DspMadBase`. This is the deliberate R40 safety stop.

## 7. A868 / A86C

NOT observed (no validated VA). The static chain (R33–R37) remains the only evidence: `A868`/`A86C` are 32-bit
published DTS-output-path fields inside the SND R2 SHM.

## 8. Verified / Strong inference / Unresolved

### VERIFIED
- `/dev/miomap` = `mstar_miu` driver (`MDrv_MIOMAP_MMap` etc.); a second node `/dev/malloc` is the same driver with
  a different minor and is what the audio service uses for the MAD window.
- Audio service (pid 306) maps the DSP MAD/SHM window via `/dev/malloc` at offsets `0x22500000 / 0x24000000 /
  0x24400000 / 0x24830000 / 0x25850000` (and MIU-bank regions via `/dev/miomap`).
- `libutopia.so` (audio HAL) contains the `/dev/malloc`/`/dev/miomap`/`DspMad`/`PA2KSEG1` strings → it is the code
  that maps the window.
- Kernel maps SHM via `MsOS_PA2KSEG1(DspMadBase + 0x70A000)` (SND) / `(DspMadBase + 0xE01000)` (DEC) — same MAD
  window the service mirrors.
- `HAL_AUDIO_GetDspMadBaseAddr` is a wrapper returning **`g_DSPMadBaseBufferAdr`** (runtime BSS) — `DspMadBase` is
  not a fixed/derivable constant.

### STRONG INFERENCE
- Because the SHM (`DspMadBase + 0x70A000`) sits inside a 4 MB MAD-window region the audio service already maps
  (`0x24400000` et al.), **the SHM is already mapped into the audio-service address space**; `+0x868`/`+0x86C` are
  within that mapped window. A read is structurally possible.
- The exact userspace VA of `+0x868`/`+0x86C` requires `DspMadBase`, which is not obtainable read-only.

### UNRESOLVED
- Exact `DspMadBase` value (runtime kernel BSS; no `/dev/mem`/`/dev/kmem`; silent in logs/props/tree).
- Exact `MDrv_MIOMAP_MMap` internal arithmetic (driver binary unavailable; offset==physical inferred from maps).
- Actual runtime values of `+0x868`/`+0x86C` (blocked by the above).

## 9. Decision — R40-B

The `/dev/miomap` (`mstar_miu`) driver is identified, its mmap semantics are reconstructed (offset = physical MIU
address; the audio service maps the MAD window via the same driver's `/dev/malloc` node), and the SND R2 SHM at
`DspMadBase + 0x70A000` is **structurally established to lie inside one of those already-mapped regions**. However,
the precise runtime VA — and hence a correct, non-guessing read — cannot be obtained because `DspMadBase`
(`g_DSPMadBaseBufferAdr`) is a runtime kernel global not readable from userspace. So the live read path is
impractical.

## Next missing artifact
1. **`DspMadBase` value**, safely: a vendor debug export, an ioctl that returns it, or the `g_DSPMadBaseBufferAdr`
   initializer in `mstar_miu`/`utpa2k` init code.
2. **`mstar_miu.ko` / built-in source** for `MDrv_MIOMAP_MMap` to confirm the offset→VA arithmetic (MIU bank).
3. (With either) a **validated read-only probe** of the SHM at the correct VA, with Phase-H validation against
   known init fields before reading `+0x868`/`+0x86C`.

---

## Appendix — evidence used
- `kallsyms`: `MDrv_MIOMAP_MMap`, `mstar_miu_drv_probe`, `g_DSPMadBaseBufferAdr` (BSS), `g_DSPMadMIUBank` (BSS).
- `/sys/dev/char/176:0 → virtual/miomap/miomap`.
- `/proc/306/maps`: the mapping table in §3 (audio service = `mstar.hardware.audio.service`).
- `libutopia.so` strings: `/dev/malloc`, `/dev/miomap`, `miomap`, `DspMad`×4, `PA2KSEG1`×9.
- `ut_r2shmget.asm` (kernel `HAL_DEC_R2_Get_SHM_INFO`): `MsOS_PA2KSEG1(HAL_AUDIO_GetDspMadBaseAddr() + 0xE01000)`.
- Decoded `HAL_AUDIO_GetDspMadBaseAddr` @`0x44CAE4` (utpa2k.ko) → wrapper returning `g_DSPMadBaseBufferAdr`.
- **No** SHM read was performed; no `DspMadBase` was guessed.
