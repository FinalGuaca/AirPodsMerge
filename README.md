# AirPods Merge

AirPods Merge is a Windows application that combines the AirPodsDesktop user
interface with the LibrePods Apple Accessory Protocol (AAP) feature set.

It keeps the battery, ear-detection, low-latency and popup features of
AirPodsDesktop, and adds AAP controls such as noise control, custom equalizer,
head tracking, headphone accommodation, multipoint device reporting and stem
press handling. The AAP features require the included `AapL2cap` Bluetooth
L2CAP profile driver.

The driver is test-signed. It is intended for development and personal use on a
machine where test signing is enabled.

> **Validation status:** the AAP implementation is validated with packet vectors
> and a driver loopback device. Real AirPods L2CAP/AAP traffic has not been
> validated on physical hardware in this repository.

## Repositories

- Installer and user guide (private): <https://github.com/FinalGuaca/AirPodsMerge>
- Source code (private): <https://github.com/FinalGuaca/AirPodsMerge-Source>

## Features

### Works without the AAP driver

- Battery information in the notification area.
- Automatic ear detection and media playback control.
- Low audio latency mode.
- Animated device popups, light/dark/system themes.

### Requires the AapL2cap driver and an AAP connection

- Noise control: Off, ANC, Transparency, Adaptive.
- Adaptive noise level.
- Conversational awareness.
- Hearing aid and hearing assist toggles.
- One-bud ANC.
- Adaptive volume.
- Volume swipe.
- Allow Off in the noise cycle.
- Device rename and device/firmware information.
- Three-band custom equalizer (`0x63`).
- Head tracking start/stop (`0x17`, experimental).
- Headphone accommodation / audiogram profile (`0x53`, experimental).
- Connected-device list for multipoint (`0x2E`, read-only).
- Stem press handling with local Windows actions (`0x19`, disabled by default).

Not implemented: multipoint audio-source switching, stem remapping written to the
AirPods firmware, heart-rate monitoring, VendorID spoofing.

## Requirements

- Windows 10 version 1809 (build 17763) or newer, or Windows 11, x64.
- AirPods paired with Windows over Bluetooth for real AAP features.
- Administrator rights for the installer and driver setup.
- **Test signing enabled** and **Secure Boot disabled** for the test-signed
  kernel driver.
- For building: Visual Studio 2022/2026 with C++20, Qt 6.8.3 MSVC x64,
  CMake, vcpkg, the Windows Driver Kit (driver), and NSIS 3.x (installer).

## Download and install

Download the installer from the installer repository:

<https://github.com/FinalGuaca/AirPodsMerge/blob/master/AirPodsMerge-0.6.1-win64.exe>

The installer is not code-signed, so Windows SmartScreen may show a warning
before it runs. SHA-256:

```text
da0c55dc173c1788f3253a011b856fdfcf8041fa9f6ae3d9ee4e43adec71b4d9
```

### 1. Enable test signing

Open an elevated PowerShell:

```powershell
bcdedit /set testsigning on
```

Disable Secure Boot in the machine's UEFI/BIOS settings. A test-signed driver
cannot load while Secure Boot is enabled.

Reboot, then verify:

```powershell
bcdedit /enum "{current}"
```

The output must contain:

```text
testsigning             Yes
```

### 2. Run the installer

Right-click `AirPodsMerge-0.6.1-win64.exe` and choose **Run as administrator**.

The installer:

- copies the application and Qt runtime to `C:\Program Files\AirPodsMerge`;
- installs `Driver\AapL2cap.sys`, `AapL2cap.inf`, `AapL2cap.cat` and the public
  test certificate;
- imports `AapL2capTest.cer` into `LocalMachine\Root` and
  `LocalMachine\TrustedPublisher`;
- runs `pnputil /add-driver Driver\AapL2cap.inf /install`;
- keeps an existing `ROOT\AapL2capTest` loopback device on the new driver, if
  `devcon.exe` is available;
- creates a desktop shortcut and an uninstall entry.

The installer does **not** change boot configuration. If test signing is off or
Secure Boot is on, it writes `Driver\install-warning.txt` and shows a warning.
The application still installs and the non-AAP features still work.

### 3. Connect the AirPods

1. Pair the AirPods with Windows through **Settings > Bluetooth & devices**.
2. Start **AirPods Merge**.
3. Open **Settings > Features** and scroll to the AAP cards.
4. Keep the AirPods connected while using AAP features.

The driver binds the AAP service when Windows enumerates the AirPods Bluetooth
device. AAP traffic is exposed through the device path `\\.\AapL2cap`.

## Verify the driver

Open Device Manager and look for:

- **AirPods AAP L2CAP Channel** under Bluetooth.
- **AirPods AAP L2CAP Test Device** only if the optional loopback device was
  installed.

Useful commands:

```powershell
Get-PnpDevice -FriendlyName "*AirPods AAP*"
pnputil /enum-drivers
Get-Content "$env:APPDATA\AirPodsDesktop\AirPodsDesktop.log" -Tail 50
```

A driver with **Code 52** is not signature-valid: check test signing, Secure Boot,
and the certificate stores. A driver with **Code 10** or **Code 28** usually needs
a reboot or a driver reinstall.

## Manual driver installation

If the installer's driver step fails, run an elevated PowerShell from the
installation directory:

```powershell
certutil -addstore -f Root Driver\AapL2capTest.cer
certutil -addstore -f TrustedPublisher Driver\AapL2capTest.cer
pnputil /add-driver Driver\AapL2cap.inf /install
```

Check the result:

```powershell
pnputil /enum-drivers
Get-PnpDevice -FriendlyName "*AirPods AAP*"
```

### Optional loopback test device

The loopback device lets you exercise the driver and AAP state machine without
AirPods. It is not needed for normal use and is not created by the installer.

```powershell
powershell -ExecutionPolicy Bypass -File Driver\install-driver.ps1 `
    -InstallDir "C:\Program Files\AirPodsMerge" -CreateTestDevice
```

This requires `devcon.exe`. The installer bundles it when the build machine has
the Windows Driver Kit. The test device appears as `ROOT\AapL2capTest` and the
driver creates `\\.\AapL2cap`.

## Build from source

```powershell
git clone https://github.com/FinalGuaca/AirPodsMerge-Source.git
cd AirPodsMerge

cmake -S . -B Build -DAPD_BUILD_TESTS=ON
cmake --build Build --config RelWithDebInfo
ctest --test-dir Build -C RelWithDebInfo --output-on-failure
```

The application is written to `Build/Binary/AirPodsDesktop.exe`.

### Build the driver

The driver is a KMDF Bluetooth L2CAP profile driver. It requires the Windows
Driver Kit and administrator rights.

```powershell
$env:APD_DRIVER_PFX_PASSWORD = "choose-a-test-password"
powershell -ExecutionPolicy Bypass -File Driver\AapL2cap\install-test.ps1
```

`install-test.ps1` builds the catalog, signs the driver with the test
certificate, imports the certificate, adds the driver package, and creates the
`ROOT\AapL2capTest` loopback device. For later rebuilds use:

```powershell
$env:APD_DRIVER_PFX_PASSWORD = "choose-a-test-password"
powershell -ExecutionPolicy Bypass -File Driver\AapL2cap\redeploy.ps1
```

The PFX private key and signed package are local build artifacts and are not
committed.

### Build the installer

With NSIS 3.x (`makensis.exe`) on `PATH` and the driver package staged:

```powershell
cmake -S . -B Build -DAPD_GENERATE_INSTALLER=ON
cmake --build Build --config RelWithDebInfo --target AirPodsDesktop
```

The installer is written to:

```text
Build/Installer/AirPodsMerge-0.6.1-win64.exe
```

The package includes the application, Qt runtime, translations, the signed
driver package and the public test certificate. The private signing key is not
included.

## AAP feature notes

### Custom equalizer (0x63)

Three bands (low, mid, high), each 0-100. The packet is:

```text
04 00 04 00 63 00 05 00 01 <state> <low> <mid> <high>
```

`state` is `2` for enabled and `1` for disabled.

### Headphone accommodation (0x53)

An eight-band floating-point profile. The UI exposes sliders in the range
`-12..+12 dB`; the exact gain semantics are not documented by LibrePods and
remain experimental. The packet repeats the eight values four times.

### Stem controls (0x19)

The app parses left/right single, double, triple and long presses and executes
local Windows actions:

- Play/Pause
- Previous track
- Next track
- Digital assistant (opens the Windows Start menu; Siri is not available)
- Cycle noise control

Stem actions are disabled by default to avoid double handling with the Bluetooth
media profile (AVRCP). The mapping is stored in the application settings; it is
not written to the AirPods.

### Multipoint (0x2E)

The app displays the devices reported by the AirPods, including the MAC address
and two information bytes. Audio routing remains controlled by the accessory.
Source switching is not implemented.

## Troubleshooting

### The app says "Unavailable"

- The AirPods are not paired or not connected.
- The BLE advertisement is not visible to Windows.
- The AAP service has not been enumerated yet; wait for the Bluetooth link and
  reopen the app.
- Check `%APPDATA%\AirPodsDesktop\AirPodsDesktop.log`.

### AAP cards are disabled

The AAP channel is not open. Check Device Manager for the AAP L2CAP device, check
`\\.\AapL2cap`, and confirm the driver has no error code.

### Driver Code 52

The test signature is not trusted. Confirm:

```powershell
bcdedit /enum "{current}"
```

contains `testsigning Yes`, Secure Boot is disabled, and `AapL2capTest.cer` is in
both `LocalMachine\Root` and `LocalMachine\TrustedPublisher`.

### Installer warnings

See:

```text
C:\Program Files\AirPodsMerge\Driver\install-warning.txt
C:\Program Files\AirPodsMerge\Driver\install.log
```

### Uninstall

Use **Settings > Apps > AirPods Merge > Uninstall**. The uninstaller removes the
loopback device, the AapL2cap driver package, the test certificate and the
application files. User settings in `%APPDATA%\AirPodsDesktop` are preserved.

## Source repository layout

```text
Source/Core/Aap/        AAP framing, session state, packet builders/parsers
Source/Gui/             Qt Widgets UI, including AAP settings cards
Driver/AapL2cap/        KMDF Bluetooth L2CAP profile driver and INF
Installer/              Installer driver scripts and end-user notes
Tests/                  Unit tests and driver loopback tests
Docs/                   Architecture, build and feature matrix
```

The installer repository contains this guide, `LICENSE` and the installer
executable only.

## Validation status

- AAP framing, packet vectors, handshake, custom EQ, head tracking, headphone
  accommodation, stem-press parsing and connected-device parsing are covered by
  unit tests.
- The KMDF driver's IOCTL path is tested through the `ROOT\AapL2capTest`
  loopback device.
- Real AirPods L2CAP channel opening, SDP enumeration and AAP notification
  exchange have not been validated on hardware.
- The driver is test-signed only; there is no WHQL/EV distribution signature.

## License

GPL-3.0. See [LICENSE](/LICENSE).

AirPods Merge includes code from
[AirPodsDesktop](https://github.com/SpriteOvO/AirPodsDesktop) and
[LibrePods](https://github.com/kavishdevar/librepods), both GPL-3.0.

## Third-party dependencies

- [Qt 6.8.3](https://qt.io/download-qt-installer) ([LGPLv3](https://doc.qt.io/qt-6/lgpl.html))
- [spdlog](https://github.com/gabime/spdlog) ([MIT](https://github.com/gabime/spdlog/blob/v1.x/LICENSE))
- [cxxopts](https://github.com/jarro2783/cxxopts) ([MIT](https://github.com/jarro2783/cxxopts/blob/master/LICENSE))
- [cpr](https://github.com/whoshuu/cpr) ([MIT](https://github.com/whoshuu/cpr/blob/master/LICENSE))
- [nlohmann/json](https://github.com/nlohmann/json) ([MIT](https://github.com/nlohmann/json/blob/develop/LICENSE.MIT))
- [SingleApplication](https://github.com/itay-grudev/SingleApplication) ([MIT](https://github.com/itay-grudev/SingleApplication/blob/master/LICENSE))
- [pfr](https://github.com/boostorg/pfr) ([BSL-1.0](https://github.com/boostorg/pfr/blob/develop/LICENSE_1_0.txt))
- [magic_enum](https://github.com/Neargye/magic_enum) ([MIT](https://github.com/Neargye/magic_enum/blob/master/LICENSE))
- [stacktrace](https://github.com/boostorg/stacktrace) ([BSL-1.0](https://www.boost.org/LICENSE_1_0.txt))
- [Inter](https://github.com/FinalGuaca/AirPodsMerge-Source/blob/master/Source/Resource/Font/Inter/LICENSE.txt) (SIL Open Font License 1.1)
- [Noto Sans TC](https://github.com/FinalGuaca/AirPodsMerge-Source/blob/master/Source/Resource/Font/NotoSansTC/LICENSE.txt) (SIL Open Font License 1.1)

## Credits

- [OpenPods](https://github.com/adolfintel/OpenPods)
- [Discontinued Privacy: Personal Data Leaks in Apple Bluetooth-Low-Energy Continuity Protocols](https://hal.inria.fr/hal-02394619/document)
- [MagicPods](https://magicpods.app/)
