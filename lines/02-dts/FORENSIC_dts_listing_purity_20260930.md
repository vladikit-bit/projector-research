# DTS passthrough — listing purity audit, 2026-09-30

## The defect

`SND32_DumpAll.java` / `DEC32_DumpAll.java` sweep **every** offset and emit whatever the
sleigh decoder accepts:

```java
while (off < max) {
    Instruction ins = listing.getInstructionAt(a);
    if (ins == null) { off += 1; continue; }
    ...
}
```

`getInstructionAt` decodes on demand, so a data byte that is a legal encoding is emitted as an
instruction. **Re-running with analysis does not help** — I did, and got the identical
293 721 instructions. The sweep is unconditional, so no setting can prevent it.

Visible proof in the SND listing:

```
0002CF  bg.jal 0x0000b03c
0003CF  bg.jal 0x0000b03c      <- every 0x100 bytes: a data table rendered as code
…
0017E   bt.jr  …               <- inside the first table block
```

This is the true root cause of the three function-boundary errors made in this project. They
were not careless reading — **the input data does not distinguish code from data.**

## The filter

Derive entries from `jal` targets, walk forward following branches and instruction lengths,
keep only what is reached. Implemented in Python on the existing listings:

- `aeon_validate/dec_work/dec33_realcode.txt` — **504 216 of 637 122 (79 %)** reachable,
  3139 entries from `jal`
- `aeon_validate/snd_work/snd33_realcode.txt` — **253 507 of 293 718 (86 %)** reachable,
  1844 entries

## Result 1 — every DEC finding survives

All ten load-bearing addresses are real code:

```
0x190B5  [r10+0xC] type read              REAL CODE
0x190B8  beqi r23,0x5  type gate         REAL CODE
0x190F0  jr r9          !=5 arm           REAL CODE
0x19180  j 0x15f67      ==5 arm           REAL CODE
0x4604B  jal 0x643DC    size gate         REAL CODE
0x4604F  bgtui r3,0xd   size gate         REAL CODE
0x19860  movi r23,4                       REAL CODE
0x19862  sw 0xc(r12),r23                  REAL CODE
0x19406  beqi r23,0x4                     REAL CODE
0x19434  beqi r24,0x4                     REAL CODE
```

**The DEC side of the investigation is sound.** The entry gate, both arms, the size gate, the
`=4` tag write and both `==4` gates are all genuine code. So are the three flashed patches'
targets.

## Result 2 — the SND "mirror gate" is RETRACTED

`0x4FE0`, `0x500A` and `0x5142` are **absent from the clean listing** — they were data. The
"mirror of the DEC gate on the SND side" I reported is **void**; it came from ReKo decoding a
byte range that the reachability filter shows is not code.

What survives on the SND side is narrower: there are **14** (not 18) real `sw 0xa0(r10)` sites,
all with the same struct-copy shape, so `+0xA0` on that struct is still a propagated field.
**But I never verified any instruction that *reads* `0xA0` as a DTS gate**, so the SND side has
no located gate at all and must be redone against `snd33_realcode.txt`.

## Standing rules

1. **Any count from `*_clean.txt` is an upper bound** that includes data. Say so when quoting.
2. **Never quote a count from a listing without the reachability filter.**
3. For anything that will be patched, use ReKo on an exact byte range, or confirm against
   `*_realcode.txt`.
4. Branch/offset targets in these listings are **eight** hex digits, mnemonics carry **suffixes
   and trailing commas**, offsets are **lowercase** — five separate regex traps, each of which
   silently produced a wrong number today.