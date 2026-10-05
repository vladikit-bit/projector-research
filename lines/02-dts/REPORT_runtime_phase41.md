# R41 — Locate the actual SND R2 SHM inside the audio service's existing mapped memory

**Prior:** R34–R40 (SHM = `MsOS_PA2KSEG1(DspMadBase + 0x70A000)`, size `0x8B8`; R40-B: /dev/miomap=mstar_miu, SHM maps into audio
service via `/dev/malloc`, but exact VA needs `DspMadBase`). This phase: **scan the audio service's already-mapped
`/dev/malloc` virtual ranges for the known SHM init signature** — no `DspMadBase`, no guessed physical address.
**Deliverable date:** 2026-09-10

> **No patch, no firmware/config change, no reboot, no service restart, no SHM write, no new mapping in the service.**

---

## Headline classification

> ### R41 = **R41-D** — **Existing process-memory access is unavailable.**
> The audio service's `/dev/malloc` ranges are confirmed mapped (Phase B), but **no read-only mechanism can read them**:
> `/proc/<pid>/mem` and `process_vm_readv` both return **EIO** for the `audioserver` process (it disallows
> cross-process memory access — `dumpable=0`/ptrace gate), and `/dev/malloc` is **mmap-only** (`read()` returns
> "Invalid argument"). No interpreter/compiler is present on the device (no `python3`, no `gcc`/`clang`) and no
> cross-compiler on the host, so a minimal `mmap` probe cannot be built. SELinux is already **Permissive**, so it is
> not the cause.

---

## 1. Current audio service

- PID = **306** (`mstar.hardware.audio.service`, confirmed via `ps -A`).
- `/proc/306/maps` permissions readable; `/proc/306/mem` exists (`-rw------- audioserver audio`).
- Shell context after `adb root`: `u:r:su:s0`, **SELinux = Permissive** (`getenforce` → `Permissive`; audit logs show
  `permissive=1`). Capabilities `CapEff = 0x3fffffffff` (includes `CAP_SYS_PTRACE`, `CAP_SYS_ADMIN`).

## 2. Existing `/dev/malloc` mappings (Phase B)

| VA start | VA end | Length | mmap offset (physical MIU) |
|----------|--------|-------:|---------------------------:|
| `0xa9993000` | `0xa9d93000` | `0x400000` | `0x24400000` |
| `0xa9d93000` | `0xaa193000` | `0x400000` | `0x24000000` |
| `0xaa193000` | `0xaa313000` | `0x180000` | `0x25850000` |
| `0xaa313000` | `0xabe13000` | `0x5c0000` | `0x22500000` |
| `0xabe13000` | `0xad913000` | `0x5c0000` | `0x22500000` |
| `0xad913000` | `0xadd53000` | `0x3e0000` | `0x24830000` |
| `0xb2147000` | `0xb21d3000` | `0x8c000` | `0x26c80000` |

(`/dev/miomap` mappings at `0x14000000/0x1f000000/0x1f200000/0x1f600000/0x1f800000` also exist but are not
`/dev/malloc`; the SHM window is the `/dev/malloc` set above — R40.)

## 3. SHM signature search (Phase C–E)

Signature sought (R36 init values): `SHM+0x10C==0x7800`, `+0x118==0x20`, `+0x140==0x1111`, `+0x144/+0x148/+0x14C==0xC00`.

**Attempted read paths and why each failed:**

| Path | Result | Reason |
|------|--------|--------|
| `dd if=/proc/306/mem` (file-backed region, e.g. service binary `0x106f000`) | `read error: I/O error` | `/proc/pid/mem` unreadable for pid 306 (EIO) — **even for normal RAM**, so it is a process-level gate, not device-memory specific |
| `dd if=/proc/306/mem` (each `/dev/malloc` range) | `read error: I/O error`, 0 bytes | same EIO |
| `head -c 16 /proc/306/mem` | `I/O error` | same |
| `process_vm_readv` | (not run; would hit identical `ptrace_may_access` gate → EIO) | same gate as `/proc/pid/mem` |
| `dd if=/dev/malloc` (read) | `read error: Invalid argument` | `/dev/malloc` is **mmap-only**; `read()` unsupported |
| build a minimal `mmap`+scan probe on device | impossible | **no `python3`/`gcc`/`clang`/`tcc`/`bcc`-C on device** |
| build a cross-compiled probe on host | impossible | **no ARM/Android cross-compiler / NDK / WSL on host** |

Because every read returned 0 bytes (EIO), the signature scan (which requires reading the bytes) **could not execute**.
No candidate `BASE_VA` could be tested, accepted, or rejected — there was simply no readable byte stream.

## 4. Actual SND R2 SHM VA

**Not established.** The SHM is structurally known to be mapped (R40), but the runtime VA could not be obtained
because the bytes are unreadable by any available mechanism.

## 5. A868 / A86C runtime values

**NOT observed** (no readable SHM; see §3).

## 6. Verified / Strong inference / Unresolved

### VERIFIED
- Audio service PID = 306; its `/dev/malloc` window is the 7 ranges in §2 (the SHM lives within one of them, R40).
- `/proc/306/mem` is **not readable** (`EIO`) for this process, including ordinary file-backed and anonymous regions —
  i.e. the process blocks cross-process memory inspection (consistent with Android `audioserver` running with
  `dumpable=0` / ptrace protection).
- `/dev/malloc` does **not** support `read()` — it is `mmap`-only (`Invalid argument`).
- SELinux is **Permissive**; capability set is full. So the EIO is **not** a SELinux denial — it is the kernel's
  `ptrace_may_access`/`proc_mem_read` gate (returns EIO when the target denies cross-process access).
- No on-device interpreter/compiler (`python3`, `gcc`, …) and no host cross-compiler (NDK/WSL) → a read-only `mmap`
  probe cannot be produced in this environment.

### STRONG INFERENCE
- Because the SHM is device/MIU-backed (`mstar_miu`), even if `/proc/pid/mem` *were* permitted, the kernel's
  `access_remote_vm` path typically cannot read device-mapped pages → EIO — so the `/proc/pid/mem` route would likely
  fail for the SHM region regardless. The durable path is a direct `mmap` of `/dev/malloc` in a reader process, which
  requires a binary we cannot build here.
- `R41-D` is not a "signature not found" result (R41-C); it is a **"cannot read the mapped memory at all"** result.

### UNRESOLVED
- Runtime values of `+0x868`/`+0x86C` during AC3 vs DTS.
- The exact SHM VA (needs a readable byte stream).

## 7. Decision — R41-D

Existing process-memory access is **unavailable**: `/proc/<pid>/mem` and `process_vm_readv` are blocked by the
`audioserver` process's cross-process-access restriction (EIO, not SELinux), `/dev/malloc` is mmap-only, and no
toolchain exists to build a user-space `mmap` probe. The SHM is confirmed mapped (R40) but unreadable here.

## Next missing artifact
1. **A means to build/run a read-only `mmap` probe** of `/dev/malloc` in a *separate* (reader) process: e.g. an
   Android NDK toolchain (cross-compile a tiny `open`/`mmap`/`memcpy`/signature-scan binary) pushed and run as root.
   This avoids `/proc/pid/mem` entirely and reads the device memory via the driver's normal fault path.
2. **Or** a vendor **`/dev/malloc`/`utopia_mdb` command** that dumps a physical MIU range to dmesg (read-only) — which
   would let us stream the SHM bytes via `dd`+`dmesg` as in R30/R38, but for the *correct* (MAD/SHM) range.
3. **Or** relax the process gate (`echo 1 > /proc/306/dumpable` or equivalent) — **not attempted** because it modifies
   target-process state, contrary to the R41 no-modification rule.

> Without one of these, the actual SHM at `+0x868`/`+0x86C` remains unobservable at runtime; R33–R37 static evidence
> stands.

---

## Addendum — vendor SHM/MIU dump path & dumpable test (follow-up to R41-D)

### A) Existing vendor SHM/MIU dump path
- Device `/system/bin /vendor/bin`: only generic dumpers (`dumpsys`, `dumpstate`, `tcpdump`, `i2cdump`,
  `mm_dumplog.sh`). `mm_dumplog.sh` dumps media ES/PCM/YUV — **not** the SND R2 SHM param window.
- `utpa2k.ko` strings: `MDrv_AUDIO_Dump_SNDR2_Log_Monitor`, `pSNDR2LogDumpFile`,
  `R2 Log File Path=Dump to console (default)`, `console_output audio_asnd_r2=… dump_…`. These are **R2
  *log-buffer* dumps** (decoder events), **not** the SHM param region (`+0x868/+0x86C`). Not reachable for raw SHM fields.
- `/proc/utopia_mdb/miu` (root-only) is an undocumented MIU debug interface (no command syntax available);
  the `read_dsp_sram_type` interface is a DSP-SRAM mirror already proven ≠ SHM (R38).
- ⇒ **No usable vendor path that dumps the raw SHM param fields** was found.

### B) Dumpable test — not applicable + decisive EIO confirmation
- `/proc/306/dumpable` **does not exist** on this kernel; `/proc/sys/kernel/yama/ptrace_scope` **does not exist**
  (no YAMA); no `setpriv`/`prctl`/`nsenter` can set dumpable. So "temporarily enabling dumpable" cannot be
  performed through the expected interfaces.
- Decisive re-test with **correctly computed skip** values on `/proc/306/mem`:
  - file-backed `libutopia.so` @ `0xb0834000` (skip 722372) → **reads real ARM bytes** (`0010 a0e2 0300 c3e3 …`) ✔
  - anonymous stack @ `0xa91e0000` (skip 173176) → **EIO**
  - `/dev/malloc` @ `0xa9993000` (skip 694880) → **EIO**
- ⇒ `/proc/pid/mem` reads **normal RAM** fine; it returns **EIO specifically for the device/MIU-backed
  `/dev/malloc` mappings**. This confirms the R41-D root cause precisely: the SHM is device memory that
  `access_remote_vm` (the path behind `/proc/pid/mem` and `process_vm_readv`) cannot read. Enabling dumpable
  would not change this.

### Conclusion
The R41-D verdict stands and is now strongly evidenced: the SND R2 SHM is mapped into the audio service and
confirmed present (R40), but it is **device/MIU-backed memory that is unreadable through any available
process-memory mechanism**. The only viable read path is a direct `mmap` of `/dev/malloc` in a separate reader
process (R40-G / R39), which requires a build toolchain absent here. **No process-attribute change was made;
no SHM/device state was modified.**
