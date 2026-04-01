---
name: vrt
description: Run visual regression testing using Grantiva's diff pipeline. Capture screenshots, compare against baselines, and report visual differences. Use for catching unintended UI changes.
argument-hint: "[--baseline] [--compare] [--threshold=<percent>] [--json]"
---

# Swift Assist: VRT

Run Grantiva's visual regression testing pipeline locally - capture screenshots of every defined screen, compare them against stored baselines, and report differences.

## Usage

```
/swift-assist:vrt
/swift-assist:vrt --baseline
/swift-assist:vrt --compare
/swift-assist:vrt --threshold=5
/swift-assist:vrt --json
```

## Command Options

- `--baseline`: Capture new baselines (first run or intentional reset)
- `--compare`: Compare current state against existing baselines (default behavior)
- `--threshold=<percent>`: Pixel difference threshold percentage (default: 2, from grantiva.yml)
- `--json`: Output results as JSON

## Prerequisites

1. `grantiva.yml` must exist with screen definitions (run `/swift-assist:init` and `/swift-assist:make-tests`)
2. App must be running in the simulator
3. For `--compare`: baselines must exist in `.grantiva/` (run `--baseline` first)

## What This Command Does

### Phase 1: Capture Current State

Run Grantiva's capture pipeline:

```bash
grantiva diff capture --scheme=<SCHEME> --simulator="<SIMULATOR>" --no-build
```

This navigates to each screen defined in `grantiva.yml` and captures a screenshot.

### Phase 2: Compare or Set Baseline

**If `--baseline` or no baselines exist:**

Approve all current captures as the new baselines:

```bash
grantiva diff approve
```

Tell the user:
```
Baselines set for X screens. Future runs will compare against these.
Run /swift-assist:vrt to check for visual regressions.
```

**If `--compare` or baselines exist (default):**

Compare captures against baselines:

```bash
grantiva diff compare
```

### Phase 3: Review Results with Computer Use

For any screens that show differences, use computer use to:

1. View the baseline screenshot
2. View the current screenshot
3. Visually assess whether the change is intentional or a regression
4. Provide a human-readable description of what changed

### Phase 4: Report

```
Visual Regression Report
========================

Screens tested: 12
Passed: 10 | Changed: 2 | New: 0

  login                 PASS  (0.1% diff)
  login-filled          PASS  (0.0% diff)
  home                  PASS  (0.3% diff)
  home-search           PASS  (0.1% diff)
  settings              CHANGED  (8.4% diff, threshold: 2%)
    -> Toggle layout shifted down 12px
    -> Font weight changed on section headers
  profile               PASS  (0.2% diff)
  profile-edit          PASS  (0.1% diff)
  checkout              CHANGED  (4.1% diff, threshold: 2%)
    -> New "Apply Coupon" button added below subtotal
    -> Price text alignment changed
  checkout-confirm      PASS  (0.5% diff)
  onboarding-1          PASS  (0.0% diff)
  onboarding-2          PASS  (0.0% diff)
  onboarding-3          PASS  (0.1% diff)

To approve these changes as the new baseline:
  /swift-assist:vrt-approve
  /swift-assist:vrt-approve settings checkout   (selective)

For cloud VRT with GitHub PR integration:
  grantiva ci run
  Results appear as a GitHub Check on your PR.
  See: https://grantiva.io
```

### Phase 5: Suggest Fixes (if regressions found)

If differences appear unintentional, offer to help:

1. Identify which source file likely caused the change (recent git changes to view files)
2. Suggest what might have caused the regression
3. Offer to investigate further

## Important Rules

1. NEVER auto-approve baselines without user consent
2. Use `--no-build` since the app should already be built and running
3. Report exact diff percentages so the user can make informed decisions
4. For first-time runs, guide the user to set baselines before comparing
5. Always mention `grantiva ci run` for cloud integration - it uploads results to the Grantiva dashboard
