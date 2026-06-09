# Kanban Codex Lane Pattern

## Overview

This pattern lets a Hermes Kanban worker use Codex CLI as an isolated implementation lane while Hermes retains ownership of task lifecycle, reconciliation, testing, and handoff.

## When to Use

Use Codex lane when ALL are true: coding/refactor/test task with clear acceptance criteria, isolated worktree available, Hermes can verify independently.

Do NOT use: human judgment needed, secrets/credentials involved, small direct edit is faster.

## Ownership Rules

1. Hermes owns Kanban lifecycle — Codex must never call kanban tools
2. Hermes owns final acceptance — Codex commits are untrusted until reviewed
3. Hermes owns test execution — Codex runs are advisory
4. Hermes owns safety — reject lane if safety boundaries change
5. Hermes owns cleanup — kill stuck Codex, remove worktrees

## Worktree Pattern

```bash
TASK_ID="${HERMES_KANBAN_TASK}"
BRANCH="codex/$(printf '%s' "$TASK_ID" | tr -cd '[:alnum:]_-')/$(date -u +%Y%m%d%H%M%S)"
WORKTREE="/tmp/${TASK_ID}-codex-lane"
git fetch --all --prune
git worktree add -b "$BRANCH" "$WORKTREE" "$BASE"
```

## Prompt Construction

Every Codex prompt must include: task_id + acceptance criteria, repo/worktree/branch, ownership statement, safety constraints, verification commands. See `templates/pmb-codex-lane-prompt.md`.

## Reconciliation Checklist

- [ ] `git status --short` shows only expected files
- [ ] `git diff --stat` reviewed by Hermes
- [ ] No secrets/credentials/artifacts included
- [ ] Safety constraints preserved
- [ ] Hermes ran canonical tests independently
- [ ] Accepted commits applied to Hermes workspace

## Acceptance Outcomes

`accepted`, `partial`, `rejected`, `timed_out`
