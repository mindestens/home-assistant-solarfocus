# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Fixed

- Restored clean Solarfocus device info display for API 25.050 by returning `model` and `sw_version` as plain strings (no set formatting like `{'Vampair'}`).
- Fixed Solarfocus options flow initialization for newer Home Assistant versions by aligning `OptionsFlow` construction with current API behavior.
- Fixed German translation placeholder parity for `component.solarfocus.entity.sensor.bb_status.state.21` by keeping `{RGT_Start}` in `de.json`.

## [6.0.1] - 2026-04-18

### Changed

- Updated pysolarfocus dependency to v5.1.6 (Python 3.14 compatibility)

---

## Type of Change

- [x] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature
- [ ] Breaking change

## Impact

- Fixes integration startup failure on Home Assistant installations running Python 3.14
- No functional changes to the integration itself
- No changes required for existing configurations

---

## Summary

This PR updates the `pysolarfocus` dependency from `5.1.5` to `5.1.6` to restore compatibility with Home Assistant instances running Python 3.14.

Integration version bumped from `6.0.0` → `6.0.1`.

---

## Problem

Recent Home Assistant container updates ship with Python 3.14. All `pysolarfocus` releases up to and including `5.1.5` declare `Requires-Python: ~=3.13.0`, which means pip refuses to install them on Python 3.14:

```
ERROR: Package 'pysolarfocus' requires a different Python: 3.14.x not in '~=3.13.0'
```


## [6.0.0]

### Added

- **Support for Solarfocus API version v25.050** (now default for new installations)
- Added v25.050 to available API version selections in config flow

### Changed

- Default API version for new installations: `25.050` (previous: `23.020`)
- Updated pysolarfocus dependency to v5.1.5
- Compressor speed unit changed from "rpm" to "U/min" (German-speaking target audience)
- Improved German translations consistency:
  - Fixed capitalization: "Aussen" → "Außen", "warte" → "Warte", "keine" → "Keine", "zweiter" → "Zweiter", "kein" → "Kein"
  - All sentence beginnings now consistently capitalized per German orthography

### Fixed

- **Critical Bug Fix**: Fixed incorrect heating circuit register offsets in pysolarfocus library
  - Affected API versions: < v25.030
  - Wrong offsets: 4, 5, 6 → Correct offsets: 5, 6, 7
  - Parameters: Target Supply Temperature, Target Room Temperature, Indoor Temperature External
  - Impact: Users were reading wrong temperature values from incorrect Modbus registers
  - **Resolves ValueError**: `Sensor sensor.solarfocus_heating_circuit_1_state provides state value 'xx', which is not in the list of options provided`
    - Root cause: Wrong register offsets caused state sensor to read from incorrect register, returning invalid enum values (e.g., 50)
    - Fix: Correct offsets now ensure state sensor reads valid values (0-36, 200-228)
- **Extended heating circuit state enum values**: Added missing values 32-36 to heating circuit state sensor
  - 32: Indoor target temperature cooling reached
  - 33: No time approval for cooling mode
  - 34: Supply temperature limit due to heat pump error
  - 35: Waiting for cooling approval
  - 36: Waiting for cooling approval, heating circuits are active
- Fixed entity unique_id generation by adding public `config_entry` property to coordinator
- Fixed Options Flow Handler initialization (restored `self.config_entry` assignment)

## Description

This PR addresses two critical issues and adds support for the latest Solarfocus API version:

### 1. Critical Bug Fix: Heating Circuit Register Offsets

**Problem:**
The pysolarfocus library was using incorrect Modbus register offsets for heating circuit sensors, causing multiple issues:
- Wrong temperature values being read from the system
- `ValueError: Sensor sensor.solarfocus_heating_circuit_1_state provides state value '50', which is not in the list of options provided`
- Invalid enum values due to reading wrong registers

**Root Cause Analysis:**

The ValueError occurred because of a **cascade effect** from misaligned register offsets. Here's what happened:

**Heating Circuit 1 Register Map (Base Address: 1100):**

| Register | Parameter | Expected Offset | Wrong Offset Used | Result |
|----------|-----------|-----------------|-------------------|---------|
| 1100 | Supply Temperature | 0 | 0 | ✅ Correct |
| 1101 | Room Temperature | 1 | 1 | ✅ Correct |
| 1102 | Humidity | 2 | 2 | ✅ Correct |
| 1103 | Limit Thermostat | 3 | 3 | ✅ Correct |
| 1104 | *(Reserved by Solarfocus - skipped!)* | - | - | ❌ Not documented |
| 1105 | Heating Circuit Pump | 4 | **4** | ✅ Correct |
| 1106 | Mixer Position | 5 | **4** | ❌ Read pump instead! |
| 1107 | **Heating Circuit State** | 6 | **5** | ❌ Read mixer position! |

**The Chain Reaction:**
1. Solarfocus **skipped register 1104** in their documentation (reserved/unused)
2. pysolarfocus assumed **consecutive offsets** without gaps: 0, 1, 2, 3, 4, 5, 6
3. Reality with gap: 0, 1, 2, 3, *(gap)*, 4, 5, 6, **7** ← State is actually here!
4. When reading "Target Supply Temperature" (should be offset 5), code used offset 4 → read wrong register
5. When reading "State" (should be offset 6/7), code read **mixer position value** instead
6. Mixer position returned value like `50` (meaning 50% valve position)
7. Home Assistant expected state enum (0-36, 200-228), got `50` → **ValueError!**

**Additional Issue:**
Even with correct offsets, the enum definition was incomplete:
- **Before:** `range(32)` = values 0-31 only
- **After:** `range(37)` = values 0-36 (added missing cooling states 32-36)

**Concrete Example:**
```python
# BEFORE (Wrong):
TargetSupplyTemperature: offset 4  # Actually reading pump state (0/1)
TargetRoomTemperature: offset 5     # Actually reading mixer position (0-100%)
State: offset 6                     # Actually reading undefined/random value

# AFTER (Correct):
TargetSupplyTemperature: offset 5  # Reads correct temperature register
TargetRoomTemperature: offset 6     # Reads correct temperature register  
State: offset 7                     # Reads correct state register (0-36, 200-228)
```

**Root Cause:**
According to the Solarfocus Modbus TCP documentation (ecomanager-touch_Modbus-TCP_Registerdaten_Anleitung.pdf), register 32604 is **not used/documented** between 32603 and 32605, creating a gap that pysolarfocus didn't account for.

| Parameter | Wrong Offset | Correct Offset | Impact |
|-----------|--------------|----------------|---------|
| Target Supply Temperature | 4 | 5 | Reading wrong register data |
| Target Room Temperature | 5 | 6 | Reading wrong register data |
| Indoor Temperature External | 6 | 7 | Reading wrong register data |

**Solution:**
Updated pysolarfocus to v5.1.5 with corrected register offsets matching official documentation.

### Technical Details

#### Why v25.050 as Default?

Solarfocus has released newer firmware versions (v25.xxx series) with improved Modbus implementations. Setting v25.050 as default ensures:
- New users get the most recent and stable API version
- Better compatibility with current Solarfocus firmware
- Reduced need for manual configuration during setup
- Existing users on older API versions (23.020, 23.040, etc.) are not affected

#### Root Cause Analysis

The register offset bug was documented in the official Solarfocus Modbus TCP documentation (ecomanager-touch_Modbus-TCP_Registerdaten_Anleitung.pdf). 

**Problem:** Solarfocus documentation shows register 32604 is reserved/skipped, but pysolarfocus assumed consecutive offsets, causing all subsequent registers to be offset by -1.

**Example:**
```python
# Before (incorrect)
TargetSupplyTemperature: InputRegister(4, RegisterDataType.FLOAT_SIGNED, 1, 10)
# After (correct) 
TargetSupplyTemperature: InputRegister(5, RegisterDataType.FLOAT_SIGNED, 1, 10)
```

The fix is implemented in pysolarfocus v5.1.5 and affects all users with API versions before v25.030.

#### Testing

Tested with:
- Home Assistant 2025.1.x
- Solarfocus Vampair heat pump  
- API version v25.050
- Configuration: 2 heating circuits, 1 buffer, 1 boiler, 1 heat pump
- Docker test environment

All entities created successfully with:
- ✅ Correct temperature readings from proper Modbus registers
- ✅ v25.050 API working correctly
- ✅ No breaking changes to existing installations
- ✅ Proper unique_id generation
- ✅ ValueError fix validated

## Previous Versions

See [GitHub Releases](https://github.com/LavermanJJ/home-assistant-solarfocus/releases) for older versions.
