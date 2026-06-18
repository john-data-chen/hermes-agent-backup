# Diagnostic Output Reference

Real error transcripts from a broken Hermes + fnm session.

## Problem 1: Hermes symlinks

```bash
$ ls -la ~/.local/bin/node
lrwxr-xr-x 1 johnchen staff 37 Jun 18 07:47 node -> /Users/johnchen/.hermes/node/bin/node

$ ~/.local/bin/node --version
v22.22.3   # Hermes' bundled Node, not fnm's
```

## Problem 2: Poisoned FNM_* vars

```bash
$ fish -c 'fnm current'
error: invalid value '' for '--node-dist-mirror <NODE_DIST_MIRROR>':
       relative URL without a base

$ fish -c 'set -S | grep FNM_'
$FNM_ARCH: set in global scope, exported, with 1 elements
$FNM_ARCH[1]: ||
$FNM_ARCH: originally inherited as ||
$FNM_COREPACK_ENABLED: ...
# ALL FNM_* vars show "originally inherited as ||" (empty)

$ fish -c 'echo $FNM_DIR'
(nothing — empty string)

$ fish -c 'echo $FNM_NODE_DIST_MIRROR'
(nothing — empty string)
```

## Root Cause Chain

1. User has fnm working with `conf.d/fnm.fish` that runs `fnm env --use-on-cd | source`
2. `fnm env` normally sets FNM_DIR, FNM_MULTISHELL_PATH, etc. to proper values
3. Hermes app launches, inherits FNM_* vars from parent environment
4. Hermes app (or its setup process) sets FNM_* to empty strings and exports them
5. Hermes in-app terminal (fish) inherits these empty vars
6. Fish `conf.d/fnm.fish` runs `fnm env`, but `fnm` binary reads empty FNM_* vars
   and treats them as user-specified values → CLI argument parsing fails
7. `fnm env` never produces output → FNM_MULTISHELL_PATH stays empty → fnm broken

## After Fix

```bash
$ fish -c 'fnm current'
v24.16.0

$ fish -c 'which node; node --version'
/Users/johnchen/.local/state/fnm_multishells/68758_1781741812117/bin/node
v24.16.0

$ fish -c 'fnm list'
* v24.16.0 default
* system
```
