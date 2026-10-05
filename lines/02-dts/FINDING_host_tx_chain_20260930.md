# FINDING — 2026-09-30 (late): the HOST side of the TX chain, fully named, and the IPAUTH line must be RE-OPENED

**Source: `patch_baseline/utpa2k_stock.ko` — the kernel module is NOT stripped (59 896 symbols).
This is now the project's primary instrument. Everything below is symbol-verified ARM32.**

---

## 1. The host TX-arming chain, end to end

```
codec change
  └─ MDrv_AUDIO_SYSTEM_Control            @0x411C44   (the codec dispatcher)
       ├─ calls MDrv_AUDIO_Get_Decoder_Support(arg = r6+17)   [0x4126C0]
       └─ writes [g_AudioVars2+0xC]                    (the "system" word)
  └─ MDrv_AUDIO_SetDecodeSystem           @0x424E78
       └─ HAL_AUDIO_SPDIF_SetMode([g_AudioVars2+0xC], [g_AudioVars2+0x1CC])   [0x424F4C]
  └─ HAL_AUDIO_SPDIF_SetMode              @0x448A10   (arg mostly ignored;
       │                                               switches on [g_AudioVars2+0x1C4], 7-way table)
       └─ tail-call HAL_AUDIO_SPDIF_ApplySetting             [0x448C40 JUMP24]
  └─ HAL_AUDIO_SPDIF_ApplySetting         @0x44678C (≈5.6 KB)
       ├─ head reads [g_AudioVars2+0x4F8]  ← THE codec word from the memory note; first read ever mapped
       ├─ calls HAL_AUDIO_Get_CodeTypeByDecodeID (0x4446C0)
       ├─ HAL_SND_R2_Set_SHM_PARAM(110, …) and (157, …)        [0x446900/0x446914]
       └─ 8-WAY SWITCH on [g_AudioVars2+0x25FC]  @0x446EEC..0x446F24
            case 1 → mute SPDIF + HDMI_ARC      (0x446F28)
            case 3 → AbsWriteMaskByte ×8 (channel status block)
            case 4 → SRM-timer path             (0x447064)
            case 5 → HAL_AUDIO_SPDIF_Tx_SetNonPCM([g_AudioVars2+0xFF])   @0x4470D8
                     ← THE ONLY non-PCM TX arming site in the module
            cases 0/2/6/7 → other mode arms
```

`[g_AudioVars2+0x25FC]` is written **only inside ApplySetting itself** (the case arms) and in
`HAL_AUDIO_ResetDefaultVars` — the transition PCM↔BYPASS is decided by the callers above.
`spdif_mode`'s readback line `[User: …] [Driver: …]` almost certainly reflects this word.

## 2. The licence chain — every link named

```
MDrv_AUDIO_Get_DTS_License              @0x422AA0
  ├─ MDrv_AUTH_IPCheck(15)  → 0x422AAC
  ├─ MDrv_AUTH_IPCheck(58)  → 0x422AC0     ← exactly the 4 IPAUTH bits from the 2026-09-27
  ├─ MDrv_AUTH_IPCheck(18)  → 0x422AD0        provisioning work (7/15/18/58)
  ├─ MDrv_AUTH_IPCheck(7)   → 0x422AE0
  ├─ HAL_AUDIO_AbsReadReg(0x112CF0) → 0x422B28   ← the SAME register measured 0xFF00 (bit7=1)
  └─ returns {IP15,IP58,IP18,IP7} AND bit7-of-top-byte(0x112CF0)

MDrv_AUTH_IPCheck(bit)                  @0x001390C
  └─ reads byte array at [gIpAuthVars+0xA], indexed from the END by bit/8 —
     the uploaded 140-byte IPAUTH bitmap. gIpAuthVars is referenced by EXACTLY
     TWO functions in the whole module:
        MDrv_AUTH_IPCheck  (read)
        utopia_proc_ioctl  @0x13D58  (write)  ← the upload lands via the utopia proc ioctl

MDrv_AUDIO_Get_Decoder_Support          @0x41F72C
  ├─ MDrv_AUDIO_Get_MPEGH_License       (0x41FAB4, same pattern)
  ├─ MDrv_AUDIO_Get_DTS_License         (0x41FB20) → cmp r0,#0; beq 0x41FD00
  └─ licence==0 → SUPPORT=0 **UNLESS** ([sp] & 0x1A002000) != 0   [0x41FD00..0x41FD14]
     (a capability bitmask escape: bits 13,19,21,23,27,28 of a local flags word)
```

## 3. THE REFRAME — the IPAUTH line was killed by void evidence

The 2026-09-29 conclusion "provisioning is falsified as the lever" rested on
"DTS npcm = 0 after provisioning". **Every npcm measurement of that era is now void**:
Kodi's `Player.Stop` was silently failing (no `playerid`) and `[Audio Mon Task]` was wedged
in state `D` — the same defects that produced today's false zeros. The clean re-test of the
provisioning hypothesis **has never been run**.

What survives from that era: `UtopiaSetIPAUTH` rc=0 (userspace return value only — it does
NOT prove the kernel bitmap landed), `MApi_AUTH_Process` never running, `u8Customer_info`
all zeros (a userspace copy). The kernel-side `gIpAuthVars` bitmap — the thing
`MDrv_AUTH_IPCheck` actually reads — was **never observed**.

Note: the `dtsorder` boot service runs `vendup` (→ `UtopiaSetIPAUTH`) on every boot with
rc=0, so every DTS measurement made after a boot was nominally "provisioned" — yet the
bitmap may never have reached `gIpAuthVars` if libutopia's ioctl path differs from what
`utopia_proc_ioctl` expects.

## 4. The decisive tests (device was offline when attempted; first actions next session)

1. **`spdif_mode` readback during playback, per codec.** AC-3 → expect
   `Driver:SPDIF_OUT_BYPASS`; DTS → if `Driver:SPDIF_OUT_PCM` persists, the block is proven
   to sit in this exact host chain, above the TX arming.
2. **Verify the kernel bitmap directly.** `utopia_proc_ioctl` @0x13D58 shows the ioctl that
   writes `gIpAuthVars`; read it back through the same path (or an mmap-free accessor) to
   check whether bits 15/58/18/7 are actually set in the running kernel.
3. **Clean re-test of provisioning**: upload the all-FF bitmap, confirm it landed (test 2),
   then DTS via Kodi with the R2 log running (`play=1`, `state=3,0`), ES dump, and the
   Pioneer as the final oracle.
4. Static follow-ups: SYSTEM_Control's support→`[0xC]` mapping; what fills `[sp]` for the
   `0x1A002000` escape; whether `HAL_AUDIO_SPDIF_SetMode`'s `[+0x1C4]` switch can force
   BYPASS for DTS (a second candidate patch site, host-side).

## 5. Relationship to the DSP-side work

Both chains are now live and must be reconciled by measurement, not preference:
* DSP: pump latch `[+0x4EE4]`, B-latch `[+0x0]/[+0x10]` — gates the DEC output stage
  (`MUSE_TASK_b_latch_20260930.md`, assigned to Muse).
* Host: IPAUTH bitmap → Get_Decoder_Support → `[g_AudioVars2+0xC]` → SPDIF mode →
  `Tx_SetNonPCM` — gates the TX arming.

The host chain is the easier target (kernel module, no DSP-image signing problem; the
module loads patched images, and the `.ko` can be patched like any ELF — though the module
is part of the boot image and must be repacked). A "support=1 despite licence=0" patch at
`0x41FB28` (`cmp r0,#0` / `beq`) or forcing the `0x1A002000` escape bits are the two
narrowest host-side candidates — both untested, Pioneer-oracle rules apply.
