# Hot Target

Native macOS menu-bar utility that automatically adjusts fan speed to maintain a temperature you choose.

[**Download the latest direct release →**](https://github.com/kargnas/hottarget/releases/latest)

[![macOS 14+](https://img.shields.io/badge/macOS-14%2B-111827?logo=apple)](https://www.apple.com/macos/)
[![Swift 6](https://img.shields.io/badge/Swift-6-F05138?logo=swift&logoColor=white)](Package.swift)
[![GPL-3.0-only](https://img.shields.io/badge/license-GPL--3.0--only-2563eb)](LICENSE)

<p align="center">
  <img height="300" alt="hottarget" src="https://github.com/user-attachments/assets/c8bbd4ee-6527-4187-b5ca-e9f0ff0a29b1" />
</p>

## Maintain a target temperature

Choose **Maintain Target Temperature** from the menu and select 40–85 °C in 5 °C steps. The privileged helper reads the CPU temperature every second and adjusts each fan's existing SMC target to bring the latest 10-sample average toward your selection.

The feedback loop changes fan speed gradually instead of jumping straight between fixed speeds:

- Above the target, it requests 10 additional RPM per degree of error.
- Below the target, it requests 20 fewer RPM per degree of error.
- A separate stabilizer bounds subsequent changes to 100 RPM per second upward and 200 RPM per second downward.
- Inside a ±0.5 °C deadband, it holds the current target to avoid fan hunting.
- At an instantaneous 90 °C, it immediately requests the hardware maximum for every fan.

The signed privileged helper remains the source of truth and reapplies manual fan targets after sleep or an SMC reset. Quitting the app or losing the helper connection restores Apple automatic control.

While target-temperature control is active, Hot Target writes `timestamp,target_celsius,actual_celsius` once per second to daily CSV files in `~/Library/Logs/hottarget/`. Files whose last sample is older than 72 hours are deleted automatically. Choose **Temperature Log…** to inspect the retained history in a native window.

## Preset fan modes

Presets are available as secondary controls under **Preset Fan Modes**. Only one target-temperature or preset controller can be active at a time.

| Mode | Minimum | Behavior |
|---|---:|---|
| System Default | Apple automatic | Returns every fan to macOS control. |
| Quiet | 1,500 RPM | Holds a quiet floor until the hot threshold, then ramps toward maximum at 90 °C. |
| Ultra | Maximum | Runs every fan at 100%. |

Quiet ignores 3 °C cooldown fluctuations and applies bounded RPM changes to reduce audible fan hunting.

## Live temperature and fan status

| Screen | What it does |
|---|---|
| <img src="docs/menu-bar-combined.png" width="254" alt="Thermometer and exact CPU temperature in the macOS menu bar"> | Shows the animated thermometer and exact CPU temperature together. |
| <img src="docs/menu-bar-icon.png" width="350" alt="Thermometer-only menu-bar mode"> | Keeps the menu bar compact while color and mercury level carry the thermal state. |
| <img src="docs/fan-status.png" width="400" alt="Live fan RPM values"> | Reports the current speed of every detected fan. |

The paired-wave animation sends a left mote followed by a right mote after 0.45 seconds, then repeats every 3.6 seconds. Color moves continuously from healthy green at 45 °C through yellow to danger red at 80 °C.

## Stack at a glance

| Layer | Tech |
|---|---|
| Menu bar UI | AppKit + Core Animation |
| Temperature and fan sensors | IOKit + AppleSMC |
| Fan control | ServiceManagement + signed privileged XPC helper |
| Automatic updates | Sparkle 2 + EdDSA-signed notarized DMG |
| Build and tests | Swift 6 + Swift Package Manager |

Hot Target is distributed directly because exact AppleSMC temperature readings and privileged fan control are not available through public sandboxed macOS APIs.

## Install and enable fan control

Download the latest signed DMG from [GitHub Releases](https://github.com/kargnas/hottarget/releases/latest). Hot Target requires macOS 14 or later.

1. Open the DMG and drag **Hot Target.app** into **Applications**.
2. Quit other fan-control applications.
3. Choose **Maintain Target Temperature → Enable Fan Control…**.
4. Approve **Hot Target.app** in **System Settings → General → Login Items & Extensions**. Hot Target opens that pane for you and floats a chip under the window showing which switch to turn on; the chip disappears on its own once the approval lands.
5. Hot Target starts holding 60 °C as soon as the approval lands and confirms it in the same chip. Pick a different target under **Maintain Target Temperature** at any time.

### If target temperatures do not take effect

Quit other fan-control applications, then check the helper state:

```bash
"/Applications/Hot Target.app/Contents/MacOS/hottarget" --helper-status
```

If it reports `not registered` or `not found`, request registration directly:

```bash
"/Applications/Hot Target.app/Contents/MacOS/hottarget" --register-helper
```

`Operation not permitted` followed by `approval required` means registration is waiting for user approval. Enable **Hot Target.app** in **System Settings → General → Login Items & Extensions**, then run `--helper-status` again. `Fan helper: enabled` is the ready state. While target-temperature control is active, `--print-fans` reports manual mode and `~/Library/Logs/hottarget/` receives one CSV sample per second.

## Automatic updates

Release builds check for updates once per day through Sparkle. A downloaded update is verified with Hot Target's EdDSA key and installed when the app quits. Choose **Check for Updates…** in the status menu to run a manual check at any time.

Development bundles keep automatic checks and installation disabled so a local build is never replaced by a release build. Manual checks remain available in bundled development builds.

## Uninstall

1. Choose **Maintain Target Temperature → Preset Fan Modes → System Default**.
2. Before deleting the app, unregister its helper:

   ```bash
   "/Applications/Hot Target.app/Contents/MacOS/hottarget" --unregister-helper
   ```

   If Hot Target is installed elsewhere, replace `/Applications/Hot Target.app` with its actual path.
3. Quit Hot Target and remove **Hot Target.app**.
4. Delete `~/Library/Logs/hottarget/` to remove retained CSV history. To clear display preferences too, remove `~/Library/Preferences/as.kargn.hottarget.plist`.

## Build locally

```bash
git clone https://github.com/kargnas/hottarget.git
cd hottarget
./script/build_and_run.sh --verify
```

A local build requires a Developer ID Application or Apple Development signing identity. When a registered helper already exists, the build script unregisters it before replacing the signed app bundle and registers the new helper afterward.

## Verify the build

```text
$ swift test
Executed 17 tests, with 0 failures
```

Licensed under [GPL-3.0-only](LICENSE). MIT notices for Stats, smctl, and MacFanControl remain in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
