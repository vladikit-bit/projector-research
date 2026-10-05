# R39 — Direct Read-Only Access to the ACTUAL SND R2 Shared Memory

**Prior:** R34-C (window = SND R2 SHM, host mapping `MsOS_PA2KSEG1(DspMadBase + K)`) · R35-D (no reader) · R36-C (region) ·
R37-B (lifecycle) · R38-D (`DM[]` mirror ≠ SHM, so runtime values unobservable that way)
**This phase:** read-only runtime **feasibility** investigation. No patch, no firmware/config change, no kernel-module change,
no reboot, no SHM write.
**Deliverable date:** 2026-09-10

> **CRITICAL CORRECTION (affects R34–R38 wording):** the host-side SHM offset is **`0x70A000`**, not `0x7000A000`.
> The asm is `movw r2,#0xa000 ; movt r2,#0x70` ⇒ `0x70 << 16 | 0xA000 = 0x70A000`. The earlier reports wrote
> `0x7000A000` (an extra `00`); the low 16 bits (`0xA000`) are what the R2 firmware sees as its SHM base, so the
> firmware-side layout (`0xA000+0x868`) is unaffected. SHM physical = `DspMadBase + 0x70A000`.

---

## Headline classification

> ### R39 = **R39-C** — **Mapping is derivable; no operative read-only userspace mechanism exists.**
> The translation (`g_virSndR2shm = MsOS_PA2KSEG1(DspMadBase + 0x70A000)`, size `0x8B8`) is established from code, and
> an existing host mechanism — **`/dev/miomap`** (world-`rw`, already mapped by `mstar.hardware.audio.service`) — is the
> interface that reaches DSP physical memory. **But the runtime `DspMadBase` cannot be obtained by any safe read-only
> means**, so the SHM's physical address (and thus an actual read of `+0x868`/`+0x86C`) remains infeasible.

---

## 1. Access path (derived, not executed)

```text
HAL_AUDIO_GetDspMadBaseAddr()   -> DspMadBase  (runtime MIU-physical, kernel BSS  g_DSPMadBaseBufferAdr)
        + 0x70A000             -> SHM physical base
MsOS_PA2KSEG1(...)              -> g_virSndR2shm  (kernel KSEG1 virtual, uncached)
SHM spans [base, base + 0x8B8) ; +0x868 / +0x86C inside it
        |
        | existing host mechanism (audio service already does this):
        v
/dev/miomap  (char dev 176, mode 0666)  mmap( fd, ..., offset = DspMadBase + 0x70A000 )
        |
        v   (read-only: MAP_PRIVATE + read)  ->  +0x868 / +0x86C
```
The **only missing link is `DspMadBase`**.

## 2. API analysis (Phase A/B) — `utpa2k.ko` (unstripped)

| Function / interface | Actual behaviour (from code/symbols) | Arbitrary SHM read? | Read-only? | Result |
|---|---|---:|---:|---|
| `HAL_SND_R2_Get_SHM_PARAM` | dispatch via jump table `params 0x59…0xCF` → offsets `0x104…0x194` (`HAL_SND_R2_Set_SHM_PARAM` mirror) | **No** (fixed param block only) | Yes (returns value to caller) | **Cannot target +0x868/0x86C** |
| `HAL_SND_R2_Get_SHM_INFO` (`0x458E24`) | returns SHM info / region descriptors | No | Yes | Not an offset read |
| `HAL_SND_R2_Get_SHM_INFO_64bit` (`0x45C284`) | 64-bit SHM info | No | Yes | Not an offset read |
| `HAL_SND_R2_Get_SIGNED_SHM_INFO` (`0x4590A4`) | signed SHM info | No | Yes | Not an offset read |
| `HAL_SND_R2_Get_SHM_COMMOM_PARAM` (`0x459CD0`) | param-block read | No | Yes | Cannot target tail |
| `HAL_SND_R2_BackupShareMemory` / `RestoreShareMemory` | bulk copy SHM↔backup | No (whole-region) | **No** (writes backup) | Not a read probe |
| `HAL_SND_R2_init_SHM_param` | `memset(shm,0,0x8b8)` + inits | No | No (init) | — |
| `MDrv_AUDIO_Dump_*_Monitor` (`MDrv_AUDIO_Dump_SNDR2_Log_Monitor`, `…SpdifNpcm…`, `…HdmiNpcm…`) | dump/monitor funcs (print to dmesg) — likely invoked via ioctl | Possibly whole-region | Yes | No single-offset read API exposed to shell |

**Conclusion:** no existing host API returns an **arbitrary** SHM offset; the `Get_SHM_*` family only covers the
`0x104…0x194` parameter block. So a direct SHM `+0x868` read is **not** available through a HAL/ioctl API.

## 3. Address derivation (Phase C/F)

```
HAL_AUDIO_GetDspMadBaseAddr()        [utpa2k.ko @0x44cae4]
   -> returns DspMadBase  (physical MIU address; stored in BSS g_DSPMadBaseBufferAdr [kallsyms], with MIU bank in g_DSPMadMIUBank)
+ 0x70A000                          [movw #0xa000 ; movt #0x70]   ← corrected typo, was 0x7000A000
MsOS_PA2KSEG1(...)                  -> uncached KSEG1 kernel virtual
= g_virSndR2shm
```
`DspMadBase` is a **runtime** value. It is **not** a fixed constant in the binary; it is written into `g_DSPMadBaseBufferAdr`
at module init. No safe read-only source exposes it (see §7).

## 4. Validation (Phase H) — NOT PERFORMED

Because the SHM cannot be read, the required Phase-A validation (known SHM field `0x118=0x20`, `0x10c=0x7800`,
`0x140=0x1111`, `0x144/8/c=0xc00` appearing at the observed cells) was **not performed**. No `A868`/`A86C` observation
was attempted (and the `DM[]` mirror from R38 is explicitly excluded per R38).

## 5. A868/A86C observations — N/A

Not observed (no validated access path).

## 6. Decision — R39-C

The physical/virtual mapping is **derivable** (code-proven formula above) and an existing host mechanism
(`/dev/miomap`, used by the audio service) is the right interface, **but no operative read-only userspace path
reaches the SHM** because the runtime `DspMadBase` (and its MIU-bank encoding) cannot be obtained without guessing,
which R39 forbids.

## 7. Next missing artifact (specific)

One of:
1. **`DspMadBase` value** from a safe source: a vendor debug/export that prints it, an ioctl that returns it, or the
   AEON/loader config that initialises `g_DSPMadBaseBufferAdr`. (Not `/dev/mem`/`/dev/kmem` — both **absent** here.)
2. **`/dev/miomap` offset / MIU-bank translation** for the SND R2 SHM — i.e. the documented rule that maps
   `DspMadBase + 0x70A000` to the correct `mmap` offset (MStar MIU addresses carry a bank field; `g_DSPMadMIUBank`
   is present, confirming the encoding).
3. **Vendor SDK source** for `HAL_AUDIO_GetDspMadBaseAddr` and `MsOS_PA2KSEG1` — would let the exact physical address
   be derived without guessing.

> Without one of these, the actual SHM at `+0x868`/`+0x86C` is **not observable** at runtime; the static chain
> R33–R37 remains the only evidence.

---

## Appendix — what WAS established on-device (read-only, no state change)

- Device reachable via `adb connect 192.168.0.183:5555` + `adb root` (userdebug, permitted). No reboot/config change.
- Audio service: `mstar.hardware.audio.service` (pid 306). Its `/proc/306/maps` shows **`/dev/miomap`** mappings at
  offsets `0x1f800000`, `0x22500000`, `0x24400000`, `0x24830000` (physical MIU regions) — proof the host already
  maps DSP physical memory to manipulate the SHM.
- `/dev/miomap` exists as **`crw-rw-rw- 176,0`** (world R/W) — a legitimate, existing, read-capable interface.
- `/proc/utopia_mdb/audio` exists but its `read_dsp_sram_type` mirror is the **wrong** memory (R38 proved it is not
  the SHM).
- `kallsyms` confirms the symbols `g_DSPMadBaseBufferAdr` (BSS) and `g_DSPMadMIUBank` (BSS) — the runtime base
  holders — but their values are **not** readable from userspace (no `/dev/mem`, `/dev/kmem`).
- `device-tree` has **no** audio/MAD/DSP node exposing a base; `dmesg`/`getprop`/`dumpsys` do not print `DspMadBase`.
