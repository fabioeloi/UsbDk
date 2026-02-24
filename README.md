[![Build Status](https://ci.appveyor.com/api/projects/status/p3s6bdbx8mq8o0hu?svg=true)](https://ci.appveyor.com/api/projects/status/p3s6bdbx8mq8o0hu?svg=true)

# UsbDk

UsbDk (USB Development Kit) is a open-source library for Windows meant
to provide user mode applications with direct and exclusive access to
USB devices by detaching those from Windows PNP manager and device drivers
and providing user mode with API for USB-specific operations on the device.

The library is intended to be as generic as possible, support  all types of
USB devices, bulk and isochronous transfers, composite devices etc.

Library supports all Windows OS versions starting from Windows XP/2003.
ARM64 (Windows 11 on ARM) support is available for Win10/Win11 targets.
See [daynix/UsbDk#92](https://github.com/daynix/UsbDk/issues/92) for the
upstream tracking issue.

## Documentation

* See ARCHITECTURE document in the source tree root.
* See Documentation folder in the source tree root.
* See UsbDkHelper\UsbDkHelper.h UsbDkHelper\UsbDkHelperHider.h for API documentation

## Building

**Tools required (x86/x64):**

* Visual Studio 2015/Visual Studio 2015 Express update 3 or newer
* WDK 10
* Windows 10 SDK
* Wix Toolset V3.8 (for building MSI installer)
* WDK 7.1 (for Windows XP/2003/Vista/2008 builds)

### ARM64 Build Prerequisites

To build the ARM64 driver and user-mode components the following additional
tools are required:

* **Visual Studio 2022** (or later) with the **ARM64 build tools** component
  installed (`MSVC v143 - VS 2022 C++ ARM64 build tools`).
* **WDK 10.0.22621+** (Windows 11 WDK) — this is the first WDK version that
  ships full ARM64 KMDF support (`WindowsKernelModeDriver10.0` toolset with
  ARM64 target).  Install from
  <https://learn.microsoft.com/windows-hardware/drivers/download-the-wdk>.
* **Windows 11 ARM64 test device or VM** (e.g., QEMU + EDK2 UEFI on Apple
  Silicon, or a Snapdragon X / Ampere device).

> **Important:** Use *pure ARM64*, **not** ARM64EC.  ARM64EC is a user-mode
> ABI for interoperability and is invalid for kernel-mode drivers.

***Compilation (x86/x64)***

Just open UsbDk.sln from the source tree root in Visual Studio 2015 and compile
desired configuration.

***Compilation (ARM64)***

Open `UsbDk.sln` in Visual Studio 2022, select the `Win10 Release|ARM64`
(or `Win10 Debug_NoSign|ARM64`) configuration, and build.

Alternatively, from the command line:

```cmd
msbuild UsbDk.sln /p:Configuration="Win10 Release" /p:Platform=ARM64 /m
```

## Installing and running

Use UsbDkController.exe to install/uninstall and verify basic operation.
Run UsbDkController.exe without parameters for command line options.

## Testing on ARM64

1. **Enable test-signing** on the target device (development only):
   ```cmd
   bcdedit /set testsigning on
   ```
   Reboot the device.

2. **Copy the build output** from
   `Install\ARM64\Win10 Release\` to the ARM64 device.

3. **Install the driver** using `UsbDkController.exe`:
   ```cmd
   UsbDkController.exe -i
   ```

4. **Verify** the driver is loaded:
   ```cmd
   UsbDkController.exe -s
   ```

5. For production deployment, the driver must be **EV-signed** or
   submitted for **WHQL signing**. The `.cat` file submission must declare
   `NTarm64` in its OS attribute list.

## Known issues

* Installation on 64-bit versions of Windows 7 fails if security update
  [3033929](https://technet.microsoft.com/en-us/library/security/3033929)
  is not installed. Reason: UsbDk driver is signed by SHA-256 certificate. Without this update
  Windows 7 does not recognize the signature properly and fails to load the driver.
