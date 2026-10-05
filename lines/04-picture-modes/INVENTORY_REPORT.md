# Workspace Inventory Report: Display & EDID Architecture

**Date:** 2026-09-04
**Target Device:** Thundeal TD98PRO / C50A (MediaTek MT5889 / mt5889_k32, Android 11 / SDK 30, Linux kernel 4.19.116)
**Total Inventoried Core Artifacts:** 56

## 1. Classification and Artifact Matrix

| Filename | Category | Size (bytes) | MD5 Hash | Role | Pipeline Relation | Analyzed Before |
| --- | --- | --- | --- | --- | --- | --- |
| `ree_payload.bin` | Firmware Image | 1,525,350,400 | `SKIPPED (>50MB)...` | Raw firmware REE payload (opaque container, untouched per instruction) | Contains system partitions (not analyzed directly) | Partially surveyed in audio branch |
| `mik.ko` | Kernel Module | 8,084,248 | `c1421040dbcfff41...` | MediaTek Integration Kernel (MI) core driver | Implements MI_DISP, MI_EXTIN, MI_AOUT, HDMIRX EDID loader, sysfs | Extensively analyzed in audio branch |
| `utpa2k.ko` | Kernel Module | 25,381,336 | `2fc6e9fc46b6402d...` | Utopia 2K kernel driver | Underlying MStar hardware interface, IP auth, audio/video registers | Extensively analyzed in audio branch |
| `dtv_driver.ko` | Kernel Module | 3,451,776 | `6d72fbebe034a213...` | DTV / Video demod & transport driver | Hardware video/audio paths | Analyzed in audio branch |
| `mstar_fbdev_mi.ko` | Kernel Module | 69,344 | `a11db5d4f2576fcb...` | MStar Framebuffer device driver for MI display | Linux framebuffer /dev/graphics/fb0 interface | NEW |
| `kdrv_xc.ko` | Kernel Module | 1,557,136 | `b6e64e77ba3d678d...` | MStar XC (video scaler / display engine) kernel driver | Controls video scaler, panel timings, timing generation | NEW |
| `xcker.ko` | Kernel Module | 677,224 | `047aca283fd79365...` | XC kernel driver core | Low-level video scaler and display pipeline control | NEW |
| `mwgifker.ko` | Kernel Module | 135,120 | `45e14076dee84dd8...` | MStar Window Graphic Interface kernel module | Graphics overlay and window plane management | NEW |
| `kdrv_dolby_vision.ko` | Kernel Module | 651,948 | `03903a3107a47071...` | Dolby Vision kernel driver | HDR/Dolby Vision video path | NEW |
| `kdrv_alg.ko` | Kernel Module | 2,055,940 | `7da47a92cab91113...` | MStar algorithm kernel driver (PQ / picture quality) | Picture quality enhancement | NEW |
| `kdrv_ldm_sys.ko` | Kernel Module | 38,200 | `c61b95aa1a031281...` | Local dimming system driver | Backlight/local dimming | NEW |
| `mdrv_ldm.ko` | Kernel Module | 315,340 | `d3f57d1aa87f4e85...` | Local dimming driver | Backlight/local dimming | NEW |
| `hwcomposer.mt5889.so` | Display HAL | 475,016 | `2d9f3965c85db585...` | MediaTek MT5889 Hardware Composer HAL module | Directly handles display mode enumeration, vsync, surface composition | NEW |
| `android.hardware.graphics.composer@2.4-service` | Display Service | 100,704 | `4d43c44da41edd94...` | HIDL Graphics Composer 2.4 Service executable | Exposes IComposer 2.4 to SurfaceFlinger | NEW |
| `surfaceflinger` | Android System | 8,940 | `1864e1de625e14e9...` | Android SurfaceFlinger native daemon | Coordinates composition, calls HWC getDisplayModes / getActiveConfig | NEW |
| `libsurfaceflinger.so` | Android System | 1,577,908 | `7c2a88004817db90...` | SurfaceFlinger core library | Display mode filtering, DisplayDevice setup | NEW |
| `libgui.so` | Android System | 721,236 | `e310674f0737c8b2...` | Android GUI native library | ISurfaceComposer client interfaces | NEW |
| `libui.so` | Android System | 217,672 | `3f1e7f0b33ed90a4...` | Android UI native library | DisplayMode, DisplayInfo structs | NEW |
| `gralloc.default.so` | Graphics HAL | 13,064 | `6f19057351c4cc06...` | Default gralloc HAL | Buffer allocation | NEW |
| `android.hardware.graphics.allocator@4.0-impl-arm.so` | Graphics HAL | 145,572 | `3c127636caf721aa...` | Gralloc 4.0 allocator implementation | Buffer allocation | NEW |
| `android.hardware.graphics.mapper@4.0-impl-arm.so` | Graphics HAL | 199,920 | `e69490452badb309...` | Gralloc 4.0 mapper implementation | Buffer mapping | NEW |
| `libmi3.so` | Vendor Middleware | 750,608 | `c2b5c57dba4f8922...` | MediaTek MI3 userspace middleware | Userspace API to mik.ko and utpa2k | Analyzed for audio in audio branch |
| `libutopia.so` | Vendor Middleware | 19,353,764 | `1956a5f689997c35...` | MStar Utopia userspace library | Userspace Utopia API | Analyzed for audio in audio branch |
| `vendor.mediatek.tv.mtkdmservice@1.0-service` | Vendor Service | 28,496 | `595d0e4fc11c9d8e...` | MediaTek TV Display/Data Manager service | Vendor TV display management | NEW |
| `mstar.hardware.imgdisp@1.0-service` | Vendor Service | 6,164 | `7bc9bfca95ffa494...` | MStar Image Display service | Vendor image display pipeline | NEW |
| `mstar.hardware.mmdisp@1.0-service` | Vendor Service | 6,132 | `5071f5a590312405...` | MStar Multimedia Display service | Vendor multimedia display pipeline | NEW |
| `services.jar` | Framework | 13,236,286 | `f1f586549ec2104b...` | Android system_server services | Contains DisplayManagerService, DisplayModeDirector, LocalDisplayAdapter | NEW |
| `framework.jar` | Framework | 27,286,714 | `8baf2de8f87d3394...` | Android base framework classes | Contains android.view.Display, Display.Mode | NEW |
| `edid_cfg.ini` | Board Config | 2,532 | `c63518a896c0c02a...` | HDMI RX EDID port configuration INI | Defines which EDID binary is assigned to HDMI RX ports 1-4 and default version | Discovered in audio branch, fully parsed here |
| `edid_cfg.ini` | Board Config | 2,640 | `22d67144f39f2e19...` | HDMI RX EDID port configuration INI (EWS variant) | Alternative board EDID configuration with freesync | Discovered in audio branch |
| `Customer_1.ini` | Board Config | 24,368 | `61e37d6a54ada02d...` | Customer 1 board/panel model configuration | Specifies active panel INI file path, EDID paths, PQ configs | NEW |
| `board.ini` | Board Config | 78,028 | `e3c629dbd2a24f9d...` | Hardware board pinmux/interface configuration | Board hardware definitions | NEW |
| `panel_swing_cfg.ini` | Board Config | 40 | `6010bee151887d9e...` | Panel LVDS swing level configuration | Panel electrical interface levels | NEW |
| `tcon_cfg.ini` | Board Config | 1,322 | `dd3a7a3aa1c5bebe...` | Timing controller configuration | TCON interface settings | NEW |
| `UD_VB1_16LANE_CSOT_URSA.ini` | Panel Config | 21,442 | `c618071e2017b62e...` | Active internal panel timing & geometry configuration | Defines 3840x2160 panel geometry, HTotal, VTotal, DCLK, OSD 1920x1080 | NEW |
| `FullHD_VB1_8LANE_DLP_PROJECTOR_120_3D.ini` | Panel Config | 32,539 | `80e28fc08c250a17...` | Projector 3D 120Hz panel timing configuration | 1080p 120Hz 3D projector timing configuration | NEW |
| `sys.ini` | System Config | 1,763 | `d71fe5b2726b5518...` | System configuration master INI | Directs loader to Customer_1.ini, panel directory, PQ directory | NEW |
| `mik_init.ini` | System Config | 6,110 | `140e961251914048...` | MI kernel driver initialization parameters | MI subsystem init config | NEW |
| `PcModeTimingTable.ini` | Timing Table | 22,211 | `9074edb0f176f740...` | PC Mode timing detection database (91 modes) | Timing parameters (HFreq, VFreq, HTotal, VTotal) for auto video detect | NEW |
| `hdr.ini` | PQ Config | 165 | `5d9248b7bedc9659...` | HDR picture quality configuration | HDR10 / HLG / Dolby Vision PQ parameters | NEW |
| `dolby.bin` | PQ / HDR Binary | 14,235 | `5e2e0da96e9e1645...` | Dolby Vision firmware binary blob | Dolby Vision processing blob | NEW |
| `dolby_factory.cfg` | PQ / HDR Config | 2,767 | `84da03b9b0f86425...` | Dolby Vision factory configuration | Dolby Vision config | NEW |
| `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin` | EDID Binary | 256 | `1352c0c360439adc...` | HDMI Port 1 default EDID profile (EDID 2.1 4K HDR) | Presented to external HDMI source on Port 1 | NEW |
| `Mstar_EDID1_v2.0_4K_2K_3D_HDR.bin` | EDID Binary | 256 | `dd40b3bdc9903488...` | HDMI Port 1 2.0 EDID profile (EDID 2.0 4K HDR) | Presented to external HDMI source when 2.0 selected | NEW |
| `Mstar_EDID1_v1.4_3D_Frame_SideHalf_Top.bin` | EDID Binary | 256 | `3b4c1c31c9ae084b...` | HDMI Port 1 1.4 EDID profile (EDID 1.4 1080p 3D) | Presented to external HDMI source when 1.4 selected | NEW |
| `Mstar_EDID2_v2.1_4K_2K_3D_HDR.bin` | EDID Binary | 256 | `431ddae10239c72b...` | HDMI Port 2 default EDID profile (EDID 2.1 4K HDR) | Presented to external HDMI source on Port 2 | NEW |
| `Mstar_EDID1_v1.4_2K.bin` | EDID Binary | 256 | `99594ebc080a0f16...` | HDMI 1.4 2K standard EDID profile | Alternative 1080p-only HDMI RX EDID | NEW |
| `VGA_EDID.bin` | EDID Binary | 128 | `dc7532681da44c56...` | VGA input standard EDID (128 bytes) | Presented on VGA/PC analog input | NEW |
| `REPORT_forensic_synthesis.md` | Report | 17,213 | `ab3c9d8267851869...` | Forensic synthesis report from audio investigation | Details mik.ko SetHdmiAutoMode, utpa2k auth, EDID audio gates | Completed in audio branch |
| `REPORT_forensic_deepdive_static.md` | Report | 25,151 | `469e0c50428d2938...` | Static deep-dive report | Instruction-level analysis of mik.ko and utpa2k.ko | Completed in audio branch |
| `REPORT_static_final.md` | Report | 23,464 | `413a4f250c7de940...` | Final static closure report | Distinguishes RX vs TX EDID, SetDigitalMode, etc. | Completed in audio branch |
| `final_edid.py` | Script | 4,119 | `a39edadad1d9a747...` | EDID investigation tool script from audio branch | Searches EDID callers and .rodata blobs in mik.ko | Completed in audio branch |
| `final_edid.txt` | Analysis Log | 26,351 | `21d73931896d268f...` | Output of final_edid.py | Contains disasm of SetEDID callers and .rodata strings in mik.ko | Completed in audio branch |
| `props_all.txt` | Runtime Log | 32,103 | `3759dfdbb6bd3172...` | Full Android system properties dump from projector | Contains ro.sf.lcd_density, vendor.display-size=3840x2160, etc. | Captured in audio branch |
| `kodi_log.txt` | Runtime Log | 63,129 | `f52504846a8aaf1c...` | Kodi Android runtime log on projector | Captures display mode adjustment 1920x1080@60Hz and 23.976 search failure | Captured in audio branch |
| `kodi_guisettings.xml` | Runtime Log | 33,693 | `7a48a90545800140...` | Kodi GUI settings XML from projector | Shows videoscreen.whitelist with only 1080p60 and 1080p50 | Captured in audio branch |

## 2. EDID Binary Directory Summary

Found **60 static EDID binary templates** in `/vendor/cusdata/bsp/common/EDID_BIN/`.
Every profile is exactly 256 bytes (2 blocks: Base EDID 1.3/1.4 + CTA-861 Extension), except `VGA_EDID.bin` which is 128 bytes (Base only).

| EDID Binary | Size | MD5 Hash | Category / Profile |
| --- | --- | --- | --- |
| `HDMI_EDID.bin` | 256 | `80c5eb4b1d9328de723c875aa59ca676` | HDMI RX static template |
| `Mstar_EDID1_v1.4_2K.bin` | 256 | `99594ebc080a0f161bc8ddc5990cd37a` | HDMI RX static template |
| `Mstar_EDID1_v1.4_3D_20130131.bin` | 256 | `7ef0e05b056888a5bbc894f8d964a3e4` | HDMI RX static template |
| `Mstar_EDID1_v1.4_3D_4K_2K.bin` | 256 | `774de6b9b4b81f3dd3150f51c3e0cb1e` | HDMI RX static template |
| `Mstar_EDID1_v1.4_3D_Frame_SideHalf_Top.bin` | 256 | `3b4c1c31c9ae084b678a3a4c4af8394e` | HDMI RX static template |
| `Mstar_EDID1_v2.0_4K_2K.bin` | 256 | `629c04cf7a8e820891c68c05987c8b7d` | HDMI RX static template |
| `Mstar_EDID1_v2.0_4K_2K_30_YUV420.bin` | 256 | `79fda9e3fde4af52ab53d3734fb4dde0` | HDMI RX static template |
| `Mstar_EDID1_v2.0_4K_2K_3D.bin` | 256 | `2980ac6ff89761f7e55391ff43967141` | HDMI RX static template |
| `Mstar_EDID1_v2.0_4K_2K_3D_HDR.bin` | 256 | `dd40b3bdc990348841fe3f03d49587c5` | HDMI RX static template |
| `Mstar_EDID1_v2.0_4K_2K_3D_HDR_freesync.bin` | 256 | `f3c3e7c6df14e7331f74611e2a189781` | HDMI RX static template |
| `Mstar_EDID1_v2.1_4K_2K_30_YUV420_ALLM_5867.bin` | 256 | `39fc9123a4a1523c9a0c5127b2f7ba82` | HDMI RX static template |
| `Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin` | 256 | `1352c0c360439adcf93946a7c800d2fb` | HDMI RX static template |
| `Mstar_EDID1_v2.1_4K_2K_3D_HDR_freesync.bin` | 256 | `71671fd4415dd6453d6babc6c0930d23` | HDMI RX static template |
| `Mstar_EDID2_v1.4_2K.bin` | 256 | `678496b46af9ef4d927892631e8a686a` | HDMI RX static template |
| `Mstar_EDID2_v1.4_3D_20130131.bin` | 256 | `c11ca03ecaa5000660c3018a5ca988b1` | HDMI RX static template |
| `Mstar_EDID2_v1.4_3D_4K_2K.bin` | 256 | `35f7b712f447855524e8fdbb03b37e5c` | HDMI RX static template |
| `Mstar_EDID2_v1.4_3D_Frame_SideHalf_Top.bin` | 256 | `0243f9c02d67688c1afa4cbc8e9684e1` | HDMI RX static template |
| `Mstar_EDID2_v2.0_4K_2K.bin` | 256 | `e47802bf8db282d5fc504f4ed4910d6a` | HDMI RX static template |
| `Mstar_EDID2_v2.0_4K_2K_30_YUV420.bin` | 256 | `885ce42cffa0ce04d763678bfddff9d0` | HDMI RX static template |
| `Mstar_EDID2_v2.0_4K_2K_3D.bin` | 256 | `2bf00df8e279b9649fcfaf884e307b57` | HDMI RX static template |
| `Mstar_EDID2_v2.0_4K_2K_3D_HDR.bin` | 256 | `9ee1cf60d55cd7be4bdd67e95f3d897d` | HDMI RX static template |
| `Mstar_EDID2_v2.0_4K_2K_3D_HDR_freesync.bin` | 256 | `c9c095cb1ca7614bf233a633f464493a` | HDMI RX static template |
| `Mstar_EDID2_v2.1_4K_2K_30_YUV420_ALLM_5867.bin` | 256 | `50e567ce57cddec4c020c9be9b233e3d` | HDMI RX static template |
| `Mstar_EDID2_v2.1_4K_2K_3D_HDR.bin` | 256 | `431ddae10239c72b07f435c6a0994d41` | HDMI RX static template |
| `Mstar_EDID2_v2.1_4K_2K_3D_HDR_freesync.bin` | 256 | `7850273d8765bdfe272fc1613967c0c1` | HDMI RX static template |
| `Mstar_EDID3_v1.4_2K.bin` | 256 | `c66ea8f6de39e12e97a45a6ba9de6390` | HDMI RX static template |
| `Mstar_EDID3_v1.4_3D_20130131.bin` | 256 | `8e87a7b52f886ce0c4c9ddc5df48409b` | HDMI RX static template |
| `Mstar_EDID3_v1.4_3D_4K_2K.bin` | 256 | `8aba5b0a493873f97ae3ff4724411307` | HDMI RX static template |
| `Mstar_EDID3_v1.4_3D_Frame_SideHalf_Top.bin` | 256 | `461affd037b61a28deb298f915a927c9` | HDMI RX static template |
| `Mstar_EDID3_v2.0_4K_2K.bin` | 256 | `16d843192f44a3fdbbadca0123056c12` | HDMI RX static template |
| `Mstar_EDID3_v2.0_4K_2K_30_YUV420.bin` | 256 | `3b1c72d6a1fc1237fcfc97968e9a7ada` | HDMI RX static template |
| `Mstar_EDID3_v2.0_4K_2K_3D.bin` | 256 | `3ca284ee8986060926cbb7995747adc1` | HDMI RX static template |
| `Mstar_EDID3_v2.0_4K_2K_3D_HDR.bin` | 256 | `2a20fc25bcb8d87153bb4573555b35da` | HDMI RX static template |
| `Mstar_EDID3_v2.0_4K_2K_3D_HDR_freesync.bin` | 256 | `292b1cd515d603a97a039786e3de7840` | HDMI RX static template |
| `Mstar_EDID3_v2.1_4K_2K_30_YUV420_ALLM_5867.bin` | 256 | `66431655c92231b1a3d82c3f8dcfc501` | HDMI RX static template |
| `Mstar_EDID3_v2.1_4K_2K_3D_HDR.bin` | 256 | `1409c621d92cc7597397b564740c4091` | HDMI RX static template |
| `Mstar_EDID3_v2.1_4K_2K_3D_HDR_NoFRL.bin` | 256 | `114c13ba12dabd441f5484e86cc5b894` | HDMI RX static template |
| `Mstar_EDID3_v2.1_4K_2K_3D_HDR_freesync.bin` | 256 | `fca413aacf860bd01d7b5b03b6c6b193` | HDMI RX static template |
| `Mstar_EDID4_v1.4_2K.bin` | 256 | `2af209980fdf180e742cd9069cd817b3` | HDMI RX static template |
| `Mstar_EDID4_v1.4_3D_20130131.bin` | 256 | `cdad74f8f7f141838f1016fae0c4a7bc` | HDMI RX static template |
| `Mstar_EDID4_v1.4_3D_4K_2K.bin` | 256 | `a1ecabbdf18f12c67807a5ffa4ef4248` | HDMI RX static template |
| `Mstar_EDID4_v1.4_3D_Frame_SideHalf_Top.bin` | 256 | `6b9b07ea59401d1f7fa14756c13403e0` | HDMI RX static template |
| `Mstar_EDID4_v2.0_4K_2K.bin` | 256 | `29b382074b34d091a88755da9ca69939` | HDMI RX static template |
| `Mstar_EDID4_v2.0_4K_2K_30_YUV420.bin` | 256 | `ba7fa5256ea8dd2dc9f7883009abb883` | HDMI RX static template |
| `Mstar_EDID4_v2.0_4K_2K_3D.bin` | 256 | `4e1f77e70347bae88ba0e604b565962a` | HDMI RX static template |
| `Mstar_EDID4_v2.0_4K_2K_3D_HDR.bin` | 256 | `59d3825838f2f93d58404eb552d3544f` | HDMI RX static template |
| `Mstar_EDID4_v2.0_4K_2K_3D_HDR_freesync.bin` | 256 | `3c07967b4c85aa718b0c380f4bafe048` | HDMI RX static template |
| `Mstar_EDID4_v2.1_4K_2K_30_YUV420_ALLM_5867.bin` | 256 | `d39ce3bb6cebcd18249f9bfe0f9aeebb` | HDMI RX static template |
| `Mstar_EDID4_v2.1_4K_2K_3D_HDR.bin` | 256 | `8b7d6a23d12562c45b50d3e4e7f9251f` | HDMI RX static template |
| `Mstar_EDID4_v2.1_4K_2K_3D_HDR_NoFRL.bin` | 256 | `0ef743b51df620f5d8d683d9fd107f38` | HDMI RX static template |
| `Mstar_EDID4_v2.1_4K_2K_3D_HDR_freesync.bin` | 256 | `368eff21361899b74b70a842519bb81e` | HDMI RX static template |
| `Mstar_EDID_v1_3_1080p_1_0_0_0.bin` | 256 | `790f8de55f376e25ed5b16e8bcc89260` | HDMI RX static template |
| `Mstar_EDID_v1_3_1080p_2_0_0_0.bin` | 256 | `db07234d34a94468f275e54f72c9f854` | HDMI RX static template |
| `Mstar_EDID_v1_3_1080p_3_0_0_0.bin` | 256 | `85472dac3e366ad16a10e8b983e9f515` | HDMI RX static template |
| `Mstar_EDID_v1_3_1080p_4_0_0_0.bin` | 256 | `f489189a275fce915fd5de0e68ab0451` | HDMI RX static template |
| `T31_Mstar_EDID1_v2.1_4K_2K_3D_HDR.bin` | 256 | `bf755e58b709bc1703d8a143a2963bb1` | HDMI RX static template |
| `T31_Mstar_EDID2_v2.1_4K_2K_3D_HDR.bin` | 256 | `78ebe09bf284553044b412d8182e3bb2` | HDMI RX static template |
| `T31_Mstar_EDID3_v2.1_4K_2K_3D_HDR.bin` | 256 | `c6521ced0385722fb5d2db3d48351c91` | HDMI RX static template |
| `T31_Mstar_EDID4_v2.1_4K_2K_3D_HDR.bin` | 256 | `9c137841c9dcaa436140ee1e329f3d9c` | HDMI RX static template |
| `VGA_EDID.bin` | 128 | `dc7532681da44c56175d0feffe06b742` | HDMI RX static template |
