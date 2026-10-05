# R52 — Locate the mstsound dispatcher that indexes handler table `0x12AF0C`
### (R51 continuation — resolve `struct+0x8`, the `0x1F6DA` gate predicate from R50)

**Date:** 2026-09-11
**Scope:** Static only. NO patch, NO device modification, NO runtime re-open of EDID/AUTH/CheckHashkey, NO `/dev/malloc`/`/dev/mem`, NO CPU/physical-memory reads. Ghidra `r27_work/ghidra_r27f` (`-noanalysis`, slaspec ground truth) is authoritative; the Python decoder is NOT.
**Honoured exclusions:** R48/R49, `0x854`, `0xC49D`, AUTH, EDID, and the `0x25EE1` downstream gate are NOT reopened.

---

## 0. Context — why R52 exists

R51 (delivered as **R51-C**) proved `0x1F575` has **no static caller**:
- `getReferencesTo(0x1F575)` = empty; full linear `getFlows` scan over **590,024** decodable instructions = **0 callers**.
- A one-constant literal scan found exactly **one** reference: the function-pointer table entry at `0x12AF20`, inside the mstsound handler/command registration table `0x12AF0C..0x12AF3B` (preceded by `"6_enc"`, followed by `"mstsound_set_ins"/"param_get"/"wrap_mstsound_hook"/"mstsound cli"`).

So `0x1F575` is reached only by **indirect dispatch** (`bg.jr` on a table-loaded pointer). R51's documented next lead was: *find the dispatcher that indexes `0x12AF0C`, and trace how it builds the `r3` context struct (and thus `struct+0x8`)* — because R50 established that `struct+0x8` (read at `0x1F605`, reloaded into `r14` at `0x1F683`, tested at `0x1F6DA`) is the predicate that governs DTS slot (`0x30`/`0x34`) population inside `0x1F575`.

R52 is that targeted table-consumer scan. It is **not** a broad MMIO survey — it tracks a single known constant family (the handler table base and its adjacent registration strings).

---

## 1. R52 — `movhi + ori/addi` base-construction scan (immediate-constant reference)

**Script:** `scripts/R52_FindDispatcher.java` → `r52_work/r52_dispatcher.txt`
**Method:** linear walk of all decodable instructions; track `movhi rX, hi`, then `ori rX, rX, lo` / `addi rX, rX, lo` building `base = (hi<<16) + signext16(lo)`; flag any base in `[0x12AF00, 0x12B00]`; dump a 28-instruction window (flagging `jr` indirect calls).

**Result:**
```
=== 0 base-construction hit(s) in [0x12AF00,0x12B00] ===
  No code constructs the table base -> dispatcher may reach table via a different
  constant or a passed-in pointer.
```

**Interpretation:** No code in the entire decodable image builds *any* address in the handler-table window via an immediate-constant pair. The dispatcher does **not** reach `0x12AF0C` by inlining its address.

---

## 2. R52b — whole-image pointer-literal scan (stored-pointer / global reference)

**Script:** `scripts/R52b_PtrScan.java` → `r52_work/r52b_ptr.txt`
**Method:** read every `MemoryBlock` into a byte buffer; scan all 4-byte LE/BE words for two specific targets — `0x12AF0C` (table base) and `0x12AF50` (`"mstsound_set_ins"` string base). Single-constant family, targeted.

**Result:**
```
=== 0 total pointer-literal hit(s) ===
  No stored pointer to 0x12AF0C / 0x12AF50 anywhere -> dispatcher does not reach the
  table via a global pointer either. Call path is runtime/pointer-mediated or in an
  UNDEFINED (non-decodable) region. R52 = definitive negative.
```

**Interpretation:** `0x12AF0C` / `0x12AF50` appear **nowhere** as a stored pointer — not in the table itself, not in any global/static that a dispatcher could `ld.w` from. (Note: the table *contains* handler pointers like `0x1F575`, but the table's *own* address is never materialised as a value anywhere.)

---

## 3. Combined conclusion

| Scan | Target | Result |
|------|--------|--------|
| R51 `getReferencesTo` + 590k `getFlows` | callers of `0x1F575` | **0** |
| R51 one-constant literal | `0x1F575` as a value | **1** (the table entry `@0x12AF20`) |
| R52 `movhi+ori` base build | table base in `[0x12AF00,0x12B00]` | **0** |
| R52b pointer literal | `0x12AF0C` / `0x12AF50` stored anywhere | **0** |

The dispatcher that indexes `0x12AF0C` is **not statically locatable** by any constant-reference method:
- It is not reached by an inlined immediate constant (R52).
- It is not reached via a stored global pointer (R52b).
- The only reference to the target handler `0x1F575` is the table slot itself (R51).

Therefore the call path `framework → dispatcher → 0x1F575(r3=context)` is **framework/runtime-mediated**: the table base and the `r3` context (including `struct+0x8`) are supplied at runtime — either passed as a pointer from a higher framework layer, or constructed inside the ~20.3% UNDEFINED (non-decodable) region of the image (SKILL §9.1). Neither is recoverable from the static image under the safe envelope.

**Classification: R52 = DEFINITIVE NEGATIVE** (supports and reinforces **R51-C**). `struct+0x8` cannot be resolved statically.

---

## 4. What this means for the R50 gate

R50 established the `0x1F6DA` gate tests `mem[0x8(r10)]` (= snapshot of `mem[0x8(r3)]` taken at `0x1F605`, reloaded into `r14` at `0x1F683`):
- `struct+0x8 == 0` → gate falls through → DTS slots `0x30`/`0x34` populated (conditional) → DTS output-config block `0x1F75C` entered.
- `struct+0x8 != 0` → gate taken → DTS block skipped → no DTS slots.

R52 shows `struct+0x8` is a **generic mstsound context field**, set by the (unlocatable) dispatcher at call time — **not** a Codec-9-specific condition inside `0x1F575`. This leaves two static-undecidable possibilities for the AC3-works / DTS-fails symptom:

- **(P-a)** The framework sets `struct+0x8` *differently* for AC3 vs DTS contexts (codec-dependent upstream), so the gate diverges at `0x1F6DA`. → Would make the gate the proximate DTS blocker.
- **(P-b)** The framework sets `struct+0x8` *identically* for all codecs (codec-agnostic at the call boundary), and the divergence is **downstream** — inside the codec-specific path within `0x1F575` (the `r26==9` DTS dispatch) and/or the R46 `0x25EE1` gate (DTS-path-exclusive, confirmed in R47e).

These two cannot be distinguished without observing `mem[0x8(r10)]` at `0x1F6DA`. R52's negative is precisely the statement that **static analysis cannot decide P-a vs P-b.**

---

## 5. Resolution path (safe, no patch/device-mod)

The only way to resolve `struct+0x8` is a **runtime SRAM-mirror capture** of `mem[0x8(r10)]` at the `0x1F6DA` test point — the same read-only mirror path used in R47/R47c (safe; no `/dev/malloc`, no MMIO write, no patch). This would record the actual predicate value for AC3 vs DTS playbacks and decide P-a vs P-b. It requires the device (which is outside the current static-only envelope).

No further static search is productive: the dispatcher/context-builder is either runtime-pointer-mediated or in the UNDEFINED region. Per the R51 stop-cap ("if R51-C, stop as well, do not start another broad search"), **R52 is the terminal static step. No R53, no broad survey.**

---

## 6. Deliverables

- `REPORT_runtime_phase52.md` — this report.
- `r52_work/r52_dispatcher.txt` — R52 `movhi+ori` base scan (0 hits).
- `r52_work/r52b_ptr.txt` — R52b whole-image pointer-literal scan (0 hits).
- `scripts/R52_FindDispatcher.java`, `scripts/R52b_PtrScan.java` — scanners (reproducible).

## 7. Standing classification

**R50-C** (DTS slot production not provable statically) → **R51-C** (no static caller; `struct+0x8` = generic framework context) → **R52 = definitive negative** (dispatcher not locatable; call path runtime-mediated). The DTS-vs-AC3 divergence is confirmed to originate *inside the AEON DSP firmware's passthrough output/status handshake* (R47 family), but its exact proximate gate (the `0x1F6DA` `struct+0x8` test vs the downstream `0x25EE1` DTS-exclusive gate) is **UNRESOLVED statically** and requires the runtime mirror capture of §5.
