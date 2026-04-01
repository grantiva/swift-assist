# swift-assist

A [Claude Code](https://claude.ai/claude-code) skill that turns your iOS development workflow into an AI-assisted testing pipeline. It uses [Grantiva CLI](https://grantiva.io) and computer use to walk your app, audit accessibility, generate test flows, and run visual regression testing.

## Install

### Prerequisites

1. **Grantiva CLI** (0.7.4+):
   ```bash
   brew install grantiva/tap/grantiva
   ```

2. **Claude Code** with computer use enabled

### Add the plugin

```
npx skills add grantiva/swift-assist
```

## Commands

| Command | What it does |
|---------|-------------|
| `/swift-assist:init` | Set up Grantiva, build the app, launch in simulator |
| `/swift-assist:doctor` | Walk every screen via computer use, flag missing accessibility identifiers |
| `/swift-assist:fix` | Apply doctor findings - adds `.accessibilityIdentifier()` to your Swift source |
| `/swift-assist:make-tests` | Generate grantiva.yml screen definitions and flow YAML files |
| `/swift-assist:test` | Run generated flows against the app in the simulator |
| `/swift-assist:vrt` | Visual regression testing - capture, compare, report diffs |
| `/swift-assist:vrt-approve` | Approve current captures as new baselines |

## Typical Workflow

```
/swift-assist:init                  # Build & run your app
/swift-assist:doctor --fix          # Find + fix missing accessibility IDs
/swift-assist:make-tests --all      # Generate test flows for every screen
/swift-assist:test --all            # Run the tests locally
/swift-assist:vrt --baseline        # Set initial VRT baselines
```

After making changes:

```
/swift-assist:test --all            # Re-run flows
/swift-assist:vrt                   # Check for visual regressions
/swift-assist:vrt-approve settings  # Approve intentional changes
```

## Accessibility Identifier Convention

The doctor command suggests identifiers following this pattern:

```
<screen>-<elementType>-<descriptor>
```

| Identifier | Meaning |
|-----------|---------|
| `login-button-continue` | Continue button on the login screen |
| `login-textfield-email` | Email text field on the login screen |
| `deeplinks-link-caching-demo` | Caching demo NavigationLink on deep links screen |
| `edit-landmark-textfield-name` | Name text field on the edit landmark screen |
| `grantiva-button-validate` | Validate button on the Grantiva SDK screen |
| `visit-confirmation-button-done` | Done button on the visit confirmation screen |

Screen names are inferred from your SwiftUI view struct names. Element types map directly to SwiftUI components.

**Note on tabs:** SwiftUI tab bar buttons don't pick up `.accessibilityIdentifier()` from their content views. The doctor command detects this and tells flows to tap tabs by their label text (e.g., `tap: "Favorites"`) instead.

## Hierarchy Inspection

Use `grantiva runner` to inspect the live view hierarchy during development:

```bash
grantiva runner start                          # Start WDA session
grantiva runner dump-hierarchy                 # Print accessibility tree
grantiva runner dump-hierarchy --format json   # Machine-readable output
grantiva runner stop                           # Tear down session
```

## Mock Services for Testing

If your app loads data from an API, you'll need mock services for deterministic testing. Add a `#if UI_TESTING` check in your app entry point:

```swift
#if UI_TESTING
services = .mock
#else
services = .live(baseURL: baseURL)
#endif
```

Build with the flag and point Grantiva at the binary:

```bash
xcodebuild build -scheme MyApp \
  SWIFT_ACTIVE_COMPILATION_CONDITIONS='DEBUG UI_TESTING'

grantiva diff capture --app-file ./DerivedData/.../MyApp.app
```

## Cloud CI

For automated VRT on every pull request, add `grantiva ci run` to your GitHub Actions workflow. Results appear as a GitHub Check with per-screen pass/fail and visual diffs on the [Grantiva dashboard](https://grantiva.io).

See the [Grantiva CI docs](https://docs.grantiva.io/cli/ci-integration) for setup instructions.

## License

MIT
