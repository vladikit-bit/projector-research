# The projector's SPDIF menu is a 4-item view onto the `SPDIF_MODE_FMT` value space

Date: 2026-09-26
Corrects the value-mapping discussion in
`forensic_space_bunny_spdif_control_surfaces_20260926.md` and the decoding of
`TVMenuSettingsService.updatePackageChanged` in
`forensic_space_bunny_gate_live_test_20260926.md`
Evidence: `forensic_space_bunny_spdif_value_space_evidence_20260926.txt`

---

## 1. Measured mapping — deterministic, sequential, one value at a time

| projector OSD item | index written | `g_audio__spdif` read back |
|---|---|---|
| off | 0 | 0 |
| PCM/ARC | 1 | **2** |
| RAW/ARC | 2 | **1** |
| AUTO | 3 | **6** |

Earlier readings of this key in this session (2, 6, 1, 7 and out-of-order values) were
taken while the key was being written by a second party and were not reliable. A
clean sequential test with settling time gives a stable one-to-one mapping.

## 2. It is exactly the `SPDIF_MODE_FMT_*` enum

```
Lcom/mediatek/twoworlds/tv/common/MtkTvConfigTypeBase;
    SPDIF_MODE_FMT_OFF                 = 0   <- "off"
    SPDIF_MODE_FMT_RAW                 = 1   <- "RAW/ARC"
    SPDIF_MODE_FMT_PCM16               = 2   <- "PCM/ARC"
    SPDIF_MODE_FMT_PCM24               = 3   <- (not exposed by the menu)
    SPDIF_MODE_FMT_DDP_PASS_THROUGH    = 4   <- (not exposed by the menu)
    SPDIF_MODE_FMT_DDP                 = 5   <- (not exposed by the menu)
    SPDIF_MODE_FMT_AUTO2               = 6   <- "AUTO"
```

**`g_audio__spdif` is not an OSD index — it is a `SPDIF_MODE_FMT` value.** The
projector menu is a 4-item view onto a 7-value format space. RAW/ARC is literally
`SPDIF_MODE_FMT_RAW`.

This also reconciles the Android list, which is a *different* enumeration over the same
key but whose two usable entries happen to land on the same numbers:

| Android «Цифровий вихід» | written | lands in `SPDIF_MODE_FMT` as |
|---|---|---|
| PCM | 2 | PCM16 = 2 — correct |
| Bypass | 4 | DDP_PASS_THROUGH = 4 — correct |
| Auto | 7 | out of range (0..6) |
| Dolby Digital | 1 | RAW = 1 — **name mismatch** |
| Dolby Digital Plus | 5 | DDP = 5 — correct |

The two settings the user actually uses, PCM (2) and Bypass (4), are both coherent in
this space. That is consistent with AC3 working in exactly those modes.

## 3. The RAW path — structurally dead, not merely unresearched

Verified across every APK pulled from the device: `g_audio__spdif` appears in only two
places.

```
TvSettingsPlus/classes7.dex  SoundFragment      -> writes it (the Android list)
TvServices/classes.dex       updatePackageChanged -> reads it (the Netflix gate)
```

**There is exactly one reader**, the Netflix gate. And its logic is:

```java
if (isNetflix != wasNetflix) {          // a com.netflix.ninja start/stop transition
    int v = getConfigValueInt("g_audio__spdif");
    if (v == 7) {                        // only armed by Android "Auto" == 7
        setSpdifFmt(isNetflix ? 5 : 7);  // sends DDP(5) or 7
    }
}
```

Consequences:

1. **RAW/ARC writes `v = 1`, and the gate's only test is `v == 7`. Selecting RAW/ARC
   therefore guarantees the gate never fires.** The user's chosen format is never read
   by anything, except as a yes/no "is this Auto?" flag.
2. **Even when the gate does fire, it sends `5` or `7` — never `1`.** So
   `setAudioInfoValue(0, SPDIF_MODE_FMT_RAW)` is **never called on this device**, by
   any app, in any state.
3. `7` is not a member of the `SPDIF_MODE_FMT` space at all. The gate's `== 7` test is
   a *UI-intent* test ("user chose Auto"), not a format test; the values it then sends
   (5 = DDP) are a Netflix-specific output override.

So the correct reading of the Netflix code is: **"when the user has Auto selected and
Netflix starts, force the output format to DDP"** — a per-app override for one
certified app, not a general passthrough mechanism.

**RAW is a fully specified, natively reachable format that this firmware has no path to
command.** `SPDIF_MODE_FMT_RAW` = 1, the API is `setAudioInfoValue(0, 1)`, the JNI is
live (proven: `setAudioInfoValue type=0 val=5` and `val=7` both reached
`MtkTvAvMode_jni`). Only the caller is missing.

## 4. What this means for DTS

The DTS work therefore does not need a new mechanism. It needs one line that forwards
the user's choice instead of the hardcoded Netflix literals:

```
currently:  setSpdifFmt(isNetflix ? 5 : 7)      under  if (v == 7)
needed:     setSpdifFmt(v)                       under  if (v == 1)   // RAW/ARC selected
```

Both `v` and the target enum are already in the same space, so no translation table is
required — which the earlier "four disagreeing value spaces" conclusion got wrong. The
OSD index and `SPDIF_MODE_FMT` are the same space after one indirection; the only
foreign values come from the Android list's `7`.

## 5. Open

- Whether the *native* treats `1` as raw passthrough is still unproven — the two values
  that were exercised (5, 7) do not establish it. The service-side handler for
  `a_mtktvapi_audio_info_set_value` remains unexamined.
- Whether the decoder actually starts for DTS (`+0x4D8`, `0x582`, SHM38) is still
  unproven and is the next question after the mode is commanded.
- `libmi3_dts_v3.so` (`|= 0x070002E1` in `MI_AUDIO_GetCaps`) is a *capability* change,
  not a mode change. The user's recollection is that it was tried without solving the
  problem, and that a licence-hash problem appeared during the broader attempts. Those
  are separate axes; neither establishes it as a dead end.
