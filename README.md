# Hermes Agent - Local Deployment Stack

This repository contains a complete containerized configuration to deploy and manage an autonomous **Hermes Agent** (by Nous Research) on your own infrastructure (e.g., Raspberry Pi 4 / Linux Server). 

The stack integrates LLM orchestration (OpenAI API, Google Gemini), a network Gateway API, a Web Dashboard UI, and a direct communication interface via a Telegram bot.

## 🚀 Architecture & Features
- **Multi-Provider LLM Support:** Native compatibility with OpenAI models (including GPT-5/GPT-4o lines) and Google Gemini.
- **Persistent Context:** Robust identity framework via `SOUL.md` along with automated memory compression utilizing `USER.md` and `MEMORY.md`.
- **Telegram Bot Integration:** Full, remote agent control directly from your favorite messenger.
- **Resource Constrained:** Optimized for edge deployments, hardware-limited at the container level to 4GB RAM and 2 vCPUs.

---

## 🛠️ Prerequisites
- Docker and Docker Compose V2 installed on the host system.
- Valid API keys for chosen providers (OpenAI, Google AI Studio).
- A Telegram Bot Token (generated via `@BotFather`).

---

## 📦 Directory Structure

This repository is designed to run self-contained. All application configurations, identity profiles, and memories are mounted directly from the repository's root directory:

```text
.
├── docker-compose.yaml # Docker multi-container specification
├── .env.example        # Template for your local environment variables
├── .env                # Your local secrets (Ignored by Git)
├── config.yaml         # Main agent configuration file
├── SOUL.md             # Bot identity and personality definition
└── memories/
    ├── USER.md         # Static user preferences and demographics
    └── MEMORY.md       # Environment topology and infrastructure description (IoT, Network)

```

---

## 🧠 Context & Identity Initialization (`~/.hermes/`)

The core personality and memory architecture of the agent depend on three markdown files. Inside the container, the application expects these files to reside in the user's home configuration directory (`~/.hermes/` and `~/.hermes/memories/`).

Through Docker volume mapping, the files located in your local repository directory are mounted directly to those target paths inside the running container.

⚠️ **Crucial Timing:** You must populate and verify the content of `SOUL.md`, `memories/USER.md`, and `memories/MEMORY.md` **BEFORE running the deployment** (`docker compose up -d`). The agent reads its identity framework and infrastructure topography during its bootstrap phase; changes made after deployment will require a container restart to take effect.

---

## ⚙️ Model Configuration Example (`config.yaml`)

To ensure accurate provider mapping and prevent *Unknown provider* runtime exceptions, the `model` section must target the explicit provider identifier:

```yaml
model:  
  default: gpt-5-nano  # or gpt-4o-mini / gemini-1.5-flash
  provider: openai-api # Use 'openai-api' or 'gemini'

```

---

## 🚀 Deployment

1. **Clone this repository** to your target workspace on your Raspberry Pi:

```bash
git clone git@github.com:kszymczycha/hermes-agent-stack.git
cd hermes-agent-stack

```

2. **Prepare the environment file** by copying the provided example template:

```bash
cp .env.example .env

```

3. **Configure your secrets** inside the newly created `.env` file using `nano` or your preferred text editor:

```bash
nano .env

```

> 💡 *Note: Secure your session signing by generating a random hex string for `HERMES_DASHBOARD_SECRET` directly on your Pi via: `openssl rand -hex 32*`

4. **Initialize identity and context documents:**
Review and modify `SOUL.md`, `memories/USER.md`, and `memories/MEMORY.md` to feed the agent its personality, your preferences, and your local network/IoT environment details **before** starting the engine.
5. **Deploy the stack** in the background (detached mode):

```bash
docker compose pull
docker compose up -d

```

6. **Verify operational status** and view runtime gateway logs:

```bash
docker compose ps
docker compose logs -f

```

---

## 🩺 Diagnostics & CLI Operations

All internal agent tasks can be issued directly within the context of the running container:

* **Verify configuration and connectivity (Doctor):**

```bash
docker exec -it hermes hermes doctor

```

* **Launch interactive CLI chat session:**

```bash
docker exec -it hermes hermes chat

```

* **Force application core update:**

```bash
docker exec -it hermes hermes update

```

---

## 📊 Dashboard Access

When the `HERMES_DASHBOARD=1` flag is set, the administrative panel and execution logs can be accessed via your browser at:

```text
http://<SERVER-IP>:9119

```

*Authentication requires the credentials specified by the `HERMES_DASHBOARD_USER` and `HERMES_DASHBOARD_PASSWORD` variables, while sessions are securely managed by `HERMES_DASHBOARD_SECRET`.*

---

## 🤖 Dynamic Model Swaps via Telegram

Once the bot is bound to Telegram, global model switching can be handled on-the-fly directly within the chat window using the following commands:

```text
/model gpt-5-nano --provider openai-api --global

```

```text
/model gemini-1.5-pro --provider gemini --global

```
