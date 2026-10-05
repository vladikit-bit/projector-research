# TD98 Pro / C50A — Static SPDIF Mode, EDID Gate, and Hidden-Menu Map

**Date:** 2026-09-25  
**Revision note:** amended after independent native-command and EDID-ownership passes; a direct `MApi_AUDIO_SYSTEM_Control` mode writer was verified, the HDMI-TX sink cache was corrected to `_u32CurEdidSupportList` rather than `_astHdmiInfo`, and `ree_payload.bin` is user-confirmed out of scope. OSD/factory/GAM reachability remains unproven.  
**Scope:** read-only static analysis of the local TD98/C50A corpus.  
**Device:** not accessed in this pass. No ADB, runtime, playback, reboot, settings write, module reload, firmware write, or patch deployment was performed.  
**TCL/T615:** not searched and not used.  
**Preservation:** no existing report, log, JSONL, Ghidra project, binary, firmware image, or configuration was edited, renamed, deleted, or overwritten.

## 0. Verdict

The projector's **Auto / PCM / Raw-BYPASS / Transcode vocabulary is real in the HAL**, a real factory-IR key-filter and generic TV HIDL bridge exist, and `utpa2k` has a direct native `SYSTEM_Control` mode writer. However, the available static corpus does **not** contain a verified hidden-engineering-menu command that reaches DTS capability state, the HDMI-TX sink EDID bitmap, or the authoritative EDID-monitor decision.

The decisive output gate remains the MI EDID-driven path:

```text
sink HDMI-TX EDID bitmap
    -> _MI_AOUT_SetHdmiAutoMode
    -> HDMI-TX mode argument r6
    -> SPDIF mode argument r4
    -> MApi_AUDIO_SPDIF_SetMode
    -> HAL_AUDIO_SPDIF_{Pcm,Auto,Bypass,Transcode}Mode
```

For the DTS-family arm of `SetHdmiAutoMode`:

```text
EDID DTS-HD bit 0x800 present  -> HDMI-TX r6=3, SPDIF r4=3 (BypassMode)
EDID DTS-core bit 0x080 present -> HDMI-TX r6=1, SPDIF r4=2 (AutoMode)
neither bit present             -> HDMI-TX r6=0, SPDIF r4=0 (PcmMode)
```

The normal projector RX EDID profiles selected by the board files contain LPCM/AC-3/E-AC-3 descriptors but no CTA format `0x07` DTS-core descriptor. A separate stored legacy EDID family does contain DTS, but it is an **RX-presented** profile and is not statically connected to the HDMI-TX sink bitmap consumed by `SetHdmiAutoMode`.

The smallest previously identified code candidate is therefore still only a candidate:

```text
mik_stock.ko, file offset 0x9B54C, VA 0x98A58
00 60 A0 E3   mov r6,#0        (DTS-core absent)
-> 01 60 A0 E3   mov r6,#1     (force the DTS-present sub-mode)
```

It was **not applied**. Even if applied, it would only remove this one EDID-derived PCM decision. It would not repair a decoder/license state, fill `0x582`, inject compressed frames, bypass every HAL support clamp, or prove that a physical receiver locks DTS.

## 1. New independent findings and corrections

### 1.1 The old mode labels were not reliable

`tools/mik_SetHdmiAutoMode.asm:4` labels the arguments as `2=BYPASS, 3=TRANSCODE`. The stock `utpa2k_stock.ko` relocation and jump-table evidence resolves the actual SPDIF dispatch as:

| Numeric mode | Worker reached by `HAL_AUDIO_SPDIF_SetMode` | Evidence |
|---:|---|---|
| 0 | `HAL_AUDIO_SPDIF_PcmMode` | jump entry `0x448b60`, call relocation `0x448b64` |
| 1 | `HAL_AUDIO_SPDIF_PcmMode` | jump entry `0x448c00`, call relocation `0x448c04` |
| 2 | `HAL_AUDIO_SPDIF_AutoMode` | jump entry `0x448b6c`, call relocation `0x448b70` |
| 3 | `HAL_AUDIO_SPDIF_BypassMode` | jump entry `0x448b78`, call relocation `0x448b7c` |
| 4 | `HAL_AUDIO_SPDIF_TranscodeMode` | jump entry `0x448b84`, call relocation `0x448b88` |

Some older API notes label numeric mode `1` as a raw/DD-raw request. In this stock module's actual dispatch table it reaches `PcmMode`; that older label is not used as evidence of a raw path.

The same binary's `sSpdifModeNameToEnumTable` at VA `0x16A8D0` resolves as:

```text
SPDIF_OUT_PCM       = 0
SPDIF_OUT_AUTO      = 2
SPDIF_OUT_NONPCM    = 2   (numeric alias in this table)
SPDIF_OUT_BYPASS    = 3
SPDIF_OUT_TRANSCODE = 4
SPDIF_OUT_NONE      = 5
SPDIF_OUT_MULTIPCM  = 6
```

At `0x448A10`, input `5` is normalized to `0` by `subs r4,r0,#5; movne r4,r0`; therefore a name-table value of `NONE=5` reaches the PCM worker in this implementation. The old `2=BYPASS / 3=TRANSCODE` wording is therefore not used in this report.

The companion HDMI-TX name table in the same `utpa2k_stock.ko` is `sHdmiTxModeNameToEnumTable` at VA `0x16A908` (file offset `0x7C3BA8`):

```text
HDMI_OUT_PCM        = 0
HDMI_OUT_NONPCM     = 1
HDMI_OUT_8CH_NONPCM = 2
HDMI_OUT_BYPASS     = 3
HDMI_OUT_TRANSCODE  = 4
```

No separate native HDMI-ARC name table was found. ARC uses HDMI/eARC APIs, so an exact ARC enum alias is not proven. This distinction matters: `SetHdmiAutoMode` passes `r6` to the HDMI-TX API and `r4` to the SPDIF API; the two numeric domains must not be conflated.

The dedicated string parsers add an important qualification:

```text
MDrv_AUDIO_DigitalOut_ParseSpdifOutputMode @ 0x41F168
  PCM=0, AUTO=2, BYPASS=3, TRANSCODE=4, unknown/default=5

MDrv_AUDIO_DigitalOut_ParseHdmiTxOutputMode @ 0x41F200
  PCM=0, AUTO=1, BYPASS=1 (alias to HDMI_OUT_NONPCM),
  TRANSCODE=4, unknown/default=0

MDrv_AUDIO_DigitalOut_ParseOutputType @ 0x41F05C
  AC3=1, AC3P=2, AAC/MAT=3, NONE=4, ATMOS=5, DTS=6,
  numeric 200/210/202/212 -> 7
```

The HDMI parser's `BYPASS=1` result conflicts with the generic HDMI name table's `BYPASS=3`. This is a real vendor-domain alias/inconsistency. A UI label must therefore be traced through the parser that receives it; the name table alone is not a complete UI-to-hardware map.

### 1.2 CTA format correction

`REPORT_static_final.md:307` calls CEA format `0x0A` “DTS-HD.” The standard mapping is:

```text
0x0A = E-AC-3
0x0B = DTS-HD
```

This does not change the directly tested DTS-core bit `0x80`; it only corrects the DTS-HD label used in older narrative text.

## 2. Exact MI gate map

### 2.1 Codec dispatch

The stock `mik_stock.ko` dispatch table at VA `0x989E8` maps codec IDs as follows:

```text
codec 4       -> 0x98A88
codec 5       -> 0x98B3C   (AC3 arm)
codec 6       -> 0x98B8C
codec 7       -> 0x98A88
codec 8       -> 0x98BA8
codec 9       -> 0x98A38   (DTS family)
codec 10      -> 0x98A38   (DTS family)
codec 11      -> 0x98A38   (DTS family)
codec 12..22  -> 0x98B8C
codec 23      -> 0x98A38   (DTS family)
```

The `0x98A58` default is therefore not in the codec-5 AC3 arm. It is in the arm used by codec IDs 9, 10, 11, and 23.

### 2.2 DTS-core absent/present branches

Source listing: `spdif_audio_investigation/tools/mik_SetHdmiAutoMode.asm`.

```text
0x98A38  tst r0,#0x800       ; DTS-HD capability test
0x98A44  tst r0,#0x080       ; DTS-core capability test
0x98A50  bne 0x98C04         ; DTS-core present
0x98A54  mov r5,#1
0x98A58  mov r6,#0           ; DTS-core absent default
...
0x98C04  mov r5,#7
0x98C08  mov r6,#1           ; DTS-core present
...
0x98C64  cmp r6,#1
0x98C68  bne 0x98C74
0x98C6C  mov r4,#2           ; r6=1 -> SPDIF AutoMode argument
0x98C74  mov r4,#0           ; otherwise PCM argument
```

The DTS-HD branch at `0x98AD4` sets `r6=3`; the common tail later converts that to `r4=3` at `0x98DCC`.

The tail is explicit about the two different APIs:

```text
0x98DD0  MApi_AUDIO_HDMI_TX_SetMode(r6)
0x98DDC  MApi_AUDIO_SPDIF_SetMode(r4)
0x98DE8  str r6,[_stAoutSndParam,#0x7C]
0x98DEC  str r5,[_stAoutSndParam,#0x80]
0x98DF0  str r6,[_stAoutSndParam,#0x6C]
0x98DF4  str r5,[_stAoutSndParam,#0x70]
```

Thus the old shorthand “r4=2 means bypass” is too imprecise. The stock SPDIF worker for `r4=2` is `AutoMode`; `r4=3` is `BypassMode`.

### 2.3 Source of the sink capability bitmap (correction)

The verified HDMI-TX sink-side state used by the monitor is `_u32CurEdidSupportList`, not `_astHdmiInfo`:

```text
_MApi_HDMITx_GetDataBlockLengthFromEDID
_MApi_HDMITx_GetRxAudioFormatFromEDID
  -> _u32CurEdidSupportList
  -> _MI_AOUT_MonitorTask / _MI_AOUT_HdmiInfoMonitor
  -> _MI_AOUT_SetHdmiAutoMode
```

`_astHdmiInfo` is a set of four RX/EXTIN records at `0x39CD0`; direct writes around `0x17EEA4/0x17EF78` accompany the external `MI_DISP_IMPL_XC_HDMIRx_SetEDID` call at `0x17EF00`. It must not be treated as the HDMI-TX sink cache. This corrects the symbol ownership used in older narrative text.

The consumption of bits `0x4`, `0x80`, `0x400`, `0x800`, and `0x1000` by `SetHdmiAutoMode` is verified. The producer is now also closed in the monitor listing: `ubfx r2,r2,#3,#4` extracts the CEA format code and `orr r8,r8,ip<<r2` sets the corresponding bit in `_u32CurEdidSupportList`; the resulting list is passed to `SetHdmiAutoMode` at `0x97C30`. The remaining unknown is which external provider/runtime-selected EDID supplies the bytes, not the bit-index operation.

## 3. What the HAL mode menu can and cannot do

`audio.primary.mt5889.so` contains the following output vocabulary:

```text
SetSpdifOutputMode=PCM
SetSpdifOutputMode=AUTO
SetSpdifOutputMode=BYPASS
SetSpdifOutputMode=TRANSCODE
SetSpdifOutputType=AC3
SetSpdifOutputType=DTS
SetSpdifOutputType=NONE
SetHdmiTxOutputMode=...
SetHdmiArcOutputMode=...
SetHdmiArcOutputType=...
```

This proves that the vendor HAL has a digital-output request model. It does **not** prove that the projector's OSD reaches that model or the separate native `SYSTEM_Control` writer, nor that the request changes the sink EDID bitmap.

The HAL's exact request-key strings are present in `audio.primary.mt5889.so`:

```text
hdmi_tx_mode       file offset 0x14BDE
hdmi_tx_type       file offset 0x163B0
hdmi_arc_mode      file offset 0x176BC
hdmi_arc_type      file offset 0x19820
spdif_mode         file offset 0x18940
spdif_type         file offset 0x1BB87
```

The static parser evidence identifies `spdif_type` as the live type key; the `spdif_mode` string has no setter xref in the inspected HAL. The parsed type values are:

```text
0x09000000 = AC3
0x0A000000 = E-AC3
0x0B000000 = DTS
0x0C000000 = DTS-HD
```

The corresponding literal log strings include `SpdifOutputMode=BYPASS` at `0x1BC50`, `SpdifOutputType=DTS` at `0x1BC6A`, `SpdifOutputMode=AUTO` at `0x1C0DF`, `SpdifOutputMode=PCM` at `0x18F6C`, and `SpdifOutputMode=TRANSCODE` at `0x1CA08`.

The static body of `MI_AOUT_SetDigitalMode` (`mik_stock.ko`, VA `0x7FAF8`) writes the passed digital-mode structure at `+0x4`, `+0x8`, and `+0xC` (`tools/final_digitalmode.txt:167-175`). It does not write `_u32CurEdidSupportList` or the MI state offsets `0x6C/0x70/0x7C/0x80`.

`libmi3.so` exports `MI_AOUT_SetDigitalMode` at dynamic VA `0x6049C` and `MI_AOUT_GetDigitalMode` at `0x605C1`. The HAL-side callers are `utils_ApplyDigitalOutputSetting` at VA `0x28685` and `utils_get_digital_output_mode` at `0x27FBD` in `audio.primary.mt5889.so`. This is the desired-state path, not the final EDID writer.

The available TV API service additionally exposes generic factory-bridge entry points:

```text
a_mtktvapi_gam_client_send_cmd   file offset 0x6AB0
a_mtktvapi_get_audio_apd_text  file offset 0x6AD0
a_mtktvapi_get_base_config_file_path file offset 0x6AF0
GAM log text                    file offsets 0x18440..0x184B0
MTCEC_GetEARCStatus/SetEARCEnable strings around 0xF0F0..0xF120
```

The GAM path parses a fixed `HIDL_MTKTVAPI_GAMCMD_INFO_T` command buffer and dispatches through a service vtable. Static strings show generic command/parameter plumbing, but no SPDIF/DTS/BYPASS/RAW mode enum. No xref connects that generic bridge to `MI_AOUT_FactorySetAttr`, the EDID bitmap, or DTS AUTH state. It is a credible place to inspect if the missing factory executable becomes available, not a proven unlock today.

Therefore:

```text
Android HAL menu request -> desired digital-mode structure
native SYSTEM_Control   -> direct SPDIF/ARC mode command
EDID monitor            -> normal sink-capability-derived mode
```

A menu item can be real and still fail to make the DTS gate pass if it only changes the first arrow. A factory command can also be real while targeting IR filtering or communication-audio attributes rather than DTS capability. The native command path is the newly identified bridge candidate.

### 3.1 Secondary HAL gates

Even after the MI gate chooses a non-zero mode, `HAL_AUDIO_SPDIF_SetMode` has additional conditions:

1. `_MApi_AUDIO_SPDIF_SetMode` and `MDrv_AUDIO_SPDIF_SetMode` have an initialization/version branch when `g_AudioVars2+0x4C8 >= 3`; the inspected bodies do not by themselves prove a license rejection.
2. The worker reads byte `g_AudioVars2+0x25B7` at `0x448ABC`.
3. If that byte is set and the per-output support table satisfies the checked condition, the code forces the mode back to `0` (PCM).
4. `AutoMode` separately reads `g_AudioVars2+0x582`; when it equals `1`, it writes `0x583` for DTS:X output formatting.

These are real static conditions, but the local corpus does not establish their live values. They are not hidden-menu commands.

### 3.2 Direct native system-command writer — correction

The earlier conclusion that there was no direct native mode writer was too broad. `utpa2k_stock.ko` contains a second command surface:

```text
MApi_AUDIO_SYSTEM_Control     VA 0x3FE754, size 0x184
_MApi_AUDIO_SYSTEM_Control   VA 0x3EE684, 4-byte tail jump
MDrv_AUDIO_SYSTEM_Control    VA 0x411C44, size 0x1088
```

`MApi_AUDIO_SYSTEM_Control` formats the caller string and sends `UtopiaIoctl` command `0xE5` at `0x3FE81C`. `_MApi_AUDIO_SYSTEM_Control` tail-relocates to `MDrv_AUDIO_SYSTEM_Control`, whose stock parser directly recognizes:

The system-control parser has an early `g_AudioVars2+0x4C8` initialization/version branch before the mode dispatch, but the inspected body does not prove a license rejection there; it proceeds to the cached-mode write. The direct command is therefore not a proven license bypass, and the actual `0x4C8`/support behavior must not be overstated as an Auth failure without a resolved target. The retained userspace copy `libutopia.so` does contain a non-trivial `MDrv_AUDIO_HDMI_TX_SetMode` at `0x3542E8` (mode dispatch 0/1/2/3/4), while its `HAL_AUDIO_HDMI_TX_SetMode` at `0x39745C` and the kernel copy at `0x44E108` are four-byte `bx lr` no-ops. This is a possible alternate implementation, but no caller, registration, or runtime replacement tying it to the kernel command path is statically closed. It remains an HDMI-TX override unknown, not evidence that the stock SPDIF path is fixed.

```text
SetSpdifOutputMode=
SetHdmiTxOutputMode=
SetHdmiArcOutputMode=
SetSpdifOutputType=
SetHdmiTxOutputType=
SetHdmiArcOutputType=
```

The direct writer behavior is:

```text
SPDIF mode:    PCM=0, AUTO=2, BYPASS=3, TRANSCODE=4, unknown/default=5
HDMI-ARC mode: PCM/AUTO/BYPASS/TRANSCODE = 0/2/3/4;
               unknown/default=5 is ignored at the state-store path
HDMI-TX mode:  PCM=0, AUTO/BYPASS=1 (alias), TRANSCODE=4
```

The SPDIF mode branch calls `MDrv_AUDIO_SPDIF_SetMode` at `0x4125E8`. That function (`0x40587C`) writes `g_AudioVars2+0x0C` and then tail-calls the HAL worker. The accepted HDMI-ARC mode branch also converges to the same `0x4125E8` SPDIF setter after storing ARC state at `g_AudioVars2+0x10`; its recognized values are PCM=0, AUTO=2, BYPASS=3, TRANSCODE=4, while unknown/default 5 is rejected. The HDMI-TX branch calls `HAL_AUDIO_HDMI_TX_SetMode` at `0x4124F4`, but that stock HAL symbol at `0x44E108` is only a four-byte `bx lr`, so the direct HDMI-TX call is a no-op unless a runtime callback replaces it. `HAL_AUDIO_SPDIF_SetMode` later calls `HAL_AUDIO_eARC_SetMode` at `0x448C34`, so an accepted SPDIF/ARC mode request can propagate an eARC request as well. Type state is stored at `+0x4F0` (SPDIF), `+0x4F4` (ARC), and `+0x4F8` (HDMI-TX).

The same public names are present in `libutopia.so` (`MApi_AUDIO_SYSTEM_Control` VA `0x34D3F8`, `_MApi` VA `0x33D424`, `MDrv` VA `0x35E1DC`), and `libmi3.so` imports the public MApi symbol. The userspace copy also exports `MApi_AUDIO_SPDIF_SetMode` (VA `0x344328`, ioctl `0x5E`) and `MApi_AUDIO_HDMI_TX_SetMode` (VA `0x34555C`, ioctl `0x6D`). The stock `mik.ko` has 32 direct `R_ARM_CALL` relocations to `MApi_AUDIO_SYSTEM_Control`; the known `MI_AOUT_Open` (`0x7DB2C`, `0x7DD10`, `0x7E368`) and `_MI_AOUT_MonitorTask` (`0x979FC`, `0x97AA4`, `0x97F84`, `0x9802C`) subset is type/mute/eARC/capability traffic, not `Set*OutputMode` calls. A raw string check finds `Set*OutputType=` strings but no `SetSpdifOutputMode=`, `SetHdmiTxOutputMode=`, or `SetHdmiArcOutputMode=` strings in `mik.ko`. Thus a native command/export surface really exists. What is still missing is the reachability proof from the projector OSD, factory service, or GAM HIDL to the mode strings.

This is a concrete bridge target and a correction to the earlier “no direct writer” wording. It is not yet a no-patch DTS unlock: the EDID monitor is the authoritative writer for normal sink-reconfiguration events, but a direct command invoked afterward can write SPDIF mode later; disable and DD-plus monitor paths can also reapply the stored mode. There is no unconditional single global last writer across all event orderings. Other observed reapply/disable sites include `MDrv_AUDIO_SetDecodeSystem` (`0x424F4C`), `HAL_AUDIO_HDMI_SetNonpcm` (`0x44B6A8`), `HAL_MAD_Monitor_DDPlus_SPDIF_Rate` (`0x460B30`), and output initialization in `MI_AOUT_Open`.

The cached state and the worker state must also be separated: `MDrv_AUDIO_SPDIF_SetMode` updates `g_AudioVars2+0x0C`, whereas `HAL_AUDIO_SPDIF_SetMode` updates worker fields `+0x14/+0x18` and arms/clears the nonPCM fields. `HAL_AUDIO_HDMI_SetNonpcm` can call `HAL_AUDIO_SPDIF_SetMode(0,0)` on its disable path without updating the cached `+0x0C`. Thus “last writer” is event- and layer-dependent. The direct command does not set DTS AUTH/decoder state.

The inspected `AutoMode`/`BypassMode` worker ranges have no direct relocation to `_MApi_HDMITx_GetEDIDData` or `_u32CurEdidSupportList`; they consume global audio/support state instead. `BypassMode` still calls `HAL_MAD_GetAudioInfo2`, so an active decoder/output state remains a prerequisite. That makes a reachable native command a plausible way to request nonPCM independently of the EDID monitor, but it does not remove support/decoder gates; direct AUTH behavior is not proven by this body, and physical DTS transport remains unproven.

## 4. Board configuration: route selection, not DTS unlock

The extracted board profiles contain a real SPDIF path and different ARC routing:

```text
BD_MT5889_H2V1-B4-S/board.ini
  line 840: BOARD_AUDIO_PATH_2 = 53   (SPDIF)
  line 842: BOARD_AUDIO_PATH_3 = 255  (HDMI TX disabled in this profile)
  line 850: BOARD_AUDIO_PATH_7 = 55   (HDMI ARC)

BD_MT5889_H2V1-B4-S_EWS/board.ini
  line 793: BOARD_AUDIO_PATH_2 = 53   (SPDIF)
  line 795: BOARD_AUDIO_PATH_3 = 55   (HDMI TX)
  line 803: BOARD_AUDIO_PATH_7 = 255  (HDMI ARC disabled in this profile)
```

The corresponding `Customer_Module.ini` files select ARC HDMI port `23` or `25`. `Board.h:230-232` sets:

```text
ROCKET_AUDIO_INPUT_MODE = EN_LTH_HDMITX_AUDIO_SPDIF
```

These constants describe input/path/ARC routing. They do not set RAW, BYPASS, PCM, DTS capability, or the sink EDID bitmap. The existence of an ARC route explains why an ARC menu can be visible without proving that compressed DTS is selected.

## 5. EDID profiles and the DTS bit

### 5.1 Selected RX profiles

The board EDID files select the `Mstar_EDID1..4` families, including the v2.0/v2.1 HDR and v1.4 frame-side profiles:

```text
.../board/BD_MT5889_H2V1-B4-S/edid_cfg.ini:1-6
.../board/BD_MT5889_H2V1-B4-S/model/Customer_1.ini:81-136
```

A read-only CTA-861 Audio Data Block parse of the stored `Mstar_EDID1..4` files found:

```text
format 1  = LPCM
format 2  = AC-3
format 10 = E-AC-3 in the newer profiles
format 7  = absent
```

The separate `Mstar_EDID_v1_3_1080p_1..4_*.bin` files do contain format `7`, but they are not referenced by the board `edid_cfg.ini` files examined here.

### 5.2 RX and TX are different directions (ownership correction)

Static symbols and relocation sites establish two separate EDID directions:

```text
RX / projector-presented:
  board EDID files
  -> MI_SYS_CfgLoadHdmiEdidInfo / MI_EXTIN_SetAttr
  -> MI_DISP_IMPL_XC_HDMIRx_SetEDID
  -> _astHdmiInfo (RX/EXTIN state)

TX / downstream-sink:
  _MApi_HDMITx_GetDataBlockLengthFromEDID
  _MApi_HDMITx_GetRxAudioFormatFromEDID
  -> _u32CurEdidSupportList
  -> _MI_AOUT_MonitorTask / HdmiInfoMonitor
  -> SetHdmiAutoMode
```

The prior shorthand that `_astHdmiInfo` was the HDMI-TX sink bitmap is corrected here. `_astHdmiInfo` references are concentrated in the RX/EXTIN region; the verified sink support list used by the monitor is `_u32CurEdidSupportList`. `dtv_driver.ko` separately exports `MTGVDECEX_HDMI_SetEdidData`, `MTGVDECEX_HDMI_SetEdidIndex`, and `MTGCEC_SetSAD`, which are RX/CEC EDID/SAD glue, not the TX sink-list writer.

A stored RX EDID profile can tell an external source what the projector accepts. It is not the same thing as the EDID read from the AVR/receiver on HDMI-OUT. No static call path was found that makes selecting the RX profile write `_u32CurEdidSupportList`.

`mi_aout_GetCurEdid` reads the same `_u32CurEdidSupportList` (`0x8C364` region), but the inspected relocation set shows no corresponding sysfs setter; the known write is the monitor update at `0x97BE4`. `Customer_SAD.ini`, `EdidBinPath`, and the `/sys/kernel/mik/MI_AUDIO` EDID-control strings exist in the binary, but the local extracted corpus does not contain `Customer_SAD.ini`, and no path from those controls to `_u32CurEdidSupportList` is closed. They remain configuration candidates, not proven menu unlock mechanisms.

## 6. Hidden engineering/factory menu audit

A native service named `vendor.mediatek.tv.mtktvfactory@1.0-service` was running in the retained historical log:

```text
spdif_audio_investigation/runtime_phase1/R1_baseline_logcat.txt:1031
spdif_audio_investigation/props_audio.txt:8,18,35
```

That establishes factory-service infrastructure, but not its menu semantics. The retained historical log identifies the actual third-menu activity:

```text
PackageName: mediatek.factorymenu.ui
MainActivity: mediatek.tvsetting.factory.ui.designmenu.DesignMenuActivity
```

Source: `spdif_audio_investigation/iec_logcat.txt:32-38` and repeated factory launches at `:166-172`. Therefore the user's third menu is indeed the MediaTek factory **DesignMenu**, not the ordinary Android audio settings.

Static `MI_AOUT_FactorySetAttr` handles factory audio/eARC/channel-status attributes (selectors `0x1001/0x1002`, nested value `7`, commands `0x8143/0x8146`), not a proven `SetSpdifOutputMode` command. Nearby `mik.ko` diagnostics include `Invalid Output source.`, `Set eARC channel status`, and `Does not support ePath:%d to set eARC instruction!`, which places the known factory setter in the eARC/channel-status family. `Non-linear adjust` and `Volume Tbl path` therefore remain most plausibly volume/amplifier controls; their exact APK-to-native mapping is not present in the local corpus.

Both available module configuration files contain an empty `[M_FACTORY]` section:

```text
.../board/BD_MT5889_H2V1-B4-S/module/Customer_Module.ini:79-80
.../board/BD_MT5889_H2V1-B4-S/module/MStar_Default_Module.ini:74-75
```

No factory feature assignment is present. The only factory configuration path found in the binary is the string `vendor/tvconfig/config/factory_keypad.ini` (`tools/final_edid.txt:491`); that file is absent from the extracted corpus. This is additional evidence that no configured factory-audio feature is exposed by the available image, though it does not exclude an unextracted service-side menu.

### 6.1 A real factory IR mode exists, but it is an IR filter gate

The `dtv_driver.ko` factory path is independently verified:

```text
_CLI_IR_CtrlKey          VA 0x13D0CC, size 0x64
_MTGIR_CtrlKey           VA 0xB6770, size 0x54
__GlueIR_ConfigCtrlKey   VA 0xB65E8, size 0x188
__GlueIR_ConfigBlockKey  VA 0xB6024, size 0x5C4
```

The control switch accepts case `4`. That branch loads the factory-key enable/invert bytes:

```text
bIRFactoryEnableBlockKey  VA 0x16CB40
bIRFactoryInvertBlockKey  VA 0x16CB42
```

and calls `__GlueIR_ConfigBlockKey` at `0xB66C4`. The embedded key `ir:m_pIrFactoryModeConfig_File` is present at raw file offset `0x1FE260`; the related usage strings identify control `4` as `factory`.

The downstream calls are IR calls (`MI_IR_SetAttr` relocations at `0xB6578`, `0xB66F8`, and `0xB6740`). No DTS, SPDIF, HDMI, ARC, or audio-mode call is present in this IR routine range. This is a credible explanation for a hidden **factory key filter**, but it is not an audio unlock.

The extracted `ir_config.ini` contains normal MENU/SETTINGS mappings and `KEY_FN_F8`/`KEY_FN_F9` profile/dashboard keys, but no factory-mode key table:

```text
.../board/BD_MT5889_H2V1-B4-S/ir_config.ini:35-53,80-105
```

### 6.2 Factory audio attributes are not a proven DTS switch

`MI_AOUT_FactorySetAttr` is a real exported factory API in `mik_stock.ko`:

```text
VA 0x8928C, file offset 0x8BD80, size 0x378
selector 0x1001 -> nested value must equal 7 -> command 0x8143
selector 0x1002 -> nested value must equal 7 -> command 0x8146
```

Both command values are passed to `MApi_AUDIO_SetCommAudioInfo` at the common call site `0x89534`. The inspected body contains no DTS literal and no decoder-start call. These are communication-audio factory attributes, not a proven DTS or RAW/BYPASS enable.

The nearest DTV wrappers do not close the gap either:

```text
MTGADEC_MTAUD_SetSPDIFEnable  VA 0x93A98
  bool + command 0x1200 -> MI_AOUT_SetAttr; enable/disable only

MTGADEC_MTAUD_SetAudioOutMode VA 0x93C90
  8-byte no-op: mov r0,#0; bx lr

MTGADEC_MTAUD_GetDtsVersion  VA 0x9A14C
  query command 0x5002; no mode write

MTGCEC_SetArcEnable          VA 0xACCE4
  ARC enable/port state only

MDrv_PM_STR_CheckFactoryPowerOnModePassword
  utpa2k_stock.ko VA 0x784F0; factory power-on password check
```

`media_codecs.xml:270-317` registers AC3, DTS, DTS-HD, DTS-HD-HRA/MA/LBR, and `OMX.MS.Passthrough.Decoder`. `audio_policy_configuration.xml:70-128,217-232` declares DTS/DTS-HD profiles, but the `Spdif` device port at lines 231-232 is empty. Codec registration and policy declaration are not a selected output mode.

### 6.3 Generic HIDL factory bridge is the strongest unresolved lead

The available TV API service exposes generic:

```text
a_hidl_tvapi_set_property
a_hidl_a_mtktvapi_gam_client_send_cmd
```

The GAM method parses a fixed `HIDL_MTKTVAPI_GAMCMD_INFO_T` buffer and dispatches through a service vtable. No SPDIF/DTS/BYPASS mode literal or mode enum was found in the available `mtktvapi.so` or service executable, and no xref connects this generic bridge to `MI_AOUT_FactorySetAttr`, the native `MApi_AUDIO_SYSTEM_Control` writer, `_u32CurEdidSupportList`, or DTS AUTH state. A raw string comparison finds the `Set*OutputMode=` vocabulary in `audio.primary.mt5889.so`, but not in `libmi3.so` or the available `mtktvapi.so`. The `libmi3.so` import/PLT relocation `0xB2624` is exercised from `mi_hwcaps_GetAudioCaps` (call at `0x45F14`) as a capability query, not as an output-mode setter. The service also depends on `vendor.mediatek.custom.mtktvapicustom@1.0-impl.so`, `libmediatek_tv_basic.so`, and `libmediatek_tv_rpcwrapper.so`, which are absent from the local extracted inventory. `C:\firmware_temp\ree_payload.bin` is user-confirmed to be a cut/undecryptable package fragment; it is explicitly out of scope for this audit and is not treated as a recoverable source. The available service itself is 312,920 bytes with SHA-256 `7a0c466e7d33a17522c64ad8697064a72faa32ca8f5ecd29d0b1228843db9783`.

If the missing `mtktvfactory` executable becomes available, this generic command bridge and the native `SYSTEM_Control` API are the first static targets to trace. It is not safe to infer a command ID or claim an unlock from the current corpus.

## 6.4 Which of the three menus matters

| Menu | Static match | Interpretation |
|---|---|---|
| Projector OSD: `Off`, `Raw/Arc`, `Pcm/Arc`, `Auto` | Values align with `SPDIF_OUT_PCM=0`, `AUTO=2`, `BYPASS=3` | Most likely direct SPDIF/ARC mode control; highest-priority menu for the native command question |
| Android audio settings: downmix, passthrough, DD, DD+ | Matches `audio.primary.mt5889.so` digital-output vocabulary and `MI_AOUT_SetDigitalMode` desired-state path | Android HAL/policy layer; does not statically write the sink EDID list |
| MediaTek `DesignMenu` factory menu | `DesignMenuActivity` is confirmed by retained log; known factory setter is eARC/channel-status oriented | Correct factory surface, but `Audio source/output` and volume items are not yet proven SPDIF-mode controls |

Therefore: the **first projector menu is the best candidate for the actual SPDIF mode bridge**; the third menu is the correct factory menu to inspect, but its known `MI_AOUT_FactorySetAttr` commands are eARC/channel-status controls rather than `SetSpdifOutputMode`. The second Android menu is the least likely to reach the native command directly.

## 7. No-patch assessment

### 7.1 What is proven

- The HAL recognizes the menu vocabulary `PCM`, `AUTO`, `BYPASS`, and `TRANSCODE`.
- The stock MI EDID gate can force SPDIF PCM when DTS-core bit `0x80` is absent.
- The selected project RX EDID profiles do not advertise DTS-core.
- The stored legacy DTS RX profiles are not the selected board profiles and are not the TX sink EDID.
- No verified menu/property path to the TX sink bitmap was found.
- A factory IR key-filter mode exists, but its calls are IR-only.
- `MI_AOUT_FactorySetAttr` exists, but its inspected commands are communication-audio attributes with no DTS literal or decoder-start call.
- `MTGADEC_MTAUD_SetAudioOutMode` is an 8-byte no-op, refuting it as a hidden RAW/BYPASS setter.
- A direct native `MApi_AUDIO_SYSTEM_Control` command writer exists and can request SPDIF modes; its OSD/factory/GAM reachability is unproven.
- The factory service exists, but its implementation is missing from the local corpus.
- Both extracted `[M_FACTORY]` module sections are empty, and `factory_keypad.ini` is absent.

### 7.2 What is not proven

- That a projector OSD `Raw/ARC` selection reaches `MI_AOUT_SetDigitalMode` on this exact image.
- That the UI's `Raw` label maps to numeric mode `3` rather than a vendor-specific request path.
- That a sink-side `Customer_SAD.ini`/EDID override can set `_u32CurEdidSupportList` bit `0x80`.
- That the generic GAM/property HIDL bridge forwards a factory command to `MI_AOUT_FactorySetAttr` or the native `MApi_AUDIO_SYSTEM_Control` writer.
- That the direct native command's SPDIF request survives the subsequent EDID-monitor reapplication.
- That forcing mode argument `2` produces a valid physical DTS stream or receiver lock.
- That stock decoder/license state is already sufficient for direct DTS on the currently deployed image.

### 7.3 Potential no-patch route newly identified

The direct `MApi_AUDIO_SYSTEM_Control` command surface means a no-binary-patch route is now statically plausible:

```text
vendor TV-settings/GAM command
  -> MApi_AUDIO_SYSTEM_Control
  -> ioctl 0xE5
  -> MDrv_AUDIO_SYSTEM_Control
  -> MDrv_AUDIO_SPDIF_SetMode
```

A command such as `SetSpdifOutputMode=BYPASS` could request SPDIF mode `3` through this path. The accepted `SetHdmiArcOutputMode=...` branch also converges to the SPDIF setter, so the projector's `PCM/ARC` and `Raw/ARC` vocabulary may be relevant to this native bridge. However, the caller chain from the actual projector OSD/factory service is still unproven, the command may be transient, and `_MI_AOUT_MonitorTask` can subsequently re-apply the EDID-derived mode. This is a **bridge to investigate**, not an instruction to execute and not a confirmed unlock.

## 8. Bounded patch targets — not applied

### Candidate A: restore the DTS-present MI sub-mode

Do not apply this candidate before the native command/EDID bridge is exhausted: the direct `SYSTEM_Control` route may solve the mode-selection problem without a binary change.

```text
file:   C:/firmware_temp/patch_baseline/mik_stock.ko
offset: 0x9B54C
VA:     0x98A58
old:    00 60 A0 E3   mov r6,#0
new:    01 60 A0 E3   mov r6,#1   (candidate only)
```

Static effect: in the DTS-family no-`0x80` branch, the common tail selects SPDIF argument `2` (`AutoMode`) rather than `0` (`PcmMode`), and passes HDMI-TX argument `1`.

Limits:

- It does not change the EDID parser or the physical receiver.
- It does not set `0x582`, clear a license mask, or start a DTS decoder.
- It does not bypass the `0x25B7` support clamp or any DSP/IEC transport condition.
- It changes the shared HDMI-TX/SPDIF decision for the DTS-family arm; it is not a generic “all outputs raw” switch.

### Candidate B: explicit BypassMode instead of AutoMode

A more aggressive design would force the SPDIF tail argument to `3` (`BypassMode`) for the DTS-family no-bit path. That is not justified by the current static evidence as the first experiment: the normal DTS-core-present path uses `r4=2` (`AutoMode`), so Candidate A is the narrower parity candidate. Any such change would require a separate regression plan and explicit authorization.

### Candidate C: sink-side DTS SAD/EDID

A DTS SAD injected into the actual HDMI-TX sink EDID would be the conceptually correct no-code gate solution. Static evidence proves that RX EDID/SAD mechanisms exist, but does not prove that the available controls feed the TX sink bitmap. No EDID file or configuration was changed.

### Separate upstream prerequisite

If the current stock image does not start the DTS decoder because of `0x440`, `0x4D8`, `0x582`, Auth, or DSP state, an output-mode patch alone cannot create a DTS elementary stream. The static upstream chain is:

```text
MDrv_AUDIO_CheckHashkey
  -> g_AudioVars2+0x4D8 / +0x582
  -> HAL_AUDIO_SetSystem2 reads 0x4D8
  -> decoder command byte 0x97 when 0x4D8 == 3
```

That is a separate decoder/license target. No byte-level change is endorsed in this report, and it is not a hidden-menu unlock. The order for any later authorized work must be:

```text
1. prove/ensure DTS decoder start and state 0x4D8/0x582
2. prove/ensure sink EDID bit 0x80 or deliberately bypass that gate
3. inspect HAL 0x25B7/support clamp
4. only then assess physical IEC61937/SDO output
```

This ordering prevents treating a PCM-mode symptom as a decoder or transport proof.

## 9. Evidence index

| Claim | Classification | Main evidence |
|---|---|---|
| MI tests EDID `0x80` and absent branch defaults to PCM | VERIFIED BY BINARY | `mik_SetHdmiAutoMode.asm:33-42,168-178` |
| MI codec 9/10/11/23 use the DTS arm | VERIFIED BY BINARY | stock dispatch table at `0x989E8` |
| SPDIF mode 2=Auto, 3=Bypass, 4=Transcode | VERIFIED BY BINARY | `utpa2k_stock.ko` jump table and call relocations |
| `SPDIF_OUT_NONE=5` is normalized toward PCM | VERIFIED BY BINARY | `HAL_AUDIO_SPDIF_SetMode` `0x448A10` |
| `AutoMode` reads `0x582` and sets `0x583` | VERIFIED BY BINARY | `final_setmode.txt:605-610` |
| `MI_AOUT_SetDigitalMode` does not write sink bitmap/global mode offsets | VERIFIED BY BINARY | `final_digitalmode.txt:167-175` |
| Selected Mstar EDID1..4 profiles lack format 0x07 | VERIFIED BY STATIC BYTE PARSE | board EDID files + local CTA parser output |
| Legacy RX EDID files contain format 0x07 | VERIFIED BY STATIC BYTE PARSE | stored `Mstar_EDID_v1_3_1080p_*` files |
| Sink EDID support list is `_u32CurEdidSupportList` | VERIFIED BY BINARY | `mik_aoutmonitor.asm:438-474,607-680`; format nibble becomes `1 << code` |
| RX EDID and TX sink EDID are separate paths | VERIFIED BY BINARY | `MI_DISP_IMPL_XC_HDMIRx_SetEDID` / `_astHdmiInfo` vs HDMI-TX provider / `_u32CurEdidSupportList` |
| Board SPDIF/ARC constants are routing, not mode unlock | VERIFIED BY CONFIG | `board.ini`, `Customer_Module.ini`, `Board.h` |
| Factory service exists | VERIFIED BY RETAINED LOG ONLY | `R1_baseline_logcat.txt:1031` |
| `[M_FACTORY]` sections are empty and `factory_keypad.ini` is absent | VERIFIED BY CONFIG/INVENTORY | `Customer_Module.ini`, `MStar_Default_Module.ini`, `final_edid.txt:491` |
| Factory IR control 4 gates factory key filtering | VERIFIED BY BINARY | `dtv_driver.ko` `__GlueIR_ConfigCtrlKey` / `__GlueIR_ConfigBlockKey` |
| Factory audio API exists but commands 0x8143/0x8146 are not DTS setters | VERIFIED BY BINARY | `MI_AOUT_FactorySetAttr` `0x8928C..0x89604` |
| `MTGADEC_MTAUD_SetAudioOutMode` is a no-op | VERIFIED BY BINARY | `dtv_driver.ko` `0x93C90`, 8 bytes |
| Factory HIDL GAM/property bridge exists | VERIFIED BY SYMBOL/STRING | available TV API service and `mtktvapi.so` |
| Direct native `MApi_AUDIO_SYSTEM_Control` mode writer exists | VERIFIED BY BINARY | `utpa2k_stock.ko` `0x3FE754` → `0x411C44`; ioctl `0xE5` |
| Userspace SPDIF/HDMI mode APIs exist | VERIFIED BY BINARY | `libutopia.so` `0x344328` ioctl `0x5E`, `0x34555C` ioctl `0x6D` |
| Normal `mik.ko` callers use type/mute/capability commands | VERIFIED BY RELOCATION/STRING | Open/Monitor call sites listed in §3.2 |
| Native SPDIF mode command reaches `MDrv_AUDIO_SPDIF_SetMode` | VERIFIED BY BINARY | `MDrv_AUDIO_SYSTEM_Control` call `0x4125E8` |
| Native HDMI-TX command reaches a no-op stock HAL | VERIFIED BY BINARY | call `0x4124F4`, HAL `0x44E108` = `bx lr` |
| SPDIF string parser maps BYPASS=3 | VERIFIED BY BINARY | `MDrv_AUDIO_DigitalOut_ParseSpdifOutputMode` `0x41F168` |
| HDMI-TX string parser maps BYPASS=1 | VERIFIED BY BINARY | `MDrv_AUDIO_DigitalOut_ParseHdmiTxOutputMode` `0x41F200` |
| Factory menu can set DTS/EDID | NOT ESTABLISHED | factory binary absent; no OSD/GAM xref found |
| Mode-2 patch forces a valid physical DTS stream | NOT ESTABLISHED | mode gate is only one prerequisite |

## 10. Final answer to the mode/menu question

The menu is not imaginary: the HAL has explicit PCM, AUTO, BYPASS, and TRANSCODE workers, the board has SPDIF/ARC routing, the factory stack has an IR filter plus a generic HIDL command bridge, and `utpa2k` exposes a direct native `MApi_AUDIO_SYSTEM_Control` mode writer. The static evidence still does **not** show which menu/service command reaches that writer, whether its SPDIF request survives the EDID monitor, or whether any path changes DTS AUTH state or the sink EDID bit.

The most defensible static patch target is the single missing-DTS default at `mik_stock.ko:0x9B54C`, changing `mov r6,#0` to `mov r6,#1` so the DTS-family path uses the same `r4=2` AutoMode argument as the DTS-core-present path. It is a candidate only. It was not applied, and it is not sufficient by itself to establish physical DTS passthrough.

The companion raw evidence extract is:

```text
C:/firmware_temp/forensic_space_bunny_spdif_menu_mode_evidence_20260925.txt
```
