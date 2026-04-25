---
name: docker-project-isolation
description: Restrict an AI coding agent's filesystem access to a specific project folder using Docker, preventing accidental modification of files outside the project scope.
---

# Docker Project Isolation for AI Coding Agents

Use Docker to restrict an AI agent's filesystem access to a specific project folder, preventing accidental modification or deletion of files outside the project scope.

## Use Case
When you want an AI coding agent (Hermes, Claude Code, Codex, etc.) to work only within a specific project directory on macOS or Linux.

## Architecture
- **Image**: `node:24-bookworm` (Debian base, Node 24 LTS) — swap for your language image
- **Project mount**: read-write (`:rw`) by default — allows agent to modify code; Docker namespace provides isolation so container cannot see/access paths outside the mount
- **node_modules**: named volume to avoid ARM/Linux binary compatibility issues with host
- **pnpm store**: named volume to persist package cache across sessions

## Setup Files

### Dockerfile
```dockerfile
FROM node:24-bookworm
RUN npm install -g pnpm@latest
WORKDIR /app
CMD ["bash"]
```

### docker-compose.yml (read-only mode — for build/lint/test only)
```yaml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app:ro
      - node_modules:/app/node_modules
      - pnpm_store:/root/.local/share/pnpm
    environment:
      PNPM_CONFIRM: "0"
    stdin_open: true
    tty: true

volumes:
  node_modules:
  pnpm_store:
```

### docker-compose.dev.yml (read-write mode — for code modifications)
```yaml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app
      - node_modules:/app/node_modules
      - pnpm_store:/root/.local/share/pnpm
    environment:
      PNPM_CONFIRM: "0"
    stdin_open: true
    tty: true

volumes:
  node_modules:
  pnpm_store:
```

### docker-sh (entry script — read-write mode)
```bash
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
COMPOSE_FILE="$SCRIPT_DIR/docker-compose.dev.yml"
PROJECT_NAME="myproject-dev"

docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" build --quiet 2>/dev/null || true

if [ $# -eq 0 ]; then
  docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" run --rm dev
else
  docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" run --rm dev "$@"
fi
```
```bash
chmod +x docker-sh
```

### docker-sh.ro (entry script — read-only mode)
```bash
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
COMPOSE_FILE="$SCRIPT_DIR/docker-compose.yml"
PROJECT_NAME="myproject-ro"

docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" build --quiet 2>/dev/null || true

if [ $# -eq 0 ]; then
  docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" run --rm dev
else
  docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" run --rm dev "$@"
fi
```
```bash
chmod +x docker-sh.ro
```

## Usage
```bash
# Read-write mode (for code modifications)
./docker-sh pnpm install
./docker-sh pnpm lint --fix
./docker-sh pnpm build

# Read-only mode (for build/lint/test that don't modify files)
./docker-sh.ro pnpm build
./docker-sh.ro pnpm lint
```

## Critical Pitfalls Discovered

### 1. Multiple docker-compose files auto-merge
Docker Compose automatically picks up ALL `docker-compose*.yml` files in a directory and merges them. If you have both `docker-compose.yml` and `docker-compose.dev.yml`, Docker will try to merge them, causing conflicts and errors like "yaml: line 2: did not find expected key".

**Solution**: Always use `-p <unique-project-name>` flag and explicitly specify `-f <your-file>` when running docker compose. This prevents Docker from auto-discovering other compose files.

### 2. write_file tool can silently fail
The `write_file` tool may create files with 0 bytes in some environments (possibly due to permission issues or path problems). Always verify file contents after creation:
```bash
ls -la your-file.yml
wc -l your-file.yml
```

If a file is empty (0 bytes), recreate it using `terminal()` with a heredoc:
```bash
cat > your-file.yml << 'YAMLEOF'
# your content here
YAMLEOF
```

### 3. The :ro mount is often too restrictive
If you want the agent to modify code (which is usually the case), use read-write mount (`:rw` or just `:`). Docker's namespace isolation already prevents the container from accessing paths outside the mount — the `:ro` flag is only needed for specific cases like running CI/test pipelines.

## Key Insight
Docker provides namespace-level isolation — the container cannot see or access any path that is not explicitly mounted. This means:
- The container has no knowledge of `/Users/`, `/System/`, `/Applications/` on the host
- Even with `:rw` mount, the container cannot escape the `/app` directory
- Use `:ro` only when you explicitly want to prevent file modifications (e.g., CI pipelines)

The `workdir` parameter in sub-agents only restricts terminal commands — tools like `read_file()`, `write_file()`, `patch()` have no path restrictions. Docker isolation is the only reliable way to enforce filesystem boundaries across all tools.

## Adapt for Other Languages
Replace the Dockerfile base image and package manager installation:
- Python: `FROM python:3.12-slim` + `pip install uv`
- Go: `FROM golang:1.23-alpine` (no extra install needed)
- Rust: `FROM rust:1.75-slim`

## OrbStack Note
OrbStack is fully Docker-compatible. All commands work the same — just ensure the OrbStack Docker daemon is running (`docker ps` should work) before executing container commands.
