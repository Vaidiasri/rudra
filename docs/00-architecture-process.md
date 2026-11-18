# How we build Rudra: architecture-first process

Rudra is a production system that will hold users' work data and act on their behalf. Mistakes are expensive: a bad data model, a leaky tenant boundary, an agent that sends the wrong email. So we design before we code, write decisions down, and gate each phase on review.

**Rule:** no production code is written for a component until its LLD is approved. Spikes (throwaway experiments that answer a design question) are allowed, must be labelled as spikes, and are never merged.

---

## 1. Phases and gates

| # | Phase | Question it answers | Deliverables | Gate (exit criteria) |
|---|---|---|---|---|
| 1 | **Research** | What exists, what works, what fails, what's legal? | `02-research/*` | Sources cited; each finding labelled verified/unverified; "implications for Rudra" written |
| 2 | **Product definition** | Who exactly is it for, which problems are worth solving, what is *not* in scope? | `01-product/prd.md`, personas, feature catalogue with admission scores, success metrics | Every feature passes the admission test (§3); non-goals listed; MVP scope frozen |
| 3 | **Requirements** | How well must it work? | `03-requirements/` functional requirements (FR-x), non-functional requirements (NFR-x): latency, availability, scale, cost/user, privacy, security | Every NFR is measurable; traceable to PRD |
| 4 | **High-level design (HLD)** | What are the big pieces and how do they talk? | `04-hld/`: C4 context + container diagrams, data flows, deployment, multi-tenancy model, failure modes; ADRs for each major choice | Each FR/NFR mapped to a component; all major choices have an ADR |
| 5 | **AI system design** | How do the agents, auditor, memory and models work, and how do we know they work? | `05-ai/`: agent architecture, contracts and auditing, model routing, context and prompt management, memory, guardrails and prompt-injection defence, **eval strategy** | Each AI behaviour has an eval plan with pass thresholds |
| 6 | **Data design** | What do we store, how, for how long, and who can see it? | `06-data/`: ERD, table specs, indexes, RLS policies, retention, PII classification, migration strategy, backup and restore | Every table has an owner, tenant scoping, a retention rule and a PII class |
| 7 | **Low-level design (LLD)** | Exactly how does each component work? | `07-lld/<component>.md`: interfaces, state machines, sequence diagrams, error handling, idempotency, config, test plan | Reviewed; open questions = 0 for the MVP path |
| 8 | **API and event contracts** | What do clients and services exchange? | `08-api/`: OpenAPI spec, SSE event catalogue, webhook payloads, versioning policy | Contracts generated/validated in CI before implementation |
| 9 | **Security and privacy** | What can go wrong and how do we prevent it? | `09-security/`: STRIDE threat model, data-flow trust boundaries, secrets management, compliance checklist (DPDP, GDPR) | Every high risk has a mitigation owner |
| 10 | **Engineering standards and ops** | How do we build, test, ship and run it? | `10-engineering/`: coding standards, testing strategy (unit/contract/integration/eval/e2e), CI/CD, environments, observability, SLOs, incident runbooks | CI enforces the standards |
| 11 | **Build plan** | In what order, with what milestones? | `11-roadmap.md`: milestones sliced from the LLDs, each with acceptance tests | Each milestone delivers a usable vertical slice |

Phases 4–9 overlap and inform each other; the gate is that **all** are approved for a component before it is built.

---

## 2. Document conventions

- **Diagrams:** Mermaid in Markdown (renders on GitHub, diffable). C4 levels for architecture; sequence diagrams for flows; state diagrams for lifecycles.
- **Decisions:** every significant or hard-to-reverse choice gets an **ADR** in `adr/` (template: `adr/0000-template.md`). ADRs are immutable once accepted; a change of mind is a new ADR that supersedes the old one.
- **IDs and traceability:** requirements `FR-###` / `NFR-###`, features `F-###`, risks `R-###`. LLDs and tests reference them, so we can always answer "why does this code exist?".
- **Evidence:** research claims cite sources. Design claims about performance or cost cite a measurement or a calculation.
- **Status header** on every doc: `Draft | In review | Approved | Superseded`, owner, last updated.

---

## 3. Feature admission test (no filler features)

A feature enters the MVP only if it scores well on all of these:

1. **Real pain:** it solves a problem a target persona has today, frequently. There is evidence (user interviews, research, our own usage), not just a guess.
2. **Measurable outcome:** we can state how we'll know it works, e.g. "saves ≥30 min/week", "≥70% of drafts approved without edits".
3. **Differentiating or essential:** either it's why someone picks Rudra over alternatives, or the product can't function without it.
4. **Trustworthy at our quality bar:** we can make it reliable enough with today's models, and the auditor can verify it.
5. **Affordable:** fits the cost-per-user budget in the NFRs.

Features that fail go to a **parking lot** with the reason, not into the backlog.

---

## 4. Engineering principles (apply to every design)

1. **Correctness over cleverness.** Explicit state machines, typed contracts, idempotent operations.
2. **Security and privacy by default.** Least privilege, tenant isolation enforced at the database, minimal data collection, encryption, auditability of every action Rudra takes.
3. **Agents are untrusted.** Their output is verified, their tools are scoped, their side effects are staged.
4. **Measure AI like software.** Every prompt or agent change runs against an eval suite in CI; regressions block merges.
5. **Design for failure.** Providers rate-limit, APIs go down, models hallucinate. Every external call has a timeout, a retry policy and a fallback or escalation path.
6. **Cost is a requirement.** Every design states its cost per user per month at target scale.
7. **Observable by default.** Traces, structured logs and metrics are part of the design, not added later.
8. **Simple until proven otherwise.** Prefer one Postgres over three datastores, a modular monolith over microservices, until a measured need says otherwise.

---

## 5. Current status

| Phase | Status |
|---|---|
| 1 Research | **Approved**: three reports + [synthesis](02-research/00-synthesis.md); decisions in [ADRs 0001–0010](adr/) |
| 2 Product definition | **In review**: [PRD](01-product/prd.md) |
| 3–11 | Not started |

The earlier [PLAN.md](PLAN.md) is the **vision draft** that framed this work. It will be superseded by the PRD, HLD and ADRs as they're approved. Code on branch `feat/m0-foundation` is a **spike** and will not be merged as-is.
