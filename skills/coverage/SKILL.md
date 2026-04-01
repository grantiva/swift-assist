---
name: coverage
description: Report test coverage across the app - how many screens have flows, how many elements have accessibility identifiers, and what to tackle next. Use for a quick health check on your testing setup.
argument-hint: "[--json] [--detailed]"
---

# Swift Assist: Coverage

Produce a testing coverage dashboard: count screens with flows defined, count interactive elements with accessibility identifiers, cross-reference with the last doctor report, and recommend what to add next.

## Usage

```
/swift-assist:coverage
/swift-assist:coverage --detailed
/swift-assist:coverage --json
```

## Command Options

- `--detailed`: Include per-screen breakdowns in addition to the summary totals
- `--json`: Output the full report as JSON for tooling integration

## Prerequisites

1. `grantiva.yml` must exist with screen definitions
2. A Grantiva runner session should be active for live element counting (falls back to source scan if not)
3. The doctor report at `.grantiva/doctor-report.json` is used if available (run `/swift-assist:doctor` first for the most accurate identifier coverage data)

## What This Command Does

### Phase 1: Count Screens and Flows from grantiva.yml

Read the config:

```bash
cat grantiva.yml
```

Count:
- Total screens defined in `screens:` section
- Screens that have a non-trivial `path:` (more than just `launch`) vs. screens with only `path: launch`

### Phase 2: Count Source Views

Scan Swift source files to find all SwiftUI views:

```bash
grep -rl "struct.*:.*View" --include="*.swift" .
```

For each file, check whether the view appears in `grantiva.yml`. A view "has coverage" if there is at least one screen in `grantiva.yml` whose path navigates to it.

This gives: total SwiftUI views in source vs. views covered by at least one defined screen.

### Phase 3: Count Accessibility Identifiers

**If the doctor report exists** (`.grantiva/doctor-report.json`):

Read it directly - it already contains the element count and identifier coverage from the last doctor run.

**If the doctor report does not exist** but the runner is active:

Run a quick hierarchy dump to count elements:

```bash
grantiva runner dump-hierarchy --format json
```

Count:
- Total interactive elements in the current screen's hierarchy
- Elements with a non-empty `accessibilityIdentifier`

Note that this only reflects the current screen - it is a partial count. Mention this limitation in the report.

**If neither is available:**

Fall back to a source scan:

```bash
grep -rn "\.accessibilityIdentifier" --include="*.swift" .
```

Count `.accessibilityIdentifier(` call sites as a rough proxy for coverage. Note that this undercounts (one call site covers one element) and cannot detect missing identifiers.

### Phase 4: Check Baseline Coverage

Check which screens have approved baselines:

```bash
ls .grantiva/baselines/ 2>/dev/null
```

Count baseline PNG files and match their names against the screens defined in `grantiva.yml`.

### Phase 5: Generate the Dashboard

Print a structured coverage report:

```
Test Coverage Report
====================

Screens
  Defined flows:      8 / 12 screens  (67%)
  Visual baselines:   6 / 12 screens  (50%)
  SwiftUI views:      15 total, 8 covered by flows

Accessibility
  Source scan:        37 / 54 identifiers  (69%)
  (Based on doctor report from 2026-03-28 14:22)

Overall health: PARTIAL

  Screens without flows:
    CategoryList       -> no path defined
    FeaturedBanner     -> no path defined
    ReviewSheet        -> no path defined
    PlanVisitModal     -> no path defined

  Screens without baselines:
    edit-landmark
    plan-visit

  What to tackle next:
    1. Add flows for CategoryList and ReviewSheet (most visible gaps)
    2. Approve baselines for edit-landmark and plan-visit
    3. Add identifiers to remaining 17 elements (run /swift-assist:doctor --fix)
```

Health thresholds:
- **GOOD**: flows >= 80%, identifiers >= 80%, baselines >= 80%
- **PARTIAL**: at least one category is between 50-79%
- **LOW**: any category is below 50%

### Phase 6: Detailed Breakdown (with --detailed)

If `--detailed` is specified, print a per-screen table:

```
Per-Screen Breakdown
====================

Screen               Flow Defined    Baseline    Identifier Coverage
--------------------------------------------------------------------
Landmarks            yes             yes         8/8   (100%)
Favorites            yes             yes         5/5   (100%)
LandmarkDetail       yes             yes         6/9   (67%)  *
EditLandmark         yes             no          3/7   (43%)  *
CategoryList         no              no          0/4   (0%)   *
DeepLinks            yes             yes         4/4   (100%)
CachingDemo          yes             yes         3/3   (100%)
GrantivaConfig       yes             no          2/4   (50%)  *
Feedback             yes             yes         4/4   (100%)
PlanVisitModal       no              no          0/3   (0%)   *
FeaturedBanner       no              no          0/6   (0%)   *
ReviewSheet          no              no          0/5   (0%)   *

* = attention needed
```

### Phase 7: Actionable Suggestions

Always end with a ranked list of next steps, ordered by impact:

```
Recommended next steps:
  High impact:
    /swift-assist:make-tests --flow=CategoryList
    /swift-assist:make-tests --flow=ReviewSheet
    /swift-assist:doctor --fix   (17 elements missing identifiers)

  Medium impact:
    /swift-assist:vrt --baseline  (approve baselines for edit-landmark, plan-visit)
    /swift-assist:run-flow GrantivaConfig --approve

  Already good:
    Landmarks, Favorites, DeepLinks, CachingDemo, Feedback - fully covered
```

## Important Rules

1. NEVER modify source code or `grantiva.yml` - this skill is read-only
2. If the doctor report is stale (older than 7 days), note the date and suggest re-running `/swift-assist:doctor`
3. If the runner is not active and no doctor report exists, use source scanning as a fallback but clearly state it is an estimate
4. Do not count `#Preview` views or `_` prefixed views (internal helpers) in the SwiftUI view total
5. Sort the "what to tackle next" suggestions by estimated impact - screens with zero coverage first, then partial coverage
6. If coverage is already above 80% across all categories, celebrate it and suggest running `/swift-assist:setup-ci` to lock it in with automated PR checks
