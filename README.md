# Rudra

**A proactive AI ops assistant for tech professionals. It learns how you work, offers to automate what you repeat, and audits every run it does.**

> **Status:** planning / pre-alpha. Nothing runnable yet. The full design and roadmap are in [docs/PLAN.md](docs/PLAN.md).

---

## The problem

Developers, tech leads and PMs lose hours every week to repetitive "ops" work: compiling status updates from Jira, Git and Slack, first-pass PR reviews, triaging email, researching and summarizing, keeping tickets tidy.

Today's automation tools (Zapier, n8n, agent builders) share two gaps:

1. **You have to spot the task, then describe the automation yourself.** Most people never get around to it.
2. **Agents report "done" whether or not the work was done right.** Nobody checks their output, so you can't trust them with anything that matters.

## What Rudra does

Rudra closes both gaps.

- **It finds the work for you.** Rudra observes your desktop and browser activity (on-device, privacy-first), spots repeated patterns, and *proposes* automations: *"You build the sprint status mail every Friday (~35 min). Want me to draft it every Thursday at 5pm?"*
- **It adapts to you.** It learns your tone, working hours, tools and preferences from what you do and from every edit or rejection you make.
- **It runs agents for each task.** The **Rudra orchestrator** plans each workflow and spawns specialized agents (researcher, writer, PR reviewer, triager, browser operator…) with only the tools they need.
- **It audits every run.** Every step has a **success contract** defined *before* it runs. An independent auditor checks the real evidence against it. If the work falls short, Rudra re-runs it with feedback, re-plans, or asks you. It never pretends it's done.
- **It talks to you where you are:** voice, desktop notifications, Slack and the web app, within an interruption budget so it doesn't become noise.

## How it works

```
  Observe ──▶ Learn ──▶ Propose ──▶ Plan ──▶ Execute ──▶ Audit ──┬─▶ Commit ✓
 (desktop,   (patterns, (proactive  (workflow (agents,   (contract│
  browser,    style,     suggestion) spec +    staged     check)  ├─▶ Retry with feedback
  connected   memory)                contracts) actions)          ├─▶ Re-plan
  tools)                                                          └─▶ Ask you
```

### Core ideas

| Concept | What it means |
|---|---|
| **Success contract** | Each step declares upfront what "done right" means: deterministic checks (all tickets covered, ≤300 words, correct recipients), LLM-judged qualities (tone matches yours) and security rules (only the declared tools were used). |
| **Independent auditor** | Sees the contract and the evidence, never the agent's own explanation. Runs cheap deterministic checks first and calls an LLM judge only when needed. |
| **Stage → audit → commit** | Side effects (emails, PR comments, Slack posts) are staged as drafts and committed only after they pass the audit, so a retry never sends anything twice. |
| **Trust ladder** | Each workflow earns autonomy: *Suggest → Draft → Supervised → Autonomous*. Clean audited runs promote it; failures or rejections demote it. |
| **Workflow graduation** | Once a workflow is stable, its deterministic steps are compiled into plain code, so it gets cheaper and more reliable over time. |

## Architecture

```
Clients:   Web app (React) · Desktop app (Tauri, Windows) · Browser extension · Slack bot
              │  REST + Server-Sent Events
Backend:   Python (FastAPI) modular monolith
              ├─ Orchestrator: Planner · Agent Factory · Auditor · Recovery · Trust Ladder
              ├─ Durable execution: Temporal
              ├─ LLM gateway: priority scheduler + LiteLLM (multi-provider, fallbacks, BYOK)
              ├─ Tools: MCP servers (Gmail, Calendar, Slack, GitHub, Jira, web search, browser)
              ├─ Memory: profile · episodic · procedural (pgvector)
              └─ Proactive engine + pattern miner
Data:      Postgres + pgvector (Supabase) · Langfuse (LLM tracing)
```

## Tech stack

| Area | Choice |
|---|---|
| Backend | Python 3.12+, FastAPI, Pydantic, SQLAlchemy, Alembic, `uv` |
| Agents | Google ADK |
| LLMs | Gemini (primary), Groq (fallback), with OpenAI / Claude / Grok pluggable via LiteLLM. Cloud APIs only. |
| Workflows | Temporal |
| Data & auth | Supabase (Postgres, pgvector, Auth, Storage) |
| Tools | Model Context Protocol (MCP), Playwright |
| Frontend | React, Vite, TypeScript, Tailwind, shadcn/ui, pnpm + Turborepo |
| Desktop | Tauri 2 (Windows first) |
| Realtime | Server-Sent Events + REST |
| Observability | Langfuse, OpenTelemetry |

**Cost model:** free tiers during development; in production, each tenant brings their own LLM keys (BYOK).

## Privacy principles

- Raw screen and keyboard activity **never leaves your machine**. It's redacted and summarized on-device first.
- Observation is **opt-in**, with pause, per-app exclusions and a visible indicator.
- Every external action is staged and auditable; nothing is sent behind your back.
- Your data is exportable and deletable at any time.

## Roadmap

| Milestone | Scope |
|---|---|
| **M0** Foundation | Monorepo, Supabase schema, FastAPI skeleton, Temporal worker, LLM gateway |
| **M1** Rudra core | Workflow specs, agent factory, auditor, retry/re-plan/escalate |
| **M2** Interfaces + connectors | Web app, Slack bot, Gmail/Calendar/GitHub/Jira, triggers, trust ladder |
| **M3** Memory & user model | Learn style and preferences from feedback |
| **M4** Desktop app | Tauri app, notifications, voice, on-device observation |
| **M5** Proactive engine | Pattern mining, opportunity scoring, suggestions |
| **M6** SaaS hardening | Multi-tenancy, BYOK, security, cloud deploy |
| **M7** Graduation & polish | Compiled workflows, Mac app, more connectors |

Details, exit criteria and task lists: [docs/PLAN.md](docs/PLAN.md).

## Repository layout (planned)

```
backend/       Python backend (api, orchestrator, workflows, gateway, tools, memory…)
apps/web/      React web app
apps/desktop/  Tauri desktop app
apps/extension/ Chrome extension
packages/      Shared UI, app core, generated API client
mcp-servers/   Custom MCP servers
infra/         Deployment config
docs/          Design docs
```

## License

TBD
