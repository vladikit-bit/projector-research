# GLM — E1 live experiment: IPAUTH bitmap upload (state report, session paused)

**Date:** 2026-09-27. User-approved runtime test, paused by user request mid-experiment.
**Companion to:** `forensic_glm_ipauth_provisioning_differential_20260927.md` (static chain).
**Device:** 192.168.0.183:5555 (C50A), `adb root` active. adb wrapper: `aeon_validate/adbr.sh`
(port 5038).

---

## 1. What was executed

1. Built ARM32 tools with NDK r30 (`armv7a-linux-androideabi29-clang`), sources and
   binaries in `C:\firmware_temp\e1_ipauth\`, pushed to `/data/local/tmp/`:
   - `e1_upload` — opens `/proc/utopia` O_RDWR, issues
     `ioctl 0xC08C5506` (140 B: hdr[9]=0x80, bitmap[0x0A..0x89]=0xFF) and
     `ioctl 0xC0125507` (18 B: gCusID=0x0000 + KeyCustomer[0] hash
     `285bf7c694cf2c6b8b67ffdae8c01845`).
   - `e1_probe` — dlopen `/vendor/lib/libutopia.so`, `info` = call
     `MApi_AUDIO_GetAudioInfo2(0, 0x30, 0)`, `init` = call `MApi_AUDIO_Initialize()`
     (NOT yet run on device).
2. Ran `e1_upload`: **both ioctls returned 0 (accepted)** — the 140-byte bitmap and the
   18-byte credential are in kernel RAM.
3. A/B capture around DTS playback (`test_dts_51.mp4` via Kodi, the known
   `am start -n net.kodinerds.maven.kodi22/.Splash` invocation):
   - decoder status via `/proc/utopia_mdb/audio` `show_all_decoder_status` (slot 0 rule);
   - RIU via `/proc/utopia_mdb/riuinfo` (`Dump,0x112e`, `Dump,0x112a`; results go to dmesg);
   - `dump_spdif_npcm=1 path=1` → `/data/DUMP_audio_spdifNpcm_00.bin`.

## 2. Results

| observable | A (pre-upload, DTS playing) | B (post-upload, DTS playing) |
|---|---|---|
| decoder slot 0 | format DTS, PLAY, hash UNSUPPORTED | (played out before check) |
| mdb bank 0x112E +0x98 | 0xC404 (idle: 0x0003) | 0x8404 |
| mdb bank 0x112A +0x84/85 | **0x84/0x1F** | **0x84/0x1F** (unchanged) |
| mdb bank 0x112A +0x82/83 | 0x00/00 | **0x00/00 (unchanged)** |
| npcm dump file | — | **0 bytes — gate still closed** |

- The `0x84/0x1F` pair is exactly the static `SET_IPAUTH_GROUP` group-1 signature
  (cmd 0x84 at 0x112A84, 0x1F at 0x112A85) — the command fires on every SetDecodeSystem,
  and its flags byte (`0x112A82`) is **0x00** — the empty-verdict state, live-confirmed.
- **E1 outcome: upload accepted but inert as applied.** Nothing recomputes the verdicts
  after the upload.

## 3. Why — the newly-discovered CheckHashkey single-run semantics (static, byte-verified)

```
MDrv_AUDIO_CheckHashkey @0x423494:
  0x4234C0  ldrb r1,[AudioVars2+0x503]; cmp #1; beq EXIT     ← SKIP GATE
  0x4234D8  str 0,[+0x440]      ; group0 flags = 0   (bits SET on pass)
  0x4234DC  str 0x00FFFFFF,[+0x444] ; group1 flags = all-1 (bits CLEARED on fail — reversed polarity)
  ... ~90 IPCheck verdicts → +0x440/0x444/0x4D8 ...
  0x4248A8  bl <cond>; if ==1 → strb 1,[+0x503]              ← SELF-DISABLE after first run
MDrv_AUDIO_DspReboot @0x405aa4:
  0x405AD8  strb 0,[+0x503]        ← the ONLY zero-writer (re-arms)
```

The device booted → first audio init ran CheckHashkey **once with a NULL bitmap** (all
verdicts fail) → set `+0x503=1` → every later CheckHashkey (SetDecodeSystem, SetSystem,
SetAudioParam2, Initialize) is a no-op. Our upload changed the bitmap source but nothing
re-reads it. `MDrv_AUDIO_DspReboot`'s single caller is `_MApi_Audio_Monitor` @0x3E32CC
(trigger: `[global] != [sp+0x70]` — a version/state mismatch, not yet mapped to a
public trigger).

## 4. Mapping correction (important for all future register reads)

`HAL_AUDIO_AbsWriteByte(addr)` @0x441CDC computes its window byte offset as
**`2*(addr - 0x100000)`** (`sub r1,r5,#0x100000; rsb r7,r2,r1,lsl#1` with the window base
from GOT). So the ARM "0x112E9A"/"0x112A82" conventions do NOT necessarily equal mdb
`riuinfo` banks `0x112E/0x112A`; mdb banks `0x12E/0x12A` read all-zero. The reliable live
read of the engine word is the exported query itself — `MApi_AUDIO_GetAudioInfo2(0, 0x30,
c)` — whose wrapper (libutopia @0x34BD44) packs `{ret@+0, a@+1, b@+5, c@+9}` and issues
`UtopiaIoctl(h, 0xCF, payload)`. Note the wrapper returns only payload **byte 0**.

## 5. Next steps (exactly where we stopped)

1. Push the rebuilt `e1_probe` (3-arg `info` call; the 2-arg attempt segfaulted the probe
   process — signature now fixed locally, push was interrupted):
   `MSYS_NO_PATHCONV=1 adbr.sh push C:/firmware_temp/e1_ipauth/e1_probe /data/local/tmp/`
2. `/data/local/tmp/e1_probe info` — idle and during DTS playback: the ARM-side view of the
   0x30 engine word (static expects low5==5 read; expect 4/"unlicensed" now).
3. `/data/local/tmp/e1_probe init` — `MApi_AUDIO_Initialize()` (risk: middleware re-init;
   recovery = restart `vendor.audio-hal`, done before). It calls CheckHashkey+ApplyHashkey
   directly and, if it reaches DspReboot, clears `+0x503`.
4. Re-run DTS → re-check: 0x30 word (0x97?), `Dump,0x112a` flags byte, npcm dump size.
5. Interpretation:
   - flags now non-zero AND npcm > 0 → **unlocked**;
   - flags non-zero, npcm still 0 → SE rejects CPU-provided flags (outcome В) → E2:
     real `MApi_AUTH_Process` via dlopen with chip string + expected hash (trace
     mi_union ctx+0x4C source), or DEC/T615-blob differential;
   - 0x30 word stays 0x04 after init+playback → the recompute path needs more (map the
     Monitor's DspReboot condition).

## 6. Device state left behind

- `/data/local/tmp/e1_upload`, `/data/local/tmp/e1_probe` (harmless diagnostics; delete
  with `rm`).
- Kernel holds the uploaded bitmap+hash (RAM, gone on reboot; harmless while
  CheckHashkey is self-disabled — the all-licensed bitmap only affects future verdict
  computations).
- `e1_probe` `init` was **NOT executed** — the middleware was not re-initialized.
- Kodi last launched for the B-side capture; file `test_dts_51.mp4` in `/sdcard/Movies/`
  as before; AC3 baseline untouched this session.


---

## 7. ADDENDUM (2026-09-27, evening) — live continuation results

### 7.1 What was attempted after the pause

1. `e1_probe` (dlopen libutopia + MApi calls) — **crashes in userspace**: tombstone shows
   SIGSEGV in `UtopiaOpen+44` (NULL framework global — per-process utopia framework not
   initialized; `UtopiaInit` also faults). Foreign-process MApi driving is a dead end
   without full MsOS/framework init (only the HAL does that).
2. `MApi_AUDIO_Initialize` = `UtopiaIoctl(hAUDIO, cmd 0, NULL)` (userspace wrapper
   @0x33D508 confirmed; kernel ops+0x00 slot) — untestable for the same reason.
3. HAL restart (`kill mstar.hardware.audio.service`) — does NOT re-run kernel
   `_MApi_AUDIO_Initialize`; no CheckHashkey banner afterwards.
4. **Hand-crafted kernel module** (`e1_ipauth/dspreboot1.ko`, donor = running
   `/vendor/lib/modules/utpa2k.ko`, md5 = 2fc6e9fc… = stock): after fixing
   `e_flags` (EABI_VER5 0x5000200), `this_module` name offset (**0x0C**, not 0 — vendor
   struct has state+list first), `mod->init` relocation (@0xD8) and `__versions` CRCs
   (`module_layout`=0xe22991c4; `MDrv_AUDIO_DspReboot`=**0** — exporter registered zero),
   the module **loaded and executed `MDrv_AUDIO_DspReboot(0)`… and the projector went
   down** (user power-cycled). Root cause of the crash not captured (no console). This
   route is **destructive in the tested form** — DspReboot from a foreign module context
   with the HAL active is unsafe.
5. `MDrv_AUDIO_Init` (short Initialize path) does NOT call DspReboot / touch +0x503 —
   suspend/resume route is useless for re-arming CheckHashkey.

### 7.2 The boot-race test (safe variant) and its result

After the reboot, with NO decoder opened yet (no MI_AUDIO_Open in dmesg), the bitmap was
re-uploaded **before any playback** — then DTS playback started (decoder slot 0: DTS/PLAY;
the first post-boot SetDecodeSystem ran WITH the bitmap in the kernel). Result:
- no CheckHashkey banner (`===== Check Audio Decoder Protection from hash-key IP =====`)
  in a tight window → **CheckHashkey did not run** → the +0x503 gate was already closed:
  early-boot audio activity (HAL init SetSystem/SetAudioParam2) consumed the first
  CheckHashkey **before network adb is reachable** (adbd ~31.5 s; HAL audio init earlier).
- SE flags byte (bank 0x112A +0x82) still 0x00; npcm dump 0 bytes.
- The boot-race via adb is **unwinnable**: HAL audio init precedes network adbd.

### 7.3 Consolidated state of the unlock chain

```
upload path        WORKS   (kernel accepts bitmap+hash any time; tool e1_upload)
gate re-arm        BLOCKED (+0x503=1 set by first post-init SetDecodeSystem during boot)
DspReboot trigger  UNSAFE  (module-context call = projector down; only safe caller =
                            _MApi_Audio_Monitor's DSP-hang recovery)
foreign-process    BLOCKED (Utopia framework not initializable standalone)
MApi call surface  BLOCKED (UtopiaOpen crashes; HAL-only)
```

### 7.4 Remaining options (need decision)

- **R1 (risky, one more shot):** "quiet" DspReboot — stop the audio HAL first
  (`kill mstar.hardware.audio.service`), then insmod the trigger module (DspReboot with no
  active traffic), rmmod, let the HAL restart → its first SetDecodeSystem runs
  CheckHashkey with the already-uploaded bitmap. Same failure class as the crash that
  power-cycled the box — **explicit user approval required**.
- **R2 (safe, static):** T615 DEC/SND DSP-blob differential — does the DTS-working
  platform have the same `0xB000001E==5` gate, and what arms it there (the answer may
  expose the real SE provisioning input).
- **R3 (safe, static):** reverse the `send_cmd=`/`dec_r2_cmd=` mdb commands (driver-side
  command dispatch may include a DSP-reload command reachable from the shell).

### 7.5 Device state after all tests

Fresh reboot performed by the user; nothing persistent changed. Left on device
(/data/local/tmp/): e1_upload, e1_probe, e1_probe3, dltest, dltest2, dsptest1.ko,
dspreboot1.ko — all inert diagnostics, removable. Kernel holds a freshly uploaded bitmap
(harmless — gate closed). Audio: DTS decoded on slot 0 during tests but **no speaker
sound** was heard this evening (unconfirmed whether DTS-on-speaker was ever audible
before; AC3 baseline NOT re-verified — recommend an AC3 ear-check).


---

## 8. ADDENDUM 2 (2026-09-27, night) — second shutdown; module line CLOSED

### 8.1 What happened

`dsprearm2` (donor-surgery on the running utpa2k image: movw/movt retargeted to
`g_AudioVars2`, `ldr` + `strb 0,[r0+0x503]`, early return; **no** DspReboot call — the
previous trigger reloc was neutralized by an early `pop`) was insmod'ed → the adb session
died and the projector powered off again (user power-cycled). Whether the byte write
executed is unknown (no serial console, no logs). **Both donor-surgery module attempts
ended in a system shutdown. The module line is cancelled entirely** — do not retry any
variant without a serial console attached.

### 8.2 User correction adopted

The T615/V440 firmware has **no runtime proof** that DTS (or AC3) passthrough worked on
that exact image (standing corpus constraint since 2026-09-21; user re-confirmed). The
phrase "DTS-working platform" used earlier for T615 is **retracted**. Correct statement:
the DEC-blob gate code (`0x21F40..0x21F80` region, `d3650092` L1 instruction) is
byte-identical between TD98's `mst_codec_r2_MS12V22` and the T615-embedded blob
(two copies at file offsets 0x7B433A / 0xA4E20C of the T615 `utpa2k.ko`; blob extracted to
`e1_ipauth/t615_dec_blob.bin`, 2 727 634 B — a newer, larger build with the licence string
at blob offset 0x1D1A6B vs TD98's 0x144904; DTS-branch region differs only 77/1280 bytes).
Interpretation: the gate is universal for this DSP code lineage; **nothing is concluded
about any platform's SE provisioning state**.

### 8.3 R3 result (static, closed)

- mdb command strings (`send_cmd`, `dec_r2_cmd`, `show_all_decoder_status`) exist in
  utpa2k_stock.ko .data but have **zero relocations referencing them** (verified: symbol
  scan, section-symbol+addend scan, named-symbol scan) — dead data; the live
  /proc/utopia_mdb/audio write handler is NOT in utpa2k.ko (kallsyms shows only generic
  mdb_* node handlers for rtc/mbx/sc subsystems builtin in the kernel; no audio node
  symbols visible). R3 static route exhausted without a reachable DSP-reload command.
- Exported-API survey: `MApi_AUDIO_RebootDsp` → `HAL_AUDIO_RebootDecDSP` (SIF cmds
  0x44/0x45 + writes 0x112A80=2,3) — does **not** touch +0x503; `MApi_AUDIO_ExitAudioSystem`
  → `SetPowerOn(0)` — does not touch +0x503 or INIT_FLAG (AudioVars2[8]);
  `HAL_AUDIO_GET/SET_INIT_FLAG` = AudioVars2[8]; `MDrv_AUDIO_DspReboot` clears +9/+0x503
  and contains CheckHashkey+ApplyHashkey — the only complete re-arm, and destructive when
  invoked from a foreign module context with the HAL loaded.

### 8.4 Where the unlock chain stands (night state)

```
bitmap upload            WORKS (repeatable, accepted)
gate re-arm (+0x503=0)   ONLY via MDrv_AUDIO_DspReboot — destructive from module context
safe driver-side re-arm  NOT FOUND (R3 closed)
T615 differential        gate universal; provisioning difference unproven (V440 runtime unknown)
```

The realistic remaining paths, in order of safety:
1. **Serial console** (UART) attached → capture the module crash reason → then, and only
   then, reconsider a corrected module (or understand the actual fault).
2. Static: find TD98's MI-union/middleware binary on the projector and check whether its
   `MI_SYS_Init` lacks the `MApi_AUTH_PROCESS` call that T615's has (the upstream reason
   nobody uploads) — pure corpus work, no device risk.
3. Static: full crypt reconstruction of `MApi_AUTH_Process` (drvIPAUTH.c) so that, IF a
   safe execution context is ever found, the correct arguments are ready.


---

## 9. ADDENDUM 3 (2026-09-27, night) — the upstream is found: TD98 middleware lacks the AUTH caller

### 9.1 TD98 HAS the same tvservice architecture as T615

`/vendor/bin/cpu_audio.sh` launches `/mnt/vendor/tvservice/bin/cpu_audio` (7 375 624 B —
nearly identical size to T615's 7 371 528) via a glibc rootfs
(`/mnt/vendor/tvservice/glibc/ld-linux.so.3`). The projector runs the same MStar
middleware tree as the TCL set.

### 9.2 The provisioning caller is ABSENT on TD98

Exhaustive string search of the whole `/mnt/vendor` tree:
- `MI_SYS_Init` — present in cpu_audio ✓
- `MApi_AUTH_PROCESS` — **absent everywhere** (T615 has it: ipauth provider lib + the
  MI-union binary importing it, called from MI_SYS_Init @0xAB32C)
- `UtopiaSetIPAUTH` — only in libutopia.so and (as framework references) in
  glibc/libMsOS.so, glibc/liblinux.so
- `/mnt/vendor/tvservice/bin/` contains only `cpu_audio`, `midaemon`, `script/` — the
  T615-style MI-union binary (3.8 MB, jenkins `9615_AmatiGB`) **does not exist** on TD98.

**Conclusion: the missing piece of the whole chain is the middleware CALLER.** The kernel
(handler, zero-validation upload), libutopia.so (complete engine: Process + ipauth crypto +
UtopiaSetIPAUTH + KeyCustomerList), and the DSP gate are all present and functional; the
boot-time uploader binary was cut/never shipped on this projector.

### 9.3 TD98's own MApi_AUTH_Process is complete (annotated disassembly saved)

`e1_ipauth/auth_process_td98.txt` — full 748-line annotated disassembly of
MApi_AUTH_Process @0x4C6768 in the running `/vendor/lib/libutopia.so`
(PLT/GOT-resolved). Call graph: `__android_log_print ×11, __strlen_chk ×5,
MDrv_AUTH_AllocateVars ×3, strlen ×3, MDrv_SYS_GetChipID ×3, helper 0x4C7BF4 ×3,
__aeabi_memclr8 ×2, strncpy ×2, __strncpy_chk2 ×2, MDrv_SYS_Init ×1, memcmp ×1,
MDrv_SYS_GetChipType ×1, MDrv_SYS_Query ×1, **UtopiaSetIPAUTH ×1**, __stack_chk_fail ×1`.

Partially reconstructed so far:
- arg0 = NUL-terminated chip hex string (≤0x31 chars enforced via __strlen_chk);
  bitmap size byte = ((strlen+0x1F0)>>1)&0xFF (for len=0x30 → 0x10, same value as
  T615's ((len-0x10)>>1) formula) → written to AudioVars2[9] AND to a global @0x127E0A8.
- A 32-char chunk of the tail of arg0 is memcmp'ed against 32 ASCII '0's (padding check,
  const @0x1D7C61) — chunks are then split via __strncpy_chk2 into two 16-byte halves
  (sp+0x140/sp+0x150) with limit 0x31.
- `MDrv_SYS_GetChipType`/`MDrv_SYS_GetChipID` → chip-ID table walk @0x10D5B04
  (24-byte entries {u32 limit; u8 value[17]; pad}): entry0 {limit 0, value =
  **285bf7c694cf2c6b8b67ffdae8c01845** = KeyCustomer entry0}; entry1 {limit 0xFFFFFFFF}
  is a terminator (always skipped). Selected 17-byte value lands in a local buffer.
- A helper @0x4C7BF4(buf, str, len) (called ×3 — the MD5/encode stage) then the compare
  and the UtopiaSetIPAUTH(bitmap, cusID+hash) call.

Still to reconstruct (next session, from the saved file): helper 0x4C7BF4 internals, the
final bitmap-bit derivation loop, gCusID derivation, and the exact UtopiaSetIPAUTH argument
layout for this build.

### 9.5 Note on "dead" mdb strings (user caution adopted)

The mdb command strings in utpa2k .data show zero static relocations, but the user
recalls a past bootloop from "unused" regions that turned out to be live at boot. The
"dead data" conclusion for the mdb table is therefore **provisional**; the
/proc/utopia_mdb/audio write handler location remains unlocated (kernel builtin or
dynamically bound).


---

## 10. ADDENDUM 4 (2026-09-28) — MApi_AUTH_Process reconstruction, TD98 build (major)

Source: `e1_ipauth/auth_process_td98.txt` (748 annotated lines, running
`/vendor/lib/libutopia.so`, MApi_AUTH_Process @0x4C6768, size 0xC58).

### 10.1 Full reconstructed flow

```
MApi_AUTH_Process(arg0 = char* hex_string, arg1 = u8 expected_hash[16])
  1. AllocateVars if needed; MDrv_SYS_Init; *g_IpAuthVars header init:
     [8]=0x8000>>? — strh 0x8000 at [+8] (16-bit!), bitmap [+0xA..] cleared (vst1 zeros),
     i.e. table[8..9] = 0x00,0x80 → size field [9]=0x80 preset.
  2. len = strlen(arg0);  size = ((len + 0x1F0) >> 1) & 0xFF  → AudioVars2[9] AND
     global @0x127E0A8 (for len=0x30 → 0x10; equivalent to T615's ((len-0x10)>>1)).
  3. Copy arg0 → local (0x111 buffer); if strlen ≥ 0x31: take last 0x40 chars,
     split into two 0x20 halves; memcmp(half1, const@0x1D7C61+8, 0x20)
     — the constant region contains 32 ASCII '0's → padding check on the tail.
  4. ChipType/ChipID → chip table walk @0x10D5B04 (3 × 24B {u32 limit; u8 value[17]; pad}):
     entry0 {0x00000000, 285bf7c6…1845} (KeyCustomer hash) is copied when chipID ≤ 0
     (i.e. always for ID 0); entry1 {0xFFFFFFFF, "IpAuth_Mutex" ASCII} is a terminator —
     its copy branch is always skipped. Selected 17B value → local.
  5. MD5/encode helper 0x4C7BF4 ×3:
     a) (buf=sp+0x178 seeded with a const pointer + selected customer key, arg0, strlen)
     b) (buf, fixed pc-rel string, len derived from helper-a's 64-bit counter: ubfx #3,#6,
        threshold 0x38/0x78 → 0x38 or 0x78 minus field)
     c) (buf, 8 bytes saved from helper-a's counter, 8)
  6. First 16 bytes of the buffer vs arg1[0..15] — 16 byte compares. Mismatch does NOT
     abort: sets global auth-flag @lit(0xdb75f1) to 0 and logs; match keeps it 1
     (set before the compare). arg1 (16 B) is copied to a global = **gCusHash** either way.
  7. Hex parse of arg0 chars [8..15] (two pairs → 2 bytes) → **gCusID** global (2 B)
     and bitmap table[8]; more nibble math (NEON vcgt/vbsl hex-convert) → a u32 →
     compared: if (chipID & 0xFFFE6666) == 0x6666 or chipID == parsed value → OK,
     else "Wrong Chip ID" log + auth-flag = 0.
  8. Bitmap fill, two branches:
     a) strlen(arg0) == 0x30: for i in 0..15: hex-pair arg0[0x10+2i], arg0[0x11+2i]
        → byte → bitmap[ (global_byte_at_r8 - 6) + i ]   ← 16 licence bytes from the
        string land in the bitmap at an offset taken from a global;
     b) else: if global_byte != 0 → zero-fill bitmap[0xA .. 0xA+global_byte].
```

### 10.2 Interpretation

**arg0 is not a chip serial — it is a 48-hex-char LICENCE BLOB** (24 bytes of payload:
[0..7] unknown, [8..15] → gCusID + bitmap[8], [16..47] → 16 licence bytes written into
the IP bitmap), authenticated by arg1 = 16-byte digest (MD5-family keyed with the
customer key + fixed strings). The licence DATA therefore arrives from outside the
function — the caller (T615: MI_SYS_Init ctx+0x4C; missing on TD98) supplies both the
blob and its expected digest. On a licensed device these come from the vendor's
provisioning data; the function only verifies (keyed digest) and distributes them
(gCusID, gCusHash, bitmap bits) before UtopiaSetIPAUTH uploads to the kernel.

Consequence for a TD98 activation: calling Process with arbitrary arg0/arg1 will fail the
keyed-digest check (auth flag 0) — but note the mismatch **does not abort**: the flow
still parses the hex blob and fills the bitmap/gCusID/gCusHash and calls
UtopiaSetIPAUTH. Whether the SE accepts a "flag-0" provision is exactly what a live test
would answer — but all safe execution contexts are exhausted (see §7-8); only a serial
console changes that.

### 10.3 Still open in the reconstruction

- helper 0x4C7BF4 internals (MD5 transform wrapper vs custom encoder) — needed to compute
  valid (arg0, arg1) pairs offline;
- the identity of the pc-rel fixed string appended in helper call b;
- the meaning/origin of the 8 bytes appended in call c;
- the global supplying the bitmap write offset (branch a).


---

## 11. ADDENDUM 5 (2026-09-28) — crypto fully identified: MD5 with customer-key IV

### 11.1 Helper 0x4C7BF4 = standard MDrv_AUTH_MD5Update

Context layout: `{u32 state[4] @+0x00; u64 bitcount @+0x10; u8 buffer[64] @+0x18}`.
Verified from the disassembly: bitcount held in BITS (`[ctx+0x10] += len<<3` with carry
into +0x14), byte offset in block = `(bits>>3)&0x3F`, buffer at +0x18, transform called
at 0x4C7DA0. The transform contains the complete standard MD5 K-constants as movw/movt
pairs (0xd76a/a478, 0xe8c7/b756, 0x2420/70db, 0x8b44/f7af, 0xf57c/0faf, …) — **plain
MD5, no whitening**.

### 11.2 The three Update calls = manual MD5 final

```
ctx.state[0..3] = IV built from the selected chip-table entry (24 B @0x10D5B04):
                  word0 = *(table+8) (key bytes 4..7 LE), state[1..3] = key bytes 1..12
                  (exact byte order to be pinned; the whole IV derives from
                   285bf7c6 94cf2c6b 8b67ffda e8c01845 — no external secret)
Update#1 (arg0 string, strlen)
Update#2 (0x80 + zero padding, padlen = 0x38-len%64 if <0x38 else 0x78-len%64)
         — the "fixed string" @0x10D5B2D is the 0x80 byte after "IpAuth_Mutex "
Update#3 (8 bytes = ctx.bitcount lo/hi)   ← length suffix as a separate update
digest = ctx.state[0..3]  (16 B)
```

So **digest = MD5_with_custom_IV(arg0_string)**, and arg1 must equal it. Everything
needed to compute valid (arg0, arg1) pairs offline is embedded in the firmware: the IV
bytes (customer table), the transform (standard MD5), the padding scheme (standard).
Only the IV byte-order and the count-suffix byte order still need pinning.

### 11.3 Consequences

1. A valid (arg0, arg1) licence pair is **computable offline** for ANY chosen 16-byte
   licence payload (the bitmap bits) — no vendor secret is involved; the "key" is the
   firmware's own customer hash.
2. The digest check failing does not block distribution (flag 0, flow continues), so
   even a mismatched pair would still deliver bitmap bits + gCusID + gCusHash to the
   kernel via UtopiaSetIPAUTH.
3. The remaining operational blockers are unchanged and now precisely bounded:
   a) a process context that may call MApi_AUTH_Process (absent middleware on TD98), or
      equivalently direct ioctls with offline-computed payloads (works — e1_upload);
   b) the +0x503 single-run gate on CheckHashkey (closes at boot before adb is up);
   c) the SE's acceptance of the resulting group flags (untested — the crash-risky
      DspReboot re-arm was the only known way to reach this test).

### 11.4 Connection to the user's firmware research

The community pattern "native TCL player works, Kodi/Emby/Nova do not" fits the
mechanism: DTS passthrough on TCL is wired through the private middleware pipeline that
performs this provisioning (MI_SYS_Init → MApi_AUTH_PROCESS → … → SE), while third-party
players never trigger it. The "working" firmwares (T615T01 R074/V098, T615T03 V474) are
therefore expected to contain the SAME gate but a middleware that provisions — pulling
`utpa2k + libutopia + tvservice bin` from R074/V098/V474 and diffing
(a) the MI_SYS_Init call site and (b) the licence blob construction would yield a real
(arg0, arg1) sample pair and confirm the IV byte-order empirically.


---

## 12. ADDENDUM 6 (2026-09-28) — THE CHAIN IS ALIVE ON TD98; it breaks inside MI_SYS_Init

### 12.1 The middleware chain is fully present and RUNNING on the projector

- `/mnt/vendor/tvservice/glibc/libdrvIPAUTH.so` (26 148 B) — **exports the complete
  IPAUTH family including MApi_AUTH_Process**; KeyCustomerList identical (285bf7c6…).
- `/mnt/vendor/tvservice/glibc/libmi.so` (1 110 720 B — a reduced build vs V474's
  3 784 200) — **imports MApi_AUTH_PROCESS** (.rel.plt slot 0x11BB20, PLT stub 0x17204)
  and **calls it at 0xB0114 inside MI_SYS_Init (@0xAE8B4)** — same call shape as
  V440/V474: `r0 = &chip_string_global (@0x128060, .bss)`, `r1 = ctx->[0x4C]`.
- `cpu_audio` (PID 486) runs since boot with BOTH libs mapped (verified in
  /proc/486/maps; the earlier "MApi_AUTH_PROCESS not found anywhere" was a
  **case-sensitivity artifact** — the symbol is mixed-case `MApi_AUTH_Process`).

### 12.2 Live memory proof of where the chain breaks

Reading the running process via /proc/486/mem (custom `memread` tool, pread64):
- libmi.so base 0xB1630000; libdrvIPAUTH base 0xB0BD2000 (ELF header verified in-memory).
- KeyCustomerList (live .data) = the same three customer entries (285bf7c6…).
- **g_IpAuthVars (libdrvIPAUTH .bss +0x16588) = 0x00000000** → MDrv_AUTH_AllocateVars
  never ran → **MApi_AUTH_Process never executed** in the live middleware.
- **The 48-byte chip-string global (libmi .bss +0x128060) = all zeros** → the input data
  was never filled; the `'0'`-padding writes (0xB0058-74) never executed → MI_SYS_Init
  exits (or skips the AUTH block) before the string is built.

### 12.3 The corrected causal model

```
TD98 boot:
  cpu_audio runs, libmi.so + libdrvIPAUTH.so loaded
  MI_SYS_Init runs... exits BEFORE building the 48-hex chip string
    → MApi_AUTH_PROCESS never called (g_IpAuthVars NULL)
    → no UtopiaSetIPAUTH, no kernel bitmap
  audio HAL first SetDecodeSystem → CheckHashkey runs ONCE with NULL bitmap
    → all verdicts fail → +0x503 = 1 (self-disable) → the gate is closed forever
TCL TV (T615):
  same chain, but the chip string IS filled (from TCL provisioning data)
    → AUTH OK → bitmap uploaded → SE armed → DTS passthrough (native player only)
```

The "native player only" community pattern and the projector's silence now have the same
root: the provisioning needs the chip-string licence data that ships with TCL TVs.

### 12.4 Next leads (all static/safe)

1. **Who fills the chip string @libmi+0x128060?** MI_SYS_Init references it ×4
   (0xAED08/0xAF0E0/0xAFCD8/0xAFF08) and `MI_SYS_GetConfigData`/`SetConfigData` sit
   nearby — the data likely comes from a **config file** (tclconfig/tvconfig partition or
   /mnt/vendor/tvservice/etc). Find the reader → the file path → check whether the
   projector has that file (empty/absent on TD98 = the root cause).
2. V474's `tclconfig.img` (125 MB, inside the OTA zip) is the prime candidate for the
   licence/chip data source — extract and search for the 48-hex-char pattern /
   IPAUTH-related keys.
3. V098 (ATV branch) comparison for the same file.

Everything above is static/live-read only; the device was not modified.


---

## 13. ADDENDUM 7 (2026-09-28, night) — E2′ executed: direct SE injection REJECTED

### 13.1 Calibration (important — validates the indicator)

`dump_spdif_npcm` IS a valid gate indicator:
- during **working AC3 passthrough** (decoder AC3P, PLAY): the dump captures
  **~1 MB** (`DUMP_audio_spdifNpcm_01/02.bin`, 927 744 / 1 148 928 B) — payload flows;
- during **DTS decode** (decoder DTS, PLAY): dump = **0 bytes** — nothing reaches the
  transmitter. (Earlier AC3P-instead-of-DTS readings were Kodi's transcode setting,
  user-confirmed.)

### 13.2 E2′ sequence and result

With the bitmap re-uploaded (e1_upload accepted), three injection rounds — before decode,
during decode, and repeated:
```
ByteW,0x112a,0x82,0xFF,0xFF   (group-1 licence flags = all)
ByteW,0x112a,0x84,0x84,0xFF / ByteW,0x112a,0x85,0x1F,0xFF  (re-latch command)
ByteW,0x112e,0x98,0x97,0xFF / ByteW,0x112e,0x9a,0x97,0xFF  (engine byte = DTS licensed)
```
Readbacks: **0x112E98 = 0x97 holds** (even across decode start — the driver did not
overwrite it on this path); **0x112A82 = 0x00** — the SE does not accept a direct flags
write (consumed/reset). Decoder DTS/PLAY, npcm dump **0 bytes** → gate closed.

### 13.3 Conclusion

The SE does not accept forged group flags through raw register writes. The licence
verdict at 0xB000001E requires the proper kernel-side command path (SE-IDMA handshake +
whatever internal state the SE keeps), and that path is gated by CheckHashkey's single-run
flag. E2′ is a clean negative: **the SE validates, not just stores.**

### 13.4 Updated blocker ladder

```
1. +0x503 re-arm            — only DspReboot (destructive from module context)
2. SE acceptance            — rejects raw flag injection (needs kernel command path)
3. Both together imply      — the only viable unlock = the middleware calling the chain
                              for real (valid licence blob) BEFORE the gate closes at boot,
                              or a serial-console-guided deeper intervention
```

Realistic continuations: (a) V098 ATV-branch analysis (in progress by Muse) — check its
tvservice middleware for a DIFFERENT provisioning trigger (ATV + Toslink is the reported
working combo); (b) find the chip-string source on a working TCL (tclconfig/tvconfig were
clean — the blob may live in the native player APK or arrive via network/TV services);
(c) serial console for a captured, controlled DspReboot.


---

## 14. ADDENDUM 8 (2026-09-28) — licence-kit GENERATION (user-approved)

### 14.1 The generator

`e1_ipauth/gen_licence.py` — implements the reconstructed MApi_AUTH_Process crypto:
- custom-IV MD5 (state = KeyCustomer entry0 as 4 LE words: IV =
  {0xC6F75B28, 0x6B2CCF94, 0xDAFF678B, 0x4518C0E8} from 285bf7c6 94cf2c6b 8b67ffda e8c01845);
- driver's finalisation: padlen = 0x38-(len%64) if <0x38 else 0x78-(len%64); the 8-byte
  bit-count appended as data (for 48-char arg0: 0x80+7 zeros + u64le(384) — one block);
- the transform validated against hashlib with the STANDARD IV (5/5 vectors).

### 14.2 Generated kit (licence payload = all 128 IP bits set)

```
arg0 = "0000000000000000ffffffffffffffffffffffffffffffff"   (48 hex chars)
arg1 = 8b8fc17688e820323cb640f0ee533a9b                     (16-B custom-IV digest)
kit_bitmap140.bin  — 140-B ioctl 0xC08C5506 payload (hdr[9]=0x80; all 1024 IP bits set)
kit_hashinfo18.bin — 18-B ioctl 0xC0125507 payload (gCusID=0 + gCusHash=arg1)
self-check: single-block compress(IV, block) == padded-stream result ✓
```

### 14.3 What this completes and what it does not

COMPLETE: the data side. A self-consistent (arg0, arg1) pair whose digest verifies
under TD98's own MApi_AUTH_Process algorithm, plus the exact kernel payloads. If any
execution context ever calls Process with this pair, the checks pass legitimately and
the chain runs end-to-end (bitmap → verdicts → flags → SE).

NOT solved by this (unchanged blockers):
1. +0x503 single-run gate — CheckHashkey will not re-read the bitmap until
   MDrv_AUDIO_DspReboot re-arms it (destructive from a foreign module context; needs a
   serial console or boot-order control);
2. SE acceptance — proved to reject raw register injection (§13); it only trusts the
   kernel's own SET_IPAUTH_GROUP command path, which is (1)-gated.

### 14.4 Ready-to-fire test (needs explicit approval — two prior shutdowns)

R1-final: stop the audio HAL (kill mstar.hardware.audio.service) → upload the kit
(e1_upload) → insmod dspreboot1.ko (DspReboot re-arms +0x503, re-runs CheckHashkey with
the uploaded bitmap, ApplyHashkey pushes the flags to the SE, reloads the DSP — all from
the driver's own context with no active streams) → rmmod → restart the HAL → play DTS →
npcm + decoder + receiver. Risk: the same class as the two shutdowns, mitigated by the
quiesced HAL; recovery = power cycle (recovery USB stick exists).


---

## 15. ADDENDUM 9 (2026-09-28, night) — R1-final BLOCKED: vendor kernel module allowlist

### 15.1 What happened

Preparing R1-final: fresh donor (device `/vendor/lib/modules/utpa2k.ko`, md5 2fc6e9fc)
copied → init-surgery (reloc @0x20 → MDrv_AUDIO_DspReboot idx 39411, early return) →
push. insmod → **-ENOEXEC**. Bisect:
- exact v1 replica (name 'utpa2k' at this_module+0xC preserved): **"File exists"** —
  the loader processed the WHOLE module (layout, symbol resolution incl the DspReboot
  import, relocations, version checks) and only stopped at the name dedup;
- the same file with the name changed to 'trigger1' → **-ENOEXEC**;
- 'zutpa2kz' (substring probe) → -ENOEXEC; 'kheaders' (a loaded module's name!) →
  -ENOEXEC.

**Conclusion: the vendor kernel enforces an exact module-name allowlist**
(kallsyms: `module_names` rodata @0xc0d15178; kcore//dev/mem not readable to dump it).
Any module whose name is not on the list → -ENOEXEC, silently. Arbitrary modules cannot
load on this projector, period.

### 15.2 Re-interpretation of the two earlier shutdowns

dspreboot1/dsprearm2 (the two shutdown attempts) were ALSO ENOEXEC-blocked (the same
generator pipeline) — **the modules never loaded, DspReboot never executed**. The two
power-offs had a different cause (adb push of 25 MB + service churn are the suspects;
or an unrelated power glitch). DspReboot-from-module is therefore NOT proven dangerous —
it was never actually reached. The destructive reputation is retracted pending evidence.

### 15.3 State of the unlock chain (end of night)

```
data side          COMPLETE (gen_licence.py: valid arg0/arg1 + kernel payloads)
kernel upload      WORKS (ioctls accepted, bitmap in place)
CheckHashkey re-arm BLOCKED: only MDrv_AUDIO_DspReboot; module route closed by allowlist;
                   Monitor auto-recovery path unexplored (auto_recovery mdb command)
SE acceptance      UNTESTABLE until the flags actually flow through the kernel path
middleware chain   PRESENT but dormant (empty chip string — the data source is missing)
```

### 15.4 Options forward

1. **auto_recovery probe (cheap, low risk):** `echo auto_recovery=1 > /proc/utopia_mdb/audio`
   + a controlled DSP-idle condition — if the Monitor's stall detection fires,
   DspReboot re-arms +0x503 with our bitmap already in place.
2. **chip-string source hunt (static):** MI_SYS_GetConfigData in libmi_td98.so — find the
   config file path; if it exists on /mnt/vendor (writable), our computed blob can be
   served to the REAL middleware path.
3. **serial console** — for anything deeper.


---

## 16. ADDENDUM 10 (2026-09-28, night close) — session consolidation

### 16.1 Tonight's probes and results

| probe | result |
|---|---|
| E2′ direct SE injection (flags 0xFF + engine 0x97 via mdb) | **negative** — SE consumes/resets 0x112A82, gate closed |
| dump_spdif_npcm calibration | **valid**: working AC3P → ~1 MB captured; DTS → 0 B |
| auto_recovery=1 + DTS | no effect (no DSP stall detected) |
| dec_r2_cmd=1..6 sweep | no CheckHashkey trigger |
| licence kit generation | **DONE** (gen_licence.py, MD5 validated vs hashlib) |
| module loading | **BLOCKED** — vendor kernel allowlist (`module_names` @0xc0d15178) |

### 16.2 The one remaining wall

`AudioVars2+0x503` (CheckHashkey single-run). Every path to re-arm it:
- `MDrv_AUDIO_DspReboot` (kernel context) — the module route to reach it is closed by
  the name allowlist; **BUT**: `MDrv_AUDIO_Debug_Cmd_Write` (the 35 KB mdb command
  parser) references `AudioVars2+0x24C0` — the exact flag the Monitor's DspReboot path
  requires (`+0x24C0==1 && +9!=1`). **There is an mdb command that sets the DSP-reload
  request** — it is inside the 35 KB parser, not yet identified (dec_r2_cmd=1..6 did
  not trigger it; the parser needs a systematic command-table extraction).
- Middleware provisioning (MI_SYS_Init) — dormant on TD98 (empty chip string); the
  trigger is runtime-only (no static call site — confirmed by Muse across V440/V474/V098).

### 16.3 Next session priority

**Extract the mdb command table from `MDrv_AUDIO_Debug_Cmd_Write`** (utpa2k_stock.ko
@0x408B30, 0x8C28 B): map every accepted command name to its handler; identify the one
that writes `+0x24C0=1` (or otherwise requests a DSP reload). With that command:
```
e1_upload (bitmap+hash) → mdb command (reload request) → Monitor: DspReboot
→ +0x503=0 → CheckHashkey reads OUR bitmap → flags → SE armed
→ next DTS SetDecodeSystem → gate 0xB000001E==5 → passthrough
```
All shell-side, no modules, no kernel patches. Fallbacks unchanged: serial console,
V098 middleware diff, native-player APK hunt.

### 16.4 Artifacts (this session)

`e1_ipauth/`: gen_licence.py, kit_bitmap140.bin, kit_hashinfo18.bin, e1_upload,
e1_probe/e1_probe3, memread, trig*.ko (blocked), libdrvIPAUTH_td98.so, libmi_td98.so,
libutopia_device.so, auth_process_td98.txt, t615_dec_blob.bin, mega_nodes.json.
Device /data/local/tmp/: e1_upload, e1_probe, e1_probe3, memread, dltest*, trig.ko
(name='kheaders'), dsplog.txt. Nothing persistent modified; audio services restarted.


---

## 17. ADDENDUM 11 (2026-09-28, night) — the customer-data source identified; the wall mapped

### 17.1 The chip-string source: platform customer info

libmi_td98.so symbols reveal the data flow:
```
PT_GetCustomerInfo (DEF @0xd3c80)        ← platform layer (factory/NVM data)
u8Customer_info   (DEF @0x12ab80)        ← the 48-hex chip string buffer (global)
u8Customer_hash   (DEF @0x12abbc)        ← the 16-B expected digest (global)
MI_SYS_GetCustomerBondingInfo (DEF @0xa9014)
```
**Live read** (memread, cpu_audio PID 486, libmi base 0xB1630000):
`u8Customer_info` = **all zeros** (48 B), `u8Customer_hash` = **all zeros** (16 B).
The platform provides NO customer data on the projector → MI_SYS_Init exits before
the AUTH call → the chain stays dormant. The customer data is factory/NVM-provisioned
on TCL TVs — not present in any firmware image (tclconfig/tvconfig scanned clean).

### 17.2 The mdb +0x24C0 writer — located but not yet identified

`MDrv_AUDIO_Debug_Cmd_Write` (@0x408B30, 0x8C28 B — the 35 KB mdb command parser)
references `AudioVars2+0x24C0` — the Monitor's DspReboot-request flag
(`+0x24C0==1 && +9!=1` → DspReboot). The specific mdb command that sets it is inside
the parser, not yet identified (dec_r2_cmd=1..6 → no trigger). This is the last
shell-reachable re-arm candidate.

### 17.3 The complete map (final state of this phase)

```
DATA (offline, complete):    gen_licence.py → valid (arg0, arg1) + kernel payloads
UPLOAD (works):              e1_upload → kernel bitmap + gCusID/gCusHash
MIDDLEWARE (present, dumb):  cpu_audio runs; libmi+libdrvIPAUTH loaded;
                             customer info EMPTY → Process never called
KERNEL GATE (+0x503):        closed at boot; only DspReboot re-arms;
                             module route closed (name allowlist); SE rejects raw writes
TRIGGER (unidentified):      the mdb command writing +0x24C0=1 (in the 35 KB parser)
```

### 17.5 Next session

1. Parse MDrv_AUDIO_Debug_Cmd_Write's command dispatch → find the +0x24C0=1 command
   (static, bounded: 35 KB, one function).
2. Fire it via mdb → Monitor DspReboot → CheckHashkey re-runs with our uploaded bitmap
   → flags → SE → DTS test.
3. Fallbacks: V098 middleware diff (Muse), serial console.


---

## 17a. ADDENDUM 12 (2026-09-28, night final) — the trigger is ARMED; the last condition mapped

### 17a.1 auto_recovery write VERIFIED

The driver's own readback: `Auto recovery enable = 1` — AudioVars2+0x24C0 = 1 is SET and
persists in kernel RAM. The Monitor checks it EVERY healthy cycle
(clear-fail-counters → built-in string → +0x24C0 check).

### 17a.2 The second condition decoded: +9 = the DSP power state

`HAL_AUDIO_SetPowerOn` (@0x442A58, strb imm9) writes AudioVars2+9:
- `SetPowerOn(1)` at Initialize → +9 = 1 (DSP powered) → the Monitor's DspReboot
  condition (+0x24C0==1 && +9!=1) FAILS while the DSP is on;
- `SetPowerOn(0)` (ExitAudioSystem / audio suspend) → +9 = 0 → the NEXT Monitor cycle
  fires DspReboot → +0x503=0 → CheckHashkey reads OUR uploaded bitmap → ApplyHashkey →
  SE armed → DecoderLoadCode → the DSP re-inits fully provisioned.

**The system is now ARMED**: with +0x24C0=1 and the all-licensed bitmap uploaded, ANY
audio power-down event (system suspend that works, DSP idle power-down, or any future
natural DSP restart) triggers the full re-provision chain automatically. The DTS
passthrough unlocks the moment that happens — the data is in place and valid.

### 17a.3 The one missing event

A reliable +9=0 (audio power-down) without a full reboot. Candidates:
- system suspend that actually suspends (this box rebooted instead — needs testing of
  standby/suspend variants: `echo standby` vs `echo mem`, OSD power-off, HDMI-CEC off);
- the native player / middleware path on a TCL device (does it call ExitAudioSystem
  before re-provisioning? — the V098/V474 middleware diff would show);
- serial console (unlocks everything).

### 17a.4 Bottom line

Every element of the provisioning chain is now reverse-engineered, computed, uploaded,
or armed:
```
+0x24C0 = 1          ✓ set (auto_recovery=1, driver-verified)
bitmap (all-FF)      ✓ uploaded, in kernel RAM
gCusID/gCusHash      ✓ uploaded (0 + customer-consistent digest)
algorithm            ✓ fully reconstructed, offline-computable
waiting for          the single event: +9 → 0 (audio power-down)
```
The box must simply experience one audio power-cycle without losing RAM (no reboot).


---

## 17b. ADDENDUM 13 (2026-09-28) — NO-SERIAL PATH FOUND: raw /dev/mik!audio ioctls

### 17b.1 The node

The userspace framework crashed because it looks for `/dev/utopia` — which does not exist
on TD98. The real node: **`/dev/mik!audio`** (char 246:11).

### 17b.2 The full ioctl ABI (recovered from the userland UtopiaOpen/UtopiaIoctl)

```
fd = open("/dev/mik!audio", O_RDWR);
u32 open_par[3] = { module_id, r2, r3 };   // module_id = 0x34 (AUDIO) per UtopiaOpen
ioctl(fd, 0xC00C5501, open_par);           // create/bind the AUDIO instance on this fd
u32 disp_par[2] = { cmd, arg_ptr };
ioctl(fd, 0xC0085503, disp_par);           // dispatch one MApi_AUDIO_* command
```
A ~40-line ARM32 tool can therefore drive the MApi layer directly, bypassing the broken
userland framework — **no module, no console, no patch needed**. The vehicle is
`MApi_AUDIO_SetPowerOn(0)` or `MApi_AUDIO_ExitAudioSystem` → SetPowerOn(0) → AudioVars2+9=0
→ the Monitor's armed `+0x24C0==1` condition fires → DspReboot → CheckHashkey reads OUR
bitmap → ApplyHashkey → SE armed → DTS opens.

### 17b.3 Remaining step: the exact MApi command numbers

The dispatch table in utpa2k (AUDIOIoctl @0x3FFA44) has 239 entries (cmd 0x00-0xEE,
table base 0x3FFB40, u32 words, plus a second sub-table at 0x401064). The two known cmd
values from the mdb logs (SetCommAudioInfo=137, GetCommAudioInfo=145) anchor the inverse
map; the four target cmds (SetPowerOn / Initialize / ExitAudioSystem / Suspend) still need
resolving — a bounded lookup, no new unknowns.

### 17b.4 Correction: the mdb parser is KERNEL-BUILTIN

The `MApi_CMD_AUDIO_*` strings and the `[Audio][mapiAudio.c]` log format exist in NEITHER
utpa2k's .data nor .rodata — the mdb audio parser lives in the vmlinux (kallsyms:
`mdb_*_node_write`), which is exactly why the utpa2k string blob is dead. The user's
bootloop memory ("unused regions turned out to be live") is validated by this.

---

## 18. NO-SERIAL VEHICLE BUILT AND PROVEN (2026-09-28)

### 18.1 The chain is closed and unique (static proof)

Exhaustive per-function scan of `utpa2k_stock.ko` (every `STT_FUNC` symbol disassembled in its
own ARM/Thumb mode, not a linear sweep — a linear sweep desynchronises on every Thumb blob and
silently misses the audio code):

* `AudioVars2+0x503` — the "hashkey already evaluated" flag — is written in **exactly one place in
  the entire module**: `MDrv_AUDIO_DspReboot` @0x405AD8 (`strb r1,[r0,#0x503]` with r1=0).
  `MDrv_AUDIO_CheckHashkey` @0x4234C0 reads it as `ldrb r1,[r0,#0x503]; cmp r1,#1; beq <exit>`
  — i.e. **+0x503==1 means "already done, skip"**, the opposite of the earlier note. DspReboot
  therefore *re-arms* the check by clearing it; nothing else in the module can.
* `MDrv_AUDIO_DspReboot` has **exactly one caller**: `_MApi_Audio_Monitor` @0x3E32CC.
* The Monitor fires it under: `+0x24C0 == 1` (armed) `&& +9 != 1` (@0x3E31DC-0x3E31F4).
* `+0x24C0` is settable from a shell: `echo auto_recovery=1 > /proc/utopia_mdb/audio` (prints
  "Auto recovery enable = 1") — already proven.
* `+9` is written by only three functions: `MDrv_AUDIO_DspReboot`, `HAL_AUDIO_SetPowerOn`,
  `HAL_AUDSP_DspLoadCode`.

So the *only* way to re-run the licence chain without a reboot is: get `+9` out of the running
state, then let the armed Monitor fire.

### 18.2 The command number: cmd 1 == MApi_AUDIO_SetPowerOn

`AUDIOIoctl` @0x3FFA44 dispatches with `cmp r4,#0xee` / `add r0,pc,#4` @0x3FFB34 (→ r0 = 0x3FFB40,
verified) / `ldr pc,[r0,r4,lsl #2]`. Table entries are per-command stubs; each stub loads the
MApi function pointer out of the instance ops table that `AUDIOOpen` @0x3FEF74 fills from
relocation-tagged literal pairs.

* `table[1] = 0x3FFF80` → `ldrb r0,[r5]` (one byte from the payload) + `ldr r1,[r1,#4]` → `ops[+0x04]`.
* `AUDIOOpen` stores `_MApi_AUDIO_SetPowerOn` at `+0x04` (`str r1,[r0,#4]` @0x3FF024, value loaded
  by the movw/movt pair @0x3FF018/0x3FF01C whose relocations are both `_MApi_AUDIO_SetPowerOn`).

**Cross-validation** (this is what makes it trustworthy rather than a guess): the same inversion
maps the two command numbers independently observed in the mdb logs correctly —
`cmd 137 → ops+0x26c = SetCommAudioInfo`, `cmd 145 → ops+0x28c = GetCommAudioInfo`.

### 18.3 The node is /proc/utopia, not /dev/mik!audio

`/dev/mik!audio` (char 246:11) is the module's own device node with a legacy fops ABI — it
returns EINVAL for the utopia commands. The real protocol node is **`/proc/utopia`**: the live
`cpu_audio` process (pid 477) holds fds 14 and 15 on it. The kernel handler
`utopia_proc_ioctl` @0x13A90 confirms the ABI read from ground truth:

* `0xC00C5501` — copies **12 bytes**, `par[0]` = module id, valid range 0..0x52 (0x34 = AUDIO ✓)
* `0xC0085503` — copies **8 bytes** `{cmd, arg_ptr}`, then `blx` the module ioctl vtable
* `0xC0045504` — UtopiaClose, arg ignored

### 18.4 The tool

`e1_ipauth/pwoff.c` → ARM32 binary (`/data/local/tmp/pwoff`), NDK r30, 7 KB:

```c
fd = open("/proc/utopia", O_RDWR);
u32 op[3] = {0x34, 0, 0};  ioctl(fd, 0xC00C5501, op);          // "instance opened"
u8  payload[64] = {0};      payload[0] = power;                 // 0 = off, 1 = on
u32 disp[2] = {cmd, (u32)payload};
ioctl(fd, 0xC0085503, disp);                                    // rc = 0, accepted
```

### 18.5 Live result

```
e1_upload            -> both ipauth ioctls accepted
auto_recovery=1      -> "Auto recovery enable = 1"
pwoff run 1 0        -> rc=0   AND THE AUDIO REALLY WENT OFF
```

The user was watching YouTube and reported audibly hearing the audio cut out and stop. So the
vehicle is real end-to-end: raw `/proc/utopia` ioctl → `MApi_AUDIO_SetPowerOn` → `HAL_AUDIO_SetPowerOn`
→ the `AudioPowerOffTbl` register sequence → audio dead. The kernel then logged
`_MApi_AUDIO_SetAudioParam2() : Audio system is not ready yet`.

### 18.6 New machine-checkable oracle found

`echo show_all_decoder_status > /proc/utopia_mdb/audio` prints per-decoder blocks
(ID 0..4) with `Decoder format / Deocder hash key / play state / sample rate / channel /
frame count / Input source type`. Baseline captured at 4982 s. Also `dump_spdif_npcm=1 path=1`
captures the raw optical SPDIF stream, so a successful DTS passthrough is verifiable from a
file (IEC61937 sync 0x7FFE) and not only by ear.

### 18.7 Full mdb command table recovered

The 93-command table lives in `.data` as fixed-stride records (0x210 bytes apart, names at
offset 0) — `speaker_mute` @0x7B7B74 … `help` @0x7C3934. The `help` command dumps the full
reference into dmesg, including `dump_spdif_npcm=1 path=1`, `pcm_capture`, `show_all_decoder_status`,
`reg_bank/mask_value/reg_value/write_mask_reg`, and `read/write_dsp_sram_type` (direct PM/DM
DSP-SRAM access). **There is no power/reboot command in the set** — 15 candidate names were
probed and all came back `Unsupport Debug Commad`, which is what forced the ioctl route.

Checked and rejected: the DSP DM SRAM does *not* hold `AudioVars2` (all zeros at 0x000-0x210
and 0x500-0x508), so `write_dsp_sram_type` cannot be used to poke the flags directly.

### 18.8 Current state and the one remaining event

Audio is powered off, `auto_recovery=1` is armed, the licence kit is uploaded. The last thing
needed is a natural audio re-init (the HAL reopening the device), which runs
`HAL_AUDSP_DspLoadCode` → `+9` back to 0 → the armed Monitor fires `DspReboot` →
`CheckHashkey` re-runs the whole IPAUTH chain against **our** bitmap → `ApplyHashkey` →
`SET_IPAUTH_GROUP` → SE. A watcher (`/data/local/tmp/watch.sh`) is sampling the decoder state
every 2 s so the transition is captured.

---

## 19. INCIDENT — DD PASSTHROUGH BROKEN BY THE RAW-IOCTL VEHICLE, AND RECOVERED (2026-09-28)

### 19.1 What happened

The no-serial vehicle from §18 worked (the audio audibly cut out on `pwoff run 1 0`,
proving `/proc/utopia` → `MApi_AUDIO_SetPowerOn` → `HAL_AUDIO_SetPowerOn` end to end).
Three further dispatches were then run on a **second** utopia instance:

* `cmd 59` `MApi_AUDIO_ReleaseDecodeSystem` → `_MApi_AUDIO_ReleaseDecodeSystem() : Audio
  system is not ready yet` (the second instance was never properly initialised)
* `cmd 0`  `MApi_AUDIO_Initialize`
* `cmd 1`  `MApi_AUDIO_SetPowerOn(1)` (to undo the power-off)

plus, on **every** run of the tool, `ioctl(fd, 0xC0045504, 0)` = **`UtopiaClose` on the
AUDIO instance**. That close tears down module-wide state (`ExitAudioSystem` →
`SetPowerOn(0)` path).

**Result: the digital-out path never came back.** `audio_status` went from
`spdif[e30000,e30000]` to a persistent `spdif[000000,000000]`, the HAL refused calls with
`Audio system is not ready yet`, and the **user confirmed DD passthrough — which had worked
all session — stopped working.** `SetPowerOn(1)` alone could not recover it.

### 19.2 Recovery

Reboot. All of this state lives in kernel RAM (`AudioVars2`, `g_IpAuthVars`, the instance
bookkeeping), so a reboot is a complete and clean rollback — nothing persisted, nothing
patched, no firmware touched. `adb -P 5038 root` is needed afterwards because adbd comes
back as `uid=2000(shell)`.

Post-reboot the signature is healthy again: `audio_status` → `spdif[ffff00,ffff00]`, and the
**user confirmed DD passthrough works again.**

### 19.3 Operational rules that follow

* **NEVER issue `0xC0045504` (UtopiaClose) on an AUDIO instance.** The tool now leaves the
  instance open by default (`PWOFF_CLOSE=1` re-enables the old behaviour for reference).
* **Do not dispatch `Initialize` / `ReleaseDecodeSystem` / `SetDecodeSystem` on a second
  instance.** They report "Audio system is not ready yet" and leave the module degraded.
  A second instance is safe only for `SetPowerOn` (which has a fallback path).
* `Audio system is not ready yet` is an **instance-level** check, not the global audio state.
  It was misread earlier as the global state being down — that confusion cost time.

### 19.4 Health oracle found (use this instead)

```
echo audio_status > /proc/utopia_mdb/audio
  spdif[ffff00,ffff00]  -> digital-out ALIVE
  spdif[000000,000000]  -> digital-out DEAD
```
This is the cheap, unambiguous check to run before and after any audio poking.

### 19.5 Correction to §18.6 — `DigitalOutCodecCapability` is NOT the licence verdict

§18.6 proposed `ARC-DigitalOutCodecCapability:[...][DTS not support]` as the machine-readable
licence oracle. **That is wrong.** The same line reads `[DD not support]` while DD/AC3
passthrough is demonstrably working, so it reports the currently *negotiated* digital-out
capability (the live stream is PCM), not the IPAUTH verdict.

The only trustworthy oracle remains **`dump_spdif_npcm`** (≈1 MB during working AC3, 0 bytes
during DTS), which is file-based and was already calibrated in §13.

### 19.6 Other observations

* `spdif_mode` User value is `SPDIF_OUT_AUTO` in normal use, `SPDIF_OUT_BYPASS` after a
  reboot — it tracks the HAL's output routing, unrelated to licensing. `help` lists only the
  get form for it.
* `auto_recovery` reads back **1 immediately after boot**, so `+0x24C0` is set at boot and
  the Monitor's DspReboot condition effectively reduces to `+9 != 1`.

---

## 20. VERIFICATION PASS 2026-09-28 — what holds, and two things I had to retract

### 20.1 Runtime state (device verified reachable, adbd restored to root)

```
uptime 2:32 · uid=0(root) after `adb -P 5038 root` · cpu_audio ALIVE
Auto recovery enable = 1   ← armed at BOOT, not 0
```

The armed flag is **1 immediately after boot**, so the Monitor's condition reduces to
`+9 != 1`. Confirms the earlier observation.

### 20.2 RETRACTION — the `spdif[ffff00,ffff00]` health oracle is WRONG

§19.4 claimed `audio_status` shows `spdif[ffff00,ffff00]` when the digital-out is alive and
`spdif[000000,000000]` when it is dead. **Measured today: `spdif[000000,000000]` while the
user confirms DD passthrough is working.** Those counters track *stream activity*, not system
liveness (they were `000000` in an earlier working window too, and `e30000` only while a
stream was live). **There is no such cheap health oracle.** Use the user's ears, or
`dump_spdif_npcm`.

### 20.3 BREAKTHROUGH — the kernel logs every MApi command's number AND name

Setting **`debug_level=5`** (range 0~5) makes the kernel-builtin mdb audio layer
(`[Audio][mapiAudio.c]`, vmlinux) log the AUDIOIoctl dispatch verbosely:

```
<UTPA_DEBUG>[Utopia][[AUDIO][VERBOSE]]: [Audio][mapiAudio.c] cmd 145  switch
<UTPA_DEBUG>[Utopia][[AUDIO][VERBOSE]]: AUDIOIoctl - MApi_CMD_AUDIO_GetCommAudioInfo
<UTPA_DEBUG>[Utopia][[AUDIO][VERBOSE]]: [Audio][mapiAudio.c] cmd 145  Release
```

I had searched utpa2k for these strings earlier (§17b.4), found nothing, and concluded the
oracle was unreachable. **It was there all along — it is simply behind a log level.** This is
a third instance of the "dead data proved live" pattern the user predicted.

**Live traffic map** (counts since the level was raised):

| n | command |
|---|---|
| 230 | `GetCommAudioInfo` (cmd 145) |
| 139 | `SetCommAudioInfo` (cmd 137) |
| 119 | `GetAudioInfo2` |
|  20 | `MM2_CheckAesInfo` |
|  10 | `MM2_InputAesFinished` |
|  10 | `GetDSPBaseAddr` |
|   3 | `SPDIF_Monitor` |
|   2 | `Monitor` |
|   2 | `HDMI_Tx_GetStatus` |

**This live-calibrates the static inversion**: the kernel itself pairs `cmd 137` with
`SetCommAudioInfo` and `cmd 145` with `GetCommAudioInfo` — the exact two anchors the
`AUDIOIoctl` table inversion was validated against. The numbering is now confirmed
observationally, not only statically. `MApi_CMD_AUDIO_Monitor` in the traffic independently
confirms the Monitor thread is alive and being called.

Note the logging is attached to the **kernel mdb path**, not to raw `/proc/utopia` ioctls — a
raw dispatch produced no log line. So it cannot name our `pwoff` commands directly, but it
*is* a live map of what the running system actually does, which is new.

### 20.4 `module_debug_level` is a DEAD command

It is present in the 93-entry table, listed in `help` with "PARAM Range(0~6)", and returns
`Unsupport Debug Commad (module_debug_level) !!`. Same phenomenon as the dead mdb strings
the user called out. `debug_level` (0~5) is the working one.

### 20.5 RETRACTION — the `+9` live readout did NOT materialise

§19 promised a self-verifying test: with the log level up, the Monitor would print `+9` as a
bare `%ld` line every 100 ms, and the stream would stop the moment `+9` hit 0.

**Measured: zero bare-number lines, and zero Monitor log output at `debug_level=5`.**
The `.L.str.19 = "%ld"` printk sits behind `UtopiaLogSystem(.L.str.18)` and is not reached at
level 5; `module_debug_level` — the one that might raise it — is unsupported.

**Consequence: the trigger test is NOT self-verifying as designed.** Without a live `+9`
readout we cannot tell, from logs alone, whether the `+9 != 1` condition was met. The test
still has value (the npcm result is decisive), but it must be read as a binary outcome, not
as a step-by-step diagnosis.

### 20.6 Still standing (re-verified or untouched this pass)

* DTS licence = IPAUTH bits **7, 15, 18, 58**; AC3 = 14 bits; **same `IPCheck` channel** —
  there is no separate Dolby licence channel (earlier claim retracted).
* `AudioVars2+0x503` has exactly one writer module-wide (`MDrv_AUDIO_DspReboot` @0x405AD8);
  checked for `strb`, `strh` and `str`.
* `MDrv_AUDIO_DspReboot` has exactly one caller (`_MApi_Audio_Monitor` @0x3E32CC).
* Monitor gate: `+0x24C0==1 && +9!=1`, **plus a newly found second gate**,
  `AudioVars2+0x2098 != 1` — that byte is set by `HAL_AUDSP_DspVerifySegmentCode` while a DSP
  code-segment verification is in flight; while it is 1 the Monitor skips its whole body.
* `cmd 1 == MApi_AUDIO_SetPowerOn` — double-verified statically and now anchor-calibrated
  live via §20.3.
* The utopia node is **`/proc/utopia`**, not `/dev/mik!audio` (the latter returns EINVAL).

---

## 21. AUTOMATED BASELINE RUN 2026-09-28 — the test harness had a routing flaw

### 21.1 What ran

Pushed `test_ac3_51.mp4` (5 683 543 B) and `test_dts_51.mp4` (7 545 794 B) to
`/data/local/tmp/`, drove playback with `am start -a android.intent.action.VIEW -d file://… -t
video/mp4`, captured the full kernel stream with `debug_level=5` (`capstart/capstop` →
14.3 MB / 151 584 lines for the DTS run), and ran `dump_spdif_npcm=1 path=1` around each.

**User-observed result: AC3/DD — sound present. DTS — no sound.** That matches every prior
session and is recorded as the baseline.

### 21.2 The npcm dump DOES work — my path was wrong

Earlier I reported "no file". Wrong path on my part: the dump lands in
**`/data/DUMP_audio_spdifNpcm_NN.bin`**, not in `/data/local/tmp/`.

```
DUMP_audio_spdifNpcm_00.bin  1 284 096 B  (today 23:36)
DUMP_audio_spdifNpcm_01.bin  1 824 768 B  (today 23:37)
DUMP_audio_spdifNpcm_03.bin          0 B  (2026-09-28 02:16, historical DTS)
DUMP_audio_spdifNpcm_04.bin          0 B  (2026-09-28 03:09, historical DTS)
```

So the capture scales with playback and the historical DTS runs really are 0 bytes.

### 21.3 CRITICAL — neither of today's captures contains a single IEC61937 sync word

```
npcm_00: 0x7FFE hits = 0     head: 72 f8 1f 4e 01 00 00 30 77 0b 37 …
npcm_01: 0x7FFE hits = 0     head: 00 00 00 00 …
```

**No non-PCM framing at all.** Therefore the audio in this run did **not** traverse the optical
S/PDIF transmitter — it went another way (HDMI / internal). Consequences:

* the AC3 "sound works" observation is **not** evidence of SPDIF passthrough;
* the DTS run is **not** a valid passthrough measurement;
* every conclusion I was about to draw from "DTS npcm = 0 bytes" would have been unsound.

**The npcm oracle is only meaningful when the audio is actually routed to the optical output.**
That routing is a configuration fact I cannot infer from the shell, and it gates the whole
test design.

### 21.4 Other observations

* `spdif_mode` readback: `[User:SPDIF_OUT_BYPASS] [Driver:SPDIF_OUT_PCM]`. The user layer is
  already in Bypass; the driver line tracks the live stream, not a setting.
* `spdif_mode` has **no SET form in `help`** (get only) — consistent with the other partial
  entries in the 93-command table.
* **The device rebooted during this session** (uptime was 2:32, then 6 min). Cause not
  established; possibly the `am start` media session. All on-device state (bitmap, arming)
  was lost as a result — which is the expected behaviour, not a new failure.

---

## 22. THE TRIGGER IS PROVABLY UNREACHABLE — and the boot race is the only window (2026-09-29)

### 22.1 `UtopiaSetIPAUTH` fully recovered — and the "better mechanism" was a false lead

`libutopia.so` (the device copy at `/vendor/lib/libutopia.so`, 19 353 764 B, is byte-identical
to the local `libutopia_device.so`). Symbol `UtopiaSetIPAUTH` @0x50ba28, size 0x178, **ARM
mode** — the address is even, so there is no Thumb alignment problem; a linear Thumb sweep was
the wrong reading. Full body:

```
int UtopiaSetIPAUTH(void *pBitmap140, u16 *pCusID, void *pCusHash16)
  open("/proc/utopia", O_RDWR)                       ; r4 = fd
  buf = malloc(0x8c); memcpy(buf, pBitmap140, 0x8c) ; verbatim 140 B
  ioctl(fd, 0xC08C5506, buf)
  h = malloc(0x12); *(u16*)h = *pCusID; memcpy(h+2, pCusHash16, 16)
  ioctl(fd, 0xC0125507, h)
  free/close; return 0
```

**There is no hidden auth flag in the uploader** — it lives in `MApi_AUTH_Process`, the
missing caller. So our hand-written `e1_upload` is functionally identical; the vendor
function was never a better mechanism. (An earlier claim that it "also sets an auth flag"
was wrong and is retracted.)

Built `e1_ipauth/vendup.c` → `dlopen("/vendor/lib/libutopia.so")` + `dlsym("UtopiaSetIPAUTH")`,
and **ran the vendor's own provisioning on the device: returned 0**, `gCusHash =
8b8fc17688e820323cb640f0ee533a9b` (matches our generated kit).

### 22.2 Path A (the Monitor trigger) — FAILED, and now explained

Sequence run: `auto_recovery=1` armed → vendor provisioning → user released the audio device →
15 s. Result: **DTS npcm = 0 B** (AC3 control the same morning = 2 113 536 B), and the capture
(28.6 MB / 306 281 lines) contains **no** reboot, re-init or licence line.

### 22.3 What `+0x503` really depends on

`HAL_AUDIO_GET_INIT_FLAG` is one byte:

```
0x4411ac: movw r0, #0 ; g_AudioVars2
0x4411b4: ldr  r0, [r0]
0x4411bc: ldrbne r0, [r0, #8]        ; returns AudioVars2[+8]
```

and CheckHashkey's tail:

```
0x4248a8: bl   HAL_AUDIO_GET_INIT_FLAG
0x4248ac: cmp  r0, #1
0x4248b4: moveq r1, #1
0x4248b8: strbeq r1, [r0, #0x503]    ; +0x503=1 ONLY IF +8==1
```

So `+8` is the "audio system ready" byte, and the run-once gate closes only while the system
is ready.

### 22.4 `SetPowerOn(0)` is a ONE-WAY DOOR (third audio incident — user-confirmed DD loss)

Proven by direct experiment:

```
SetPowerOn(0)  → +8=0, +9=1, optical path dead (AC3 npcm = 0 B)
SetPowerOn(1)  → does NOT restore +8 or +9; audio stays dead
cmd 0 MApi_AUDIO_Initialize (separate instance) → does NOT restore either (AC3 npcm = 0 B)
reboot         → the only recovery
```

I ran this on a system already known to be unrecoverable in-session (§19) and broke DD a
second time. The user lost audio again; recovered by reboot. **No further state-mutating
commands without a verified in-session recovery.**

### 22.5 The structural conclusion — the host path is closed

```
+0x503 cleared by : MDrv_AUDIO_DspReboot        (the only writer, module-wide)
DspReboot called by: _MApi_Audio_Monitor        (the only caller)
Monitor fires when: +0x24C0==1 && +9!=1 && +0x2098!=1
+9 = 0 written by  : DspReboot, or HAL_AUDSP_DspLoadCode(r6=0)
```

`+9 = 0` therefore comes only from `DspReboot` (circular) or from `DspLoadCode(0)`, which runs
only during a genuine DSP failure (we saw `HAL_DEC_R2_SafeResetR2() : u8DecR2WhileCnt1:58af`
fire once naturally). **Any reachable trigger state is also a state with no audio, and the
only exit — reboot — destroys the upload.** Not "hard": closed.

### 22.6 The one real window: the boot race

At boot, before the HAL's first `CheckHashkey`, **both** `+0x503 == 0` and `+9 == 0` hold. The
first CheckHashkey consumes the gate and closes it. That is exactly the window the earlier
boot-race attempt lost (§11): early HAL audio activity runs before network adbd (~31.5 s).

So the only viable host-side approach is to **land the bitmap inside that window**, which
means a boot-time service that races the audio HAL — not an adb-time injection. The crypto is
already reconstructed offline, so such a service needs no secret: it computes (arg0, arg1),
opens `/proc/utopia`, writes the two payloads. This is the "permanent fix" shape discussed
earlier, and it now has a precise purpose: **win the boot race, survive reboots.**
