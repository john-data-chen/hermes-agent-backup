---
name: hermes-config-backup
description: "Backup Hermes config with mandatory pre-zip security scan."
version: 1.0.0
author: agent
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [hermes, backup, security, config]
---

# Hermes Config Backup

Backup Hermes Agent config files to a timestamped zip. **Security scan must complete BEFORE zipping** — never zip first.

## Trigger

- User asks to backup Hermes configs
- Scheduled cron backup job
- Pre-upgrade snapshot

## What to include

| Path | Note |
|------|------|
| `.hermes/skills/` | All skills + references/templates/scripts |
| `.hermes/config.yaml` | Main config |
| `.hermes/memories/` | MEMORY.md + USER.md |
| `.hermes/cron/jobs.json` | May not exist — skip if missing |

## What to exclude

- `.hermes/cron/output/` — historical run logs (large, no config value)
- `.hermes/.env` — **NEVER include**, contains real API keys
- `.hermes/sessions/` — session transcripts
- `.hermes/logs/` — runtime logs
- `.hermes/state.db` — session store
- `.hermes/auth.json` — OAuth tokens

## Critical workflow order

```
1. AGGREGATE files to staging directory
2. SCAN staging for security issues (see below)
3. ZIP only after confirming clean
```

**Never zip before scanning.** If scan finds issues, fix them in staging first, re-scan, then zip.

## Security scan (step 2)

After aggregating to a staging dir, scan EVERY text file for:

### Phone numbers
```bash
grep -rnoP '09\d{2}[-\s]?\d{3}[-\s]?\d{3}' <staging>       # TW mobile
grep -rnoP '\+?\d{1,3}[-\s]?\d{3,4}[-\s]?\d{3,4}[-\s]?\d{3,4}' <staging>  # international
```
Filter out dates (2026-06-18), version numbers (1.2.3), and LaTeX/PDF noise.

### API keys
```bash
grep -rnoP '(?:sk-[A-Za-z0-9]{20,}|sk-ant-[A-Za-z0-9]+|AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{20,}|OR-[A-Za-z0-9_\-]+)' <staging>
```
Filter out example/demo values in SKILL.md documentation.

### Passwords & secrets
```bash
grep -rnoP '(?:password|passwd|pwd|secret)\s*[=:]\s*["\x27]?\S{4,}["\x27]?' <staging>
```
Filter out documentation examples, empty values, and config keys (not values).

### JWT tokens & private keys
```bash
grep -rnoP 'eyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+' <staging>
grep -rnoP 'BEGIN (?:RSA|EC|DSA|OPENSSH|PGP) PRIVATE KEY' <staging>
```

### Config.yaml deep check
Verify all `api_key`, `password`, `secret`, `token` fields are EMPTY strings:
```bash
grep -n 'api_key\|password\|secret\|token' <staging>/.hermes/config.yaml
```
Any non-empty value is a finding — must be investigated.

### Verify .env is NOT present
```bash
find <staging> -name '.env*' -o -name '*.env'
```
Should return nothing.

## Step 3: Create zip

```bash
rm -f ~/hermes-backup-*.zip
BACKUP_NAME="hermes-backup-$(date +%Y%m%d-%H%M%S).zip"
cd ~ && zip -r "$HOME/$BACKUP_NAME" .hermes/skills/ .hermes/config.yaml .hermes/memories/ .hermes/cron/jobs.json -x '.hermes/cron/output/*'
```

## Pitfalls

- **Do NOT zip then scan.** Order is: aggregate → scan → zip. The user explicitly requires this.
- `cron/jobs.json` may not exist — that's fine, zip will warn but continue.
- Stale backups: always `rm -f ~/hermes-backup-*.zip` before creating new one.
- False positives from SKILL.md examples: grep patterns may match demo API keys in documentation. Always verify hits manually — documentation examples like `sk-or-example-key` are safe.
- LaTeX `.sty` files produce noise in long-string pattern scans — filter them out or scan only `.md`, `.yaml`, `.json`, `.txt`, `.py` files.

## Verification

After zip created:
```bash
ls -lh ~/hermes-backup-*.zip
unzip -l ~/hermes-backup-*.zip | tail -1   # file count
```
