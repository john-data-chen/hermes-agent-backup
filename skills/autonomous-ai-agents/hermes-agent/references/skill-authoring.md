# Skill Authoring for Hermes Agent (In-Repo)

## Overview

There are two places a SKILL.md can live:

1. **User-local:** `~/.hermes/skills/<maybe-category>/<name>/SKILL.md` — personal, not shared. Created via `skill_manage(action='create')`.
2. **In-repo:** `skills/<category>/<name>/SKILL.md` — committed, shipped with the package. Use `write_file` + `git add`. `skill_manage(action='create')` does NOT target this tree.

## When to Use

- User asks you to add a skill "in this branch / repo / commit"
- You're committing a reusable workflow that should ship with hermes-agent
- You're editing an existing skill under `skills/` (use `patch` for small edits, `write_file` for rewrites)

## Required Frontmatter

Source of truth: `tools/skill_manager_tool.py::_validate_frontmatter`. Hard requirements:

- Starts with `---` as the first bytes (no leading blank line).
- Closes with `\n---\n` before the body.
- Parses as a YAML mapping.
- `name` field present.
- `description` field present, ≤ **1024 chars** (`MAX_DESCRIPTION_LENGTH`).
- Non-empty body after the closing `---`.

Peer-matched shape:

```yaml
---
name: my-skill-name               # lowercase, hyphens, ≤64 chars (MAX_NAME_LENGTH)
description: Use when <trigger>. <one-line behavior>.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [short, descriptive, tags]
    related_skills: [other-skill, another-skill]
---
```

`version` / `author` / `license` / `metadata` are NOT enforced by the validator, but every peer has them.

## Size Limits

- Description: ≤ 1024 chars (enforced).
- Full SKILL.md: ≤ 100,000 chars (enforced as `MAX_SKILL_CONTENT_CHARS`, ~36k tokens).
- Peer skills sit at **8-14k chars**. If pushing past 20k, split into `references/*.md`.

## Peer-Matched Structure

```markdown
# <Title>

## Overview
One or two paragraphs: what and why.

## When to Use
- Bulleted triggers
- "Don't use for:" counter-triggers

## <Topic sections specific to the skill>
- Quick-reference tables
- Code blocks with exact commands
- Testing recipes (scripts/run_tests.sh, etc.)
- Hermes-specific paths

## Common Pitfalls
Numbered list of mistakes and fixes.

## Verification Checklist
- [ ] Checkbox list of post-action verifications
```

`Overview` + `When to Use` + actionable body + pitfalls are the minimum.

## Directory Placement

```
skills/<category>/<skill-name>/SKILL.md
```

Categories currently: `autonomous-ai-agents`, `creative`, `data-science`, `devops`, `dogfood`, `email`, `gaming`, `github`, `leisure`, `mcp`, `media`, `mlops/*`, `note-taking`, `productivity`, `red-teaming`, `research`, `smart-home`, `social-media`, `software-development`.

Pick the closest existing category.

## Cross-Referencing Other Skills

`metadata.hermes.related_skills` unions both trees at load time. You CAN reference a user-local skill from an in-repo skill, but it won't resolve for other users who clone the repo fresh. Prefer referencing only in-repo skills.

## Workflow

1. Survey peers in the target category.
2. Check validator constraints in `tools/skill_manager_tool.py`.
3. Draft with `write_file` to `skills/<category>/<name>/SKILL.md`.
4. Validate locally (check frontmatter, description length, total size).
5. `git add + commit` on the active branch.

## Common Pitfalls

1. **Using `skill_manage(action='create')` for an in-repo skill.** It writes to `~/.hermes/skills/`, not the repo tree. Use `write_file`.
2. **Leading whitespace before `---`.** The validator checks `content.startswith("---")`.
3. **Description too generic.** Start with "Use when ..." and describe the *trigger class*.
4. **Forgetting the author/license/metadata block.** Not validator-enforced, but every peer has it.
5. **Writing a skill that duplicates a peer.** Prefer extending an existing skill.
6. **Expecting the current session to see the new skill.** The skill loader is initialized at session start. Verify in a fresh session.
7. **Linking to skills that don't exist in-repo.** Prefer only in-repo links.

## Verification Checklist

- [ ] File is at `skills/<category>/<name>/SKILL.md` (not in `~/.hermes/skills/`)
- [ ] Frontmatter starts at byte 0 with `---`
- [ ] `name`, `description`, `version`, `author`, `license`, `metadata.hermes.{tags, related_skills}` all present
- [ ] Name ≤ 64 chars, lowercase + hyphens
- [ ] Description ≤ 1024 chars and starts with "Use when ..."
- [ ] Total file ≤ 100,000 chars
- [ ] Structure: `# Title` → `## Overview` → `## When to Use` → body → pitfalls → checklist
- [ ] `git add skills/<category>/<name>/ && git commit` completed
