# MySquad & MyGuy — n8n Automation Workflows

A collection of production n8n automation workflows powering the **MySquad** AI assistant platform and **MyGuy** personal AI agent. Built on a self-hosted n8n instance with PostgreSQL, Groq LLM, WhatsApp Business API, Telegram, and Google integrations.

---

## Workflows

### 🤖 AI Agents

| Workflow | Description |
|---|---|
| `MyGuy Sub Workflow` | Core AI agent — handles incoming messages, runs Groq LLM with Postgres chat memory, executes tools (RSS, Google, reminders, tasks) |
| `AI Bot` | General-purpose AI bot with Google Drive document ingestion and PGVector semantic search |
| `Google Sub Workflow` | Sub-agent for Google Workspace actions (Gmail, Calendar, Sheets, Docs) via the FastAPI MCP server |

### 📱 Platform Interfaces

| Workflow | Description |
|---|---|
| `MySquadWHATSAPP` | WhatsApp Business webhook receiver — routes messages to the AI agent, handles media via media server |
| `MySquadTELEGRAM` | Telegram bot interface — same routing logic as WhatsApp |
| `MySquadAPP` | REST API interface for the MySquad mobile/web app |

### 📚 Applications

| Workflow | Description |
|---|---|
| `MentorMe` | Full mentorship program automation — 150-node workflow managing mentor/mentee pairing, weekly emails, goal setting, AI feedback, and quarterly cycles |
| `Academic Research Assistant` | Google Drive-triggered workflow that processes research papers, extracts metadata, and surfaces scholarship opportunities |
| `Content Creator` | Multi-platform content pipeline — generates and publishes to Instagram and other platforms via AI |
| `Task Management` | Webhook-driven task and project management system backed by PostgreSQL |
| `TreasAlert` | Treasury alert system — monitors financial data, generates PDF reports, sends email/SMS alerts |
| `Daily Morning Brief` | Scheduled morning briefing — fetches news per user interest, combines with reminders, delivers via WhatsApp |

### 🔧 Sub-workflows & Utilities

| Workflow | Description |
|---|---|
| `RSS Sub Workflow` | Fetches, filters, and shortens RSS articles for a given interest topic |
| `Speedy Sub Workflow` | Handles media forwarding to the Speedy WhatsApp gateway (with and without image) |
| `Reminder Alerts` | Scheduled reminder delivery with AI-enhanced messaging and recurrence logic |
| `Token Logger` | Logs LLM token usage and tool call metrics to Google Sheets for cost tracking |

---

## Prerequisites

| Requirement | Purpose |
|---|---|
| n8n (self-hosted, v1.x+) | Workflow engine |
| PostgreSQL + pgvector | Chat memory, user data, vector search |
| Groq API key | LLM inference |
| WhatsApp Business API | WhatsApp integration |
| Telegram Bot Token | Telegram integration |
| Google Cloud OAuth app | Gmail, Calendar, Sheets, Docs |
| MinIO instance | Media file storage |
| SMTP / SendGrid | Email delivery |

---

## Setup

### 1. Import a workflow
In your n8n instance: **Settings → Import from file** → select the JSON file.

### 2. Configure credentials
Each workflow uses placeholder credential IDs (e.g. `__GROQ_CRED_ID__`, `__POSTGRES_CRED_ID__`). After importing, open each node that shows a credential error and connect your own credentials from **Settings → Credentials**.

### 3. Required credentials to create

| Credential Type | Used By |
|---|---|
| Groq API | All AI agent workflows |
| PostgreSQL | MySquad*, MyGuy, Task Management, MentorMe |
| WhatsApp API | MySquadWHATSAPP, Speedy Sub Workflow |
| Telegram API | MySquadTELEGRAM |
| Google API (OAuth2) | Google Sub Workflow, Token Logger, TreasAlert |
| HTTP Header Auth | Speedy Sub Workflow (media server) |
| SMTP | MentorMe, TreasAlert |
| Google Sheets | Token Logger, TreasAlert |

---

## Security

All workflows in this repository have been sanitized:
- API keys, tokens, and credentials replaced with `REDACTED` or `__PLACEHOLDER__`
- Instance IDs, workflow IDs, and version IDs replaced with `__INSTANCE_ID__` etc.
- Phone numbers, email addresses, and platform user IDs removed from test data
- Webhook URLs anonymised

Never commit `.env` files or raw workflow exports without sanitizing first.

---

## Related Repositories

- **[mysquad-server](https://github.com/Adekunle-1/mysquad-server)** — FastAPI backend (Google MCP server + media server)
