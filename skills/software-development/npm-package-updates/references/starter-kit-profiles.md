# Starter Kit Profiles

Per-repo quirks discovered during actual update runs.

## next-dnd-starter-kit

- **Type**: Standard Next.js (pnpm)
- **@types/node location**: Root `package.json` `devDependencies`
- **Pin style**: `"^24.13.2"` (with caret)
- **Check for**: Changes in root `package.json`
- **Last updated**: 2026-06-20
  - lucide-react: ^1.20.0 → ^1.21.0
  - mongoose: ^9.7.0 → ^9.7.1

## turborepo-starter-kit

- **Type**: Turborepo (pnpm, catalogs)
- **@types/node location**: `pnpm-workspace.yaml` `catalog:` section AND `overrides:` section
- **Pin style in catalog**: `"24.13.2"` (exact, no caret)
- **Check for**: Root `package.json` AND `pnpm-workspace.yaml`
- **Sub-packages check**: `apps/*/package.json`, `packages/*/package.json`
- **Last updated**: 2026-06-20 — zero change (only @typescript/native-preview format bump)
- **Note**: Overrides section must be checked separately — it may pin @types/node independently of catalog

## sveltekit-starter-kit

- **Type**: Standard SvelteKit (pnpm)
- **@types/node location**: Root `package.json` `devDependencies`
- **Pin style**: `"24.13.2"` (exact, no caret — differs from next-dnd's caret style)
- **Check for**: Changes in root `package.json`
- **Last updated**: 2026-06-20
  - pg: ^8.21.0 → ^8.22.0
  - @inlang/paraglide-js: ^2.19.0 → ^2.20.0
  - @sveltejs/adapter-vercel: ^6.3.3 → ^6.3.4
  - @sveltejs/kit: ^2.65.1 → ^2.66.0

## ultra-light-monorepo

- **Type**: Turborepo (pnpm, catalogs, Hono + SvelteKit)
- **@types/node location**: Only in `apps/web/package.json` (NOT at root)
- **Pin style in sub-package**: `"24.13.2"` (exact, no caret)
- **Catalog entries** (in pnpm-workspace.yaml): typescript, vitest, @vitest/coverage-v8, tsx, zod, svelte, @prisma/client, @prisma/adapter-pg, pg, prisma
- **Check for**: Root `package.json`, `pnpm-workspace.yaml`, AND all `apps/*/package.json`, `packages/*/package.json`
- **Last updated**: 2026-06-20 — zero change
