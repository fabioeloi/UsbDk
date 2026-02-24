# ARM64 Porting Notes — UsbDk

**Date:** 2026-02-24  
**Reference issue:** daynix/UsbDk#92  
**Audit document:** ARM64_AUDIT.md

---

## Overview

This document records every architectural change made to port UsbDk to
Windows 11 ARM64 (pure ARM64 kernel-mode driver) and provides rationale,
known limitations, and validation steps.

---

## Changes Made

### 1. Source Code

#### `UsbDk/UsbDkCompat.cpp` (audit finding #10)

**Change:** The XP-era Control Flow Guard stub functions
(`__guard_check_icall_fptr`, `__guard_dispatch_icall_fptr`) were wrapped in
`#ifdef _WIN64`. Since `_WIN64` is also defined when compiling for ARM64, the
64-bit stubs would compile but are logically dead code (ARM64 never targets
Windows XP). The guard was tightened to:

```cpp
#if defined(_WIN64) && !defined(_M_ARM64)
```

**Rationale:** ARM64 requires Windows 10 1607 or later; XP compatibility
stubs are never needed and including them for ARM64 could cause linker issues.

#### `UsbDkController/UsbDkController.cpp` (audit finding #11)

**Change:** Removed `__cdecl` from the `_tmain` declaration.

```cpp
// Before
int __cdecl _tmain(int argc, TCHAR* argv[])

// After
int _tmain(int argc, TCHAR* argv[])
```

**Rationale:** ARM64 MSVC does not support explicit calling-convention
decorators on entry-point functions. The decorator is silently ignored but
generates a warning and is incorrect for ARM64 kernel-mode code paths.

### 2. INF File — `UsbDkHelper/UsbDk.inf` (audit finding #14)

**Change:** Added `[UsbDk.NTarm64.Wdf]` section specifying KMDF 1.33, which
is the minimum KMDF version shipped with the Windows 11 WDK (10.0.22621+) and
required for ARM64 kernel-mode drivers.

**Rationale:** The existing `[UsbDk.NT.Wdf]` section only covers x86/x64 via
`KmdfLibraryVersion = 1.11`. ARM64 targets require KMDF ≥ 1.25 (RS1) and the
Windows 11 WDK ships 1.33. Without an `NTarm64` WDF section the driver
co-installer will fail on ARM64.

### 3. Build System — vcxproj files (audit findings #1–#8)

**Change:** Added `Win10 Debug|ARM64`, `Win10 Debug_NoSign|ARM64`, and
`Win10 Release|ARM64` platform configurations to all five project files:

- `UsbDk/UsbDk.vcxproj` (kernel driver)
- `UsbDk Package/UsbDk Package.vcxproj` (driver package)
- `UsbDkHelper/UsbDkHelper.vcxproj` (user-mode DLL)
- `UsbDkController/UsbDkController.vcxproj` (user-mode tool)
- `UsbDkInstHelper/UsbDkInstHelper.vcxproj` (installer helper)

Each ARM64 configuration block:
- Uses `<PlatformToolset>WindowsKernelModeDriver10.0</PlatformToolset>` (driver)
  or `WindowsApplicationForDrivers10.0` (user-mode).
- For user-mode projects (`UsbDkHelper`, `UsbDkController`, `UsbDkInstHelper`),
  sets `<WindowsSDKDesktopARM64Support>true</WindowsSDKDesktopARM64Support>`;
  this property is **not** set for the kernel driver project (`UsbDk.vcxproj`),
  because it is only intended for Desktop ARM64 user-mode apps and causes
  include path issues for kernel drivers.
- Defines `_WIN64;_ARM64_;ARM64` preprocessor symbols (replacing `_AMD64_`).
- Omits `<CallingConvention>Cdecl</CallingConvention>` for user-mode
  projects since ARM64 MSVC ignores this property.
- Sets `KMDF_VERSION_MINOR=33` for Win10 ARM64 targets.

**Rationale:** Only Win10 ARM64 configurations are added because ARM64
Windows never ran on Windows 7/8/8.1/XP. Adding non-Win10 ARM64 targets
would be invalid.

### 4. Build System — `UsbDk.sln` (audit finding #9)

**Change:** Added `Win10 Debug|ARM64`, `Win10 Debug_NoSign|ARM64`, and
`Win10 Release|ARM64` to both `GlobalSection(SolutionConfigurationPlatforms)`
and `GlobalSection(ProjectConfigurationPlatforms)` for all five projects.

### 5. Build Script — `buildAll.bat` (audit findings #16–#17)

**Change:**
- Added `arm64` to the platform loop but guarded it so ARM64 is only built
  for Win10 configurations (not Win7/Win8/XP which don't support ARM64).
- Added `call :make1tmf arm64\Win10%1` to the `maketmf` function.

### 6. Installer (audit findings #18–#20)

#### `Tools/Installer/UsbDkInstaller.wxs`

Added a `<?ifdef UsbDkArm64 ?>` branch (checked first) that sets:
- `UsbDkPlatform=arm64`
- `UsbDkProgramFilesFolder=ProgramFiles64Folder`
- `UsbDkWin64=yes`
- `System32Dir="System64Folder"`
- `SystemWOW64Dir="SystemFolder"`

#### `Tools/Installer/UsbDkFiles.wxi`

Parameterized all `ProcessorArchitecture="x64"` attributes to use
`ProcessorArchitecture="$(var.UsbDkArch)"` where `UsbDkArch` defaults to
`"x64"` if not defined by the caller.

#### `Tools/Installer/buildmsi.bat`

Added ARM64 MSI build sections that:
- Push into `Install_Debug\ARM64` / `Install\ARM64`
- Call `candle.exe` with `-dUsbDkArm64=1 -dUsbDkArch=arm64`
- Produce `UsbDk_<version>_arm64.msi` / `UsbDk_Debug_<version>_arm64.msi`

### 7. CI — `.github/workflows/build-arm64.yml` (audit finding #21)

Added a GitHub Actions workflow `build-arm64` that:
- Runs on `windows-latest`
- Installs WDK 10.0.22621+ via `winget`
- Builds `Win10 Release|ARM64` and `Win10 Debug_NoSign|ARM64`
- Uploads `.sys`, `.inf`, `.cat` artifacts

---

## Architecture-Specific Notes

### Kernel-Mode Driver (`UsbDk.sys`)

- **Platform toolset:** `WindowsKernelModeDriver10.0` — unchanged from x64.
  WDK 10.0.22621+ supports ARM64 with this toolset.
- **Pure ARM64:** The ARM64 configurations do **not** use `ARM64EC`.
  ARM64EC is a user-mode ABI for interoperability and is not valid for
  kernel-mode drivers.
- **Calling conventions:** ARM64 uses a single uniform calling convention.
  `__cdecl`/`__stdcall` are ignored by the compiler in ARM64 mode.
- **CFG stubs:** The `__guard_check_icall_fptr` / `__guard_dispatch_icall_fptr`
  XP-compatibility stubs are excluded from ARM64 builds (ARM64 requires
  Windows 10+, so XP stubs are never needed).
- **Interlocked operations:** `InterlockedIncrement`, `InterlockedDecrement`,
  `InterlockedIncrement64` (used in `UsbDkUtil.h` and `Alloc.h`) are defined
  as intrinsics in `<wdm.h>` / `<ntddk.h>` for ARM64 and expand to native
  `LDREX`/`STLEX` sequences. **No source change was required.**
- **KMDF version:** 1.33 is specified for ARM64 builds (vs. 1.11 for older
  x86/x64 builds). KMDF 1.33 ships with WDK 10.0.22621+.

### User-Mode Components

- **Calling conventions:** `<CallingConvention>Cdecl</CallingConvention>` is
  omitted from ARM64 `ItemDefinitionGroup` blocks in user-mode project files.
  The existing x86/x64 blocks are preserved unchanged.
- **WOW64 shim:** The existing installer already ships an x86 DLL as
  `UsbDkHelper_x86.dll` alongside the x64 native build. For ARM64, an
  ARM64-native `UsbDkHelper.dll` is produced by the ARM64 build. If support
  for 32-bit processes on ARM64 is needed, the x86 build can still be used
  as a WOW64 shim.

---

## Known Deferred Items / Limitations

1. **`UsbDkHelper/UsbDk_XP.inf`** — No ARM64 section added. Windows XP never
   ran on ARM64. This INF is intentionally x86/x64 only.

2. **`Tools/Driver.Initial.props`** — The `Windows10` target version mapping
   (`0x0A00`) is reused for ARM64 builds; a separate `Windows11` mapping was
   not added since `0x0A00` is sufficient for ARM64 on Windows 10/11.

3. **`.cat` signing submission** — The `.cat` file must declare `NTarm64` in
   its OS attribute list for WHQL/production deployment. This is handled
   automatically by `inf2cat` when the ARM64 configurations are built with
   a WDK-aware MSBuild toolchain.

4. **AppVeyor CI** — AppVeyor does not currently offer ARM64 build images.
   ARM64 CI is provided via GitHub Actions only (`.github/workflows/build-arm64.yml`).

---

## Validation Steps

1. Open `UsbDk.sln` in Visual Studio 2022 with WDK 10.0.22621+ installed.
2. Select `Win10 Release|ARM64` configuration.
3. Build → verify zero errors.
4. Copy build output to a Windows 11 ARM64 device with test-signing enabled:
   ```cmd
   bcdedit /set testsigning on
   ```
5. Install:
   ```cmd
   UsbDkController.exe -i
   ```
6. Verify driver loads:
   ```cmd
   UsbDkController.exe -s
   sc query UsbDk
   ```
7. Run USB device enumeration and verify pass-through access.

---

## References

- [daynix/UsbDk#92](https://github.com/daynix/UsbDk/issues/92) — upstream tracking issue
- [ARM64_AUDIT.md](ARM64_AUDIT.md) — full audit report with findings table
- [WDK Download](https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk)
- [KMDF Version History](https://learn.microsoft.com/windows-hardware/drivers/wdf/kmdf-version-history)
