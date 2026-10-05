# FINDING — 2026-10-01: Dolby Vision on this projector is HARDWARE and fully declared — my earlier "maybe software" doubt is refuted

**Self-correction first:** I earlier searched `media_codecs.xml` with a case-sensitive
`grep -c dvhe`, got **0**, and concluded "no dvhe decoder is declared". That was wrong — the XML
uses `DVHE` in upper case. Case-insensitively there are **6 matches**. The DV decoder family is
present and is the vendor's dedicated MStar engine exposed through OMX.

## 1. What is declared

`/vendor/etc/media_codecs.xml` — the full Dolby Vision decoder family, each with a `.secure`
variant and `concurrent-instances=2`:

| Decoder name | profile |
|---|---|
| `OMX.MS.DOLBY_VISION.DVHE.DTR.Decoder` | DVHE single-layer (dual-track) |
| `OMX.MS.DOLBY_VISION.DVHE.STN.Decoder` | DVHE single-track |
| `OMX.MS.DOLBY_VISION.DVHE.ST.Decoder` | DVHE (ST) |
| `OMX.MS.DOLBY_VISION.Decoder` | generic DV |
| `OMX.MS.AVC.DOLBY_VISION.Decoder` | DV **enhanced** (dvhe/e, RPU on AVC) |
| `OMX.MS.DOLBY_VISION.DVAV.SE.Decoder` | Dolby Vision + AV1 sync-layer |
| `OMX.MS.DOLBY_VISION.DVAV1.Decoder`, `OMX.MS.AV1.DOLBY_VISION.Decoder` | AV1-based DV |

`/vendor/tvconfig/config/MM/media_codecs.xml` (the vendor TV config) declares the generic
`video/dolby-vision` entries (`OMX.MS.DOLBY_VISION.Decoder`, `OMX.MS.AVC.DOLBY_VISION.Decoder`,
`OMX.MS.AV1.DOLBY_VISION.Decoder`) each with `<Feature name="adaptive-playback"/>` — that is the
per-codec feature Kodi surfaces in its codec list.

**Kodi reporting `amc-dvhe(S) (HW)` is consistent with this**: `OMX.MS.*` Dolby Vision decoders
are the vendor's DV hardware block; the codec is declared, so the DV menu is **not cosmetic**.

## 2. What the earlier evidence says about this conclusion

The archive recorded `mi_decoder_open: format 9 got decoder handle 0x19000000` for DTS and the
DV banner appearing for the first time on 2026-09-30. The R2 log, the `HdrTypes=[1,2,3,4]`
capability and the now-declared DV decoders all line up: the DV pipeline exists end to end
(HDMI EDID → decode → panel) and only the **display calibration** is missing.

## 3. So the likely cause of the two symptoms the user reported

* **Wrong colours at Kodi start** — the DV PQ engine runs with an **all-zero calibration**
  (11 fields, see `VIDEO_SETTINGS_CHEATSHEET.md` §24): no white points, no gamma, no primaries.
  Without a display reference its tone mapping is undefined.
* **Terrible fps on re-entering the video** — separate symptom; most likely the DV engine being
  re-initialised per playback (the decoders are limited to `concurrent-instances=2`), not a
  missing hardware path.

## 4. Practical consequence

The DV menu is real. To get correct DV colour there are only two honest routes: **fill the PQ
calibration** (needs a colourimeter, or the factory panel data — it is not in the image), or
**use the Color Tuner / picture controls instead of the DV path**. There is no third option
where a flag makes an uncalibrated panel tone-map correctly.

Unrelated but worth remembering from the same search: **DTS is absent from every media codec
config** — consistent with the DTS audio problem being a build decision, and with the EDID
short-audio-descriptors carrying AC-3/E-AC-3 only.