# Hermes s6-overlay Container Supervision

## Architecture

```
/init                                  ← PID 1 (s6-overlay v3.2.3.0)
├── cont-init.d                        ← oneshot setup, runs as root
│   ├── 01-hermes-setup                ← docker/stage2-hook.sh
│   │   ├── UID/GID remap
│   │   ├── chown /opt/data
│   │   ├── seed .env / config.yaml / SOUL.md
│   │   └── skills_sync.py
│   └── 02-reconcile-profiles          ← hermes_cli.container_boot
│       └── recreate /run/service/gateway-<name>/
│
├── s6-rc.d (static services)
│   ├── main-hermes/run                ← exec sleep infinity (no-op)
│   └── dashboard/run                  ← if HERMES_DASHBOARD=1
│
├── /run/service (s6-svscan; tmpfs)
│   └── gateway-<name>/
│       ├── type ("longrun")
│       ├── run ("exec s6-setuidgid hermes hermes -p <name> gateway run")
│       └── log/run (s6-log → $HERMES_HOME/logs/gateways/<name>/current)
│
└── CMD → /opt/hermes/docker/main-wrapper.sh
```

## Key files

| Path | Role |
|------|------|
| `Dockerfile` | s6-overlay install + cont-init.d wiring + ENTRYPOINT |
| `docker/stage2-hook.sh` | UID remap, chown, seed, skills sync. Runs as cont-init.d/01-hermes-setup |
| `docker/cont-init.d/02-reconcile-profiles` | Restores profile gateway slots from persistent volume |
| `docker/main-wrapper.sh` | Routes user args to hermes via `s6-setuidgid` |
| `docker/s6-rc.d/main-hermes/run` | No-op `sleep infinity` — main hermes runs as CMD |
| `docker/s6-rc.d/dashboard/run` | Conditional service gated on `HERMES_DASHBOARD` |
| `hermes_cli/service_manager.py` | `S6ServiceManager`: register/unregister/start/stop profile gateways |
| `hermes_cli/container_boot.py` | `reconcile_profile_gateways()` regenerates s6 slots |

## Why Architecture B (CMD as main program, not s6-supervised)

1. **cont-init.d scripts receive no CMD args** — can't parse `docker run <image> chat -q "hi"` inside a service run script.
2. **`/run/s6/basedir/bin/halt` does NOT propagate exit code** — containers always exit 143. Confirmed by skarnet.

So: `ENTRYPOINT ["/init", "/opt/hermes/docker/main-wrapper.sh"]`. User args land on the wrapper, not intercepted by /init. The wrapper drops to hermes via `s6-setuidgid` and exec's the program. Exit code propagates normally.

## Quick recipes

### Verify s6 is PID 1
```sh
docker exec <c> sh -c 'cat /proc/1/comm; readlink /proc/1/exe'
```

### Inspect profile gateway service
```sh
docker exec <c> /command/s6-svstat /run/service/gateway-<name>
# "up (pid …)" → running
# "down (exitcode 1) … normally up, want up" → crash loop
```

### Bring a service up/down manually
```sh
docker exec <c> /command/s6-svc -u /run/service/gateway-<name>   # up
docker exec <c> /command/s6-svc -d /run/service/gateway-<name>   # down
docker exec <c> /command/s6-svc -t /run/service/gateway-<name>   # SIGTERM (restart)
```

### Watch cont-init reconciler log
```sh
docker exec <c> tail -n 50 /opt/data/logs/container-boot.log
```

### Add a new static service
1. Create `docker/s6-rc.d/<name>/type` with `longrun\n` and `docker/s6-rc.d/<name>/run`
2. Drop to hermes via `s6-setuidgid hermes`
3. Create empty `docker/s6-rc.d/<name>/dependencies.d/base`
4. Create empty `docker/s6-rc.d/user/contents.d/<name>`
5. The Dockerfile `COPY` picks it up automatically

### Change per-profile gateway run command
Edit `S6ServiceManager._render_run_script` in `hermes_cli/service_manager.py`.

### Run docker test harness
```sh
docker build -t hermes-agent-harness:latest .
HERMES_TEST_IMAGE=hermes-agent-harness:latest scripts/run_tests.sh tests/docker/ -v
```

## Common pitfalls

- **`/command/` not on `docker exec` PATH** — always use absolute path (`/command/s6-svstat`)
- **Profile directory ownership** — cont-init reconciler runs as hermes. `stage2-hook.sh` chowns profiles on every boot.
- **Files written by `docker exec` are root-owned** — pass `--user hermes` or rely on boot sweep.
- **"s6-supervise not running"** — service dir is on tmpfs, wiped on restart. Check reconciler log.
- **Gateway starts then immediately exits** — profile has no model/auth configured. Run `hermes -p <profile> setup` first.
- **Reconciler skipped a profile** — keys on presence of `SOUL.md`. Add one to opt back in.
- **Container exits 143** — CMD must exit normally. Don't invoke halt or s6-svscanctl.
