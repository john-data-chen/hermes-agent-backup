Hermes config backup content: .hermes/skills, .hermes/config.yaml, .hermes/memories, .hermes/cron/jobs.json. Exclude: .hermes/cron/output/ (historical run logs). Use rm -f before zip to avoid stale entries.
§
npm approve-scripts 對全域套件無效（不支援 -g flag）。正確做法：在 npm prefix -g 目錄下執行 npm init -y 建立 package.json，加上 symlink 指向 lib/node_modules，然後執行 npm approve-scripts <pkg>。或用 npx 代替。之後記得清理 symlink。
§
Monday update cron jobs schedule (cron trigger time UTC+8): 05:00 next-dnd-starter-kit, 05:30 turborepo-starter-kit, 06:00 sveltekit-starter-kit, 06:30 Hermes Agent + backup.
§
For ALL repo update tasks (npm packages, etc.), the CRITICAL first step is: checkout main, pull latest, THEN create update branch. Never skip git pull on main — even when intervening in an existing branch, always start from scratch: delete the old branch, pull main, create fresh branch. The user explicitly called this out as a hard requirement for all repo update workflows.
§
@types/node rule: pin to latest 24.x.x version (not hardcoded to 24.12.4). Before each repo update, run `npm view @types/node versions --json | python3 -c "import json,sys; vs=[v for v in json.load(sys.stdin) if v.startswith('24.')]; print(vs[-1])"` to find latest 24.x.x, then pin to that. Affects: npm-package-update-workflow skill, all 3 cron job prompts (next-dnd, turborepo, sveltekit).
§
npm-package-update-workflow zero-change rule: If `pnpm update --latest` results in NO changes to `package.json` (only lockfile changes), abort entirely — do NOT commit, push, or create a PR. Instead: checkout main, delete all local branches except main/dev, prune remote tracking refs, and report "No package version changes". This rule also means that any existing update branch from the same day must be cleaned up before aborting.