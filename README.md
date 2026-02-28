# 🔐 TokPinch
**Stop burning money on AI API calls.**

TokPinch is a cost management proxy for [OpenClaw](https://github.com/openclaw/openclaw). It sits between your agent and LLM providers, tracking every token, enforcing budgets, detecting runaway loops, and routing to cheaper models — automatically.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Tests](https://img.shields.io/badge/tests-97%2F97-brightgreen)]()
[![Node](https://img.shields.io/badge/node-%3E%3D20-blue)]()

---

## Why TokPinch?

**One line to set up:**
```bash
ANTHROPIC_BASE_URL=http://localhost:4100/v1
```

**What you get:**
- 📊 Real-time cost tracking across Anthropic & OpenAI
- 💰 Budget enforcement — daily & monthly limits that actually block requests
- 🔄 Loop detection — catches runaway agents before they drain your wallet
- 🧠 Smart routing — auto-downgrade cheap tasks to Haiku (save 10–50%)
- 📱 Telegram & email alerts with daily cost digests
- 🖥️ Beautiful dark-mode dashboard with live WebSocket updates
- 🔒 Security hardened — API keys never stored or logged
- 🐳 Docker ready with one command

---

## Quick start

```bash
# 1. Pull and run (SQLite data persisted in a named volume)
docker run -d \
  --name tokpinch \
  -p 4100:4100 \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -e DASHBOARD_PASSWORD=changeme \
  -v tokpinch-data:/app/data \
  ghcr.io/your-org/tokpinch:latest

# 2. Open the dashboard
open http://localhost:4100/dashboard
```

Or with Docker Compose (recommended):

```bash
cp .env.example .env   # fill in ANTHROPIC_API_KEY and DASHBOARD_PASSWORD
docker compose up -d
```

> **Note — passwords containing `!`:** Bash expands `!` in double-quoted strings and some
> single-quoted contexts (history expansion). When using `curl` to call
> `/api/auth/login`, write the JSON body to a file and use `--data @file` to
> avoid shell escaping issues:
>
> ```bash
> echo '{"password":"YourP@ss!"}' > /tmp/body.json
> curl -s -X POST http://localhost:4100/api/auth/login \
>   -H 'Content-Type: application/json' \
>   --data @/tmp/body.json
> ```

---

## OpenClaw / Claude Code integration

One environment variable is all it takes. Point your agent at TokPinch instead of Anthropic directly:

```bash
# Before
ANTHROPIC_BASE_URL=https://api.anthropic.com

# After
ANTHROPIC_BASE_URL=http://localhost:4100/v1
```

Your `ANTHROPIC_API_KEY` stays in place — TokPinch forwards it upstream transparently.

---

## Environment variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | Yes* | — | Forwarded to Anthropic for every proxied request |
| `OPENAI_API_KEY` | Yes* | — | Forwarded to OpenAI for every proxied request |
| `DASHBOARD_PASSWORD` | **Yes** | — | Password for the web dashboard (min 8 chars) |
| `JWT_SECRET` | No | auto-generated | Secret for signing dashboard JWTs (set for stable restarts) |
| `PORT` | No | `4100` | Port the proxy listens on |
| `DB_PATH` | No | `./data/tokpinch.db` | SQLite database path |
| `BUDGET_DAILY` | No | unlimited | Daily spend limit in USD (e.g. `10.00`) |
| `BUDGET_MONTHLY` | No | unlimited | Monthly spend limit in USD (e.g. `100.00`) |
| `LOOP_DETECTION_ENABLED` | No | `true` | Detect and throttle looping agents |
| `LOOP_MAX_RPM` | No | `20` | Max requests/min before loop cooldown triggers |
| `LOOP_COOLDOWN_SECONDS` | No | `300` | How long to pause a looping session (seconds) |
| `SMART_ROUTING_ENABLED` | No | `false` | Enable automatic model downgrading rules |
| `LOG_LEVEL` | No | `info` | Pino log level (`fatal`/`error`/`warn`/`info`/`debug`) |
| `ALERT_TELEGRAM_TOKEN` | No | — | Telegram bot token for budget/loop alerts |
| `ALERT_TELEGRAM_CHAT_ID` | No | — | Telegram chat ID to send alerts to |
| `ALERT_EMAIL_TO` | No | — | Email address to receive alerts |
| `ALERT_EMAIL_SMTP_HOST` | No | — | SMTP server hostname |
| `ALERT_EMAIL_SMTP_PORT` | No | `587` | SMTP server port |
| `ALERT_EMAIL_SMTP_USER` | No | — | SMTP username |
| `ALERT_EMAIL_SMTP_PASS` | No | — | SMTP password |

*At least one of `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` is required depending on which provider you proxy.

---

## Configuration

### Smart routing rules

Routing rules live in `data/routing-rules.json` and define which models get downgraded and to what tier. Example:

```json
{
  "models": {
    "claude-opus-4": {
      "mid":   "claude-sonnet-4-5",
      "cheap": "claude-haiku-4-5-20251001"
    }
  }
}
```

> **Note:** Routing rules are loaded from `data/routing-rules.json` at startup and cached in memory. Changes to this file require a server restart to take effect.

---

## Dashboard

Navigate to `http://localhost:4100/dashboard` and log in with your `DASHBOARD_PASSWORD`.

<!-- Add dashboard screenshots here -->
![Overview](screenshots/overview.png)
![Sessions](screenshots/sessions.png)
![Budget](screenshots/budget.png)

Pages: **Overview** · **Sessions** · **Budget** · **Alerts** · **Settings**

---

## Architecture

```
  Your Agent / IDE
  (Claude Code, Cursor, OpenClaw, ...)
        │
        │  POST /v1/messages  (same API, just different host)
        ▼
┌───────────────────────────────────────────┐
│               TokPinch Proxy              │
│                                           │
│  ┌─────────────┐   ┌──────────────────┐  │
│  │ Loop        │   │ Budget Check     │  │
│  │ Detector    │──▶│ (daily/monthly)  │  │
│  └─────────────┘   └──────────────────┘  │
│         │                   │             │
│  ┌──────▼───────────────────▼──────────┐ │
│  │         Model Router                │ │
│  │  (optional: downgrade costly models)│ │
│  └─────────────────────────────────────┘ │
│                    │                      │
│         ┌──────────▼──────────┐           │
│         │  Upstream API call  │           │
│         │  Anthropic / OpenAI │           │
│         └──────────┬──────────┘           │
│                    │                      │
│         ┌──────────▼──────────┐           │
│         │  Cost calculation   │           │
│         │  SQLite logging     │           │
│         │  WebSocket broadcast│           │
│         └─────────────────────┘           │
│                                           │
│  ┌──────────────────────────────────────┐ │
│  │  Dashboard  (React · /dashboard)     │ │
│  │  REST API   (/api/*)  + JWT auth     │ │
│  │  WebSocket  (/ws)     live feed      │ │
│  └──────────────────────────────────────┘ │
└───────────────────────────────────────────┘
        │
        ▼
   SQLite  +  Alerter
   (data/tokpinch.db)   (Telegram · Email)
```

---

## Proxy endpoints

| Endpoint | Forwards to |
|---|---|
| `POST /v1/*` | `https://api.anthropic.com/v1/*` |
| `POST /openai/v1/*` | `https://api.openai.com/v1/*` |

Set your agent's base URL to `http://localhost:4100` (Anthropic) or `http://localhost:4100/openai` (OpenAI).

### Session tracking header

To enable loop detection and per-session cost grouping, include a session ID on every request:

```
x-openclaw-session: my-agent-session-1
```

The alias `x-session-id` is also accepted for compatibility with tools that already set it:

```
x-session-id: my-agent-session-1   # equivalent — TokPinch accepts both
```

If neither header is present, requests are still proxied and logged, but loop detection is disabled for that request.

---

## Contributing

1. Fork the repo and create a branch: `git checkout -b feat/your-feature`
2. Install dependencies: `npm install && cd dashboard && npm install`
3. Run the proxy in dev mode: `npm run dev`
4. Run the dashboard in dev mode (separate terminal): `npm run dashboard:dev`
5. Run tests: `npm test`
6. Open a pull request — please include a short description of what changed and why

Code style: TypeScript strict mode, no `any` without a comment, Zod for all external input validation, `pino` for logging (no `console.log`).

---

## License

MIT © 2025 TokPinch contributors

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
