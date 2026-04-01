---
name: doctor
description: Walk the running app using computer use, inspect the view hierarchy with Grantiva, and report elements missing accessibility identifiers. Use for accessibility audits or test preparation.
argument-hint: "[--screen=<ViewName>] [--json] [--fix]"
---

# Swift Assist: Doctor

Walk every screen of the running iOS app using computer use, inspect the view hierarchy via `grantiva runner dump-hierarchy`, and produce a report of interactive elements missing `.accessibilityIdentifier()`.

## Usage

```
/swift-assist:doctor
/swift-assist:doctor --screen=LoginView
/swift-assist:doctor --fix
/swift-assist:doctor --json
```

## Command Options

- `--screen=<ViewName>`: Only audit a specific screen instead of the full app
- `--fix`: Automatically run `/swift-assist:add-identifiers` after the audit completes
- `--json`: Output findings as JSON for tooling integration

## Prerequisites

1. The app must be running in a simulator (run `/swift-assist:init` first)
2. The Grantiva runner session must be active:
   ```bash
   grantiva runner start
   ```
3. Computer use must be available for visual navigation

## Naming Convention

All suggested accessibility identifiers follow this pattern:

```
<screen>-<elementType>-<descriptor>
```

Where:
- **screen**: Inferred from the SwiftUI view struct name (lowercased, hyphenated). e.g., `LoginView` becomes `login`
- **elementType**: The UI element type: `button`, `textfield`, `securefield`, `toggle`, `slider`, `picker`, `image`, `tab`, `link`, `label`
- **descriptor**: A short, descriptive name for what the element does, derived from the element's label, text, or purpose

Examples:
```
login-button-continue
login-textfield-email
login-securefield-password
onboarding-image-hero-banner
settings-toggle-notifications
deeplinks-link-caching-demo
edit-landmark-button-done
grantiva-textfield-user-id
```

## What This Command Does

### Phase 1: Source Code Pre-Scan

Before walking the app, scan the Swift source to build a map of views and their elements:

1. Find all SwiftUI view files:
   ```bash
   grep -rl "struct.*:.*View" --include="*.swift" .
   ```

2. For each view file, catalog interactive elements:
   - `Button`, `TextField`, `SecureField`, `Toggle`, `Slider`, `Picker`, `DatePicker`
   - `NavigationLink`, `Image` (when interactive), `.onTapGesture`
   - `TabView` tabs, `Link`

3. Note which elements already have `.accessibilityIdentifier()` applied

4. Build a lookup table: `ViewName -> [element, hasIdentifier, suggestedId]`

### Phase 2: Walk the App with Computer Use

**This phase requires computer use (MCP browser tools targeting the simulator).**

#### Navigation Strategy

1. **Take a screenshot** of the current screen
2. **Dump the hierarchy** - run `grantiva runner dump-hierarchy` to get the live accessibility tree
3. **Identify the current view** by matching visible elements to the source code scan
4. **Record findings** - compare live hierarchy against source scan, noting:
   - Elements with no accessibility identifier
   - Elements with generic/unhelpful identifiers
   - Elements that are interactive but not accessible
5. **Navigate to the next screen**:
   - If TabView exists, visit each tab
   - Follow NavigationLinks and back-navigate
   - Tap buttons that lead to new screens (sheets, modals, pushes)
   - If `--screen` was specified, navigate only to that screen

#### Important: Tab Identifiers

Tab bar buttons in SwiftUI do NOT receive the `.accessibilityIdentifier()` you set on the tab content. The hierarchy shows tab buttons as:

```
[XCUIElementTypeButton] name="map" label="Landmarks"
[XCUIElementTypeButton] name="heart" label="Favorites"
```

The `name` is the SF Symbol, the `label` is the display text. Test flows should tap tabs by their **label text** (e.g., `tap: "Favorites"`), not by a custom identifier. Still add identifiers to tab content for code organization, but note in the report that tabs are tapped by label.

#### For Each Screen

1. Screenshot the screen
2. Run `grantiva runner dump-hierarchy` to get the element tree
3. Use `grantiva runner dump-hierarchy --format json` for programmatic parsing
4. Cross-reference with source code to identify the SwiftUI view struct
5. For each interactive element without an identifier:
   - Determine the `screen` from the view struct name
   - Determine the `elementType` from the UI component
   - Determine the `descriptor` from the element's label, text content, or SF Symbol name
   - Generate the suggested identifier

### Phase 3: Generate Report

Produce a structured report:

```
Accessibility Doctor Report
============================

Scanned: 11 views, 54 interactive elements
Missing identifiers: 54

ContentView (5 missing)
  - Tab "Landmarks"           -> content-tab-landmarks (tap by label)
  - Tab "Favorites"           -> content-tab-favorites (tap by label)
  - Tab "Grantiva"            -> content-tab-grantiva (tap by label)
  - Tab "Feedback"            -> content-tab-feedback (tap by label)
  - Tab "Deep Links"          -> content-tab-deeplinks (tap by label)

LandmarkDetailView (3 missing)
  - NavigationLink (category) -> landmark-detail-link-category
  - NavigationLink "Plan Visit" -> landmark-detail-link-plan-visit
  - Button (favorite)         -> landmark-detail-button-favorite

EditLandmarkView (4 missing)
  - TextField "Name"          -> edit-landmark-textfield-name
  - TextField "Location"      -> edit-landmark-textfield-location
  - Button "Cancel"           -> edit-landmark-button-cancel
  - Button "Done"             -> edit-landmark-button-done

Coverage: 0/54 elements have identifiers (0%)
```

### Phase 4: Save Findings

Save the report to `.grantiva/doctor-report.json` so that `/swift-assist:add-identifiers` can consume it.

### Phase 5: Optional Auto-Fix

If `--fix` was specified, immediately invoke `/swift-assist:add-identifiers` with the saved report.

## Important Rules

1. NEVER modify source code in this command (unless `--fix` is specified, which delegates to `/swift-assist:add-identifiers`)
2. Always take screenshots as evidence of what was found on each screen
3. If you cannot reach a screen (e.g., requires login or API data), note it as "unreachable" and suggest adding mock services
4. Do not suggest identifiers for non-interactive decorative elements
5. If an element already has a good identifier, do not suggest replacing it
6. The `grantiva runner dump-hierarchy` output is ground truth - if source scan and live hierarchy disagree, trust the live hierarchy
7. Stop walking after visiting all reachable screens - do not loop infinitely
