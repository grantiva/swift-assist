---
name: make-tests
description: Walk the app using computer use and generate Grantiva screen definitions and flow YAML files for every discovered user flow. Use after /swift-assist:doctor for reliable tests.
argument-hint: "[--flow=<name>] [--all] [--output=<dir>]"
---

# Swift Assist: Make Tests

Walk the running app with computer use, discover user flows, and generate grantiva.yml screen definitions plus standalone flow YAML files that can be run with `grantiva ci run` or locally with `grantiva diff capture`.

## Usage

```
/swift-assist:make-tests
/swift-assist:make-tests --flow=onboarding
/swift-assist:make-tests --flow=login
/swift-assist:make-tests --all
/swift-assist:make-tests --output=flows/
```

## Command Options

- `--flow=<name>`: Generate tests for a specific flow (e.g., onboarding, login, checkout, settings)
- `--all`: Discover and generate tests for all reachable flows
- `--output=<dir>`: Output directory for flow YAML files (defaults to `flows/`)

## Prerequisites

1. App must be running in the simulator
2. Grantiva runner session must be active (`grantiva runner start`)
3. Ideally, run `/swift-assist:doctor` and `/swift-assist:add-identifiers` first so elements have accessibility identifiers

If no doctor report exists, warn:
```
No doctor report found. Tests will be more reliable with accessibility
identifiers. Run /swift-assist:doctor --fix first, or continue without them.
Continue anyway? (y/n)
```

## What This Command Does

### Phase 1: Discover App Structure

Using computer use, walk the app to map its structure:

1. **Launch the app** and screenshot the initial screen
2. **Identify navigation patterns**:
   - TabView? Record each tab name and what screen it leads to
   - NavigationStack? Record the root and all push destinations
   - Modal presentations? Record triggers and destinations
3. **Build a screen graph**: Each screen is a node, each navigation action is an edge

Use the Grantiva driver to inspect the hierarchy at each screen:
```bash
# The driver exposes the accessibility tree - use it to identify tappable elements
```

### Phase 2: Define User Flows

Group the discovered screens into logical user flows:

| Flow Name | Description | Screens |
|-----------|-------------|---------|
| `onboarding` | First-launch experience | Welcome -> Features -> Permissions -> Home |
| `login` | Authentication flow | Login -> Home |
| `signup` | New account creation | Signup -> Verify -> Home |
| `settings` | Settings navigation | Home -> Settings -> (each setting) -> Back |
| `profile` | Profile management | Home -> Profile -> Edit -> Save |
| `checkout` | Purchase flow | Cart -> Payment -> Confirmation |

If `--flow` is specified, only generate that flow. Otherwise:
- If `--all`, generate all discovered flows
- If neither, show the discovered flows and ask the user which to generate

### Phase 3: Generate grantiva.yml Screen Definitions

For each screen in the selected flows, add a screen entry to `grantiva.yml`:

```yaml
screens:
  - name: Landmarks
    path: launch

  - name: Favorites
    path:
      - tap: "Favorites"

  - name: DeepLinks
    path:
      - tap: "Deep Links"

  - name: CachingDemo
    path:
      - tap: "deeplinks-link-caching-demo"
      - wait: 1

  - name: LandmarkDetail
    path:
      - tap: "Landmarks"
      - wait: 5
      - tap: "Golden Gate Bridge"
      - wait: 1
```

Note the mix: tabs are tapped by label text (`"Favorites"`), inner elements by accessibility identifier (`"deeplinks-link-caching-demo"`), and list items by visible content (`"Golden Gate Bridge"`). Use whatever the hierarchy actually exposes for each element.

### Phase 4: Generate Flow YAML Files

Create standalone Maestro-compatible flow files in the output directory:

**flows/browse-landmarks.yaml:**
```yaml
appId: com.example.myapp
---
- launchApp
- assertVisible: "Landmarks"
- assertVisible: "Featured"
- takeScreenshot: "Landmarks Home"
- tapOn: "Golden Gate Bridge"
- assertVisible: "Plan Visit"
- takeScreenshot: "Landmark Detail"
```

**flows/plan-visit.yaml:**
```yaml
appId: com.example.myapp
---
- launchApp
- assertVisible: "Landmarks"
- tapOn: "Golden Gate Bridge"
- tapOn: "landmark-detail-link-plan-visit"
- assertVisible: "Visit Scheduled!"
- takeScreenshot: "Visit Confirmation"
- tapOn: "visit-confirmation-button-done"
- assertVisible: "Landmarks"
```

**flows/deep-links.yaml:**
```yaml
appId: com.example.myapp
---
- launchApp
- tapOn: "Deep Links"
- assertVisible: "Deep Links"
- tapOn: "deeplinks-link-caching-demo"
- assertVisible: "Caching Demo"
- takeScreenshot: "Caching Demo Screen"
```

### Phase 5: Handle Test State (v1)

For apps that load data from an API, the recommended approach is mock services with a `UI_TESTING` build flag:

1. Check if the app has `.mock` or `.preview` service variants
2. If yes, tell the user to build with `SWIFT_ACTIVE_COMPILATION_CONDITIONS='DEBUG UI_TESTING'`
3. Use `grantiva diff capture --app-file <path-to-built-app>` to test with the mock binary
4. If no mock services exist, suggest adding them and explain the pattern

For flows that require specific app state (logged in, onboarded, etc.), use `run_flow` to reference shared setup:

```yaml
appId: com.example.myapp
---
- launchApp
- tapOn: "Deep Links"
- tapOn: "deeplinks-link-caching-demo"
- assertVisible: "Caching Demo"
- takeScreenshot: "Caching Demo"
```

Note to user:
```
Test state management (v1): Flows use a shared setup file for authentication.
For more robust state management (mock servers, seed data, launch arguments),
this will be expanded in a future version.
```

### Phase 6: Validate Generated Flows

For each generated flow, do a quick sanity check:

1. Verify all referenced accessibility identifiers exist in the doctor report
2. Check that `run_flow` references point to files that exist
3. Warn about any identifiers used in flows that weren't found in the scan

### Phase 7: Summary

```
Test Generation Report
======================

Flows generated: 5
Screen definitions added to grantiva.yml: 12
Flow YAML files created: 7

  flows/navigate-to-home.yaml        (shared setup)
  flows/_setup-authenticated.yaml     (auth setup)
  flows/onboarding.yaml              (4 steps)
  flows/login.yaml                   (6 steps)
  flows/settings.yaml                (8 steps)
  flows/profile.yaml                 (5 steps)
  flows/checkout.yaml                (7 steps)

Next steps:
  /swift-assist:test  - Run these flows locally
  /swift-assist:vrt   - Capture visual baselines for each screen

For cloud CI integration, push your grantiva.yml and flows/ directory,
then add `grantiva ci run` to your GitHub Actions workflow.
See: https://docs.grantiva.io/cli/ci-integration
```

## Important Rules

1. ALWAYS use accessibility identifiers for `tap` actions when available - never use coordinates
2. If an element has no identifier, note it in a comment: `# TODO: needs accessibilityIdentifier`
3. Add `wait` steps after navigation actions to allow transitions to complete
4. Add `assert_visible` after navigation to verify the expected screen loaded
5. Keep flows focused on one user journey - don't combine unrelated flows
6. Use `run_flow` for shared setup (like authentication) instead of duplicating steps
7. Generated flows should be human-readable and easy to modify
