argument-hint: [--report=<path>] [--dry-run] [--screen=<ViewName>]
description: Apply accessibility identifier fixes from a doctor report to your Swift source files. Adds .accessibilityIdentifier() modifiers to interactive elements. Use after running /swift-assist:doctor. Triggers on "fix accessibility", "add identifiers", "apply doctor".
---

# Swift Assist: Fix

Read the doctor report and apply `.accessibilityIdentifier()` modifiers to Swift source files for every interactive element that is missing one.

## Usage

```
/swift-assist:fix
/swift-assist:fix --dry-run
/swift-assist:fix --screen=LoginView
/swift-assist:fix --report=.grantiva/doctor-report.json
```

## Command Options

- `--report=<path>`: Path to doctor report JSON (defaults to `.grantiva/doctor-report.json`)
- `--dry-run`: Show what changes would be made without modifying files
- `--screen=<ViewName>`: Only fix elements in a specific view

## Prerequisites

A doctor report must exist. If `.grantiva/doctor-report.json` is not found:
```
No doctor report found. Run /swift-assist:doctor first to scan your app.
```

## What This Command Does

### Phase 1: Load Report

1. Read the doctor report from `.grantiva/doctor-report.json` (or `--report` path)
2. Filter findings to only elements where `has_identifier` is `false`
3. If `--screen` is specified, filter to only that view
4. Group findings by file path

### Phase 2: Apply Fixes

For each finding, edit the Swift source file to add `.accessibilityIdentifier()`.

#### SwiftUI Element Patterns

**Button:**
```swift
// Before
Button("Continue") {
    login()
}

// After
Button("Continue") {
    login()
}
.accessibilityIdentifier("login-button-continue")
```

**TextField:**
```swift
// Before
TextField("Email", text: $email)

// After
TextField("Email", text: $email)
    .accessibilityIdentifier("login-textfield-email")
```

**SecureField:**
```swift
// Before
SecureField("Password", text: $password)

// After
SecureField("Password", text: $password)
    .accessibilityIdentifier("login-securefield-password")
```

**Toggle:**
```swift
// Before
Toggle("Notifications", isOn: $notifications)

// After
Toggle("Notifications", isOn: $notifications)
    .accessibilityIdentifier("settings-toggle-notifications")
```

**Image (interactive):**
```swift
// Before
Image("hero-banner")
    .resizable()

// After
Image("hero-banner")
    .resizable()
    .accessibilityIdentifier("onboarding-image-hero-banner")
```

**NavigationLink:**
```swift
// Before
NavigationLink("Profile") {
    ProfileView()
}

// After
NavigationLink("Profile") {
    ProfileView()
}
.accessibilityIdentifier("home-link-profile")
```

**TabView tabs:**
```swift
// Before
ProfileView()
    .tabItem {
        Label("Profile", systemImage: "person")
    }

// After
ProfileView()
    .tabItem {
        Label("Profile", systemImage: "person")
    }
    .accessibilityIdentifier("home-tab-profile")
```

#### Placement Rules

1. Add `.accessibilityIdentifier()` as the LAST modifier on the element, after all visual modifiers
2. If the element already has accessibility modifiers (`.accessibilityLabel`, `.accessibilityHint`), place the identifier after those
3. Match the indentation style of the existing code (tabs vs spaces, indent level)
4. Do NOT add identifiers to elements inside `ForEach` unless they can be made unique (append index or data ID)

### Phase 3: Verify Changes

After applying all fixes:

1. Read each modified file to confirm the edits look correct
2. Check that the code still compiles:
   ```bash
   grantiva diff capture --no-build
   ```
   Or if a full rebuild is needed, use `grantiva diff capture` (without `--no-build`).
3. If build fails, immediately revert the last set of changes and report the error

### Phase 4: Summary

```
Accessibility Fix Report
========================

Files modified: 6
Identifiers added: 23

  LoginView.swift         +5 identifiers
  HomeView.swift          +3 identifiers
  SettingsView.swift      +2 identifiers
  ProfileView.swift       +4 identifiers
  OnboardingView.swift    +6 identifiers
  CheckoutView.swift      +3 identifiers

Coverage: 47/47 elements now have identifiers (100%)

Next steps:
  /swift-assist:make-tests - Generate test flows using these identifiers
  /swift-assist:vrt        - Set up visual regression baselines
```

If `--dry-run` was specified, show the same summary but prefix with "DRY RUN - no files were modified" and show the diffs that would be applied.

## Important Rules

1. NEVER change existing accessibility identifiers - only add missing ones
2. NEVER modify any code logic - only add `.accessibilityIdentifier()` modifier calls
3. If a file has unsaved changes (check git status), warn the user before modifying
4. Preserve exact whitespace and formatting of the existing code
5. If the build fails after applying fixes, revert ALL changes to that file and report the error
6. For `ForEach` elements, skip them and note in the report: "Skipped: element inside ForEach requires unique ID strategy"
