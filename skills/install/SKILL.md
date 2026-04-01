---
name: install
description: Build, install, and launch the app on the simulator using Grantiva. Use when you need the full build-install-launch cycle, mirroring what CI does. Prefer this over build when you want the app running immediately after compilation.
argument-hint: "[--scheme=<name>] [--simulator=<name>]"
---

# Swift Assist: Install

Build, install, and launch the app on the simulator using Grantiva. This mirrors the full CI flow. Use this when you want the app running and ready for testing immediately after a code change.

The difference from `/swift-assist:build`: `build` compiles only. `install` compiles, installs on the simulator, and launches the app.

## Usage

```
/swift-assist:install
/swift-assist:install --scheme=MyApp
/swift-assist:install --scheme=MyApp --simulator="iPhone 16 Pro"
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

### Phase 2: Install and Launch

Use Grantiva to build, install, and launch:

```bash
grantiva install
```

This resolves the project, boots the simulator if needed, builds the app, installs it, and launches it - mirroring the CI flow.

### Phase 3: Start Runner

Start the Grantiva runner so the hierarchy and doctor commands can inspect the app:

```bash
grantiva runner start --bundle-id <BUNDLE_ID> --simulator "<SIMULATOR>"
```

### Phase 4: Confirm

```
Install complete.

  Scheme:     <SCHEME>
  Simulator:  <SIMULATOR>
  Bundle ID:  <BUNDLE_ID>

App is running. Run /swift-assist:doctor to audit or /swift-assist:test to run flows.
```

## Important Rules

1. Verify the `.app` path exists before attempting to install
2. Do not modify source code
3. If the runner fails to start, still report the install as successful - the runner is not required for basic testing
