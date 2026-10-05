
---

# 2026-09-30 — RETRACTION: the window is timing counters, not a ring

I called the `0x103F` / `0x105C` pair "a live ring readout". **That was wrong.** Stopping
playback and watching them settle the question:

```
DTS playing : 103f  1a1aa7 → 1a2254 → 1a2a14
STOPPED     : 103f  1a4a07 → 1a5a1d → 1a6a41 → 1a7a2c    Δ ≈ 4096 per 3.6 s
```

**They keep advancing at the same rate with nothing playing.** The constant `0x83`–`0x87`
separation is a fixed offset between two free-running counters, not a ring occupancy. Their
advance rate is also two orders of magnitude below the elementary-stream rate
(`dump_es` ≈ 208 KB/s vs ≈ 1.1–1.5 K units/s), so they are not on the DTS data path either.

**So: the readable window contains timing and state counters. It is not the ring, not the
elementary-stream path, and not the decoder SHM.** The "live ring readout" claim in the previous
section is retracted.

This is the **fourth** runtime claim I have had to retract this session (the SND mirror gate,
the `0x400` cell, the SRAM completeness negative, and now this). In every case the mistake was
reading structure into a handful of changing numbers. The standing rule — verify by re-reading
under a changed condition before interpreting — caught all four, but only after I made them.

## What survives, and it is the part that matters

Everything checked **statically** stands, because it was verified against the listing rather than
against moving values:

* the entry gate `beqi r23,0x5` on `[r10+0xC]`, `jr r9` vs tail-call `j 0x15f67`;
* the size gate reducing exactly to `[0x14] − [0x0C] > 0x0D` (verified `F_643DC` and
  `F_64256`→`F_64296` from `dec33_realcode.txt`);
* the `=4` tag write at `0x19862` and both `==4` gates at `0x19406` / `0x19434`;
* the burst arm's `[0x38] == 0x400`;
* the AEON `sw_0` / `lwz` encoder, verified on 24 859 instructions;
* the four flashed patches, all null, all reverted, AC-3 control passing every time.

## The final honest position

The DTS output path is present and complete in the firmware: decoder, conversion, IEC61937 burst
builder, type plumbing. The chain fails on two runtime words, with no codec branch anywhere
upstream. Four patches failed because they all acted on code. The block is in memory that is
**provably not** the readable window (a positive-control write to the SHM changes nothing there),
not host-writable, and not observable by any command found.

What remains is the SND side — where the burst is actually consumed and where no analysis has
ever been done, which is now possible for the first time because `snd33_realcode.txt` exists.
That is assigned to Muse. Everything else is closed.
