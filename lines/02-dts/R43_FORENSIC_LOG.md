# R43 Forensic Log — device reset during `miu_mmap_smoke` execution

**Trigger:** User reported the projector rebooted/powered off when running `miu_mmap_smoke`. Per the STOP
instruction, all runtime execution of `miu_mmap_smoke` and any raw `/dev/malloc`/MIU reads is **halted**. The
direct `/dev/malloc` CPU-read path is treated as **unsafe until proven otherwise**. This log records only
read-only, non-invasive forensic evidence (no MIU access, no probe re-run).

## Evidence collected (device: 192.168.0.183:5555, Android 11 / MT5889)

| Check | Command | Result |
|-------|---------|--------|
| connectivity | `adb devices` | `192.168.0.183:5555  device` (reachable at time of forensics) |
| uptime | `cat /proc/uptime` | `295.39 793.80` → **≈295 s** → boot ≈ 03:44 (wall clock 03:49:02) |
| boot reason | `getprop ro.boot.bootreason` | **`kernel_pnc`** (non-clean reset; consistent with kernel panic/watchdog) |
| date | `date` | `Thu Sep 10 03:49:02 EEST 2026` |
| probe output file | `ls /data/local/tmp/s_0x22500000.txt` | **No such file** (tmpfs cleared by reboot) |
| pstore/ramoops | `ls /sys/fs/pstore/` | **No such file** (kernel lacks pstore) |
| last_kmsg | `ls /proc/last_kmsg` | **No such file** |
| current dmesg tail | `dmesg \| tail` | only AVC denials (bluetooth hidraw / binder); **no panic/oops/MIU messages** |
| probe binary | `ls /data/local/tmp/miu_mmap_smoke` | **gone** (tmpfs cleared by reboot) |
| tombstones | `ls /data/tombstones/` | permission denied (not inspected further) |

## Interpretation

1. **A reboot occurred** ~03:44, i.e. shortly after the probe execution window. `bootreason=kernel_pnc` indicates the
   **kernel itself reset** (panic/watchdog), not a clean user-initiated reboot.
2. **No panic log survived** (no pstore/last_kmsg; current-boot dmesg is clean). The probe's stdout/`s_*.txt` output
   file is also gone because `/data/local/tmp` is tmpfs and was wiped by the reboot. Therefore the **exact failure
   point (during `mmap()` setup vs the first CPU read vs later) cannot be determined** from available evidence.
3. **Most plausible mechanism:** the independent `mmap()` of `/dev/malloc` followed by a CPU load from a
   device/MIU-backed page triggered an **unhandled external abort / bus fault** that escalated to a kernel panic.
   This is consistent with the R42 finding that MIU/device memory is **not** readable via `access_remote_vm`
   (`/proc/pid/mem` EIO) — i.e. the region is not safely CPU-accessible through a naive user-space `mmap()` either.
4. The event is **correlated with** running the probe (user observation + `kernel_pnc` + tmpfs-wiped artifacts), but a
   strict causal proof is not obtainable because the crash log did not persist.

## Conclusion

The independent `/dev/malloc` `mmap()` + CPU-read transport (R43-A/B smoke test) is **declared UNSAFE / ABORTED**.
It is **not** to be re-run; no other offsets are to be tested; `/dev/miomap` is **not** to be substituted;
`O_RDONLY` is **not** to be changed to `O_RDWR`; `sndr2_shm_probe` is **not** to be built/run on target.

Local artifacts preserved (do NOT deploy/execute):
- `r43a/miu_mmap_smoke` (SHA-256 `99c3cd02ee14f8314f90a53f61ef61ebb87b6947d2dc16871b834f1f92cda94d`) — host copy only.
- `r43a/miu_mmap_smoke.c` (hardened R43-B source) — host copy only.
