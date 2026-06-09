# Debugging Hermes TUI Slash Commands

## Overview

Hermes slash commands span three layers — Python command registry, tui_gateway JSON-RPC bridge, and the Ink/TypeScript frontend. When a command misbehaves (missing from autocomplete, works in CLI but not TUI, config persists but UI doesn't update), the bug is almost always one layer being out of sync with another.

## Architecture

```
Python backend (hermes_cli/commands.py)     <- canonical COMMAND_REGISTRY
       │
       ▼
TUI gateway (tui_gateway/server.py)         <- slash.exec / command.dispatch
       │
       ▼
TUI frontend (ui-tui/src/app/slash/)        <- local handlers + fallthrough
```

Command definitions must be registered consistently across Python and TypeScript. The Python `COMMAND_REGISTRY` is the source of truth for: CLI dispatch, gateway help, Telegram BotCommand menu, Slack subcommand map, and autocomplete data shipped to Ink.

## Investigation Steps

1. **Check if the command exists in the TUI frontend:**
   ```bash
   search_files --pattern "/commandname" --file_glob "*.ts" --path ui-tui/
   search_files --pattern "/commandname" --file_glob "*.tsx" --path ui-tui/
   ```

2. **Examine the TUI command definition:**
   ```bash
   read_file ui-tui/src/app/slash/commands/core.ts
   ```

3. **Check if the command exists in the Python backend:**
   ```bash
   search_files --pattern "CommandDef" --file_glob "*.py" --path hermes_cli/
   search_files --pattern "commandname" --path hermes_cli/commands.py --context 3
   ```

4. **Examine the gateway implementation:**
   ```bash
   search_files --pattern "complete.slash|slash.exec" --path tui_gateway/
   ```

## Fix: Missing Command Autocomplete

1. Add a `CommandDef` entry to `COMMAND_REGISTRY` in `hermes_cli/commands.py`:
   ```python
   CommandDef("commandname", "Description", "Session",
              cli_only=True, aliases=("alias",),
              args_hint="[arg1|arg2|arg3]",
              subcommands=("arg1", "arg2", "arg3")),
   ```
2. Pick `cli_only` vs gateway availability carefully.
3. Ensure `subcommands` matches the TUI's tab-completion options.
4. Add handler in `cli.py::process_command()` and/or `gateway/run.py`.

## Common Issues

1. **Command shows in TUI but not in autocomplete.** Missing from `COMMAND_REGISTRY`.
2. **Command shows in autocomplete but doesn't work.** Check `tui_gateway/server.py` and the frontend handler.
3. **Command behavior differs between CLI and TUI.** Different implementations in `cli.py::process_command` vs TUI's local handler.
4. **Command persists config but doesn't apply live.** Must patch nanostore state immediately (`patchUiState(...)`), not just `config.set`.
5. **Gateway dispatch silently ignores the command.** Check `GATEWAY_KNOWN_COMMANDS` includes the canonical name.

## Debugging Tactics

- **Python side:** use `python-debugpy` skill to break inside `_SlashWorker.exec`
- **Ink side:** use `node-inspect-debugger` skill to break in `app.tsx`
- **Registry mismatch:** compare canonical `COMMAND_REGISTRY` against TUI's local command list

## Pitfalls

- Set the appropriate category for the command in `CommandDef`.
- Make sure aliases are properly registered in the `aliases` tuple.
- For commands with subcommands, ensure `subcommands` matches TUI code.
- `cli_only=True` commands won't work in gateway — unless `gateway_config_gate` is truthy.
- After adding live UI state, thread through all render paths (live `StreamingAssistant`/`ToolTrail` and transcript/pending `MessageLine` rows).
- Rebuild the TUI (`npm --prefix ui-tui run build`) before testing.

## Verification

1. Rebuild the TUI: `cd <repo> && npm --prefix ui-tui run build`
2. Run TUI: `hermes --tui`
3. Type `/` and verify command appears in autocomplete
4. Execute and confirm behavior + config update + live UI state
5. For gateway-available commands: test from messaging platform or run gateway tests
