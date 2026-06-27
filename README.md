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

## 📦 Directory Structure & Setup

Before spinning up the container, ensure the following configuration architecture exists within the user's home directory:

```text
~/.hermes/
├── .env                # Application environment variables
├── config.yaml         # Main agent configuration file
├── SOUL.md             # Bot identity and personality definition
└── memories/
    ├── USER.md         # Static user preferences and demographics
    └── MEMORY.md       # Environment topology and infrastructure description (IoT, Network)

```

---

## 🔑 Environment Variables (`~/.hermes/.env`)

Create and populate the `.env` file inside the `~/.hermes/` directory before deploying the stack:

```env
# API Keys
OPENAI_API_KEY=sk-proj-YourOpenAIApiKey...
GOOGLE_API_KEY=AIzaSy...

# Telegram Bot Integration
TELEGRAM_BOT_TOKEN=123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ
TELEGRAM_ALLOWED_USERS=YourTelegramID  # Comma-separated if specifying multiple IDs

# Dashboard Security
HERMES_DASHBOARD_USER=admin
HERMES_DASHBOARD_PASSWORD=ChooseASecureDashboardPassword
HERMES_DASHBOARD_SECRET=GenerateARandomStringForSessionSigning

```

---

## ⚙️ Model Configuration Example (`~/.hermes/config.yaml`)

To ensure accurate provider mapping and prevent *Unknown provider* runtime exceptions, the `model` section must target the explicit provider identifier:

```yaml
model:  
  default: gpt-5-nano  # or gpt-4o-mini / gemini-1.5-flash
  provider: openai-api # Use 'openai-api' or 'gemini'

```

---

## 🚀 Deployment

1. Clone this repository to your target workspace:
```bash
git clone <your-repository-url>
cd <repository-folder-name>

```


2. Pull the latest image layers and deploy the stack in the background (detached mode):
```bash
docker compose pull
docker compose up -d

```


3. Verify operational status and view runtime gateway logs:
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

*Authentication requires the credentials specified by the `HERMES_DASHBOARD_USER` and `HERMES_DASHBOARD_PASSWORD` environment variables.*

---

## 🤖 Dynamic Model Swaps via Telegram

Once the bot is bound to Telegram, global model switching can be handled on-the-fly directly within the chat window using the following commands:

```text
/model gpt-5-nano --provider openai-api --global

```

```text
/model gemini-1.5-pro --provider gemini --global

```
