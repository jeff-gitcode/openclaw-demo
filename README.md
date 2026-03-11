# OPENCLAW SETUP GUIDE
> Last updated: 2026-03-11

## Overview

OpenClaw is a self-hosted AI agent platform. It runs a local gateway that connects AI models (local via Ollama or cloud via OpenRouter) to channels like Telegram.

---

## System Architecture

   ╔══════════════════════════════════════════════════════════════════╗
   ║                      OPENCLAW SYSTEM                             ║
   ╚══════════════════════════════════════════════════════════════════╝

     ┌─────────────────────┐         ┌─────────────────────────────┐
     │   TELEGRAM (Cloud)  │         │     DASHBOARD (Browser)     │
     │  Polling mode       │         │  http://127.0.0.1:18789/    │
     │  Bot: @<TELEGRAM_BOT_NAME>    │  SSH tunnel from client     │
     └──────────┬──────────┘         └──────────────┬──────────────┘
                │ messages                           │ HTTP + token
                ▼                                   ▼
     ╔══════════════════════════════════════════════════════════════╗
     ║            OPENCLAW GATEWAY                                  ║
     ║            ws://127.0.0.1:18789                              ║
     ║                                                              ║
     ║   Auth token: <GATEWAY_AUTH_TOKEN>                           ║
     ║   Config: ~/.openclaw/openclaw.json                          ║
     ║   Log: /tmp/openclaw/openclaw-YYYY-MM-DD.log                 ║
     ╚═══════════╤══════════════════════════════════════════════════╝
                 │
                 │  model requests
                 │
         ┌───────┴────────────────────────────────┐
         │                                        │
         ▼  PRIMARY                               ▼  FALLBACK
     ┌──────────────────────┐        ┌────────────────────────────┐
     │  OPENROUTER (Cloud)  │        │    OLLAMA  (Local)         │
     │  openrouter/auto     │        │    http://localhost:11434  │
     │  API key: <OPENROUTER_API_KEY>│                            │
     │  Context: 1,953k     │        │  ┌─────────────────────┐  │
     │  Input: text+image   │        │  │ qwen3:8b    (128k)  │← fallback #1
     └──────────────────────┘        │  │ qwen3:latest (40k)  │← fallback #2
                                     │  │ qwen3-fast:latest   │  │
                                     │  │ llama3.2:3b         │  │
                                     │  │ gpt-oss:latest(13GB)│  │
                                     │  │ qwen2.5:0.5b        │  │
                                     │  └─────────────────────┘  │
                                     └────────────────────────────┘

     ┌──────────────────────────────────────────────────────────────┐
     │                   FILE SYSTEM                                │
     │                                                              │
     │  ~/.openclaw/                                                │
     │    ├── openclaw.json          ← main config                  │
     │    ├── agents/main/agent/                                    │
     │    │     ├── auth-profiles.json  ← OpenRouter API key        │
     │    │     └── models.json                                     │
     │    ├── workspace/             ← agent memory & identity      │
     │    │     ├── SOUL.md                                         │
     │    │     ├── USER.md                                         │
     │    │     └── AGENTS.md                                       │
     │    ├── memory/main.sqlite     ← long-term memory             │
     │    └── credentials/           ← Telegram pairing             │
     │                                                              │
     │  /mnt/c/Users/<USERNAME>/     ← Windows drive (mounted)      │
     │    └── OPENCLAW_SETUP.md      ← your setup guide             │
     └──────────────────────────────────────────────────────────────┘

     USER ──→ Telegram ──→ Gateway ──→ OpenRouter/Ollama ──→ reply
---

## Server Info

| Item | Value |
|------|-------|
| OS | Linux (server) |
| Gateway URL | http://127.0.0.1:18789 |
| Dashboard URL | http://127.0.0.1:18789/#token=<GATEWAY_AUTH_TOKEN> |
| Auth Token | `<GATEWAY_AUTH_TOKEN>` |
| Config file | `~/.openclaw/openclaw.json` |
| Log file | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| OpenClaw binary | `~/.nvm/versions/node/vXX.XX.X/bin/openclaw` |

To access dashboard remotely, SSH tunnel from your client machine:
```bash
ssh -N -L 18789:127.0.0.1:18789 <your-user>@<server-ip>
```
Then open: `http://127.0.0.1:18789/#token=<GATEWAY_AUTH_TOKEN>`

---

## Models

### Primary: OpenRouter (Cloud)
- **Provider**: OpenRouter
- **Model**: `openrouter/auto` (auto-selects best available model)
- **API Key**: `<OPENROUTER_API_KEY>`
- **Auth file**: `~/.openclaw/agents/main/agent/auth-profiles.json`

### Fallbacks: Ollama (Local)
| Model | Size | Speed | Notes |
|-------|------|-------|-------|
| qwen3:8b | ~5GB | Medium | Fallback #1, reasoning |
| qwen3:latest | ~5GB | Medium | Fallback #2, reasoning |
| qwen3-fast:latest | — | Fast | Good for quick tasks |
| llama3.2:3b | ~2GB | Fast | Lightweight |
| gpt-oss:latest | 13GB | Slow (~11s) | Large, capable |
| qwen2.5:0.5b | ~400MB | Very Fast | Tiny, limited |

Ollama runs on: `http://localhost:11434`

---

## Channels

### Telegram
- **Status**: Enabled, running, polling mode
- **Bot Token**: `<TELEGRAM_BOT_TOKEN>`
- **Group policy**: Allowlist (only user ID `<TELEGRAM_USER_ID>` allowed)
- **DM policy**: Pairing required

---

## Key Commands

### Gateway
```bash
# Start gateway
openclaw gateway run

# Start in background (detached)
nohup openclaw gateway run > /tmp/openclaw-gw.log 2>&1 &

# Check if running
pgrep -fa openclaw
curl http://127.0.0.1:18789/health
```

### Models
```bash
openclaw models list                    # list configured models
openclaw models set <model-id>          # change primary model
openclaw models auth paste-token --provider openrouter   # update API key
```

### Channels
```bash
openclaw channels status                # check Telegram connection
openclaw channels status --deep         # deep health check
```

### Dashboard & TUI
```bash
openclaw dashboard                      # print dashboard URL with token
openclaw tui                            # terminal UI
```

### Logs
```bash
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

---

## Troubleshooting

### "LLM request timed out"
- Check Ollama is running: `curl http://localhost:11434/api/tags`
- Check config port is 11434 (not 11435): `grep baseUrl ~/.openclaw/openclaw.json`
- Restart gateway after config changes

### "agent failed" in Telegram
1. Check logs: `tail -50 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log`
2. Test model directly: `echo "hi" | ollama run qwen3:8b`
3. Restart gateway: kill old PID, run `openclaw gateway run`

### "unauthorized: gateway token missing" on dashboard
- Open: `http://127.0.0.1:18789/#token=<GATEWAY_AUTH_TOKEN>`
- Or run `openclaw dashboard` to get the full URL with token

### OpenRouter not responding
- Verify API key in `~/.openclaw/agents/main/agent/auth-profiles.json`
- Check OpenRouter status: https://openrouter.ai
- Fall back to Ollama: `openclaw models set ollama/qwen3:8b`

---

## File Locations

| File | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main config |
| `~/.openclaw/agents/main/agent/auth-profiles.json` | API keys |
| `~/.openclaw/agents/main/agent/models.json` | Model overrides |
| `~/.openclaw/workspace/` | Agent workspace (SOUL.md, USER.md, etc.) |
| `~/.openclaw/memory/main.sqlite` | Agent long-term memory |
| `/tmp/openclaw/` | Runtime logs |

---

## Update OpenClaw
```bash
npm update -g openclaw
```
