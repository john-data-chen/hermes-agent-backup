---
name: npm-package-update-workflow
description: Use when running automated npm/package updates across projects. Handles dependency upgrades, verification, and PR creation.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [npm, packages, update, dependencies, cron]
    related_skills: [github-pr-workflow]
---

# npm Package Update Workflow

## Overview

Automated workflow for keeping npm packages up-to-date across projects. Checks for outdated dependencies, updates them to latest versions, verifies the build passes, and creates a PR for review.

## When to Use

- Scheduled cron job updating packages in a project repository
- Manual triggered update for a specific project
- Batch updates across multiple monorepo packages

## Workflow Steps

### 1. Ensure Main Branch is Up-to-Date

**CRITICAL: Always pull latest main before creating the update branch.**

```bash
cd ~/projects/<project-name>
git checkout main
git pull
```

This prevents the update branch from being based on stale code when others have committed to main since your last sync.

### 2. Create Update Branch

```bash
git checkout -b feat/john/update-packages-YYYYMMDD
```

Branch naming convention: `feat/john/update-packages-YYYYMMDD` for user branches.

### 3. Environment Setup

Check and create environment files if needed:

```bash
# Check for .env, .env.test
# If AUTH_SECRET is a placeholder, generate a new one:
openssl rand -base64 32
```

### 4. Update Dependencies

**Turborepo / monorepo projects — run pnpm install first:**
```bash
cd ~/projects/<project-name>
CI=true pnpm install
CI=true pnpm update --latest
```

**Single-package projects:**
```bash
cd ~/projects/<project-name>
pnpm update --latest
```

**IMPORTANT: Hold `@types/node` at `^24.12.4`** — do not update it even if a newer version exists. If `pnpm update --latest` bumps it, immediately revert:
```bash
pnpm add @types/node@^24.12.4
```

Or for npm:
```bash
npm update --save
# Then hold types/node:
npm install @types/node@^24.12.4 --save-exact
```

### 5. Verify Changes

Only proceed if `package.json` changed. Run verification in order:

```bash
pnpm lint
pnpm test
pnpm build
```

If any step fails:
- Fix the issue manually
- Or revert problematic packages (e.g., `pnpm add <package>@<old-version>`)
- Document the blocker in the PR

### 6. Commit and Push

```bash
git add -A
git commit -m "chore: update npm packages to latest versions"
git push -u origin HEAD
```

### 7. Create PR

```bash
gh pr create --title "chore: update npm packages $(date +%Y-%m-%d)" --body "Automated update"
```

Or via GitHub web UI if token unavailable.

## Common Pitfalls

1. **Skipping `git pull` on main** — Update branch may conflict with new commits on main, causing merge difficulties.

2. **Not running all verification steps** — Skipping `build` can merge breaking changes. Always run lint + test + build.

3. **Missing environment files** — Some projects require `.env` or `.env.test` to exist before tests pass.

4. **Type dependency version drift** — When updating `@types/node`, other packages may need matching type versions. Hold types at a compatible version if needed.

5. **Lockfile conflicts** — If lockfile is outdated, run `pnpm install` before committing.

6. **PR shows wrong branch/version** — Always confirm the update branch is what the user is looking at. When the user says "the version is wrong in the PR," they may be looking at an old branch. Check `git branch -a` to find all remote branches including in-progress feature branches, not just main.

7. **Turborepo pnpm update --latest scope** — `pnpm update --latest` without filters only updates the root. Use `--filter <pkg>` to update specific workspace packages. Always verify the intended packages were actually updated.

8. **Lockfile mismatch on monorepo** — When pnpm throws `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH` during `pnpm install`, run `pnpm install --no-frozen-lockfile` instead. This rebuilds the lockfile to match the current catalog configuration.

## Verification Checklist

- [ ] `git checkout main && git pull` completed before branch creation
- [ ] Branch name follows `feat/john/update-packages-YYYYMMDD` format
- [ ] `.env` / `.env.test` created if missing and AUTH_SECRET is placeholder
- [ ] `pnpm update --latest` ran and package.json changed (turborepo: `CI=true pnpm install` first)
- [ ] `@types/node` held at `^24.12.4` — not updated
- [ ] `pnpm lint` passed (warnings acceptable)
- [ ] `pnpm test` passed (turborepo: use `CI=true`)
- [ ] `pnpm build` passed
- [ ] Changes committed and pushed
- [ ] PR created or link provided for manual PR creation

## Project-Specific Notes

For turborepo projects, see `references/turborepo-update.md` for CI flags, lockfile fixes, and version pin rules.