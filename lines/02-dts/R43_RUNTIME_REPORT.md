# R43 Runtime Report — MIU mmap probe + live SND R2 SHM observation

**Status of this phase: ABORTED / UNSAFE.** The R43-A smoke probe, when executed on the target, is correlated with a
device reset (`bootreason=kernel_pnc`, uptime ≈295 s at forensic time). Per the STOP instruction, all runtime
execution of `miu_mmap_smoke` and any raw `/dev/malloc`/MIU reads is halted; the direct MIU CPU-read path is treated
as **unsafe until proven otherwise**. This report records the smoke-test result and the pivot to safer mechanisms.

---

## 1. Exact device-side smoke-test results

| Offset tested | Result |
|---------------|--------|
| `0x22500000` (and others) | **No usable stdout.** Process output was lost; device reset during/after execution. See §11. |

- The probe (`miu_mmap_smoke`, built R43-A, host-validated ELF32/ARM PIE) was pushed to `/data/local/tmp` and run as
  root. It produced **no observable output**; the device subsequently **rebooted** (kernel panic, see §11).
- **Decision point from R43-B:** none of A/B/C/D could be confirmed — the run was cut short by the device reset before
  any `open`/`mmap`/read result could be observed. The path is therefore **NOT validated as working**.

## 2. Exact working `/dev/malloc` mapping offset(s)
**None established.** The transport test was aborted; no offset was confirmed readable.

## 3. Exact SHM base discovered
**N/A** — discovery was never reached.

## 4. Validation-field values
**N/A.**

## 5. Actual `+0x868` value
**N/A — UNKNOWN (unobservable).**

## 6. Actual `+0x86C` value
**N/A — UNKNOWN (unobservable).**

## 7. Repeatability test
**Not performed — unsafe.** The single execution is already correlated with a device reset.

## 8. Live polling results
**N/A.**

## 9. Codec/output-state correlation
**N/A.**

## 10. Comparison with R42 static writer trace
No new runtime data was obtained. The **R42 static trace remains the only evidence** for `+0x868`/`+0x86C`:
- `+0x868`: 1 producer (`0xCF86` @`0xD16C`/`0xD15E`) + 1 clear (`0xC8BC`/`0xC8C2`), DSP-side, no host reader in artifacts.
- `+0x86C`: 3 producers (`0xCC9B` @`0xCE32`/`0xCE26`, `0xCE7E`/`0xCE72`, `0xCF54`/`0xCF48`) + 1 clear (`0xCAA4`/`0xCAAC`),
  DSP-side, no host reader in artifacts.

## 11. Device-reset evidence (forensic)

| Evidence | Value | Meaning |
|----------|-------|---------|
| `cat /proc/uptime` | `295.39 793.80` | boot ≈ 03:44 (at forensic time 03:49) → recent reboot |
| `ro.boot.bootreason` | **`kernel_pnc`** | non-clean reset; consistent with **kernel panic/watchdog** |
| `/sys/fs/pstore/` | absent | no ramoops → prior-boot crash log **not preserved** |
| `/proc/last_kmsg` | absent | no prior-boot log |
| `dmesg` (current boot) | AVC denials only | no panic/oops residue |
| `/data/local/tmp/s_0x22500000.txt` | gone | **tmpfs wiped by reboot** → cannot see how far probe got |
| `/data/local/tmp/miu_mmap_smoke` | gone | tmpfs wiped |

- **Exact failure point unknown:** the probe's buffered stdout and output file are gone; no pstore/last_kmsg exists.
  Whether the abort occurred during `mmap()` setup or the first CPU read of a device/MIU-backed page **cannot be
  determined** from the surviving evidence.
- **Most plausible cause:** a user-space CPU load from an MIU/device page mapped via `/dev/malloc` triggered an
  unhandled external abort / bus fault → kernel panic (`kernel_pnc`) → reboot. This is consistent with R42's finding
  that this memory is **not** safely CPU-readable even via `access_remote_vm` (EIO on `/proc/pid/mem`).

---

## Safety decision — `miu_mmap_smoke` path declared UNSAFE / ABORTED

The following are **explicitly NOT performed** (per STOP instruction):
- re-running `miu_mmap_smoke`;
- testing other `/dev/malloc` offsets;
- substituting `/dev/miomap`;
- changing `O_RDONLY` → `O_RDWR`;
- building/running `sndr2_shm_probe` or any raw MIU reader on target;
- any `ioctl`/`/proc/utopia_mdb` writes, `/dev/mem`, or `/proc/pid/mem` access.

The direct, independent `/dev/malloc` `mmap()` + CPU-read transport is **not a viable read-only mechanism** on this
platform as exercised — it risks a kernel panic. This was the R43-B hypothesis; it is now negatively resolved.

## Pivot — safer research direction

1. **Static analysis (already done, R29–R42):** the producer/consumer chain for `+0x868`/`+0x86C` is established;
   no host-side reader exists in the available artifacts, so runtime values could not be cross-checked.
2. **Vendor-native, kernel-sanctioned dump paths (preferred):** pursue mechanisms where the **kernel** performs the
   MIU read and emits it (e.g. `MDrv_AUDIO_Dump_SNDR2_Log_Monitor` via an existing ioctl/Utopia command, or a vendor
   `proc`/`sysfs` dump that prints to `dmesg`) — these do not require a user-space process to `mmap()`/`read()`
   device memory and therefore avoid the panic path.
3. **If no kernel-sanctioned dump exists,** the correct conclusion is: **the runtime values of `+0x868`/`+0x86C`
   are not safely observable on this device**, and further attempts via direct MIU CPU reads are unsafe.

## 12. Safer-path (kernel-sanctioned) investigation — pivot result

Per the STOP/pivot instruction, the unsafe `/dev/malloc` `mmap()` path was abandoned and the investigation was
continued on the **kernel-sanctioned** dump path (vendor proc node → kernel reads, prints to dmesg/file; no
user-space `mmap` of device memory, therefore no bus-fault/panic risk). Two vendor interfaces were examined
statically **and** one was safely executed on-device:

### 12.1 `/proc/utopia_mdb/miu` — MIU config/info only
- Verified example commands (literal strings in `utpa2k.ko`): `echo miu_mask info > /proc/utopia_mdb/miu`,
  `echo miu_BW v3 info > /proc/utopia_mdb/miu`, `echo miu_protect > …`, `echo miu_select …`.
- Handler messages enumerated: `MTK MIU mask Info`, `MTK MIU Bandwidth Info`, `MTK MIU SET mask/unmask`,
  `MTK MIU Protect`, `MTK MIU DDR phase …`, `MTK MIU DRAM Size Info`.
- The `saddr=`/`eaddr=`/`Address should be aligned to 4KB` tokens exist (address-aware), but they drive **MIU
  configuration/select/protect**, **not a dump of an arbitrary MIU/SHM memory range**. **No subcommand dumps the
  SHM param region.**

### 12.2 `/proc/utopia_mdb/audio` `dump_r2_log_start` — R2 event log (executed safely)
- Command (literal example in `utpa2k.ko`): `echo 'dump_r2_log_start=0 9E=0x16 PATH=1 8A(88)=0x1' > /proc/utopia_mdb/audio`.
- **Executed on device** (quoted to avoid shell `(` error). dmesg: `Start Dump DECR2 9E:0x16 8A:0x0 88:0x0 Log !!`;
  kernel thread `Dump_DECR2_Log_` wrote `Log path: /data/AudioDECR2_9E0x16_8A0x0_880x0_00.log`.
- **Device remained stable** (no reboot/panic) — confirms the kernel-sanctioned path is safe.
- Log content (18 450 bytes) is the **decoder/R2 runtime event log**: `dec_id`, `main_es_id`, `decType`,
  `smpRate` (48000), `ES=… PCM=… type<3> … frmCnt … avDly …` frames. It is **NOT a hex dump of the SHM param
  region**; searching it for `0x868`/`0x86c`/`SHM`/`COOK`/`param` returns nothing.
- The binary also references `Stop dump SNDR2 Log!!` / `SNDR2 dump is using !!` (an SNDR2 variant exists), but DECR2
  proved the monitor emits **decoder/sound events, not the raw SHM param window** (`+0x868`/`+0x86C` live in the
  SND R2 SHM param region and are not surfaced by this log). `R2_SHM_PARAM_COOK_*` strings belong to SHM-param
  *setup* logging, not to this runtime dump.

### 12.3 Conclusion of the safe-path pivot
- A safe (non-panicking) kernel-sanctioned read interface exists, but **none of the discovered vendor commands
  dumps the raw SND R2 SHM param region** where `+0x868`/`+0x86C` reside.
- `/proc/utopia_mdb/miu` → MIU config only. `/proc/utopia_mdb/audio dump_r2_log*` → R2 event log (not SHM param).
  `read_dsp_sram_type` (R30/R38) → DSP-SRAM mirror (proven ≠ SHM).
- Therefore the **runtime values of `+0x868`/`+0x86C` are not observable through any safe mechanism** available.

## Final determination

Two distinct read strategies are now exhausted:
1. **Independent user-space `mmap()` of `/dev/malloc`** — the only path that reaches the raw SHM, but **UNSAFE**
   (correlated with a kernel panic / `bootreason=kernel_pnc`; aborted).
2. **Kernel-sanctioned vendor dump** (`/proc/utopia_mdb/miu`, `dump_r2_log`) — **safe** but **does not expose the
   SHM param region** (verified).

> **The runtime values of `+0x868`/`+0x86C` cannot be obtained without the unsafe path, which is forbidden.**
> The static chain (R29–R42 + this pivot) fully establishes *what* these fields are (32-bit DTS-output-path
> fields written only by the R2 DSP, no host-side reader in available artifacts) but not their *runtime values*.

## Artifacts (host-only; not deployed/executed)
- `r43a/miu_mmap_smoke.c` (R43-A original + R43-B hardened source).
- `r43a/miu_mmap_smoke` (SHA-256 `99c3cd02ee14f8314f90a53f61ef61ebb87b6947d2dc16871b834f1f92cda94d`).
- `r43a/R43-A_BUILD_REPORT.md`, `r43a/R43_FORENSIC_LOG.md`.
