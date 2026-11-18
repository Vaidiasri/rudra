# Research synthesis: what the evidence says and the decisions it forces

- **Status:** Approved (2026-11-18). Decisions recorded as [ADR-0001 to ADR-0010](../adr/)
- **Last updated:** 2026-11-18
- **Inputs:** [01 Market & competitors](01-market-and-competitors.md) · [02 AI architecture](02-ai-architecture.md) · [03 Observation & privacy](03-observation-and-privacy.md)

This document turns three research reports into proposed decisions. Approved decisions become ADRs and feed the PRD. Claims marked *unverified* in the source reports stay unverified here.

---

## 1. The five things that matter most

1. **A general "AI that does your work" product is a losing position.** Claude Cowork, Microsoft Copilot Cowork, Google Gemini Spark and ChatGPT's agent features are bundled into subscriptions people already pay for. Several independent assistant startups shut down or pivoted in 2026 (Relay.app, Ollie, Zivy; Rewind sold to Meta) for exactly this reason. [01]
2. **The open gap is proof that the work was done.** No end-user product ships per-workflow success contracts with independent verification against real system state and automatic retry. Agents falsely reporting success is a documented, measured problem, and LLM judges detect it poorly (AUROC ≤ 0.65 in one 2026 study). Verification must therefore check **real external state** (Jira/GitHub/Slack/Gmail APIs), with an LLM judge only as a secondary signal. [01, 02]
3. **Observation is a discovery feature, not the product.** Activity-based automation discovery is already partially occupied (Cofia, Screenpipe, Highlight, enterprise task mining). It's a good entry point and a moat only when paired with verified execution. It is feasible on weak machines only as **event/metadata capture** (no screenshots, no keyboard hooks). [01, 03]
4. **Selling observation to teams is legally loaded.** Employer-deployed activity capture is workplace monitoring: GDPR DPIA, German works councils, New York notice law, and EU AI Act high-risk classification from Dec 2027 for systems that monitor or evaluate workers. A team product must never give managers per-employee activity or productivity views. [03]
5. **"Free LLM APIs" can't serve real users.** Gemini's free tier trains on data and allows human review, and Google's terms restrict EEA/UK/CH users to paid services. Groq's free plan doesn't train, but forbids using it to avoid fees (no pooling of free accounts) and caps throughput. Free tiers are for development with synthetic data. Production needs BYOK or paid usage, so **unit economics must be designed in, not assumed away.** [02]

---

## 2. Proposed product decisions

| ID | Decision | Evidence | Replaces |
|---|---|---|---|
| **PD-1** | **Positioning:** "Automation for engineering teams that proves it worked." The core value is *verified* automation: success contracts, state-based verification, retry and earned autonomy. | Gap in [01] §5; judge limits in [02] §3 | "Proactive AI assistant + agents" (too broad) |
| **PD-2** | **MVP focuses on 5–6 workflows that are verifiable through APIs:** sprint/status reports, ticket hygiene, PR-review *digests* (not our own PR reviewer), Slack/email triage digests, research summaries, meeting-to-ticket follow-ups. Final list set in the PRD via the admission test. | [01] §8; CodeRabbit etc. own PR review | Open-ended "automate anything" |
| **PD-3** | **MVP surfaces: web app + Slack + API connectors.** Discovery in the MVP comes from **connected-tool data** (Jira/GitHub/Slack/Calendar/Gmail activity via APIs), which is cheap, needs no installer and carries less legal risk. The **Windows desktop observer ships in the next phase**, once verified automation has proven its value. | Code-signing cost and EDR risk [03] §7; discovery partly commoditized [01] | Desktop observation in v1 |
| **PD-4** | **Observation rules (when the desktop app ships):** events and metadata only. Foreground window/title, idle, focused-element UIA, browser extension on an **allow-list of work sites**. Never screenshots (except explicit demo recording), never low-level keyboard hooks, user-owned data, opt-in, pausable, deletable. | [03] §1–5, Recall redesign | "Full desktop + browser activity" |
| **PD-5** | **Team product principle:** employees own their observation data. Managers see shared workflow *outputs* only, never individual activity, scores or comparisons. No feature that evaluates worker performance. | [03] §6, EU AI Act | — |
| **PD-6** | **Voice is out of v1.** Interaction is web + Slack + notifications. | [01] §8 (low value, high cost) | Voice in v1 |
| **PD-7** | **Pricing principle:** users never pay for failed or unverified runs. Free tier = discovery + drafts. Price points (≈ $24–30 Pro, $30–40/seat Team, per [01]) to be validated in the PRD. | Credit-burn complaints [01] §2 | — |

## 3. Proposed technical decisions (become ADRs)

| ID | Decision | Evidence | Replaces |
|---|---|---|---|
| **TD-1** | **Custom orchestrator** (planner, DAG, auditor, staging, trust ladder) with **Pydantic AI as the agent runtime**. Google ADK stays as a documented fallback. | ADK 2.x breaking-change churn, Gemini/GCP-first; Pydantic AI native multi-provider clients, typed output, MCP, OTel [02] §1 | Google ADK |
| **TD-2** | **No LiteLLM on the critical path.** Use Pydantic AI's native provider clients behind our own gateway (routing, rate limits, priorities, fallbacks, metering). Pin and hash-lock all dependencies; supply-chain scanning in CI. | LiteLLM PyPI compromise 2026-03-24 [02] | LiteLLM |
| **TD-3** | **Durable execution behind an interface. Temporal vs DBOS to be decided by ADR after a spike.** DBOS runs on our existing Postgres (one less system to operate); Temporal is more mature with GA AI integrations. Either way: claim-check large payloads, bounded histories. | [02] §2 | Temporal (assumed) |
| **TD-4** | **Verification hierarchy:** (1) security/provenance checks, (2) deterministic checks against **real external state**, (3) LLM judge from a **different model family**, as a secondary signal only, (4) sampled human review to calibrate. Trust-ladder promotion uses measured false-success and override rates. | [02] §3 | Judge-centric auditing |
| **TD-5** | **Prompt-injection architecture:** plan is fixed before untrusted content is read; agents that read untrusted content get **no side-effect tools**; a code-level **provenance gate** checks where outbound tool arguments came from (CaMeL-lite); Rule of Two enforced at agent creation; all writes staged → audited → committed. | [02] §4 | Generic "scoped tools" |
| **TD-6** | **Data and observability:** Postgres + pgvector (no graph DB until measured need); Langfuse Cloud free plan via OpenTelemetry, so the backend is swappable. | [02] §5–6 | Unchanged (confirmed) |
| **TD-7** | **LLM usage policy:** free Gemini = dev only, synthetic data only. Groq free = allowed within ToS, no account pooling. Production = tenant BYOK or paid platform keys. Free-tier keys refused for EEA/UK/CH tenants and for workflows touching email/PII. | [02] §7 | "Free APIs in production" |
| **TD-8** | **Realtime:** SSE + REST (already decided). | — | — |

## 4. Risks surfaced

| ID | Risk | Mitigation direction |
|---|---|---|
| R-1 | Platform giants ship verified automation | Depth in engineering workflows + state-based verification across tools they don't own end-to-end; move fast on the niche |
| R-2 | LLM costs make unit economics negative | BYOK, cheap-model cascades, deterministic-first verification, workflow graduation to code; cost per user as an NFR |
| R-3 | Corporate EDR quarantines the desktop agent | No keyboard hooks, signed binaries, EDR lab testing before team sales |
| R-4 | Chrome Web Store rejects the extension | Single purpose, allow-listed sites, in-extension disclosure |
| R-5 | Compliance gaps (DPDP, GDPR, EU AI Act) | Privacy by design now; lawyer review before team launch; no worker evaluation features |
| R-6 | Supply-chain compromise of AI dependencies | Hash-pinned lockfiles, dependency scanning, isolate secrets from LLM libraries |
| R-7 | Free-tier limits make dev/testing painful | Recorded fixtures for tests, Gemini Flash-Lite and Groq for dev, small eval sets |

## 5. Open questions (to resolve in PRD / HLD)

1. Individuals first or teams first? This affects onboarding, pricing, auth and compliance order.
2. Temporal vs DBOS (ADR after spike).
3. Who pays for LLM usage at launch: BYOK only, or a paid platform pool with usage caps?
4. Final DPDP timeline and Chrome Web Store policy text (re-check before the desktop phase; lawyer for DPDP).
5. EDR and UIA cost measurements on real machines (desktop phase).
6. Price points: validate with target users before launch.
