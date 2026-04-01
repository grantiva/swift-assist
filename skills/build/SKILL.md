---
name: build
description: Rebuild the iOS app after code changes and install it on the booted simulator. Use after editing Swift source files when you need a fresh build without re-running init.
argument-hint: "[--scheme=<name>] [--simulator=<name>]"
---

# Swift Assist: Build

Rebuild the app and install it on the simulator using Grantiva's build pipeline. Use this after making code changes - it's faster than re-running `/swift-assist:init` when the project is already configured.

## Usage

```
/swift-assist:build
/swift-assist:build --scheme=MyApp
/swift-assist:build --scheme=MyApp --simulator="iPhone 16 Pro"
```

## Command Options

- `--scheme=<name>`: Xcode scheme to build (reads from `grantiva.yml` if omitted)
- `--simulator=<name>`: Target simulator (reads from `grantiva.yml` if omitted)

## What This Command Does

### Phase 1: Read Config

Load scheme and simulator from `grantiva.yml` if not provided via arguments:

```bash
cat grantiva.yml
```

If `grantiva.yml` doesn't exist, tell the user to run `/swift-assist:init` first.

### Phase 2: Build

Use Grantiva to build:

```bash
grantiva build
```

This resolves the project, boots the simulator if needed, and compiles the app. If `build_settings` are defined in `grantiva.yml`, they're applied automatically.

If the build fails, report the full error output and stop. Do not proceed.

### Phase 4: Restart Runner

If the Grantiva runner was active, restart it so it reconnects to the new build:

```bash
grantiva runner start --bundle-id <BUNDLE_ID> --simulator "<SIMULATOR>"
```

### Phase 5: Confirm

```
Build complete.

  Scheme:     <SCHEME>
  Simulator:  <SIMULATOR>
  Bundle ID:  <BUNDLE_ID>

App is running. Run /swift-assist:doctor to audit or /swift-assist:test to run flows.
```

## Important Rules

1. NEVER proceed past a failed build - stop and show the full error
2. Read scheme and simulator from `grantiva.yml` before asking the user
3. Do not modify source code
4. Use `grantiva` for building - never call `xcodebuild` directly
