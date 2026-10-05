# Addendum — the SPDIF format request rides on the mtktvapi `audio_info` private API

Date: 2026-09-26
Parent report: `forensic_space_bunny_spdif_control_surfaces_20260926.md`
Companion evidence: `forensic_space_bunny_spdif_native_private_api_evidence_20260926.txt`

No patch, no device change. All work offline on
`factory_menu_analysis_20260926/native/libcom_mediatek_twoworlds_tv_jni.so`
(sha256 `90c356b8023a6d3c6d100fbecf44245d99d0b769b6df34ded4e3aa7142688bdf`).

---

## 1. Correction to a conversational claim

I stated in chat that `setAudioInfoValue_native` resolves to
`a_mtktvapi_videoinfo_reg_common_nfy`. **That was wrong** and is withdrawn. The
PLT-entry addresses I derived from the stub contents do not survive a call-site
cross-check (§3). The written reports already listed this callee as unresolved
(`evidence §5` and `§11`), so no artifact contained the bad claim.

What survives is stronger and does not depend on PLT layout (§2).

---

## 2. Confirmed: the private API family is `a_mtktvapi_audio_info_*`

The JNI library imports 608 `a_mtktvapi_*` symbols. The audio-info family is
present in full:

```
a_mtktvapi_audio_info_set_value
a_mtktvapi_audio_info_get_value
a_mtktvapi_audio_info_register_callback
a_mtktvapi_audio_info_get_available_record
a_mtktvapi_audio_info_get_inputsource_available_record
a_mtktvapi_audio_info_get_current_lanuage
a_mtktvapi_audio_info_set_next_lanuage
```

These correspond 1:1, by name, to the exported JNI entry points:

| JNI symbol (`Java_com_mediatek_twoworlds_tv_TVNative_*`) | private API |
|---|---|
| `setAudioInfoValue_native`  | `a_mtktvapi_audio_info_set_value` |
| `getAudioInfoValue_native` | `a_mtktvapi_audio_info_get_value` |
| `getAudioAvailableRecord_native` | `a_mtktvapi_audio_info_get_available_record` |
| `getAudioInfo_native`      | `a_mtktvapi_get_current_audio_info` |
| (lang get/set)             | `a_mtktvapi_audio_info_get_current_lanuage` / `_set_next_lanuage` |

Five independent name matches, no collisions, all read straight out of
`readelf --dyn-syms` with no layout assumptions. The JNI is a thin shim over the
MediaTek TV private API, and **the SPDIF format request is an `audio_info`
set_value call** — not a video-info call.

This is exactly the private-API shape described for the TCL/Disney reference:
a vendor OTT-platform API that carries an output-format request, reachable in
principle from any app, wired here only for one hardcoded package.

---

## 3. Why the PLT disassembly is not trustworthy here

Two attempts, both negative:

- Decoding `.plt` stubs from instruction content (Thumb and ARM) produced a
  self-consistent map of 710 of 785 `R_ARM_JUMP_SLOT` entries in relocation
  order — but yielded `0x82b00 → a_mtktvapi_audio_info_set_value`,
  `0x82b30 → a_mtktvapi_videoinfo_reg_common_nfy`.
- Scanning every executable byte of the library for `bl`/`blx` into those
  addresses returned **0 call sites** for all three `audio_info_*` entries.

Conclusion: the stub-derived addresses are wrong. This binary's `.plt` mixes
encodings (Thumb-style stubs around `0x81c70`, ARM-style around `0x82ac0`) and
straight-line 4-byte stepping produces coincidental matches. The relocation
table itself is authoritative; the address-to-symbol binding is not recoverable
this way from a stripped binary without the relocation's PLT-index field.

Recorded as an unresolved item, not as a result.

---

## 4. OTT source map — Kodi is not special-cased

`Lcom/mediatek/tv/agent/MonitorActivityService$MonitorActivityController;->addMap()`:

```
0  com.mediatek.wwtv.mediaplayer          (MMP)
1  com.netflix.ninja                      (NETFLIX)
2  air.com.vudu.air.DownloaderTablet      (VUDU)
3  com.google.android.youtube.tv          (YOUTUBE)
4  "network"
```

`setOTTSrc(String)` writes the matched value into the `OTT_SRC_TYPE` TVConfig key
via `MtkTvConfigBase.setConfigValue`. There is **no Disney** entry and **no Kodi**
entry on this image. Kodi falls into the generic/network bucket, so the
Netflix-gated `setSpdifFmt` path never applies to it — consistent with the
runtime captures.

---

## 5. Open items carried forward

1. Exact PLT entry ↔ call-site binding in the stripped 32-bit JNI. Needs a
   non-stripped build of the same library, or a debugger attach.
2. Whether `audio_info_set_value` accepts the literal `7` that
   `updatePackageChanged()` sends (shipped `SPDIF_MODE_FMT_*` stop at `AUTO2 = 6`).
3. How `SPDIF_MODE_FMT_*` (0..6) maps onto the HAL's `spdif_mode` /
   `spdif_type` axes. Still unreconciled.
4. Device was offline at the time of writing (`ping 192.168.0.183` — 100% loss,
   no ARP entry), so `vendor.mediatek.api.mtktvapi@1.0-cwrapper.so` and
   `vendor.mediatek.api.mtktvapi@1.0.so` could not be pulled. The service-side
   handler for `audio_info_set_value` is therefore still unexamined — that is
   where the value→register translation would be found.
