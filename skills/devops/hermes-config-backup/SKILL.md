---
name: hermes-config-backup
description: "Backup Hermes config, skills, memories, and cron jobs. Exclude secrets. Check for info leakage before compressing."
version: 1.0.0
author: johnchen
metadata:
  hermes:
    tags: [hermes, backup, config, skills, memories]
---

# Hermes Config Backup

Backup essential Hermes files. Exclude secrets and sensitive data.

## What to Backup

| Include | Exclude |
|---------|---------|
| `config.yaml` | `.env` (API keys) |
| `memories/MEMORY.md` | `auth.json` (OAuth tokens) |
| `memories/USER.md` | `state.db` (session store) |
| `skills/` (all) | `sessions/` (transcripts) |
| `cron/jobs.json` | `*.lock` (0-byte) |
| | `hermes-agent/` (source) |
| | `logs/`, `cache/`, `audio_cache/`, `image_cache/` |
| | `.DS_Store` |

## Workflow

### 1. Prepare temp directory

```bash
BACKUP_DIR="/tmp/hermes-backup-$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"
cd ~/.hermes
```

### 2. Copy files

```bash
cp config.yaml "$BACKUP_DIR/"
mkdir -p "$BACKUP_DIR/memories"
cp memories/MEMORY.md memories/USER.md "$BACKUP_DIR/memories/"
mkdir -p "$BACKUP_DIR/skills"
rsync -a --exclude='*.lock' --exclude='.DS_Store' --exclude='node_modules' skills/ "$BACKUP_DIR/skills/"
mkdir -p "$BACKUP_DIR/cron"
cp cron/jobs.json "$BACKUP_DIR/cron/"
```

### 3. Check for info leakage

**Before compressing**, scan for sensitive data:

```bash
# API keys (non-empty)
grep -rE 'api_key:\s*[^'\'']' "$BACKUP_DIR/" | grep -v node_modules

# Tokens
grep -rE 'ghp_|gho_|github_pat_|sk-[a-zA-Z0-9]{20,}' "$BACKUP_DIR/" | grep -v node_modules | grep -v 'example\|sample\|xxx'

# Phone numbers (exclude numeric config values)
grep -rE '\+?[0-9]{1,4}[-.\s]?\(?\d{1,4}\)?[-.\s]?\d{1,4}[-.\s]?\d{1,4}[-.\s]?\d{1,9}' "$BACKUP_DIR/" | grep -v 'timeout\|limit\|max\|min\|bytes\|chars\|sample_rate\|bit_rate'

# Email addresses
grep -rE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' "$BACKUP_DIR/"
```

If real secrets found → redact before compressing.

### 4. Compress as zip

```bash
ZIP_FILE="$HOME/backups/hermes/hermes-backup-$(date +%Y%m%d_%H%M%S).zip"
mkdir -p "$(dirname "$ZIP_FILE")"
cd "$BACKUP_DIR" && zip -r "$ZIP_FILE" . -x '*.DS_Store'
```

### 5. Cleanup

```bash
rm -rf "$BACKUP_DIR"
```

## Pitfalls

> **Never include .env or auth.json.** These contain API keys and OAuth tokens.

> **Always check for info leakage.** Even config.yaml may contain API URLs or voice IDs. Mask if needed.

> **Exclude *.lock files.** They are 0-byte and not needed.

> **Use zip format.** Not tar.gz.

> **Backup path:** `~/backups/hermes/hermes-backup-YYYYMMDD_HHMMSS.zip`
