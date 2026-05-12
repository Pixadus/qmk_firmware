# Mac Boot Selector Fix - Attempt #2

## What Changed From Previous Attempt

### Previous Approach (Didn't Work):
- Focused on **speeding up boot time**
- Reduced `CAL_SAMPL_CNT` to 2 (too aggressive)
- Tried to disable USB enumeration waits
- **Result:** Made it worse - keyboard became less recognizable

### New Approach (Current):
- Focus on **USB compatibility** not speed
- Present keyboard as **simple boot protocol device**
- Disable complex USB features that confuse boot firmware
- Use conservative calibration settings

## Root Cause Analysis

The issue is **NOT about boot speed**. It's about **USB device recognition**.

Mac's boot firmware (EFI/UEFI) runs **before macOS loads**. It only recognizes:
1. Simple HID Boot Protocol keyboards
2. 6KRO mode (not NKRO)
3. Single-endpoint keyboards
4. Standard USB descriptors

The K4 HE was presenting itself as:
- **Shared endpoint** keyboard (keyboard + extras combined)
- **NKRO** mode (complex report descriptor)
- **Multi-interface** device (keyboard + joystick + RAW HID + XInput)
- **Complex USB descriptor** that boot firmware doesn't understand

## Critical Changes

### 1. Disabled USB Shared Endpoint
**File:** `rules.mk`
```makefile
# Commented out:
# OPT_DEFS += -DSHARED_EP_ENABLE -DKEYBOARD_SHARED_EP
```
**Impact:** Forces dedicated keyboard endpoint that boot firmware recognizes

### 2. Disabled NKRO
**File:** `config.h`
```c
#define FORCE_NKRO 0
```
**Impact:** Forces 6KRO mode which boot firmware understands

### 3. Increased USB Polling Interval
**File:** `config.h`
```c
#define USB_POLLING_INTERVAL_MS 10
```
**Impact:** Slower polling (10ms vs 1ms) is more compatible with older USB implementations

### 4. Conservative Calibration
**File:** `config.h`
```c
#define CAL_SAMPL_CNT 5  // Was 2, now more conservative
```
**Impact:** Ensures stable keyboard operation during early boot

## Expected Behavior

### Before These Changes:
- Mac boot firmware doesn't recognize keyboard at all
- Option key never triggers boot selector
- Keyboard only works after macOS loads its drivers

### After These Changes:
- Mac boot firmware sees a simple HID boot keyboard
- Option key should be recognized during EFI/UEFI boot phase
- Keyboard works both in boot firmware AND in macOS

## Testing Instructions

1. **Flash firmware:**
   ```bash
   make keychron/k4_he/ansi:keychron:flash
   ```

2. **Reset Mac NVRAM** (important!):
   - Shut down Mac
   - Power on holding `Command + Option + P + R`
   - Hold until startup sound twice or Apple logo twice
   - This clears USB device cache

3. **Test boot selector:**
   - Shut down Mac
   - Connect K4 HE via USB (Cable mode)
   - Hold left Option key
   - Power on Mac (keep holding Option)
   - Boot menu should appear

4. **If still not working:**
   - Try USB 2.0 ports (not USB 3.0/Thunderbolt)
   - Try a different USB cable
   - Try connecting before powering on Mac
   - Hold Option key for 10+ seconds
   - Try powered USB hub

## What to Check If It Still Doesn't Work

### Verify USB Descriptor:
On Mac, after flashing:
```bash
system_profiler SPUSBDataType | grep -A 20 "K4 HE"
```

Look for:
- `bInterfaceClass: 3` (HID)
- `bInterfaceSubClass: 1` (Boot Interface)
- `bInterfaceProtocol: 1` (Keyboard)
- `bMaxPacketSize0: 64`

### Alternative: Use Apple's Native Keyboard
If this still doesn't work, the issue may be:
1. Mac's security settings (T2 chip restrictions)
2. FileVault enabled (only allows blessed keyboards)
3. USB port power issues
4. Cable quality issues

For Macs with T2 chips, you may need to add the keyboard to "allowed boot devices" in Startup Security Utility.

## Reverting Changes

If you want to restore NKRO and shared endpoint for normal use:

**File:** `rules.mk`
```makefile
OPT_DEFS += -DSHARED_EP_ENABLE -DKEYBOARD_SHARED_EP  # Uncomment
```

**File:** `config.h`
```c
#define FORCE_NKRO 1  # Change 0 to 1
#define USB_POLLING_INTERVAL_MS 1  # Change 10 to 1
```

## Files Modified

1. `keyboards/keychron/k4_he/config.h`
   - Added USB compatibility settings
   - Set FORCE_NKRO to 0
   - Increased USB_POLLING_INTERVAL_MS to 10
   - Set CAL_SAMPL_CNT to 5 (was 2)

2. `keyboards/keychron/k4_he/rules.mk`
   - Disabled SHARED_EP_ENABLE and KEYBOARD_SHARED_EP

3. `keyboards/keychron/k4_he/BOOT_OPTIMIZATION.md`
   - Updated with new approach and explanation

4. `keyboards/keychron/k4_he/MAC_BOOT_SELECTOR_FIX_V2.md`
   - This file (summary of changes)
