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

**If `git checkout main` fails due to uncommitted changes**, stash first:
```bash
git stash
git checkout main
git pull
git checkout -b feat/john/update-packages-YYYYMMDD
```
Leave the stash on the old branch — the user can pop it when they return.

**NEVER modify main directly.** Always create a feature branch and open a PR.

### 2. Run upgrade

```bash
pnpm up -r --latest
```

**If the upgrade fails** (e.g. a package's latest version has a broken transitive dep), see the "Handling broken upstream deps" section in Pitfalls below.

### 3. Enforce version caps

After upgrade, **check and revert** these packages if they were bumped beyond the allowed range:

| Package | Cap | Where |
|---------|-----|-------|
| `@types/node` | latest `24.x.x` (e.g. `^24.13.2`, not `^24.0.0`) | `package.json` or `pnpm-workspace.yaml` (catalog) |
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

### 6. Verify

Run the project's verification suite to confirm nothing broke. Read the project's `AGENTS.md` for the exact commands; the standard set is:

```bash
pnpm lint
pnpm build
pnpm check
```

If any step fails, fix immediately before committing. Common issues: version incompatibilities between upgraded packages, deprecated API usage surfaced by stricter types.

### 7. Commit, push, open PR

```bash
git add -A
git commit -m "chore(deps): update all packages to latest versions"
git push -u origin feat/john/update-packages-YYYYMMDD
```

Then open a PR via `gh pr create` or the GitHub API.

## Pitfalls

> **Never modify main.** All changes go through a feature branch + PR. User reviews before merge. If you accidentally commit to main (e.g. while fixing a subagent's missed step), recover: `git reset --soft HEAD~1 && git stash && git checkout <feature-branch> && git stash pop`, then force-push main back: `git checkout main && git reset --hard origin/main && git push --force origin main`.

> **@types/node cap is strict.** `pnpm up --latest` will bump it to 26.x or higher. Always check and revert — use the latest 24.x.x version (e.g. `^24.13.2`), not `^24.0.0`.

> **@typescript/native-preview stays at "beta".** Never upgrade this package.

> **turborepo-starter-kit has two upgrade phases.** General upgrade first, then `pnpm mobile:expo:upgrade` for the Expo mobile app. Don't skip the Expo step.

> **Subagent delegation pitfall (turborepo-starter-kit).** When delegating to subagents, the `pnpm mobile:expo:upgrade` step often gets skipped even when listed in the prompt. Subagents treat `pnpm up -r --latest` as "done" and move on. **Mitigation:** Either (a) run turborepo-starter-kit manually (not delegated), or (b) verify the subagent ran the command by checking `git log` for an expo-specific commit before approving. If missing, run it yourself on the feature branch and amend.

> **pnpm install is last.** Run it after all version reverts are done so the lockfile reflects the correct versions.

> **Transient registry errors on `pnpm up`.** Newly published package versions may not propagate to all registry mirrors immediately. If `pnpm up -r --latest` fails with `ERR_PNPM_NO_MATCHING_VERSION` for a package that should exist (e.g. `@vitest/spy@4.1.10`), wait a few seconds and retry. Check availability with `pnpm view <pkg> versions --json | tail` before concluding the version doesn't exist.

> **Branch naming convention:** `feat/john/update-packages-YYYYMMDD` (date-based).

> **Handling broken upstream deps.** Sometimes `pnpm up -r --latest` fails because a package's latest version has a missing transitive dependency (e.g. vitest 4.1.10 → @vitest/pretty-format@4.1.10 which doesn't exist). Workaround:
> 1. Pin the broken package to its last working exact version (no caret) in **whichever file declares it** (root `package.json` or `pnpm-workspace.yaml` catalog)
> 2. Run `pnpm up -r --latest` — it skips the pinned package and upgrades everything else. Note: pnpm preserves the range style — no caret stays no caret.
> 3. **Re-add the caret manually** if the original had one (e.g. `"4.1.10"` → `"^4.1.10"`)
> 4. Check `git diff` — if the broken version was written back, manually revert to the last working version with caret
> 5. Run `pnpm install`, then caps enforcement, then verify (lint/build/check)
>
> **Race condition note:** The missing transitive dep may appear within minutes (npm publish is eventually consistent). After the workaround succeeds, check with `pnpm view <pkg> versions --json | tail` — if the missing version now exists, you can safely upgrade to it.

> **Monorepo sequential upgrade with --filter.** If `pnpm up -r --latest` fails at the root level, you can upgrade sub-packages first: `pnpm up --latest --filter '!.'` then handle the root separately. Useful when the failure is root-specific (e.g. vitest in root but not in sub-packages).

## Project-Specific Notes

See `references/projects.md` for the list of repos and their specific configurations.
