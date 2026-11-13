# Project Plan — Hermes Agent Local Stack

Last updated: 2026-11-13
Maintainer: Kamil Szymczycha ([@kszymczycha](https://github.com/kszymczycha))

This file is the shared "where we left off" document for future sessions. Read it
first, then check `git log` and `git status` before changing anything.

---

## Current state

- **Purpose:** self-hosted Hermes Agent stack (gateway + dashboard, optional
  OpenAI-compatible API) for a Raspberry Pi 4 / Linux host.
- **Image:** `nousresearch/hermes-agent`, pinned to `v0.23.1` via
  `HERMES_IMAGE_TAG` in `.env`.
- **Deployment model:** single container, all state in `$HERMES_DATA_DIR`
  (default `~/.hermes`) mounted at `/opt/data`.
- **Dashboard:** `127.0.0.1:9119` only, basic auth required.
- **API server:** disabled by default; port `8642` mapping is commented out.
- **Secrets split:** Compose settings live in the repo-level `.env`; Hermes
  secrets (LLM keys, bot tokens) live in `$HERMES_DATA_DIR/.env`.
- **Default documented provider:** DeepSeek via OpenRouter
  (`provider: openrouter`, `deepseek/deepseek-v4.1-flash-1017`). Direct OpenAI
  remains an optional alternative.
- **License:** MIT (`LICENSE`).
- **Provider examples:** DeepSeek via OpenRouter is the primary example; OpenAI
  direct is optional. Do not reintroduce previously removed provider examples.

## Completed

- [x] Pin the image and make the tag configurable.
- [x] Fix the `HERMES_DATA_DIR` / `~/.hermes` documentation mismatch.
- [x] Move LLM keys and bot tokens out of Compose into the data-dir `.env`.
- [x] Bind the dashboard to loopback; make the API server opt-in.
- [x] Add healthcheck, log rotation, resource limits, graceful shutdown.
- [x] Add `HERMES_UID` / `HERMES_GID` ownership handling.
- [x] Rewrite README: setup, security, backup/restore, upgrades, troubleshooting.
- [x] Add templates: `config.yaml.example`, `SOUL.md.example`,
      `memories/USER.md.example`, `memories/MEMORY.md.example`.
- [x] Use DeepSeek via OpenRouter (`deepseek/deepseek-v4.1-flash-1017`).
- [x] Add the MIT license.
- [x] Add `AGENTS.md` (OpenCode project instructions), `opencode.json` (loads
      `PLAN.md`), and the stack operations skill at
      `.agents/skills/hermes-agent-stack/SKILL.md`.
- [x] Create this plan.

## Backlog

Pick the first unchecked item in priority order unless the user says otherwise.
Keep the table statuses accurate: `todo`, `in progress`, `blocked`, `done`.

| ID | Priority | Task | Status |
| --- | --- | --- | --- |
| P2-1 | High | CI validation workflow (Compose + YAML) | todo |
| P2-2 | High | `Makefile` for setup/up/logs/doctor/backup | todo |
| P3-1 | Medium | API server via Compose override file | todo |
| P3-2 | Medium | "Migrating from the previous version" README section | todo |
| P3-3 | Low | Optional digest pinning for the image | todo |
| P3-4 | Medium | Wire `SKILL.md` into project-local skill discovery | done |
| P4-1 | Later | Reverse-proxy example (Caddy/nginx + TLS) | todo |

---

## P2-1 — CI validation workflow

**Goal:** every push/PR validates the Compose file and the YAML examples before
a bad config reaches a Raspberry Pi.

**Steps:**

1. Create `.github/workflows/validate.yml` triggered on `push` and
   `pull_request`.
2. Use `ubuntu-latest` (Docker Compose v2 is preinstalled).
3. Create dummy required variables in a temporary env file, e.g.:

   ```bash
   cat > /tmp/ci.env <<'EOF'
   HERMES_DASHBOARD_PASSWORD=ci-password
   HERMES_DASHBOARD_SECRET=0123456789abcdef0123456789abcdef
   EOF
   docker compose --env-file /tmp/ci.env config >/dev/null
   ```

4. Validate YAML with `yamllint` (pip install) for `docker-compose.yaml` and
   `config.yaml.example`.
5. Optional: fail on broken README links (e.g. `lychee`) once the workflow is
   stable.

**Acceptance criteria:**

- Workflow fails when a required variable is missing or YAML is invalid.
- Green run on the current `main`.
- No secrets or real credentials in the workflow.

**Files:** `.github/workflows/validate.yml` (new).

---

## P2-2 — Makefile

**Goal:** replace the long manual command sequences in the README with
idempotent targets, so setup and operations are one-liners.

**Suggested interface:**

```text
make setup     # create .env, data dir, copy templates (never overwrite data)
make up        # docker compose pull && docker compose up -d
make down      # docker compose down
make restart   # docker compose restart hermes
make ps        # docker compose ps
make logs      # docker compose logs -f --tail=100
make config    # docker compose config (resolved configuration)
make doctor    # docker exec -it hermes hermes doctor
make backup    # stop, tar data dir + .env, start
```

**Constraints:**

- `DATA_DIR ?= $(HOME)/.hermes`, overridable.
- `setup` must not clobber an existing `config.yaml`, `SOUL.md`, memories,
  `.env` or data-dir `.env`.
- Do not print secret values.
- `backup` must stop the container first (SQLite WAL consistency).

**Acceptance criteria:**

- On a fresh clone, `make setup && make up` works.
- `make config` fails clearly if the required dashboard variables are missing.
- README quick start is updated to use the targets (keep raw commands in a
  "manual setup" note).

**Files:** `Makefile` (new), `README.md`.

---

## P3-1 — API server via Compose override file

**Goal:** enable the API server without editing the main Compose file.

**Steps:**

1. Add `docker-compose.api.yaml`:

   ```yaml
   services:
     hermes:
       ports:
         - "127.0.0.1:8642:8642"
   ```

2. Document usage:

   ```bash
   docker compose -f docker-compose.yaml -f docker-compose.api.yaml up -d
   ```

3. Require `API_SERVER_ENABLED=true` and a key of at least 8 characters in
   `.env`; keep the port on loopback.
4. Remove the commented `8642` block from `docker-compose.yaml` and point the
   README at the override file instead.

**Acceptance criteria:**

- `docker compose config` without the override still has no `8642` mapping.
- `docker compose -f docker-compose.yaml -f docker-compose.api.yaml config`
  contains the `8642` mapping.
- README explains enabling and disabling it.

**Files:** `docker-compose.api.yaml` (new), `docker-compose.yaml`, `README.md`.

---

## P3-2 — Migration notes for the previous layout

**Goal:** help users of the old stack (dashboard on `0.0.0.0:9119`, keys in the
repo `.env`, `~/.hermes` hardcoded) upgrade without surprises.

**Add a README section covering:**

1. The data directory path is unchanged; existing `~/.hermes` data is reused.
2. Move `OPENROUTER_API_KEY` (and `OPENAI_API_KEY` if used),
   `TELEGRAM_BOT_TOKEN` and similar values from the repo `.env` to
   `$HERMES_DATA_DIR/.env`; remove them from the Compose `.env`.
3. The dashboard is no longer reachable from other machines; use an SSH tunnel:
   `ssh -L 9119:127.0.0.1:9119 user@host`.
4. Recreate the container after changing ports/auth:
   `docker compose up -d --force-recreate`.
5. Verify with `docker compose ps`, `docker compose logs`, and
   `docker exec -it hermes hermes doctor`.

**Acceptance criteria:** a user with the old layout can migrate using only the
README section; no data files are deleted.

**Files:** `README.md`.

---

## P3-3 — Optional image digest pinning

**Goal:** offer fully immutable deployments for users who want them, while
keeping the tag-based flow as the default.

**Steps:**

1. Document the trade-off: tag = readable, digest = immutable but opaque.
2. Show how to resolve a digest, e.g.:
   `docker buildx imagetools inspect nousresearch/hermes-agent:v0.23.1`
   (record the `linux/arm64` and `linux/amd64` digests separately).
3. Explain how to set `image: nousresearch/hermes-agent@sha256:...` and how to
   upgrade it deliberately.
4. Keep the `HERMES_IMAGE_TAG` override path documented for normal users.

**Acceptance criteria:** README contains copy-paste commands and the security /
reproducibility rationale.

**Files:** `README.md` (possibly `docker-compose.yaml` comments only).

---

## P3-4 — Wire SKILL.md into project-local discovery (done)

**Status:** implemented on 2026-11-13.

**Goal:** make OpenCode and Hermes load the stack operations skill
automatically for sessions started in this repository.

**Implementation:**

- Canonical instructions: `AGENTS.md` (OpenCode V2 reads this file).
- `opencode.json` uses the `instructions` array to load `PLAN.md` automatically
  in every OpenCode session.
- Canonical skill:
  `.agents/skills/hermes-agent-stack/SKILL.md`. Both OpenCode V2 (project
  compatibility source) and Hermes (project-local skill directory) discover
  `.agents/skills/`.
- Root-level symlinks were removed; the canonical files are the only copies.
- Hermes requires a one-time `hermes skills trust` inside the repository.

**Acceptance criteria (met):**

- An OpenCode session sees the `hermes-agent-stack` skill automatically.
- A Hermes session sees it after `hermes skills trust`.
- Only one copy of each document is tracked (no duplicate symlinks).
- `README.md` and `AGENTS.md` describe the activation step.

**Files:** `.agents/skills/hermes-agent-stack/SKILL.md`, `AGENTS.md`,
`opencode.json`, `README.md`.

---

## P4-1 — Reverse proxy example (optional)

**Goal:** document a safe public deployment path.

**Scope:**

- A minimal Caddy or nginx config that terminates TLS and proxies to
  `127.0.0.1:9119`.
- Required `dashboard.public_url` and a bounded `dashboard.trusted_proxies`
  entry in `config.yaml`.
- Explicit warning not to change the port mapping to `0.0.0.0`.

**Acceptance criteria:** example is self-contained and does not weaken the
loopback default.

**Files:** `README.md`, possibly `examples/Caddyfile`.

---

## Validation before every commit

```bash
# 1. Compose resolves with dummy required variables
cat > /tmp/hermes-ci.env <<'EOF'
HERMES_DASHBOARD_PASSWORD=ci-password
HERMES_DASHBOARD_SECRET=0123456789abcdef0123456789abcdef
EOF
docker compose --env-file /tmp/hermes-ci.env config >/dev/null

# 2. YAML examples parse
python3 - <<'PY'
import yaml
for path in ("docker-compose.yaml", "config.yaml.example"):
    with open(path) as fh:
        yaml.safe_load(fh)
    print(path, "OK")
PY

# 3. No accidental secrets, whitespace errors, or stale provider references
git diff --check
grep -RniE 'gemini|google' README.md .env.example config.yaml.example \
  SOUL.md.example memories/ || true
grep -q 'provider: openrouter' config.yaml.example
grep -q 'default: deepseek/deepseek-v4.1-flash-1017' config.yaml.example

# 4. Agent integration: canonical files exist
test -f AGENTS.md
test -f opencode.json
python3 -m json.tool opencode.json >/dev/null
test -f .agents/skills/hermes-agent-stack/SKILL.md
```

Run these from the repository root. Do not commit `.env`, `config.yaml`,
`SOUL.md`, the real `memories/*.md`, or `$HERMES_DATA_DIR` contents.

## Conventions and guardrails

- **Never** publish `9119` or `8642` on `0.0.0.0` by default.
- **Never** put LLM keys or bot tokens in the repo-level `.env`.
- **DeepSeek = OpenRouter:** use `provider: openrouter` with
  `deepseek/deepseek-v4.1-flash-1017`. Do not switch the example to the native
  `deepseek` provider.
- **Provider examples:** use DeepSeek via OpenRouter as the primary example;
  OpenAI direct is optional. Do not reintroduce previously removed provider
  examples.
- **Pinned image:** update `HERMES_IMAGE_TAG` deliberately; never use `latest`.
- **One gateway per data directory:** never run two containers against the same
  `$HERMES_DATA_DIR`.
- Commit messages: short English imperative subject, optional bullet body.
- Update this plan in the same commit when a backlog item is completed.

## How to resume in a new session

1. `git status --short --branch` and `git log --oneline -5`.
2. Read `AGENTS.md` (working rules) and this file; pick the first `todo` item by
   priority.
3. Mark it `in progress` mentally (or in the table if the session is long).
4. Implement, run the validation block, update the table to `done`, and add a
   line to the changelog below.
5. Commit and push only after the user asks (or if the current task explicitly
   includes it).

## Changelog

- **2026-11-13:** Plan created. Recorded the hardening work, the DeepSeek →
  OpenRouter correction, the MIT license, and the remaining P2/P3/P4 backlog.
- **2026-11-13:** Added the initial agent guide and the P3-4 task for wiring the
  skill into project-local discovery.
- **2026-11-13:** Standardized the repo on OpenRouter-first:
  `provider: openrouter`, `deepseek/deepseek-v4.1-flash-1017`. OpenAI direct is
  now only an optional alternative; removed the stale third-party key from the
  migration notes.
- **2026-11-13:** Added OpenCode V2 integration: canonical `AGENTS.md`,
  skill moved to `.agents/skills/hermes-agent-stack/SKILL.md` (shared by
  OpenCode and Hermes), `opencode.json` loads `PLAN.md`, plus temporary
  compatibility links for legacy filenames.
- **2026-11-13:** Cleaned up: removed the redundant compatibility symlinks and
  kept only the canonical `AGENTS.md` and the skill in `.agents/skills/`.
