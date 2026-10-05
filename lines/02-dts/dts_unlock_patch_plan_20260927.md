# DTS Unlock Patch Plan — LongCat Investigation 2026-09-27

**Status:** Analysis complete, patch plan prepared
**Constraint:** NOT APPLIED — awaiting authorization
**Target:** `patch_baseline/utpa2k_stock.ko` (MD5 `2fc6e9fc46b6402d3b9cbe0a`)

---

## 1. Root Cause Summary

The stock `utpa2k.ko` firmware contains **205 self-loop instructions** (`0xEBFFFFFE` = `BL .`) that replace original `BL MDrv_AUTH_IPCheck` calls in `CheckHashkey` (VA `0x42DCA8`). This is a **MStar factory patch** — AUTH checks are disabled at the firmware level.

**Consequence:** DTS IPIDs `{0xf, 0x3a, 0x12, 7}` are never checked. The device relies solely on OTP/strap state, which is not provisioned for DTS on this unit.

---

## 2. Patch Strategy

### Option A: Restore AUTH BL Instructions (Recommended)

**Goal:** Replace patched BL instructions with proper calls to `MDrv_AUTH_IPCheck` (VA `0x1390C`).

**Critical patch sites (DTS-related):**

| VA | Original | Patched | Context |
|---|---|---|---|
| `0x42FA20` | `BL 0x1390C` | `0xEBFFFFFE` | DTS core check (IPID 0xf) |
| `0x42FA4C` | `BL 0x1390C` | `0xEBFFFFFE` | DTS:X check (IPID 7) |
| `0x42FA5C` | `BL 0x1390C` | `0xEBFFFFFE` | Error path |
| `0x42FA60` | `BL 0x1390C` | `0xEBFFFFFE` | Post-error handling |
| `0x42FA7C` | `BL 0x1390C` | `0xEBFFFFFE` | Final check |

**BL instruction calculation:**
```
Target = 0x1390C (MDrv_AUTH_IPCheck)
BL at 0x42FA20: offset = (0x1390C - 0x42FA20 - 8) >> 2 = 0xEF8FB9
BL instruction = 0xEBEF8FB9
```

**Patch bytes:**
```
VA 0x42FA20: 0xEBFFFFFE -> 0xEBEF8FB9 (BL MDrv_AUTH_IPCheck)
```

**Note:** Only patch the DTS-related sites (5 instructions). Do NOT patch all 205 sites — this may break other functionality.

### Option B: Patch DEC Gate (Alternative)

**Goal:** Modify the DEC gate to skip the license field check.

**Critical instruction:**
```
DEC VA 0x021fd6: bg.lwz r4,0x2e4(r11)  ; Read license field
```

**Patch options:**
- Replace with `bg.movhi r4,0x1` (force non-zero value)
- Or replace with NOP (skip the check)

**Note:** This requires DEC image patching, which is riskier than ARM patching.

### Option C: Use SYSTEM_Control (No Patch)

**Goal:** Use `MApi_AUDIO_SYSTEM_Control` to force DTS mode.

**Command:**
```
SetSpdifOutputMode=BYPASS
```

**Note:** This does NOT bypass the AUTH gate. It only changes SPDIF mode.

---

## 3. Recommended Patch (Option A)

### 3.1 Patch Site 1: DTS Core Check

**File:** `patch_baseline/utpa2k_stock.ko`
**VA:** `0x42FA20`
**File offset:** `0x43D074` (VA + 0xD654 - 0x10000)

**Original bytes:** `FE FF FF EB` (0xEBFFFFFE, little-endian)
**Patched bytes:** `B9 8F EF EB` (0xEBEF8FB9, little-endian)

**Verification:**
```python
import struct
with open('patch_baseline/utpa2k_stock.ko', 'rb') as f:
    data = f.read()
# Check original bytes at file offset 0x43D074
offset = 0x43D074
original = struct.unpack_from('<I', data, offset)[0]
assert original == 0xEBFFFFFE, f"Expected 0xEBFFFFFE, got 0x{original:08X}"
# Apply patch
struct.pack_into('<I', data, offset, 0xEBEF8FB9)
```

### 3.2 Patch Site 2: DTS:X Check

**File:** `patch_baseline/utpa2k_stock.ko`
**VA:** `0x42FA4C`
**File offset:** `0x43D0A0`

**Original bytes:** `FE FF FF EB`
**Patched bytes:** `B9 8F EF EB` (same target)

### 3.3 Patch Site 3-5: Error Path

**File:** `patch_baseline/utpa2k_stock.ko`
**VA:** `0x42FA5C`, `0x42FA60`, `0x42FA7C`
**File offsets:** `0x43D0B0`, `0x43D0B4`, `0x43D0D0`

**Original bytes:** `FE FF FF EB`
**Patched bytes:** `B9 8F EF EB`

---

## 4. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Break other AUTH checks | Medium | High | Only patch DTS-related sites |
| Kernel panic | Low | High | Test in safe environment first |
| DTS still doesn't work | Medium | Medium | May need additional patches |
| Device brick | Very Low | Critical | Keep stock backup, have recovery plan |

---

## 5. Testing Plan

1. **Pre-patch verification:**
   - Record MD5 of stock `utpa2k.ko`
   - Verify device is in working state (AC3 works)
   - Record baseline behavior

2. **Patch application:**
   - Apply patch to `utpa2k.ko`
   - Verify patched MD5
   - Push to device (if authorized)

3. **Post-patch verification:**
   - Reboot device
   - Check AC3 still works (regression test)
   - Test DTS playback
   - Check for kernel panics

4. **Rollback plan:**
   - Keep stock `utpa2k.ko` backup
   - If issues, restore stock and reboot

---

## 6. Alternative: DEC Gate Patch (Option B)

If Option A fails, patch the DEC gate instead:

**File:** `aeon_validate/dec_work/dec_full.bin`
**VA:** `0x021fd6`
**File offset:** `0x021fd6`

**Original bytes:** `E6 02 8B EC` (bg.lwz r4,0x2e4(r11))
**Patched bytes:** `00 00 A0 E3` (MOV r0, #0 — force success)

**Note:** This is riskier as it modifies DSP firmware.

---

## 7. Conclusion

**Recommended approach:** Option A (restore AUTH BL instructions)

**Rationale:**
- Restores original firmware behavior
- Minimal patch (5 instructions)
- Reversible
- Does not modify DSP firmware

**Next steps:**
1. Obtain authorization to patch
2. Apply patch to `utpa2k.ko`
3. Test on device
4. Verify DTS works

---

*End of patch plan. Awaiting authorization to proceed.*
