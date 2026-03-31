---
name: test
description: Run generated test flows against the app in the simulator using Grantiva. Use after /swift-assist:make-tests to execute and validate your user flows.
argument-hint: [--flow=<name>] [--all] [--json]
---

# Swift Assist: Test

Run the generated flow YAML files against the app in the simulator using Grantiva's driver and capture pipeline.

## Usage

```
/swift-assist:test
/swift-assist:test --flow=login
/swift-assist:test --all
/swift-assist:test --json
```

## Command Options

- `--flow=<name>`: Run a specific flow (matches filename in flows/ directory)
- `--all`: Run all flows sequentially
- `--json`: Output results as JSON

## Prerequisites

1. App must be running in the simulator
2. Grantiva driver must be running
3. Flow YAML files must exist (run `/swift-assist:make-tests` first)
4. `grantiva.yml` must be configured

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

For each flow, execute it using Grantiva's capture pipeline which processes the screen definitions:

```bash
grantiva diff capture --scheme=<SCHEME> --simulator="<SIMULATOR>" --no-build
```

This runs through the screens defined in `grantiva.yml`, executing path actions (tap, type, swipe, assert_visible) and capturing screenshots at each screen endpoint.

### Phase 3: Monitor with Computer Use

While Grantiva executes the flows, use computer use to observe the simulator:

1. **Watch for unexpected states**: crashes, alerts, loading spinners that don't resolve
2. **Verify assertions**: when `assert_visible` steps run, visually confirm the element is actually on screen
3. **Capture evidence**: screenshot any failures or unexpected behavior
4. **Note timing issues**: if the app is slower than expected, suggest increasing `wait` durations

### Phase 4: Collect Results

After each flow completes, gather:

- Pass/fail status for each step
- Screenshots captured at each screen
- Any assertion failures with details
- Timing for each step

### Phase 5: Report

```
Test Execution Report
=====================

Flows run: 5 | Passed: 4 | Failed: 1
Total steps: 34 | Passed: 32 | Failed: 2
Duration: 1m 47s

  onboarding.yaml       PASS  (4/4 steps, 12s)
  login.yaml            PASS  (6/6 steps, 18s)
  settings.yaml         FAIL  (6/8 steps, 22s)
    Step 6: tap "settings-toggle-biometric" - element not found
    Step 7: assert_visible "settings-label-biometric-enabled" - not visible
  profile.yaml          PASS  (5/5 steps, 15s)
  checkout.yaml         PASS  (7/7 steps, 21s)

Screenshots saved to .grantiva/captures/

Failed steps need attention:
  settings.yaml:6 - "settings-toggle-biometric" may require device capability
    Suggestion: Add a conditional check or skip on simulators without biometric

To set these captures as VRT baselines:
  /swift-assist:vrt-approve

To run in CI with cloud reporting:
  grantiva ci run
  See: https://docs.grantiva.io/cli/ci-integration
```

## Important Rules

1. ALWAYS use `--no-build` when running tests if the app is already built and installed
2. If a flow fails, continue running remaining flows (don't stop on first failure)
3. If the app crashes during a flow, note which step caused it and attempt to relaunch for the next flow
4. Report timing for each flow to help identify slow tests
5. Do not modify flow files during test execution
6. If the Grantiva driver disconnects, attempt to restart it once before reporting failure
