User: John Chen (john-data-chen on GitHub)
Projects: next-dnd-starter-kit, turborepo-starter-kit (both GitHub repos under john-data-chen)
Package manager: pnpm (turborepo monorepo setup)

Critical constraint: @types/node must ALWAYS be pinned to ^24.12.3. This has caused issues twice — once in next-dnd-starter-kit package.json and once in turborepo-starter-kit pnpm-workspace.yaml catalog. The npm-package-update-workflow skill and cron job prompts now include this hold rule, but verify manually if jobs run to completion.

CI environment: Jobs run with CI=true for pnpm to avoid TTY-related errors in turborepo (ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY).

Communication style: User expects me to notice problems proactively (e.g., @types/node drift, missing gh CLI, failed job) and either fix or alert without being prompted.