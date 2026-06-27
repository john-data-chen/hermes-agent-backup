Package upgrade workflow: use `pnpm-package-upgrade` skill (software-development). Key rules: git checkout main + pull first, @types/node capped at 24.x.x, pnpm install last.
§
Recurring projects for package upgrades: next-dnd-starter-kit, social-skill-ai-coach, sveltekit-starter-kit, turborepo-starter-kit, ultra-light-monorepo — all pnpm-based, located under ~/projects/
§
User has 5 projects (next-dnd-starter-kit, social-skill-ai-coach, sveltekit-starter-kit, turborepo-starter-kit, ultra-light-monorepo) under ~/projects/. Regular package upgrades — use `package-upgrade-workflow` skill.
§
turborepo-starter-kit: apps/mobile uses Expo, must run `pnpm mobile:expo:upgrade` separately after the main `pnpm up -r --latest`. This is the last step before pnpm install.
§
Package upgrade: keep `"@typescript/native-preview": "beta"` as-is — do NOT upgrade it. Applies to all projects.
§
Hermes backup: include only config.yaml, memories/, skills/, cron/jobs.json. Exclude .env, auth.json, state.db, sessions/, *.lock, logs, caches. Check for info leakage before compressing. Use zip format.