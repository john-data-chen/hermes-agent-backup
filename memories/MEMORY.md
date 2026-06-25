Hermes config backup content: .hermes/skills, .hermes/config.yaml, .hermes/memories, .hermes/cron/jobs.json. Exclude: .hermes/cron/output/ (historical run logs). CRITICAL ORDER: (1) aggregate files to staging, (2) scan staging for security issues FIRST (phone numbers, passwords, API keys, tokens, secrets), (3) only after confirming clean, create zip. Use rm -f before zip to avoid stale entries.
§
pnpm shim at ~/Library/pnpm/pnpm is pinned to 9.15.4. Projects need pnpm 11.x (packageManager field). Fix: `npm install -g pnpm@11.8.0` installs to ~/.local/bin/pnpm, then use `export PATH="$HOME/.local/bin:$PATH"` to override the old shim.
§
Monday update cron jobs schedule (cron trigger time UTC+8): 05:00 next-dnd-starter-kit, 05:30 turborepo-starter-kit, 06:00 sveltekit-starter-kit, 06:30 Hermes Agent + backup.
§
npm package updates workflow → skill `npm-package-updates` (software-development category). Load it for the full workflow + per-repo profiles.
§
@types/node 24.x.x pinning, zero-change abort, and per-repo quirks → skill `npm-package-updates` (software-development category).
§
pnpm-workspace.yaml 中 `"@typescript/native-preview": "beta"` 是刻意固定寫法，不是需要 pin 的版本。pnpm update --latest 把它變成 pinned version 時，這是零變更（格式變化），不該被當作真實更新。