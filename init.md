argument-hint: [--scheme=<name>] [--bundle-id=<id>] [--simulator=<name>]
description: Initialize a Grantiva project, build the app, and launch it in the simulator. Use when starting a new swift-assist workflow, setting up VRT, or first-time project configuration. Triggers on "set up grantiva", "init project", "build and run".
---

# Swift Assist: Init

Set up Grantiva for the current iOS project, build the app, and launch it in the simulator.

## Usage

```
/swift-assist:init
/swift-assist:init --scheme=MyApp
/swift-assist:init --scheme=MyApp --simulator="iPhone 16 Pro"
/swift-assist:init --bundle-id=com.example.MyApp
```

## Command Options

- `--scheme=<name>`: Xcode scheme to build (auto-detected if only one exists)
- `--bundle-id=<id>`: App bundle identifier (auto-derived from build output if omitted)
- `--simulator=<name>`: Target simulator device (e.g., "iPhone 16 Pro")

## Prerequisites

Verify the Grantiva CLI is installed before proceeding:

```bash
which grantiva
```

If not installed, tell the user:
```
Grantiva CLI is required. Install it with:
  brew install grantiva/tap/grantiva
```

## What This Command Does

### Phase 1: Project Discovery

1. Find `.xcworkspace` or `.xcodeproj` in the working directory
   - Prefer `.xcworkspace` if both exist
   - If multiple found, ask the user which to use
   - If none found, stop and tell the user

2. Detect available schemes:
   ```bash
   grantiva doctor --json
   ```
   - If `--scheme` provided, use it
   - If only one scheme exists, use it automatically
   - If multiple, ask the user

### Phase 2: Environment Check

Run Grantiva's built-in environment diagnostic:

```bash
grantiva doctor
```

This checks macOS version, Xcode, Swift, simulator availability, and CLI tools. If any checks fail, report the issues and stop.

### Phase 3: Initialize Grantiva Config

Check if `grantiva.yml` already exists:

```bash
ls grantiva.yml 2>/dev/null
```

- If it exists, ask the user: "grantiva.yml already exists. Overwrite it or keep the current config?"
- If it doesn't exist, create it:

```bash
grantiva init --scheme=<SCHEME> --bundle-id=<BUNDLE_ID>
```

If `--bundle-id` was not provided, omit it (it will be derived after build).

### Phase 4: Simulator Selection

1. If `--simulator` was provided, find that device:
   ```bash
   xcrun simctl list devices available --json
   ```

2. If not provided, check for booted simulators:
   ```bash
   xcrun simctl list devices booted --json
   ```
   - Single booted simulator: use it
   - No booted simulator: find the best available iPhone, boot it:
     ```bash
     xcrun simctl boot <DEVICE_UDID>
     open -a Simulator
     ```
   - Multiple booted: ask the user which one

### Phase 5: Check for Mock Services

Before building, check if the app loads data from an API that won't be available during testing:

1. Search for `.mock` or `.preview` service variants in the source:
   ```bash
   grep -rl "\.mock\|UI_TESTING" --include="*.swift" .
   ```

2. If the app has a `#if UI_TESTING` check or mock services pattern, build with the flag:
   ```bash
   grantiva diff capture --scheme <SCHEME> --simulator "<SIMULATOR>"
   ```
   Note: if the app needs `UI_TESTING`, the user should build with
   `SWIFT_ACTIVE_COMPILATION_CONDITIONS='DEBUG UI_TESTING'` and then use
   `--app-file` to point Grantiva at the pre-built binary.

3. If no mock services exist and the app needs an API, tell the user:
   ```
   Your app loads data from an API. For reliable testing, consider adding
   a .mock services variant with a #if UI_TESTING flag. See the swift-assist
   docs for the pattern.
   ```

### Phase 6: Build & Capture Initial Screen

Use Grantiva to build and verify the app launches:

```bash
grantiva diff capture --scheme <SCHEME> --simulator "<SIMULATOR>"
```

This builds the app, installs it on the simulator, launches it, and captures the initial screen. If the build fails, stop and report the error.

If using a pre-built binary with mock services:
```bash
grantiva diff capture --app-file ./DerivedData/.../MyApp.app
```

### Phase 7: Start Runner for Inspection

Start the Grantiva runner session so the doctor command can inspect the hierarchy:

```bash
grantiva runner start --bundle-id <BUNDLE_ID> --simulator "<SIMULATOR>"
```

### Phase 8: Confirm Setup

Tell the user:

```
Project initialized successfully.

  Scheme:     <SCHEME>
  Simulator:  <SIMULATOR>
  Bundle ID:  <BUNDLE_ID>
  Config:     grantiva.yml

The app is running in the simulator. Next steps:
  /swift-assist:doctor    - Scan for missing accessibility identifiers
  /swift-assist:make-tests - Generate test flows
  /swift-assist:vrt       - Set up visual regression testing
```

## Important Rules

1. NEVER proceed past a failed build - stop and report the error
2. Do not modify any source code in this command
3. If `grantiva doctor` reports issues, stop and help the user fix them before continuing
4. Always confirm the simulator choice before building if there are multiple options
5. Keep the simulator running after this command completes
6. Use `grantiva` CLI for building - never call `xcodebuild` directly
