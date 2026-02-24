[![Build Status](https://ci.appveyor.com/api/projects/status/p3s6bdbx8mq8o0hu?svg=true)](https://ci.appveyor.com/api/projects/status/p3s6bdbx8mq8o0hu?svg=true)

# UsbDk

UsbDk (USB Development Kit) is a open-source library for Windows meant
to provide user mode applications with direct and exclusive access to
USB devices by detaching those from Windows PNP manager and device drivers
and providing user mode with API for USB-specific operations on the device.

The library is intended to be as generic as possible, support  all types of
USB devices, bulk and isochronous transfers, composite devices etc.

Library supports all Windows OS versions starting from Windows XP/2003,
and includes ARM64 support for Windows 10/11 on ARM.

## Documentation

* See ARCHITECTURE document in the source tree root.
* See Documentation folder in the source tree root.
* See UsbDkHelper\UsbDkHelper.h UsbDkHelper\UsbDkHelperHider.h for API documentation
* See [PORTING_NOTES_ARM64.md](PORTING_NOTES_ARM64.md) for ARM64 porting details.

## Building

**Tools required:**

* Visual Studio 2015/Visual Studio 2015 Express update 3 or newer
* WDK 10
* Windows 10 SDK
* Wix Toolset V3.8 (for building MSI installer)
* WDK 7.1 (for Windows XP/2003/Vista/2008 builds)

### ARM64 Build Prerequisites

To build the ARM64 targets you need:

* **Visual Studio 2022** (17.4 or newer) with the **ARM64/ARM64EC build tools** component
* **WDK 10.0.22621+** (Windows 11 WDK) — this ships KMDF 1.33 which provides full ARM64 support
* **Windows 11 SDK** (10.0.22621+)

Select the `Win10 Release|ARM64` (or Debug/Debug_NoSign) configuration in Visual Studio,
or build from the command line:

```
msbuild UsbDk.sln /p:Configuration="Win10 Release" /p:Platform=ARM64 /m
```

> **Note:** ARM64 configurations are only available for Win10 targets.
> Windows XP, 7, 8, and 8.1 never ran on ARM64 and have no ARM64 configurations.

***Compilation***

Just open UsbDk.sln from the source tree root in Visual Studio 2015 and compile
desired configuration.

## Installing and running

Use UsbDkController.exe to install/uninstall and verify basic operation.
Run UsbDkController.exe without parameters for command line options.

### Testing on ARM64

1. Obtain a Windows 11 ARM64 device or VM (e.g., Parallels on Apple Silicon,
   QEMU with UEFI, or a Snapdragon-based laptop).

2. Enable test signing (required for non-WHQL-signed drivers):
   ```
   bcdedit /set testsigning on
   ```
   Reboot the device.

3. Copy the ARM64 build output to the device:
   - `UsbDk.sys`, `UsbDk.inf`, `UsbDk.cat`
   - `UsbDkHelper.dll`
   - `UsbDkController.exe`

4. Install the driver:
   ```
   UsbDkController.exe -i
   ```

5. Enumerate USB devices to verify operation:
   ```
   UsbDkController.exe -n
   ```

6. For production deployment, the driver must be EV-signed or submitted to
   Microsoft for WHQL certification.

## Known issues

* Installation on 64-bit versions of Windows 7 fails if security update
  [3033929](https://technet.microsoft.com/en-us/library/security/3033929)
  is not installed. Reason: UsbDk driver is signed by SHA-256 certificate. Without this update
  Windows 7 does not recognize the signature properly and fails to load the driver.
