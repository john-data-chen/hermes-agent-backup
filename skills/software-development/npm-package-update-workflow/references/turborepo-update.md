# Turborepo Update Reference

## Projects
- `~/projects/turborepo-starter-kit/` — pnpm workspace monorepo (8 workspace packages)
- `~/projects/next-dnd-starter-kit/` — single-package Next.js app

## Turborepo Cron Job IDs
- `bfddf898b8cf` — 更新 turborepo packages
- `6612902dca66` — 更新 next-dnd-starter-kit

## CI Flag Requirement
Turborepo jobs MUST use `CI=true` prefix for all pnpm commands:
```bash
CI=true pnpm install
CI=true pnpm update --latest
CI=true pnpm test
```
Without this, pnpm aborts with `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY`.

## Lockfile Fix
When `pnpm install` fails with `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`:
```bash
pnpm install --no-frozen-lockfile
git add pnpm-lock.yaml && git commit -m "chore: update pnpm lockfile"
```

## Version Pin — `@types/node`
Both projects hold `@types/node` at `^24.12.3`:
- `next-dnd-starter-kit/package.json`: `"@types/node": "^24.12.3"`
- `turborepo-starter-kit/pnpm-workspace.yaml` catalog: `"@types/node": "^24.12.3"`

If `pnpm update --latest` bumps it, revert immediately:
```bash
pnpm add @types/node@^24.12.3
```
For turborepo, also update `pnpm-workspace.yaml` catalog entry.

## pnpm Store Error
`ERR_PNPM_UNEXPECTED_STORE` indicates node_modules is linked from a different store location. Run:
```bash
pnpm install
```
to re-sync before continuing.

## Job Execution Log Location
Logs: `~/.hermes/logs/agent.log` — search for `cron_<job_id>` to find specific run sessions.

Job outputs: `~/.hermes/cron/output/<job_id>/<timestamp>.md`