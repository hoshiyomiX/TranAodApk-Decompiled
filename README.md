# TranAodApk — Always-On Display (AOD) Modded

> **Device:** Infinix X6873 (Hot 50 Pro+)  
> **Firmware:** X6873-15.0.3.116SP01 (Android 14 / XOS 15)  
> **Package:** `com.transsion.aod` (sharedUserId: `android.uid.system`)  
> **Target SDK:** 34  
> **Source:** [rama-firmware-dumps/infinix/Infinix-X6873](https://gitgud.io/rama-firmware-dumps/infinix/Infinix-X6873.git)

---

## Overview

This repository contains the fully decompiled source of **TranAodApk.apk** — Transsion/Infinix's Always-On Display system service — with **always-on patches** applied. The modifications remove all conditions that would automatically turn off the AOD display, making it truly "always on" regardless of battery level, power saving mode, display time mode, or screen tap timeout.

### What Was Changed?

The stock AOD behavior on Infinix X6873 is described by this string found inside the app:

> *"When in standby mode, the screen displays time, date, missed calls, and other information, and it automatically turns off in power saving mode or when the battery level is below 10%."*

Additionally, when the display time mode is set to "tap to show," the AOD only displays for **5 seconds** after tapping the screen, then auto-hides.

**This mod removes ALL of those restrictions.** After applying these patches, the AOD will stay on permanently in all conditions.

---

## Understanding the Stock AOD Timeout System

Before explaining the patches, it's important to understand how the stock AOD decides when to turn itself off. There are **four independent kill-switches**, and any one of them can destroy the AOD service:

### 1. Screen Tap Timeout (5-second auto-hide)

When the AOD is activated by tapping the screen (rather than being always-on), the system posts a `HideUIRunnable` to a handler with a **5-second delay** (`0x1388` milliseconds = 5000ms). After 5 seconds, this runnable executes and hides the AOD UI.

**Key constants in the code:**
```
WAKELOCK_HELD_TIMEOUT_FIRST = 0x1388L  // 5000ms (5 seconds) — used on first activation
WAKELOCK_HELD_TIMEOUT       = 0xbb8L   // 3000ms (3 seconds) — used on subsequent activations
SUSPEND_STATE_TIMER         = 0x1388L  // 5000ms (5 seconds) — suspend state timer
```

**How it works:**
- Inside `AlwaysOnDisplay.smali`, there are **22 different code paths** that post `mHideUIRunnable` with a delayed execution
- The `HideUIRunnable.run()` method calls `AODStateUpdater.hideAOD()` which destroys the AOD view
- The 5-second timer is hardcoded as `const-wide/16 v3, 0x1388` and passed to `postDelayed()`

### 2. Power Saving Mode

When the device enters **super power saving mode**, the AOD reads the `Settings.Secure` key `"super_power_saving_mode"`. If this setting is `1` (enabled), the code posts a `HideUIRunnable` with a 5-second delay, effectively killing the AOD.

**The decision flow:**
```
Read Settings.Secure("super_power_saving_mode")
    ├── Value == 0 (OFF)  → Continue showing AOD normally
    └── Value == 1 (ON)   → Post HideUIRunnable with 5s delay → AOD dies
```

Additionally, there's a `"child_mode_open"` check for flip/fold devices (AE11 model) that can also trigger the same hide behavior.

### 3. Low Battery Level (< 10%)

The method `lowBatteryLevelHide()` is called during AOD initialization and also from a battery change callback. It checks the current battery percentage:

```
if (batteryLevel <= 10) {
    if (display_time_mode == 1 || display_time_mode == 2) {
        log("low battery level = " + batteryLevel);
        onDestroy();  // ← Kills the entire AOD service
    }
}
```

This means even if you have AOD set to "always show," once your battery drops to 10% or below, the AOD service is completely destroyed and won't show again until the next screen-on event.

### 4. Time-Based Display Schedule

The `aod_display_time_mode` preference has three values:
- **`0`** = Always show (mAlwaysShow = true)
- **`1`** = Scheduled show (mAlwaysShow depends on current time vs. begin/end time)
- **`2`** = Tap to show (mAlwaysShow = false, 5-second timeout)

The `checkTiming()` method compares the current time against user-configured begin/end times (default: 6:00–22:00). If the current time is outside the scheduled range, it sets `mAlwaysShow = false`, which prevents the AOD from staying visible.

---

## Patches Applied

All patches are applied at the **smali level** (Dalvik bytecode) since this is a pre-compiled APK. Each patch includes a `# PATCH:` comment in the source for easy identification.

### Patch 1: Disable Battery Level Check

**File:** `smali/com/transsion/aod/dreams/AlwaysOnDisplay.smali`  
**Method:** `lowBatteryLevelHide()V` (line ~6352)

**What was changed:**
```
.method private lowBatteryLevelHide()V
    .locals 4

    # PATCH: Always-on — skip battery <= 10% auto-destroy
    return-void          ← Added this line

    # ... (original battery check code follows, now unreachable)
```

**Effect:** The method returns immediately without checking the battery level. The AOD will no longer destroy itself when battery drops to 10% or below.

**Why this is safe:** The dead code after `return-void` is perfectly valid smali — unreachable instructions don't cause crashes, they just consume a tiny amount of dex storage space.

---

### Patch 2: Skip Power Saving Mode Check

**File:** `smali/com/transsion/aod/dreams/AlwaysOnDisplay.smali`  
**Location:** Inside the large initialization method (line ~9751)

**What was changed:**

Original code:
```smali
const-string/jumbo v6, "super_power_saving_mode"
invoke-static {v5, v6, v2}, Landroid/provider/Settings$Secure;->getInt(...)I
move-result v5
if-ne v1, v5, :cond_17     # Compare setting value
goto :goto_8
:cond_17
move v1, v2                # v1 = 1 (power saving ON)
:goto_8
if-nez v3, :cond_18        # If child_mode, skip to show AOD
if-eqz v1, :cond_1a        # If power saving OFF, skip to show AOD
```

Patched code:
```smali
# PATCH: Always-on — force skip power saving mode check
const/4 v5, 0x0            # v5 = 0 (fake "power saving OFF")
move v1, v5                # v1 = 0

if-nez v3, :cond_18        # Jump to show AOD (unconditional)
if-eqz v1, :cond_1a        # v1 is 0, so this always jumps to show AOD
```

**Effect:** By setting `v1 = 0` (pretending power saving is always OFF), the conditional branch that would post the `HideUIRunnable` is never reached. The AOD continues to display normally even in super power saving mode.

**Why this approach:** Instead of removing the code entirely (which could break register allocation in the method), we simply force the comparison variable to `0`, making all subsequent branches take the "keep AOD alive" path.

---

### Patch 3: Disable Time-Based Scheduling

**File:** `smali/com/transsion/aod/dreams/AlwaysOnDisplay.smali`  
**Method:** `checkTiming()V` (line ~4085)

**What was changed:**
```
.method private checkTiming()V
    .locals 13

    # PATCH: Always-on — skip time-based scheduling (always show regardless of time)
    return-void          ← Added this line

    # ... (original time comparison code follows, now unreachable)
```

**Effect:** The `checkTiming()` method is called during AOD initialization and can override `mAlwaysShow` to `false` if the current time is outside the user's scheduled display window. By returning immediately, this override never happens.

**What this means:** Even if the user has configured a time window like 6:00–22:00, the AOD will ignore it and stay on 24/7. The `mAlwaysShow` flag will only be controlled by our Patch 4.

---

### Patch 4: Force `mAlwaysShow = true`

**File:** `smali/com/transsion/aod/dreams/AlwaysOnDisplay.smali`  
**Location:** Inside the large initialization method (line ~10711)

**What was changed:**

Original code:
```smali
:cond_6
const-string v3, "aod_display_time_mode"
invoke-direct {p0, v3}, ...->getIntSharePreferencesType(...)I  # Read preference
move-result v3
if-eqz v3, :cond_7         # if mode == 0 (always), goto cond_7
move v3, v2                # v3 = 0 (false)
goto :goto_1
:cond_7
move v3, v1                # v3 = 1 (true)
:goto_1
iput-boolean v3, p0, ...->mAlwaysShow:Z    # Set mAlwaysShow
```

Patched code:
```smali
# PATCH: Always-on — force mAlwaysShow = true regardless of aod_display_time_mode
:cond_6
const/4 v3, 0x1            # v3 = 1 (true) — hardcoded, preference ignored

:goto_1
iput-boolean v3, p0, ...->mAlwaysShow:Z    # mAlwaysShow = true
```

**Effect:** Regardless of what `aod_display_time_mode` is set to (always, scheduled, or tap-to-show), the `mAlwaysShow` field is always set to `true`. This is the master flag that controls whether the AOD persists or auto-hides.

**Why this is the most important patch:** Even if all other patches somehow fail, this one ensures the AOD's core behavior flag is locked to "always show." It's the root cause fix.

---

### Patch 5: Neutralize the HideUIRunnable

**File:** `smali/com/transsion/aod/dreams/AlwaysOnDisplay$HideUIRunnable.smali`  
**Method:** `run()V` (line ~47)

**What was changed:**
```
.method public run()V
    .locals 3

    # PATCH: Always-on — neutralize HideUIRunnable (no more 5s auto-hide)
    return-void          ← Added this line

    # ... (original hide logic follows, now unreachable)
```

**Effect:** This is the **nuclear option** patch. There are 22 different code paths throughout the AOD service that post `mHideUIRunnable` with various delays. Instead of patching all 22 call sites individually, we patch the runnable itself — making its `run()` method do nothing. No matter when or where it's triggered, it will silently do nothing.

**What the original code did:**
1. Check if AOD is in a special state (pocket mode, flip device second screen)
2. Call `AODStateUpdater.hideAOD()` which destroys the AOD view
3. Post a `SuspendStateRunnable` to handle the post-hide state

**All of this is now bypassed.**

---

### Bonus: Updated Description String

**File:** `res/values/strings.xml` (line 503)

**Original:**
```xml
<string name="switch_text_info">When in standby mode, the screen displays time, date,
missed calls, and other information, and it automatically turns off in power saving
mode or when the battery level is below 10%.</string>
```

**Patched:**
```xml
<string name="switch_text_info">When in standby mode, the screen always displays time,
date, missed calls, and other information. Always-on mode has been enabled.</string>
```

This is purely cosmetic — it updates the AOD toggle description in the device settings to reflect the new always-on behavior.

---

## How to Rebuild the Modded APK

If you want to build a flashable APK from these modified sources, follow these steps:

### Prerequisites

1. **Java JDK 8+** (for apktool)
2. **apktool v2.10.0** or later
3. **The 4 framework APKs** installed in your apktool framework directory:
   - `framework-res.apk` (AOSP base)
   - `mediatek-res.apk` (MediaTek extensions)
   - `transsion-res.apk` (Transsion resource overlay)
   - `thub-res.apk` (Transsion THUB resource overlay)

### Step 1: Install Frameworks

```bash
# Install each framework (order matters)
apktool if framework-res.apk
apktool if mediatek-res.apk
apktool if transsion-res.apk
apktool if thub-res.apk
```

These frameworks are required because TranAodApk references resources from all four layers. Without them, apktool cannot properly decode/re-encode the resources.

### Step 2: Build the APK

```bash
cd /path/to/TranAodApk_decoded
apktool b . -o TranAodApk_AlwaysOn.apk
```

### Step 3: Sign and Flash

```bash
# Sign with your test key or a custom key
apksigner sign --ks your.keystore --ks-key-alias your_alias TranAodApk_AlwaysOn.apk

# Push to device (requires root)
adb push TranAodApk_AlwaysOn.apk /system_ext/priv-app/TranAodApk/TranAodApk.apk
adb reboot
```

### Important Notes

- **System app:** TranAodApk is a privileged system app (`system_ext/priv-app/`) running with `android.uid.system` UID. It must be installed in the system partition and requires root access to replace.
- **Framework dependency:** The framework APKs must match your device's firmware version exactly. Using frameworks from a different firmware version will cause resource mismatches.
- **Testing:** After flashing, enable AOD in Settings → Lock Screen → Always On Display and verify it stays on during:
  - Power saving mode
  - Battery below 10%
  - All times of day
  - After tapping the screen

---

## File Structure

```
TranAodApk_decoded/
├── AndroidManifest.xml          # App manifest (package, permissions, activities)
├── apktool.yml                  # apktool metadata (framework IDs, package info)
├── assets/                      # Raw assets (fonts, 3D models, shader configs)
├── lib/                         # Native libraries (ARM64, ARMv7)
│   ├── arm64-v8a/
│   └── armeabi-v7a/
├── original/                    # Original META-INF and unmodified files
├── res/                         # Resources
│   ├── drawable/                # Icons, backgrounds, shapes
│   ├── layout/                  # 581 XML layout files
│   ├── raw/                     # 266 MP4 animation files
│   ├── values/                  # Strings, colors, integers, bools
│   │   ├── strings.xml          # ← Modified: switch_text_info
│   │   └── integers.xml         # Duration configs for animations
│   └── xml/                     # Preference screens, searchable configs
├── smali/                       # Main DEX bytecode (com.transsion.aod)
│   └── com/transsion/aod/
│       ├── dreams/
│       │   ├── AlwaysOnDisplay.smali             # ← Modified (Patches 1,2,3,4)
│       │   ├── AlwaysOnDisplay$HideUIRunnable.smali  # ← Modified (Patch 5)
│       │   ├── DreamService.smali               # Base AOD dream service
│       │   └── AODStateUpdater.smali            # AOD state machine
│       ├── util/
│       │   ├── BatteryUtil.smali                # Battery monitoring
│       │   ├── Utils.smali                      # Utility functions
│       │   └── AodPocketManager.smali           # Pocket/proximity detection
│       ├── fragment/
│       ├── preference/
│       ├── service/
│       ├── widget/
│       └── view/
├── smali_classes2/              # Secondary DEX (support libraries, SDKs)
│   ├── com/transsion/hubsdk/    # Transsion Hub SDK
│   ├── com/transsion/surfaceengine/  # 3D rendering (cute pet, etc.)
│   ├── com/google/android/filament/   # Google Filament 3D engine
│   └── androidx/                # AndroidX support libraries
└── smali_classes3/              # Tertiary DEX
```

---

## Technical Deep Dive: How the AOD Service Works

### Architecture

The AOD system is implemented as an Android **DreamService** (screensaver). When the device screen turns off, the system launches `AlwaysOnDisplay` as a dream service, which takes over the screen with a dimmed always-on display.

```
┌─────────────────────────────────────────────────┐
│              Android DreamService                │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │        AlwaysOnDisplay (Main Controller)    │  │
│  │                                             │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │ DreamSvc │  │ AODState │  │ Pocket   │  │  │
│  │  │ (base)   │  │ Updater  │  │ Manager  │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  │  │
│  │                                             │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │ HideUI   │  │ Suspend  │  │ Battery  │  │  │
│  │  │ Runnable │  │ Runnable │  │ Util     │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  │  │
│  └────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### AOD Lifecycle

1. **System triggers AOD** when the screen turns off (Doze mode)
2. `AlwaysOnDisplay.onCreate()` initializes all subsystems:
   - Reads `aod_display_time_mode` preference
   - Reads `aod_is_show` preference
   - Sets up battery monitoring, pocket detection, light sensor
   - Calls `checkTiming()` to determine if current time is in schedule
   - Calls `lowBatteryLevelHide()` to check battery level
3. `onAttachedToWindow()` sets up the UI:
   - Loads the clock style, wallpaper background, widgets
   - Configures screen brightness to minimum
   - Calls `upateAodStartTime()` to record when AOD started
4. **During display:**
   - `HideUIRunnable` may be posted with 5s delay (for tap-to-show mode)
   - Battery change broadcasts can trigger `lowBatteryLevelHide()`
   - Pocket/proximity sensor can temporarily hide AOD
   - Power saving mode changes trigger hide via system settings observer
5. **AOD destruction:**
   - `onDestroy()` cleans up all handlers, receivers, and sensors
   - `AODStateUpdater.exitAOD()` or `hideAOD()` transitions the state

### Key SharedPreferences

| Key | Type | Values | Purpose |
|-----|------|--------|---------|
| `aod_display_time_mode` | int | 0=Always, 1=Scheduled, 2=Tap-to-show | Controls display mode |
| `aod_is_show` | int | 1=ON, 0=OFF | Master AOD toggle |
| `display_mode_begin_time` | string | e.g. "6:00" | Scheduled mode start time |
| `display_mode_end_time` | string | e.g. "22:00" | Scheduled mode end time |
| `pocket_mode_always_show` | int | 0/1/2 | Pocket detection behavior |
| `random_select` | int | 0/1 | Random style selection |

---

## Reverting the Patches

If you want to restore the original behavior, you can do so by:

1. **Remove the `# PATCH:` comments and `return-void` lines** from:
   - `AlwaysOnDisplay.smali` — `lowBatteryLevelHide()` and `checkTiming()` methods
   - `AlwaysOnDisplay$HideUIRunnable.smali` — `run()` method

2. **Restore the original code** in `AlwaysOnDisplay.smali`:
   - Power saving section: restore the `Settings.Secure.getInt("super_power_saving_mode")` call
   - `mAlwaysShow` section: restore the `getIntSharePreferencesType("aod_display_time_mode")` read

3. **Restore the original string** in `res/values/strings.xml`:
   ```xml
   <string name="switch_text_info">When in standby mode, the screen displays time,
   date, missed calls, and other information, and it automatically turns off in
   power saving mode or when the battery level is below 10%.</string>
   ```

All original code is preserved after the `return-void` / patch lines, making it easy to revert.

---

## Device Compatibility

This mod was developed specifically for the **Infinix Hot 50 Pro+ (X6873)** running firmware version **X6873-15.0.3.116SP01** with Android 14 / XOS 15.

It **may work** on other Transsion/Infinix/Tecno/Itel devices running similar XOS versions, provided:
- The device uses the same `com.transsion.aod` package
- The firmware includes the same `TranAodApk.apk` with similar smali structure
- The framework APKs match the target firmware

However, **do NOT flash this on a different device** without first decompiling and verifying the smali structure matches. Different firmware versions may have different obfuscation, different class names, or additional logic that could cause crashes.

---

## Disclaimer

This modification is provided for educational and personal use only. Modifying system APKs may void your warranty, cause device instability, or create security vulnerabilities. Always back up your device before flashing modified system files. The authors assume no liability for any damage caused by using these modifications.
