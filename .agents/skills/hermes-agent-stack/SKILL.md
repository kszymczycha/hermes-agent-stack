---
name: hermes-agent-stack
description: Operate and upgrade a self-hosted Hermes Agent stack.
version: 1.0.0
platforms: [linux]
metadata:
  hermes:
    tags: [hermes, docker, compose, deployment, operations, raspberry-pi]
    category: devops
---

# Hermes Agent Stack Operations

## When to Use

Use this skill when the user wants to deploy, inspect, back up, troubleshoot, or
upgrade the Hermes Agent Docker Compose stack contained in this repository
(`hermes-agent-stack`).

Typical triggers:

- "deploy Hermes", "start the stack", "why is the container unhealthy?"
- "back up the agent", "upgrade Hermes", "change the model"
- "the dashboard is not reachable", "Telegram bot is not responding"
- Any work involving `docker-compose.yaml`, `$HERMES_DATA_DIR`, or the Hermes
  gateway container.

Do **not** use this skill to edit Hermes' application source code. The image is
immutable; upgrades happen by changing the image tag.

## Procedure

### 1. Inspect the current state

```bash
git status --short --branch
docker compose ps
docker compose config
```

Check `PLAN.md` for known work in progress. Read `.env.example` and the README
if the repository layout is unfamiliar.

### 2. First-time deployment

```bash
[ -f .env ] || cp .env.example .env
chmod 600 .env
# Fill in HERMES_DASHBOARD_PASSWORD and HERMES_DASHBOARD_SECRET.
# Set HERMES_UID / HERMES_GID to `id -u` / `id -g`.

DATA_DIR="$HOME/.hermes"   # or the value of HERMES_DATA_DIR
mkdir -p "$DATA_DIR/memories"
cp -n config.yaml.example "$DATA_DIR/config.yaml"
cp -n SOUL.md.example     "$DATA_DIR/SOUL.md"
cp -n memories/USER.md.example   "$DATA_DIR/memories/USER.md"
cp -n memories/MEMORY.md.example "$DATA_DIR/memories/MEMORY.md"

[ -e "$DATA_DIR/.env" ] || install -m 600 /dev/null "$DATA_DIR/.env"
# Add provider keys and bot tokens here, e.g.:
#   OPENROUTER_API_KEY=...
#   TELEGRAM_BOT_TOKEN=...
#   TELEGRAM_ALLOWED_USERS=...

docker compose pull
docker compose up -d
docker compose ps
```

`cp -n` and the `[ -e ... ]` guard keep existing state intact. Never overwrite a
live `config.yaml`, `SOUL.md`, memories, or data-dir `.env`.

Never put Hermes API keys in the repository `.env`; that file is only for
Compose interpolation.

### 3. Verify

```bash
docker compose ps                  # expect healthy after the start period
docker compose logs --tail=100
docker exec -it hermes hermes doctor
grep -A2 '^model:' "${DATA_DIR:-$HOME/.hermes}/config.yaml"
```

Expected provider block:

```yaml
model:
  provider: openrouter
  default: deepseek/deepseek-v4.1-flash-1017
```

The dashboard is reachable only on the host:

```text
http://127.0.0.1:9119
```

From another machine, ask the user to tunnel it:

```bash
ssh -L 9119:127.0.0.1:9119 user@host
```

### 4. Day-2 operations

```bash
docker compose logs -f --tail=100 hermes
docker compose restart hermes
docker exec -it hermes hermes doctor
docker exec -it hermes hermes logs --follow
```

`hermes chat` and `hermes model` are interactive and need a TTY. In a
non-interactive session, ask the user to run them.

Changing the model in `config.yaml` affects new sessions. Restart the gateway
to force all sessions to pick up the change.

### 5. Backup

Stop the container first so SQLite WAL state is consistent:

```bash
DATA_DIR="$HOME/.hermes"
docker compose stop hermes
tar -czf "hermes-backup-$(date +%F).tar.gz" \
  -C "$(dirname "$DATA_DIR")" "$(basename "$DATA_DIR")"
cp .env "hermes-compose-env-$(date +%F).backup"
docker compose start hermes
```

Keep backups off the host. Restore by stopping the stack, extracting the
archive over the data directory, and starting again.

### 6. Upgrade

```bash
# 1. Back up first (see above).
# 2. Update HERMES_IMAGE_TAG in .env to a release tag.
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 hermes
```

The image ships immutable code, so `hermes update` inside the container is not
the upgrade path. To roll back, restore the previous tag and repeat.

### 7. Security check

- `docker compose config` must show `127.0.0.1` for published ports.
- `grep -E 'PASSWORD|SECRET|API_KEY' .env` must not reveal real secrets in
  tracked files; `.env` is ignored by Git.
- `$HERMES_DATA_DIR/.env` must be mode `600`.
- `TELEGRAM_ALLOWED_USERS` must be set when Telegram is enabled.
- `config.yaml` uses `provider: openrouter` with
  `deepseek/deepseek-v4.1-flash-1017`, unless the user explicitly chose another
  provider.
- The API server stays disabled unless `API_SERVER_ENABLED=true`,
  `API_SERVER_KEY` is set, and the `8642` mapping was intentionally enabled.

## Pitfalls

- **Two `.env` files are intentional.** Deployment values go in the repo
  `.env`; Hermes secrets go in `$HERMES_DATA_DIR/.env`. Mixing them causes
  confusing auth/provider failures.
- **Dashboard auth is mandatory on a non-loopback bind.** Do not "fix" a 401 by
  exposing the port or disabling auth; fix the credentials or the tunnel.
- **Empty allowlists are dangerous.** An empty `TELEGRAM_ALLOWED_USERS` can let
  arbitrary Telegram users command the agent.
- **`HERMES_UID` / `HERMES_GID` must match the host user** that owns
  `$HERMES_DATA_DIR`; otherwise the container writes root-owned files.
- **Healthcheck needs a listener.** It passes when either `9119` (dashboard) or
  `8642` (API) is up. If both are disabled, the container reports unhealthy.
- **Never run two gateways against one data directory.** Session and memory
  files are not safe for concurrent writers.
- **Do not reintroduce removed provider examples.** The documented alternative
  is DeepSeek via OpenRouter:
  `provider: openrouter`, `deepseek/deepseek-v4.1-flash-1017`.
- **Do not use `latest`.** Pin a release tag and upgrade deliberately.
- **Do not commit live state.** `config.yaml`, `SOUL.md`, real memories and
  `$HERMES_DATA_DIR` are user data.

## Verification

The task is complete when:

- `docker compose config` resolves without errors.
- `docker compose ps` reports the container `healthy` (or the user has been
  told why it cannot be).
- `docker exec -it hermes hermes doctor` reports no blocking problems.
- The dashboard is reachable through loopback or the user's tunnel.
- Any config change is documented in `README.md` and, for backlog work, marked
  in `PLAN.md`.
