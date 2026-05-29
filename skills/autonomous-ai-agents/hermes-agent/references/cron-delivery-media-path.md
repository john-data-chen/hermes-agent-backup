# Cron Delivery: MEDIA Protocol Path Requirements

## The Bug

The `MEDIA:` protocol requires an **absolute path** (e.g. `MEDIA:/Users/johnchen/hermes_backup.zip`).

Using `MEDIA:~/hermes_backup.zip` (tilde shorthand) will NOT work — the delivery system does NOT expand `~` to the user's home directory. The file silently fails to send.

## Root Cause

This applies to **cron job prompts and any automated output**. In a cron session there is no interactive shell expansion — the MEDIA path is used literally. Both the `deliver` field and inline `MEDIA:` references in the response need absolute paths.

## Fix Pattern

**Wrong (file never delivered):**
```
MEDIA:~/hermes_backup.zip
```

**Correct:**
```
MEDIA:/Users/johnchen/hermes_backup.zip
```

## Scope

Check ALL cron job prompts that send files:
- `hermes cron list` → inspect each job's `prompt_preview`
- Search for `MEDIA:` in the full prompt of each job
- Replace any `~/` with the absolute home path

## Examples of Affected Jobs

| Job | Prompt pattern |
|-----|----------------|
| 備份 Hermes（被動觸發） | `MEDIA:~/hermes_backup.zip` → `MEDIA:/Users/johnchen/hermes_backup.zip` |
| 更新 Hermes Agent + 條件備份 | `MEDIA:~/hermes_backup.zip` → `MEDIA:/Users/johnchen/hermes_backup.zip` |
