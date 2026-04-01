---
name: install
description: Install a pre-built app binary onto the simulator without rebuilding. Use when you have a compiled .app file and want to skip the build step, such as in CI or when using a build artifact.
argument-hint: "[--app-file=<path>] [--simulator=<name>]"
---

# Swift Assist: Install

Install a pre-built `.app` binary onto the simulator using Grantiva. Use this when you already have a compiled build - for example, when CI produced a binary, when you built with custom flags outside of Grantiva, or when you want to test a specific artifact without rebuilding.

## Usage

```
/swift-assist:install --app-file=./build/MyApp.app
/swift-assist:install --app-file=./DerivedData/Build/Products/Debug-iphonesimulator/MyApp.app
/swift-assist:install --app-file=./MyApp.app --simulator="iPhone 16 Pro"
```

## Command Options

- `--app-file=<path>`: Path to the `.app` bundle to install (required)
- `--simulator=<name>`: Target simulator (reads from `grantiva.yml` if omitted)

## What This Command Does

### Phase 1: Locate the App File

If `--app-file` was not provided, search for a `.app` bundle in common locations:

```bash
find . -name "*.app" -path "*/Debug-iphonesimulator/*" -not -path "*/node_modules/*" | head -5
```

If multiple are found, list them and ask the user which to use. If none are found, tell the user to run `/swift-assist:build` first.

### Phase 2: Confirm Booted Simulator

Check that the target simulator is booted:

```bash
xcrun simctl list devices booted --json
```

If not booted, boot it:

```bash
xcrun simctl boot <DEVICE_UDID>
open -a Simulator
```

### Phase 3: Install and Launch

Use Grantiva to install the pre-built binary:

```bash
grantiva diff capture --app-file <APP_FILE_PATH>
```

This installs the `.app` on the simulator and launches it. No compilation step.

### Phase 4: Read Bundle ID

Extract the bundle ID from the installed app:

```bash
/usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" <APP_FILE_PATH>/Info.plist
```

### Phase 5: Start Runner

Start the Grantiva runner so the hierarchy and doctor commands can inspect the app:

```bash
grantiva runner start --bundle-id <BUNDLE_ID> --simulator "<SIMULATOR>"
```

### Phase 6: Confirm

```
Install complete.

  App:        <APP_FILE_PATH>
  Simulator:  <SIMULATOR>
  Bundle ID:  <BUNDLE_ID>

App is running. Run /swift-assist:doctor to audit or /swift-assist:test to run flows.
```

## Common Use Cases

**Testing a UI_TESTING build:**
```bash
xcodebuild build -scheme MyApp \
  SWIFT_ACTIVE_COMPILATION_CONDITIONS="DEBUG UI_TESTING" \
  -derivedDataPath ./DerivedData
/swift-assist:install --app-file=./DerivedData/Build/Products/Debug-iphonesimulator/MyApp.app
```

**Installing a CI artifact:**
```
/swift-assist:install --app-file=./artifacts/MyApp.app
```

## Important Rules

1. Verify the `.app` path exists before attempting to install
2. Do not modify source code
3. If the runner fails to start, still report the install as successful - the runner is not required for basic testing
