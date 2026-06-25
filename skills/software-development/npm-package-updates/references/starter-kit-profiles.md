# Starter Kit Profiles

Per-repo quirks discovered during actual update runs.

## next-dnd-starter-kit

- **Type**: Standard Next.js (pnpm)
- **@types/node location**: Root `package.json` `devDependencies`
- **Pin style**: `"^24.13.2"` (with caret)
- **Check for**: Changes in root `package.json`
- **Last updated**: 2026-06-24
  - lucide-react: ^1.20.0 → ^1.21.0
  - mongoose: ^9.7.0 → ^9.7.2
  - @commitlint/*: ^21.0.2 → ^21.1.0
  - @playwright/test: ^1.61.0 → ^1.61.1
  - @typescript/native-preview: 7.0.0-dev.20260421.2 → 7.0.0-dev.20260623.1
  - oxfmt: ^0.55.0 → ^0.56.0
  - oxlint: ^1.70.0 → ^1.71.0

## turborepo-starter-kit

- **Type**: Turborepo (pnpm, catalogs)
- **@types/node location**: `pnpm-workspace.yaml` `catalog:` section AND `overrides:` section
- **Pin style in catalog**: `"24.13.2"` (exact, no caret)
- **Check for**: Root `package.json` AND `pnpm-workspace.yaml`
- **Sub-packages check**: `apps/*/package.json`, `packages/*/package.json`
- **Last updated**: 2026-06-24
  - @commitlint/*: ^21.0.2 → ^21.1.0
  - lint-staged: ^17.0.7 → ^17.0.8
  - oxfmt: ^0.55.0 → ^0.56.0
  - oxlint: ^1.70.0 → ^1.71.0
- **Note**: Overrides section must be checked separately — it may pin @types/node independently of catalog
- **Note**: `"@typescript/native-preview": "beta"` is intentional — do NOT treat format→pinned as a real update

## social-skill-ai-coach

- **Type**: Standard Next.js (pnpm, monorepo with packages/)
- **@types/node location**: Root `package.json` `devDependencies`
- **Pin style**: `"^24.13.2"` (with caret)
- **Check for**: Changes in root `package.json`
- **Last updated**: 2026-06-24
  - @ai-sdk/react: ^3.0.210 → ^3.0.211
  - ai: ^6.0.208 → ^6.0.209
  - @commitlint/*: ^21.0.2 → ^21.1.0
  - @playwright/test: ^1.61.0 → ^1.61.1
  - @vitejs/plugin-react: ^6.0.2 → ^6.0.3
  - oxfmt: ^0.55.0 → ^0.56.0
  - oxlint: ^1.70.0 → ^1.71.0

## sveltekit-starter-kit

- **Type**: Standard SvelteKit (pnpm)
- **@types/node location**: Root `package.json` `devDependencies`
- **Pin style**: `"24.13.2"` (exact, no caret — differs from next-dnd's caret style)
- **Check for**: Changes in root `package.json`
- **Last updated**: 2026-06-24
  - @inlang/paraglide-js: ^2.20.0 → ^2.20.1
  - @playwright/test: ^1.61.0 → ^1.61.1
  - @sveltejs/kit: ^2.66.0 → ^2.67.0
  - @testing-library/svelte: ^5.3.1 → ^5.4.2
  - eslint-plugin-oxlint: ^1.70.0 → ^1.71.0
  - globals: ^17.6.0 → ^17.7.0
  - lint-staged: ^17.0.7 → ^17.0.8
  - oxfmt: ^0.55.0 → ^0.56.0
  - oxlint: ^1.70.0 → ^1.71.0
  - svelte: ^5.56.3 → ^5.56.4
  - svelte-check: ^4.6.0 → ^4.7.0
  - typescript-eslint: ^8.61.1 → ^8.62.0
  - vite: ^8.0.16 → ^8.1.0

## ultra-light-monorepo

- **Type**: Turborepo (pnpm, catalogs, Hono + SvelteKit)
- **@types/node location**: Only in `apps/web/package.json` (NOT at root)
- **Pin style in sub-package**: `"24.13.2"` (exact, no caret)
- **Catalog entries** (in pnpm-workspace.yaml): typescript, vitest, @vitest/coverage-v8, tsx, zod, svelte, @prisma/client, @prisma/adapter-pg, pg, prisma
- **Check for**: Root `package.json`, `pnpm-workspace.yaml`, AND all `apps/*/package.json`, `packages/*/package.json`
- **Last updated**: 2026-06-24
  - @commitlint/*: ^21.0.2 → ^21.1.0
  - @playwright/test: ^1.61.0 → ^1.61.1
  - eslint-plugin-oxlint: ^1.70.0 → ^1.71.0
  - globals: ^17.6.0 → ^17.7.0
  - lint-staged: ^17.0.7 → ^17.0.8
  - oxfmt: ^0.55.0 → ^0.56.0
  - oxlint: ^1.70.0 → ^1.71.0
  - typescript-eslint: ^8.61.1 → ^8.62.0
