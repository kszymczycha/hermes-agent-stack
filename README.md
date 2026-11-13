# Hermes Agent — Local Deployment Stack

A self-contained Docker Compose setup for running [Hermes Agent](https://github.com/NousResearch/hermes-agent) (by Nous Research) on your own hardware — a Raspberry Pi 4, a home server, or a small VPS.

The stack runs the messaging gateway (Telegram, Discord, Slack, …), the web dashboard, and an optional OpenAI-compatible API server. All agent state lives in a single directory on the host, so upgrading the container never loses configuration, memory, or sessions.

## What this stack gives you

- **Pinned image tag** — reproducible deploys instead of `latest`.
- **Single source of truth** — `config.yaml`, `.env`, `SOUL.md`, memories, sessions and logs in `$HERMES_DATA_DIR` (default `~/.hermes`).
- **Safer defaults** — dashboard bound to loopback, mandatory basic auth, API server disabled unless you opt in.
- **Operational basics** — healthcheck, log rotation, memory/CPU limits, graceful shutdown.
- **Templates** for `config.yaml`, `SOUL.md`, `USER.md` and `MEMORY.md`.

## Repository layout

```text
.
├── docker-compose.yaml     # Container definition (pinned image, limits, healthcheck)
├── .env.example            # Deployment variables for Compose (copy to .env)
├── config.yaml.example     # Agent config          -> $HERMES_DATA_DIR/config.yaml
├── SOUL.md.example         # Agent identity        -> $HERMES_DATA_DIR/SOUL.md
└── memories/
    ├── USER.md.example     # User profile          -> $HERMES_DATA_DIR/memories/USER.md
    └── MEMORY.md.example   # Environment facts     -> $HERMES_DATA_DIR/memories/MEMORY.md
```

Runtime state is **not** stored in this repository. By default it lives in `~/.hermes`:

```text
$HERMES_DATA_DIR/
├── config.yaml       # All agent settings (model, tools, compression, …)
├── .env              # Hermes secrets: LLM API keys, bot tokens
├── SOUL.md           # Agent identity (slot #1 in the system prompt)
├── memories/         # USER.md, MEMORY.md — durable memory
├── sessions/         # Conversation history
├── skills/           # Agent-created skills
├── home/             # HOME for tool subprocesses (git, ssh, gh, npm, …)
├── logs/             # Rotated Hermes logs
└── state.db          # SQLite session database
```

> **Two `.env` files, two jobs.** The repo-level `.env` holds deployment settings for Compose (dashboard password, data path, resource limits). The data-dir `.env` holds Hermes secrets and is read by the agent inside the container at `/opt/data/.env`. Do not mix them.

---

## Prerequisites

- Docker Engine and Docker Compose v2 (`docker compose version`).
- An LLM API key supported by Hermes — for example `OPENAI_API_KEY` (provider `openai-api`) or `DEEPSEEK_API_KEY` (provider `deepseek`). Any provider listed in the [Hermes docs](https://hermes-agent.nousresearch.com/docs/integrations/providers) works.
- Optional: a Telegram bot token from [@BotFather](https://t.me/BotFather).
- On a Raspberry Pi: a 64-bit OS (arm64), at least 2 GB of free RAM and ~2 GB of disk for the image. 4 GB RAM is comfortable.

---

## Quick start

### 1. Clone the repository

```bash
git clone git@github.com:kszymczycha/hermes-agent-stack.git
cd hermes-agent-stack
```

### 2. Create the Compose environment file

```bash
cp .env.example .env
chmod 600 .env
```

Edit `.env` and set at least:

| Variable | Purpose |
| --- | --- |
| `HERMES_DASHBOARD_PASSWORD` | Dashboard login password (required). |
| `HERMES_DASHBOARD_SECRET` | Session signing secret. Generate with `openssl rand -hex 32`. |
| `HERMES_UID` / `HERMES_GID` | Host UID/GID that should own the data directory (`id -u`, `id -g`). |
| `HERMES_DATA_DIR` | Optional absolute data path. Defaults to `$HOME/.hermes`. |
| `HERMES_IMAGE_TAG` | Pinned Hermes release (default `v0.23.1`). |
| `HERMES_MEMORY_LIMIT` / `HERMES_CPUS` | Container limits (default `3G` / `2.0`). |

### 3. Prepare the data directory from the templates

```bash
# If you changed HERMES_DATA_DIR in .env, use that path here instead.
DATA_DIR="$HOME/.hermes"

mkdir -p "$DATA_DIR/memories"
cp config.yaml.example "$DATA_DIR/config.yaml"
cp SOUL.md.example     "$DATA_DIR/SOUL.md"
cp memories/USER.md.example   "$DATA_DIR/memories/USER.md"
cp memories/MEMORY.md.example "$DATA_DIR/memories/MEMORY.md"
```

Then edit the copies — the templates contain placeholders:

- `config.yaml` — set the provider and model.
- `SOUL.md` — the agent's personality and operating principles.
- `memories/USER.md` — your stable preferences and context.
- `memories/MEMORY.md` — hostnames, network topology, services, conventions.

> The agent reads these files at startup. Changes made while the container is running require a restart (`docker compose restart hermes`) to take effect.

### 4. Add Hermes secrets

Create the data-dir `.env` and fill in your keys:

```bash
install -m 600 /dev/null "$DATA_DIR/.env"
$EDITOR "$DATA_DIR/.env"
```

Minimum contents for an OpenAI + Telegram setup:

```dotenv
OPENAI_API_KEY=sk-...
# DEEPSEEK_API_KEY=...

TELEGRAM_BOT_TOKEN=123456789:AA...
# Comma-separated numeric Telegram user IDs allowed to talk to the bot.
TELEGRAM_ALLOWED_USERS=111111111
```

`.env` is read by the agent at startup; restart the container after editing it.

> **Alternative:** instead of the manual steps above, run the interactive setup wizard, which writes `config.yaml`, `.env` and `SOUL.md` into the data directory for you:
>
> ```bash
> docker run -it --rm \
>   -v "$DATA_DIR:/opt/data" \
>   -e HERMES_UID="$(id -u)" \
>   -e HERMES_GID="$(id -g)" \
>   nousresearch/hermes-agent:v0.23.1 setup
> ```
>
> Keep the tag in sync with `HERMES_IMAGE_TAG` in `.env`.

### 5. Start the stack

```bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f --tail=100
```

The first boot can take a minute or two (volume ownership fix, config migration, skill sync). `docker compose ps` reports `healthy` once the dashboard is listening.

### 6. Open the dashboard

The dashboard is published on **loopback only** (`127.0.0.1:9119`). From the machine running Docker, open:

```text
http://127.0.0.1:9119
```

From another computer, use an SSH tunnel:

```bash
ssh -L 9119:127.0.0.1:9119 user@your-server
# then open http://127.0.0.1:9119 locally
```

Log in with `HERMES_DASHBOARD_USER` / `HERMES_DASHBOARD_PASSWORD` from `.env`.

To expose the dashboard on a LAN or the internet, put a TLS-terminating reverse proxy (Caddy, nginx, Traefik) in front of it and configure `dashboard.public_url` plus a bounded `dashboard.trusted_proxies` list in `config.yaml`. Do **not** change the port mapping to `0.0.0.0:9119:9119` — basic auth over plain HTTP leaks credentials.

---

## Configuration

Hermes resolves settings in this order (highest priority first):

1. CLI flags (per command).
2. `$HERMES_DATA_DIR/config.yaml` — non-secret settings.
3. `$HERMES_DATA_DIR/.env` — secrets and environment variables.
4. Built-in defaults.

### Model example

```yaml
model:
  provider: openai-api      # openai-api | deepseek | openrouter | anthropic | ...
  default: gpt-5-nano       # example only; use a model your key can access
```

For DeepSeek:

```yaml
model:
  provider: deepseek
  default: deepseek/deepseek-v4.1-flash-1017
```

Provider IDs and model catalogues change over time. The reliable way to pick a model is the dashboard's **Models** page or, inside the container:

```bash
docker exec -it hermes hermes model
```

`hermes model` is the full provider wizard; `/model` inside a chat only switches between providers you have already configured.

### Dynamic model switches via Telegram

```text
/model <model-id> --provider openai-api --global
/model deepseek/deepseek-v4.1-flash-1017 --provider deepseek --global
```

`--global` persists the choice to `config.yaml`; without it, the switch applies only to the current session.

---

## Telegram

1. Create a bot with [@BotFather](https://t.me/BotFather) and put the token in `$HERMES_DATA_DIR/.env` as `TELEGRAM_BOT_TOKEN`.
2. Put your numeric Telegram user ID in `TELEGRAM_ALLOWED_USERS` (comma-separated for multiple people). Find your ID with [@userinfobot](https://t.me/userinfobot).
3. Restart the stack: `docker compose restart hermes`.

> **Allowlist matters.** If `TELEGRAM_ALLOWED_USERS` is empty, the bot may accept commands from any Telegram user who finds it. Always set it explicitly.

For groups, use the dedicated Hermes group settings (`TELEGRAM_GROUP_ALLOWED_USERS`, etc.) documented in the [messaging docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/).

---

## Optional: OpenAI-compatible API server

The API server (port `8642`) is **disabled by default**. To enable it:

1. In `.env`, set:
   ```dotenv
   API_SERVER_ENABLED=true
   API_SERVER_KEY=<at least 8 characters, e.g. openssl rand -hex 32>
   ```
2. Uncomment the `8642` mapping in `docker-compose.yaml`:
   ```yaml
   - "127.0.0.1:8642:8642"
   ```
3. Recreate the container:
   ```bash
   docker compose up -d
   ```

Keep the API bound to loopback. If you must expose it, terminate TLS and require the bearer key at a reverse proxy. The endpoint is documented in the [API server docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server).

---

## Operations

### Logs and diagnostics

```bash
docker compose ps
docker compose logs -f --tail=100 hermes
docker exec -it hermes hermes doctor
docker exec -it hermes hermes logs --follow
```

Hermes also writes rotated per-profile logs to `$HERMES_DATA_DIR/logs/` (for example `logs/gateways/default/current`), which survive container restarts. Docker's own `json-file` logs are capped at 3 × 10 MB by this stack.

### Interactive chat

```bash
docker exec -it hermes hermes chat
```

Or run a one-off container against the same data directory:

```bash
docker run -it --rm \
  -v "$HOME/.hermes:/opt/data" \
  nousresearch/hermes-agent:v0.23.1
```

Never run two gateways against the same data directory at the same time — session and memory files are not designed for concurrent writers.

### Backups

Everything that matters is in the data directory. Stop the container first so the SQLite WAL is consistent:

```bash
DATA_DIR="$HOME/.hermes"   # match HERMES_DATA_DIR
docker compose stop hermes
tar -czf "hermes-backup-$(date +%F).tar.gz" \
  -C "$(dirname "$DATA_DIR")" "$(basename "$DATA_DIR")"
cp .env "hermes-compose-env-$(date +%F).backup"
docker compose start hermes
```

Restore by stopping the stack, extracting the archive over the data directory, and starting again. Keep backups off the Pi (NAS, restic, S3, …).

### Updating Hermes

The image ships immutable application code, so `hermes update` inside the container is not the upgrade path — replace the image instead:

```bash
# 1. Pick a new tag: https://hub.docker.com/r/nousresearch/hermes-agent/tags
# 2. Update HERMES_IMAGE_TAG in .env
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 hermes
```

Config-schema migrations run automatically on boot. To roll back, set the previous tag in `.env` and repeat the pull/up commands. Back up before upgrading.

---

## Security notes

- Both `.env` files should be mode `600`. They contain API keys and bot tokens.
- The dashboard is bound to `127.0.0.1`; use an SSH tunnel or a TLS reverse proxy. Basic auth over plain HTTP is not safe on an untrusted network.
- The API server is disabled by default. If enabled, use a long random `API_SERVER_KEY` and keep it on loopback.
- Set `TELEGRAM_ALLOWED_USERS`; an open bot is remote code execution with extra steps.
- Hermes executes shell commands on your behalf. Review the approval and tool settings in `config.yaml`, and consider `terminal.backend: docker` if you want the agent's commands sandboxed. See [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security).
- Do not mount the Docker socket into the container unless you fully understand the consequences.
- This stack does not publish any port on `0.0.0.0` by default.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `docker compose config` fails on `HERMES_DASHBOARD_PASSWORD` | Set the password (and secret) in `.env`. They are intentionally required. |
| Dashboard returns `401` after changing credentials | Recreate the container: `docker compose up -d --force-recreate`. |
| `permission denied` in `$HERMES_DATA_DIR` | Set `HERMES_UID`/`HERMES_GID` to `id -u` / `id -g`, then `sudo chown -R "$(id -u):$(id -g)" "$DATA_DIR"`. |
| Container is `unhealthy` | Either the dashboard or API server must listen. If you disabled the dashboard, enable the API or adjust the healthcheck. Check `docker compose logs hermes`. |
| `bind: address already in use` | Something else uses 9119 (or 8642). Change the host side of the mapping, e.g. `127.0.0.1:19119:9119`. |
| Agent answers with a provider/model error | The model ID or provider in `config.yaml` is wrong/unavailable. Run `docker exec -it hermes hermes model` and pick a valid one. |
| 32-bit Raspberry Pi | The image is published for `amd64` and `arm64`. Use a 64-bit OS. |
| Need to inspect the effective Compose config | `docker compose config` renders the resolved file, including defaults and paths. |
