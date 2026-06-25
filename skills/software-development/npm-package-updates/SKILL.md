---
name: npm-package-updates
description: >-
  Run `pnpm update --latest` across npm/pnpm projects with @types/node pinning,
  catalog-aware change detection, and zero-change abort. For recurring dep
  updates on starter-kit monorepos and standard projects.
---

# npm-package-updates

Run automated dependency updates on pnpm-based projects (starter kits, monorepos).
Handles the full lifecycle: checkout → branch → update → pin → diff → commit/abort.

## When to load

- User asks: "run package updates on X repo" or "update deps" or "do the weekly update"
- Cron job triggers a package update run
- User says "check for outdated packages" or "bump dependencies"

## Workflow

### 1. Prepare the repo

```bash
cd ~/projects/<repo-name>
# Stash any local changes that would block checkout
git stash
git checkout main
git pull origin main
```

> [!CAUTION]
> **Never skip `git pull` on main.** Even when intervening mid-branch, always start from scratch: delete old branch → pull main → create fresh branch. This is a hard requirement.

### 2. Check @types/node version

Latest 24.x.x is what we pin to (26.x is too new for Node >=24 engines).

```bash
npm view @types/node versions --json 2>/dev/null | python3 -c \
  "import json,sys; vs=[v for v in json.load(sys.stdin) if v.startswith('24.')]; print(vs[-1])"
```

Record the result — you'll need it after `pnpm update --latest`.

> [!TIP]
> This npm view command also works in pnpm workspaces for a quick check.

### 3. Create update branch

```bash
git checkout -b chore/update-deps-$(date +%Y-%m-%d)
```

### 4. Run `pnpm update --latest`

```bash
pnpm update --latest 2>&1
```

This bumps all packages to their latest version within semver range.

### 5. Pin @types/node back

`pnpm update --latest` often bumps `@types/node` to 26.x.x, but the rule is to pin to latest 24.x.x (currently 24.13.2).

**Where to pin depends on the project structure:**

| Project type | Where @types/node lives | How to pin |
|---|---|---|
| Standard project (next-dnd, sveltekit) | `package.json` `devDependencies` | `patch` the version string |
| Turborepo with catalog | `pnpm-workspace.yaml` `catalog:` section | `patch` the catalog entry |
| Turborepo with catalog + overrides | Also check `pnpm-workspace.yaml` `overrides:` section | Pin there too |
| Multi-package (no root @types/node) | Check each sub-package.json (e.g. `apps/web/package.json`) | Patch the sub-package |

After patching, run `pnpm install` to sync the lockfile.

### 6. Check for real changes

```bash
git diff main -- '**/package.json' pnpm-workspace.yaml
```

**Zero-change rule**: If `pnpm update --latest` results in NO changes to any `package.json` (only pnpm-lock.yaml changes), abort entirely:
- `git checkout main`
- `git branch -D chore/update-deps-YYYY-MM-DD`
- `git restore pnpm-lock.yaml`
- Report: "No package version changes"

> [!IMPORTANT]
> For catalog-based repos (turborepo), also check if `pnpm-workspace.yaml` changed. A format-only change (e.g. "beta" → pinned version) that doesn't affect runtime packages also counts as zero-change.

### 7. Commit and push (only if meaningful changes)

```bash
git add package.json pnpm-lock.yaml pnpm-workspace.yaml
# Use a conventional commit message listing the updated packages
git commit -m "chore(deps): update <pkg1> to vX, <pkg2> to vY"
git push origin chore/update-deps-YYYY-MM-DD
```

Report the PR creation URL to the user.

## Project-specific quirks

See `references/starter-kit-profiles.md` for per-repo details (where @types/node lives, catalog structure, pin style).

## Pitfalls

1. **@types/node auto-bump**: `pnpm update --latest` always picks the latest major. Always pin back to latest 24.x.x — this is not optional.
2. **Catalog changes**: In turborepo catalogs, a version change in `pnpm-workspace.yaml` counts as a real change. A format-only change (e.g. "beta" → pinned version hash) does not.
3. **pnpm-lock.yaml after abort**: `git checkout main` leaves modified lockfile in working directory. Always `git restore pnpm-lock.yaml` after aborting.
4. **Missing lint-staged**: If `git commit` triggers lint-staged that finds no staged files matching patterns, it still proceeds — this is normal.
5. **`@typescript/native-preview`**: This package is pinned to `"beta"` in some catalogs — that's intentional. If `pnpm update --latest` changes it to a specific pinned version, revert it back to `"beta"` and treat the change as zero-change.
6. **`head` truncation**: When inspecting diffs, never truncate with `head` — you'll miss real changes in root `package.json`. Always check the full diff.
