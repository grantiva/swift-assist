---
name: test
description: Run generated test flows against the app in the simulator using Grantiva. Use after /swift-assist:make-tests to execute and validate your user flows.
argument-hint: "[--flow=<name>] [--all]"
---

# Swift Assist: Test

Run the generated flow YAML files against the app in the simulator using `grantiva run`.

## Usage

```
/swift-assist:test
/swift-assist:test --flow=login
/swift-assist:test --all
```

## Command Options

- `--flow=<name>`: Run a specific flow (matches filename in flows/ directory)
- `--all`: Run all flows sequentially

## Prerequisites

1. App must be running in the simulator
2. Flow YAML files must exist (run `/swift-assist:make-tests` first)

If no flows exist:
```
No flow files found in flows/. Run /swift-assist:make-tests first.
```

## What This Command Does

### Phase 1: Discover Flows

1. List available flow files:
   ```bash
   ls flows/*.yaml 2>/dev/null
   ```

2. If `--flow` specified, find matching file
3. If `--all` specified, collect all non-underscore-prefixed flows (skip `_setup-*.yaml` helper flows)
4. If neither specified, show available flows and ask which to run

### Phase 2: Run Flows via Grantiva

For each flow, execute it using `grantiva run`:

```bash
grantiva run flows/<flow-name>.yaml
```

### Phase 3: Report Results

After all flows complete:

```
Test Execution Report
=====================

Flows run: 5 | Passed: 4 | Failed: 1

  onboarding.yaml       PASS
  login.yaml            PASS
  settings.yaml         FAIL
  profile.yaml          PASS
  checkout.yaml         PASS
```

Report any error output from failed flows so the user can diagnose and fix.

## Important Rules

1. If a flow fails, continue running remaining flows (don't stop on first failure)
2. Do not modify flow files during execution
