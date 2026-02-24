# ARM64 Audit Report — UsbDk

**Date:** 2026-02-24  
**Scope:** Full repository audit for Windows 11 ARM64 (pure ARM64 kernel-mode driver) portability  
**Reference issue:** daynix/UsbDk#92

---

## Executive Summary

UsbDk currently targets Win32 (x86) and x64 only. No ARM64 platform configurations exist
anywhere in the solution. The driver source itself is largely portable C++/KMDF with only
two architecture-specific constructs; the main work is in the build system, installer, and INF
files. The table below lists every finding with a proposed fix.

---

## Findings Table

| # | File | Line(s) | Category | Issue | Proposed Fix |
|---|------|---------|----------|-------|--------------|
| 1 | `UsbDk\UsbDk.vcxproj` | All `<ProjectConfiguration>` blocks | Build System | No `ARM64` platform configuration exists; only `Win32` and `x64` are present | Duplicate each `x64` `<ProjectConfiguration>` block and rename `Platform` to `ARM64`. Add corresponding `<PropertyGroup>` and `<ItemDefinitionGroup>` blocks with `<PlatformTarget>ARM64</PlatformTarget>` and `<WindowsSDKDesktopARM64Support>true</WindowsSDKDesktopARM64Support>` |
| 2 | `UsbDk Package\UsbDk Package.vcxproj` | All `<ProjectConfiguration>` blocks | Build System | No `ARM64` platform configuration; only `Win32` and `x64` | Same as #1; add ARM64 variants of every `Win10 Debug\|x64`, `Win10 Release\|x64`, etc. configuration block |
| 3 | `UsbDkHelper\UsbDkHelper.vcxproj` | All `<ProjectConfiguration>` blocks | Build System | No `ARM64` platform configuration | Add ARM64 configurations mirroring the x64 ones |
| 4 | `UsbDkController\UsbDkController.vcxproj` | All `<ProjectConfiguration>` blocks | Build System | No `ARM64` platform configuration; only `x64` (no Win32) | Add ARM64 configurations mirroring the x64 ones |
| 5 | `UsbDkInstHelper\UsbDkInstHelper.vcxproj` | All `<ProjectConfiguration>` blocks | Build System | No `ARM64` platform configuration | Add ARM64 configurations mirroring the x64 ones |
| 6 | `UsbDkHelper\UsbDkHelper.vcxproj` | ~572, ~596, ~620 … (every config) | Build System | `<CallingConvention>Cdecl</CallingConvention>` — ARM64 MSVC does not support explicit calling-convention overrides; the MSBuild property must be absent or set to the default for ARM64 configurations | Omit `<CallingConvention>` from ARM64-specific `<ItemDefinitionGroup>` blocks (or wrap the existing x64 blocks in a `Condition` that excludes ARM64) |
| 7 | `UsbDkController\UsbDkController.vcxproj` | ~566, ~587, ~608 … (every config) | Build System | Same `<CallingConvention>Cdecl</CallingConvention>` issue as #6 | Same fix as #6 |
| 8 | `UsbDkInstHelper\UsbDkInstHelper.vcxproj` | ~566, ~587, ~608 … (every config) | Build System | Same `<CallingConvention>Cdecl</CallingConvention>` issue as #6 | Same fix as #6 |
| 9 | `UsbDk.sln` | `GlobalSection(SolutionConfigurationPlatforms)` and `GlobalSection(ProjectConfigurationPlatforms)` | Build System | No ARM64 entries in the solution-level configuration lists; Visual Studio and MSBuild cannot target ARM64 without them | Add `Win10 Debug\|ARM64`, `Win10 Release\|ARM64`, `Win10 Debug_NoSign\|ARM64` (and equivalents for Win8.1, Win8, Win7) to the solution configuration list, and map each project GUID to corresponding ARM64 project configurations |
| 10 | `UsbDk\UsbDkCompat.cpp` | 45–60 | Source — Architecture Guard | `#ifdef _WIN64` wraps CFG (Control Flow Guard) stub functions `__guard_check_icall_fptr` and `__guard_dispatch_icall_fptr`. `_WIN64` **is** defined on ARM64, so the stubs compile, but: (a) these XP-era stubs are only compiled when `TARGET_OS_WIN_XP` is defined, and (b) the x64 64-bit path (`NTSTATUS __guard_check_icall_fptr(...)`) does not explicitly exclude ARM64. ARM64 kernel-mode CFG uses the same symbol names, so the stubs are correct **only if** ARM64 also needs them at the XP compat level (it does not — ARM64 never ran XP). The guard should be tightened. | Change the guard to `#if defined(_WIN64) && !defined(_M_ARM64)` for the 64-bit stubs, or simply exclude the entire `extern "C"` block when `TARGET_OS_WIN_XP` is combined with ARM64 (which is an impossible target anyway) |
| 11 | `UsbDkController\UsbDkController.cpp` | 360 | Source — Calling Convention | `int __cdecl _tmain(int argc, TCHAR* argv[])` — `__cdecl` is not a valid calling-convention specifier on ARM64; MSVC emits warning C4242/C4996 and ignores it. Not a compile error but is incorrect for ARM64. | Remove `__cdecl` from the `_tmain` declaration, or guard it: `#ifndef _M_ARM64` … `#endif` |
| 12 | `UsbDk\UsbDkUtil.h` | 147, 160, 161, 619 | Source — Interlocked Operations | Uses `InterlockedIncrement64`, `InterlockedIncrement`, and `InterlockedDecrement`. These are defined as intrinsics in `<wdm.h>` / `<ntddk.h>` for all supported Windows architectures including ARM64. **No change required** for correctness, but callers should ensure they link against the ARM64 WDK libraries. | No source change required; verify ARM64 library paths in the vcxproj |
| 13 | `UsbDk\Alloc.h` | 195, 200, 205 | Source — Interlocked Operations | Same `InterlockedIncrement` / `InterlockedDecrement` usage as #12 | No source change required |
| 14 | `UsbDkHelper\UsbDk.inf` | 1–16 (entire file) | INF | The KMDF co-installer INF specifies `KmdfLibraryVersion = 1.11`. ARM64 requires WDK 10.0.22621+ which ships KMDF 1.33. There is no `NTarm64` section and no `SourceDisksFiles.arm64` section. | Add `[UsbDk.NTarm64.Wdf]` section specifying `KmdfLibraryVersion = 1.33` (or the minimum version supported by the target WDK). If the INF is also used for device install, add `[Manufacturer]` with `NTarm64` decoration, a `[Standard.NTarm64]` device list section, `[UsbDk_Install.NTarm64]`, `[UsbDk_Install.NTarm64.Services]`, and `[SourceDisksFiles.arm64]` sections |
| 15 | `UsbDkHelper\UsbDk_XP.inf` | 1–16 (entire file) | INF | Windows XP never ran on ARM64. The XP INF specifies `KmdfLibraryVersion = 1.09` which predates ARM64 KMDF support. | No ARM64 section needed; document explicitly that this INF is x86/x64 only and is not applicable to ARM64 targets |
| 16 | `buildAll.bat` | 13 (`for %%z in (win32, x64)`) | Build Script | The outer build loop only iterates over `win32` and `x64`; ARM64 is not built | Add `arm64` to the loop: `for %%z in (win32, x64, arm64) do` and guard XP/Win7 configurations that are not valid for ARM64 |
| 17 | `buildAll.bat` | `maketmf` function (~28–42) | Build Script | TMF (trace message format) extraction only processes `x64\` and `x86\` subdirectories; no `arm64\` path | Add `call :make1tmf arm64\Win10%1` etc. for ARM64 Win10 configurations |
| 18 | `Tools\Installer\UsbDkInstaller.wxs` | 4–11 (platform detection) | Installer | Only detects `x64` and falls back to `x86`; no `arm64` variant. ARM64 Windows also sets `VersionNT64` so the 32-bit guard would not trigger, but the installer would silently deploy x64 binaries. | Add a third `<?ifdef UsbDkArm64 ?>` branch setting `UsbDkPlatform=arm64` and `Platform="arm64"` in the `<Package>` element |
| 19 | `Tools\Installer\UsbDkFiles.wxi` | 6–22 (all `<File>` elements) | Installer | All `<File>` elements have `ProcessorArchitecture="x64"` hardcoded | Parameterise with `ProcessorArchitecture="$(var.UsbDkArch)"` where `UsbDkArch` is defined as `x64` or `arm64` by the build script |
| 20 | `Tools\Installer\buildmsi.bat` | 8–47 (all pushd/light.exe calls) | Installer | Only builds `x86` and `x64` MSI packages | Add an ARM64 section that pushes into `Install\arm64`, calls `candle.exe -dUsbDkArm64=1 …`, and produces `UsbDk_%UsbDkVersion%_arm64.msi` |
| 21 | `.appveyor.yml` | Entire file | CI | AppVeyor build matrix only builds x86 and x64 (`buildAll.bat` is called without an ARM64 target). No GitHub Actions workflows exist. | Either extend AppVeyor to add an ARM64 image (not currently available on AppVeyor), or add a `.github/workflows/build-arm64.yml` GitHub Actions workflow using a `windows-latest` runner with WDK 10.0.22621+ installed |
| 22 | `Tools\Driver.Initial.props` | 18–29 | Build Props | No `TargetVersion=Windows11` / `_NT_TARGET_VERSION` mapping; the highest is `Windows10` (`0x0A00`). ARM64 support is only fully present from Windows 10 version 1607, but KMDF ARM64 officially requires Windows 10 RS1+. | Add `<PropertyGroup Condition="'$(TargetVersion)'=='Windows11'"><_NT_TARGET_VERSION>0x0A00</_NT_TARGET_VERSION></PropertyGroup>` or rely on the existing `Windows10` mapping, and document that ARM64 targets must use `Win10` configurations |

---

## Category Summary

| Category | Count | Files Affected |
|----------|-------|---------------|
| Build System (vcxproj / sln) | 9 | `UsbDk.vcxproj`, `UsbDk Package.vcxproj`, `UsbDkHelper.vcxproj`, `UsbDkController.vcxproj`, `UsbDkInstHelper.vcxproj`, `UsbDk.sln` |
| Source — Architecture Guard | 1 | `UsbDkCompat.cpp` |
| Source — Calling Convention | 1 | `UsbDkController.cpp` |
| Source — Interlocked Ops (no change) | 2 | `UsbDkUtil.h`, `Alloc.h` |
| INF | 2 | `UsbDk.inf`, `UsbDk_XP.inf` |
| Build Script | 2 | `buildAll.bat` |
| Installer | 3 | `UsbDkInstaller.wxs`, `UsbDkFiles.wxi`, `buildmsi.bat` |
| CI | 1 | `.appveyor.yml` |
| Build Props | 1 | `Driver.Initial.props` |

---

## ARM64-Specific Notes

### Kernel Mode (UsbDk.sys)

- **Platform toolset:** Use `WindowsKernelModeDriver10.0` (unchanged); ARM64 is supported by this toolset when WDK 10.0.22621+ is installed.
- **Pure ARM64 vs ARM64EC:** The driver **must** target pure `ARM64`, not `ARM64EC`. ARM64EC is a user-mode ABI for interoperability; it is not valid for kernel-mode drivers.
- **Calling conventions:** ARM64 kernel mode uses a single uniform calling convention. Any `__cdecl` / `__stdcall` decoration inside kernel-mode code paths will be silently ignored by the compiler but should be removed to avoid confusion.
- **CFG stubs (`__guard_*`):** The XP-compatibility stubs in `UsbDkCompat.cpp` are only needed for pre-Windows 8 targets. ARM64 never supports those OS versions; the stubs are logically dead code for ARM64 and should be excluded.
- **Interlocked intrinsics:** WDK-provided `Interlocked*` intrinsics (`InterlockedIncrement`, `InterlockedDecrement`, `InterlockedIncrement64`) expand to native `LDREX`/`STLEX` sequences on ARM64. No source change is needed.
- **KMDF version:** ARM64 requires KMDF ≥ 1.25 (Windows 10 RS1). For Windows 11 targeting, use KMDF 1.33 as shipped with WDK 10.0.22621+.

### User Mode (UsbDkHelper.dll, UsbDkController.exe, UsbDkInstHelper.exe)

- **Calling conventions:** `__cdecl` in project files is ignored on ARM64; however, for cleanliness the MSBuild `<CallingConvention>` property should be absent from ARM64 configurations.
- **WOW64 helper:** The installer currently ships an x86 `UsbDkHelper.dll` as a WOW64 shim alongside the x64 version. For ARM64, an ARM64 native `UsbDkHelper.dll` is required; an x86-on-ARM64 WOW64 shim may also be needed for 32-bit guest processes (i.e., ship the existing x86 build as `UsbDkHelper_x86.dll` inside the ARM64 installer).

### Signing

- ARM64 kernel drivers must be EV-signed or WHQL-submitted for production deployment on Windows 11 ARM64 (same requirement as x64).
- During development, enable test-signing mode: `bcdedit /set testsigning on`.
- The `.cat` submission must declare `NTarm64` in its OS attribute list.

---

## Recommended Fix Order

1. **Build system** — Add ARM64 platform configurations to all five `.vcxproj` files and to `UsbDk.sln` (findings #1–#9). This is a prerequisite for everything else.
2. **Source guards** — Fix `UsbDkCompat.cpp` (`_WIN64` / ARM64 guard) and `UsbDkController.cpp` (`__cdecl`) (findings #10–#11).
3. **INF** — Add `NTarm64` sections to `UsbDk.inf` (finding #14).
4. **Build script** — Add ARM64 to `buildAll.bat` loop (findings #16–#17).
5. **Installer** — Add ARM64 MSI support (findings #18–#20).
6. **CI** — Add a GitHub Actions ARM64 build workflow (finding #21).
7. **Docs** — Update `README.md` with ARM64 prerequisites and testing instructions.
