![Bluesnooze logo](images/icon.png)

# Bluesnooze

![Screenshot of the Bluesnooze menu with the Selected devices submenu open](images/screenshot.png)

> [!IMPORTANT]
> This is my fork of [odlp/bluesnooze](https://github.com/odlp/bluesnooze).
> It started as a small personal fork because the upstream had not seen
> a commit since July 2023, and I wanted Bluesnooze to keep working well
> on my setup. Since then, the fork has grown into a broader refresh of
> the app. I may keep improving it, but I make no promises about ongoing
> maintenance and I am not accepting pull requests, feature requests, or
> bug reports. If you need different behavior, please fork it.

Bluesnooze is a macOS menu bar app for people who pair the same
Bluetooth audio device with both a Mac and another device, such as a
phone.

When a sleeping Mac wakes briefly in the background, it can reconnect to
headphones or speakers and interrupt audio playing elsewhere. Bluesnooze
prevents that by disconnecting Bluetooth devices when the Mac sleeps and
restoring them when the Mac wakes.

This fork requires macOS 13 (Ventura) or later. The app now uses Apple's
modern login-item API, which is only available on macOS 13+.

## What this fork changes

The original app solved the basic problem by turning Bluetooth off on
sleep and on again at wake. This fork keeps that mode, but adds a few
things I needed for daily use:

- **Disconnect selected devices on sleep**

  Instead of turning the whole Bluetooth controller off, Bluesnooze can
  disconnect only the devices you pick, usually headphones or speakers.
  The controller stays on, so a Bluetooth keyboard or mouse can still
  wake the Mac.

- **Restore previous Bluetooth state on wake**

  Bluesnooze remembers whether Bluetooth was on before sleep. If it was
  already off, it stays off after wake. In selected-device mode, this
  also means only devices that were actually connected before sleep are
  reconnected.

- **Hide the menu bar icon**

  The icon can be hidden from the menu. In the original app, restoring
  it required a `defaults` command. In this fork, just launch
  Bluesnooze again from Finder, Spotlight, Raycast, or a similar app
  launcher. The already-running instance will show the icon again.

- **Faster and safer reconnect handling**

  Selected devices are reconnected asynchronously after wake, with
  duplicate-wake debouncing. The first attempt waits about a second for
  the Bluetooth stack to finish waking (an immediate request is often
  refused), then retries continue for about 25 seconds. A connection
  attempt that macOS never reports back on only blocks retries for a few
  seconds, so one lost callback cannot stop the reconnect.

## What was modernized

This fork is also a cleanup of the project itself:

- Migrated dependency management from Carthage to Swift Package Manager.
- Replaced `LaunchAtLogin-Legacy` with
  [`LaunchAtLogin-Modern`](https://github.com/sindresorhus/LaunchAtLogin-Modern),
  which uses Apple's `SMAppService` API.
- Removed the Objective-C/C bridging header. The codebase is now pure
  Swift.
- Replaced `@NSApplicationMain` with `@main`.
- Replaced C-style `os_log` calls with Swift's `Logger` API.
- Resolved the two private `IOBluetooth` power-state symbols at runtime
  instead of using `@_silgen_name`.
- Split the old all-in-one `AppDelegate.swift` into smaller files:
  - `AppDelegate.swift`
  - `BluetoothController.swift`
  - `BluetoothDeviceMenuItem.swift`
  - `Preferences.swift`
- Added SwiftLint locally and in CI.
- Added a small `Makefile` for local development.
- Added GitHub Actions CI for linting and building.
- Bumped the minimum macOS version to 13.

## Settings

All settings live in the menu bar icon:

- **Launch at login**
- **Restore previous Bluetooth state on wake**
- **Disconnect selected devices on sleep**
- **Selected devices**
- **Hide icon**
- **Quit**

## Installation

Build and install from source (Xcode required, no Apple developer
account needed):

```sh
git clone https://github.com/tehjaymo/bluesnooze.git
cd bluesnooze
xcodebuild -project Bluesnooze.xcodeproj -scheme Bluesnooze \
  -configuration Release -derivedDataPath build/DerivedData build
ditto build/DerivedData/Build/Products/Release/Bluesnooze.app \
  /Applications/Bluesnooze.app
open /Applications/Bluesnooze.app
```

Quit the running copy first when reinstalling over an existing one.

Prebuilt releases are available from
[mre/bluesnooze](https://github.com/mre/bluesnooze/releases/latest),
but those use the upstream `com.oliverpeate.Bluesnooze` bundle
identifier, not this fork's. The original app is at
[odlp/bluesnooze](https://github.com/odlp/bluesnooze) or via
`brew install bluesnooze`.

## Code signing

The project signs **ad-hoc** (`CODE_SIGN_IDENTITY = "-"`, no
`DEVELOPMENT_TEAM`). That means:

- **It builds on any Mac with Xcode.** No Apple ID, certificate or team
  is needed.
- **It runs on the Mac that built it.** A locally built app has no
  quarantine flag, so Gatekeeper does not block it.
- **It is not notarized.** If you copy the app to another Mac (download,
  AirDrop, zip), Gatekeeper will block it. Allow it under System
  Settings → Privacy & Security → Open Anyway, or clear the flag with
  `xattr -dr com.apple.quarantine /Applications/Bluesnooze.app`.
- **Each rebuild is a new signature.** Ad-hoc signatures are tied to the
  exact binary, so after reinstalling a rebuilt copy macOS may ask for
  Bluetooth access again, and **Launch at login** may need toggling off
  and on. Settings are normally kept, since they are stored by bundle
  identifier.

`make build` (used by CI) disables signing entirely. That is fine for
checking the code compiles, but the resulting app lacks its sandbox and
Bluetooth entitlements, so do not install it. Use the `xcodebuild`
command above instead.

To sign with your own Apple developer team, which gives a stable
identity across rebuilds, pass it on the command line:

```sh
xcodebuild -project Bluesnooze.xcodeproj -scheme Bluesnooze \
  -configuration Release -derivedDataPath build/DerivedData \
  CODE_SIGN_STYLE=Automatic CODE_SIGN_IDENTITY="Apple Development" \
  DEVELOPMENT_TEAM=YOURTEAMID build
```

or set it under the target's **Signing & Capabilities** in Xcode.
Distributing to other people additionally needs a Developer ID
certificate and notarization.

## Migrating from upstream

This fork uses its own bundle identifier, `net.thejaymo.Bluesnooze`, so
settings from the upstream app (`com.oliverpeate.Bluesnooze`) are not
picked up. Reselect your devices and options once after installing.

If you use **Launch at login**, toggle it once after upgrading. This
fork uses `SMAppService`, and the old login-item registration does not
carry over.

## Caveats

- Bluesnooze still relies on private Bluetooth APIs. That is why it
  cannot be distributed through the App Store.
- The selected-devices list only includes Bluetooth Classic devices.
  BLE devices are hidden because `IOBluetoothDevice.closeConnection` is
  unreliable for them.
- Some Apple peripherals using randomized Bluetooth addresses may appear
  as `Unnamed device (12-34-...)`. The legacy `IOBluetooth` API does not
  always expose the friendly name that System Settings shows.
- It is not compatible with "Allow your Apple Watch to unlock your Mac".

## Development

Install local tools:

```sh
make bootstrap
```

Run linting and build locally:

```sh
make lint
make build
```

Run the same checks CI runs:

```sh
make ci
```

## Logs

Bluesnooze uses Swift's `Logger` API.
Here are some useful commands if you'd like to check how quickly Bluesnooze responds to sleep/wake events:

```sh
log stream --predicate 'process == "Bluesnooze"'
log show --process Bluesnooze --info --last 1h
```

## Configuration via `defaults`

```sh
# Read all settings
defaults read net.thejaymo.Bluesnooze

# Always force Bluetooth on at wake
defaults write net.thejaymo.Bluesnooze restorePreviousStateOnWake -bool false

# Per-device disconnect mode
defaults write net.thejaymo.Bluesnooze disconnectDevicesOnSleep -bool true
defaults write net.thejaymo.Bluesnooze devicesToDisconnectOnSleep \
    -array "12-34-4a-f0-19-02" "00-1d-43-aa-bb-cc"

# Hide / show the menu bar icon from the terminal
defaults write net.thejaymo.Bluesnooze hideIcon -bool true && killall Bluesnooze
defaults delete net.thejaymo.Bluesnooze hideIcon && killall Bluesnooze

# You can also show the icon again by launching Bluesnooze normally.
```

## Credits

Original app by [Oliver Peate](https://github.com/odlp).
The selected Bluetooth devices feature was originally proposed by
[ramatah](https://github.com/ramatah) in
[stefansundin/bluesnooze#6](https://github.com/stefansundin/bluesnooze/pull/6).
