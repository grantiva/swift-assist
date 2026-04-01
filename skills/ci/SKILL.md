---
name: ci
description: Generate a GitHub Actions workflow that runs Grantiva VRT in CI on every pull request. Use to add cloud visual regression testing with GitHub Check integration to your project.
argument-hint: "[--output=<path>]"
---

# Swift Assist: CI

Generate a GitHub Actions workflow file that runs Grantiva's visual regression tests on every pull request, uploads results to the Grantiva cloud dashboard, and posts a GitHub Check directly on the PR.

## Usage

```
/swift-assist:ci
/swift-assist:ci --output=.github/workflows/vrt.yml
```

## Command Options

- `--output=<path>`: Where to write the workflow file (default: `.github/workflows/grantiva-vrt.yml`)

## Prerequisites

1. `grantiva.yml` must exist with screen definitions (run `/swift-assist:make-tests` first)
2. A Grantiva account and API key from https://grantiva.io
3. The GitHub repo must have the `GRANTIVA_API_KEY` secret set

## What This Command Does

### Phase 1: Read Config

Load project details from `grantiva.yml`:

```bash
cat grantiva.yml
```

Extract `scheme`, `simulator`, and `bundle_id` so the workflow targets the right build.

### Phase 2: Check for Existing Workflow

```bash
ls .github/workflows/
```

If a Grantiva workflow already exists, show the current file and ask whether to overwrite or update it.

### Phase 3: Generate the Workflow File

Create `.github/workflows/grantiva-vrt.yml` (or the path from `--output`):

```yaml
name: Visual Regression Tests

on:
  pull_request:
    branches: [main]

jobs:
  vrt:
    name: Grantiva VRT
    runs-on: macos-15

    environment: grantiva

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Grantiva CLI
        run: brew install grantiva-io/tap/grantiva

      - name: Run VRT
        env:
          GRANTIVA_API_KEY: ${{ secrets.GRANTIVA_API_KEY }}
        run: grantiva ci run

      - name: Upload diff artifacts on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: grantiva-diffs
          path: .grantiva/diffs/
          retention-days: 7
```

### Phase 4: Explain the Setup

Tell the user what `grantiva ci run` does end-to-end:

1. Boots the configured simulator
2. Builds and installs the app
3. Navigates to each screen defined in `grantiva.yml` and captures a screenshot
4. Compares captures against the baselines committed in the repo (`.grantiva/baselines/`)
5. Uploads results to the Grantiva dashboard and posts a GitHub Check on the PR

The GitHub Check appears as **"Grantiva VRT"** in the PR checks list. If any screen exceeds the configured diff threshold, the check fails and blocks merge (if branch protection is enabled).

### Phase 5: Add the GitHub Secret

Print clear instructions for adding the API key:

```
To finish setup, add your Grantiva API key as a GitHub secret:

  1. Go to https://github.com/<owner>/<repo>/settings/secrets/actions
  2. Click "New repository secret"
  3. Name: GRANTIVA_API_KEY
  4. Value: your key from https://grantiva.io/settings/api-keys

Or use the GitHub CLI:
  gh secret set GRANTIVA_API_KEY --body "grv_..."
```

### Phase 6: Explain Baselines in the Repo

Baselines are stored in `.grantiva/baselines/` and committed to the repo. This means:

- The CI run always compares against the last approved baseline on the default branch
- When you approve changes locally with `/swift-assist:vrt-approve`, commit and push `.grantiva/baselines/` so CI picks them up
- Do NOT add `.grantiva/baselines/` to `.gitignore` - they need to be in source control

If `.gitignore` currently excludes `.grantiva/`, print a warning and offer to fix it:

```bash
grep -n "\.grantiva" .gitignore
```

If found, suggest removing the line or scoping it to only exclude `.grantiva/captures/` and `.grantiva/diffs/`.

### Phase 7: Confirm

```
CI workflow created.

  File:      .github/workflows/grantiva-vrt.yml
  Trigger:   pull_request -> main
  Command:   grantiva ci run
  Secret:    GRANTIVA_API_KEY (add to GitHub repo secrets)

Commit the workflow and baselines to activate:
  git add .github/workflows/grantiva-vrt.yml .grantiva/baselines/
  git commit -m "Add Grantiva VRT to CI"
  git push

After pushing, open a PR and the Grantiva VRT check will appear automatically.
See results at: https://grantiva.io/dashboard
```

## Important Rules

1. NEVER commit or display the actual API key value - only reference `${{ secrets.GRANTIVA_API_KEY }}`
2. Always use `macos-15` or later for the runner - earlier versions may lack required simulator runtimes
3. If `.grantiva/baselines/` is gitignored, warn the user - CI cannot run without committed baselines
4. Do not modify `grantiva.yml` - this skill only generates the workflow file
5. If the user specifies `--output`, write to that path exactly (create intermediate directories if needed)
