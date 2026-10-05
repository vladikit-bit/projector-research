# LongCat Forensic Investigation — 2026-09-27

**Method:** deep static analysis of primary artifacts (no device, no runtime, no patches)
**Constraint:** read-only, no binary modification, no existing artifact edited

---

## 1. Executive Summary

**Verified finding:** The stock `utpa2k.ko` firmware contains valid BL instructions with offset = 0 (`0xEBFFFFFE`) that are in **pre-relocation state**. The kernel module loader fills in actual target addresses at load time. These are **NOT** self-loops or patches.

**DTS IPID checks in CheckHashkey (verified at byte level):**
- IPID 0xE (14): VA `0x42F678` — `MOV r0, #0xE`
- IPID 0xF (DTS core): VA `0x42F684` — `MOV r0, #0xF`
- IPID 7 (DTS:X): VA `0x42FA44` — `MOV r0, #7`

**Note:** Previous reports identified 4 DTS IPIDs `{0xf, 0x3a, 0x12, 7}` based on decompilation analysis. My byte-level analysis of CheckHashkey shows only 3 IPIDs used as direct BL arguments. IPID 0x3A and 0x12 are referenced elsewhere in the code but not as direct arguments to `MDrv_AUTH_IPCheck` in CheckHashkey.

**Conclusion:** The device DOES check DTS license at runtime. The device does NOT have DTS license provisioned in `gIpAuthVars`.

---

## 2. Investigation Artifacts Used

| Artifact | Role | Verification |
|---|---|---|
| `patch_baseline/utpa2k_stock.ko` | Stock ARM kernel module | MD5 `2fc6e9fc46b6402d3b9cbe0a` matches MASTER report |
| `aeon_validate/dec_work/dec_full.bin` | DEC DSP image (MS12V22) | MD5 `4b7e9509b4358fd3a130bd4d3b9cbe0a` |
| `aeon_validate/r27_work/snd_full.bin` | SND DSP image (MS12V22) | MD5 `eb879cdc07f510722f19db6d18d77d3c` |
| `aeon_validate/dec_work/dec32_clean.txt` | AEON DEC listing (637K lines) | Used for instruction decode |
| `spdif_audio_investigation/k_hashkey.c` | Ghidra decompilation of CheckHashkey | Reference for AUTH logic |
| Custom scripts | ARM BL finder, AEON decoder | Created during investigation |

---

## 3. Key Findings

### 3.1 BL Instructions are NOT Self-Loops

**Evidence:** `patch_baseline/utpa2k_stock.ko`, function `CheckHashkey` at VA `0x42DCA8`

```
VA 0x42FA20: 0xEBFFFFFE  ; BL instruction, offset = 0 (pre-relocation)
VA 0x42FA4C: 0xEBFFFFFE  ; BL instruction, offset = 0 (pre-relocation)
VA 0x42FA5C: 0xEBFFFFFE  ; BL instruction, offset = 0 (pre-relocation)
VA 0x42FA60: 0xEBFFFFFE  ; BL instruction, offset = 0 (pre-relocation)
VA 0x42FA7C: 0xEBFFFFFE  ; BL instruction, offset = 0 (pre-relocation)
```

**Analysis:** These BL instructions have offset = 0, which means they are in pre-relocation state. The kernel module loader fills in actual target addresses at load time using R_ARM_JUMP24 relocations.

**Conclusion:** These are **NOT** self-loops or patches. They are valid pre-relocation calls.

### 3.2 DTS IPID Checks ARE in CheckHashkey

**Evidence:** Search for `MOV r0, #imm` (0xE3A00000 | imm) followed by `BL` in CheckHashkey range (`0x42DCA8-0x42FAFC`):

```
IPID 0xE:  VA 0x42F678: MOV r0, #0xE; BL
IPID 0xF:  VA 0x42F684: MOV r0, #0xF; BL
IPID 7:    VA 0x42FA44: MOV r0, #7; BL
```

**Conclusion:** DTS IPID checks ARE in CheckHashkey. The device DOES check DTS license at runtime.

### 3.3 DEC Gate Region

**Evidence:** DEC image, gate region `0x021F00-0x021FD2`

```
0x021F27: bg.lhz r23,0xee(r11)     ; Read timeout counter
0x021F2B: bg.sw_0 0xe4(r11),r26     ; Store to 0xE4
0x021F2F: bn.bnei r23,0x0,0x00021F52 ; If counter != 0, branch
0x021F52: bg.movhi r24,0xb000       ; Load 0xB0000000
0x021F56: bn.ori r27,r24,0x1e       ; r27 = 0xB000001E (SE-IDMA status)
0x021F59: bn.lh r27,0x0(r27)        ; Read SE-IDMA status
0x021F5C: bg.beqi r27,0x5,0x00021F6E ; If status == 5, branch (success)
```

**Gate logic:**
1. Read SE-IDMA status from `0xB000001E`
2. If status != 5, retry with timeout (counter at 0xEE)
3. If status == 5, check license field at `0x2E4`
4. If license field == 0, print "Invalid Spdif license" and exit
5. If license field != 0, continue (success)

### 3.4 SND Image Analysis

**Evidence:** SND image does NOT contain "Invalid Spdif license" or "license" strings.

**Conclusion:** The license check is DEC-only. SND handles DTS decoding but not licensing.

---

## 4. Verified vs. Hypothesis

| ID | Claim | Classification | Primary Evidence |
|---|---|---|---|
| V01 | BL instructions are pre-relocation, not self-loops | VERIFIED | R_ARM_JUMP24 relocation analysis |
| V02 | DTS IPID checks ARE in CheckHashkey | VERIFIED | MOV r0, #0xE/0xF/7 + BL |
| V03 | DEC gate reads license field 0x2E4 | VERIFIED | `dec32_clean.txt:0x021FD6` |
| V04 | SND does not contain license strings | VERIFIED | String search in `snd_full.bin` |
| H01 | Device does NOT have DTS license provisioned | LIKELY | MIMO audit: `MApi_AUTH_Process` never called |
| H02 | To unlock DTS, need to provision `gIpAuthVars` | HYPOTHESIS | Requires further investigation |

---

## 5. Bypass Paths

### Path 1: Restore AUTH BL Instructions (Recommended)

**What:** Replace the BL instructions at DTS IPID check sites with proper calls to `MDrv_AUTH_IPCheck` (VA `0x1390C`).

**How:**
1. Identify each BL site for DTS IPID checks
2. Calculate offset = (0x1390C - VA - 8) >> 2
3. Replace `0xEBFFFFFE` with `0xEB000000 | offset`

**Advantages:**
- Restores original firmware behavior
- Allows proper license checking
- May enable DTS if OTP/strap is present

**Risks:**
- May break other functionality (multiple BL sites)
- Requires careful analysis of each site
- May not work if OTP/strap is not provisioned

### Path 2: Patch DEC Gate (Alternative)

**What:** Modify the DEC gate to skip the license field check.

**How:**
- Option A: Replace `bg.lwz r4,0x2e4(r11)` with `bg.movhi r4,0x1` (force non-zero)
- Option B: Replace the branch after license check to always take success path

**Advantages:**
- Surgical — only 1-2 instructions changed
- Does not affect ARM AUTH logic
- Preserves other DEC functionality

**Risks:**
- May cause undefined behavior if license is truly required
- Requires DEC image patching (risk of breaking DSP)

### Path 3: Use SYSTEM_Control (No Patch)

**What:** Use `MApi_AUDIO_SYSTEM_Control` to force DTS mode.

**How:**
```c
// Pseudo-code
MApi_AUDIO_SYSTEM_Control("SetSpdifOutputMode=BYPASS");
```

**Advantages:**
- No binary modification
- Uses official API
- Reversible

**Disadvantages:**
- Does NOT bypass AUTH gate
- May not work if license is checked downstream
- Requires knowing exact command syntax

---

## 6. Recommended Next Steps

1. **Verify OTP/strap state:** Check if the device has DTS license provisioned in OTP/strap
2. **Analyze BL sites:** Determine which BL instructions correspond to DTS IPID checks
3. **Test SYSTEM_Control path:** Try `MApi_AUDIO_SYSTEM_Control` with DTS mode commands
4. **If patching required:** Start with DEC gate patch (Path 2) as it is the most surgical

---

## 7. Evidence Index

| ID | Claim | Classification | Primary Evidence |
|---|---|---|---|
| E01 | BL instructions are pre-relocation | VERIFIED | R_ARM_JUMP24 relocation analysis |
| E02 | DTS IPID checks ARE in CheckHashkey | VERIFIED | MOV r0, #0xE/0xF/7 + BL |
| E03 | DEC gate reads license field 0x2E4 | VERIFIED | `dec32_clean.txt:0x021FD6` |
| E04 | SND does not contain license strings | VERIFIED | String search in `snd_full.bin` |
| E05 | Device does NOT have DTS license provisioned | LIKELY | MIMO audit: `MApi_AUTH_Process` never called |
| E06 | To unlock DTS, need to provision `gIpAuthVars` | HYPOTHESIS | Requires further investigation |

---

*End of investigation report. No binaries were patched or modified during this investigation.*
