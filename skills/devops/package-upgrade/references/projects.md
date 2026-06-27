# Projects

Repos under `~/projects/` that use this upgrade workflow.

| Repo | Package Manager | Special Notes |
|------|----------------|---------------|
| `next-dnd-starter-kit` | pnpm | Single app |
| `social-skill-ai-coach` | pnpm | Workspace (2 projects) |
| `sveltekit-starter-kit` | pnpm | Single app, Prisma + SvelteKit |
| `turborepo-starter-kit` | pnpm | Monorepo (8 workspaces), Expo mobile, `@types/node` + `@typescript/native-preview` in catalog |
| `ultra-light-monorepo` | pnpm | Monorepo (7 workspaces), SvelteKit + Hono, `@types/node` in `apps/web/package.json` |

## Version Cap Locations

How each project declares `@types/node`:

- `next-dnd-starter-kit` → `package.json` devDependencies
- `social-skill-ai-coach` → `package.json` devDependencies
- `sveltekit-starter-kit` → `package.json` devDependencies
- `turborepo-starter-kit` → `pnpm-workspace.yaml` catalog entry
- `ultra-light-monorepo` → `apps/web/package.json` devDependencies

How each project declares `@typescript/native-preview`:

- `next-dnd-starter-kit` → `package.json` devDependencies
- `social-skill-ai-coach` → `package.json` devDependencies
- `turborepo-starter-kit` → `pnpm-workspace.yaml` catalog entry
- Others: not present

## GitHub Remotes

All repos are under `https://github.com/john-data-chen/<repo-name>.git`.

`gh` CLI may not be available — fall back to the GitHub REST API with token from `git credential fill`.
