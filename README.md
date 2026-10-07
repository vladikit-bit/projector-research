# Projector research — Thundeal TD.98 Pro (MStar MT5889)

Forensic reverse-engineering of an Android 11 projector (Thundeal TD.98 Pro / C50A, MStar MT5889, Ntech ODM firmware), Aug–Oct 2026. Goal: unlock what the stock firmware caps — audio passthrough, Dolby Vision, display modes, 3D.

**Start here:**
- **[TIMELINE.md](TIMELINE.md)** — restored project history: what was changed, which hypotheses were tested, what was confirmed or refuted (2026-08-26 → 2026-10-04)
- **[STATUS.md](STATUS.md)** — current state of every research line + next steps
- **[CROSS_REFERENCES.md](CROSS_REFERENCES.md)** — 23 hypothesis → refutation chains
- **[DEVICE.md](DEVICE.md)** — device identification

**Five research lines** (in `lines/`, each with a SYNOPSIS + primary reports):

| Folder | Line | Status |
|---|---|---|
| `01-dd-spdif/` | DD/AC3 passthrough over SPDIF | ✅ works — libmi3 caps patch + Kodi RAW sink |
| `02-dts/` | DTS passthrough | ⏳ open — decoder runs, output blocked by two runtime words in the DEC DSP (170 documents) |
| `03-dolby-vision/` | Dolby Vision | ⏳ HW present; PQ binary lacks customer IP mode; colour calibration in progress |
| `04-picture-modes/` | Display modes beyond 1080p 50/60 Hz | ⏳ root cause proven (2 hardcoded HWC configs); ~250-byte patch designed, not emitted |
| `05-3d/` | 3D | ⏳ 120 Hz DLP panel present in firmware; Android plumbing missing |

**Companion repos:** [projector-research-base](https://github.com/vladikit-bit/projector-research-base) (all 285 research .md files, unsorted) · [projector-diagnostic-tools](https://github.com/vladikit-bit/projector-diagnostic-tools) (scripts & tooling) · [projector-passthrough-dd-restore](https://github.com/vladikit-bit/projector-passthrough-dd-restore) (OTA recovery guide + fix binaries) · [tcl-t615t-firmware-analysis](https://github.com/vladikit-bit/tcl-t615t-firmware-analysis) (donor firmwares for differential analysis).

> Research documents are written in Ukrainian (see [README.ua.md](README.ua.md)); this README is the English entry point.
