# Rudra — System Design & Build Plan

> Rudra is a proactive AI ops assistant for tech professionals. It learns how you work, proposes automations for what you repeat, spawns agents to run them, and **audits every run** — re-running or escalating when the result doesn't meet its success contract.

---

## 1. Decisions log

| # | Decision | Choice |
|---|---|---|
| D1 | Audience | Multi-user SaaS |
| D2 | Niche | Ops work of tech professionals (devs, leads, PMs): status reports, PR review, email/Slack triage, research, ticket hygiene |
| D3 | Clients | Web app + Windows desktop app (shared React UI). Mac later. |
| D4 | Observation | Full desktop + browser activity, processed **on-device**, only redacted episodes leave the machine |
| D5 | Talk channels | Voice, desktop notifications, Slack (+ web chat) |
| D6 | Hosting | Local during development → cloud (GCP) for deployment |
| D7 | LLMs | Cloud APIs only, **no local models**. **Gemini primary, Groq fallback**; Grok/OpenAI/Claude pluggable later. Provider-agnostic via LiteLLM. |
| D8 | Budget | ~Zero. Free tiers in dev; **BYOK** (tenant brings own keys) in production |
| D9 | Backend | Python |
| D10 | Build order | Orchestrator + Auditor core first; desktop observation later |

| D11 | Dev environment | Machine has 8 GB RAM / 2-core i3, so **no Docker in daily dev**. Supabase cloud + Langfuse cloud (free tiers), native Temporal dev server. Docker only for deploy-image testing. See §15. |

---

## 2. Architecture

```
┌──────────────────────────── CLIENTS ─────────────────────────────────────────┐
│ Web App (React)          Desktop App (Tauri 2 + same React UI)   Slack Bot   │
│  chat, runs, approvals    + Observers (UI Automation, clipboard)  approvals, │
│  workflows, settings      + Browser extension (MV3)                chat,     │
│                           + Sessionizer + rule-based redactor      digests   │
│                           + Voice (push-to-talk, Win STT/TTS)                │
│                           + Notifications + Local Actor                      │
└──────────────┬───────────────────────────┬───────────────────────┬───────────┘
               │ REST + SSE                │ episodes / commands   │ Slack events
┌──────────────▼───────────────────────────▼───────────────────────▼───────────┐
│                         BACKEND (Python, modular monolith)                    │
│  api ─────────── auth, tenants, REST, SSE stream hub, webhooks                │
│  orchestrator ── Planner · Agent Factory · Auditor · Recovery · Trust Ladder  │
│  workflows ───── Temporal workflows/activities (durable execution)            │
│  gateway ─────── LLM Scheduler (priority queues, rate limits, fallback) +     │
│                  LiteLLM + response cache + usage metering                    │
│  tools ───────── MCP client hub + staged-action manager                       │
│  memory ──────── profile / episodic / procedural memory, retrieval            │
│  proactive ───── opportunity scoring, interruption policy, channel router     │
│  miner ───────── episode store, sequence mining, pattern naming (later)       │
├───────────────────────────────────────────────────────────────────────────────┤
│ Postgres + pgvector (Supabase) │ Temporal │ Object storage │ Langfuse tracing │
│ MCP servers: Gmail, Calendar, Slack, GitHub, Jira, Web search, Web fetch,     │
│              Browser (Playwright)                                             │
└───────────────────────────────────────────────────────────────────────────────┘
```

**Why a modular monolith:** one Python codebase, two processes (`api` server and `temporal worker`). It's cheap to run on a light machine and splits into services later along the module boundaries.

---

## 3. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Backend | Python 3.12, **FastAPI**, Pydantic v2, SQLAlchemy 2 + Alembic | `uv` for env/deps |
| Agents | **Google ADK** | `LlmAgent` + `LiteLlm` model wrapper → any provider |
| LLM access | **LiteLLM** behind our own **Scheduler** | Gemini, Groq, Grok, OpenAI, Claude |
| Durable execution | **Temporal** (`temporal server start-dev` locally) | retries, timers, schedules, signals for approvals |
| DB / Auth / Storage | **Supabase** (Postgres + pgvector + Auth + Storage), free tier | Row-Level Security for tenant isolation |
| Tools | **MCP** servers (official where they exist, custom otherwise) | Playwright for browser actions |
| Web search | Gemini Google-Search grounding; Tavily / Brave free tier as fallback | |
| Embeddings | Gemini embedding API (batched) | |
| Web app | **React + Vite + TypeScript**, TanStack Router/Query, Tailwind, shadcn/ui | |
| Desktop | **Tauri 2** (Rust) reusing the web UI packages | Windows UI Automation, `Windows.Media.Ocr`, Windows STT/TTS |
| Browser ext | Chrome MV3 (TypeScript) | Observe + act |
| Slack | Slack Bolt (Python) | Block Kit approval buttons |
| Observability | **Langfuse** (LLM traces, cost), OpenTelemetry, structured logs | |
| JS monorepo | pnpm workspaces + Turborepo | |
| Deploy (later) | Docker → Cloud Run / GKE, Temporal Cloud or self-hosted | |

---

## 4. Repository layout

```
rudra/
├─ backend/                      # Python (uv project)
│  ├─ rudra/
│  │  ├─ api/                    # FastAPI routers, SSE hub, webhooks, auth deps
│  │  ├─ orchestrator/
│  │  │  ├─ planner.py           # intent → WorkflowSpec
│  │  │  ├─ agent_factory.py     # template + spec → ADK agent
│  │  │  ├─ auditor.py           # contract verification
│  │  │  ├─ recovery.py          # failure classification → retry/replan/escalate
│  │  │  ├─ trust.py             # trust ladder
│  │  │  └─ templates/           # agent templates (yaml)
│  │  ├─ workflows/              # Temporal workflows + activities, worker entrypoint
│  │  ├─ gateway/                # scheduler, litellm config, cache, metering
│  │  ├─ tools/                  # MCP hub, staged actions, check registry
│  │  ├─ memory/
│  │  ├─ proactive/
│  │  ├─ miner/
│  │  ├─ integrations/slack/
│  │  ├─ domain/                 # pydantic models: WorkflowSpec, Contract, Verdict…
│  │  └─ db/                     # sqlalchemy models, repositories
│  ├─ migrations/
│  └─ tests/
├─ apps/
│  ├─ web/                       # React web app
│  ├─ desktop/                   # Tauri app (src-tauri/ in Rust)
│  └─ extension/                 # Chrome MV3
├─ packages/
│  ├─ ui/                        # shared React components
│  ├─ app-core/                  # shared screens, state, hooks
│  └─ api-client/                # TS client generated from OpenAPI
├─ mcp-servers/                  # custom MCP servers (e.g. jira, rudra-internal)
├─ infra/                        # docker-compose, env templates, deploy
└─ docs/
```

---

## 5. Core concepts

### 5.1 WorkflowSpec
The planner's output and the unit Rudra executes. A DAG of steps, each with its own agent config, budget, side-effect policy and **success contract**.

```yaml
id: wf_weekly_status
name: Weekly sprint status mail
trigger: { type: schedule, cron: "0 17 * * THU", tz: "Asia/Kolkata" }
trust_level: 1                     # see 5.5
inputs: { sprint: "current" }
steps:
  - id: gather
    agent: { template: researcher, model_tier: fast, tools: [jira.search, github.list_prs] }
    budget: { max_steps: 10, max_tokens: 20000, timeout_s: 180, max_retries: 2 }
    output_schema: TicketDigest
    contract:
      deterministic:
        - check: covers_all
          args: { source: "jira.search(sprint=current, updated>=-7d)", field: tickets }
  - id: draft
    depends_on: [gather]
    agent: { template: writer, model_tier: standard, tools: [mail.draft] }
    budget: { max_steps: 8, max_tokens: 30000, timeout_s: 180, max_retries: 2 }
    side_effects: { kind: email, mode: stage_then_commit }
    contract:
      deterministic:
        - check: recipients_equal
          args: { expected: ["team@example.com"] }
        - check: max_words
          args: { n: 300 }
        - check: mentions_all
          args: { from_step: gather, field: tickets }
      llm_judge:
        - rubric: tone_matches_profile
          threshold: 0.8
        - rubric: no_fabricated_facts
      security:
        - only_declared_tools
```

### 5.2 Run lifecycle (state machine)

```
Run:  PENDING → PLANNING → RUNNING → AWAITING_APPROVAL → COMPLETED
                                  ↘ ESCALATED / FAILED / CANCELLED

Step attempt:
  EXECUTING → STAGED → AUDITING ─pass→ COMMITTING → DONE
                                └fail→ classify → RETRY | REPLAN | ESCALATE | HALT
```

Implemented as one Temporal workflow per run. Each step is a child workflow running `execute → stage → audit → commit` activities. Human approvals arrive as Temporal **signals** from the web app, desktop or Slack.

### 5.3 Auditor
- **Independent.** It sees the contract and the evidence (staged payloads, structured outputs, fresh tool read-backs). It never sees the worker's reasoning or self-assessment.
- **Layered, cheapest first:**
  1. **Security checks:** tool calls ⊆ declared tools, no unexpected recipients or domains.
  2. **Deterministic checks:** Python functions in a registry (`covers_all`, `max_words`, `recipients_equal`, `http_ok`, `file_exists`, `json_schema`…).
  3. **LLM judge:** only if layers 1–2 pass. Uses a rubric, returns structured JSON. Prefers a **different provider/model** from the worker.
- **Verdict output:**
```json
{ "passed": false,
  "failure_class": "AGENT_ERROR",
  "failed_checks": [{"check": "mentions_all", "detail": "missing PROJ-142, PROJ-151"}],
  "feedback": "Draft omits 2 tickets moved to Done: PROJ-142, PROJ-151.",
  "confidence": 0.97 }
```

### 5.4 Recovery policy

| Failure class | Example | Action | Limit |
|---|---|---|---|
| `TRANSIENT` | 429, network, tool 5xx | Temporal retry with backoff (not counted as an attempt) | per-activity policy |
| `AGENT_ERROR` | contract violated | re-run the step with auditor feedback injected | `max_retries` (default 2) |
| `SPEC_ERROR` | contract unsatisfiable, wrong plan | planner revises the spec from the failure context | 1 replan per run |
| `BLOCKED` | missing permission, ambiguous input | escalate: ask the user via their preferred channel | — |
| `SECURITY` | undeclared tool call, injection suspected | **halt**, discard staged actions, alert the user | — |

### 5.5 Side effects: stage → audit → commit
- Every external write (send mail, post Slack, comment on PR, create ticket) is first a **StagedAction** with an **idempotency key**. Examples: a Gmail draft, a dry-run payload.
- It's committed only after the audit passes and, if the trust level requires it, after user approval.
- Retries never duplicate side effects. A failed attempt's staged actions are discarded.

### 5.6 Trust ladder (per workflow)

| Level | Behaviour |
|---|---|
| L0 Suggest | Rudra only proposes the workflow |
| L1 Draft | Runs; every side effect needs approval |
| L2 Supervised | Internal effects auto-commit; external (mail to others, PR comments) need approval |
| L3 Autonomous | Auto-commits; the user gets a post-run summary |

- **Promotion:** 5 consecutive audited passes with no user edits/rejections.
- **Demotion:** any escalation, user rejection, or security flag.

### 5.7 Agent Factory
- **Templates** (yaml): `researcher`, `writer`, `pr_reviewer`, `triager`, `analyst`, `browser_operator`, `comms`. Each declares base instructions, allowed tool families and a default model tier.
- The factory merges template + step config + relevant **memory** (user style, prior feedback) → builds an ADK `LlmAgent` with a `LiteLlm` model and only the whitelisted MCP tools.
- Hard limits per agent: steps, tokens, wall time. Per tenant: concurrent agents cap.

### 5.8 Planner
- Input: user intent (chat/voice), a proactive suggestion, or a failure context (replan).
- Retrieves similar existing workflows (procedural memory) before writing a new one.
- Output: a `WorkflowSpec` validated by Pydantic. Invalid output → one self-repair attempt → escalate.
- New specs start at **L1** and are shown to the user before the first run.

---

## 6. LLM gateway & scheduler

```
caller → Scheduler (priority queue + token buckets per provider/key)
       → LiteLLM (router: tier → model list, fallbacks)
       → provider
       ← response → cache + usage log (Langfuse)
```

- **Priorities:** `P0` live voice/chat → `P1` auditor → `P2` workflow agents → `P3` background (mining, memory consolidation, nightly batches).
- **Tiers (config, not code):**

| Tier | Primary | Fallbacks | Used by |
|---|---|---|---|
| `fast` | Gemini Flash-Lite | Groq (small Llama/Qwen) | classification, extraction, gather steps |
| `standard` | Gemini Flash | Groq (large Llama/Qwen) | most agents |
| `strong` | Gemini Pro-class | Groq (largest available) → OpenAI/Claude/Grok (if key) | planner, LLM-judge |
| `embed` | Gemini embedding | — | memory, retrieval |

- **Rate-limit handling:** token buckets sized from config. On a 429 the scheduler marks the provider cooling-down and routes to the next fallback. `P3` work waits instead of falling back.
- **Cost control:** response cache (hash of model + prompt + tools), deterministic-first auditing, background batching, and later workflow graduation (§11).
- **BYOK:** per-tenant encrypted keys. The scheduler keeps per-key buckets. Free-tier keys are marked `dev_only`, because free tiers may use prompts for training.

---

## 7. Data model (Postgres)

| Table | Key columns |
|---|---|
| `tenants`, `users`, `memberships` | role, settings |
| `connections` | tenant, user, provider, scopes, encrypted tokens, status |
| `llm_keys` | tenant, provider, encrypted key, tier, `dev_only` |
| `agent_templates` | name, version, instructions, tool families, default tier |
| `workflows` | tenant, owner, name, trigger, trust_level, status, stats |
| `workflow_versions` | workflow, version, spec (jsonb), author (`user`/`rudra`), compiled_ref |
| `runs` | workflow_version, trigger_source, status, started/ended, cost |
| `step_attempts` | run, step_id, attempt, agent template, model, status, output (jsonb), tokens, trace_id |
| `audits` | step_attempt, passed, failure_class, checks (jsonb), feedback, judge_model |
| `staged_actions` | step_attempt, kind, payload, idempotency_key, status (`staged/committed/discarded/compensated`) |
| `approvals` | run / staged_action, channel, decision, edited_payload, decided_by |
| `memories` | tenant, user, type (`profile/episodic/procedural`), content, embedding `vector`, source, salience |
| `suggestions` | user, pattern_ref, proposed spec, score, status, channel, shown_at |
| `episodes` *(M4)* | user, started/ended, app sequence, actions (jsonb, redacted), embedding |
| `patterns` *(M5)* | user, signature, frequency, est_minutes, confidence, name, description |
| `llm_calls` | tenant, key, provider, model, tier, tokens, latency, cache_hit |
| `events` | id (monotonic), tenant, user, type, payload (jsonb), created_at. SSE replay buffer, short retention |

All tenant tables carry `tenant_id` with Supabase **RLS** policies.

---

## 8. Memory & user model

| Type | What | Written from | Used by |
|---|---|---|---|
| Profile | working hours, focus blocks, tone samples, preferred channels, tool stack, approval habits | onboarding, sent mails/messages, settings, observed behaviour | agents (style), proactive policy |
| Episodic | what happened: runs, approvals, edits, rejections and *why* | run outcomes, approval diffs | planner, writer agents, auditor rubrics |
| Procedural | workflows, their audit history, known failure fixes | workflow store | planner reuse, recovery |

User edits to a staged draft are diffed and stored as episodic feedback ("prefers bullet lists, never says 'Hope you are well'"). These are consolidated nightly (`P3`) into profile facts.

---

## 9. Interfaces

- **Web app:** chat with Rudra, run timeline (steps, attempts, audit verdicts, traces), approval inbox (approve / edit / reject), workflow library and spec viewer, trust levels, connections, API keys, suggestions inbox, privacy controls.
- **Desktop app:** the same UI plus a tray icon, native notifications with approve/reject actions, push-to-talk voice (Windows STT → Rudra → Windows TTS; Gemini audio as fallback), observation controls (pause, excluded apps, live indicator).
- **Slack bot:** DM Rudra, approval cards (Block Kit), daily digest of suggestions and run results, `/rudra` commands.
- **Voice** is an input/output mode over the same chat API, not a separate brain.

### Realtime transport: SSE + POST (no WebSockets)
- **Server → client:** one SSE stream per client session, `GET /events` (`text/event-stream`). It carries run/step status, audit verdicts, approval requests, notifications, suggestions, and commands for the desktop actor. Chat replies stream token-by-token on `POST /chat` itself, as an SSE response.
- **Client → server:** plain REST `POST`: chat messages, approvals, desktop episode batches, actor results.
- **Reconnect & replay:** every event gets a monotonic id and is kept in an `events` table (short retention). On reconnect the client sends `Last-Event-ID` and missed events are replayed. Long-lived streams also send a heartbeat comment every ~20s, and clients reconnect transparently on timeouts (e.g. Cloud Run's request limit).
- **Fan-out across instances:** Postgres `LISTEN/NOTIFY`, so no extra broker is needed. Redis/pub-sub only if we outgrow it.
- **Limits:** one multiplexed stream per client (event `type` field), served over HTTP/2, to avoid the browser's ~6-connections-per-host cap on HTTP/1.1.
- **Realtime audio (future):** the desktop connects directly to the provider's live API using a short-lived token minted by our backend, so audio never streams through our servers.
- **Implementation:** FastAPI + `sse-starlette` on the server; `fetch`-based SSE client in the web and desktop apps (allows an `Authorization` header, which native `EventSource` doesn't).

---

## 10. Desktop observation pipeline *(M4–M5)*

```
UIA events / window focus / browser ext events / clipboard meta
   → local event buffer (SQLite, encrypted)
   → rule-based redactor (regex: emails, phones, tokens, keys, card numbers; password
     fields never captured; excluded apps dropped)
   → sessionizer (groups events into episodes by app/task/idle gaps)
   → episode = ordered action tokens + minimal context (app, page type, entity kinds)
   → upload (batched) → server
server:
   → sequence mining (PrefixSpan-style) over action tokens → candidate patterns
   → LLM names/describes top-K candidates only → patterns table
   → proactive engine scores → suggestions
```

- **No raw screenshots or keystrokes leave the device.** Windows OCR is used only for apps without accessible text, and the result is redacted locally.
- **Opportunity score** = `freq/week × est_minutes × confidence × feasibility(tools available)`.
- **Interruption policy:** no pings in focus blocks. Suggestions are batched into a daily digest. Only high-score items or blocked runs ping immediately. Max N proactive pings/day (user-tunable).

---

## 11. Workflow graduation *(post-MVP)*
After K clean L3 runs, the compiler analyzes step traces. Steps with deterministic behaviour (same tool calls, templated outputs) become plain Python activities, and the LLM stays only for fuzzy steps. The contract and auditor stay in place. A graduated step that fails falls back to its agent version.

---

## 12. Security & privacy
- Tenant isolation: RLS on every table, tenant-scoped Temporal namespaces/task queues.
- Secrets: OAuth tokens and API keys encrypted (envelope encryption; KMS in cloud), never sent to an LLM.
- Prompt injection: content from email and the web is wrapped as untrusted data. Agents get least-privilege tools. The auditor's security layer runs first. All writes are staged.
- Desktop: on-device redaction, pause/exclude controls, visible capture indicator, retention limits, one-click data export/delete.
- Free-tier LLM keys flagged `dev_only`. Production requires BYOK or a paid tier.

---

## 13. Milestones

Durations are rough, for one developer.

### M0 — Foundation (~1 wk)
- [ ] Monorepo scaffold: `backend/` (uv), `apps/web`, `packages/*` (pnpm + Turborepo)
- [ ] Supabase project: Postgres + pgvector + Auth; Alembic migrations for core tables
- [ ] FastAPI skeleton: health, auth dependency (Supabase JWT), tenant context
- [ ] Temporal dev server + worker entrypoint + hello-world workflow
- [ ] Gateway v0: LiteLLM config (Gemini + Groq/Grok), tiers, simple rate limiter, Langfuse tracing
- [ ] `.env.example`, `make`/`just` tasks, CI (lint, type-check, tests)

**Exit:** `POST /chat` returns a Gemini answer through the gateway, and the trace is visible in Langfuse.

### M1 — Rudra Core: orchestrator + auditor (~2–3 wks)
- [ ] Domain models: `WorkflowSpec`, `Step`, `Contract`, `Verdict`, `StagedAction`
- [ ] Agent templates + Agent Factory on ADK with LiteLlm and per-step tool whitelists and budgets
- [ ] Temporal run workflow: DAG execution, step child-workflow (`execute → stage → audit → commit`)
- [ ] Check registry (deterministic checks) + LLM-judge with rubrics + security layer
- [ ] Recovery: failure classification, retry-with-feedback, replan, escalate, halt
- [ ] Planner: intent → spec, validation + self-repair
- [ ] Built-in tools: web search, web fetch, a sandbox "outbox" for staged actions
- [ ] Scheduler v1: priority queues, per-key buckets, 429 fallback

**Exit:** "Research X and write a 1-page brief with ≥5 cited sources" runs end to end. A seeded defect (e.g. 2 sources) is caught by the auditor, fixed on retry, and the whole history is visible in the DB/traces.

### M2 — Interfaces + connectors (~2–3 wks)
- [ ] Web app: chat, run timeline with audit verdicts, approval inbox, workflow library, connections
- [ ] SSE event stream (`/events`, `Last-Event-ID` replay, Postgres LISTEN/NOTIFY fan-out); approvals via POST → Temporal signals
- [ ] OAuth connections + MCP: Gmail, Google Calendar, Slack, GitHub, Jira
- [ ] Triggers: manual, schedule (Temporal schedules), webhooks (GitHub PR opened, Slack mention, Gmail push)
- [ ] Slack bot: DM chat, approval cards, daily digest
- [ ] Trust ladder (L0–L3) with promotion/demotion

**Exit:** two real workflows run on a schedule with approvals from web and Slack:
1. "Weekly sprint status mail"
2. "First-pass PR review comment on new PRs"

### M3 — Memory & user model (~1–2 wks)
- [ ] Memory store + retrieval (pgvector) wired into the Agent Factory
- [ ] Approval-edit diffs → episodic feedback; nightly consolidation → profile
- [ ] Tone learning from the user's sent mail/Slack samples
- [ ] Onboarding flow: connect tools, working hours, channel preferences

**Exit:** after 3 edited drafts, the 4th draft needs no edits for the same stylistic issue.

### M4 — Desktop app (~3–4 wks)
- [ ] Tauri shell reusing `packages/ui` + `app-core`; device login
- [ ] Tray, native notifications with approve/reject actions
- [ ] Push-to-talk voice: Windows STT/TTS, Gemini audio fallback
- [ ] Observers: Windows UI Automation (focus, controls, text), clipboard metadata
- [ ] Browser extension: navigation, form and page-type events; Playwright-compatible actor
- [ ] Local pipeline: buffer, redactor, sessionizer, batched episode upload
- [ ] Privacy controls: pause, excluded apps, capture indicator, data delete

**Exit:** a day of normal work produces clean, redacted episodes in the DB, with zero raw sensitive strings in a redaction test suite.

### M5 — Pattern mining & proactive engine (~2–3 wks)
- [ ] Action tokenization + sequence mining over episodes
- [ ] LLM naming of top candidates; patterns table
- [ ] Opportunity scoring + interruption policy + channel router
- [ ] Suggestion → draft WorkflowSpec (L0 → user accepts → L1)

**Exit:** Rudra independently suggests at least one real repeated task from a week of usage, and its draft workflow runs successfully.

### M6 — SaaS hardening & deploy (~2–3 wks)
- [ ] RLS audit, tenant-scoped Temporal queues, quotas, per-tenant agent caps
- [ ] BYOK UI + encrypted key storage; `dev_only` enforcement
- [ ] Containerize; deploy to Cloud Run/GKE; Temporal Cloud or self-hosted
- [ ] Observability dashboards, alerting, backups; privacy policy and data export/delete

### M7 — Workflow graduation & polish (later)
- [ ] Trace analysis → compiled deterministic steps with agent fallback
- [ ] Mac desktop build; wake word; more connectors (Teams, Outlook, Notion, Confluence)

---

## 15. Local development environment

Dev machine: 7.7 GB RAM (~2 GB free in normal use), Intel i3 2C/4T, Windows 11. The goal is to keep total local footprint under ~1 GB.

| Component | Runs where | Notes |
|---|---|---|
| Postgres + pgvector + Auth + Storage | Supabase cloud, free tier | one project for dev |
| Temporal | native `temporal server start-dev --db-filename .temporal/dev.db` | Web UI at :8233 |
| LLM tracing | Langfuse cloud, free tier | |
| LLMs | Gemini API + Groq API (free keys, `dev_only`) | |
| Backend | `uvicorn` (api) + `python -m rudra.workflows.worker` | |
| Web app | Vite dev server | |
| Docker | off by default; used only to build/test deploy images (M6) | |

**Prerequisites to install:**
- Node **22 LTS** (current Node 18 is end-of-life)
- `uv` (Python package manager)
- `pnpm` (via `corepack enable`)
- Temporal CLI
- Rust toolchain, deferred to M4 (Tauri)

**Accounts / keys needed:** Gemini API key, Groq API key, Supabase project (URL, anon key, service key, DB URL), Langfuse keys.

## 14. Key risks

| Risk | Mitigation |
|---|---|
| Free-tier rate limits throttle agents | Scheduler priorities, fallbacks, caching, batching, BYOK |
| Auditor rubber-stamps (same-model bias) | Evidence-only inputs, deterministic-first, different provider for the judge |
| Retries cause duplicate emails/comments | Stage → audit → commit, idempotency keys |
| Prompt injection via email/web | Least-privilege tools, untrusted-content wrapping, security checks, staged writes |
| Users distrust desktop observation | On-device redaction, transparency, controls; observation is opt-in and comes after the core is useful on its own |
| Proactivity becomes noise | Interruption budget, digests, learning from dismissals |
| Desktop UI automation is brittle | API/MCP first, browser second, UI automation last |
