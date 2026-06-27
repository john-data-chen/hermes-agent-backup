---
name: package-upgrade
description: "Batch package upgrades across multiple repos: pnpm up --latest, version caps, Expo mobile, branch+PR workflow."
version: 1.0.0
author: johnchen
metadata:
  hermes:
    tags: [npm, pnpm, dependencies, upgrade, monorepo, expo]
---

# Package Upgrade Workflow

Batch-upgrade npm packages across multiple repos using pnpm.

## Prerequisites

- All repos use **pnpm** as package manager
- GitHub CLI (`gh`) or API access for opening PRs
- Repos live under `~/projects/`

## Workflow (MANDATORY ORDER)

For **each** repo:

### 1. Prepare branch

```bash
cd ~/projects/<repo>
git checkout main
git pull
git checkout -b feat/john/update-packages-YYYYMMDD
```

**NEVER modify main directly.** Always create a feature branch and open a PR.

### 2. Run upgrade

```bash
pnpm up -r --latest
```

### 3. Enforce version caps

After upgrade, **check and revert** these packages if they were bumped beyond the allowed range:

| Package | Cap | Where |
|---------|-----|-------|
| `@types/node` | `24.x.x` only | `package.json` or `pnpm-workspace.yaml` (catalog) |
| `@typescript/native-preview` | `"beta"` — do NOT upgrade | `package.json` or `pnpm-workspace.yaml` (catalog) |

Revert with `sed` or manual edit, then confirm with `grep`.

### 4. Expo mobile (turborepo-starter-kit only)

For **turborepo-starter-kit**, the `apps/mobile` workspace uses Expo and requires a separate upgrade step:

```bash
pnpm mobile:expo:upgrade
```

Run this **after** the general `pnpm up -r --latest` and version cap enforcement, but **before** `pnpm install`.

### 5. Install

```bash
pnpm install
```

This regenerates the lockfile after any manual version reverts.

### 6. Commit, push, open PR

```bash
git add -A
git commit -m "chore(deps): update all packages to latest versions"
git push -u origin feat/john/update-packages-YYYYMMDD
```

Then open a PR via `gh pr create` or the GitHub API.

## Pitfalls

> **Never modify main.** All changes go through a feature branch + PR. User reviews before merge.

> **@types/node cap is strict.** `pnpm up --latest` will bump it to 26.x or higher. Always check and revert to 24.x.x.

> **@typescript/native-preview stays at "beta".** Never upgrade this package.

> **turborepo-starter-kit has two upgrade phases.** General upgrade first, then `pnpm mobile:expo:upgrade` for the Expo mobile app. Don't skip the Expo step.

> **pnpm install is last.** Run it after all version reverts are done so the lockfile reflects the correct versions.

> **Branch naming convention:** `feat/john/update-packages-YYYYMMDD` (date-based).

## Project-Specific Notes

See `references/projects.md` for the list of repos and their specific configurations.
