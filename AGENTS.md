# AGENTS.md

Operating instructions for AI coding agents (OpenCode, Claude Code, Codex, and
similar) working on this repository. For Hermes' runtime identity and project
skills, see `SOUL.md.example` and
`.agents/skills/hermes-agent-stack/SKILL.md`.

## Tool loading

- **OpenCode V2** loads this `AGENTS.md` automatically and also loads
  `PLAN.md` through the `instructions` array in `opencode.json`.
- **OpenCode and Hermes** both discover the project skill at
  `.agents/skills/hermes-agent-stack/SKILL.md`.
- Hermes needs a one-time `hermes skills trust` inside the repository before it
  loads project skills.
- There are no duplicate copies: edit `AGENTS.md` and
  `.agents/skills/hermes-agent-stack/SKILL.md` directly.

## What this repository is

A self-hosted Docker Compose stack for **Hermes Agent** (Nous Research). One
container runs the messaging gateway and dashboard; all state lives in
`$HERMES_DATA_DIR` (default `~/.hermes`) mounted at `/opt/data`.

Key constraints:

- The container image is pinned via `HERMES_IMAGE_TAG` (currently `v0.23.1`).
- Hermes secrets (for example `OPENROUTER_API_KEY` and bot tokens) live in
  `$HERMES_DATA_DIR/.env`, **not** in the repo `.env`.
- The dashboard is published on `127.0.0.1:9119` only.
- The API server is disabled by default; its `8642` mapping is commented out.
- DeepSeek examples use **OpenRouter** (`provider: openrouter`) with
  `deepseek/deepseek-v4.1-flash-1017`.
- The project no longer documents the previously used third-party provider. Do
  not reintroduce old provider examples.

## Non-negotiable rules

1. **Never commit live state or secrets:** `.env`, `config.yaml`, `SOUL.md`,
   `memories/USER.md`, `memories/MEMORY.md`, or anything from
   `$HERMES_DATA_DIR`.
2. **Never publish ports on `0.0.0.0` by default.** Keep `9119` and `8642`
   bound to `127.0.0.1`. Public access goes through an SSH tunnel or a
   TLS-terminating reverse proxy.
3. **Keep the secret split.** Compose/deployment settings belong in the repo
   `.env`; LLM keys and bot tokens belong in `$HERMES_DATA_DIR/.env`.
4. **Never use `image: ...:latest`.** Bump the pinned tag deliberately and
   document the change.
5. **DeepSeek is reached through OpenRouter.** Use `provider: openrouter` and
   `deepseek/deepseek-v4.1-flash-1017`; do not switch examples to the native
   `deepseek` provider.
6. **One gateway per data directory.** Never point two containers at the same
   `$HERMES_DATA_DIR`.
7. **`hermes update` is not the Docker upgrade path.** Replace the image via
   `HERMES_IMAGE_TAG`, `docker compose pull`, and `docker compose up -d`.
8. **Do not run `docker compose up -d` on a user's host without being asked.**
   Validation with `docker compose config` is safe; starting containers is not.

## Repository map

| Path | Purpose |
| --- | --- |
| `docker-compose.yaml` | Container, ports, volume, env mapping, healthcheck, limits |
| `.env.example` | Deployment variables for Compose (copy to `.env`) |
| `config.yaml.example` | Agent configuration template |
| `SOUL.md.example` | Agent identity template |
| `memories/USER.md.example` | User-profile memory template |
| `memories/MEMORY.md.example` | Environment/long-term memory template |
| `README.md` | User-facing setup, security, backup and troubleshooting guide |
| `PLAN.md` | Backlog and session handoff notes — read it first |
| `AGENTS.md` | OpenCode project instructions (this file) |
| `opencode.json` | OpenCode V2 config; loads `PLAN.md` as instructions |
| `.agents/skills/hermes-agent-stack/SKILL.md` | Project skill discovered by OpenCode and Hermes |
| `LICENSE` | MIT |

## Session start

Before changing anything:

```bash
git status --short --branch
git log --oneline -5
sed -n '1,80p' PLAN.md
```

Then pick the first `todo` item from the backlog in `PLAN.md`, unless the user
asked for something else.

## Required validation

Run from the repository root before every commit:

```bash
# Compose resolves with the required variables present
cat > /tmp/hermes-ci.env <<'EOF'
HERMES_DASHBOARD_PASSWORD=ci-password
HERMES_DASHBOARD_SECRET=0123456789abcdef0123456789abcdef
EOF
docker compose --env-file /tmp/hermes-ci.env config >/dev/null

# YAML examples parse
python3 - <<'PY'
import yaml
for path in ("docker-compose.yaml", "config.yaml.example"):
    with open(path) as fh:
        yaml.safe_load(fh)
    print(path, "OK")
PY

# No whitespace errors, stale provider examples, or wrong default provider
git diff --check
grep -RniE 'gemini|google' README.md .env.example config.yaml.example \
  SOUL.md.example memories/ || true
grep -q 'provider: openrouter' config.yaml.example
grep -q 'default: deepseek/deepseek-v4.1-flash-1017' config.yaml.example

# Agent integration: canonical files exist
test -f AGENTS.md
test -f opencode.json
python3 -m json.tool opencode.json >/dev/null
test -f .agents/skills/hermes-agent-stack/SKILL.md
```

## Common tasks

### Bump the Hermes image

1. Pick a release tag from
   <https://hub.docker.com/r/nousresearch/hermes-agent/tags>.
2. Update `HERMES_IMAGE_TAG` in `.env.example` (and mention it in `README.md`
   if the version is listed there).
3. Remind the user to back up `$HERMES_DATA_DIR` before upgrading.
4. Validation target: `docker compose --env-file /tmp/hermes-ci.env config`
   shows the new image.

### Change a provider or model example

- **Document DeepSeek via OpenRouter as the primary example.** OpenAI direct
  may remain only as an optional alternative.
- Use exact provider IDs (`openrouter`, `openai-api`, …) — wrong IDs cause
  runtime "unknown provider" errors.
- Model catalogues change; point users to `hermes model` or the dashboard
  instead of hardcoding long lists.

### Change ports or exposure

- Keep loopback bindings. If a new port is needed, use
  `127.0.0.1:<host>:<container>`.
- Update the README security section and `PLAN.md` if the change is part of a
  backlog item.

### Update the plan

- Mark items `done` in the backlog table.
- Add a dated line to the changelog in `PLAN.md`.
- If a decision changes (provider, ports, image policy), update the
  "Conventions and guardrails" section.

## Style

- Repository docs are written in English; talk to the user in the language
  they use.
- Keep commands copy-pasteable and test them when possible.
- Prefer small, focused commits with English imperative subjects.
- Do not add emojis to headings unless the user asks.
- When a change depends on external facts (image tags, provider IDs, model
  names), verify them against the official Hermes docs instead of guessing.

## Handoff

At the end of a session, leave `PLAN.md` accurate enough that the next session
can continue without reading the whole conversation:

- what was completed,
- what is `in progress`,
- what is blocked and why,
- the next concrete command or file to touch.
