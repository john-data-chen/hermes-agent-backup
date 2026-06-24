Hermes config backup content: .hermes/skills, .hermes/config.yaml, .hermes/memories, .hermes/cron/jobs.json. Exclude: .hermes/cron/output/ (historical run logs). CRITICAL ORDER: (1) aggregate files to staging, (2) scan staging for security issues FIRST (phone numbers, passwords, API keys, tokens, secrets), (3) only after confirming clean, create zip. Use rm -f before zip to avoid stale entries.
§
npm approve-scripts 對全域套件無效（不支援 -g flag）。正確做法：在 npm prefix -g 目錄下執行 npm init -y 建立 package.json，加上 symlink 指向 lib/node_modules，然後執行 npm approve-scripts <pkg>。或用 npx 代替。之後記得清理 symlink。
§
Monday update cron jobs schedule (cron trigger time UTC+8): 05:00 next-dnd-starter-kit, 05:30 turborepo-starter-kit, 06:00 sveltekit-starter-kit, 06:30 Hermes Agent + backup.
§
npm package updates workflow → skill `npm-package-updates` (software-development category). Load it for the full workflow + per-repo profiles.
§
@types/node 24.x.x pinning, zero-change abort, and per-repo quirks → skill `npm-package-updates` (software-development category).