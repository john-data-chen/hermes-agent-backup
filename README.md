# Backup of Hermes Agent

I use [Hermes Agent](https://github.com/nousresearch/hermes-agent) to execute routine jobs — this repo is an example of how I work with agents.

I created a skill so the Agent can back itself up and send me the zip file via WhatsApp, with a cron job handling it automatically every week.

I won't make it auto push new backup to the public repo until I double-check and make sure nothing is leaked — this is how I work with agents.

## What this repo includes

- `cron/` — Cron job definitions (`jobs.json` only)
- `memories/` — Persistent memories about the user and agent notes
- `skills/` — Skills for agents (content only; see excludes below)
- `SOUL.md` — Agent identity / personality

## What is intentionally excluded

- `config.yaml` — model/provider change often; not part of this backup
- Secrets: `.env`, `auth.json`
- Runtime state: `state.db`, `sessions/`, locks, logs, caches
- Skills machine meta: `.usage.json`, `.bundled_manifest`, `.curator_state`
- Bulky regenerable skill assets: `*.pdf`, PowerPoint `office/schemas/` (XSD)
- OS junk: `.DS_Store`

Restore bulky skill assets by reinstalling/refreshing the skill from hub or source if needed.
