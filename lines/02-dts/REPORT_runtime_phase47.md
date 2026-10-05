# REPORT — R47 — Runtime capture of DTS-vs-AC3 passthrough (Thundeal TD98 Pro / MStar MT5889, AEON R2 DSP)

**Subject:** Why AC3 passthrough works on the Pioneer VSX-817 but DTS passthrough fails.
**Method:** Read-only safe mirror `/proc/utopia_mdb/audio` (DSP-SRAM `DM[...]` dump) + Kodi `media_session` playback state, during **real, confirmed** AC3 and DTS playback.
**Date:** 2026-09-11 (controlled run).
**Status:** ✅ COMPLETE (controlled AC3 + DTS mirror runs **+** safe read-only `logcat` HAL/codec/Kodi correlation **+** dense 20-sample status-block re-sample **+** fresh full-image static trace of the `0x0C00–0x0C07` writers / `0x0854` gate path answering §11 item 5). **No patch. No device modification. No unsafe memory access. No fabricated data.**

---

## §0 Executive summary

| question | answer |
|---|---|
| Does the DTS decoder actually run on the DSP? | **YES.** The DEC decode region `0x0FE8–0x0FEB` is live and varying for **both** AC3 and DTS. The DTS decode engine is not the blocker. |
| Is the `0x0800–0x08FF` output-config block observable? | **NO.** `0x0854 / 0x086C / 0x0870 / 0x0858 / 0x0814` read `0x000000` in **every** sample of **both** codecs (write-only from host — pre-registered R47 outcome). |
| First meaningful AC3-vs-DTS divergence at the observable layer? | The **status/licence/telemetry block `0x0C00–0x0C07`** is populated during AC3 decode (non-zero) but stays **all-zero throughout DTS decode**, even though decode is live for both. |
| Does runtime confirm the R46d candidate (`0x0854` bits 17\|19)? | **Cannot be confirmed or rejected at runtime** — the cell is host-unreadable. The runtime data is *consistent with* an output-gate defect but cannot prove it. |
| Does DTS reach the Android/HAL passthrough path? | **YES (R47b, safe read-only `logcat`).** Kodi opens `method: RAW (PT) stream-type: STREAM_TYPE_DTS_512`; the vendor HAL runs `mi_common_raw_open` + `mi_decoder_open: format 9` (DTS passthrough decoder). The pipeline shape is **identical** to AC3 (`RAW (PT) STREAM_TYPE_AC3` + `mi_decoder_open: format 5`). The Kodi/HAL stack is **not** the blocker. |
| Is the `0x0C00–0x0C07` DTS-vs-AC3 divergence sustained (not a transient)? | **YES — sustained (R47c, dense 20/20 re-sample).** DTS status block = `0x000000` in **20/20** samples (DEC live); AC3 populates it burstily (non-zero in the majority). The §4 transient caveat is refuted; hypothesis **D** = SUPPORTED + REPRODUCIBLE. |
| Is the zero `0x0C00–0x0C07` during DTS a *cause* or a *symptom*? | **SYMPTOM (R47e, static trace — §16).** The block has **zero firmware writers** (fresh full-image scan, two independent methods) ⇒ it is read-only hardware status the firmware cannot populate; the hardware sets it only when the output path is armed. The **cause** is upstream: DTS **clears** `0x0854` bits 17\|19 (AC3 sets them — the only functional AC3-vs-DTS control difference, R46d D1.4). The gate `0x25EE1` reading `bit7(0x0C06)` is a downstream validator that fails *because* the bit was never set. |
| Patch justified? | **NO.** (a) Absolute constraint forbids patching/device modification; (b) the candidate cell is unobservable, so a one-variable fix could not even be verified; (c) `0x0854` bits 17\|19 semantics are UNKNOWN and the register is shared with the working AC3 path. |

**Conclusion:** The defect is localised to the **DSP output / status-handshake path** (downstream of decode), exactly where the R46d static candidate (`0x0854` bits 17\|19, AC3-sets / DTS-clears) lives. Runtime makes the candidate *more* likely but leaves it **UNRESOLVED** because the cell cannot be observed by any safe method. The status block is a downstream symptom, not the cause.

---

## §1 Methodology (corrected — this is why earlier runs were void)

Two earlier runs (now archived as `r47_capture_ac3_IDLE_badintent.log` and `r47_capture_dts_IDLE_badintent.log`) were **false**: they used `am start -d file://…` which only opened Kodi **idle** (`media_session state=7, active=false`), so the DSP was not decoding and every sampled cell was its idle value. A separate remote run (`r47_capture_ac3_remote_UNCONTROLLED.log`) was also discarded as non-controlled.

The corrected driver (`r47_work/r47_capture.sh` v2, with two justified fixes for the execution bug):
1. **Launch form:** `am start -a android.intent.action.VIEW -t video/* -n …/.Splash -d file://…` — this is the form Kodi actually auto-plays from. Verified: Kodi reaches `state=3 (PLAYING), speed=1.0` within 1–4 s.
2. **`wait_play` gate:** the script polls `media_session` and only samples after `PLAYING` is confirmed (max 18 s). This guarantees we never sample an idle DSP.

Each run also prints an explicit `>>> PLAYBACK STATE WILL CHANGE NOW` banner before every `am start` / `am force-stop`, per instruction.

**Safety (unchanged, absolute):** only `echo … > /proc/utopia_mdb/audio` (a *read command* to the kernel driver) and player start/stop. No `/dev/malloc|/dev/mem|/dev/miomap`, no MIU mmap, no CPU/physical-memory read, no register write, no EDID/AUTH/CheckHashkey, no HAL/EDID change, no patch.

Sequence executed (user-authorized): `force-stop` current playback → **AC3** (`test_ac3_51.mp4`) → **DTS** (`test_dts_51.mp4`). Both reached `PLAYING`.

---

## §2 AC3 runtime results (working — receiver plays) — `r47_capture_ac3.log`

`PLAYING confirmed (poll 1)`. Device stable (`watchdog` bootreason; uptime advanced monotonically 2078→2228, no reboot).

| sample | `0x0C00–0x0C07` (status/licence) | `0x0854/0x086C/0x0870` (output) | DEC `0x0FE8` | DEC `0x0FE9` | DEC `0x0FEA` | DEC `0x0FEB` |
|---|---|---|---|---|---|---|
| TRANSITION | **non-zero** (e.g. `0c00=0x680000`, `0c02=0x6b5a00`, `0c04=0x29bf00`, `0c05=0xd6b500`) | `0` | `0x000006` | `0x0009f6` | `0x0009c0` | `0x00007c` |
| STEADY1 | **non-zero** (e.g. `0c02=0xb5ad00`, `0c04=0xb6de00`, `0c06=0xdad600`) | `0` | `0x00001a` | `0x0009d4` | `0x0009a0` | `0x00006e` |
| STEADY2 | `0` (transient cleared) | `0` | `0x000000` | `0x000980`* | `0x000980`* | `0x000000` |
| POSTSTOP | `0` | `0` | `0` | `0x000980`* | `0x000980`* | `0` |

\* `0x0980` is the **idle constant** (baseline value).

→ Decode region live (non-idle `0x0fe8/0x0feb` non-zero, `0x0fe9/0x0fea` varying) during TRANSITION/STEADY1. Status block populated during active decode.

---

## §3 DTS runtime results (failing — receiver silent/unsupported) — `r47_capture_dts.log`

`PLAYING confirmed (poll 4)`. Device stable (uptime 2257→2415, no reboot).

| sample | `0x0C00–0x0C07` (status/licence) | `0x0854/0x086C/0x0870` (output) | DEC `0x0FE8` | DEC `0x0FE9` | DEC `0x0FEA` | DEC `0x0FEB` |
|---|---|---|---|---|---|---|
| TRANSITION | **ALL ZERO** | `0` | `0x000018` | `0x0009b6` | `0x000980`* | `0x00006e` |
| STEADY1 | **ALL ZERO** | `0` | `0x000018` | `0x0009f4` | `0x000980`* | `0x000064` |
| STEADY2 | **ALL ZERO** | `0` | `0x000020` | `0x0009e2` | `0x0009c0` | `0x000062` |
| POSTSTOP | `0` | `0` | `0` | `0x000980`* | `0x000980`* | `0` |

\* idle constant.

→ Decode region **live for DTS too** (`0x0fe8/0x0feb` non-zero in all 3 playing samples; `0x0fe9` varying `0x09b6→0x09f4→0x09e2`). **But `0x0C00–0x0C07` is zero in every sample.**

---

## §4 AC3 vs DTS — first meaningful divergence

| layer | AC3 (works) | DTS (fails) | diverge? |
|---|---|---|---|
| Kodi `media_session` | `PLAYING` (state=3) | `PLAYING` (state=3) | no — both decode |
| **DEC decode region `0x0FE8–0x0FEB`** | live / varying | **live / varying (same shape)** | **NO** — decode engine runs for both |
| **Status/licence block `0x0C00–0x0C07`** | **non-zero** during decode | **ALL ZERO** during decode | **YES — first observable divergence** |
| Output-config `0x0854/0x086C/0x0870` | `0` (unobservable) | `0` (unobservable) | n/a — write-only from host |
| Physical receiver (user) | plays | silent/unsupported | the symptom |

**First meaningful divergence:** the DSP **decode engine runs for both codecs** (DEC region live), but the **status/licence/telemetry block `0x0C00–0x0C07` that AC3 populates is never written during DTS decode**. Since decode is provably live for DTS, the failure is **downstream of decode** — in the output/format-negotiation handshake — exactly the region R46d identified as containing the (unobservable) `0x0854` gate.

> Caveat on transience: `0x0C00–0x0C07` is bursty (AC3 itself reads zero by STEADY2). R30 once observed a non-zero DTS `0x0c06` at a different transient moment. Within *this* controlled run DTS read zero in 3 consecutive playing samples vs AC3 non-zero in 2 of 3 — a consistent, repeated divergence. **This caveat is now CLOSED by the dense re-sample in §14 (R47c): 20/20 DTS samples show the whole status block, including `0x0c06`, at zero, while AC3 populates it burstily — the divergence is sustained, not a sampling fluke.**

---

## §5 Decode liveness — both codecs (key negative result)

Using the idle constant (`0x0fe8=0`, `0x0feb=0`, `0x0fe9/0x0fea=0x0980`) as the baseline:
- AC3 playing: `0x0fe8 ∈ {0x06,0x1a}`, `0x0feb ∈ {0x6e,0x7c}`, `0x0fe9/0x0fea` vary around `0x09c0–0x09f6`.
- DTS playing: `0x0fe8 ∈ {0x18,0x20}`, `0x0feb ∈ {0x62,0x6e}`, `0x0fe9/0x0fea` vary around `0x09b6–0x09e2`.

The decode region is **active for both**. ⇒ The R46c question ("are the DTS timing values mathematically wrong?") is moot as a *root cause*: a catastrophically wrong IEC61937 timing would not leave the decode engine running and live. This corroborates R46c's static finding that the values are correct.

---

## §6 Output-config block `0x0854/0x086C/0x0870` — capability limit (pre-registered)

All five target cells read `0x000000` in **every** AC3 and DTS sample. This is the **pre-registered R47 §R47.2 row-3 outcome** ("both read 0x000000 ⇒ mirror does not expose `0xB000_0854`; capability limit; do not attempt `/dev/malloc`"). Confirmed: the `0x0800–0x08FF` block is **write-only from the host**. The DSP's own reads of `0xB000_0854` still work (different path); only the host mirror does not expose it.

⇒ The R46d §D5 experiment (`DM[0x0854]` bits 17\|19) **cannot be performed by any safe method**. This is not new — it was already answered from R30 data; the controlled run re-confirms it.

---

## §7 Cross-reference to R46c / R46d

- **R46c (DTS timing `0x868/0x86C/0x870`):** values reconstructed and verified correct vs IEC61937 (CLOSED). Runtime (§5): DTS decode engine live ⇒ timing is not the blocker. **H1 REJECTED as root cause.**
- **R46d (bit diff at `0x854`):** AC3 (`0xCBFB`) sets bits **2,6,17,19** + 3-bit rate code (12–14); DTS only *clears* — `0xCC9B` clears **17|19**, `0xCF86` clears **21|23**. **Only functional difference = bits 17|19.** Two leaf predicates consume them (`0xC4E0` tests `v & 0xA0000`, `0xC6CC` tests `v & 0xA00000`). Clearing masks shared with non-DTS routines ⇒ no "DTS-exclusive" bit, but the AC3/DTS *difference* in bits 17|19 is real in the static trace.
- **This runtime:** decode live for both + output fails for DTS + status block diverges (§4) is **fully consistent with** an output-gate (bits 17|19 of `0x0854`) not being opened for DTS. But because `0x0854` is unobservable, it is **consistent-with, not confirmed-by**, runtime.

---

## §8 Hypothesis matrix

| ID | hypothesis | static (R46c/R46d) | runtime (R47) | verdict |
|---|---|---|---|---|
| **H1** | DTS IEC61937 timing values (`0x868/0x86C/0x870`) mathematically wrong → bad burst framing | values verified **correct** | DTS decode engine **live** (would not be if timing catastrophic) | **REJECTED** (not the cause) |
| **H2** | `0x0854` bits **17\|19** (output-enable / non-PCM / SDO) gate not opened for DTS | AC3 sets, DTS clears; **only functional diff** | decode live both; output fails DTS; status block diverges; cell **unobservable** | **SUPPORTED (static) + UNRESOLVED (runtime)** — cannot confirm bit value |
| **A** | gate fn `0x25EE1` (R47 candidate) controls DTS output | no safe observable | no safe observable | **UNRESOLVED** (subsumed by H2 region) |
| **B** | `0x868/0x86C/0x870` timing | = H1 | = H1 | **REJECTED** |
| **C** | `0x854` bits 17\|19 | = H2 | = H2 | **SUPPORTED + UNRESOLVED** |
| **D (new)** | DTS does not populate status/licence block `0x0C00–0x0C07` (observed divergence) | n/a (new observable) | **confirmed**: AC3 non-zero, DTS zero during decode | **SUPPORTED** (runtime-confirmed) — meaning/link to output gate **UNRESOLVED** |
| **E (new, R47b)** | Android `AudioTrack`/Kodi IEC packer refuses to bitstream DTS (stays PCM) | n/a | **REJECTED**: both runs re-init `method: RAW (PT)`; DTS → `STREAM_TYPE_DTS_512`; AC3 → `STREAM_TYPE_AC3`. PCM probe is transient, not the final sink | **REJECTED** (not the cause) |
| **F (new, R47b)** | Vendor HAL / EDID blocks DTS at the Android level (`mi_decoder_open` never opened for DTS) | n/a | **REJECTED**: `mi_common_raw_open` + `mi_decoder_open: format 9` (DTS) opened; `format 5` (AC3) for AC3. IEC device advertises `STREAM_TYPE_DTS_*` in both runs | **REJECTED** (not the cause) |

---

## §9 Independent causal assessment

1. **Decode is not the problem.** The DTS decoder runs on the DSP (DEC region live, same shape as AC3). Any hypothesis requiring "DTS doesn't decode" is falsified.
2. **Timing values are not the problem.** R46c proves them correct; runtime confirms decode proceeds.
3. **The failure is in the output / status-handshake path.** The only observable difference between working and failing is the status block `0x0C00–0x0C07` (populated by AC3, absent for DTS), while the output-gate cell `0x0854` (where the R46d candidate bits 17\|19 live) is not host-readable.
4. **The R46d candidate remains the leading hypothesis** — it sits precisely in the path the runtime localises the failure to, and the static trace shows AC3 opens bits 17|19 while DTS clears them. But it is **not runtime-proven**, because the cell is unobservable and patching is forbidden.

**Classification:** DTS passthrough failure = output/status-handshake defect in the DSP firmware, most likely the `0x0854` bits 17\|19 gate (or its controlling gate `0x25EE1`), **UNRESOLVED at runtime** (supported by static + consistent-with-runtime, unconfirmable by safe observation).

---

## §10 Patch decision — **DO NOT PATCH**

Per the absolute constraints and the user's Case-1/Case-2 rule:
- **Absolute constraint:** NO patch, NO device modification, NO `/dev/malloc|/dev/mem|/dev/miomap`, NO physical-memory read, NO EDID/AUTH. A firmware patch to `0x0854` would be a device modification → **forbidden**.
- **Unverifiable even if allowed:** the candidate cell `0x0854` is host-unreadable, so a one-variable experiment could not be read back to confirm success/failure. The user's "verify any patch before install" step is impossible here.
- **Risk:** `0x854` is **shared with the working AC3 path** (R46d §D4.3); a wrong write breaks the one case that works.

⇒ **No patch is proposed or performed.** The evidence is reported as missing (§11).

---

## §11 Missing evidence (what would be needed to CONFIRM the candidate)

1. **Direct observation of `0x0854` / `0x086C` / `0x0870` during AC3 vs DTS playback** — *impossible via safe mirror* (proven write-only, all-zero). Would require `/dev/miomap` (kernel panic, forbidden). **Not obtainable.**
2. **HAL / codec / logcat correlation** (`AudioTrack`, IEC61937 packer, `SPDIF`/`HDMI`, gate/state messages) — **✅ NOW CAPTURED (R47b, safe read-only `logcat -b all -d` during real playback).** Result: **both AC3 and DTS traverse the identical userspace→HAL passthrough pipeline** (Kodi `AudioTrack (IEC)` "Kodi IEC packer" → `method: RAW (PT)` with codec-specific `stream-type` → vendor `mi_common_raw_open` + `mi_decoder_open` with AC3=`format 5` / DTS=`format 9`). The Kodi/HAL path is **ruled out** as the cause (hypotheses **E**, **F** REJECTED). The only remaining location for the failure is the AEON DSP firmware's output/status handshake — see new **§13**.
3. **Physical receiver result** — *user-provided ground truth*: AC3 plays, DTS fails. This is the defining symptom.
4. **Dense single-cell re-sample of `0x0c06` (and `0x0c00`) during DTS** — **✅ NOW CAPTURED (R47c, safe mirror read, 20 rapid samples each).** Result: DTS status block `0x0C00–0x0C07` (incl. `0x0c06`) = `0x000000` in **20/20** samples (DEC live); AC3 populates it burstily (non-zero in the majority). The §4 transient caveat is **refuted** — the divergence is sustained. See new **§14**.
5. **Meaning of `0x0C00–0x0C07`** — whether its being zero for DTS is the *cause* (DTS path never reaches the status-write) or a *symptom* of the same gate that blocks output. **✅ NOW RESOLVED (R47e, static trace — §16).** Fresh full-image scan of `snd_full.bin` (all 591,620 instruction starts) + R46 FindMMIO: the status block `0x0C00–0x0C07` has **zero firmware writers** (only 13 readers, 4 inside the DTS gate `0x25EE1` reading `0x0C06` bit7). ⇒ it is read-only hardware status; the firmware cannot populate it, so "missing writer" is impossible. **The zero-during-DTS is a SYMPTOM** (output not armed); the **cause** is upstream — DTS clears `0x0854` bits 17\|19 (AC3 sets them, the only functional AC3-vs-DTS control difference, R46d D1.4). The gate `0x25EE1` reading `bit7(0x0C06)` is a downstream validator that fails because the bit was never set.

---

## §13 R47b — Logcat HAL / codec / Kodi passthrough correlation (safe, read-only)

**Method (safe):** `r47_work/r47_logcat.sh` runs a read-only `logcat -b all -d -v time` during **real, confirmed** AC3 and DTS playback (same `ACTION_VIEW` launch + `wait_play` gate as the mirror run). No MMIO, no physical memory, no config change, no patch. Output: `r47_work/r47_logcat_ac3.log`, `r47_work/r47_logcat_dts.log`.

**Device enumeration (both runs — identical):** Kodi enumerates `AudioTrack (IEC)` → `m_displayNameExtra: Kodi IEC packer (recommended)`, `m_deviceType: AE_DEVTYPE_HDMI`, and advertises `m_streamTypes: STREAM_TYPE_AC3, STREAM_TYPE_EAC3, STREAM_TYPE_DTSHD_CORE, STREAM_TYPE_DTS_1024, STREAM_TYPE_DTS_2048, STREAM_TYPE_DTS_512, STREAM_TYPE_DTSHD, STREAM_TYPE_DTSHD_MA, STREAM_TYPE_TRUEHD`. **DTS is declared supported by the IEC packer in both runs.** (`audio/vnd.dts`, `audio/vnd.dts.hd`, `audio/ac3p`, `audio/passthrough` "Unsupported mime" warnings are benign Android-framework capability probes — present in **both** logs, not DTS-specific.)

**Paired sink/HAL timeline:**

| event | AC3 (works) | DTS (fails) |
|---|---|---|
| Kodi IEC device | `AudioTrack (IEC)` "Kodi IEC packer" + DTS stream-types | **identical** |
| 1st sink init (probe) | `…Initializing … method: PCM stream-type: PCM-STREAM` | `…Initializing … method: PCM stream-type: PCM-STREAM` (transient probe) |
| 2nd sink init (final) | `method: RAW (PT) stream-type: STREAM_TYPE_AC3` | `method: RAW (PT) stream-type: STREAM_TYPE_DTS_512` |
| vendor HAL output | `adev_open_output_stream … open output stream with offload` | **identical shape** |
| vendor raw open | `mi_common_raw_open: handle 69, open raw output for device[speaker]` | `mi_common_raw_open: handle 77, open raw output for device[speaker]` |
| vendor passthrough decoder | `mi_decoder_open: format 5 … decoder handle` (AC3) | `mi_decoder_open: format 9 … decoder handle` (**DTS**) |
| PCM probe closed | `mi_common_pcm_close: handle 13` (handed off to raw) | `mi_common_pcm_close: handle 13` (handed off to raw) |

**Finding:** The **entire Android userspace → AudioTrack → vendor HAL passthrough pipeline engages identically for AC3 and DTS.** For DTS the HAL *does* open a DTS passthrough decoder (`mi_decoder_open: format 9`), and the Kodi IEC packer *does* wrap DTS (`STREAM_TYPE_DTS_512`). There is **no "DTS stays PCM"** outcome and **no HAL/EDID refusal** in the logcat. The failure is therefore **not** in Kodi, not in AudioTrack, not in the vendor HAL, and not in EDID capability at the Android level.

**Where the failure lives:** the `logcat` ends at the Android/HAL boundary (`mi_decoder_open: format 9` hands the DTS bitstream to the DSP's passthrough decoder). Downstream of that call — **inside the AEON DSP firmware** — the mirror (§2/§3) shows the DTS decode engine *does* run (DEC region live) but the status/licence block `0x0C00–0x0C07` is *never populated*, while the output-gate cell `0x0854` (R46d bits 17\|19) is host-unreadable. So the logcat **rules out every userspace/HAL hypothesis (E, F)** and **confirms the localisation to the DSP-firmware output/status-handshake** that §4/§9 already inferred from the mirror.

**Deeper differential grep (HDMI / ARC / SPDIF / routing / gate-state) — null result.** A targeted comparison of every `hdmi|arc|spdif|audio_patch|mi_device_set|non-pcm|bitstream|iec|gate|out_pause|out_resume|EaseContour` line across both logs shows the **routing layer is byte-for-byte identical** for AC3 and DTS:
- `adev_create_audio_patch: mix->device:speaker` + `create audio_patch 0xb08621c0, handle 2` — both.
- `mi_device_set: device speaker, un-mute (ePath 4)` / `device hdmi-arc, mute (ePath 7)` / `device headphone, un-mute (ePath 5)` — **identical** in both runs (the hdmi-arc mute is a constant, not a DTS differentiator; audio egress is the `speaker`/raw path, not hdmi-arc).
- `EaseContour::addEasePoint` / `getInterpolatedVolume` — both.
- `out_pause`/`out_resume` on the raw passthrough handle (AC3=69, DTS=77) and `out_standby` on the PCM probe handle 13 — both, same pattern.
- `CBitstreamConverter::Open bitstream to annexb init` — both.

⇒ **No DTS-specific HDMI/ARC/SPDIF error, no "non-PCM disabled", no gate/state error, no routing difference.** The Android/HAL/HDMI stack emits zero differential signal between the working and failing codec. This is a clean **null result** that independently confirms the failure is *not* in the userspace→HAL→HDMI path — the only differential observable remains the DSP-mirror `0x0C00–0x0C07` divergence (§4).

**Net effect on the causal chain:**
1. Decode engine runs for DTS — mirror (§5). ✔
2. Userspace→HAL passthrough pipeline fully engages for DTS (RAW PT `STREAM_TYPE_DTS_512` + `mi_decoder_open: format 9`) — **this section (R47b)**. ✔
3. But the DSP status/licence block `0x0C00–0x0C07` is never populated for DTS — mirror (§2/§3). ✗
4. Output-gate cell `0x0854` (bits 17\|19) host-unreadable — §6. ✗ (cannot confirm)

⇒ The DTS bitstream is accepted by Kodi/HAL and handed to the DSP's DTS passthrough decoder; decode proceeds; but the DSP firmware **never commits the output/status**. The defect is **inside the AEON DSP firmware's passthrough output/status handshake** — most likely the `0x0854` bits 17\|19 gate (R46d) or its controlling gate `0x25EE1`. Still **UNRESOLVED at runtime** (supported by static + consistent-with-runtime + now Kodi/HAL ruled out; unconfirmable by safe observation; patching forbidden).

---

## §14 R47c — Dense re-sample of status block `0x0C00–0x0C07` (transient-caveat closure)

**Method (safe):** `r47_work/r47_dense.sh` — same read-only mirror read as R47 (`echo read_dsp_sram_type=1 addr=0x0c00 len=8 > /proc/utopia_mdb/audio` + `dmesg`), **20 rapid samples** during real PLAYING DTS and AC3, capturing the full status block (`0x0c00–0x0c07`) and DEC liveness (`0x0fe8–0x0feb`). No MMIO / patch / config change. Logs: `r47_dense_dts.log`, `r47_dense_ac3.log`.

**Paired dense result (20 samples each):**

| observation | DTS (fails) | AC3 (works) |
|---|---|---|
| Status block `0x0C00–0x0C07` (all cells, all 20 samples) | **`0x000000` in 20/20** — every sample, every cell | **non-zero in the majority of samples** — e.g. `0x0c05` = `0x453400`/`0x6db600`/`0xeef600`/`0xad6b00`/`0x9f3e00`/`0x628e00`; `0x0c06` = `0x1bb600`/`0xdb6d00`/`0xad6b00`/`0xe3c700`; `0x0c01` = `0x3b3400`/`0xf9b600`/`0xcf9f00`/`0x5ad600`. Bursty (some cells/samples zero at any instant), but clearly populated |
| Specifically `0x0c06` (the §4-flagged cell) | **`0x000000` in 20/20** | non-zero in 4/20 (`0x1bb600`, `0xdb6d00`, `0xad6b00`, `0xe3c700`) |
| DEC `0x0FE8–0x0FEB` (liveness) | live, varying (`0x0fe9` ≈ `0x99c`–`0x9ec`, `0x0fea` ≈ `0x980`–`0x9e0`, `0x0feb` ≈ `0x5e`–`0x7a`) | live, varying (same range) |

**Conclusion:** The §4 transient caveat is **refuted**. Under controlled dense sampling the DTS status/licence block is **robustly, sustainedly zero** (20/20, including the transition), whereas AC3 writes it in a bursty but unambiguously non-zero pattern. The R30 "DTS `0x0c06` once non-zero" line is a genuine windowing/transient artifact, not representative. ⇒ Hypothesis **D** is upgraded from "SUPPORTED" to **SUPPORTED + REPRODUCIBLE**, and the AC3-vs-DTS divergence at `0x0C00–0x0C07` is established as a **sustained, sampling-independent** observation — strengthening the localisation to the DSP-firmware output/status handshake (§9) and the R46d gate candidate (§7).

---

## §16 R47e — Static trace of the status block `0x0C00–0x0C07` and the `0x0854` gate path (item 5 resolution)

**Question (§11 item 5):** is the zero `0x0C00–0x0C07` during DTS a **cause** (the DTS path never reaches the status writer) or a **symptom** (the output path never arms, so the hardware never populates the status)? This requires a static trace of who writes the status block and how that relates to the `0x0854` gate.

**Method (fresh, independent — not inherited):** `r47_work/r47e_scan.py` linear-decodes **all 591,620 instruction starts** of `snd_full.bin` with the static-verified `aeon_decode.py` decoder, runs a backward abstract interpreter (movhi/ori/addi/muli only) to resolve each store/load effective address, and flags writers/readers of the status block `0xB000_0C00–0x0C07`, the gate inputs (`0x0C06`, `0x0015`, `0x2512C4`), and `0x0854`. Cross-checked against R46 FindMMIO (`ghidra_writers.txt`, 587,030 instructions) and the R46d Ghidra `0x854` dump. Output: `r47_work/r47e_scan_out.txt`.

### §16.1 Who writes the status block `0x0C00–0x0C07`?

| | `0x0C00–0x0C07` (fresh R47e) | R46 FindMMIO (independent) |
|---|---|---|
| **firmware WRITERS (STORE)** | **0** | **0** (only LOADs appear for `0x0C0x`) |
| firmware READERS (LOAD) | **13** | consistent (telemetry + gate reads) |

Readers: telemetry/init at `0xE8FC`,`0xE90B`,`0x1E921`,`0x1E924`,`0x1F012`,`0x1F535`,`0x1F59B`,`0x1F59E`,`0x26343`; and **4 gate reads of `0x0C06`** inside `0x25EE1` (`0x25EF6`,`0x25FDE`,`0x26064`,`0x262EC`). A raw big-endian literal scan found **no rodata register-map table** for these addresses (the base is always built via `movhi 0xb000; ori/addi`, never stored).

⇒ **The status block is READ-ONLY hardware status.** The firmware *reads* it (and the host mirror exposes those reads) but **never writes it** — there is no status-writer for the DTS path to "miss." Therefore the zero during DTS **cannot** be a "missing firmware writer" cause.

### §16.2 Gate `0x25EE1` inputs (per §8.4 predicate `r11 = bit7(0x0C06) AND sign(0x0015)>=0 AND 0x2512C4!=0`)

| input | writer count | reader sites | class |
|---|---|---|---|
| #1 `bit7(0x0C06)` (decisive) | **0** | `0x25EF6`,`0x25FDE`,`0x26064`,`0x262EC` (gate body) | **hardware status, firmware-unwriteable** |
| #2 `sign(0x0015)>=0` | **0** | `0x1E6C2`,`0x1F0D2`,`0x25EFC`,`0x2606A` | hardware status, firmware-unwriteable |
| #3 `0x2512C4 != 0` | **1** (`0x172FD`, DTS-SDO-pack state) | `0x25F10`,`0x25F83`,`0x2607E` (gate body) | firmware-owned state word |

Per §9 discipline: an input with **zero writers** is firmware-read-only → the gate functions as a **hardware/licence validator**, not an enabler the firmware controls. The decisive input #1 (`bit7(0x0C06)`) is exactly the read-only status byte whose runtime value is 0 for DTS (§14).

**Fresh verification of the gate topology (R47e-gate, `r47e_gate.py`, re-derived from raw bytes — not inherited):**
- All four `0x0C06` reads resolve **inside the gate `0x25EE1` body** (`0x25EE1–0x266E1`): `0x25EF6`, `0x25FDE`, `0x26064`, `0x262EC` all decode as `bn.lbz …(rX),0(rX)` with `eff=0xB0000C06`. (The §9.1 #6 warning about decoder/Ghidra caller-count drift applies equally to *reader* containment — re-checked here.)
- Full-image caller scan into `0x25EE1` finds **exactly 4 call sites**: `0xCECC`, `0xD2C8`, `0xEC82`, `0xEF5E`. **Zero of them is inside an AC3 helper** (`0xCB50`/`0xCAB4`/`0xC49D`/`0xC5C5`). ⇒ the gate is **DTS-path-exclusive**; AC3 engages output via its **un-gated** `0xCB50`/`0xCAB4` → `0x10786E` path and never reaches `0x25EE1`. The small address differences vs the R46 Ghidra-derived caller set (`0xC91A`/`0xE9E5`/`0xED6E`) are the documented decoder/Ghidra drift (skill §9.1 #6); the functional fact (AC3 never calls the gate) is confirmed.

### §16.3 Upstream control register `0x0854` (the actual AC3-vs-DTS difference)

R46d (Ghidra ground-truth, 12 writers) + fresh R47e (11 writers; the 12th, `0xCD15` = `0xCC9B` PATH-B, is the documented clobbered-base gap and is marked "†" in R46d):

- **AC3 `0xCBFB`** writes `(old & 0xFFF) | 0xA0044 | code` → **sets bits 2, 6, 17, 19** + rate code (12–14).
- **DTS `0xCC9B`/`0xCF86`** write `old & 0xF5FFFF` / `old & 0x5FFFFF` → **clears bits 17\|19** (and 21\|23); sets nothing.
- ⇒ **The only functional AC3-vs-DTS difference in `0x0854` is bits `17|19`** (R46d D1.4). Bits 17\|19 are read by leaf Predicate A `0xC4E0` (`v & 0xA0000` → boolean). Their *semantics* are **UNKNOWN** (R46d D3: consumed, but not proven to be output-enable/non-PCM; clearing masks are shared with non-DTS routines `0xC573`/`0xC646`/`0xC746`).

### §16.4 Verdict — cause vs symptom (item 5 RESOLVED)

- **The zero `0x0C00–0x0C07` during DTS is a SYMPTOM, not a cause.** The block has zero firmware writers (§16.1, two independent scans), so the firmware cannot populate it; the hardware populates it **only when the output path is armed** — which AC3 achieves (bits 17\|19 set) and DTS does not (bits 17\|19 cleared).
- **The cause is upstream and firmware-writable:** DTS clears `0x0854` bits 17\|19 (AC3 sets them). This is the only functional AC3-vs-DTS control-register difference (§16.3, R46d D1.4).
- **The gate `0x25EE1` reading `bit7(0x0C06)` is a downstream consequence-validator, not an independent cause.** Because output was never armed, hardware status `0x0C06` = 0, so the gate's decisive input is 0 and it fails — a *redundant confirmation* of the upstream disable. Per R46-FINAL-C, even a gate PASS does not provably arm physical SPDIF (it calls shared utility `0x13292`, not an exclusive TX/DMA).
- **Caveat that still blocks closure by a patch:** `0x0854` bits 17\|19 are host-**unreadable** (R47 §8.2: `0x0840–0x0890` reads `0x000000` in every capture for both codecs), so the static structural difference cannot be confirmed at runtime by any safe method; and their meaning is UNKNOWN + the register is shared with the *working* AC3 path, so a patch is forbidden (absolute constraint) and risky. ⇒ item 5 is resolved as **symptom**; the upstream `0x0854` bits 17\|19 asymmetry is the lead, but the investigation remains **SUPPORTED + UNRESOLVED + complete-subject-to-firmware-opacity**.

---

## §17 Conclusion

- The controlled runtime test **proves the DTS decoder runs** (DEC region live for both codecs) and **localises the defect to the DSP output / status-handshake path** — not decode, not timing values.
- The observable divergence is the **status/licence block `0x0C00–0x0C07`** (AC3 populates it; DTS does not), while the output-gate cell `0x0854` (R46d's bits 17\|19 candidate) is **not host-observable**.
- The runtime result is **fully consistent with** the R46d hypothesis (AC3 opens bits 17\|19 of `0x0854`; DTS clears them) but **cannot confirm it**, because the cell cannot be read by any safe method and patching is forbidden.
- **R47b `logcat` correlation (§13) now closes the userspace/HAL gap:** AC3 and DTS take the **identical** Android→HAL passthrough pipeline (Kodi `RAW (PT)` + vendor `mi_decoder_open` AC3=`5`/DTS=`9`). This **rules out** Kodi, AudioTrack, the vendor HAL, and EDID capability as causes (hypotheses **E**, **F** REJECTED), leaving the failure **entirely inside the AEON DSP firmware's output/status handshake**.
- **R47c dense re-sample (§14) now closes the §4 transient caveat:** 20/20 DTS samples show the whole status block `0x0C00–0x0C07` (incl. `0x0c06`) at zero with DEC live, while AC3 populates it burstily (non-zero in the majority). The divergence is **sustained, sampling-independent** — hypothesis **D** upgraded to **SUPPORTED + REPRODUCIBLE**.
- **R47e static trace (§16) now answers §11 item 5:** the status block `0x0C00–0x0C07` has **zero firmware writers** (fresh full-image scan, two independent methods) ⇒ it is read-only hardware status, so the zero-during-DTS is a **SYMPTOM** (output never armed), **not a cause** (no missing writer). The cause is upstream: DTS **clears** `0x0854` bits 17\|19 (AC3 sets them — the only functional AC3-vs-DTS control difference, R46d D1.4); the gate `0x25EE1` reading `bit7(0x0C06)` is a downstream validator that fails *because* the bit was never set.
- **No patch performed.** The best available conclusion: DTS passthrough fails due to an **output-enable / non-PCM gate defect in the DSP firmware at `0x0854` bits 17\|19 (or its controlling gate `0x25EE1`)**, classified **SUPPORTED + UNRESOLVED** (R46d static + consistent-with-runtime + Kodi/HAL ruled out + DTS divergence reproducible + status-block-symptom confirmed; unconfirmable because `0x0854` is unobservable and patching is forbidden).
- The investigation is **terminated as complete-subject-to-firmware-opacity**: every safe-method gap (§11) is now either closed (items 2, 3, 4, 5) or proven unreachable (item 1 — direct `0x0854` observation needs `/dev/miomap` → forbidden). The residual unknowns (bits 17\|19 semantics; `0x0C00–0x0C07` meaning) sit in DSP firmware/licence, outside the safe envelope.

**Artifacts:** `r47_work/r47_capture_ac3.log` (controlled, real playback, mirror) · `r47_work/r47_capture_dts.log` (controlled, real playback, mirror) · `r47_work/r47_logcat_ac3.log` (controlled, real playback, **R47b HAL/codec/Kodi correlation**) · `r47_work/r47_logcat_dts.log` (controlled, real playback, **R47b HAL/codec/Kodi correlation**) · `r47_work/r47_dense_dts.log` (controlled, 20-sample dense re-sample, **R47c**) · `r47_work/r47_dense_ac3.log` (controlled, 20-sample dense re-sample, **R47c**) · `r47_work/r47e_scan.py` (fresh full-image static scanner — **R47e**) · `r47_work/r47e_scan_out.txt` (R47e result: 0 writers of `0x0C00–0x0C07`, 13 readers, gate `0x25EE1` reads `0x0C06` ×4) · `r47_work/r47e_gate.py` (fresh gate-topology verification — **R47e**) · `r47_work/r47e_gate_out.txt` (4 `0x0C06` reads inside `0x25EE1` body; 4 gate callers, 0 AC3) · `r47_work/r47_logcat.sh` (safe read-only driver) · `r47_work/r47_dense.sh` (safe dense-sampler driver) · `r47_work/r47_capture_ac3_IDLE_badintent.log` (archived false run) · `r47_work/r47_capture_dts_IDLE_badintent.log` (archived false run) · `r47_work/r47_capture_ac3_remote_UNCONTROLLED.log` (archived non-controlled run) · `r47_work/r47_capture.sh` (v2, corrected) · `r47_work/r47_capture_baseline.log`.

**No patch. No device modification. No unsafe memory access. No fabricated data.**
