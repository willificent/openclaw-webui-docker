# NOTE: This repo is currently under development! Please try it out and reach out if you have any tips to make it functional.

# OpenClaw + Open WebUI Docker Stack

A near-zero-config Docker Compose setup for spinning up an OpenClaw agent with an Open WebUI frontend. Pre-configured for Ollama Cloud API — just add your credentials and go.

## What This Is

This stack gives you a **fully functional AI agent** in under a minute:

- **OpenClaw Gateway** — The agent runtime with tool access, memory, and model routing
- **Open WebUI** — A ChatGPT-like frontend for chatting with your agent
- **Browser Container** — For agents that need web browsing capabilities

**Key features:**
- Single `docker compose up -d` to launch
- One external port (3000)
- Pre-paired authentication between gateway and UI
- Works with any OpenAI-compatible API (Ollama Cloud, local Ollama, etc.)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Your Browser → http://localhost:3000                  │
│                     │                                    │
│              Open WebUI (port 3000)                     │
│                     │                                    │
│         ┌──────────┴──────────┐                          │
│         │   Shared Secret     │                          │
│         │   (open-handshake)  │                          │
│         └──────────┬──────────┘                          │
│                    │                                     │
│           OpenClaw Gateway                               │
│         (internal:8080, external:18789)                   │
│                    │                                     │
│         ┌──────────┴──────────┐                          │
│         │  Model Providers    │                          │
│         │  (Ollama Cloud API) │                          │
│         └─────────────────────┘                          │
└─────────────────────────────────────────────────────────┘
```

## Why This Architecture?

During development, we evaluated several approaches:

1. **Fat container (all-in-one)** — Easier to deploy, but harder to update individual components
2. **Separate containers with manual pairing** — Flexible, but required manual token generation
3. **Two-container with pre-shared secret** — Best balance: fast spin-up, clear separation, no manual steps

We landed on **option 3**: Two containers (gateway + webui) on a shared Docker network, with a pre-configured shared secret for instant pairing. The browser container is optional but included for agents that need web access.

## Prerequisites

- Docker and Docker Compose installed
- An API key from [Ollama Cloud](https://ollama.com) (or another OpenAI-compatible provider)
- (Optional) A different shared secret token for production use

## Quick Start

### 1. Clone and Configure

```bash
git clone https://github.com/willificent/openclaw-webui-docker.git
cd openclaw-webui-docker
```

### 2. Edit `openclaw.json`

You **must** customize the configuration before starting:

```bash
cp openclaw.json.example openclaw.json
# Edit openclaw.json with your credentials
```

**Required changes:**

| Field | Description | Example |
|-------|-------------|---------|
| `models.providers.ollama-cloud.apiKey` | Your Ollama Cloud API key | `abc123...` |
| `gateway.auth.token` | Shared secret for gateway-ui auth | `my-secret-token-2026` |

**Important:** The `gateway.auth.token` must match the `OPENAI_API_KEY` in the `open-webui` service environment. The docker-compose.yml uses `open-handshake-2026` as a placeholder — change both if you want a different secret.

### 3. Launch

```bash
docker compose up -d
```

### 4. Access and Select Your Agent

1. Open http://localhost:3000 in your browser.
2. **Crucial Model Selection:** By default, Open WebUI might not show anything in the chat.
3. Click the **Model Selector** dropdown at the top center of the chat interface.
4. Select **`agent:main`** (or `agent:{your-agent-id}`). 
5. If the dropdown is empty, go to **Settings > Connections** and verify that the OpenAI API URL is `http://openclaw:8080/v1` and the API Key matches your `OPENCLAW_GATEWAY_TOKEN`.

---

## 🛠️ Step-by-Step UI Verification

If you are a first-time user, follow these exact steps to ensure the "Handshake" is active:

1. **Check the Gateway Logs:** Run `docker compose logs -f openclaw`. Look for `[info] gateway started on port 18789`.
2. **Verify Connections in Open WebUI:**
   - Click your profile icon (bottom left) → **Settings**.
   - Select **Connections**.
   - Under **OpenAI API**, ensure the status is green.
   - Click the "Refresh" (recycle) icon next to the URL—it should pull the models and show **`agent:main`** in the list.
3. **Set the Default:** You can set `agent:main` as your default model in **Settings > General** so it's always ready when you open the lab.

## Configuration Details

### openclaw.json

This is the main OpenClaw configuration. Key sections:

**Models:**
```json
"providers": {
  "ollama-cloud": {
    "baseUrl": "https://ollama.com/v1",
    "apiKey": "{YOUR-API-KEY}",
    "api": "openai-completions",
    "models": [...]
  }
}
```

**Agents:**
```json
"list": [
  {
    "id": "main",
    "name": "Generic Agent",
    "workspace": "/data/workspace"
  }
]
```

The agent ID `main` appears as `agent:main` in Open WebUI's model selector.

### docker-compose.yml

**Environment variables:**

| Variable | Purpose | Default |
|----------|---------|---------|
| `OPENCLAW_GATEWAY_TOKEN` | Auth token for gateway | `open-handshake-2026` |
| `OPENAI_API_KEY` | Must match gateway token | `open-handshake-2026` |
| `WEBUI_AUTH` | Disable login screen | `false` |
| `WEBUI_NAME` | Display name in UI | `Generic Openclaw Lab` |

**Ports:**

| External | Internal | Service |
|----------|----------|---------|
| 3000 | 8080 | Open WebUI |
| 18789 | 18789 | OpenClaw Gateway (external API) |

## Customization

### Using Local Ollama Instead of Cloud

Edit `openclaw.json`:

```json
"providers": {
  "ollama-local": {
    "baseUrl": "http://host.docker.internal:11434/v1",
    "api": "ollama",
    "models": [
      { "id": "llama3.2", "name": "Llama 3.2" }
    ]
  }
}
```

Then update `agents.defaults.model.primary` to `ollama-local/llama3.2`.

### Adding More Agents

Add entries to `agents.list`:

```json
{
  "id": "research",
  "name": "Research Agent",
  "workspace": "/data/workspace",
  "model": { "primary": "ollama-cloud/glm-5:cloud" }
}
```

Each agent appears as `agent:{id}` in Open WebUI.

### Persistent Storage

Data is stored in `./data/` by default:
- `./data/openclaw/` — Gateway state, memory, plugins
- `./data/open-webui/` — UI preferences, chat history
- `./data/browser-config/` — Browser container state

## Security Notes

⚠️ **This setup is designed for local/ephemeral use.** For production:

1. Change the shared secret to a strong, unique token
2. Enable `WEBUI_AUTH=true` and configure user accounts
3. Remove `allowInsecureAuth: true` from gateway config
4. Bind gateway to `localhost` instead of `lan`
5. Put everything behind a reverse proxy with TLS

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Container orchestration |
| `openclaw.json.example` | Template configuration (copy to `openclaw.json`) |
| `openclaw.json` | Your customized config (gitignored) |
| `data/` | Persistent storage (created on first run) |

## Troubleshooting

**Agent not appearing in Open WebUI:**
- Check gateway logs: `docker compose logs openclaw`
- Verify the token matches between `openclaw.json` and `docker-compose.yml`
- Ensure gateway health check passes: `curl http://localhost:18789/healthz`

**API errors:**
- Verify your Ollama Cloud API key is valid
- Check the `api` field is `"openai-completions"` for Ollama Cloud
- Ensure `baseUrl` ends with `/v1`

**Container keeps restarting:**
- Check for port conflicts (3000, 18789)
- Verify Docker has enough memory (2GB+ recommended)

## Credits

Created by [Clawde](https://github.com/willificent) as part of the OpenClaw ecosystem.

OpenClaw: https://github.com/openclaw/openclaw
Open WebUI: https://github.com/open-webui/open-webui

## License

MIT — do whatever you want with it. Just don't ship your API keys.
