---
name: hermes-node-manager-fix
description: >
  Fix Node.js version managers (fnm, nvm) broken by Hermes Agent installation.
  Hermes installs its own Node.js and may leak empty FNM_* env vars that break
  version managers. Use when user says "Hermes broke fnm", "node version wrong
  after Hermes install", "fnm not working after Hermes", or "can't switch node
  versions since installing Hermes".
version: 1.0.0
---

# Hermes Node Manager Fix

Hermes Agent installation can break pre-existing Node.js version managers
(fnm, nvm, volta, asdf-nodejs) in two ways. This skill diagnoses and fixes both.

## Triggers

- "Hermes broke fnm/nvm/node"
- "Can't switch Node versions since installing Hermes"
- "fnm use not working", "nvm use broken"
- `node --version` shows wrong version after Hermes install
- "node.js 版本無法使用 fnm 的設定"

## Diagnostic Checklist

Run these first to confirm the issue:

```bash
# 1. Check which node resolves
which node
# If → /Users/$USER/.local/bin/node (Hermes' node) → Problem 1

# 2. Check fnm in fish subshell (or bash/zsh)
fish -c 'fnm current'
# If → "relative URL without a base" or similar empty-var error → Problem 2
# If → "a value is required for '--fnm-dir'" → Problem 2

# 3. Check for poisoned FNM_* env vars
fish -c 'set -S | grep FNM_'
# If vars show "originally inherited as ||" (empty) → Problem 2
```

## Problem 1: Hermes Node Symlinks in ~/.local/bin

**Symptom:** `which node` → `/Users/$USER/.local/bin/node` (v22.x, Hermes' bundled Node)

**Cause:** Hermes installer creates `node`, `npm`, `npx` symlinks in
`~/.local/bin/` pointing to `~/.hermes/node/bin/`. This shadows the
version manager's shim.

**Fix:** Remove the Hermes symlinks.

```bash
rm ~/.local/bin/node ~/.local/bin/npm ~/.local/bin/npx
```

Verify: `ls ~/.local/bin/node` should say "No such file".

After removal, the version manager's shim (fnm multishell path, nvm shim,
etc.) will take priority.

## Problem 2: Poisoned FNM_* Environment Variables

**Symptom:** `fnm` commands fail with errors about empty values:
- "relative URL without a base" (FNM_NODE_DIST_MIRROR empty)
- "a value is required for '--fnm-dir'" (FNM_DIR empty)
- "a value is required for '--log-level'" (FNM_LOGLEVEL empty)

**Cause:** Hermes app inherits FNM_* vars from parent environment and
re-exports them as empty strings. These empty values propagate to child
shells and break fnm's CLI argument parsing.

**Fix for fish shell:** Edit `~/.config/fish/conf.d/fnm.fish` (create if
missing). Add `set -e` for ALL FNM_* vars BEFORE sourcing fnm env:

```fish
# Clear inherited empty FNM vars (set by Hermes app env)
set -e FNM_DIR FNM_NODE_DIST_MIRROR FNM_LOGLEVEL FNM_ARCH \
       FNM_COREPACK_ENABLED FNM_RESOLVE_ENGINES \
       FNM_VERSION_FILE_STRATEGY FNM_MULTISHELL_PATH
fnm env --use-on-cd --shell fish | source
```

If `conf.d/fnm.fish` doesn't exist, create it with the above content.

**Fix for bash/zsh:** Add to `~/.bashrc` / `~/.zshrc`:

```bash
unset FNM_DIR FNM_NODE_DIST_MIRROR FNM_LOGLEVEL FNM_ARCH \
      FNM_COREPACK_ENABLED FNM_RESOLVE_ENGINES \
      FNM_VERSION_FILE_STRATEGY FNM_MULTISHELL_PATH
eval "$(fnm env --use-on-cd)"
```

**Fix for nvm users (bash/zsh):** Similar pattern — unset any NVM_*
vars before sourcing nvm.sh:

```bash
unset NVM_DIR NVM_CD_FLAGS NVM_BIN NVM_INC
source ~/.nvm/nvm.sh
```

## Verification

After applying fixes, open a NEW terminal and verify:

```bash
fnm current          # Should show your default version, no errors
which node           # Should show fnm multishell path (not ~/.local/bin)
node --version       # Should match fnm default
fnm use <other_ver>  # Should switch versions
```

## Pitfalls

- **Don't delete ~/.hermes/node/.** Hermes needs its own Node to run.
  Only remove the symlinks in `~/.local/bin/`.

- **Hermes updates may recreate symlinks.** If Hermes auto-updates and
  `~/.local/bin/node` reappears, re-apply Problem 1 fix.

- **Environment variables are sticky.** Exporting a variable as empty
  is different from not setting it. fnm treats "set to empty" as "user
  explicitly set this" and tries to use the empty value. Must `unset`/
  `set -e`, not just set to default.

- **Only affects Hermes in-app terminal and child processes.** External
  terminals (Terminal.app, Ghostty, iTerm2) launched outside Hermes are
  unaffected by Problem 2, but Problem 1 (symlinks) affects all terminals
  because `~/.local/bin` is in system PATH.

## NVM-Specific Notes

If using nvm instead of fnm, the same diagnosis applies. Check:
- `which node` for `~/.local/bin/node`
- `nvm use` for errors about empty NVM_DIR

Fix is the same: remove Hermes symlinks + unset empty env vars before
sourcing nvm.
