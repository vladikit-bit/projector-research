# 23. BOOT-TIME PROVISIONING — WORKS, AND IT DOES NOT OPEN DTS (2026-09-29)

The hypothesis under test: DTS is blocked because the IPAUTH IP bitmap is never provisioned
on this projector, and if we land it before the HAL's first `CheckHashkey` it should open.

**Result: the hypothesis is falsified.** The provisioning is now provably correct and
provably early, and DTS is still blocked.

## 23.1 The window, measured

Boot timeline from the current dmesg (free, non-destructive):

```
 5.78 s   our oneshot service starts
 6.32 s   /proc/utopia appears  (service waits for it, up to 10 s)
 6.59 s   vendor UtopiaSetIPAUTH returns 0     ← provisioning complete
13.03 s   first audio subsystem activity (_MApi_AudioAmp_ExtendCmd)
~43 s     adbd
```

Ordering is **deterministic, not a race**: `cpu-audio` is `service … class main / oneshot`,
Android init starts `oneshot` services in a class serially, and
`/vendor/etc/init/aaa_dtsorder.rc` parses before `cpu_audio.rc` alphabetically. The service
runs in `seclabel u:r:mt_cpu_audio-svc:s0`, the audio HAL's own context.

So our provisioning lands **6.4 s before the first audio activity** — the earliest point
reachable from userspace, using the vendor's own function.

## 23.2 The service

`/vendor/bin/dtsorder.sh` (oneshot, class main, seclabel `mt_cpu_audio-svc`) waits for
`/proc/utopia`, runs `/vendor/bin/dtskit` (dlopen of `/vendor/lib/libutopia.so` →
`dlsym("UtopiaSetIPAUTH")`), and reports via `setprop dtsorder.{startup,utopia,rc,done}`.

Two instrumentation lessons, both hard-won:
* **`/dev/kmsg` is not writable from the service context** — the first attempt produced no
  output at all. `setprop` works and is the readable channel.
* **The failed `> /vendor/etc/dts/last.log` redirect returned `rc=1` with empty output**,
  which looked like the tool failing. The tool was fine all along. Capture output into a
  shell variable and `setprop` it instead of redirecting.

## 23.3 Result

```
DTS npcm = 0 B
AC3 npcm = 1 843 200 B     (control — device healthy, audio path fine)
```

DTS remains completely blocked, with the bitmap provisioned by the vendor's own code, at the
earliest reachable moment, 6.4 s before the first audio call.

## 23.4 The table is not being wiped

`gIpAuthVars` is allocated by the `0xC08C5506` handler with `kmem_cache_alloc_trace` only
when NULL, and `kmem_cache_free` **does not appear anywhere in the module**. The only other
ipauth ioctl, `0xC0145505`, ends in `MsOS_MPool_Mapping_Dynamic` — a mapping call, not a
clear. So nothing frees or resets the table between our write and the first `CheckHashkey`.

## 23.5 What this means

**Provisioning is not the blocker.** We had assumed the projector simply never feeds the
IPAUTH table; it now provably does not matter whether it does, because when *we* feed it —
with the vendor's own code, at the earliest possible instant — DTS stays blocked.

The remaining candidates, in order:

1. **The SE refuses the resulting verdict.** `E2′` already showed the SE validates rather
   than stores. Feeding correct flags is not the same as the SE accepting them; it may
   cross-check against something we do not know how to influence.
2. **The `gCusHash` / `gCusID` path matters after all** — Muse's relocation-based proof that
   it feeds only the CEC path is strong, but it is proof about the *ARM* side only. The
   firmware/SE side is not covered by any static analysis we have done.
3. **DTS is gated somewhere else entirely** and the IPAUTH/SE path is a red herring — the
   `*(u16*)0xB000001E == 5` read in the DEC firmware may be satisfied by a completely
   different producer than the one we have been attacking.

**Practical consequence:** the provisioning work is finished and correct; it is simply not
the lever. Further effort belongs on the SE/verdict side, where we currently have no
readable signal and no write path that the SE accepts.


## 24. THE DTS GATE HAS A SECOND GRANT PATH — via 0xB000001D, bypassing the SE entirely

Found by reading the DEC firmware disassembly (`aeon_validate/dec_work/dec32_clean.txt`, the
authoritative 32-bit listing of `dec_full.bin`), not from any earlier note.

### 24.1 The gate is not `verdict == 5`

DTS gate, 0x21AA4:

```
021aa4  movhi r24, 0xb000
021aa8  ori   r25, r24, 0x1e     ; r25 = 0xB000001E
021aab  lh    r26, 0(r25)         ; r26 = *(u16*)0xB000001E   ← "the verdict"
021ac3  beqi  r26, 0x5, 0x00021ad9      ; ==5              → PASS
021ac7  ori   r24, r24, 0x1d      ; else r24 = 0xB000001D
021aca  lbz   r23, 0(r24)         ; r23 = *(u8*)0xB000001D
021acd  andi  r23, r23, 0x7f
021ad0  beqi  r23, 0x0, 0x00021ad9      ; (byte & 0x7F)==0  → ALSO PASS
```

All four licence gates share this shape; gates 2 and 4 use a **signed** load instead:

| gate | second grant condition |
|---|---|
| 0x21AC3 (DTS) | `(*(u8*)0xB000001D & 0x7F) == 0` |
| 0x21B19 (DTS) | `(int8)0xB000001D <= -1` |
| 0x21F0E (AC3) | `(*(u8*)0xB000001D & 0x7F) == 0` |
| 0x21F5C (AC3) | `(int8)0xB000001D <= -1` |

### 24.2 The two conditions are satisfied by the same byte

**`0x80` passes all four:**
* `0x80 & 0x7F == 0` → gates 1 and 3 pass
* `(int8)0x80 == -128 <= -1` → gates 2 and 4 pass

`0x00` passes gates 1/3 but not 2/4. `0xFF` passes 2/4 but not 1/3. **`0x80` is the unique
single-byte value that satisfies all four.**

### 24.3 Likely one 32-bit register

`0xB000001D` (byte) and `0xB000001E` (halfword) are adjacent. If a 32-bit register lives at
`0xB000001C`, then in little-endian: byte `0x1D` = `R[15:8]`, halfword at `0x1E` = `R[31:16]`.
The gate is therefore `(R >> 16) == 5  ||  ((R >> 8) & 0x7F) == 0` — i.e. setting the *upper
byte of the low halfword* to 0x80 grants licence without the SE verdict being 5.

### 24.4 On the failure path

Failing the gate does not abort: execution falls through with `r27 = byte & 0x7F`, later used
as a multiplier at 0x21AF1 (`muls r24,r27,r16`), which zeroes the product and sends the
`bles` comparison the other way. So the gate is a **functional degradation**, and 0x80 makes
the downstream arithmetic identical to the pass path.

### 24.5 Why this matters, and what is still unknown

Everything we did in §19–§23 pushed on the *producer* of `*(u16*)0xB000001E` (feed the bitmap,
win the boot race, call the vendor's own uploader). This gate says the **consumer** accepts a
second, purely local grant condition that never consults the SE.

**Still unknown, and it is exactly the next task:** the host-side address of this register.
The firmware sees it at `0xB000001D` (DSP address space). The ARM side writes audio registers
through a different window — `MDrv_AUDIO_SET_IPAUTH_GROUP` writes `0x112A82–85`, and the
earlier note that ARM `0x112A82` is *not* mdb bank `0x112A` means a translation exists. So:

1. Find the host-side register that maps to DSP `0xB000001C..1F`.
   Candidates: the audio mailbox / DEC-R2 window that the ARM HAL already writes.
2. If found, test a single-byte write of 0x80 and re-run the DTS npcm test.

This is a strictly new attack surface — nothing in §19–§23 touched it, and it explains why
a correctly provisioned bitmap still did not open DTS.

## 25. RESEARCH PASS: who produces the verdict? (no writes performed)

Requested scope: (1) find the host-side address that maps to DSP `0xB000001C..1F`,
(2) determine whether the SE keeps its own copy of the 140-byte IP table. No device writes.

### 25.1 The DSP never sees the IP table — that question is answered

Across the whole DEC firmware disassembly there is **no** reference to a 140-byte IP table
and no other licensing store. The firmware's entire view of the licence is **one 32-bit
register at `0xB000001C`**: halfword `0x1E` (the "verdict") and byte `0x1D`.

Full chain, therefore:

```
140-B bitmap ──► MDrv_AUTH_IPCheck (4 bits: 7,15,18,58)  [kernel, module]
              ──► host flags ──► SET_IPAUTH_GROUP (0x112A82-85, cmd 0x83/0x84)
              ──► SE ──► ONE 32-bit register at DSP 0xB000001C
              ──► firmware gate: (R[31:16]==5) || ((R[15:8] & 0x7F)==0)
```

**There is no table copy on the DSP side.** The SE is the only intermediary, and it is handed
*flags*, not a table. So `gCusHash` can only matter if the SE itself consumes it — which
cannot be settled by any static analysis we can perform on the ARM module.

### 25.2 The firmware only READS the register

Every access to `0xB000001C..1F` in `dec_full.bin` is a load (`lbz` / `lh`). There is no
store. The register is produced by the SE or by the host through another alias.

### 25.3 There are more licence gates than the four we knew

The same dual-condition block appears at **0x021AC3, 0x021B19, 0x021F0E, 0x021F5C, 0x0138AE,
0x013D97** (six sites, two encodings: `andi 0x7F` and signed `blesi -1`). A seventh site at
0x012D9D reads `0xB000001E` as a **byte** and compares against **0x32** — a different
register semantic, likely a status/version field, not a licence.

### 25.4 Host side: DTS has a FIFTH condition we had not seen

`MDrv_AUDIO_Get_DTS_License` @0x422AA0:

```
0x422aa4  mov r0,#0xf   ; IPCheck(15)
0x422abc  mov r0,#0x3a   ; IPCheck(58)
0x422ac8  mov r0,#0x12   ; IPCheck(18)
0x422ad8  mov r0,#7     ; IPCheck(7)
  ... build r4 = bitmask of the four
0x422b20  movw r0,#0x2cf0
0x422b24  movt r0,#0x11          ; r0 = 0x112CF0
0x422b28  bl  HAL_AUDIO_AbsReadReg
  ...
0x422b64  and r4, r6, r4         ; DTS = (4 IP bits) AND 0x112CF0
```

**So DTS is gated on the host side by a hardware register `0x112CF0` as well as the four IP
bits.** A correct bitmap is not sufficient if that register is 0.

No writer of `0x112CF0` was found on the DTS path: `MDrv_AUDIO_SetDTSCommonCtrl` is a 4-byte
tail-call to `HAL_MAD_SetDTSCommonCtrl`; `HAL_AUDIO_DTSELoadCode` (@0x444010, 0x54 B) only
logs; `_MApi_AUDIO_GetDTSInfo` only logs.

### 25.5 Reading it is blocked by the mdb interface

`/proc/utopia_mdb/riuinfo` `ByteR` takes an **8-bit offset**: `ByteR,0x112,0xcf0,0,0` was
served as `Reg 0x112f0` (0xCF0 truncated to 0xF0), while `ByteR,0x112,0xa82,...` gave
`Reg 0x11282`. So the host-side register `0x112CF0` **cannot be read through the mdb path** —
it needs a 12-bit offset.

### 25.6 Where this leaves the model

There are **two independent gates**, not one:

| # | side | condition |
|---|---|---|
| 1 | host | (IP bits 7,15,18,58) AND (reg `0x112CF0`) |
| 2 | DSP | (R[31:16] == 5) OR ((R[15:8] & 0x7F) == 0), R at `0xB000001C` |

Feeding the bitmap satisfies gate 1's IP half and nothing else. That is consistent with §23:
correct provisioning, DTS still blocked. **The unexplored half of gate 1 is the hardware
register `0x112CF0`**, and we currently have no way to read it or find its writer.


## 26. THE DTS GATE REDUCES TO ONE BIT (0x112CF0 bit 7)

### 26.1 `0x112CF0` is a per-codec bitfield, not a DTS register

`0x112CF0` is read by **every** codec licence getter — AAC (0x4223F4), AC3 (0x422634), AC4
(0x4227F4), MAT (0x422994), **DTS (0x422B20)**, WMA (0x422D60), DRA (0x422E30) — and each
extracts *different* fields:

```
AC3 :  mov  r6, #0x1f                 ; mask bits 0..4
       tst  r0, #0x1000                ; + bit 12
       and  r6, r1, r0, lsl #4         ; bit12 -> bit4
       tst  r0, #3 ; orrne r6, r6, #0xf
       ands r4, r6, r4                 ; AC3 = (its IP mask) AND r6

DTS :  lsl  r0, r0, #0x18
       and  r6, r5, r0, asr #31        ; r5 = 0xF
       ands r4, r6, r4                 ; DTS = (its IP mask) AND r6
```

DTS's `lsl #24` / `asr #31` selects **bit 7** of the register. AC3 needs bits 0–4 and bit 12.

**Therefore: AC3 works ⇒ bits 0–4 are set. DTS is blocked ⇒ bit 7 is not set.** The host-side
DTS licence is a single bit of a multi-codec capability register.

### 26.2 Why the earlier "0x112CF0 is common, so it is not the differentiator" guess is wrong

The register is common, but the *fields consumed* differ per codec. AC3 passing proves nothing
about bit 7.

### 26.3 The host side is therefore fully understood and fully satisfied

With an all-FF bitmap the four IP bits (7, 15, 18, 58) are all set, so
`MDrv_AUDIO_Get_DTS_License` returns 0 only if **bit 7 of 0x112CF0 is clear**. The host does
not block DTS for any other reason we can find.

### 26.4 Who sets bit 7?

Not the DTS path: `MDrv_AUDIO_SetDTSCommonCtrl` is a 4-byte tail call to
`HAL_MAD_SetDTSCommonCtrl` (@0x464888, a 9-case dispatcher) which only issues
`HAL_DEC_R2_Set_SHM_PARAM(0x30/0x31, 0|1, 0, 0)`.

`HAL_DEC_R2_Set_SHM_PARAM` @0x45A434 is the host→DEC-R2 mailbox, and the host **does** have a
direct mapping of decoder memory:

```
bl   HAL_AUDIO_GetDspMadBaseAddr
movw r2, #0x1000 ; movt r2, #0xe0      ; + 0xE01000
adds r0, r0, r2
bl   MsOS_PA2KSEG1                      ; → g_virDecR2shm
```

with 0xD1 parameters dispatched through a jump table. The DTS ones land at:

| param | offset | when value=1 |
|---|---|---|
| 0x30 | `shm+0x1034` | `shm+0xC74` |
| 0x31 | `shm+0x1040` | `shm+0xC80` |

(`rsb r0,fp,fp,lsl #4` gives 0 or −15, then `lsl #6` → 0 or −0x3C0, then `str sl,[r0,r1]`.)

**This is a documented, parameterised write path into the decoder's shared memory** — a far
more promising surface than the IPAUTH route, because it does not involve the SE at all.

### 26.5 The open question

Whether DSP `0xB000001C` and host `0x112CF0` are the same physical register seen through two
aliases. The firmware reads the verdict as a halfword at `0xB000001E` and a byte at
`0xB000001D`; the host reads bits of `0x112CF0`. The DTS SHM parameters sit at `shm+0x1034`,
which is not obviously either. Resolving the aliasing is the next concrete task, and it is
answerable by reading the register's value through two paths and comparing.


## 27. BREAKTHROUGH: the DTS R2 mailbox is reachable from userspace, and writing it moved DTS

`MApi_AUDIO_SetDTSCommonCtrl` is dispatch **cmd 144** (statically mapped, anchor-validated).
Traced to the end:

```
MApi_AUDIO_SetDTSCommonCtrl → MDrv_AUDIO_SetDTSCommonCtrl → HAL_MAD_SetDTSCommonCtrl @0x464888
   sub r0,#6 ; cmp r0,#8 ; bhi exit        ; dispatcher over sub-commands 6..14
   case 0 -> mov r0,#0x30 ; case 1 -> mov r0,#0x31
   mov r1,#0 ; mov r2, (arg!=0) ; bl HAL_DEC_R2_Set_SHM_PARAM
HAL_DEC_R2_Set_SHM_PARAM @0x45A434
   bl   HAL_AUDIO_GetDspMadBaseAddr
   movw r2,#0x1000 ; movt r2,#0xe0   ; + 0xE01000
   bl   MsOS_PA2KSEG1                  ; → g_virDecR2shm  (direct mapping of decoder RAM)
   ... add r0,r6,r0,lsl #6 ; str sl,[r0,r1] ; bl MsOS_FlushMemory
```

So `SetDTSCommonCtrl(6,1,…)` writes **1** to `shm+0x1034` and `SetDTSCommonCtrl(7,1,…)` writes
**1** to `shm+0x1040`. **This path does not involve the SE at all.**

cmd 144's argument layout: `r0 = payload[1..4]`, `r1 = payload[5..8]` (byte-packed, not a bare
byte at [0]). `pwoff` gained a `PWOFF_HEX` / argv[5] raw-payload input.

### First attempt was invalid — recorded so it is not repeated

`pwoff run 144 0 0 0 "hex"` put the hex in argv[6] while the tool read argv[5] → payload was all
zeros → `SetDTSCommonCtrl(0,…)` fell outside the dispatcher (`sub r0,#6` wrapped) and did
nothing. Correct form: `pwoff run 144 0 <hold> "hex"`.

### Result

```
vendor provisioning (UtopiaSetIPAUTH)   rc = 0
SetDTSCommonCtrl(6,1) + (7,1)          rc = 0, payload confirmed "000600000001000000"
DTS playback  npcm = 620 544 B          ← WAS 0 before this write
AC3 control   npcm = 1 880 064 B        ← audio still healthy, nothing broken
```

**The DTS R2-mailbox write changed DTS behaviour on the optical path: 0 → 620 544 B.**

### What is still unknown

Whether this is a DTS *bitstream* or decoded PCM — the npcm dump does not expose IEC61937
framing (§21.3), so the receiver is the only oracle. And 620 544 B is close to the ambiguous
675 KB seen once before (§21), which may have been a stream transition. The state is stable
though: the write goes into decoder shared memory and persists until a DSP reload or reboot.


## 28. THE MEASUREMENT HARNESS WAS WRONG — there is no optical device in Android

The user rejected the 620 KB and 270 KB readings twice, correctly. Chasing it found the real
problem, and it invalidates part of §20–§27.

### 28.1 `am start` played nothing at all

With `am start -a android.intent.action.VIEW -d file://…` there was **no player process alive**
after the intent, and every decoder reported `format : INVALID`, `play state : STOP` — for the
**AC3 file as well as the DTS file**. The VIEW intent was being intercepted (VLC is the only
matching package) but no playback resulted.

### 28.2 VLC does play — and produces zero on the optical path

Explicitly launching `org.videolan.vlc/.gui.video.VideoPlayerActivity` finally leaves a live
process (pid 6378). The user's read: "VLC грає, але це певно PCM" — correct. The npcm dump with
VLC running: **0 bytes**.

### 28.3 Root cause: Android has no optical/SPDIF audio device

```
dumpsys audio:
  Current: 2 (speaker): …, 400 (hdmi): …, 40000000 (default): …
```

Only **`speaker`** and **`hdmi`** exist. There is no `spdif`/`optical` device in the Android
audio stack. Yet the user gets DTS/DD to the Pioneer over the optical port, so the optical path
is **not an Android audio device** — it is driven by the MStar audio HAL according to the
«Цифровий вихід» setting (Пропустити / Автоматично), as already documented for AC3.

### 28.4 Consequence for every measurement so far

`dump_spdif_npcm` taps the **SPDIF transmitter**, which only carries anything when the HAL is
driving a non-PCM stream. Any PCM-based app (VLC, and the framework's own playback) is routed
to `hdmi`/`speaker` and **cannot** put data on the optical TX. So:

* 0 bytes for VLC is the **correct** result, not a failure;
* the "AC3 control = 1.8 MB" figures from §20–§27 were not AC3 passthrough at all — the dump
  was picking up unrelated traffic and I treated it as a health check. It was not one;
* **no conclusion about DTS has been invalidated by those numbers, because none of them ever
  measured passthrough.**

### 28.5 What is required to measure at all

An application that actually requests passthrough on this platform: the TCL native player, or
Kodi with passthrough enabled and the right sink. Kodi is running (`net.kodinerds.maven.kodi22`,
pid 5332) but its **JSON-RPC does not answer** — the web server is disabled, so it cannot be
driven remotely as things stand.

### 28.6 Status of the §27 "breakthrough"

**Unproven, and not evidence of anything.** The DTS R2-mailbox write (`SetDTSCommonCtrl(6,1)`,
`(7,1)`) was measured with a harness that never played passthrough. The 620 KB and 270 KB were
artifacts, and the clean run with a 0-byte blank control returned **0 bytes** — i.e. the
mailbox write produced no observable change either.

What survives: the static chain is real and correct —
`MApi_AUDIO_SetDTSCommonCtrl` (cmd 144) → `HAL_MAD_SetDTSCommonCtrl` → `HAL_DEC_R2_Set_SHM_PARAM`
→ direct write into decoder shared memory at `shm+0x1034` / `shm+0x1040`, a path that bypasses
the SE. It simply cannot be evaluated until something actually requests passthrough.


## 29. WORKING HARNESS + THE REAL SIGNAL: the block is at DECODER-OPEN

### 29.1 The harness finally works

`am start` with a VIEW intent never played anything (§28.1). Launching the player explicitly
does:

```
am start -n org.videolan.vlc/.gui.video.VideoPlayerActivity -a android.intent.action.VIEW \
         -d file:///sdcard/Movies/test_dts_51.mp4 -t video/mp4
```

Leaves a live process and real playback. Test files staged in `/sdcard/Movies/`.

### 29.2 The oracle: the decoder `format` field

`show_all_decoder_status` reports, per decoder, `Decoder format` and `play state`. With the
harness working, both files were played back-to-back under identical conditions:

| run | Decoder 0 | npcm over 16 s |
|---|---|---|
| **AC3** | `format : AC3P`, `play state : PLAY` | 2 500 608 B |
| **DTS** | `format : INVALID`, `play state : STOP` | 0 B |

**For AC3 the hardware audio decoder opens. For DTS it never opens at all**, and the npcm
dump is empty because no non-PCM stream is ever handed to the HAL.

This corrects §28.4/§28.6 and the user's earlier suspicion in the opposite direction: the
`INVALID` readings were *not* caused by our `SetPowerOn` experiments — AC3 still opens
cleanly through the same path. The decoder is healthy; it simply refuses DTS.

### 29.3 The DTS R2-mailbox write does not lift it

`MApi_AUDIO_SetDTSCommonCtrl` (cmd 144) was driven for **all nine** sub-commands 6..14 with
value 1 (payload `00 NN 00 00 00 01 00 00 00`), after re-running the vendor provisioning.
Result: decoder 0 still `INVALID / STOP`.

So `HAL_DEC_R2_Set_SHM_PARAM(0x30/0x31,…)` — the direct, SE-bypassing write into decoder
shared memory — is not the switch. §27's "breakthrough" is retracted: the 620 KB and 270 KB
readings were artifacts, and the clean run with a 0-byte blank control returned 0.

### 29.4 Where the block actually sits

The DTS licence gate we decoded in the DEC firmware (`*(u16*)0xB000001E == 5`, with the
alternative grant at byte `0xB000001D`, §24) is at precisely the point where the decoder
decides whether to open for DTS. Failing it, the framework falls back to software decoding —
which is exactly what the user hears ("DTS в PCM схоже").

That is consistent with everything else: the DSP-side register is read-only to the firmware,
`0x112CF0` bit 7 is the host-side mirror of the same verdict, and only the SE can set it.

### 29.5 What would still be informative

A player that actually *requests* passthrough. Kodi's settings already have
`audiooutput.passthrough=true` and `dtspassthrough=true` with sink
`AUDIOTRACK:AudioTrack (RAW)|Android IEC packer` (from the earlier `kodi22_guisettings.xml`
pull), but Kodi's **web server is disabled**, so it cannot be driven from the shell. Enabling
it (one settings change, reversible) would give a proper passthrough-capable harness — the
one thing this investigation still lacks.


## 30. THE DTS DECODER OPENS — the block is ROUTING, not licensing

### 30.1 A working control channel, found by accident

Port 8080 is owned by **`ntechserver`** (550), not Kodi — that is why every JSON-RPC call
failed. Kodi also runs a **JSON-RPC TCP server on port 9090** which needs no HTTP and was
listening the whole time:

```
netstat -tlnp | grep 9090  →  8417/net.kodinerds.maven.kodi22
printf '{"jsonrpc":"2.0","method":"JSONRPC.Ping","id":1}' | nc 127.0.0.1 9090
   → {"id":1,"jsonrpc":"2.0","result":"pong"}
```

The settings edit for the HTTP server was unnecessary (and was reverted from backup).

### 30.2 DTS plays through the hardware decoder

```
Player.Open {file: /storage/emulated/0/Movies/test_dts_51.mp4}  → OK
   ↓ 6 s
Decoder ID 0 : format = DTS , play state = PLAY      ← hardware decoder engaged
Decoder 1..4 : INVALID / STOP
```

This is the first valid DTS measurement of the day, and it shows **the DTS decode path is not
blocked at all.** `MI_AUDIO_GetCaps` lies, IPAUTH provisioning, bit 7 of `0x112CF0` — none of
them prevented the hardware decoder from opening in DTS format.

### 30.3 But the optical transmitter carries nothing

```
Decoder 0 : DTS / PLAY        (verified during playback)
npcm (SPDIF TX) : 0 B         (14 s and 20 s windows, dump started before playback)
```

Kodi's passthrough sink is `AUDIOTRACK:AudioTrack (RAW)|Android IEC packer` — the **HDMI IEC
path**. The bitstream goes there; the optical S/PDIF output the Pioneer listens to is driven
by the MStar HAL only for the output that the «Цифровий вихід» setting selects.

### 30.4 The reframe

The whole §19–§25 push on the licence (bitmap, boot race, SE verdict) was aimed at a gate that
**does not block DTS playback** on this device. DTS decoding works. What remains for the
user's goal — DTS to the Pioneer over optical — is a **routing question**: get the bitstream to
the optical TX instead of the HDMI IEC path.


## 31. ATTEMPT TO READ 0x112CF0 — hit the known addressing wall again

`riuinfo` addressing is `BANK:OFFSET` with an **8-bit offset**, and the example in its own help
is `Dump,0x101e` — a **RIU** bank, not an ARM address. Reading `ByteR,0x112c,0xf0` returned:

```
Reg 0x112cf0 (8BitOffset) = 0x00
```

which would mean bit 7 = 0 (DTS refused) **and** bits 0–4 = 0 (AC3 refused) — but AC3 works, so
this register is **not** the one `HAL_AUDIO_AbsReadReg(0x112CF0)` reads. It is the same
address-space confusion the memory note already documented ("ARM 0x112E9A is NOT mdb riuinfo
banks 0x112E") and I repeated it.

**Net: `0x112CF0` cannot be read through the mdb interface.** Its value is only observable
through `HAL_AUDIO_AbsReadReg`, i.e. from inside the kernel module.

## 32. WHERE THE INVESTIGATION ACTUALLY STANDS (honest end-of-session)

**Proven working:**
* DTS *decoding* on this device is not blocked: through Kodi with passthrough enabled the
  hardware decoder runs `format : DTS, play state : PLAY` (§30.2). The §19–§25 licence work
  (bitmap, boot race, SE verdict) targeted a gate that does not block DTS *decoding*.

**The remaining block, precisely located:**
* `HAL_AUDIO_SetDigitalOut` calls `MDrv_AUTH_IPCheck` — the *digital output* stage has its own
  licence check, separate from decoder-open.
* Its DTS condition is bit 7 of `0x112CF0` (§26), which is not set, and the SE is the only
  plausible producer. We cannot read that register through mdb (§31) and have no write path.

**Not yet available:**
* Muse's V083 parse lacks `utpa2k.ko`, `mik.ko`, `libutopia.so`, `aucode*` (outside the 194 MB
  super.bin window) — a full 5 GB super is needed to extract them.

**Next actionable step, when resumed:** the firmware-side analysis of who produces
`0x112CF0` bit 7 — that is the Muse task already written, and it is the only remaining lever
on the host side.

## 33. THE EXACT SE SEQUENCE, AND WHY E2′ FAILED — three concrete errors

`MDrv_AUDIO_ApplyHashkey` @0x424B90 passes `AudioVars2+0x440` and `+0x444` to the group
writer at **0x424BD8**. Disassembled in full:

```
r6 = 0x112A82
0x424c50  add  r0, r6, #0x3e                        ; 0x112AC0
0x424c58  bl   HAL_AUDIO_AbsWriteMaskReg(0x112AC0, 0xFFF0, 0)
0x424c5c  sub  r0, r6, #4                           ; 0x112A7E      <-- PRE-STEP
0x424c68  bl   HAL_AUDIO_AbsWriteMaskByte(0x112A7E, 1, 1)
0x424c70  bl   HAL_AUDSP_CheckSeIdmaReady(8)         ; gate #1
0x424c7c  bl   HAL_AUDSP_CheckSeIdmaReady(0x10)       ; gate #2
         (if either fails -> NOTHING is written at all)
0x424c94  bl   AbsWriteByte(0x112A84, 0x83 or 0x84) ; COMMAND WRITTEN FIRST
0x424ca0  bl   AbsWriteByte(0x112A85, 0x1F)
0x424cac  bl   AbsWriteByte(0x112A82, flags>>8)      ; payload after the command
0x424cbc  bl   AbsWriteByte(0x112A83, flags>>16)
0x424cc8  bl   AbsWriteByte(0x112A82, flags & 0xFF)
0x424cd4  bl   AbsWriteByte(0x112A83, 0)
```

### Why E2′ (§13) failed — three specific errors

1. **Missing the 0x112A7E pre-step** and the 0x112AC0 masked clear.
2. **Wrong order** — E2′ wrote flags first and re-latched afterwards; the vendor writes
   **command (0x112A84) first, then 0x112A85=0x1F, then the payload bytes**.
3. **No readiness gating** — the whole block is conditional on
   `HAL_AUDSP_CheckSeIdmaReady(8)` and `(0x10)`; if the SE IDMA is not ready, the driver
   writes nothing at all. E2′ wrote unconditionally, i.e. at a moment the driver itself
   considers wrong.

The observable behaviour E2′ recorded — "flags read back 0x00, the SE consumes them" — is
exactly what you get from a sequence the SE never accepted.

### What this makes possible

The sequence is now known precisely, so it can be replayed from userspace with the correct
order, the pre-step, and payload flags set to all-ones (standing in for the stale
`+0x440`/`+0x444` that `CheckHashkey` cannot refresh while `+0x503` is closed).

**Risk, stated plainly:** these are writes into the SE window with no read-back path. Worst
case is a DSP hang and a reboot. This is the same class of risk as E2′, which is why it is
written down and not simply run.


## 34. A/B RESULT: the device answers the question itself — DTS licence = UNSUPPORTED

### 34.1 The oracle we finally had

With Kodi driven over JSON-RPC/TCP:9090 (`Player.Open` of a real DTS MP4, passthrough enabled
in Kodi's settings), the hardware decoder engages and the **per-format licence field** is
readable:

```
Decoder ID : 0
Decoder format : DTS
Deocder hash key : UNSUPPORTED
Decoder play state : PLAY
```

Earlier `show_all_decoder_status` runs showed `hash key : SUPPORT` for every decoder, but only
while all decoders were `INVALID / STOP` — i.e. with no format loaded the field is a default
and meaningless. **Once a format is actually playing, the field is real.**

### 34.2 A/B — the SE sequence changed nothing

| condition | `hash key` | npcm |
|---|---|---|
| vendor provisioning + replayed SE sequence (§33) | **UNSUPPORTED** | 0 B |
| clean reboot, no SE writes at all | **UNSUPPORTED** | 0 B |

The replayed sequence was *accepted* by the SE in the expected way — `0x112A84` (command) and
`0x112A85` (0x1F) stuck, `0x112A82`/`0x112A83` payload consumed back to 0, `0x112A7E`
pre-step consumed. That is latch behaviour, not rejection. **But the verdict did not move.**

### 34.3 What this closes

* DTS **decoding** works (`format : DTS, play state : PLAY`).
* DTS **licensing** does not (`hash key : UNSUPPORTED`, optical TX empty), on a clean boot
  with no writes of ours in play.

So neither lever we had actually reaches the licence:

* the **IPAUTH bitmap** (140 bytes, all-FF) — provisioned correctly via the vendor's own
  `UtopiaSetIPAUTH`, even 6.4 s before the first audio call at boot (§23), and the verdict
  stays unsupported;
* the **SE flag window** — replayed in the vendor's exact order including the `0x112A7E`
  pre-step and the readiness gates the module enforces, and the verdict stays unsupported.

**The remaining candidate is the SE/firmware side** — whoever produces `0x112CF0` bit 7 / the
DSP word at `0xB000001C`. We can neither read that register through mdb (§31) nor write it.
That is the Muse task already open, and it now has a much sharper target: *not* "how do we feed
the bitmap" — that is settled — but **"what makes the SE grant the DTS bit at all"**.


## 35. CORRECTION TO §34 — `hash key` is NOT a licence oracle

The AC3 control invalidates §34's reading:

| format | `Deocder hash key` | npcm |
|---|---|---|
| **AC3P** (passthrough works) | `UNSUPPORTED` | **2 426 880 B** |
| **DTS** (blocked) | `UNSUPPORTED` | 0 B |

The field is **identical for a working and a blocked format**, so it cannot be the licence
verdict. §34's "the device answers the question itself" is **retracted** — the field is
another decoy readout, in the same family as `MI_AUDIO_GetCaps`.

Two earlier claims about it are also retracted:
* that `SUPPORT` earlier vs `UNSUPPORTED` now was caused by the SE writes — the A/B refuted
  this (clean boot, no writes, also `UNSUPPORTED`);
* that the field "moved" at all — the earlier `SUPPORT` readings were from decoders in
  `INVALID / STOP` with no format loaded, i.e. a default state, not a comparison point.

## 35.1 What IS a real signal

Only the npcm contrast, and it is clean and reproducible with one player, one method:

```
AC3P : 2 426 880 B     DTS : 0 B
```

This is exactly what the user has been hearing all along, and it needs no interpretation:
DTS produces nothing on the optical transmitter while AC3 does. The mechanism behind it is
still unproven — the only candidate left standing is the SE/licence gate (§26, §33), but we
have no readable oracle for it: `0x112CF0` cannot be read through mdb (§31), and the SE flag
window does not move the verdict when written in the vendor's exact order (§34.2).


## 36. TD98 ↔ TCL ADEC, compared directly (the gap nobody had closed)

Muse compared V474↔V098 (Δ594) but **not TD98↔TCL**. Done here — the decisive pair is the
*MS12V22* variant, because our chip is MS12V22-class:

```
TD98  dec_full.bin                     1 982 492 B  md5 4b7e9509b435
TCL   aucode_adec_r2_MS12V22.bin       1 974 796 B  md5 556c9d6b108f     Δ 7 696 B
```

| | `lic[%d,%d,%d]` | `Invalid Spdif license` | `DTS` | `dts` |
|---|---|---|---|---|
| TD98 | 1 @ 0x142892 | 1 @ 0x144904 | 11 | 11 |
| TCL  | 1 @ 0x140CE7 | 1 @ 0x142B5D | 11 | 11 |

**Same build family, same licence machinery, same DTS surface.** Combined with Muse's
store-scan (0 writes to `0xB000001C..1F` in any of the four images) this settles the
firmware side:

> **The DSP firmware is not the difference.** Our image is complete and equivalent to the TCL
> one. Nothing in `aucode*` needs changing, and no firmware comparison will produce a fix.

## 37. Where the difference therefore must be

With DSP images equivalent and the kernel module equivalent (Muse: V098's `utpa2k` has the
same callees and the same `Get_DTS_License`), and with `MApi_AUTH_Process` **never called in
any of the three TCL builds either**, the only remaining difference between a TCL that plays
DTS and this projector is the **SE's own stored licence** — a manufacture-time property of
the secure element.

**Conclusion:** DTS passthrough on this projector is not blocked by anything we can reach in
software. The unit was not provisioned for DTS, and the provisioning is inside the SE.


## 38. THE LICENCE MODEL — found in M-Star's own documentation

The user was right that a mechanism must exist, and it does — documented by the vendor.

**M-Star License Utility** (docs.mstarcfd.com/3_Licensing/):

> "an interactive tool for configuring and managing M-Star licenses. It allows users to obtain
> their **host ID (MAC address) for node-locked licenses**, configure licensing on your
> computer, **verify license availability and options**, and check out '**roaming' licenses
> from a license server**."

**M-Star License Activation:**

> "launches the license activation wizard… supports activation via **license keys in the format
> `XXXX-XXXX-XXXX-XXXX`** and guides the user through completing a local license setup."

Sections in the same tree: *Workstation License*, *Floating/Network License*, *Troubleshooting
License Issues*, *Third-Party Licenses*, *Locate your MAC Address*.

### What this means for TD98

The codec licence on an M-Star TV SoC is **not a firmware artefact and not something the host
computes**. It is a **per-unit entitlement**, node-locked to the unit's MAC, issued by a licence
server and redeemed with a purchased key. Our findings line up exactly with that model:

* `MDrv_AUTH_IPCheck` reads a **host-uploaded table** — a request channel, not the licence;
* the DSP never writes the verdict register — it only reads it;
* every host-side lever (bitmap, SE flag window, R2 mailbox) leaves the verdict unmoved;
* the TD98 and TCL ADEC images are the same build family, and TCL's kernel module is
  equivalent — so the difference is not in the software at all.

**Caveat, stated honestly:** this is documentation for M-Star **CFD**, their EDA/simulation
product. It documents the vendor's licensing *model* for their own tooling, not the audio
licence inside a TV SoC. It is strong corroboration of the architecture, not a datasheet proof
of the mechanism used for DTS on MS12V22.

### The honest bottom line

A unit that was never provisioned cannot be provisioned by the owner. Everything reachable from
this side — the IPAUTH table, the SE flag window, the R2 mailbox — has been tested and does not
move the verdict, which is exactly what a MAC-locked server-issued entitlement would do.


## 39. RETRACTION OF §38 — the M-Star CFD docs say nothing about codec licences

Read `3_Licensing/thirdparty-licenses.html` in full: it is the **open-source attribution
list** for the CFD software (Pybind11/BSD, GPL v3 text, …). It contains **no mention of
Dolby, DTS, or any audio codec licence.**

The whole `3_Licensing` tree is about **M-Star CFD's own EDA-tool licensing** — MAC-locked
workstation keys, roaming licences, a licence server. It says nothing about what a TV SoC does
with its audio codecs.

**§38's architectural inference is therefore withdrawn.** The user raised both objections
("I'm not sure this is about DTS" / "I'm not sure there's no way round it") and both were
correct. This is the third time in this session that I generalised from a source that did not
support it; recorded so it is not repeated.

## 39.1 What survives, and the gap that is actually still open

Everything in §36/§37 survives — those rest on our own measurements and a direct
`dec_full.bin` vs `aucode_adec_r2_MS12V22.bin` comparison, not on vendor docs.

But there is a **real gap in our reasoning that this retraction re-opens**, and it is the
strongest remaining idea:

> **`MApi_AUTH_Process` is imported (UND) but never *called statically*.** A **runtime
> `dlopen`/`dlsym` would not appear in any static scan we did** — relocation-based analysis
> cannot see it. So "the middleware never calls it" is *not* established for a running system;
> it is established only for the three builds we read as files.

`MApi_AUTH_Process` is exactly the function that takes the 48-hex chip string and produces the
(arg0, arg1) pair whose SHA/MD5 the SE would accept. It is **defined** in the TD98
`libutopia.so` (0x4C6768, fully disassembled by us in `auth_process_td98.txt`), **UND-imported
by `libmi3`**, and we have reconstructed its crypto.

**Untested approach:** call `MApi_AUTH_Process` ourselves, from our own process, via
`dlopen` of the device's `libutopia.so`, with a self-consistent (arg0, arg1). The earlier
`e1_probe3` crash was in `UtopiaOpen+44` (framework NULL) — but `MApi_AUTH_Process` itself
writes to plain globals and then calls `UtopiaSetIPAUTH`, which is a self-contained
`open`+`ioctl` path. That path has never actually been exercised.

This is the one concrete idea left that could make the SE accept our data, and it has not
been tried.


## 40. THE DYNAMIC PROVISIONING PATH — found, and it never runs

### 40.1 It exists and is loaded right now

```
libmi_td98.so   contains the literal string  "libdrvIPAUTH.so"
/proc/499/maps   /mnt/vendor/tvservice/glibc/libdrvIPAUTH.so   r-xp / ---p / r--p / rw-p
```

`libdrvIPAUTH.so` (26 148 B) exports `MApi_AUTH_Process` @0x1CB0 (3244 B, **ARM mode**,
signature `(r0=arg0, r1=arg1)`) and carries the validation strings:

```
MDrv_SYS_GetChipID:%x
Wrong Chip ID
Wrong hash key
[Auth NG]
AUTH STATUS:%x
```

So the vendor's own provisioning entry point is **dlopen'd in the running audio process** —
something our static relocation analysis could never see, as suspected.

### 40.2 But it is never called

With the module log raised to max and real playback running (Kodi/JSON-RPC, AC3), the kernel
log contains **zero** `IPAUTH` / `[Auth` / `Wrong Chip` / `hash key` lines for this boot.

```
count: 0
```

### 40.3 Calling it ourselves — blocked by the glibc split

`dlopen("/mnt/vendor/tvservice/glibc/libdrvIPAUTH.so")` from our process:
1. `libgcc_s.so.1 not found` → fixed with `LD_LIBRARY_PATH=/mnt/vendor/tvservice/glibc`
2. `cannot locate symbol UTPA_USR_LOGLEVEL` → needs `/vendor/lib/libutopia.so` preloaded
3. then a hard conflict: the library wants **glibc 2.21** (`libc-2.21.so` in the tvservice
   rootfs) while an NDK r30 binary wants the system glibc + `libdl.so`, which **does not
   exist in the 2.21 rootfs** (in 2.21 `dlopen` lives in libc itself).

Running under the 2.21 linker works for a static binary, but a static NDK build cannot link
`dlopen` (`undefined symbol: dlopen`).

### 40.4 The coherent reading

`libmi` dlopens the library but never invokes `MApi_AUTH_Process` — and the reason is almost
certainly the one already established independently: **the chip-string global on this projector
is all zeros.** The middleware has no customer data, so the provisioning entry point is never
reached. That single fact explains, consistently:

* "Process imported but never called" (gated on empty customer data);
* the 140-byte table staying empty on the device;
* DTS never being provisioned — while AC3 is licensed through a different, Dolby-owned
  channel that needs no such data.

It also reframes the goal: the missing thing is not a *bypass* of a check but the **absence of
input data**. Which suggests a different, and so far untried, approach: supply the missing
customer data at runtime so the vendor's own code path executes as designed.


## 41. READ-BEFORE-WRITE: what is actually in the customer data

The user's discipline — read the state before touching it — was correct and immediately paid.

### 41.1 The buffer is genuinely empty, in the live process

`libmi.so` symbols (local copy, matches the device file):
```
u8Customer_info  @0x12AB80  size 49   (48 hex chars + NUL)
u8Customer_hash  @0x12ABBC  size 16
```

Read from the **running** `libmi.so` in pid 499 (base 0xB0460000):

```
libmi+0x12AB80  -> 00 00 00 …   96 B of zeros
libmi+0x12A000  -> 4096 B, zero non-zero nibbles      ← the whole region is empty
libmi+0x128000  -> 4096 B, 89 non-zero nibbles, no ASCII-hex run
libmi+0x129000  -> 4096 B, 25 non-zero nibbles, no ASCII-hex run
```

**No 48-character hex chip string exists anywhere in libmi's data.** The customer data is
absent at runtime, not just in the static file.

### 41.2 There is a gate *before* the buffer is ever filled

`mi_sys_GetCustomerInfo` @0xACBB8 (80 B, ARM):

```
0xacbc0  ldrh r3, [r3]          ; a 16-bit value from a global
0xacbc4  cmp  r3, #0x1f
0xacbc8  bhi  0xacbd4           ; >31  -> proceed
0xacbcc  mov  r0, #0             ; <=31 -> return 0, customer info NOT supported
0xacbd0  bx   lr
0xacbd4  …                      ; only here is u8Customer_info filled
```

So the chain stops **before** the data, not after it. Resolving the exact runtime address of
that 16-bit global needs the PIC addend applied to the load base; the naive `base + 0x136C90`
read gave `pread: I/O error` (beyond the mapping), so the correct address is not yet pinned.

### 41.3 The property that plausibly feeds it is zeroed

```
[ro.boot.Serial]    = 0000000000000000     ← 16 zeros
[ro.boot.serialno] = 62ED4791C2           ← a real identifier
[ro.board.platform]= mt5889
```

`ro.boot.Serial` being all zeros is consistent with `u8Customer_info` being all zeros, and it
is the kind of value a factory would write. `ro.boot.serialno` holds a real, non-zero
identifier that **nothing on this device currently reads** for licensing purposes.

### 41.4 What this rules out and what it does not

Ruled out: the theory that the data exists but is used from somewhere we have not looked.
It is not there.

Not ruled out: whether a *correct* 48-hex value can be supplied. Two independent zeros block
the path today — the zeroed property and the ≤0x1F gate — and both are data, not checks, so
they are writable in principle. What is still unknown is **what value the SE was provisioned
against**; a self-consistent (arg0, arg1) from an arbitrary chip string did not move the
verdict (§34.2), which is the strongest evidence that the value must be the *real* one.


## 42. Where the chip ID actually comes from — and a correction

### 42.1 CORRECTION — `mi_sys_GetCustomerInfo` is a printf, not a gate

§41.2 read the block at 0xACBB8 as "a gate that returns 0 before the buffer is filled".
Resolving its call target shows otherwise:

```
0xacbd4  ldr  r2, [pc,#0x24]     ; -> u8Customer_info
0xacbd8  movw r3, #0xc45
0xacbdc  ldr  r1, [pc,#0x20]     ; -> format string
0xacbe0  mov  r0, #1
0xacbf0  bl   0x16808  ->  __printf_chk        (PLT index 107)
```

It is `__printf_chk(1, fmt, u8Customer_info)` — a **debug dump of the buffer**, and the
`cmp r3,#0x1f; bhi` is a verbosity/format guard, not a licence gate. **The §41.2 "internal
gate" does not exist.** Another over-read on my part; corrected here.

### 42.2 Where the chip ID lives

```
utpa2k module :  MDrv_SYS_GetChipID @0x25988  (4 B tail-call)
                   -> SYS_GetChipID @0x28880
                      movw r0,#0 ; movt r0,#0 (stSysInfo) ; ldrh r0,[r0] ; bx lr
                  i.e. it returns stSysInfo[0], a 16-bit value cached at init.
```

A 16-bit chip ID cannot be the source of a 48-hex (24-byte) string by itself, so the string
is assembled from more than this.

### 42.3 The usable, exported getters

`libutopia.so` (the userspace copy, same file we already dlopen successfully in `vendup`)
**exports**:

```
SYS_GetChipID           @0x5F5260  (20 B)
MDrv_SYS_GetChipID      @0x5ED33C  (4 B, tail-call)
MDrv_PM_GetChipID       @0x566380  (528 B)   ← a real chip-ID reader
KeyCustomerList         @0x10D5AC0 (72 B)    ← the customer key table
MDrv_SYS_GetDolbyKeyCustomer @0x5ED520 (144 B)
```

`PT_SYS_GetCusInfo` — which `libmi` imports — is **not** exported by `libutopia.so`; it comes
from elsewhere, which is why the chain could not be followed further statically.

### 42.4 Bottom line for the requested investigation

Asked: *can the real chip ID be obtained?* **Yes, and it is cheap** — `vendup` already proves we
can dlopen this exact library from our own process, so `SYS_GetChipID` / `MDrv_PM_GetChipID`
are directly callable and purely read-only. No glibc rootfs problem (that only affected
`libdrvIPAUTH.so`, which lives in the tvservice rootfs).

**Not yet done, awaiting approval:** calling those two getters and printing the values. If the
48-hex string is derivable from them, the correct `u8Customer_info` value becomes writable
with a real input rather than an invented one — which is the one thing our earlier all-FF and
all-ones attempts could not provide.


## 43. CHIP-ID READOUT — the licence input cannot be obtained from userspace

`e1_ipauth/chipid.c` (read-only) dlopens `/vendor/lib/libutopia.so` and calls the exported
getters. Results:

**All four exported symbols resolve; `PT_SYS_GetCusInfo` is absent from this library.**

### 43.1 KeyCustomerList — read, and it matches the known keys

```
entry 0 : id=0x00  key = 285bf7c6 94cf2c6b 8b67ffda e8c01845
entry 1 : id=0xE8  key = bafcf09c ba6412d7 9432e120 4f1b7bfa68
entry 2 : id=0xF4  key = 98adbcac 968ef904 31cfc171 c6c1ac70
```

Entry 0 is the key our licence reconstruction already uses; entries 1 and 2 (0xE8, 0xF4) are
the alternative customer keys.

### 43.2 The chip ID is zero in userspace

```
SYS_GetChipID()      = 0x0000
MDrv_SYS_GetChipID() = 0x00000000
```

### 43.3 The hardware reader needs a framework we do not have

```
MDrv_PM_GetChipID:
  UTOPIA ASSERT: 0, vendor/mediatek/tv/misdk/utopia/.../pm/drvPM.c PM_Result
                 MDrv_PM_GetChipID(MS_U8 *) 1354
```

It is a Utopia **PM module** call, not a register read — it needs the framework initialised,
which is the same class of requirement that made `e1_probe3` fail earlier.

### 43.4 Consequence

The licence is bound to an identity we cannot read from userspace on this device: the platform
copy of the chip ID is zero, the only hardware reader is behind the Utopia PM framework, and
`u8Customer_info` (49 B) is empty in the live process. Combined with `ro.boot.Serial` being
`0000000000000000` while `ro.boot.serialno` is `62ED4791C2`, the picture is consistent: **the
customer identity is zeroed on this unit at the platform level**, and the provisioning input
the SE was built to check against is not recoverable from the device.

This is now a *measured* conclusion rather than an inference from three software levers: the
input itself is absent, so no amount of correct flag-writing can match it.


## 44. STAGE 1 RESULT — the chip ID is not obtainable from userspace (three paths closed)

Stage 1 was: open the Utopia PM module the way `pwoff` opens AUDIO, and call
`MDrv_PM_GetChipID`. All three possible paths are closed.

**1. The PM module has no userspace ioctl.**
```
PMIoctl @0x78EE8 (8 bytes):
  0x78ee8  mov r0, #0
  0x78eec  bx  lr
```
It returns 0 unconditionally. There is no PM command to issue, so there is no module-id to
open and no command to send.

**2. The real read is a mailbox message to the PM coprocessor.**
`MDrv_PM_GetChipID` @0x76050 does `MDrv_MMIO_GetBASE(virtBaseAddr, u32BaseSize, 0x12C)` →
`HAL_PM_SetIOMapBase` → `MDrv_MBX_SendMsg(...)`. The chip ID is delivered by the PM
subsystem, not by a plain register read we can issue.

**3. The chip block is not reachable through the mdb register interface.**

With the audio log quiet (see §44.1) the interface is verified working —
`ByteR,0x112a,0x85` → `0x1F` as before — but:
```
ByteR,0x1f01,0x2c   -> 0x00
Dump,0x1f00          -> 256 B, every halfword 0x0000
```
The whole 0x1F00xxxx chip block reads as zero through `riuinfo`, while the audio registers
in the same interface respond correctly. So the riuinfo window does not cover the chip block
(the ARM "0x11xxxx" style window that the driver uses is a different address space — the same
address-space mismatch established in §31).

### 44.1 Operational note worth keeping

The riuinfo interface appeared "dead" during this work. It was not: `debug_level=5`, left
over from an earlier test, had the Monitor flooding dmesg with PCM spam, so the ring buffer
wrapped within milliseconds and every `Reg ...` line vanished before it could be read.
**Always set `debug_level=0` before using mdb register reads.**

### 44.2 Where this leaves stage 2

Stage 2 (working out how the 48-hex string is assembled) is still worth doing as static
analysis of who writes `u8Customer_info` — but it now has a known ceiling: **even if we learn
the exact format, the value that format would have to be filled with is the chip ID, and the
chip ID is not readable on this device.** That is the honest position.


## 45. STAGE 2 — the mechanism, and the definitive reason (final)

### 45.1 The platform has a plug-point for customer data, and nothing is plugged in

`libmi` carries the whole mechanism:

```
PT_SYS_SetCustomerizeFunPtr  @0xD7D20  (0xAC)  register an OEM callback
gstSysCusFunPtr              @0x12A454 (0x3C = 15 pointers)  the callback table
PT_SYS_GetCusInfo            @0xD768C  (0x3C)
PT_SYS_SetCusInfo            @0xD75EC  (0xA0)
PT_GetCustomerInfo           @0xD3C80  (0x20)
MDrv_SYS_GetChipID / GetChipType, MDrv_SYS_QueryDolbyHashInfo   (imports)
```

`libmi` does not compute the customer identity itself. It **calls an OEM-registered
callback** to obtain it, and `MApi_AUTH_Process` consumes what that callback returns.

### 45.2 The table is empty in the running process

Read from the live `libmi.so` in pid 499 (base 0xB0460000), at +0x12A454, 60 bytes:

```
000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000
```

**All fifteen pointers are NULL.** Non-zero nibbles: **0**.

### 45.3 This is the definitive reason, measured at runtime

The chain is now complete and every link is an observation, not an inference:

1. The SoC carries the DTS:X IP (MediaTek licensed it, H2 2019 — dts.com).
2. The platform exposes a **plug-point** for the OEM to supply the customer identity.
3. **On this projector nothing is plugged in** — all 15 callback slots are null.
4. Therefore `u8Customer_info` (49 B) is empty, `ro.boot.Serial` is zeros, and
   `MApi_AUTH_Process` is never invoked (0 IPAUTH log lines all session).
5. With no provisioning, the SE never grants the DTS bit, `0x112CF0` bit 7 stays clear, and
   the DEC firmware's licence gate (`*(u16*)0xB000001E == 5`, or the `0xB000001D` alternative)
   refuses — so DTS decodes on the CPU path but produces nothing on the optical transmitter.

**This is a product-provisioning omission, not a firmware, driver, hardware or attack-surface
problem.** Every software lever we exercised (IP bitmap, SE flag window, R2 mailbox, boot-race
timing, the vendor's own `UtopiaSetIPAUTH`, and the vendor's own SE write sequence) acted at
or above a layer that is simply not initialised on this unit.

**What would actually be required:** the OEM to register its customer-data callback — i.e. the
factory to provision this SKU, exactly as they did for TCL. Nothing reachable from an adb
shell substitutes for the data the callback would have returned.


### 45.4 What the OEM would have registered

`PT_SYS_SetCustomerizeFunPtr` @0xD7D20 (0xAC) is a plain copy of 15 pointers:

```
0xd7d24  subs r3, r0, #0            ; r3 = the caller's table
0xd7d40  ldr  r2, [r1, r2]          ; r2 = &gstSysCusFunPtr
0xd7d3c  ldr  ip, [r3]              ; then 15x (ldr → str) at offsets 0x00 … 0x38
```

So the OEM passes a `struct { void *fn[15]; }` and it is memcpy'd verbatim into the table.
One of those fifteen slots is the customer-info provider that feeds `MApi_AUTH_Process`; on
this unit all fifteen are NULL, so no slot is available and the provisioning call is never
made. The exact slot index was not resolved (the library is largely Thumb-2 and the PLT/GOT
scan did not match), but the conclusion is independent of it: with an empty table there is
nothing to call.


## 46. CHECKING THE MAC THEORY — it is wrong, and that matters

The M-Star CFD docs had suggested licences are node-locked to a host ID / MAC, and the device
has a real MAC (`ro.mac = 74:25:84:D4:B8:2C`), so it looked as if the missing 48-hex string
might be derivable. Re-read `MApi_AUTH_Process` (@0x4C6768, our own annotated listing in
`e1_ipauth/auth_process_td98.txt`) to check what that string actually is.

### It is a licence, not an identity

```
0x4c6ad8  ldrb r7, [r8]      ┐
0x4c6ae4  ldrb r7, [r8,#1]   │
0x4c6af4  ldrb r6, [r8,#2]   │  reads 16 bytes from a buffer
 …                              │  and compares each against a parsed value
0x4c6be0  ldrb r1, [r8,#0xf] ┘
0x4c6c80  strb r6, [r1,#8]      writes the size byte into the IP table
```

**Sixteen bytes** of licence content are compared and installed; the size byte goes into the
140-byte table. The 48-hex blob is therefore:

```
[0..7]   hex  → 4 bytes  → gCusID + table seed
[8..47]  hex  → 16 bytes → THE LICENCE BITS
```

**It is the issued licence content itself, not a device identity.** A MAC may *bind* such a
licence (node-locked), but the content is issued data — it cannot be computed from anything on
the device. Our own reconstruction already assumed this: the kit's `arg0` was
`0000000000000000ffffffffffffffffffffffffffffffff`, i.e. seed 0 plus 16 all-ones licence bytes,
and that is precisely what failed to move the verdict.

**So the MAC line of attack is dead on arrival, and no arithmetic on `ro.mac`,
`ro.boot.serialno` or `eeprom_0` can substitute for data that DTS/Xperi issued to Thundeal.**

Recorded so the next session does not re-derive it.


## 47. The one untested avenue — tried, blocked differently than predicted

`libutopia.so` exports the register accessors that the mdb path could not reach:

```
HAL_AUDIO_AbsReadReg  @0x38A5D0      HAL_AUDIO_AbsWriteByte     @0x38AD14
HAL_AUDIO_AbsWriteMaskByte @0x38AF24  HAL_AUDIO_AbsWriteMaskReg  @0x38B1E4
HAL_AUDIO_GetDSPalive  @0x3962F8
```

`e1_ipauth/absreg.c` dlopens the library, opens an AUDIO instance on `/proc/utopia` with the
**proven** vehicle (`0xC00C5501`, module 0x34), then calls the accessors directly — no mdb,
no 8-bit offset limit.

```
HAL_AUDIO_AbsReadReg  : present
HAL_AUDIO_AbsWriteByte: present
HAL_AUDIO_GetDSPalive : present
/proc/utopia fd = 3
UtopiaOpen(AUDIO) rc=0            ← the instance DID open
GetDSPalive() = 0
AbsReadReg(0x112CF0)  →  Segmentation fault
```

**Blocked, but not by the address space** — the instance opened fine, so that hypothesis is
wrong. `HAL_AUDIO_AbsReadReg` faults inside libutopia because the surrounding framework state
is not initialised. This is the same failure class as `e1_probe3`'s crash at `UtopiaOpen+44`,
and unlike the mdb path it is not something a bare adb-shell process can satisfy: the
accessors need the full MI/Utopia bring-up, which only a real MApi client performs.

**Note on the DSP-alternative grant (§24):** the byte `0x80` at DSP `0xB000001D` would satisfy
all four licence gates without consulting the SE, and the host does hold a direct mapping of
decoder memory (`g_virDecR2shm = PA2KSEG1(MAD_base + 0xE01000)`). That path was never tested
and remains the one *theoretically* open lever — but reaching it needs a process where the
audio framework is already up (i.e. injection into the running `cpu_audio`), not a shell
process. That is a materially riskier technique than anything attempted so far, and it is not
promised to work.

### Honest position

Four independent routes are now closed by measurement, not by argument: the IP bitmap, the SE
flag window, the R2 mailbox, and now the direct register accessors. The one thing I will not
do is claim the problem is *impossible* — only that **nothing reachable from an adb shell has
moved it**, and that the remaining lead (DSP-side grant, §24) requires process injection into
the audio daemon.


## 48. The re-check that mattered: the real audio library has the DTS path

Asked to re-verify rather than close. That was right, and it corrected two things I had
concluded too early.

### 48.1 The `Abs*Reg` accessors are real, but need framework state

`libutopia.so` exports `HAL_AUDIO_AbsReadReg` @0x38A5D0, `AbsWriteByte` @0x38AD14,
`AbsWriteMaskByte` @0x38AF24, `AbsWriteMaskReg` @0x38B1E4, `GetDSPalive` @0x3962F8 — so the
mdb 8-bit offset limit was never the real barrier. With an AUDIO instance opened on
`/proc/utopia` (rc=0) the call still **segfaults inside libutopia**: the surrounding MI/Utopia
state is not initialised, and only a real MApi client brings it up. A bare adb-shell process
cannot satisfy that. So this route is closed, but for the framework reason, not the address
reason I predicted.

### 48.2 The audio stack has its own Utopia — and it is where the DTS path lives

`/vendor/lib/libutopia.so` is the **app-facing** copy (VLC, settings). The audio middleware
runs entirely from the tvservice rootfs:

```
/mnt/vendor/tvservice/glibc/libapiAUDIO.so    9 027 712 B   ← the real one
/mnt/vendor/tvservice/glibc/libmi.so, libmi_client.so, libmi_server.so
/mnt/vendor/tvservice/glibc/libdrvIPAUTH.so, libdrvSYS.so, libdrvMMIO.so
```

Pulled `libapiAUDIO.so` from the device. It carries the complete DTS path, exported and
resolved:

```
HAL_MAD_SpecifyDigitalOutputCodec    @0x8700C  (512 B)   ← "specify the digital-out codec"
HAL_MAD_GetDTSInfo                  @0x85C10  (580 B)
HAL_MAD_GetDtsInfo                  @0x861D4  (128 B)
HAL_MAD_SetDTSCommonCtrl            @0x85AE8  (296 B)
HAL_MAD_GetAudioCapability          @0x7E214  (124 B)
HAL_MAD_SetAudioOutputDeviceSelection @0x86D7C (656 B)
g_DSPMadBaseBufferAdr               @0x8ABC00  (.bss)   ← cached DSP base
```

`HAL_AUDIO_GetDspMadBaseAddr` @0x5E154 is a **pure getter** (loads a cached global, no
framework call) — callable from a shell process; in ours it returns 0 because the audio
system is not initialised here.

### 48.3 Status of the DSP-side lever

The host holds a direct mapping of decoder memory
(`g_virDecR2shm = PA2KSEG1(MAD_base + 0xE01000)` in the module). If DSP `0xB000001C` can be
located in the host address space, the licence word could in principle be read — and, since
byte `0x80` there satisfies all four gates (§24) without consulting the SE, possibly written.

Address arithmetic for `libapiAUDIO.so` in `cpu_audio` (pid 499) is now solved:
load bias `0xAFB85000`, so `g_DSPMadBaseBufferAdr` = `0xB040C00`, inside the
`rw-p` segment `afc53000–b0431000`. **The read of that exact address returns I/O error while
neighbouring addresses in the same mapping read fine** (`afc54000`→0, `b0000000`→data,
`b0200000`→data, `b03ff000`→0, then `b040000`+ fails). So there is an unmapped/anon boundary
just below the target and the exact address is **not yet pinned** — this is unfinished, not
blocked.

**No claim is made that the lever works.** It is the one concrete remaining lead: locate the
word, read it, and see whether it is 0 (DTS refused) or 5 (DTS granted but routed elsewhere).


## 49. BREAKTHROUGH: the host-side DTS licence is ALREADY OPEN by default

### 49.1 The register, finally read

`MDrv_AUDIO_Get_DTS_License` gates on bit 7 of `0x112CF0`:

```
0x422b28  bl   HAL_AUDIO_AbsReadReg     ; 0x112CF0
0x422b2c  lsl  r0, r0, #0x18
0x422b30  and  r6, r5, r0, asr #31     ; r5 = 0xF   ->  bit 7 of the 16-bit value
0x422b64  ands r4, r6, r4              ; DTS = (IP mask) & r6
```

The mdb interface addresses this fine — bank `0x112C`, offset `0xF0`, both 8-bit — but
earlier attempts were lost because `debug_level=5` (left set by an earlier test) let the
Monitor flood dmesg faster than the ring buffer could be read (§44.1). With the log quiet:

```
Reg 0x112cf0 = 0x00
Reg 0x112cf1 = 0xFF        -> the 16-bit word is 0xFF00
Reg 0x112cf2 = 0x21
Reg 0x112cf3 = 0x0F
```

**0xFF00 → bit 7 = 1 → the host grants DTS.**

### 49.2 It is the default, not our doing

```
with our all-FF bitmap uploaded :  0x112CF0=0x00  0x112CF1=0xFF
after reboot (bitmap cleared)   :  0x112CF0=0x00  0x112CF1=0xFF     <- identical
```

So the value is the power-on state, **not** something `e1_upload` produced.

### 49.3 What this corrects, and why it matters

This **falsifies my own §34 reading** in the most consequential way possible:

* DTS decoding works (`format : DTS, play state : PLAY`).
* The host-side licence gate **passes by default**.
* Therefore our IP bitmap **was never the missing piece for DTS** — consistent with §23, where
  a correctly provisioned, correctly timed bitmap changed nothing.
* The only remaining blocker is the **DSP-side gate** — `*(u16*)0xB000001E == 5` (or the
  `0xB000001D` alternative), a register the firmware only reads and the SE writes.

**This is the narrowest the target has ever been.** Everything host-side is now known-good;
one 32-bit word on the decoder side stands between us and DTS passthrough. And §24 already
established what satisfies that word without the SE: **byte `0x80` at `0xB000001D`**.

The remaining obstacle is purely mechanical — locating that word in the host's address space.
`g_DSPMadBaseBufferAdr` (libapiAUDIO.so, live in `cpu_audio` at `0xB0400C00`) holds
`0xFE4780AD` — a **kernel** pointer; the base is at `+0x90` of that struct, which
`/proc/PID/mem` cannot reach. So the next task is to get the MAD base out of the kernel
struct, not to re-litigate the licence.


## 50. The DSP gate is not reachable from userspace — three routes tried

Given the host side is now known-good, the only remaining object is the 32-bit word the DEC
firmware reads at DSP `0xB000001C` (`*(u16*)0xB000001E == 5`, or byte `0xB000001D == 0x80`).
Three ways to reach it were attempted:

**1. A device node — none exists.**
```
/dev/mem            : No such file or directory
/dev/mtk_mad (MAD)  : not present
sysfs DSP entries   : DSP2MIPS_INT, DSP_MIU_PROT_intr, MB_DSP2toMCU_INT0/1, SE_DSP2UP_intr
```
No direct path to decoder memory from a shell process.

**2. The driver's own parameter interface — decoded, and it does not reach.**
`HAL_DEC_R2_Set_SHM_PARAM` @0x45A434 dispatches 0xD1 parameters, each to a fixed SHM offset.
Decoding the offsets from the ARM handlers (59 of 0xD1 recovered) gives a contiguous band:

```
0x1014 0x101C 0x1024 0x1028 0x102C 0x1030 0x1034 … 0x11C8
```

**The smallest is 0x1014. Nothing at or below 0x1C.** So if DSP `0xB0000000` is the SHM base,
the gate at `shm+0x1C` is not writable through this interface.

**3. The cached pointer — points into the kernel.**
```
g_DSPMadBaseBufferAdr  (libapiAUDIO.so, live in cpu_audio @0xB0400C00)
  = ad 80 47 fe …  -> 0xFE4780AD   ← a KERNEL address
```
`HAL_AUDIO_GetDspMadBaseAddr` dereferences it and reads `+0x90`. `/proc/PID/mem` cannot reach
kernel memory, and `/proc/kcore` does not exist on this device.

**Status: the word is identified, its value on the DSP side is unmeasured, and no userspace
write path to it has been found.** The DTS gate therefore remains the single open object —
now much better characterised than at any earlier point: everything upstream of it is known
to be satisfied.


## 51. Both self-checks resolved

### 51.1 The 8-bit vs 16-bit doubt — RESOLVED, the reading was correct

`HAL_AUDIO_AbsReadReg` in the module:

```
0x4413f4  movw r1,#0 ; movt r1,#0xffe0 ; add r0,r1,r0,lsl #1
0x441400  ldr  r1,[_gMIO_MapBase] ; bic r0,r0,#2 ; add r0,r1,r0
0x441414  ldrh r0, [r0]        ← 16-BIT halfword load
0x441418  bx   lr
```

**It returns 16 bits.** The mdb read gave bytes `0x112CF0=0x00`, `0x112CF1=0xFF`, so the
returned value is `0xFF00`, and the DTS check computes

```
0xFF00 << 24 = 0xFF000000 ; asr #31 = 0xFFFFFFFF ; AND 0xF = 0xF
```

**`r6 = 0xF` — the host-side DTS licence check PASSES on this projector.** The doubt in §50 is
resolved in favour of the §49 reading, and the earlier worry that the whole avenue collapses if
`AbsReadReg` were byte-wide is eliminated.

Corroboration that mdb and the module address the same registers: we wrote
`0x112A84 = 0x84` through mdb and the module's `SET_IPAUTH_GROUP` value stuck (§33.2).

### 51.2 The parameter sweep — no small offset exists

Re-decoded all 0xD1 handlers with a broader matcher (74 recovered, up from 59):

```
offset range : 0x1014 .. 0x11C8
parameters below 0x200 : NONE
param 0x30 -> 0x1034     param 0x31 -> 0x104C
```

**Confirmed: no `HAL_DEC_R2_Set_SHM_PARAM` parameter writes below 0x1014**, so if DSP
`0xB0000000` is the SHM base the gate word at `shm+0x1C` is not reachable through the
driver's own parameter interface. The partial recovery (74/209) does not threaten this
conclusion: every recovered offset is ≥0x1014 and they form one contiguous control band, so a
lone parameter at 0x1C would be inconsistent with the table's structure.


## 52. `HAL_MAD_SpecifyDigitalOutputCodec` — the MAD state is in userspace memory

`libapiAUDIO.so` @0x8700C (512 B), the real audio library:

**a key/magic check first**
```
0x87050  ldr  r3, [r6]            ; first word of the caller's struct
0x87054  ldr  r1, [r2, #0x4E8]    ; a constant held in the MAD state
0x87058  eor  ip, r3, r1
0x8705c: lsr  ip, ip, #0x10        ; only the top 16 bits must agree
0x87064  cmp  ip, #0
0x87068  beq  +                    ; match -> proceed
0x8708c  mvn  ip, #0               ; no match -> -1
```

**then it writes the codec into the MAD state**
```
0x870b8  ldr  r1, [r6, #4]         ; caller's second word
0x870bc  ldr  r3, [r2, #0x4EC]
0x870e0  bl   0x150A0              ; derives something into sp+0x10
0x870f0  cmp  r3, #5               ; <<< the value 5 again
0x870f4  cmpne r3, #2
0x87100  cmp  r5, #1               ; r5 = arg1, the codec selector
0x8710c  cmp  r5, #2
0x87114  ldr  r3, [r8]             ; MAD state pointer
0x87118  ldr  r2, [sp, #0x18]
0x8711c  str  r2, [r3, #0x4F8]     ; <<< WRITE into the MAD state
```

**Two things follow.**

1. **`MAD_state + 0x4F8` is a codec/verdict word in *userspace* memory of the audio
   process**, reached through ordinary GOT-relative globals — i.e. it is inside
   `cpu_audio`'s address space and therefore **readable and writable through `/proc/PID/mem`**.
   The DSP-side gate we could not reach is not the same object as this one; this one we can.

2. The constant **`5`** appears here as well, next to `2`, as a *derived* value — the same
   number the DEC firmware gate compares against. Worth confirming whether `+0x4F8` is what
   the firmware ultimately sees, because if it is, the whole problem reduces to one word in a
   process we can already write to.

**Not yet done:** locating the MAD state pointer inside `cpu_audio` (it is reached through
GOT-relative indirection, so the static file does not name it), and reading `+0x4F8`.

