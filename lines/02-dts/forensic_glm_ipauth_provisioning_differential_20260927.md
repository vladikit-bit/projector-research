# GLM — the IPAUTH provisioning chain (TD98 ↔ TCL T615 differential)

**Date:** 2026-09-27
**Mode:** static/corpus-only. No device access, no patches, no runtime writes.
**Authority rule:** every TD98 claim below is byte-verified in `patch_baseline/utpa2k_stock.ko`
(MD5 `2fc6e9fc46b6402d73aa6f84f9599d46`) or `spdif_audio_investigation/libs/libutopia.so`.
Every T615 claim is verified in the extracted artifacts (`tclaudio/`, carved `ipauth_lib.so`,
carved `mi_union.so` from `uni_out/tvservice_a.bin`).

---

## 1. Executive summary

The DTS licence verdict chain is now closed end-to-end, and it is **not a factory blob**:

1. `MDrv_AUTH_IPCheck(ipid)` (kernel, `utpa2k_stock.ko` @.text 0x1390C) returns **bit `ipid`
   of a 140-byte table** pointed to by `*gIpAuthVars` — header byte [9] = bitmap size
   (must be 0x10..0x80), bits stored **reversed** (bit N lives at byte `+0xA + size-1-(N>>3)`,
   mask `1<<(N&7)`).
2. That table is **uploaded from userspace** through `/proc/utopia`:
   - `ioctl(fd, 0xC08C5506, buf140)` → kernel `utopia_proc_ioctl` @0x13D00 allocates
     `kmem_cache_alloc_trace(0x8C)` and `memcpy`s the user buffer into `*gIpAuthVars`
     (file off 0x13D58: `movw/movt r4,=gIpAuthVars` → `str r0,[r4]`). **Zero validation.**
   - `ioctl(fd, 0xC0125507, buf18)` → writes `gCusID` (u16 @.data 0x2B214) and
     `gCusHash[16]` (@0x2B218), read back by `MDrv_AUTH_GetHashInfo`.
3. If the upload never happens (`*gIpAuthVars == NULL`), `MDrv_AUTH_IPCheck` **logs and
   returns 0 for every IP** (fallback path @0x1395C: `UtopiaLogSystem` + rate-limited print,
   return 0). This is the TD98 steady state — consistent with the runtime finding
   "MApi_AUTH_Process never called by any binary".
4. `MDrv_AUDIO_CheckHashkey` (~90 IPCheck calls) aggregates verdicts → `r8` → `AV+0x4D8`
   → engine byte `0x112E9A` (0x97 only if the DTS IP set passes).
   `MDrv_AUDIO_SET_IPAUTH_GROUP` (@0x424BD8) sends the same aggregated group flags to the
   Secure Engine (`0x112A82..85`, cmd 0x83/0x84) — and it is invoked from
   `MDrv_AUDIO_SetDecodeSystem` / `SetSystem` / `SetAudioParam2` / `ApplyHashkey`,
   i.e. **on every decode start**.
5. The DSP gate `*(u16*)0xB000001E == 5` (four sites incl. DTS branch 0x21B19) therefore
   reads an SE state that is **armed from the userspace-uploaded bitmap**. Today it never
   arms because nobody uploads.

**The user's hypothesis is confirmed at both ends:** the licence engine and the customer
licence data are present in the TD98 firmware — the activation call is simply never made.

---

## 2. Kernel side (TD98 `utpa2k_stock.ko`, ARM32, ET_REL)

### 2.1 MDrv_AUTH_IPCheck @.text 0x1390C (0xD8 B) — exported

```
r1 = *gIpAuthVars                ; NULL → print + return 0
r2 = tbl[9]                      ; size
size-0x10 <u 0x71 ? bitmap path  ; size ∈ [0x10, 0x80]
  idx = size - (ipid>>3) - 1     ; reversed byte order
  idx < 0 → fail path
  bit = tbl[0xA + idx] >> (ipid&7) & 1
size out of range → ___ratelimit + fail
```
Data relocations: `gIpAuthVars` @0x13910/14; `gCusHash` @0x139FC/A000 (GetHashInfo only).

### 2.2 MDrv_AUTH_GetHashInfo @0x139E4 — exported
Copies 18 bytes out: 2 bytes from `gCusID` + 16 bytes from `gCusHash`.

### 2.3 utopia_proc_ioctl @0x13A90 (0x1F68 B) — the upload handler
Dispatch (signed): 0xC0045504→UtopiaClose; 0xC0085503→sub-dispatch; 0xC00C5501→main;
0xC0125507→**site B**; 0xC0145505→pool mapping; 0xC08C5506→**site A**; else -ENOTTY.

- **Site A (0x13D00):** `copy_from_user(sp, arg, 0x8C)`; if `*gIpAuthVars==NULL`
  `kmem_cache_alloc_trace(kmalloc_caches[1], GFP, 0x8C)`; `str r0,[r4]` (r4=&gIpAuthVars);
  `memcpy(*gIpAuthVars, sp, 0x8C)`.
- **Site B (0x13E38):** `copy_from_user(sp, arg, 0x12)`; `strh [sp+0] → gCusID`;
  `stm gCusHash, {sp+2..sp+0x11}` (16 bytes).

`UtopiaSetIPAUTH` kernel symbol @0x6224 is an 8-byte stub (`mov r0,#0; bx lr`) — the real
upload path is the proc ioctl, not this export.

---

## 3. Userspace side — the engine is present in BOTH firmwares

### 3.1 T615 (TCL 2022 Google TV, platform T615T03)

Two binaries inside `tvservice_a`:

- **IPAUTH provider lib** (carved → `opencode/ipauth_lib.so`, 0x8118 B, source `drvIPAUTH.c`,
  built with gcc-arm-linux-gnueabi 5.5.0). Exports the full family:
  `MApi_AUTH_Process` (0xCAC), `MDrv_AUTH_IPCheck`, `MDrv_AUTH_IPVCheck`,
  `MDrv_AUTH_AES_Set_Key/Encrypt/Decrypt/Gen_Tables`, `MDrv_AUTH_MD5Transform/Update`,
  `MDrv_AUTH_GetHashKeyData`, `MApi_AUTH_State`, … Log strings:
  `[Utopia][IPAUTH]: Wrong hash key`, `MDrv_SYS_GetChipID:%x`, `Wrong Chip ID`,
  `[Auth OK]`, `[Auth NG]`, `[NFXPTK CHK NG]`.
- **`libutopia.so` copy**: `UtopiaSetIPAUTH` @0x8BACE8 (0x178 B):
  `open("/proc/utopia", O_RDWR)` → `malloc(0x8C)`+`memcpy` → `ioctl(0xC08C5506)` →
  `malloc(0x12)` + pack `{u16 [r7], 16B [r8]}` → `ioctl(0xC0125507)`.

**`MApi_AUTH_PROCESS(arg0, arg1)` semantics (static read):**
arg0 = NUL-terminated hex-char string (≤0x30 chars; `((len-0x10)>>1)&0xFF` becomes the
kernel bitmap size field); arg1 = pointer to 16 expected bytes.
Flow: `MDrv_AUTH_InitialVars` → zero 0x111 local → `*g_IpAuthVars` header init
([8]=0, [9]=size), bitmap at +0xA cleared (0x80) → chip checks
(`MDrv_SYS_GetChipID`/`GetChipType`, fail → "Wrong Chip ID") → assemble 4 words from
`KeyCustomer` (16 B, .data) → keyed-MD5 (`MD5Update` with customer-key IV) over arg0 →
`MDrv_AUTH_Encode` → byte-compare against arg1[0..15] (mismatch → "Wrong hash key" path)
→ copy arg1 → `gCusHash` (16 B, .bss 0x17624), derive `gCusID` (0x17634) → assemble
`IPControl`/bitmap bytes (loop copying the same bytes into `IPControl`[0..] and into
`(*g_IpAuthVars)+9`…) → success/fail paths with `IPAuthState`.

- **Caller found:** `mi_union.so` (carved from tvservice_a @0x4379000, 0x39C198 B;
  jenkins path `9615_AmatiGB/Android11_alpha/vendor/media`; imports `MApi_AUTH_Process`).
  Exactly **one** call site: `MI_SYS_Init` @0xAB32C
  (`bl 0x16754` → PLT → GOT 0x110B5C), args:
  `r0 = &gChipHexStr` (global 0x11BA08; 48-byte buffer, `0`-chars `'0'` padded at [0xC..0xF],
  validated by a 48-iteration hex loop) and `r1 = ctx->[0x4C]` (pointer to the 16 expected
  bytes). After OK it republishes 33+16 bytes to globals.

### 3.2 TD98 (projector)

`spdif_audio_investigation/libs/libutopia.so` (ARM32, device-extracted) contains the
**same engine**: `MApi_AUTH_Process` (0xC58 — same size), `UtopiaSetIPAUTH` @0x50BA28
(0x178 — **instruction-equivalent**: same `open("/proc/utopia", O_RDWR)` string resolved,
same `malloc(0x8C)`/`memcpy`/`ioctl 0xC08C5506`, same `malloc(0x12)`/`ioctl 0xC0125507`),
same IPAUTH log strings (`[IPAUTH][Auth NG]`, `[IPAUTH][NFXPTK CHK NG]`, `%04x%04x%04x%04x`).
Function bodies differ byte-wise (different build), semantics identical.

**KeyCustomerList is byte-identical between TD98 and T615** (@0x10D5AC0 in TD98 .data,
@0x16098 in T615; 0x48 B = 3 entries `{u32 id; u8 hash[16]; u32 pad}`):

| id | hash (16 B) |
|----|-------------|
| 0x00000000 | `28 5b f7 c6 94 cf 2c 6b 8b 67 ff da e8 c0 18 45` |
| 0x000000E8 | `ba fc f0 9c 94 d7 12 64 4f 20 e1 32 68 fa 7b 1b` |
| 0x000000F4 | `98 ad bc ac 04 f9 8e 96 71 c1 cf 31 70 ac c1 c6` |

T615 `KeyCustomer` (active entry, .data 0x16158) = entry id 0. TD98 does not export the
symbol but the data block is the same.

**Kernel proc node:** standalone `utopia\0` name string present (file off 0x105B82F) plus
`utopia_proc_open/ioctl/write/release` symbols — the `/proc/utopia` node the middleware
opens is registered by `utpa2k.ko`.

---

## 4. Why DTS is locked today (final causal model)

```
TD98: no caller of MApi_AUTH_Process / UtopiaSetIPAUTH
  → *gIpAuthVars == NULL in kernel
  → every MDrv_AUTH_IPCheck returns 0 (log + fail)
  → CheckHashkey verdicts degrade (r8 ≠ 3 for DTS) → AV+0x4D8 ≠ 3
     → HAL_AUDIO_SetSystem2 writes engine byte 0x112E9A = 4 (not 0x97)
  → SET_IPAUTH_GROUP sends all-zero group flags to SE (on every SetDecodeSystem)
  → SE never reports SPDIF-licensed → *(u16*)0xB000001E ≠ 5
  → DEC DTS branch @0x21B19 silently skips (counter clear), no IEC61937 bursts
```
This **unifies** all previous observations: seven failed ARM patches (wrong side), the
frozen `auto/not-specified` HAL axes, hash UNSUPPORTED in mdb, and the silent DSP gate.

---

## 5. The no-patch activation experiment (E1) — REQUIRES USER APPROVAL

Root shell tool (same pattern as the existing private-API probe tool), pure upload,
reversible by reboot (kernel buffer is RAM, re-uploaded state only):

```c
fd = open("/proc/utopia", O_RDWR);
u8 bm[140]; memset(bm,0,140);
bm[9] = 0x80;                       // size field (kernel requires 0x10..0x80)
memset(bm+0x0A, 0xFF, 128);         // all 1024 IP bits set
ioctl(fd, 0xC08C5506, bm);          // upload bitmap
u8 hi[18] = { ID lo, ID hi,         // gCusID = 0x0000
  0x28,0x5b,0xf7,0xc6,0x94,0xcf,0x2c,0x6b,
  0x8b,0x67,0xff,0xda,0xe8,0xc0,0x18,0x45 };  // KeyCustomer entry 0
ioctl(fd, 0xC0125507, hi);          // upload cusID+hash
```
Then start DTS playback (Kodi, as before) and observe:
1. `MDrv_AUTH_IPCheck` verdicts flip (bitmap bits) → engine byte `0x112E9A` should become
   0x97 (needs IP-7 DTS:X pass) — readable via existing mdb/reg tools;
2. `SET_IPAUTH_GROUP` now carries non-zero DTS flags on the next SetDecodeSystem;
3. the DEC gate: absence of the silent-skip, presence of IEC61937 payload in
   `dump_spdif_npcm`, and finally physical SPDIF lock on the Pioneer.

**Open risk (this is what E1 tests):** whether the SE cross-validates the uploaded bitmap
against OTP/fuse state. If it does, E1 fails silently and the next step is E2 — calling the
*real* `MApi_AUTH_Process` in TD98's own `libutopia.so` via `dlopen` (needs arg0 = 48-char
device hex string, arg1 = expected 16 B hash; on T615 the caller gets both from
`MI_SYS_Init` context — the ctx->[0x4C] source is the follow-up trace).

---

## 6. Follow-ups

1. E1 on device (needs approval).
2. Trace `mi_union.so` ctx->[0x4C] (arg1 source) and the 48-char string construction, if E2
   becomes necessary.
3. T615 DEC/SND DSP blobs: check whether the same `0xB000001E==5` gate exists on a
   DTS-working platform (differential for the DEC-patch fallback route B5).
4. `MDrv_AUTH_GetHashKeyData` (0xC8 B) read — returns KeyCustomerList entries; cheap to map
   fully.

*No binaries were modified. Carved artifacts (`ipauth_lib.so`, `mi_union.so`) are new
read-only extracts in `C:\Users\k0994\AppData\Local\Temp\opencode\`.*
