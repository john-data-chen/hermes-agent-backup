---
name: cron-job-patterns
description: >
  Patterns and pitfalls for authoring Hermes cron job prompts. Covers prompt structure,
  toolset selection, output handling, and common failure modes.
  Use when creating, updating, or debugging cron job prompts — especially for system
  maintenance jobs (brew, npm, pip, apt, etc.).
version: 1.0.0
tags: [cron, jobs, patterns, pitfalls, prompt-engineering]
---

# Cron Job Patterns

Recurring patterns, pitfalls, and best practices for authoring Hermes cron job prompts.

---

## Pitfall: Broken Pipe on Large-Output Commands

**Symptoms:** `RuntimeError: [Errno 32] Broken pipe` when cron job runs `brew upgrade`, `npm update -g`, `pip install --upgrade`, or any command with massive stdout.

**Root cause:** The cron runtime captures stdout via a pipe. When the command produces more output than the pipe buffer can hold before the reader drains it, the writer receives SIGPIPE → Broken pipe.

**Fix:** Always redirect large-output commands to a file first, then read the file to summarize. Use `tee` so you can also tail in the terminal call for a quick sanity check:

```bash
brew upgrade 2>&1 | tee /tmp/cron-output.log
# OR
npm update -g 2>&1 | tee /tmp/cron-output.log
```

Then in the prompt, instruct the agent to `read_file /tmp/cron-output.log` and summarize. Clean up at the end: `rm /tmp/cron-output.log`.

**Template prompt snippet for cron jobs with large-output commands:**

```
1. 先把輸出導向檔案避免 Broken pipe：
   <command> 2>&1 | tee /tmp/cron-output.log
2. 用 read_file 讀取 /tmp/cron-output.log，摘要報告結果
3. 清理暫存檔：rm /tmp/cron-output.log
```

**Affected commands:** `brew upgrade`, `npm update -g`, `pip install --upgrade --dry-run`, `apt list --upgradable`, any package manager with verbose output, any build tool with extensive logging.

---

## Pitfall: MEDIA Paths Must Be Absolute

Cron job delivery does NOT expand `~` in `MEDIA:` paths. `MEDIA:~/hermes_backup.zip` silently fails. Always use absolute paths: `MEDIA:/Users/johnchen/hermes_backup.zip`.

---

## Pitfall: Stale Zip Archives

`zip -r output.zip <files>` with existing archive adds/updates listed files but leaves other files intact. Always `rm -f` first:

```bash
rm -f ~/hermes_backup.zip && zip -r hermes_backup.zip .hermes/skills .hermes/config.yaml .hermes/memories .hermes/cron/jobs.json -x "*.lock"
```

---

## Pattern: Conditional Logic in Single Job

When task B should run only if task A had results → put both in one prompt. The agent reads A's output, decides whether to run B. Cleaner than chaining two jobs.

```
1. Run hermes update
2. Check output: if "Updated" found → run backup; if "already up to date" → skip
3. Report what happened
```

Set `enabled_toolsets: ["terminal", "file"]` for such jobs.

---

## Pattern: Toolset Selection

Restrict toolsets per job to reduce token overhead:
- `["terminal", "file"]` — shell commands + file reading (most maintenance jobs)
- `["terminal", "file", "web"]` — when also doing git/PR operations
- `["terminal"]` — pure command execution, no file reading needed
- `null` (inherit all) — only when job genuinely needs browser, vision, etc.
