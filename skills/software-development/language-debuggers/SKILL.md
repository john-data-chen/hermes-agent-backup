---
name: language-debuggers
description: >-
  Debug Python and Node.js from the terminal. pdb/breakpoint(), debugpy remote
  attach, node inspect CLI, Chrome DevTools Protocol for Node. When console.log
  isn't enough.
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [debugging, python, nodejs, pdb, debugpy, inspector, breakpoints]
    related_skills: [systematic-debugging]
---

# Language Debuggers — Python & Node.js

When `print`/`console.log` isn't enough, use real debuggers. Pick by situation:

| Tool | Language | When |
|---|---|---|
| `breakpoint()` + pdb | Python | Local, interactive, simplest |
| `python -m pdb` | Python | Launch under pdb with no source edits |
| `debugpy` | Python | Remote/headless/attach to running process |
| `remote-pdb` | Python | Cleanest agent-friendly remote debug over netcat |
| `node inspect` | Node.js | Built-in, zero install, CLI REPL |
| CDP (`chrome-remote-interface`) | Node.js | Scriptable from terminal |

---

## Python: pdb Quick Reference

Inside any `(Pdb)` prompt:

| Command | Action |
|---|---|
| `n` | next line (step over) |
| `s` | step into |
| `r` | return from function |
| `c` | continue |
| `unt N` | continue until line N |
| `l` / `ll` | list source / full function |
| `w` | where (stack trace) |
| `u` / `d` | move up/down in stack |
| `p expr` / `pp expr` | print expression |
| `b file:line` | set breakpoint |
| `b func` | break on function entry |
| `cl N` | clear breakpoint N |
| `!stmt` | execute arbitrary Python |
| `interact` | full Python REPL in current scope |
| `q` | quit |

## Python: Recipe 1 — Local breakpoint

```python
breakpoint()  # drops into pdb here
```

Don't forget to remove before committing:
```bash
rg -n 'breakpoint\\(\\)' --type py
```

## Python: Recipe 2 — Launch under pdb

```bash
python -m pdb script.py
```

## Python: Recipe 3 — Debug a pytest test

```bash
# On failure:
scripts/run_tests.sh tests/test_file.py::test_name --pdb -p no:xdist

# At the START:
scripts/run_tests.sh tests/test_file.py::test_name --trace -p no:xdist
```

pdb does NOT work under xdist. Add `-p no:xdist`.

## Python: Recipe 4 — Post-mortem

```python
import pdb, sys
try:
    run_the_thing()
except:
    pdb.post_mortem(sys.exc_info()[2])
```

## Python: Recipe 5 — Remote debug with debugpy

```bash
pip install debugpy

# Source-edit approach:
import debugpy
debugpy.listen(("127.0.0.1", 5678))
debugpy.wait_for_client()

# or launch with -m:
python -m debugpy --listen 127.0.0.1:5678 --wait-for-client script.py

# Attach to running process:
python -m debugpy --listen 127.0.0.1:5678 --pid <pid>
```

## Python: Recipe 6 — remote-pdb (agent-friendly)

```bash
pip install remote-pdb
```

```python
from remote_pdb import set_trace
set_trace(host="127.0.0.1", port=4444)   # blocks until connection
```

Then: `nc 127.0.0.1 4444` for a pdb prompt. This is the cleanest agent-friendly choice.

## Python: Hermes-specific targets

- **Tests**: always `-p no:xdist` or single test with `-n 0`
- **tui_gateway subprocess**: remote-pdb at the RPC handler
- **_SlashWorker subprocess**: remote-pdb in the worker's exec path
- **Gateway (gateway/run.py)**: remote-pdb at a handler, or debugpy

## Python: Common Pitfalls

1. pdb under pytest-xdist silently hangs. Always `-p no:xdist`.
2. `breakpoint()` in CI hangs the process. Never commit it.
3. `PYTHONBREAKPOINT=0` disables all `breakpoint()` calls.
4. Attach to PID fails on hardened kernels (`ptrace_scope=1`).
5. pdb only debugs current thread. Use debugpy for multithreaded code.
6. asyncio: pdb works but `await` inside pdb needs Python 3.13+.
7. `scripts/run_tests.sh` strips credentials — debug with raw pytest first.
8. pdb doesn't follow forks. Each child needs its own `set_trace()`.

---

## Node.js: `node inspect` Quick Reference

Launch paused on first line:

```bash
node inspect path/to/script.js
node --inspect-brk $(which tsx) path/to/script.ts   # via tsx
```

Inside the `debug>` prompt:

| Command | Action |
|---|---|
| `c` / `cont` | continue |
| `n` / `next` | step over |
| `s` / `step` | step into |
| `o` / `out` | step out |
| `sb('file.js', 42)` | set breakpoint at line 42 |
| `sb(42)` | set breakpoint at line 42 of current file |
| `cb('file.js', 42)` | clear breakpoint |
| `bt` | backtrace (call stack) |
| `list(5)` | show 5 lines of source |
| `watch('expr')` | evaluate expr on every pause |
| `repl` | drop into REPL in current scope (Ctrl+C to exit) |
| `exec expr` | evaluate expression once |
| `restart` | restart script |
| `kill` | kill the script |

## Node.js: Attaching to a running process

```bash
# Enable inspector on existing process
kill -SIGUSR1 <pid>

# Attach
node inspect -p <pid>

# Or by URL
node inspect ws://127.0.0.1:9229/<uuid>

# Start with inspector from beginning
node --inspect script.js           # keep running
node --inspect-brk script.js       # pause on first line
```

## Node.js: Programmatic CDP

```bash
npm i -g chrome-remote-interface
```

Use the CDP driver script pattern — `Debugger.paused()` callback, `setBreakpointByUrl`, `evaluateOnCallFrame`. See `references/nodejs-cdp-driver.md` for a complete template.

## Node.js: Debugging Hermes ui-tui

```bash
cd <repo>/ui-tui
npm run build
node --inspect-brk dist/entry.js
# In another terminal:
node inspect -p <pid>
# Inside debug>:
sb('dist/app.js', 220)
cont
# At pause: repl → inspect props, state, etc.
```

For a running `hermes --tui`:
```bash
kill -SIGUSR1 $(pgrep -f 'ui-tui/dist/entry')
curl -s http://127.0.0.1:9229/json/list | jq -r '.[0].webSocketDebuggerUrl'
node inspect ws://...
```

## Node.js: Heap Snapshots & CPU Profiles

Via CDP driver:
- `Profiler.start()` → wait → `Profiler.stop()` → CPU `.cpuprofile`
- `HeapProfiler.takeHeapSnapshot()` → `.heapsnapshot`
Open in Chrome DevTools Performance/Memory tab.

## Node.js: Common Pitfalls

1. Wrong line numbers in TS source — break in built `dist/*.js` or enable sourcemaps.
2. `--inspect` vs `--inspect-brk` — use `-brk` to pause before any code runs.
3. Port collisions — default `9229`. Use `--inspect=0` for random port.
4. `--inspect` on parent does NOT inspect children.
5. `Ctrl+C` out of `node inspect` while target is paused → target stays paused.
6. Always bind to `127.0.0.1` unless you have an isolated network.
7. Enable sourcemaps: `node --enable-source-maps`.

## Verification Checklist (both languages)

- [ ] Debug port is actually listening: `curl -s http://127.0.0.1:9229/json/list` (Node) or `ss -tlnp | grep 5678` (Python)
- [ ] First breakpoint hits (if not, check `--inspect-brk`/`PYTHONBREAKPOINT=0`)
- [ ] `w`/`bt` shows expected call stack
- [ ] Post-debug cleanup: no stray `breakpoint()` / `set_trace()` committed
