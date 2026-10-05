# R43-A Build Report — `miu_mmap_smoke`

**Context:** R42 established that the only vendor-native path to the MIU-backed SND R2 SHM is a direct
`mmap()` of `/dev/malloc` (major 158) / `/dev/miomap` (major 176), which requires a userspace ARM binary.
R43-A builds the **transport smoke probe** only (no SHM signature scan, no `+0x868`/`+0x86C` read yet).
**Strictly read-only program:** `open(O_RDONLY)` → `mmap(PROT_READ, MAP_SHARED)` → read a few bytes → `munmap()`
→ `close()`. No writes, no ioctls, no `/proc/pid/mem`, no ptrace, no `/dev/mem`.

---

## 1. Exact source code summary

File: `miu_mmap_smoke.c` (2774 bytes). Behaviour:
- `open("/dev/malloc", O_RDONLY)`.
- `mmap(NULL, length, PROT_READ, MAP_SHARED, fd, offset)` where `offset` is a page-aligned caller argument.
- Reads `min(length, 64)` bytes, prints them as hex.
- `munmap()`, `close()`, `return 0` on success; returns `1` on open/mmap failure (prints `errno`+text),
  `2` on argument/alignment/range errors.
- CLI: `miu_mmap_smoke <offset_hex> [length_dec]`. Hex offsets (`0x22500000` …) accepted; default length 16,
  capped `1..65536`; offset must be `4096`-aligned.

(Full source kept alongside this report.)

## 2. Exact NDK / compiler invocation

```text
C:\Android\android-ndk-r30\toolchains\llvm\prebuilt\windows-x86_64\bin\clang.exe
   --target=armv7a-linux-androideabi30
   -fPIE -pie -O2 -Wall
   -o miu_mmap_smoke miu_mmap_smoke.c
```

- Clang identifies as: `Android (16134705, based on r574158c) clang version 21.0.0`, target
  `x86_64-w64-windows-gnu` (host), building for the `--target` triple below.
- `-fPIE -pie` required for Android executables. No C++ dependency; plain C + libc.

## 3. Target triple / API

```text
triple: armv7a-linux-androideabi
API:    30
```
Resulting ELF OS/ABI = UNIX System V; the `armv7a-linux-androideabi30` triple selects the Android ARMv7 sysroot
from the NDK. Machine = ARM; PIE (`FLAGS_1: NOW PIE`).

## 4. ELF validation (NDK `llvm-readelf`)

```text
Class:        ELF32
Data:         2's complement, little endian
OS/ABI:       UNIX - System V
Type:         DYN (Shared object file)   <- PIE executable
Machine:      ARM
Flags:        0x5000200                  <- ARM EABI v5 / hard-float marker
Entry:        0x1651
Program HDrs: 12   Section HDrs: 28
```

Confirmed **ELF32 / ARM / little-endian / PIE**. (`file` utility not present on host; validation relies on
`llvm-readelf -h`, which is authoritative.)

## 5. Dynamic / static linking status

**Dynamically linked** (NOT static). `llvm-readelf -d` shows:
```text
NEEDED  Shared library: [libdl.so]
NEEDED  Shared library: [libc.so]
FLAGS_1 NOW PIE
```
The binary depends on the device's `libc.so`/`libdl.so` (present on the Android target). **It is therefore
dynamically linked — do NOT describe it as statically linked.**

## 6. File size

```text
7028 bytes  (miu_mmap_smoke)
```

## 7. SHA-256

```text
99c3cd02ee14f8314f90a53f61ef61ebb87b6947d2dc16871b834f1f92cda94d
```

## 8. Exact commands for manual ADB deployment

**(Not executed here — for manual run per R43-A instructions.)**

```text
adb connect 192.168.0.183
adb push miu_mmap_smoke /data/local/tmp/
adb shell chmod 755 /data/local/tmp/miu_mmap_smoke
```

Recommended manual test invocations (as root via `adb root`):
```text
adb shell /data/local/tmp/miu_mmap_smoke 0x24400000
adb shell /data/local/tmp/miu_mmap_smoke 0x24000000
adb shell /data/local/tmp/miu_mmap_smoke 0x22500000 16
```

## 9. Expected runtime output

**Success path (mmap works from a separate process):**

```text
device=/dev/malloc
offset=0x24400000
length=16
mapped_va=0x<some addr>
first 16 bytes: <16 hex bytes; for a MIU/MAD window these will be non-zero device-memory content>
ok
```

**Possible failure paths (still a valid R43-A result — they prove the transport hypothesis wrong/needs-adjustment):**

| Symptom | errno | Meaning / next step |
|---------|-------|--------------------|
| `open(/dev/malloc, O_RDONLY) failed: Permission denied (13)` | EACCES | run as root (`adb root`) |
| `mmap() failed: Permission denied (13)` | EACCES | driver may require `O_RDWR`; document & rebuild with `O_RDWR` |
| `mmap() failed: Invalid argument (22)` | EINVAL | offset/length rejected by driver; check alignment/range |
| `mmap() failed: I/O error (5)` | EIO | region not mappable this way |

No value interpretation is performed — R43-A only confirms the **mechanism** (`open`→`mmap`→read→`munmap`).

## 10. Assumptions made

1. `/dev/malloc` (major 158, `crw-rw-rw-`, media:system) accepts `open(O_RDONLY)`; if the driver's
   `open()` requires `O_RDWR` for a `MAP_SHARED` mapping, this surfaces as `mmap` EACCES and is **documented,
   not silently worked around** (a follow-up build with `O_RDWR` would be needed).
2. The MIU physical addresses seen in the audio service's `/proc/306/maps` (e.g. `0x24400000`) are global
   physical addresses and are therefore mmap-able by an independent process — this is precisely the R43-A hypothesis.
3. The device's loader resolves `libc.so`/`libdl.so` (standard on Android; binary is PIE/dynamic).
4. The target device is reachable at `192.168.0.183` and will be used only for a read-only smoke test.

---

## R43-A success criterion

> **Met (build/validation side):** a verified **ARMv7 Android (API 30) ELF32/PIE** binary
> (`miu_mmap_smoke`, SHA `99c3cd02…`) exists and is ready to safely attempt a read-only `mmap()` of
> `/dev/malloc`. Deployment/execution on the projector is **deferred** to a manual step (not performed here).
> Whether the actual `mmap()` succeeds on-device remains to be confirmed by running the binary (left to the
> operator; see §8/§9).
