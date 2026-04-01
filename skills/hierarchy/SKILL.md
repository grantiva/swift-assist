---
name: hierarchy
description: Dump and analyze the live accessibility hierarchy for the current simulator screen. Use to debug why a test can't find an element, inspect what identifiers are set, or understand the element tree before writing flows.
argument-hint: "[--screen=<ViewName>] [--filter=<text>] [--json]"
---

# Swift Assist: Hierarchy

Dump the live accessibility hierarchy from the running app and analyze it. Use this when a test can't find an element, when you want to verify that `.accessibilityIdentifier()` modifiers are applied correctly, or when you're planning a new flow and need to know what the hierarchy actually exposes.

## Usage

```
/swift-assist:hierarchy
/swift-assist:hierarchy --screen=LandmarkDetail
/swift-assist:hierarchy --filter=button
/swift-assist:hierarchy --json
```

## Command Options

- `--screen=<ViewName>`: Navigate to a specific screen first before dumping (uses computer use)
- `--filter=<text>`: Filter output to elements whose name, label, or identifier contains this text
- `--json`: Output raw JSON instead of formatted tree

## What This Command Does

### Phase 1: Verify Runner

Check that the Grantiva runner is active:

```bash
grantiva runner start --bundle-id <BUNDLE_ID> --simulator "<SIMULATOR>"
```

Read bundle ID and simulator from `grantiva.yml`. If the runner fails to start, stop and tell the user to run `/swift-assist:init` first.

### Phase 2: Navigate to Screen (if --screen provided)

If `--screen` was provided, use computer use to navigate to that screen before dumping the hierarchy. This may involve tapping tabs, navigation links, or buttons to reach the target view.

### Phase 3: Dump Hierarchy

```bash
grantiva runner dump-hierarchy
```

This queries the live XCUITest element tree from the running app.

### Phase 4: Parse and Display

Format the raw hierarchy into a readable tree. Show element type, label, name (accessibility identifier), and value for each element:

```
[XCUIElementTypeApplication] label="MyApp"
  [XCUIElementTypeWindow]
    [XCUIElementTypeTabBar] label="Tab Bar"
      [XCUIElementTypeButton] name="home-tab" label="Home"
      [XCUIElementTypeButton] name="settings-tab" label="Settings"
    [XCUIElementTypeScrollView]
      [XCUIElementTypeStaticText] label="Welcome"
      [XCUIElementTypeButton] name="get-started-button" label="Get Started"
      [XCUIElementTypeButton] label="Sign In"  ← no identifier
```

Mark elements without an accessibility identifier with `← no identifier`.

If `--filter` was provided, show only matching elements and their ancestors for context.

### Phase 5: Summarize

After the tree, print a summary:

```
Summary
=======
Total interactive elements:  12
With identifiers:             9  (75%)
Without identifiers:          3

Elements without identifiers:
  [XCUIElementTypeButton] label="Sign In"
  [XCUIElementTypeTextField] label="Email"
  [XCUIElementTypeSwitch] label="Remember Me"

Run /swift-assist:doctor --fix to add missing identifiers.
```

### Phase 6: Highlight Testability Issues

Call out elements that tests commonly have trouble with:

- Tab bar buttons: explain that identifiers go on content, not the tab itself
- Navigation bar buttons: often use `label` not `name` for tapping
- Cells in lists: may need identifiers on inner elements, not the cell container

## Output Modes

**Default (formatted tree):** Best for human reading and debugging.

**`--json`:** Outputs the raw JSON from `grantiva runner dump-hierarchy` for piping or programmatic use.

## Important Rules

1. Always show the full tree - do not truncate unless the user asks
2. Clearly mark elements missing identifiers
3. If `--screen` navigation fails, dump the hierarchy of wherever the app currently is and note that navigation didn't reach the target
4. Do not modify source code
