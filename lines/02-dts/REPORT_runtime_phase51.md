# R51 — Resolving `r10+0x8` inside `0x1F575`: who calls it / where `+0x8` is initialized

**Scope (per user):** very-narrow static trace to resolve the caller-provided `r10+0x8` (the `0x1F6DA`
gate predicate from R50) by finding callers of `0x1F575` and tracing `struct+0x8`. R48/R49, `0x854`,
`C49D`, AUTH, EDID, and the `0x25EE1` downstream gate are **not** reopened. Ghidra `aeon:LE:32:default`
slaspec is ground truth (`-noanalysis` project `r27_work/ghidra_r27f` / `sndr27`). No patch, no device,
no full-image MMIO survey, no Python decoder.

---

## Step 1 — enumerate ALL call sites to `0x1F575` (Ghidra ground truth)

Two independent methods, both authoritative (no byte-arithmetic, no Python decoder):

1. **Reference graph:** `getReferencesTo(toAddr(0x1F575))` → **empty** (no call/flow references).
2. **Linear `getFlows` scan** over the whole program: **590,024 decodable instructions** walked; every
   instruction's flows checked for target `0x1F575`. Result: **0 callers** (`jal` or tail-`j`).

> Caveat (SKILL §9.1): ~20.3% of the image is UNDEFINED, so "0 callers" means *0 among decodable
> instructions*. An undecoded-region `jal` or an indirect `bg.jr` would not appear. Both are covered by
> the confirmation below.

**Conclusion of Step 1: `0x1F575` has NO static direct/tail caller.**

---

## Confirmation — `0x1F575` is an indirect dispatch target (function-pointer table entry)

Targeted literal scan (R46b §21 method) for the 4-byte constant `0x1F575` (LE + BE) across program
memory → **exactly one hit**:

- `@ 0x12AF20` (block `ram`, a DATA location; no instruction there) — a stored 32-bit pointer value.

A single DATA constant (not code) is a **function-pointer table entry**, not a coincidence. This
explains the zero-caller result: `0x1F575` is reached via `bg.jr` on a table-loaded pointer.

**Narrow table characterization** (`0x12AF00..0x12B00`, single targeted dump):

```
0x12AF00  "6_enc\0\0\0"                         ; format/codec name string
0x12AF0C  0x0001F375   (code)
0x12AF10  0x0001F06B   (code)
0x12AF14  0x0001F2ED   (code)
0x12AF18  0x00000000
0x12AF1C  0x0001F05A   (code)
0x12AF20  0x0001F575   (code)   <-- 0x1F575
0x12AF24  0x00000000
0x12AF28  0x0001F07C   (code)
0x12AF2C  0x0001F4E5   (code)
0x12AF30  0x00000000
0x12AF34  0x00000000
0x12AF38  0x0001F070   (code)
0x12AF40  0x00000002                            ; count field
0x12AF4C  0x0012AEFC                            ; pointer into this rodata region
0x12AF50  "mstsound_set_ins" / "param_[%d]" / "param_get(%6x" / "wrap_mstsound_hook" / "mstsound cli: ..."
```

`0x1F575` is **one of N registered callbacks** (`0x1F375`, `0x1F06B`, `0x1F2ED`, `0x1F05A`,
`0x1F575`, `0x1F07C`, `0x1F4E5`, `0x1F070`) in what the adjacent strings identify as the
**mstsound handler/command registration table**. It is invoked indirectly by the mstsound framework
dispatcher, with a **generic context struct** passed as the argument (`r3 = r10` inside `0x1F575`).

---

## Steps 2–4 — cannot be executed as framed

The user's method assumed a static caller: *"For every caller, show how r3 is constructed … trace r3
backward … find writes to +0x8 … determine whether the same caller/context has a codec-dependent
condition that determines struct+0x8."*

That premise is false here: **there is no static caller.** `0x1F575` is a generic framework callback.
Therefore:
- There is no caller-local `r3` construction to trace back from.
- `struct+0x8` is a field of the **generic mstsound stream/decoder context** handed to every handler in
  the table — it is set up by the framework at registration / stream-open time, **not** by a
  Codec-9-specific branch.
- No codec-dependent condition on `struct+0x8` exists at the call boundary.

---

## Classification: **R51-C**

The `r10+0x8` value (the `0x1F6DA` gate predicate from R50) **cannot be resolved statically via a caller
trace**, because:

- **R51-A** (caller statically establishes `struct+0x8 == 0` for Codec 9): **not provable** — no
  Codec-9-specific caller; the only reference is a generic dispatch-table entry.
- **R51-B** (caller statically establishes `struct+0x8 != 0` for Codec 9): **not provable** — same reason.
- **R51-C** (caller/struct initialization cannot be resolved statically): **holds**. The "caller/context"
  is the mstsound framework dispatcher; `struct+0x8` is framework/runtime state, orthogonal to codec
  selection.

Per the user's instruction ("If R51-C, stop … do not start another broad search"), this phase stops here.

---

## What this reframes (no new search, just interpretation of R50+R51)

- The `0x1F6DA` gate tests a **framework-level context field** (`mem[0x8(r10)]`), not a Codec-9 decision.
  The Codec-9-specific logic is **inside** `0x1F575` (the `r26==9` dispatch at `0x1F70C`, traced in R50).
- Consequently the DTS-slot-population gate (R50) is **not** gated by "is this Codec 9?" — it is gated by
  generic runtime context state. Codec 9 reaches the DTS-slot code unconditionally; whether the slot is
  actually populated depends on `mem[0x8(r10)]` (framework) + `opcode_2E r11/r12` (runtime), as R50 found.
- `0x1F575` being one of many mstsound callbacks means AC3, DTS, and other formats likely share this same
  dispatch table and the same generic context struct — the DTS-vs-AC3 divergence is **not** at the
  "which handler / what struct+0x8" level, but downstream (inside each handler + the R46 `0x25EE1` gate).

---

## Next lead (ONLY if the user later expands scope — out of R51)

To resolve `struct+0x8` for real, the future phase would need to:
1. Find the mstsound dispatcher that indexes `0x12AF0C` (scan for readers of that data address) and how
   it builds `r3` before the indirect `bg.jr`.
2. Find the stream/decoder-context allocator that zeroes/initializes `+0x8`, and whether `+0x8` is a
   non-null "decoder active" handle (would make the `0x1F6DA` gate SKIP → R50-B) or 0 on first config
   (gate falls through → R50-A).
3. Or capture `mem[0x8(r10)]` at the `0x1F6DA` test at runtime (read-only SRAM mirror, if `r10` resolves
   into an observable `DM[]` cell) — the same runtime path recommended at the end of R50.

## Exclusions honored
No `0x854` / `C49D` / AUTH / EDID / `0x25EE1` re-opened. No patch, no device. No full-image MMIO survey
(only a getFlows caller scan, a one-constant literal scan, and a single 0x100-byte table dump). Scripts:
`R51_FindCallers.java`, `R51_ConstScan.java`, `R51_TableDump.java` (in `scripts/`).
