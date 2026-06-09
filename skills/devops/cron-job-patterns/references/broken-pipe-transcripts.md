# Broken Pipe Error Transcripts

## brew-update (2026-06-09)

**Command:** `brew upgrade` then `brew cleanup`
**Error:** `RuntimeError: [Errno 32] Broken pipe`
**Output file:** `.hermes/cron/output/215931199688/2026-06-09_04-39-14.md`

No agent output at all — the error occurred before any tool call completed.

## npm-global-update (2026-06-09)

**Command:** `npm update -g`
**Error:** `RuntimeError: [Errno 32] Broken pipe`
**Output file:** `.hermes/cron/output/976ad6723e2b/2026-06-09_04-15-27.md`

No agent output at all — same pattern.

## sveltekit-starter-kit (2026-06-08)

**Command:** `pnpm update --latest` (inside cron job)
**Error:** `RuntimeError: [Errno 32] Broken pipe`
**Output file:** `.hermes/cron/output/71e788ea2d9d/` (check latest timestamp)

Same pattern — large package manager output overflows pipe buffer.
