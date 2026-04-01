---
name: run-flow
description: Run a single named flow by name. Faster than /swift-assist:test when you're debugging one broken screen and don't want to wait for the full suite.
argument-hint: "<flow-name> [--approve] [--json]"
---

# Swift Assist: Flow

Run a single named flow from `grantiva.yml` by name - capture its screenshot, compare against the baseline, and report pass or fail. Use this when you're iterating on one screen and don't want to run the entire test suite.

## Usage

```
/swift-assist:run-flow login
/swift-assist:run-flow landmark-detail
/swift-assist:run-flow checkout --approve
/swift-assist:run-flow settings --json
```

## Command Options

- `<flow-name>`: Name of the screen to test (required). Must match a `name:` entry in `grantiva.yml`
- `--approve`: Immediately approve the captured screenshot as the new baseline if the run succeeds
- `--json`: Output result as JSON

## Difference from /swift-assist:test

`/swift-assist:test` runs all flows sequentially - useful for a full suite check, but slow when you're fixing one thing. `/swift-assist:run-flow` runs exactly one screen and reports immediately, making it fast to iterate on a failing screen without waiting for the other 11 to finish.

## Prerequisites

1. App must be running in the simulator
2. Grantiva runner must be active (`grantiva runner start`)
3. `grantiva.yml` must have a `screens:` section with at least one entry
4. A baseline must exist for the named screen (unless using `--approve` to set one)

## What This Command Does

### Phase 1: Resolve the Flow Name

Read `grantiva.yml` to find all defined screens:

```bash
cat grantiva.yml
```

Match the provided `<flow-name>` against the `name:` fields in the `screens:` section. Matching is case-insensitive and tolerates hyphens vs. spaces (e.g., `landmark-detail` matches `Landmark Detail`).

If no match is found, list available screen names and stop:

```
No screen named "landmrk-detail" found in grantiva.yml.

Available screens:
  landmarks
  landmark-detail
  favorites
  settings
  edit-landmark
  caching-demo

Did you mean: landmark-detail?
```

If the input is ambiguous (partial match on multiple names), list the candidates and ask the user to be more specific.

### Phase 2: Capture the Screen

Run Grantiva's capture pipeline targeting only the matched screen. Use the screen's `name` field as reported in `grantiva.yml`:

```bash
grantiva diff capture --screen "<SCREEN_NAME>" --scheme=<SCHEME> --simulator="<SIMULATOR>" --no-build
```

The `--no-build` flag assumes the app is already running. Grantiva will navigate the path defined for that screen (the `path:` steps in `grantiva.yml`) and take a screenshot at the end.

While the capture runs, use computer use to watch the simulator and verify the navigation succeeds.

If the capture fails (navigation error, element not found, app crash), report the failure immediately with the full error and stop:

```
Capture failed: element "login-button-continue" not found

The navigator could not complete the path for screen "login":
  tap: login-textfield-email       -> OK
  type: "test@example.com"         -> OK
  tap: login-button-continue       -> FAILED (element not found)

Check that:
  1. The app is on the expected starting screen (try relaunching)
  2. The accessibility identifier "login-button-continue" is applied to the button
  3. Run /swift-assist:doctor to audit current identifiers
```

### Phase 3: Compare Against Baseline

If `--approve` was NOT specified, run the comparison:

```bash
grantiva diff compare --screen "<SCREEN_NAME>"
```

This compares the newly captured screenshot against the stored baseline in `.grantiva/baselines/`.

If no baseline exists for this screen, stop and prompt:

```
No baseline found for "login".

Run with --approve to capture and set a baseline:
  /swift-assist:run-flow login --approve

Or run /swift-assist:vrt --baseline to set baselines for all screens.
```

### Phase 4: Visual Review with Computer Use

If a diff was detected, use computer use to view both images side by side and describe what changed:

1. Open the baseline: `.grantiva/baselines/<screen-name>.png`
2. Open the capture: `.grantiva/captures/<screen-name>.png`
3. Note specific visual differences (layout shifts, color changes, missing elements, new elements)

### Phase 5: Report Result

**Pass:**

```
Flow: login
Result: PASS

  Diff: 0.3% (threshold: 2%)
  Screenshot: .grantiva/captures/login.png
  Duration: 8s
```

**Fail:**

```
Flow: login
Result: FAIL

  Diff: 12.4% (threshold: 2%)
  Screenshot: .grantiva/captures/login.png
  Baseline:   .grantiva/baselines/login.png
  Duration: 9s

Visual differences detected:
  - "Sign in" button moved down 18px
  - Email field placeholder text changed from "Email" to "Email address"

To approve this as the new baseline:
  /swift-assist:run-flow login --approve

To investigate what changed in source:
  git diff HEAD -- <path-to-login-view-file>
```

### Phase 6: Optional Auto-Approve

If `--approve` was specified and the capture succeeded, approve immediately:

```bash
grantiva diff approve --screen "<SCREEN_NAME>"
```

Confirm:

```
Flow: login
Result: APPROVED

  New baseline saved to .grantiva/baselines/login.png
  Commit this file to update the CI baseline:
    git add .grantiva/baselines/login.png
    git commit -m "Approve login baseline"
```

## Important Rules

1. NEVER auto-approve without the `--approve` flag - always require explicit intent
2. Use `--no-build` unless the user explicitly requests a rebuild (which is slow)
3. If the flow name has a close match but not an exact match, suggest the closest one - do not silently use the wrong screen
4. Report the diff percentage and threshold so the user knows how close to the limit they are
5. If the app is not running or the runner is disconnected, stop and tell the user to run `/swift-assist:init` or `/swift-assist:build`
6. Prefer `/swift-assist:test` in your output suggestions when the user needs to run the full suite
