# Understanding & Setting Up a $0/mo AI GTM Assistant with OpenClaw

## Context

This plan explains how the "Boba" AI assistant works — a self-hosted AI agent
that runs GTM (go-to-market) operations for a startup 24/7, controlled entirely
through a single Telegram chat. The goal is to understand the architecture and
provide a step-by-step guide to replicate it on a laptop, then optionally
migrate to a free Oracle ARM server for always-on operation.

---

## 1. Architecture Overview

### What OpenClaw Is

OpenClaw is an open-source (MIT) personal AI assistant framework — a self-hosted
Node.js gateway that bridges messaging platforms with AI agents powered by LLMs.

```
Telegram ──┐
WhatsApp ──┤                  ┌── Claude (LLM brain)
Discord ───┼── Gateway:18789 ─┼── Skills (API integrations)
iMessage ──┤   (WebSocket)    ├── Browser (Playwright/CDP)
Signal ────┘                  ├── Memory (SQLite + RAG)
                              └── Cron / Heartbeat (automation)
```

**Key components:**
- **Gateway** (`ws://127.0.0.1:18789`): Hub for sessions, routing, channel connections. Runs as a launchd (macOS) or systemd (Linux) service
- **Agent Runtime**: Receives messages, invokes LLM, executes tool calls, manages session context
- **Skills**: Modular SKILL.md files — 5,700+ on ClawHub registry. Each skill teaches the agent how to use a specific API or tool
- **Memory**: Markdown files indexed into per-agent SQLite DB with hybrid BM25 + vector search
- **Cron/Heartbeat**: Scheduled tasks (exact time) and periodic monitoring sweeps (approximate intervals)

### The $0/mo Stack

| Layer | Service | Why Free |
|-------|---------|----------|
| Compute | Oracle ARM free tier | 4 OCPUs, 24 GB RAM, 200 GB — Always Free |
| LLM | Claude Opus via Azure | Founders Hub credits ($1K+) |
| Embeddings | Gemini embedding-001 | Google AI free tier (1,500 req/day) |
| Network | Tailscale | Free for personal use |
| Framework | OpenClaw | MIT open source |
| Browser | Headless Chromium | Open source |
| Interface | Telegram Bot | Free via BotFather |

---

## 2. How Each GTM Feature Works

### Morning Briefing (9 AM daily)
- **Mechanism**: OpenClaw **cron job** in an isolated session
- Cron fires at 9 AM, starts a fresh agent session with a prompt like "Generate morning GTM briefing"
- Agent calls skills for: Google Calendar, Gmail, Attio CRM, Instantly cold email
- Assembles and sends formatted briefing to Telegram via `--announce --channel telegram`

### Telegram Group Monitoring (24/7)
- **Mechanism**: External **Python script** (Telethon/Pyrogram) running as a systemd service
- Connects to Telegram as a *user account* (not bot API — needed to read group messages)
- Watches 38+ groups for ICP-matching keywords
- When match found, forwards alert to the OpenClaw Telegram bot
- OpenClaw agent can then act on it (look up person, add to CRM, draft reply)

### LinkedIn Campaign Tracking (HeyReach)
- **Mechanism**: Webhook pipeline: **HeyReach → Val.town → Telegram**
- HeyReach fires webhook on new connection/reply
- Val.town serverless function formats and forwards to Telegram Bot API
- OpenClaw receives the message and can act on it

### Cold Email Tracking (Instantly)
- **Mechanism**: Custom **skill** that calls Instantly API
- Heartbeat or cron checks campaign stats, reply rates, bounce rates
- Flags anomalies and sends alerts

### CRM Management (Attio)
- **Mechanism**: Custom **skill** with Attio API integration
- Natural language via Telegram: "add this person to the pipeline"
- Skill handles REST calls to create/update records

### Lead Enrichment Pipeline
- **Mechanism**: Chained skill execution
- "Find SF-based CFOs" → Apollo API search → enrichment → Attio CRM + Notion + Google Sheets
- Agent chains multiple skills based on workflow instructions in SOUL.md

### Proactive Check-ins (every 30 min)
- **Mechanism**: OpenClaw **heartbeat** system
- Gateway fires heartbeat every 30 min during active hours
- Agent reads HEARTBEAT.md checklist, picks 2-3 items per cycle
- If nothing notable: returns `HEARTBEAT_OK` (silently suppressed)
- If urgent: sends alert to Telegram

### Persistent Memory
- **Daily logs**: `memory/YYYY-MM-DD.md` — append-only, auto-loaded
- **Long-term memory**: `MEMORY.md` — curated facts (ICP, competitors, deal history, calendar links)
- **Vector search**: Gemini embeddings indexed in SQLite — hybrid semantic + keyword retrieval

---

## 3. Local Laptop Setup (Step-by-Step)

### 3.1 Prerequisites
- **Node.js 22+**: `curl -fsSL https://fnm.vercel.app/install | bash && fnm install 22`
- macOS, Linux, or WSL2
- Telegram account
- API keys: Anthropic (or Azure endpoint), Google AI (Gemini embeddings)

### 3.2 Install OpenClaw
```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```
The onboard wizard creates `~/.openclaw/`, installs the gateway as a service, and prompts for model + channel config.

### 3.3 Verify
```bash
openclaw status --all --deep
openclaw doctor --deep
openclaw logs --follow
```

### 3.4 Configure Model Provider

Edit `~/.openclaw/openclaw.json`:
```json5
{
  "agent": {
    "model": "anthropic/claude-opus-4-6"
  },
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "sk-ant-..."
      }
    }
  }
}
```

### 3.5 Set Up Telegram Bot

1. Message `@BotFather` on Telegram → `/newbot` → copy token
2. Add to `~/.openclaw/openclaw.json`:
```json5
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "123456789:ABCdefGHI...",
      "dmPolicy": "pairing",
      "allowFrom": []
    }
  }
}
```
3. Restart gateway: `openclaw gateway restart`
4. Send a message to your bot — approve the pairing code

### 3.6 Configure Memory with Gemini Embeddings

Add to `~/.openclaw/openclaw.json`:
```json5
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "gemini",
        "model": "gemini-embedding-001",
        "remote": {
          "apiKey": "YOUR_GEMINI_API_KEY"
        }
      }
    }
  }
}
```

### 3.7 Write Your Workspace Files

In `~/.openclaw/workspace/`:

**SOUL.md** — Persona and behavioral instructions:
```markdown
You are Boba, a GTM operations assistant for [company].
You manage CRM, outreach campaigns, lead enrichment, and scheduling.
Tone: concise, data-driven, proactive.
```

**MEMORY.md** — Curated business context:
```markdown
## ICP
- Series A-C B2B SaaS, 50-500 employees
- Pain: manual outbound, low reply rates

## Competitors
- CompetitorA: enterprise focus, $5K/mo+

## Active Campaigns
- LinkedIn via HeyReach: VP Sales at mid-market SaaS
- Cold email via Instantly: 25 accounts, 5 domains

## Calendar Links
- US leads: https://cal.com/...
- EU leads: https://cal.com/.../eu
```

**HEARTBEAT.md** — Periodic monitoring checklist:
```markdown
Rotate through 2-3 checks per cycle:
- Gmail: unreplied emails older than 2 hours
- Calendar: meetings in next 60 minutes
- Instantly: campaign anomalies (bounce spikes, paused sending)
- Attio: deals that changed stage in last hour
- If urgent → alert on Telegram. Otherwise → HEARTBEAT_OK
```

### 3.8 Configure Heartbeat

```json5
// In ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m"
      }
    }
  }
}
```

### 3.9 Build Custom Skills

Create skills in `~/.openclaw/skills/<name>/SKILL.md`. Example for Attio CRM:

```markdown
---
name: attio-crm
description: Manage Attio CRM — people, companies, deals, pipeline stages
---

# Attio CRM Skill

## API Config
- Base URL: https://api.attio.com/v2
- Auth: Bearer token in Authorization header

## Endpoints
- POST /objects/people/records - Create person
- POST /objects/people/records/query - Search people
- POST /lists/{list_id}/entries - Add to pipeline
- PATCH /lists/{list_id}/entries/{entry_id} - Update stage

## Instructions
When user says "add [person] to the pipeline":
1. Search for existing record
2. If not found, create new person record
3. Add to sales pipeline list
4. Confirm with summary
```

Repeat for: Instantly, Google Calendar, Gmail, Apollo, Notion, Google Sheets.

### 3.10 Add Cron Jobs

```bash
# Morning briefing
openclaw cron add \
  --name "Morning GTM briefing" \
  --cron "0 9 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Generate morning briefing: calendar, emails, pipeline, campaigns" \
  --announce --channel telegram

# Weekly pipeline review
openclaw cron add \
  --name "Weekly pipeline review" \
  --cron "0 10 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Generate weekly pipeline report from Attio: deals by stage, win rates, stale deals" \
  --announce --channel telegram
```

---

## 4. Path to Production: Oracle ARM Migration

### 4.1 Provision Oracle ARM Instance
1. Create account at cloud.oracle.com (credit card required, never charged)
2. Compute → Instances → Create: `VM.Standard.A1.Flex`, 4 OCPU, 24 GB RAM, Ubuntu 24.04 aarch64
3. Add SSH key, create instance

### 4.2 Server Setup
```bash
ssh ubuntu@<public-ip>
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential
sudo hostnamectl set-hostname openclaw
sudo loginctl enable-linger ubuntu
```

### 4.3 Install Tailscale (Secure Access)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname=openclaw
```

### 4.4 Install Node.js + OpenClaw
```bash
curl -fsSL https://fnm.vercel.app/install | bash
source ~/.bashrc
fnm install 22 && fnm use 22
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### 4.5 Harden Firewall
In Oracle Cloud Console → VCN → Security Lists:
- Remove all default ingress rules
- Add only: UDP 41641 (Tailscale)
- Keep default egress

### 4.6 Transfer Config & Workspace
```bash
# From laptop (via Tailscale hostname)
rsync -avz ~/.openclaw/workspace/ ubuntu@openclaw:~/.openclaw/workspace/
rsync -avz ~/.openclaw/skills/ ubuntu@openclaw:~/.openclaw/skills/
# Manually copy openclaw.json with updated API keys
```

### 4.7 Deploy Telegram Monitor Script
Install as systemd service on the Oracle instance for 24/7 group monitoring.

### 4.8 Verify
```bash
systemctl --user status openclaw-gateway
tailscale status
openclaw doctor --deep
```

---

## 5. Troubleshooting

| Problem | Fix |
|---------|-----|
| Gateway not running | `openclaw gateway start` or `systemctl --user start openclaw-gateway` |
| Bot not replying | `openclaw pairing list` — approve pending, or check `allowFrom` |
| Auth expired | `openclaw models auth setup-token` |
| Memory search empty | `openclaw memory index --all` |
| Skill not loading | `openclaw doctor --deep` — check SKILL.md frontmatter |
| Oracle ARM won't provision | Retry during off-peak UTC hours |
| High API costs later | Use cheaper models for heartbeat; Claude only for complex tasks |

---

## 6. Implementation Order

1. Install OpenClaw on laptop, pair Telegram bot
2. Configure Claude model provider + Gemini embeddings
3. Write SOUL.md, MEMORY.md, HEARTBEAT.md with your GTM context
4. Build first skill (Attio CRM) and test via Telegram
5. Add morning briefing cron job
6. Enable heartbeat for proactive monitoring
7. Build remaining skills (Instantly, Calendar, Gmail, Apollo)
8. Set up HeyReach webhook pipeline (Val.town → Telegram)
9. Write Telegram group monitoring Python script
10. Test locally for 1-2 weeks
11. Provision Oracle ARM, migrate, harden, verify

---

## Key References

- Official docs: https://docs.openclaw.ai/
- GitHub: https://github.com/openclaw/openclaw
- Oracle deployment guide: https://docs.openclaw.ai/platforms/oracle
- Skills registry: ClawHub (accessible via `openclaw skills search`)
- Cron docs: https://docs.openclaw.ai/automation/cron-jobs
