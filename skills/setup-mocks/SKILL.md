---
name: setup-mocks
description: Add the UI_TESTING / .mock services pattern to the app entry point so Grantiva can run tests with deterministic data and no live server. Use before make-tests when the app talks to an API.
argument-hint: "[--entry=<file>]"
---

# Swift Assist: Setup Mocks

Add closure-based mock services to the app entry point behind a `#if UI_TESTING` compilation flag, and configure `grantiva.yml` to build with that flag active. This gives Grantiva a fully deterministic app - no running server, no flaky network calls, no authentication required.

## Usage

```
/swift-assist:setup-mocks
/swift-assist:setup-mocks --entry=Sources/MyApp/MyApp.swift
```

## Command Options

- `--entry=<file>`: Path to the app entry point file containing `@main` (auto-detected if omitted)

## Why This Matters

Tests that hit a real server are slow, flaky, and require credentials. When you build with `UI_TESTING`, the app swaps every live service for an in-memory mock that returns canned data instantly. Every test run sees the exact same state, which means screenshot baselines are stable and CI never fails because the staging environment is down.

The pattern uses closure-based dependency injection - no protocols, no generics. Live services capture real API clients. Mock services return hardcoded fixtures from a local store.

## Prerequisites

1. The app must have service types that use closures for their operations (not protocol conformance)
2. `grantiva.yml` must exist (run `/swift-assist:init` first)
3. The app entry point (`@main` struct) must be identified

## What This Command Does

### Phase 1: Find the App Entry Point

If `--entry` is not specified, search for the file with `@main`:

```bash
grep -rl "@main" --include="*.swift" .
```

Read the file and identify:
- Where services are initialized (usually in `init()` or as stored properties)
- The service type names (e.g., `DataService`, `AuthService`, `ImageService`)
- How services are passed to the environment (`.environmentObject()`, direct init args, etc.)

### Phase 2: Understand the Service Pattern

For each service, read its definition to understand:

1. What closures it exposes (e.g., `var fetchLandmarks: () async throws -> [Landmark]`)
2. Whether a `.live` static var already exists
3. Whether a `.mock` or `.preview` static var already exists

If `.mock` variants already exist for all services, tell the user and skip to Phase 5 (just update `grantiva.yml`).

### Phase 3: Generate Mock Services

For each service that lacks a `.mock` variant, generate one. The mock uses an in-memory store and returns data immediately without any async work:

```swift
// In DataService.swift (or a new DataService+Mock.swift file)
extension DataService {
    static let mock: Self = {
        var landmarks = Landmark.previewData  // use existing preview data if available

        return DataService(
            fetchLandmarks: {
                return landmarks
            },
            fetchLandmark: { id in
                guard let landmark = landmarks.first(where: { $0.id == id }) else {
                    throw DataServiceError.notFound
                }
                return landmark
            },
            updateLandmark: { updated in
                if let index = landmarks.firstIndex(where: { $0.id == updated.id }) {
                    landmarks[index] = updated
                }
            }
        )
    }()
}
```

Key rules for mock implementations:
- Use `Landmark.previewData` or similar if the app already has preview fixtures
- If no fixtures exist, generate a small realistic dataset (3-5 items) based on the model definition
- Operations that mutate state should update the local array so downstream assertions work
- Never call `URLSession`, `CoreData`, or any external dependency
- Keep it simple - mocks don't need error simulation unless the skill is called with `--error-cases`

### Phase 4: Add the #if UI_TESTING Block

In the app entry point, wrap the service initialization in a conditional:

**Before:**
```swift
@main
struct LandmarksApp: App {
    let dataService = DataService.live
    let authService = AuthService.live

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(dataService)
                .environmentObject(authService)
        }
    }
}
```

**After:**
```swift
@main
struct LandmarksApp: App {
    #if UI_TESTING
    let dataService = DataService.mock
    let authService = AuthService.mock
    #else
    let dataService = DataService.live
    let authService = AuthService.live
    #endif

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(dataService)
                .environmentObject(authService)
        }
    }
}
```

If services are initialized in a method or computed property rather than stored properties, adapt the pattern to fit the actual structure - do not force it into stored properties.

### Phase 5: Update grantiva.yml

Add `build_settings` to activate the `UI_TESTING` flag at build time:

```bash
cat grantiva.yml
```

If `build_settings` already exists, add to it. If not, add a new section:

```yaml
build_settings:
  SWIFT_ACTIVE_COMPILATION_CONDITIONS: DEBUG UI_TESTING
```

This tells `grantiva build` to pass `-DDEBUG -DUI_TESTING` to the Swift compiler, activating the `#if UI_TESTING` blocks.

The full updated `grantiva.yml` should look something like:

```yaml
scheme: Landmarks
simulator: iPhone 16 Pro
bundle_id: com.example.landmarks

build_settings:
  SWIFT_ACTIVE_COMPILATION_CONDITIONS: DEBUG UI_TESTING

screens:
  - name: Landmarks
    path: launch
```

### Phase 6: Verify the Build

After making changes, rebuild to confirm the mock build compiles:

```bash
grantiva build
```

If the build fails, show the full error output and stop. Common issues:
- Missing fixtures referenced in the mock (suggest creating a `PreviewData.swift` file)
- Services that require mandatory arguments not present in the mock

### Phase 7: Confirm

```
Mock services configured.

  Entry point:    Sources/Landmarks/LandmarksApp.swift
  Services mocked:
    DataService   -> DataService.mock  (in-memory store, 5 landmarks)
    AuthService   -> AuthService.mock  (always authenticated as "Test User")

  grantiva.yml updated:
    build_settings:
      SWIFT_ACTIVE_COMPILATION_CONDITIONS: DEBUG UI_TESTING

Build succeeded with mock services active.

Next steps:
  /swift-assist:make-tests  - Generate flows (app now has stable test data)
  /swift-assist:test        - Run flows against the mock build
```

## Important Rules

1. NEVER remove live service code - only add mock variants and the conditional
2. If `.mock` or `.preview` variants already exist, reuse them - do not create duplicates
3. Mock data must be realistic enough that UI elements render properly (non-empty strings, valid IDs, plausible values)
4. If a service is too complex to mock safely, note it and skip it - a partial mock is better than breaking the build
5. Always rebuild after making changes to confirm the mock build compiles
6. Do not add `UI_TESTING` to the non-mock build path - it must only apply inside `#if UI_TESTING`
