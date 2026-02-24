# ARM64 Porting Notes — UsbDk

**Date:** 2025-02-24
**Reference issue:** daynix/UsbDk#92

---

## Overview

This document describes all changes made to port UsbDk from x86/x64 to
Windows 11 on ARM64. The port targets **pure ARM64** (not ARM64EC), which
is required for kernel-mode drivers.

---

## Build System Changes

### 1. vcxproj Files (Findings #1–#8)

**Files modified:**
- `UsbDk/UsbDk.vcxproj` (kernel driver)
- `UsbDk Package/UsbDk Package.vcxproj` (driver package)
- `UsbDkHelper/UsbDkHelper.vcxproj` (user-mode DLL)
- `UsbDkController/UsbDkController.vcxproj` (user-mode EXE)
- `UsbDkInstHelper/UsbDkInstHelper.vcxproj` (user-mode EXE)

**What was done:**
- Added `ARM64` `<ProjectConfiguration>` entries for all Win10 configurations
  (Debug, Release, Debug_NoSign). ARM64 is only valid for Win10+ targets.
- Duplicated all `<PropertyGroup>`, `<ItemDefinitionGroup>`, and `<ImportGroup>`
  blocks conditioned on `Win10|x64` to create matching `Win10|ARM64` variants.
- Replaced `_AMD64_=1;AMD64` preprocessor definitions with `_ARM64_=1;ARM64`
  in all ARM64-conditioned blocks.
- Added `<WindowsSDKDesktopARM64Support>true</WindowsSDKDesktopARM64Support>`
  to the `<PropertyGroup Label="Globals">` in each project.
- **User-mode projects only:** Removed `<CallingConvention>Cdecl</CallingConvention>`
  from ARM64 `<ItemDefinitionGroup>` blocks, since ARM64 uses a single uniform
  calling convention and `__cdecl` is not valid.

**Rationale:** ARM64 was never supported on Windows XP, 7, 8, or 8.1, so only
Win10 configurations receive ARM64 variants. The existing Win32 and x64
configurations are completely unchanged.

### 2. Solution File (Finding #9)

**File:** `UsbDk.sln`

- Added `Win10 Debug|ARM64`, `Win10 Release|ARM64`, and `Win10 Debug_NoSign|ARM64`
  to `SolutionConfigurationPlatforms`.
- Added project configuration mappings for all 5 projects with `ActiveCfg`,
  `Build.0`, and `Deploy.0` (deploy only for driver projects).

### 3. Driver.Initial.props (Finding #22)

**File:** `Tools/Driver.Initial.props`

- Added `<PropertyGroup Condition="'$(TargetVersion)'=='Windows11'">` mapping
  `_NT_TARGET_VERSION` to `0x0A00` (same as Windows 10, since Windows 11 shares
  the NT 10.0 version number).

### 4. Build Script (Findings #16–#17)

**File:** `buildAll.bat`

- Added a separate ARM64 build loop for Win10 configurations only (after the
  existing Win32/x64 loop for all OS versions).
- Added `call :make1tmf arm64\Win10%1` to the TMF generation function.

---

## Source Code Changes

### 5. CFG Stubs — UsbDkCompat.cpp (Finding #10)

**File:** `UsbDk/UsbDkCompat.cpp`

- Wrapped the `extern "C"` block containing `__guard_check_icall_fptr` and
  `__guard_dispatch_icall_fptr` stubs with `#if !defined(_M_ARM64)`.
- **Rationale:** These stubs exist solely for the `TARGET_OS_WIN_XP` code path.
  ARM64 never ran Windows XP, so these stubs are dead code on ARM64. Additionally,
  the ARM64 kernel has its own CFG implementation that is incompatible with these
  x86/x64-specific shims.

### 6. Calling Convention — UsbDkController.cpp (Finding #11)

**File:** `UsbDkController/UsbDkController.cpp`

- Changed `int __cdecl _tmain(...)` to use a `#if defined(_M_ARM64)` guard that
  removes the `__cdecl` decorator on ARM64 (where it is not valid).
- On x86/x64, `__cdecl` is preserved unchanged.

### 7. Interlocked Operations (Findings #12–#13) — No Change

**Files:** `UsbDk/UsbDkUtil.h`, `UsbDk/Alloc.h`

- `InterlockedIncrement`, `InterlockedDecrement`, and `InterlockedIncrement64`
  are provided by WDK headers for all architectures including ARM64.
- No source changes required; the ARM64 WDK libraries provide native
  LDREX/STLEX sequences for these intrinsics.

---

## INF File Changes

### 8. UsbDk.inf (Finding #14)

**File:** `UsbDkHelper/UsbDk.inf`

- Added `[UsbDk.NTarm64.Wdf]` section specifying `KmdfLibraryVersion = 1.33`
  (as shipped with WDK 10.0.22621+, the minimum for ARM64 KMDF support).
- The existing `[UsbDk.NT.Wdf]` section with `KmdfLibraryVersion = 1.11` is
  preserved for x86/x64 targets.

### 9. UsbDk_XP.inf (Finding #15)

**File:** `UsbDkHelper/UsbDk_XP.inf`

- Added a comment documenting that this INF is x86/x64 only and not applicable
  to ARM64 targets.
- No ARM64 section added (Windows XP never ran on ARM64).

---

## CI/CD Changes

### 10. GitHub Actions Workflow (Finding #21)

**File:** `.github/workflows/arm64-build.yml`

- Added a new workflow that builds ARM64 Win10 Release and Debug configurations.
- Uses `microsoft/setup-msbuild@v2` for MSBuild discovery.
- Uploads ARM64 release artifacts (`.sys`, `.dll`, `.exe`, `.inf`, `.cat`).
- Note: Actual deployment to ARM64 hardware requires test-signing mode
  (`bcdedit /set testsigning on`) or WHQL/EV signing for production.

---

## Known Limitations and Deferred Items

1. **Installer (Findings #18–#20):** The WiX-based MSI installer
   (`Tools/Installer/`) has not been updated for ARM64 in this port. The
   installer hardcodes `ProcessorArchitecture="x64"` and lacks ARM64 build
   paths. This is deferred to a future PR.

2. **WOW64 helper DLL:** For ARM64 systems running x86 guest processes,
   an x86-on-ARM64 WOW64 shim (`UsbDkHelper_x86.dll`) may be needed. This
   is deferred.

3. **AppVeyor CI (Finding #21):** AppVeyor does not currently offer ARM64
   Windows images. The existing `.appveyor.yml` is unchanged and continues
   to build x86/x64 only. GitHub Actions handles ARM64.

4. **KMDF version for ARM64:** The ARM64 INF specifies KMDF 1.33. This
   requires the WDK 10.0.22621+ to be installed on the build machine.

---

## Validation Steps

1. **Build verification:** Open `UsbDk.sln` in Visual Studio 2022 with
   WDK 10.0.22621+ installed. Select `Win10 Release|ARM64` and build all
   projects. All 5 projects should compile without errors.

2. **Binary verification:** Confirm the output `UsbDk.sys` is a valid
   ARM64 PE binary:
   ```
   dumpbin /headers arm64\Win10Release\UsbDk.sys | findstr machine
   ```
   Expected: `AA64 machine (ARM64)`

3. **Driver installation (test-signed):**
   ```
   bcdedit /set testsigning on
   UsbDkController.exe -i
   ```

4. **Device enumeration:**
   ```
   UsbDkController.exe -n
   ```
