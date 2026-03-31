---
name: vrt-approve
description: Approve current VRT captures as new baselines. Can approve all or specific screens selectively. Use after reviewing /swift-assist:vrt results.
argument-hint: [SCREEN_NAME ...]
---

# Swift Assist: VRT Approve

Approve current visual regression captures as the new baselines, either for all screens or selectively.

## Usage

```
/swift-assist:vrt-approve
/swift-assist:vrt-approve settings checkout
/swift-assist:vrt-approve login-filled
```

## Arguments

- `[SCREEN_NAME ...]`: Optional list of specific screen names to approve. If omitted, approves all screens.

## What This Command Does

### Phase 1: Show What Will Be Approved

Before approving, show the user exactly what's changing:

```
Screens to approve:

  settings       (8.4% diff from current baseline)
  checkout       (4.1% diff from current baseline)

This will replace the stored baselines with the current captures.
Proceed? (y/n)
```

### Phase 2: Approve

Run the Grantiva approve command:

**All screens:**
```bash
grantiva diff approve
```

**Selective screens:**
```bash
grantiva diff approve settings checkout
```

### Phase 3: Confirm

```
Baselines updated.

  settings       -> new baseline saved
  checkout       -> new baseline saved

Future /swift-assist:vrt runs will compare against these new baselines.
```

## Important Rules

1. ALWAYS show what will be approved and ask for confirmation before running
2. Never approve automatically without user consent
3. If no captures exist, tell the user to run `/swift-assist:vrt` first
