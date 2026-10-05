# R42 — MIU-native SHM Observation

**Prior:** R33–R41 (SND R2 SHM = `MsOS_PA2KSEG1(DspMadBase + 0x70A000)`, size `0x8B8`; `/dev/malloc`+`/dev/miomap` = MStar MIU driver; R40-B: SHM structurally mapped into audio service; R41-D: `/proc/pid/mem`/`process_vm_readv` return EIO for device/MIU-backed memory).
**This phase:** determine whether a **vendor-native, read-only mechanism to observe the MIU-backed SHM** exists, and trace to the exact missing primitive. No SHM/device/firmware/config/process-state change.
**Deliverable date:** 2026-09-10

---

## 1. Executive Summary

**Question 1 — Is there a vendor-native way to read the MIU-backed SND R2 SHM without `access_remote_vm()`?**
> **A vendor-native transport EXISTS** (the MStar MIU driver exposes the SHM only via **`mmap`** of `/dev/malloc` major 158 / `/dev/miomap` major 176), but **no usable read interface is reachable from the available shell**:
> - `/proc/utopia_mdb/miu` **read** only emits a MIU *size/protect* dmesg line (not SHM content); its **write** command syntax is undocumented (driver built-in, code not in workspace) → cannot be used without guessing.
> - `/dev/malloc` **read()** returns `EINVAL` (mmap-only).
> - `devmem` exists but `/dev/mem` is **absent** → unusable.
> - `access_remote_vm` (`/proc/pid/mem`, `process_vm_readv`) is the blocked path (R41).
> ⇒ A vendor-native read *is conceptually present* (mmap), but it is **only usable through a custom userspace program**, which cannot be built here (no toolchain). So **no vendor-native read is usable read-only from the current environment.**

**Question 2 — Can we obtain the actual runtime values of `+0x868`/`+0x86C` read-only?**
> **No, not with available mechanisms.** The only viable transport (mmap of `/dev/malloc`/`/dev/miomap`) requires a userspace binary we cannot compile. The **exact missing primitive is a cross-compiled ARM/Android reader** for `/dev/malloc` (no more, no less); the SHM MIU address itself can be derived from the audio service's *existing* `/dev/malloc` mapping offsets (R40), so `DspMadBase` need not be guessed.

---

## 2. `/proc/utopia_mdb/miu` Analysis (R42-A)

| Property | Finding | Confidence |
|----------|---------|-----------|
| `stat` | `regular empty file`, `Access 0600 root`, `Device 4h/4d` (procfs) | CONFIRMED |
| Reading it (`cat`, read-only) | stdout empty; **dmesg gained**: `No MIU protect hit` / `---------MTK MIU DRAM Size Info---------` / `MIU0 size[1536]MB` | CONFIRMED |
| Owning module | `mstar_miu` (built-in) — `mdb_mbx_node_*`, `mdb_rtc_node_*`, `mdb_sc_node_*` handlers in `kallsyms`; MIU node handler not separately named but present (read produced MIU info) | LIKELY (code built-in, not in workspace) |
| write syntax | Unknown — driver source/built-in; no proc-parser string recovered | UNKNOWN |
| Can it dump MIU **memory**? | Read → MIU sizing only; no SHM/range dump observed. A WRITE may accept commands, but syntax unknown → **not used** (R42 rule: no speculative writes) | UNKNOWN (likely info-only for read) |

**Conclusion:** `/proc/utopia_mdb/miu` is a genuine MStar MIU debug node, but its **read** surface is an MIU information dump, not a SHM memory dump. It is **not** a usable path to `+0x868`/`+0x86C`.

## 3. `/dev/malloc` Driver Analysis (R42-B)

| Property | Finding | Confidence |
|----------|---------|-----------|
| `/dev/malloc` | `crw-rw-rw- 158, 0` (media:system) — char major **158 = `malloc`** | CONFIRMED |
| `/dev/miomap` | `crw-rw-rw- 176, 0` — char major **176 = `miomap`** | CONFIRMED |
| Driver symbols | `MDrv_MIOMAP_Open/MMap/Ioctl/Release`, `mstar_miu_drv_probe`, `miomap_cpu_page_fault_handler`, `MDrv_MIU_Kernel_*`, `MDrv_MIU_Protect*` — all built-in (no `.ko` on device) | CONFIRMED (names) |
| `.ko` on device? | None found (`find` for `*miu*.ko`/`*malloc*.ko` → empty) | CONFIRMED absent |
| `mmap` | Used by audio service (R40). Offset == physical MIU address (inferred from `/proc/306/maps`, R40) | CONFIRMED (usage), INFERRED (offset==phys) |
| `read()` | Returns `EINVAL` ("Invalid argument") — mmap-only device | CONFIRMED |
| `ioctl` | `MDrv_MIOMAP_Ioctl` exists; command set **unknown** (code not in workspace) | CONFIRMED exists, UNKNOWN semantics |
| Address translation | MIU physical address likely carries a **bank** field (`g_DSPMadMIUBank` global, R39); exact encoding needs driver source | INFERRED |

**Conclusion:** the MIU driver exposes MIU memory **only via `mmap`**. There is no read/ioctl path to copy SHM content to a caller without mapping it into the caller's own address space.

## 4. SHM Virtual → MIU/Physical Mapping (R42-C)

```text
DspMadBase (runtime kernel global g_DSPMadBaseBufferAdr, BSS)
   + 0x70A000                         ← corrected host offset (R39; NOT 0x7000A000)
   = SHM MIU/physical base
   + 0x868 / +0x86C                   → target fields
```
- `DspMadBase` is **not** a fixed constant (R39/R40); it is a runtime BSS value, unobtainable read-only (no `/dev/mem`).
- The audio service maps the MAD window via `/dev/malloc` at MIU offsets `0x22500000 / 0x24000000 / 0x24400000 / 0x24830000 / 0x25850000 / 0x26c80000` (R40/R41). The SHM (`DspMadBase+0x70A000`) lies **inside one of these** — this is the R40 structural proof.
- **MIU-bank translation** (`g_DSPMadMIUBank`) not resolved (driver code unavailable). The exact MIU address for the probe therefore needs either `DspMadBase` or the specific mapped-window offset — both derivable from the service maps without guessing (R40).

## 5. `+0x868` Access Trace (R42-D)

Re-confirmed in `snd_full.bin` (R2 firmware), this investigation:

| Role | Function/region | Address (file) | Encoding |
|------|----------------|---------------|----------|
| PRODUCER (write) | DTS helper `0xCF86` family | `0xD16C` builds base `0xA868`; store at `0xD15E` | `bg.sw_0 -0x5798(r3),r4` |
| CLEAR (write 0) | descriptor-reset path | `0xC8BC` builds `0xA868`; store at `0xC8C2` | `sw_0` |
| READER | — | **none found** in host kernel / DSP / codec R2 / `mik.ko` (R35) | — |

Field is **written by R2/DSP-side code only**; no host getter, no copy/export to an API-visible structure in available artifacts.

## 6. `+0x86C` Access Trace (R42-D)

| Role | Function/region | Address (file) | Encoding |
|------|----------------|---------------|----------|
| PRODUCER (write, ×3) | DTS helper `0xCC9B` family | `0xCE32`→store `0xCE26`; `0xCE7E`→store `0xCE72`; `0xCF54`→store `0xCF48` | `addi` builds `0xA86C`; `sw_0` |
| CLEAR (write 0) | descriptor-reset path | `0xCAA4` builds `0xA86C`; store at `0xCAAC` | `sw_0` |
| READER | — | **none found** (R35) | — |

Same conclusion: **DSP-side-only writer; no host reader** in available artifacts.

## 7. Existing Vendor APIs / Debug Paths (R42-E)

| API | Covers | +0x868/0x86C? |
|-----|--------|---------------|
| `HAL_SND_R2_Get_SHM_PARAM` | param block `0x104–0x194` only (jump table `0x59–0xCF`) | **No** |
| `HAL_SND_R2_Get_SHM_INFO` / `_64bit` / `_SIGNED` | region/size descriptors | **No** (not arbitrary offset) |
| `MDrv_AUDIO_Dump_SNDR2_Log_Monitor` | R2 **log buffers** (decoder events) | **No** (not SHM param window) |
| `/proc/utopia_mdb/miu` (read) | MIU size/protect info → dmesg | **No** (not SHM content) |
| `mm_dumplog.sh` | media ES/PCM/YUV dump | **No** |
| `dumpsys <audio>` | policy/status | **No** SHM fields |

No existing vendor API exposes `+0x868`/`+0x86C`.

## 8. Candidate MIU-native Read Paths (R42-F)

| Path | Mechanism | Viable now? | Reason |
|------|-----------|-------------|--------|
| `mmap /dev/malloc`(158) or `/dev/miomap`(176) + `memcpy` | MStar MIU driver | **No** — needs a userspace binary; **no toolchain on device or host** | MIU driver exposes memory only via mmap |
| `/proc/utopia_mdb/miu` write (range dump?) | proc node | **No** — write syntax unknown; no speculative writes | undocumented |
| `devmem` | needs `/dev/mem` | **No** — `/dev/mem` absent | confirmed absent |
| `/proc/pid/mem`, `process_vm_readv` | `access_remote_vm` | **No** — EIO on device/MIU-backed memory (R41) | blocked |

**The only theoretically-correct MIU-native mechanism is a direct `mmap` of `/dev/malloc`** — which is exactly the R39/R40-G authorized probe, but it cannot be built here.

## 9. Safety Assessment

- Entirely read-only. No SHM/device write; no `echo` to unknown proc nodes; `/proc/utopia_mdb/miu` was only **read** (it emitted a dmesg MIU-info line — a benign, read-triggered diagnostic).
- No firmware/config/EDID/ARC/AUTH change; no service restart; no process-attribute change (R41 confirmed `/proc/pid/dumpable`/YAMA absent anyway).
- SELinux already Permissive (irrelevant to the device-memory EIO).

## 10. Exact Next Step

To observe `+0x868`/`+0x86C` read-only, the **single missing primitive** is:

> **A minimal statically-linked ARM/Android reader** that: `open("/dev/malloc", O_RDWR)` → `mmap(NULL, len, PROT_READ, MAP_SHARED, fd, <MIU-offset>)` → `memcpy` the SHM base + the validation fields (`0x10C/0x118/0x140/0x144/0x148/0x14C`) then `+0x868/+0x86C` → `munmap`. Run as root.

Required inputs:
1. **Cross-compiler** (Android NDK / `arm-linux-androideabi-gcc`) — **the actual blocker** (absent on device and host; no WSL).
2. **MIU offset** for the SHM — obtainable **without guessing DspMadBase** by reusing the audio service's existing `/dev/malloc` mapping offsets (R40): the probe mmaps the *same* MIU window the service already maps (e.g. one of `0x22500000/0x24000000/0x24400000/0x24830000/0x25850000/0x26c80000`) and scans for the known init signature, then reads `+0x868/+0x86C` at the validated base.

If only (1) is supplied, the SHM becomes observable. If (2) is also needed, R40's structural proof already bounds it to the MAD window, and an in-window signature scan removes the need to know `DspMadBase`.

---

## Answers to the two questions

**Q1:** A vendor-native transport **exists** (MIU driver `mmap`), but **no usable read entry point** is reachable from the available shell — so effectively **no** vendor-native read is available *in practice* without a custom program.

**Q2:** **No** — actual runtime values of `+0x868`/`+0x86C` cannot be obtained with read-only mechanisms present. The obstacle is precisely **a missing userspace build toolchain** for a `/dev/malloc` mmap reader; all other primitives (the MIU device, the SHM already being mapped, the init-signature validation method) are already in hand. The SHM is device/MIU-backed memory that Linux `access_remote_vm` cannot read and that currently has *no* read/ioctl shortcut — making the mmap probe the unique path.

> This closes the R29–R42 read-only investigation: `+0x868`/`+0x86C` are confirmed 32-bit DTS-output-path fields written only by the R2 DSP (no host reader in available artifacts), whose runtime values remain **unobservable** under the constraint "no toolchain / no `/dev/mem`".
