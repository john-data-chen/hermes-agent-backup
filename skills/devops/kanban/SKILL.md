---
name: kanban
description: >-
  Two-role playbook for Hermes Kanban: orchestrator decomposition + worker
  lifecycle, pitfalls, and examples. Load when you need the deeper kanban
  workflow beyond the auto-injected KANBAN_GUIDANCE block.
version: 3.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [kanban, multi-agent, orchestration, routing, collaboration, pitfalls]
    related_skills: []
---

# Kanban — Orchestrator & Worker Playbook

Hermes Kanban has two roles. The orchestrator decomposes goals into tasks and routes them. The worker executes tasks and reports results. Both roles get the core lifecycle auto-injected as `KANBAN_GUIDANCE` from `agent/prompt_builder.py`; this skill is the deeper detail.

---

## Part 1: Orchestrator — Decomposition & Routing

> The core worker lifecycle (including the `kanban_create` fan-out pattern and the "decompose, don't execute" rule) is auto-injected into every kanban process. This section is the deeper playbook for the orchestrator role.

### Step 0: Discover available profiles before planning

Hermes setups vary. There is no default specialist roster. The dispatcher silently fails to spawn unknown assignee names.

```bash
hermes profile list   # discover what's available
```

Or just ask the user: "What profiles do you have set up?"

Cache the result for the conversation. Re-asking wastes tool calls.

### When to use the board (vs. just doing the work)

Create Kanban tasks when any are true:
1. Multiple specialists needed
2. Work should survive crash/restart
3. User might want to interject
4. Multiple subtasks can run in parallel
5. Review/iteration expected
6. Audit trail matters

### The anti-temptation rules

- Do not execute the work yourself.
- For any concrete task, create a Kanban task and assign it.
- Split multi-lane requests before creating cards.
- Run independent lanes in parallel (no parent links).
- Never create dependent work as independent ready cards — use `parents=[...]`.
- If no specialist fits, ask the user.

### Decomposition playbook

1. **Understand the goal** — ask clarifying questions.
2. **Sketch the task graph** — extract lanes, map to profiles, decide dependencies.
3. **Create tasks and link** — parent first, capture IDs, pass to children:
   ```python
   t1 = kanban_create(title="research: cost", assignee="<profile>")["task_id"]
   t2 = kanban_create(title="research: perf", assignee="<profile>")["task_id"]
   t3 = kanban_create(title="synthesize", assignee="<profile>", parents=[t1, t2])["task_id"]
   ```
4. **Complete your own task** with a summary of what you created.
5. **Report back** to the user in plain prose.

### Goal-mode cards (persistent workers)

For open-ended cards, pass `goal_mode=True` to wrap the worker in a Ralph-style goal loop:
- Judge re-checks after each turn
- Worker keeps going in same session (full context retained)
- Budget exhausted → blocked for human review

### Recovering stuck workers

When a worker crashes/hallucinates:
1. **Reclaim** (`hermes kanban reclaim <task_id>`) — abort and reset to ready
2. **Reassign** (`hermes kanban reassign <task_id> <new-profile> --reclaim`)
3. **Change profile model** — edit profile config, then reclaim

---

## Part 2: Worker — Pitfalls & Examples

> Auto-loaded for every dispatched worker. The lifecycle (6 steps: orient → work → heartbeat → block/complete) is in KANBAN_GUIDANCE. This is the deeper detail.

### Workspace handling

| Kind | What it is | How to work |
|---|---|---|
| `scratch` | Fresh tmp dir | Read/write freely; GC'd on archive |
| `dir:<path>` | Shared persistent dir | Treat like long-lived state |
| `worktree` | Git worktree | Commit work here; create via `git worktree add` if missing |

### Tenant isolation

If `$HERMES_TENANT` is set, prefix memory entries: `tenant: value`.

### Good summary + metadata shapes

**Coding task:**
```python
kanban_complete(
    summary="shipped rate limiter — token bucket, 14 tests pass",
    metadata={"changed_files": ["rate_limiter.py"], "tests_run": 14},
)
```

**Research task:**
```python
kanban_complete(
    summary="3 libraries reviewed; vLLM wins on throughput",
    metadata={"recommendation": "vLLM", "benchmarks": {}},
)
```

**Review task (prefer block with review-required):**
```python
kanban_comment(body=json.dumps({"changed_files": ["..."]}, indent=2))
kanban_block(reason="review-required: rate limiter needs eyes before merge")
```

### Claiming cards you created

Pass `created_cards=[c1["task_id"], c2["task_id"]]` on `kanban_complete`. Only list IDs you captured from successful `kanban_create` return values — the kernel verifies each.

### Block reasons that get answered fast

Bad: `"stuck"`. Good: one sentence naming the specific decision needed. Leave context in a comment.

```python
kanban_comment(body="Full context: ...")
kanban_block(reason="Rate limit key choice: IP or user_id?")
```

### Heartbeats worth sending

Name progress: `"epoch 12/50, loss 0.31"`, `"scanned 1.2M/2.4M rows"`. Skip for tasks under ~2 minutes.

### Retry scenarios

Check `kanban_show` for prior `runs`:
- `timed_out` — chunk or shorten work
- `crashed` — reduce memory
- `spawn_failed` — config issue; ask human
- `reclaimed` — operator archived task; check status
- `blocked` — read the unblock comment

### Do NOT

- Call `delegate_task` as a substitute for `kanban_create`
- Call `clarify` — use `kanban_block` instead
- Modify files outside `$HERMES_KANBAN_WORKSPACE`
- Complete a task you didn't finish — block it
- Create follow-up tasks assigned to yourself

### Pitfalls

- Task state can change between dispatch and startup. Always `kanban_show` first.
- Workspace may have stale artifacts (especially `dir:` / `worktree`).
- Use the `kanban_*` tools, not the CLI (CLI won't exist in containerized backends).

### CLI fallback (for scripting)

- `kanban_show` ↔ `hermes kanban show <id> --json`
- `kanban_complete` ↔ `hermes kanban complete <id> --summary "..." --metadata '{...}'`
- `kanban_block` ↔ `hermes kanban block <id> "reason"`
- `kanban_create` ↔ `hermes kanban create "title" --assignee <profile> [--parent <id>]`
