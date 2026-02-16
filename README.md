# OpenClaw GTM Assistant

A $0/mo AI assistant that runs startup GTM (go-to-market) operations 24/7 using [OpenClaw](https://github.com/openclaw/openclaw), free-tier cloud services, and Telegram as the control surface.

## What This Does

A single Telegram chat that orchestrates your entire outbound pipeline with deep business context:

- **Morning briefings** — calendar, urgent emails, pipeline by stage, campaign stats delivered at 9 AM
- **Telegram group monitoring** — watches 38+ groups 24/7 for ICP-matching leads
- **LinkedIn outreach** — manages campaigns via HeyReach, pings on new connections/replies
- **Cold email tracking** — monitors accounts across multiple domains via Instantly
- **CRM management** — natural language commands to Attio ("add this person to the pipeline")
- **Lead enrichment** — Apollo search to CRM + Notion + Google Sheets in one shot
- **Proactive check-ins** — rotates through email, calendar, and campaign checks every 30 minutes

## The Stack (All Free)

| Layer | Service | Cost |
|-------|---------|------|
| Compute | Oracle ARM free tier (4 cores, 24GB RAM) | $0 |
| AI | Claude Opus via Azure (Founders Hub credits) | $0 |
| Embeddings | Gemini on Google AI free tier | $0 |
| Network | Tailscale | $0 |
| Framework | OpenClaw (open source) | $0 |
| Browser | Headless Chromium | $0 |
| Interface | Telegram Bot | $0 |

## Architecture

```
Telegram ──┐
WhatsApp ──┤                  ┌── Claude (LLM brain)
Discord ───┼── Gateway:18789 ─┼── Skills (API integrations)
iMessage ──┤   (WebSocket)    ├── Browser (Playwright/CDP)
Signal ────┘                  ├── Memory (SQLite + RAG)
                              └── Cron / Heartbeat (automation)
```

OpenClaw acts as a gateway between messaging platforms and AI agents. The agent runtime invokes LLMs, executes tool calls via skills, and persists context through a local SQLite-based memory system with hybrid semantic + keyword search.

## Quick Start

### Prerequisites

- Node.js 22+
- macOS, Linux, or WSL2
- Telegram account
- API keys: Anthropic (or Azure), Google AI (Gemini embeddings)

### Install

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### Set Up Telegram Bot

1. Message `@BotFather` on Telegram, send `/newbot`, copy the token
2. Add to `~/.openclaw/openclaw.json`:

```json5
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TOKEN",
      "dmPolicy": "pairing"
    }
  }
}
```

3. `openclaw gateway restart` and message your bot to pair

### Add Automation

```bash
# Morning briefing at 9 AM
openclaw cron add \
  --name "Morning GTM briefing" \
  --cron "0 9 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Generate morning briefing: calendar, emails, pipeline, campaigns" \
  --announce --channel telegram
```

### Configure Heartbeat

Add to `~/.openclaw/openclaw.json`:

```json5
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "30m" }
    }
  }
}
```

Create `~/.openclaw/workspace/HEARTBEAT.md` with your monitoring checklist.

## Documentation

See [plan.md](plan.md) for the full detailed guide covering:

- How each GTM capability works under the hood
- Step-by-step local laptop setup
- Custom skill examples (Attio CRM, Instantly, Apollo)
- Memory and embedding configuration
- Oracle ARM migration for always-on $0/mo operation
- Troubleshooting reference

## Key Integrations

| Integration | Method |
|-------------|--------|
| Attio CRM | Custom OpenClaw skill (REST API) |
| Instantly (cold email) | Custom OpenClaw skill (REST API) |
| HeyReach (LinkedIn) | Webhook pipeline: HeyReach -> Val.town -> Telegram |
| Google Calendar / Gmail | OpenClaw skills |
| Apollo (lead sourcing) | Custom OpenClaw skill (REST API) |
| Telegram groups | External Python script (Telethon/Pyrogram) |

## References

- [OpenClaw](https://github.com/openclaw/openclaw) — the open-source AI assistant framework
- [OpenClaw Docs](https://docs.openclaw.ai/) — official documentation
- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) — always-free ARM compute
- [Tailscale](https://tailscale.com/) — secure mesh networking
