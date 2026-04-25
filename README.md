# Backup of Hermes Agent

I use Hermes Agent (https://github.com/nousresearch/hermes-agent) to execute routine jobs — this repo is an example of how I work with agents.

I created a skill so the Agent can back itself up and send me the zip file via WhatsApp, with a cron job handling it automatically every week.

I won't make it auto push new backup to the public repo until I double-check and make sure nothing is leaked — this is how I work with agents.

This repo includes:

- ⁠cron/ — Cron jobs
- skills/ — Skills for agents
- ⁠config.yaml — Hermes Agent config. I manually verified: no API keys, tokens, or secrets are included.
