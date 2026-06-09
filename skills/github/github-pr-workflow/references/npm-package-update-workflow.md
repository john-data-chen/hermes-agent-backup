# npm Package Update Workflow

## Overview

Automated workflow for keeping npm packages up-to-date across projects. Uses the github-pr-workflow for branch/PR lifecycle.

## When to Use

- Scheduled cron job updating packages in a project repository
- Manual triggered update for a specific project
- Batch updates across multiple monorepo packages

## Workflow Steps

### 1. Ensure Main Branch is Up-to-Date

```bash
cd ~/projects/<project-name>
git checkout main
git pull
```

**Rescue:** If an update branch already exists from a crashed cron job, delete it first (both local and remote):
```bash
git checkout main
git branch -D feat/john/update-packages-*
git push origin --delete feat/john/update-packages-* 2>/dev/null
git pull
git checkout -b feat/john/update-packages-$(date +%Y%m%d)
```

### 2. Create Update Branch

```bash
git checkout -b feat/john/update-packages-YYYYMMDD
```

### 3. Environment Setup

```bash
# Check for .env, .env.test
# If AUTH_SECRET is a placeholder, generate:
openssl rand -base64 32
```

### 4. Update Dependencies

**Turborepo / monorepo:**
```bash
CI=true pnpm install
CI=true pnpm update --latest
```

**Single-package:**
```bash
pnpm update --latest
```

**Hold `@types/node` at latest 24.x.x (not 25.x):**
First check latest 24.x.x version:
```bash
npm view @types/node versions --json | python3 -c "import json,sys; vs=[v for v in json.load(sys.stdin) if v.startswith('24.')]; print(vs[-1])"
```
Then pin to it (use ^range or exact match project's convention):
```bash
# ^range (most projects)
pnpm add -D @types/node@^<latest-24.x.x>
# exact (SvelteKit starter kit)
pnpm add -D @types/node@<latest-24.x.x>
```

### 5. Zero-Change Gate — Abort if package.json Unchanged

After `pnpm update --latest` (and `@types/node` pin), check if `package.json` actually has real version changes:

```bash
git diff --name-only | grep -q 'package.json'
```

**If `package.json` has NO changes** (only `pnpm-lock.yaml` may differ):
- **Do NOT commit, push, or create a PR** — there are no real dependency updates.
- Delete the local update branch and any other stale branches (except `main` and `dev`):
  ```bash
  git checkout main
  git branch | grep -vE '^\*|main$|dev$' | xargs -r git branch -D
  git remote prune origin 2>/dev/null
  ```
- Report: "No package version changes — nothing to update. Cleaned up local branches."
- **Stop here.** Skip verification, commit, push, and PR steps entirely.

**If `package.json` DID change** (actual version bumps), proceed to verify.

### 6. Verify Changes

```bash
# Standard order for most projects:
pnpm lint      # Must pass with zero errors
pnpm test      # Skip if test requires unavailable infra (e.g. Docker DB)
pnpm build     # Must succeed
```

**SvelteKit projects — follow project's AGENTS.md order:**
```bash
pnpm lint      # 1. Lint — zero errors
pnpm build     # 2. Build — TypeScript + Vite
pnpm check     # 3. Type/svelte-check — zero errors
# After user confirmation:
pnpm test      # 4. Unit tests
pnpm test:e2e  # 5. E2E tests
```

If any step fails: fix or revert problematic packages, document blocker. Test failures from missing infra (Docker/DB not running) are acceptable blockers — document and continue.

### 7. Commit, Push, PR

```bash
git add -A
git commit -m "chore: update npm packages to latest versions"
git push -u origin HEAD
```

**Create PR — with gh (preferred):**
```bash
gh pr create --title "chore: update npm packages $(date +%Y-%m-%d)" --body "Automated update"
```

**Create PR — git + curl fallback (gh not in PATH or not authenticated):**
First extract token from macOS keychain or env:
```bash
# Try macOS keychain
GITHUB_TOKEN=$(git credential-osxkeychain get <<< $'protocol=https\nhost=github.com' 2>/dev/null | grep "^password=" | head -1 | cut -d= -f2-)

# Fallback: ~/.hermes/.env
if [ -z "$GITHUB_TOKEN" ] && [ -f ~/.hermes/.env ]; then
  GITHUB_TOKEN=$(grep "^GITHUB_TOKEN=" ~/.hermes/.env | head -1 | cut -d= -f2)
fi

# Then create PR via API
OWNER_REPO=$(git remote get-url origin | sed -E 's|.*github\.com[:/]||; s|\.git$||')
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/$OWNER_REPO/pulls" \
  -d "{
    \"title\": \"chore: update packages $(date +%Y-%m-%d)\",
    \"body\": \"Automated weekly dependency update\",
    \"head\": \"$(git branch --show-current)\",
    \"base\": \"main\"
  }"
```

### 8. Post-PR / Post-Abort Cleanup

**After PR created** (or after aborting on zero-change):
- Ensure you're on `main`:
  ```bash
  git checkout main
  ```
- Delete all local branches **except** `main` and `dev`:
  ```bash
  git branch | grep -vE '^\*|main$|dev$' | xargs -r git branch -D
  ```
- Prune stale remote tracking refs:
  ```bash
  git remote prune origin 2>/dev/null
  ```

## Common Pitfalls

1. **Skipping `git pull` on main / reusing old branches** — CRITICAL: NEVER amend or reuse an existing update branch. Always start from scratch: delete old branch (local + remote) → `git checkout main && git pull` → create fresh `feat/john/update-packages-YYYYMMDD`. Even when "just fixing" a previous attempt, delete the old branch entirely and start clean.
2. **Not running all verification** — Always run lint + build + check + test. Order varies by project (SvelteKit: lint → build → check, then test if infra available).
3. **Missing environment files** — Some projects require `.env` to exist.
4. **Type dependency version drift** — Hold `@types/node` at latest 24.x.x (dynamic check). Before each run, look up latest 24.x.x with `npm view @types/node versions --json | python3 -c "import json,sys; vs=[v for v in json.load(sys.stdin) if v.startswith('24.')]; print(vs[-1])"`. Update THIS file and all cron job prompts when latest 24.x.x changes.
5. **Lockfile conflicts** — Run `pnpm install` before committing.
6. **Turborepo scope** — `pnpm update --latest` without filters only updates root. Use `--filter <pkg>`.
7. **Lockfile config mismatch** — `pnpm install --no-frozen-lockfile` when `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`.
8. **Cron prompt version drift** — The `@types/node` hold version is duplicated in THIS file AND in cron job prompts. Update ALL locations when bumping.
9. **Global npm packages** — `npm approve-scripts` doesn't support `-g`. See alternative workarounds.
10. **Broken pipe in cron context** — `RuntimeError: [Errno 32] Broken pipe` when output piped to filter that closes early (head/less/grep). Avoid piped commands in cron prompts. Set `GIT_PAGER=cat` to suppress git's built-in pager.
11. **gh CLI not in PATH** — `gh` may not be installed or PATH-accessible. Extract token from macOS keychain (`git credential-osxkeychain`) and use GitHub API via curl. Falls back to `npx gh` but requires auth setup each time.
12. **@types/node exact vs caret** — Some projects use exact version (`24.13.1`), others use caret (`^24.13.1`). Match the project's convention. Check `package.json` for the existing pattern before pinning. Always use the latest 24.x.x version, not hardcoded.
13. **pnpm `minimumReleaseAge` blocks fresh packages** — `pnpm-workspace.yaml` may have `minimumReleaseAge: 1440` which blocks packages released within 24 hours (including transitive dependencies). This silently skips updates — `pnpm update --latest` reports success but omits fresh packages. Fix: set `minimumReleaseAge: 0` in the project's `pnpm-workspace.yaml`. Commit this change as part of the update PR.
14. **Lockfile-only changes are not real updates** — `pnpm update --latest` may change `pnpm-lock.yaml` even when no `package.json` version changes (e.g. transitive dependency bumps). Check `git diff --name-only` for `package.json` before proceeding. Lockfile-only changes must NOT be committed, pushed, or PR'd.

## Verification Checklist

- [ ] `git checkout main && git pull` completed
- [ ] Branch name: `feat/john/update-packages-YYYYMMDD` (if proceeding)
- [ ] `.env` created if missing
- [ ] `pnpm update --latest` ran
- [ ] **Zero-Change Gate**: package.json changed → proceed; unchanged → abort + cleanup
- [ ] `@types/node` held at latest 24.x.x (check with `npm view @types/node versions ...`)
- [ ] `pnpm lint` passed
- [ ] `pnpm test` passed
- [ ] `pnpm build` passed
- [ ] Changes committed, pushed, PR created
- [ ] All non-main/dev branches deleted
