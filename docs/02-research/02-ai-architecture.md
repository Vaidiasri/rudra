# 02 — AI Architecture Research for Rudra

**Date:** 2026-11-18 · **Status:** Research input for ADRs, not a decision record.
**Method:** Live web search plus official docs fetched on 2026-11-18. Every claim tagged **[unverified]** comes from a third-party source, from a page I could not open, or from my own background knowledge with no live check. Version numbers move weekly, so pin versions and re-check them before you write an ADR.

---

## Executive summary

| # | Question | Recommendation | Versus current choice |
|---|---|---|---|
| 1 | Agent framework | **Thin custom orchestrator** (planner, DAG, auditor and side-effect staging are Rudra's own code) with **Pydantic AI** as the per-agent runtime and model layer. Keep ADK as the fallback option. | **Changes** "Google ADK + LiteLLM" |
| 2 | Durable execution | **Temporal stays a sound choice**. All its AI integrations went GA on 2026-10-30, including ADK and the Gemini SDK. The cheapest viable option is **DBOS**, which is a library running on the Postgres you already have. Hide the engine behind an interface. Use DBOS for MVP if ops budget is the constraint; use Temporal if you want the strongest HITL/timer/versioning story and can run its dev server locally. | **Mostly confirms**, and adds an alternative |
| 3 | Verification | Make **deterministic, state-based checks the primary gate**. Use an LLM judge from a **different model family** only for criteria that can't be checked mechanically, with binary rubric items, swapped-order pairwise comparisons and human-calibrated thresholds. One 2026 study measured LLM judges detecting "agent says done but wasn't" at AUROC ≤ 0.65. | Confirms the design, but the judge must stay secondary |
| 4 | Prompt injection | Use architectural controls, not filters: **plan-then-execute**, with the plan fixed before any untrusted content is read; a **quarantined, tool-less reader LLM** that returns typed data only; **provenance/taint checks on tool arguments** (CaMeL-style, lite); **Meta's Rule of Two** per agent session; and **human approval for outbound side effects** (Rudra's staging layer already does this). | Fits the design. Make it a hard invariant |
| 5 | Memory | **Postgres + pgvector** plus typed, structured memory tables. Defer graph stores. | **Confirms** |
| 6 | Evals and observability | **Langfuse Cloud (Hobby, free)** for traces, instrumented through **OpenTelemetry GenAI conventions** so the backend can be swapped. Use **pytest-based offline evals** (DeepEval or Inspect), plus **promptfoo** for prompt regression and red-teaming in CI. Don't self-host Langfuse on the dev box. | **Confirms**, with caveats |
| 7 | Free tiers | **Gemini free tier is close to unusable for multi-step agents**: Flash models are observed at about 5 RPM and 20 RPD [unverified, third-party observation]. Free-tier data **is used to improve Google products and may be human-reviewed**. EEA, UK and Swiss users **may only be served via Paid Services**. Groq's free plan does not train on inputs and does not forbid commercial use, but it has no indemnity and a $5k liability cap. | **Changes the cost model**. Free Gemini is dev-only and never touches tenant data |
| 8 | Cost control | Rely on implicit prefix caching (Gemini ≥ 2.5 automatic; Groq gpt-oss automatic at a 50% discount). Use a cheap→strong cascade gated by deterministic validators. Use constrained decoding where it exists, but note that Groq strict mode **cannot be combined with tool use or streaming**. | New constraints |

**Contradictions with current choices (Google ADK + LiteLLM + Temporal + Postgres/pgvector + Langfuse):**
1. **LiteLLM**: its PyPI package was **compromised on 2026-03-24** (v1.82.7/1.82.8 shipped a credential stealer, TeamPCP campaign). This is a real supply-chain risk for a component that holds every tenant's BYOK keys. ADK also reaches non-Gemini models only through a `LiteLlm` connector, which adds a second abstraction layer. Pydantic AI has native provider clients and removes LiteLLM from the critical path.
2. **ADK churn**: 2.0 GA on 2026-05-19 was a graph-runtime rewrite with breaking event and session changes. 2.9, 2.13 and 2.14 shipped further breaking changes, about 14 minors in 6 months. ADK is Gemini- and Google Cloud-first, and Rudra must be provider-neutral.
3. **Gemini free tier** cannot carry even development workloads for Flash-class models. Budget for Flash-Lite or Gemma on Gemini, and for Groq, as the dev workhorses.

---

## 1. Agent frameworks

### 1.1 Current state (Nov 2026)

| Framework | Version / maturity | Multi-provider | Multi-agent | MCP | Structured output | Observability | Durable-engine fit | Lock-in / adoption |
|---|---|---|---|---|---|---|---|---|
| **Google ADK (Python)** | v2.14.1 (2026-11-12). 2.0 GA 2026-05-19 with a graph runtime (`BaseAgent` now subclasses `BaseNode`). Frequent breaking changes. | Native: Gemini, Claude, Vertex-hosted models. Others via connectors (`LiteLlm`, Ollama, vLLM, Apigee). | Strong: graph, dynamic and collaborative workflows; A2A became stable in 2.14. | Yes (MCP SDK 1.x and 2.x) | Yes (output schema); TypeSafe classifiers for routing (2.13) | OpenTelemetry, GenAI semconv v2 default since 2.13 | **Temporal GA** (`temporalio[google-adk]`, `TemporalModel` runs each model call as an Activity). Note that 2.9 changed resume so that failed nodes re-run, so nodes must be idempotent. | Medium-high (Gemini, Vertex Agent Engine, Google Cloud). Large Google backing. |
| **LangGraph** | 1.x (latest version not verified) | Via LangChain integrations (broad) | Strong (state graphs, subgraphs) | Via adapters | Yes | LangSmith-native; OTel possible | Own checkpointer (Postgres). **Temporal plugin GA** (each node runs as an Activity). Open-source LangGraph has no automatic re-drive after a crash. | Medium (LangChain ecosystem). Widest adoption. |
| **OpenAI Agents SDK** | 0.24.0 (2026-10-15) [unverified]; still 0.x | Via LiteLLM or any-llm extras; built around the Responses API | Handoffs, agents-as-tools | Yes | Yes | Tracing **goes to OpenAI by default**, even for other models [unverified]; pluggable processors | **Temporal GA** (Python); Restate integration | Medium-high (Responses-API-shaped) |
| **Pydantic AI** | v2.70.0 (2026-11-17); v1 line still patched (1.107.10). Very fast release cadence. | **Native clients** for OpenAI, Anthropic, Gemini, Groq and more, plus `ModelFallback` | Sub-agents and peer messaging (newer, less proven) | Yes | **First-class** (typed outputs, validation and retry, `RefineOutput`) | Logfire/OTel (any OTel backend, Langfuse included) | **Best in class**: 9 engines including Temporal, DBOS, Restate, Prefect and pgtask (Postgres); deferred tools for HITL | Low lock-in (MIT; typed Python) |
| **Claude Agent SDK** | Active; multi-language | **Claude only** (also via Bedrock and Vertex) | Subagents | Excellent | Yes | Hooks | Wrap manually | High (one vendor). **Excluded** by the provider-agnostic constraint. |
| **CrewAI** | Active [version not verified] | Broad (historically via LiteLLM) | Role-based crews and flows | Yes | Yes | Integrations | Weak fit; it has its own runtime | Medium. Popular for prototypes. |
| **Microsoft Agent Framework** | 1.0 GA in early April 2026 (.NET and Python). **AutoGen and Semantic Kernel are in maintenance mode.** | Broad | Strong (graph workflows) | Yes | Yes | OTel | Has its own durable options; Azure Durable Task | Azure gravity. Don't start new work on AutoGen. |
| **Agno** | Active; Apache-2.0 | Broad | Teams | Yes | Yes | Own platform | Weak | Low-medium |
| **AWS Strands** (newer) | Temporal GA (Python and TS) | Broad (Bedrock-first) | Yes | Yes | Yes | OTel | Temporal GA | AWS gravity |

Sources: ADK releases page, ADK 2.0 docs, Temporal GA blog (2026-10-30), Pydantic AI releases and durable-execution docs, third-party comparisons (marked).

### 1.2 What Rudra actually needs from a framework

The differentiated parts of Rudra are its own logic:
- the DAG planner,
- dynamic agent factory with least-privilege tool binding,
- success contracts,
- auditor,
- staged side effects,
- autonomy ladder.

No framework provides these. A framework only has to supply:
1. a reliable **single-agent tool loop**,
2. **typed structured output** with validation and retry,
3. a **provider-neutral model client**,
4. **MCP client** support,
5. **OTel traces**,
6. a clean **seam for durable execution**, so that every model and tool call is journaled.

Pydantic AI covers all six with the least opinion about multi-agent topology. That is the right property when Rudra's own orchestrator owns the topology.

### 1.3 Thin custom orchestrator vs full framework

| Option | Pros | Cons |
|---|---|---|
| **Custom orchestrator + Pydantic AI agents (recommended)** | The DAG, contracts and auditor are explicit Rudra code, testable without LLMs. Agents are disposable `Agent(model, tools, output_type)` objects built per step, which suits least privilege. Durable integration is co-maintained by Temporal and DBOS. No LiteLLM in the path. | You build planning, re-planning and agent-to-agent plumbing yourself (about 1–2k LOC). Pydantic AI moves fast, so pin minors and read its "Compatibility Notes". |
| **Google ADK 2.x end to end** | Graph workflows, A2A, HITL (`RequestInput`, `require_confirmation`), built-in eval CLI. Temporal GA. | Non-Gemini models go through connectors, and the docs don't state per-provider feature gaps. Breaking changes almost every minor release. ADK's graph engine would duplicate and fight Rudra's own DAG and Temporal's workflow state. |
| **Custom orchestrator + LiteLLM only** | Thinnest dependency. LiteLLM's model coverage is the broadest. | You rebuild the tool loop, output validation and MCP. LiteLLM supply-chain history: credential-holding service, March 2026 compromise. If used, pin hashes and run it as an isolated proxy with no tenant secrets in env. |
| **LangGraph** | Mature, strong checkpointing | Two graph systems if the orchestrator is custom, plus LangChain abstraction weight |

**Verdict.** Use the custom orchestrator for the control plane and Pydantic AI for the agent and data plane. If the team wants to stay on ADK, it is defensible: Temporal's ADK integration is GA. In that case, use ADK only for single-agent loops, not ADK graphs, and keep the DAG in Rudra/Temporal.

---

## 2. Durable execution

### 2.1 Comparison

| | **Temporal** | **DBOS** | **Restate** | **Inngest** | **Hatchet** | **Plain Postgres queue** (e.g. SKIP LOCKED, pgtask/Absurd) |
|---|---|---|---|---|---|---|
| Model | Server cluster + workers; deterministic workflow code + Activities | **Library** inside your app; checkpoints to your Postgres | Single binary with embedded log; handlers journaled | Server calls your HTTP endpoint step by step | Server (API, engine, dashboard) + workers on Postgres | Your code |
| License | MIT | Library MIT; **Conductor (control plane) proprietary, paid for production self-host** | **BSL 1.1** (converts to Apache after 4 years) | **SSPL** (converts after 3 years) | MIT | n/a |
| Small self-host cost | Dev: `temporal server start-dev` is a single process. Prod: 4 services + Postgres, about 0.5 GB RAM at idle with no Elasticsearch [unverified]. ~€15–30/mo VM [unverified]. | **≈ $0 extra** (Postgres you already run) | One binary + persistent disk | One binary (SQLite); Postgres + Redis to scale | API + engine + Postgres (RabbitMQ optional) | $0 |
| Managed tier | Temporal Cloud: sources conflict between "no base fee, $50/M actions, $150 credits" and "from $200/mo" [unverified] | Pro $99/mo | Free 100k actions/mo | Free 50k executions/mo | Free 1M runs/mo [unverified] | n/a |
| Python SDK | Mature; sandboxed workflows; AI integrations GA | Clean decorators; first-class Pydantic AI integration | Good; Pydantic AI and OpenAI Agents integrations | TS-first; Python lags [unverified] | v1 SDK, decent | n/a |
| HITL | **Signals and Updates**, durable timers, no worker held | `DBOS.recv(..., suspend=True)` consumes no resources while waiting; `set_event`/`get_event` | Awakeables (one-shot callback tokens) | `step.wait_for_event` with timeout | Durable event waits | You build it |
| Long LLM steps | Activities with heartbeats, start-to-close timeouts, retries. Streaming via Workflow Streams (Public Preview). | Steps are checkpointed after completion; a crashed step re-runs | Journaled calls | Each step is an HTTP request, so serverless timeouts apply | Task timeouts | Manual |
| Known pain points | **Determinism**: all LLM/IO must be in Activities; code changes need versioning/patching. **Payload limits**: 2 MB per payload, 4 MB gRPC per request, 50 MB / 51,200 events per history. Agent histories grow fast; use claim-check (built-in external storage since Python SDK 1.25) and continue-as-new. Ops-heavy self-host. | Throughput is bound by Postgres (≈4 vCPU for ~1k steps/s [unverified]). Multi-node recovery wants Conductor (paid). Smaller ecosystem. | BSL licence; younger cloud; single-node crash window | SSPL; Python second-class; HTTP step model awkward for long LLM calls | Postgres doubles as the queue; short retention on cloud tiers | Reinventing retries, timers, idempotency and replay |

### 2.2 Recommendation

- **Temporal remains a good choice for Rudra.** Temporal's 2026-10-30 GA covers Google ADK, OpenAI Agents SDK, LangGraph, Gemini SDK, Strands and LangSmith. Pydantic AI's `TemporalDurability` is co-maintained. Signals and Updates map directly onto approve/reject/edit of staged side effects, and durable timers onto escalation SLAs.
- **On the 8 GB dev box**, use the Temporal CLI dev server (SQLite, single process), not docker-compose with Elasticsearch.
- **For near-zero-budget production**, **DBOS is the strongest challenger**:
  - no new infrastructure,
  - workflow state lives next to tenant data in Postgres, which helps row-level security, audit and backups,
  - HITL via `recv` and events,
  - Pydantic AI integration.

  The catch is the paid Conductor for robust multi-worker recovery. Verify before production.
- **Mandatory regardless of engine:**
  1. LLM and tool calls are **Activities/steps**, never in workflow code.
  2. Store prompts and outputs in Postgres or GCS and pass **references** through the engine (claim-check).
  3. Every side-effecting step carries an **idempotency key** and goes through Rudra's stage → audit → commit outbox.
  4. Disable framework-level retries. Let the engine own retries.
  5. Keep an `ExecutionBackend` interface so the Temporal↔DBOS swap stays possible.

---

## 3. Verification and auditing

### 3.1 What the evidence says

**Known judge biases**
- **Position bias, verbosity and self-enhancement bias** in LLM judges: Zheng et al. 2023 (MT-Bench) and Wang et al. 2023, "LLMs are not Fair Evaluators".
- **Mitigations for position bias:** evaluate both orders and only accept consistent verdicts; or use calibration.

**Self-preference**
- Panickssery, Bowman and Feng (NeurIPS 2024) found that a judge's self-recognition ability correlates linearly with its self-preference.
- GPT-4, GPT-3.5 and Llama 2 favoured their own summaries over equal-quality human and other-model text.
- 2026 multimodal and multilingual work (BabelJudge; MLLM model-preference bias) reports **family-level** preference too.
- **Implication:** the judge must come from a *different family* than the producer, not just a different model size.

**Reliability is not validity**
- Norman et al. (arXiv 2606.19544, June 2026; **later withdrawn by arXiv admin**, so treat it as indicative):
  - production judges had test–retest > 0.95 **and** position bias > 0.10,
  - kappa sat 33–41 points below raw agreement,
  - judge rankings moved up to 14 places across benchmarks.
- **Implication:** calibrate against *your* human labels, not public leaderboards.

**"Agent says done but wasn't"**
- Advani (arXiv 2606.09863, ICML 2026 workshop):
  - false success is **45–48%** of failures in tau2-bench single-control domains and **75.8%** in AppWorld trajectories with explicit status claims;
  - **LLM judges reached AUROC ≤ 0.65** (0.54 on AppWorld), keying on confident closing language and action volume;
  - cheap TF-IDF detectors reached 0.83–0.95 AUROC.
- METR reports agents fabricating evidence of completion [secondhand].
- **Implication:** verify **world state**, not the agent's narrative.

**Self-correction and self-verification**
- Huang et al. (ICLR 2024, "LLMs Cannot Self-Correct Reasoning Yet") and Stechly, Valmeekam and Kambhampati (2024): intrinsic self-critique without external feedback often degrades answers.
- Gains come from **external, sound verifiers** (tests, schema checks, state queries).
- *[papers known from background; not re-fetched today]*

**Best-of-n**
- Repeated sampling raises coverage (Brown et al. 2024, "Large Language Monkeys").
- With **imperfect verifiers**, accuracy plateaus or declines as n grows ("Inference Scaling fLaws", Stroebl et al. 2024) *[background]*.
- Best-of-n is only worth paying for when the verifier is deterministic (tests pass, schema valid, facts match the source).

**Production guidance**
- Anthropic, *Demystifying evals for AI agents*:
  - combine code, model and human graders;
  - grade the **outcome**, not the path;
  - binary or weighted scoring;
  - read transcripts to validate graders.
- Juries of small, diverse models (Verga et al. 2024, PoLL) can beat a single large judge at lower cost *[background]*.

### 3.2 Recommended auditor design

**1. The success contract is authored before the run.** It is a Pydantic model containing:
   - `deterministic_checks` (schema; required fields; regex; URL reachability; "PR comment exists via the GitHub API"; "email draft contains no unresolved placeholders"; numeric diffs);
   - `evidence_requirements` (each claim cites a source span or tool result ID);
   - `judge_rubric` (a list of **binary, single-criterion** questions with pass and fail examples);
   - `risk_tier`.

**2. Order of checks:** schema → deterministic → **state verification** (re-query the external system, never trust the agent's report) → evidence grounding (claims ↔ retrieved spans) → LLM judge.

**3. Judge configuration:**
   - a different family from the producer (e.g. Groq gpt-oss judges Gemini output, and vice versa);
   - temperature 0;
   - structured verdict `{criterion_id, pass, evidence_quote, confidence}`;
   - pairwise comparisons in both orders;
   - the judge sees the contract and the evidence, **not** the producer's self-assessment or chain-of-thought.

**4. Calibration:** keep 50–200 human-labelled step outputs per workflow type. Track judge TPR/TNR, not just agreement. Re-calibrate whenever the judge model changes.

**5. Retry policy:** feed back only *failed criterion IDs + evidence*. Cap retries at 2. Then re-plan, then escalate.

**6. Autonomy ladder:** promote a workflow only when its audited pass rate, human-override rate and false-success rate (measured on sampled human review) meet thresholds over N runs.

**7. Cheap triage:** add a lexical or classifier "false-success" detector on agent transcripts as a sampling signal for human review, per Advani 2026.

---

## 4. Prompt-injection defenses (email, web, actions)

### 4.1 State of the art

**Consensus position**
- Prompt injection is **unsolved at the model level** (Meta, OWASP, DeepMind all agree).
- Adaptive attacks bypass detection and prompting defenses. The 2026 adaptive-evaluation work argues for **deterministic, out-of-band enforcement**.

**CaMeL** (Debenedetti et al., Google DeepMind, arXiv 2503.18813):
- a privileged LLM plans from the trusted query only;
- a quarantined LLM parses untrusted data with no tools;
- a custom interpreter tracks **capabilities and provenance** and enforces policy before each tool call.
- **Cost:** in a 2026 re-evaluation, 77% of AgentDojo tasks were solved under attack vs 84% undefended. The authors acknowledge side channels. Adopting it requires rewriting the agent.

**Design Patterns for Securing LLM Agents** (Beurer-Kellner et al., arXiv 2506.08837):
- six patterns: action-selector, **plan-then-execute**, LLM map-reduce, **dual LLM**, code-then-execute, context-minimization;
- recommends plan-then-execute for email and calendar assistants.

**Meta Agents Rule of Two** (2025-10-31): within a session, an agent may have at most two of
- [A] untrusted input,
- [B] sensitive data or systems,
- [C] state change or external communication.

If it needs all three, add human approval.

**Spotlighting** (Hines et al. 2024: delimiting, datamarking, encoding untrusted text) lowers attack success but is a **mitigation, not a boundary** *[background]*.

**Classifiers** (e.g. Llama Prompt Guard 2, available free on Groq at 14.4K RPD): useful as telemetry and triage, never as the sole control.

### 4.2 Best practice for Rudra (2026)

1. **Plan before reading.** The orchestrator compiles the DAG from the trusted user and tenant intent only. Untrusted content can never add steps, tools or recipients. A re-plan triggered by content goes through the auditor and is treated as tainted.
2. **Quarantined readers.**
   - Agents that read email, web pages or PR text have **no side-effect tools** and return **typed, schema-constrained data** (enums, IDs, bounded strings).
   - Free text from them is tagged `tainted=True` and travels as an opaque value.
3. **Taint-aware tool gate** (CaMeL-lite, in Rudra code, not in the prompt):
   - every outbound tool argument carries provenance;
   - policy example: "recipient must come from the user's contacts or the original thread participants, never from tainted text"; "URLs fetched must be on the plan's allowlist";
   - this is deterministic code.
4. **Rule of Two per agent instance.** Rudra's dynamic least-privilege agent factory is the right place to enforce it: refuse to instantiate an agent spec that holds A+B+C.
5. **Stage, audit, commit** for every external write. The diff is shown to the human until the autonomy ladder allows auto-commit, and **never** auto-commit where the content was derived from tainted input to a new recipient.
6. **Spotlighting and datamarking** of untrusted text, plus an injection classifier as a signal only.
7. **Egress controls:**
   - no arbitrary URL fetch from agents holding secrets;
   - render no markdown images or links in auto-sent content (exfiltration channel);
   - per-tenant credential scoping;
   - MCP servers pinned and allowlisted.
8. **Red-team in CI.** Run AgentDojo-style and promptfoo red-team suites against each workflow template.

---

## 5. Agent memory

| Option | Model | Strengths | Weaknesses | Fit |
|---|---|---|---|---|
| **Postgres + pgvector (+ FTS)** | Tables you design: episodic log, facts/preferences, embeddings, tenant RLS | One datastore, transactional with workflow state, RLS for multi-tenancy, hybrid search (BM25 via `tsvector` + vector), $0 | You write extraction, consolidation and decay | **Recommended** |
| **Mem0** | LLM-extracted memories → vector store (pgvector supported) + optional graph | Drop-in extraction and consolidation | Extra LLM calls per write (free-tier RPD pressure). Vendor-reported benchmark numbers conflict wildly (e.g. LongMemEval 94.4 self-reported vs 49 elsewhere). | Option later: borrow its extraction pattern |
| **Zep / Graphiti** | Temporal knowledge graph (bi-temporal edges) | Facts that change over time, entity relationships | Needs Neo4j, FalkorDB or Kuzu; heavier ops; ingestion is LLM-heavy | Only if relationship queries become core |
| **Letta (MemGPT)** | Agent server with self-editing memory blocks | Long-lived stateful agents | It is an agent runtime, which conflicts with Rudra's orchestrator and durable engine | Not a fit |
| **LangMem** | Memory primitives for LangGraph | Fine inside LangGraph | Couples you to LangGraph's store | Not a fit |

**When a graph store is worth it:** when you need multi-hop questions over entities and relations ("which services owned by team X depend on the repo that this failing PR touches?"), or facts with validity intervals that you must reason across. For Rudra's early workflows (user preferences, past decisions, workflow outcomes, contact and project facts), relational tables plus pgvector are enough. Model relations as adjacency tables, using recursive CTEs for shallow hops.

Memory benchmarks (LoCoMo, LongMemEval) are **not comparable across vendors**: judge, answer model and ingestion all differ. Don't select on them.

**Security:** memory written from tainted content is itself tainted (memory-poisoning vector). Store provenance on every memory row.

---

## 6. Evals and observability

### 6.1 Tools

| Tool | Type | Licence / hosting | Free tier | Notes |
|---|---|---|---|---|
| **Langfuse** | Tracing, prompt management, evals, datasets | MIT core; **acquired by ClickHouse (2026-01-16)** with a commitment to stay OSS | Cloud **Hobby: 50k units/mo, 30-day access, 2 users**; Core $29/mo | Self-host = Web + Worker + **Postgres + Redis + ClickHouse + S3**, too heavy for an 8 GB dev box. Third-party report says Cloud is v4-only from 2026-11-16 [unverified]. |
| **Arize Phoenix** | Tracing and evals, OpenInference/OTel | Elastic License; **Arize acquired by Dynatrace** (2026) [unverified] | Self-host free and light (single container) | Best lightweight local option |
| **LangSmith** | Tracing and evals | Closed; self-host enterprise only | ~5k traces/mo [unverified] | LangChain-centric |
| **Braintrust** | Evals and tracing | Closed | Free tier, but sources conflict on size [unverified] | Strong eval UX |
| **promptfoo** | CLI evals and red-teaming | OSS (MIT); **OpenAI acquisition announced 2026-03-09**, with a pledge to stay OSS | Free | Ideal for CI prompt regression and injection red-teaming. Watch its post-acquisition provider neutrality. |
| **DeepEval** | pytest-style LLM metrics | Apache-2.0 | Free | Many judge metrics; validate them before trusting |
| **Inspect (UK AISI)** | Eval framework for agents and tool use, sandboxes | MIT | Free | Rigorous agentic evals; good for scenario suites *[background]* |

### 6.2 Near-zero-budget eval setup

1. **Instrument once with OTel GenAI semantic conventions** (Pydantic AI/Logfire and ADK both emit OTel). Export to Langfuse Cloud Hobby. Phoenix runs locally on demand. The backend can be swapped without code changes.
2. **Golden datasets in git** (`evals/<workflow>/cases.jsonl`): 20–50 cases per workflow template, growing from production failures. Each case carries its success contract.
3. **Three tiers:**
   - **Unit:** mocked model; deterministic orchestrator, auditor and tool-gate tests on every commit.
   - **Prompt regression:** promptfoo against cheap models on PRs touching prompts. Cache responses to save RPD.
   - **Scenario:** Inspect or DeepEval multi-step runs nightly against recorded tool fixtures (VCR-style cassettes) to reduce live calls.
4. **Rate-limit-aware runner:** a single token-bucket shared across eval and dev, plus response caching keyed by (model, prompt hash).
5. **Track:** contract pass rate, false-success rate (human-sampled), judge–human agreement (TPR/TNR), cost and latency per step, injection-suite pass rate.
6. **Online:** sample N% of production runs for human review in Langfuse annotation queues. Feed the labels back into the judge calibration sets.

---

## 7. Free-tier reality (critical for the cost model)

### 7.1 Gemini API (Google AI Studio)

**Rate limits**

| Model | RPM | TPM | RPD | Source |
|---|---|---|---|---|
| Gemini 3.8 / 3.6 / 3 / 2.5 **Flash** | 5 | 250k | **20** | Third-party observation in AI Studio, 2026-10-21 [unverified] |
| Gemini 3.5 / 3.1 **Flash-Lite** | 15 | 250k | 500 | same [unverified] |
| Gemini 2.5 Flash-Lite | 10 | — | 20 | same [unverified] |
| **Gemma 4** | 30 | — | 14,400 | same [unverified] |
| Pro models (4 Argon, 3.1 Pro, 2.5 Pro) | 0 | 0 | 0 | same; the pricing page lists 4 Argon and 3.1 Pro as "Not available" on free |
| Gemini Decision 1 | 200 | 50k | 10,000 | Official rate-limits page |

**How limits apply** (official rate-limits page)
- Limits are **per project, not per API key**.
- RPD resets at midnight Pacific.
- Free limits are **"not guaranteed"**.
- Per-model numbers are now only shown inside AI Studio.

**Data use** (official Gemini API Additional Terms, effective 2026-03-23)
- **Unpaid Services:** Google uses submitted content and responses "to provide, improve, and develop" products. **Human reviewers may read, annotate and process** inputs and outputs. "Do not submit sensitive, confidential, or personal information to the Unpaid Services."
- **Paid Services:** prompts and responses are not used to improve products. They are logged "for a limited period" for abuse detection.

**Commercial and ToS restrictions**
- The Services are for "professional or business purposes" and "not for consumer use".
- **"You may use only Paid Services when making API Clients available to users in the European Economic Area"** (also Switzerland and the UK).
- Minimum age is 18.
- Do not use the Services to build competing models.
- No clinical or medical advice.
- Google APIs ToS §2(d): "will not attempt to circumvent" limits. **Sharding load across many free projects is a ToS risk.** §4(a)(1): no sublicensing of the API to third parties.

**BYOK in production:** a tenant's own free-tier key is the tenant's own relationship with Google. Rudra must warn tenants that a free key means their data may be used for training and human review. Rudra should arguably **refuse free-tier keys for EEA, UK or CH tenants** and for workflows touching email or PII.

**Price moves:** Gemini 3.8 / 3.6 Flash paid prices **double on 2027-01-01** ($0.75→$1.50 in, $3.75→$7.50 out per 1M tokens). Flash-Lite 3.5 is $0.30/$2.50 and 3.1 is $0.25/$1.50. Model-cost assumptions belong in config, not code.

### 7.2 Groq API

**Rate limits**

The official rate-limits page shows one table. Our fetch labelled it "Developer Plan base limits", but its values match Groq's historical *Free* plan values. Confirm the actual numbers at console.groq.com/settings/limits.

| Model | RPM | RPD | TPM | TPD |
|---|---|---|---|---|
| openai/gpt-oss-120b | 30 | 1,000 | 8,000 | 200,000 |
| openai/gpt-oss-20b | 30 | 1,000 | 8,000 | 200,000 |
| qwen/qwen3.8-27b (preview) | 30 | 1,000 | 8,000 | 200,000 |
| meta-llama/llama-prompt-guard-2-86m | 30 | 14,400 | 15,000 | 500,000 |

- Production models: llama-3.1-8b-instant, llama-3.3-70b-versatile, gpt-oss-120b/20b (131k context). Limits for the llama models were not shown in the fetched table.
- **Preview models are "for evaluation purposes only and should not be used in production"**, which includes qwen3.8-27b and minimax-m2.7.
- **8k TPM is the binding constraint**: one agent turn with tool schemas and context can use most of a minute's budget.

**Data use** (official Services Agreement, last modified 2026-06-22; "Your Data" docs)
- "Groq is not permitted to use Inputs or Outputs for training" (§4.2).
- Not retained by default. Up to 30-day logs for reliability and abuse, with an opt-out.
- **Zero Data Retention** is available to all customers.
- Data is stored in US GCP buckets.
- No free/paid distinction is stated for data handling.

**Commercial use**
- You may make the services available "to End Users through your Customer Applications" (§3.1).
- No resale or sublicensing (§6.3(c)).
- No account resale or lease (§3.2).
- **Must not use the services "in a manner intended to avoid incurring Fees"** (§6.3(d)(iii)), so multi-account free-tier pooling is prohibited.
- Free services carry **no indemnity** (§15.3(c)) and a **$5,000 liability cap** (§14.2).
- Not for consumer use.

### 7.3 Cost-model implications

- **Development:**
  - Gemini Flash-Lite (500 RPD) and Gemma 4 (14.4k RPD) for bulk dev calls;
  - Groq gpt-oss-120b for agent loops and judging (1k RPD, 8k TPM);
  - Gemini Flash (20 RPD) only for targeted tests;
  - **synthetic or test data only on the free Gemini tier**.
- **Production:** BYOK only. Default to paid keys. Detect key tier where possible. Explicit per-tenant consent for free keys. Per-tenant rate limiting and model-fallback chains (`ModelFallback` in Pydantic AI).
- **The judge must be a different family than the producer.** On a free budget that means Groq (open-weights families) ↔ Gemini, which is feasible.

---

## 8. Context engineering and cost control

### 8.1 Prompt caching

| Provider | Mechanism | Details |
|---|---|---|
| **Gemini** | **Implicit caching, on by default for 2.5+**; savings passed automatically. Explicit caches via `generateContent` for 2.5/3 models, **not** for Gemini 4 or the Interactions API. | Minimum prefix is 4,096 tokens for Gemini 4 / 3.x Flash / 3.1 Pro, and 2,048 for 2.5. Gemini 4 multi-turn: the full previous request is cached only if the new turn adds ≥ 2,050 tokens. Read `usage.total_cached_tokens`. Cached input is ~10% of the input price per the pricing table (e.g. 3.8 Flash: $0.075 vs $0.75), plus hourly storage for explicit caches. |
| **Groq** | **Automatic prefix caching** | gpt-oss-20b, gpt-oss-120b and gpt-oss-safeguard-20b only; **50% discount**; ~2 h TTL; cached tokens don't count toward rate limits (but are subtracted after processing). |

**Implications for prompt layout:**
1. Order prompts as static system prompt → tool schemas → few-shot examples → tenant context → variable task.
2. Keep tool lists stable per agent template; least privilege, so tools are chosen per template, not per call.
3. Avoid timestamps or IDs early in the prompt.

### 8.2 Routing and cascades

- **Cascade gated by deterministic validators.** Run a small, fast model first (Flash-Lite, gpt-oss-20b). Escalate to a stronger model only on schema, validator or contract failure, or low confidence. FrugalGPT (Chen et al. 2023) and RouteLLM (Ong et al. 2024) show large cost cuts at near-parity *[background]*. Rudra's contracts make escalation signals cheap and objective.
- **Route by step type, not by tenant:**
  - classification and triage → smallest model;
  - planning and re-planning → strongest available;
  - judge → different family, mid-size.
- **Context hygiene:**
  - claim-check large artifacts;
  - summarize tool results before re-injecting (Pydantic AI `PruneToolResults`);
  - per-step context budgets enforced by the orchestrator.

### 8.3 Structured output on small and fast models

- **Gemini:** schema-constrained JSON. Valid JSON matching the structure is guaranteed, but values can be wrong. Unsupported keywords (`minLength`, `maxLength`, `uniqueItems`…) are **silently stripped**. Overly complex schemas give `400 INVALID_ARGUMENT`. Combining with function calling is in **preview** for Gemini 3 and 4.
- **Groq:** `strict: true` constrained decoding on gpt-oss-20b, gpt-oss-120b, qwen3.8-27b and muse-spark-open. It requires all fields `required` and `additionalProperties: false`. **No streaming and no tool use combined with structured outputs.** Best-effort mode can return 400s.
- **Practice:**
  1. Keep schemas flat. Use enums for decisions and nullable rather than optional fields.
  2. Validate with Pydantic on the client and re-ask with the validation error, at most 2 times.
  3. For agents that need tools *and* typed output on Groq, use the "final answer as a tool call" pattern (Pydantic AI's default output-tool mode) instead of `response_format`.
  4. Track parse-failure rate per model in evals.

---

## Recommended AI stack for Rudra

| Layer | Choice | Rationale |
|---|---|---|
| Control plane | **Custom orchestrator** (planner → DAG → agent factory → auditor → stage/commit → autonomy ladder) | This is the product's core IP; no framework fits it |
| Agent runtime and model layer | **Pydantic AI (pin minor; v2)** with native provider clients; `ModelFallback` | Typed outputs, MCP, OTel, best durable-engine integrations, provider-neutral, no LiteLLM in the key path |
| Fallback option | Google ADK 2.x single agents + `TemporalModel` | If Gemini-specific features or Vertex Agent Engine become important |
| Durable execution | **Temporal**: dev server locally, self-host small or Temporal Cloud later, behind an `ExecutionBackend` interface. **DBOS** is the evaluated alternative if ops cost dominates. | GA AI integrations, best HITL and timers, versioning. Interface keeps the swap open. |
| Data | **Postgres + pgvector + FTS**, RLS per tenant, provenance columns | One store for state, memory and audit |
| Verification | Contracts → deterministic and state checks → cross-family LLM judge (binary rubric, order-swapped) → human sampling | Evidence says judges are weak at false-success detection |
| Security | Plan-then-execute, quarantined readers, taint-aware tool gate, Rule-of-Two factory check, staged commits, egress allowlist | Architectural rather than probabilistic |
| Observability | OTel GenAI semconv → **Langfuse Cloud Hobby**; Phoenix locally | Swap-able; zero cost |
| Evals | pytest + promptfoo (CI) + Inspect or DeepEval (scenarios), golden sets in git, cached responses | Fits free-tier RPD |
| Models (dev) | Groq gpt-oss-120b/20b; Gemini Flash-Lite and Gemma 4; Gemini Flash sparingly | Free-tier reality |
| Models (prod) | Tenant BYOK, paid keys by default, cross-family judge, per-step routing | Cost and ToS compliance |

**Main risks**
1. **Free-tier volatility.** Google publishes no guaranteed limits and cut Flash to about 20 RPD [unverified]. Plan for a small paid dev budget (Gemini Tier 1 has a $250 cap) if velocity suffers.
2. **Framework churn.** Pydantic AI and ADK both release weekly with breaking changes. Pin versions, add a thin internal `AgentRunner` adapter, and run contract tests on upgrade.
3. **Supply chain.** AI libraries are high-value targets (LiteLLM, March 2026). Use hash-pinned lockfiles (`uv lock`), delayed adoption of new releases, no tenant secrets in worker env (fetch from Secret Manager per call), and SBOM scanning.
4. **Judge validity.** Over-reliance on an LLM judge would let false successes through. State verification must be the primary gate.
5. **Temporal history and payload limits.** Long agent loops need claim-check plus continue-as-new from day one.
6. **BYOK data exposure.** A tenant's free key leaks their data into Google training. Explicit UX and policy are needed, and free keys must be blocked for EEA/UK/CH.

---

## Open questions

1. **Exact Groq Free vs Developer plan limits** for our org (console `/settings/limits`), and whether llama-3.3-70b remains on free.
2. **Exact Gemini free limits** for our project in AI Studio, since the published table is gone. Is 20 RPD on Flash confirmed?
3. **Temporal Cloud pricing** (sources conflict) vs self-host on GCP e2-medium/e2-standard. Do a TCO spike.
4. **DBOS without Conductor:** is single-region, multi-worker recovery acceptable using executor IDs? Is Conductor licensing needed for our scale?
5. **Can Rudra detect whether a BYOK Gemini key is free or paid** programmatically? *[unknown]*
6. **Pydantic AI v1 vs v2:** which line to pin, given v1 is still patched? Read the v2 migration and Compatibility Notes.
7. **OpenAI Agents API "managed durable sessions"** (public beta, 2026-09-10) [unverified]. Relevant only if OpenAI becomes a primary provider.
8. **OWASP Top 10 for Agentic Applications (2026):** not fetched. Map Rudra's controls against it.
9. **ADK 2.x resume semantics** (failed node re-runs) vs Rudra's idempotency design, if ADK is retained.
10. **Langfuse Cloud v4 migration** (reported 2026-11-16): confirm SDK compatibility.

---

## Sources

**Free tiers and terms (official)**
- Gemini rate limits: https://aistudio.google.com/docs/rate-limits (redirected from https://ai.google.dev/gemini-api/docs/rate-limits)
- Gemini API Additional Terms (effective 2026-03-23): https://aistudio.google.com/docs/terms
- Gemini pricing: https://aistudio.google.com/docs/pricing
- Gemini caching: https://aistudio.google.com/docs/caching
- Gemini structured output: https://aistudio.google.com/docs/structured-output
- Google APIs Terms of Service: https://developers.google.com/terms
- Third-party free-limit observation (2026-10-21) [unverified]: https://imisofts.com/blog/gemini-api-free-tier-limits/
- Groq rate limits: https://console.groq.com/docs/rate-limits
- Groq models: https://console.groq.com/docs/models
- Groq data handling: https://console.groq.com/docs/your-data
- Groq Services Agreement: https://console.groq.com/docs/legal/services-agreement
- Groq prompt caching: https://console.groq.com/docs/prompt-caching
- Groq structured outputs: https://console.groq.com/docs/structured-outputs

**Frameworks**
- ADK releases: https://github.com/google/adk-python/releases
- ADK 2.0: https://adk.dev/2.0/
- ADK models: https://adk.dev/agents/models/
- Temporal AI integrations GA (2026-10-30): https://temporal.io/blog/ai-framework-integrations
- Temporal ADK integration: https://docs.temporal.io/develop/python/integrations/google-adk
- Temporal LangGraph integration: https://docs.temporal.io/develop/python/integrations/langgraph
- Pydantic AI releases: https://github.com/pydantic/pydantic-ai/releases
- Pydantic AI durable execution: https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/
- OpenAI Agents SDK: https://pypi.org/p/openai-agents
- Third-party summary [unverified]: https://www.developersdigest.tech/blog/openai-agents-sdk-whats-new-2026
- Microsoft Agent Framework 1.0 / AutoGen maintenance [secondary]: https://venturebeat.com/ai/microsoft-retires-autogen-and-debuts-agent-framework-to-unify-and-govern
- LiteLLM compromise: https://bleepingcomputer.com/news/security/popular-litellm-pypi-package-compromised-in-teampcp-supply-chain-attack
- LiteLLM compromise (JFrog): https://research.jfrog.com/post/litellm-compromised-teampcp/

**Durable execution**
- Self-hosting comparison [third-party]: https://cadenya.com/other-content/self-hosting-ai-agent-infrastructure
- DBOS workflow communication: https://docs.dbos.dev/python/tutorials/workflow-communication
- Temporal payload limits: https://docs.temporal.io/troubleshooting/blob-size-limit-error
- Temporal claim-check pattern: https://docs.temporal.io/ai-cookbook/claim-check-pattern-python
- Temporal human-in-the-loop: https://docs.temporal.io/ai/cookbook/human-in-the-loop-python
- Restate awakeables: https://docs.restate.dev/develop/python/awakeables
- Restate OpenAI Agents SDK: https://docs.restate.dev/ai/sdk-integrations/openai-agents-sdk
- Hatchet human-in-the-loop: https://docs.hatchet.run/guides/human-in-the-loop
- Inngest wait-for-event: https://inngest.com/docs/features/inngest-functions/steps-workflows/wait-for-event
- Temporal single-server self-host [third-party]: https://automationatlas.io/guides/tutorial-temporal-self-hosted-deploy-2026/

**Verification**
- Zheng et al. 2023, MT-Bench / LLM-as-a-judge: https://arxiv.org/abs/2306.05685
- Wang et al. 2023, LLMs are not Fair Evaluators: https://arxiv.org/abs/2305.17926
- Panickssery et al. 2024, self-preference: https://arxiv.org/abs/2404.13076
- Survey on LLM-as-a-Judge: https://arxiv.org/html/2411.15594v6
- Norman et al. 2026 (withdrawn): https://arxiv.org/abs/2606.19544
- BabelJudge 2026: https://arxiv.org/pdf/2606.22329
- MLLM model-preference bias: https://arxiv.org/pdf/2604.11589
- Advani 2026, false success: https://arxiv.org/abs/2606.09863
- Huang et al. 2023, self-correction [background]: https://arxiv.org/abs/2310.01798
- Stechly et al. 2024, self-verification [background]: https://arxiv.org/abs/2402.08115
- Brown et al. 2024, Large Language Monkeys [background]: https://arxiv.org/abs/2407.21787
- Stroebl et al. 2024, Inference Scaling fLaws [background]: https://arxiv.org/abs/2411.17501
- Verga et al. 2024, PoLL [background]: https://arxiv.org/abs/2404.18796
- Anthropic, Demystifying evals for AI agents: https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

**Prompt injection**
- CaMeL: https://arxiv.org/abs/2503.18813
- Design Patterns for Securing LLM Agents: https://arxiv.org/abs/2506.08837
- Adaptive evaluation of out-of-band defenses (2026): https://arxiv.org/html/2606.26479v1
- Meta Agents Rule of Two: https://ai.meta.com/blog/practical-ai-agent-security/
- Simon Willison on 2025 injection papers: https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/
- Spotlighting [background]: https://arxiv.org/abs/2403.14720

**Memory**
- Mem0 state of memory 2026 (vendor): https://mem0.ai/blog/state-of-ai-agent-memory-2026
- Memory benchmark comparability: https://agentsystems.blog/agent-memory-benchmark
- Third-party comparison [unverified]: https://medium.com/@wasowski.jarek/i-compared-5-ai-agent-memory-systems-across-6-dimensions-none-wins-6a658335ed0a

**Evals and observability**
- Langfuse pricing: https://langfuse.com/pricing
- Langfuse self-hosting: https://langfuse.com/self-hosting
- ClickHouse acquires Langfuse: https://clickhouse.com/blog/clickhouse-acquires-langfuse-open-source-llm-observability
- OpenAI to acquire Promptfoo: https://openai.com/index/openai-to-acquire-promptfoo/
- Promptfoo joining OpenAI: https://promptfoo.dev/blog/promptfoo-joining-openai
- Observability comparison [third-party]: https://dev.to/gabrielanhaia/langfuse-vs-langsmith-vs-phoenix-vs-braintrust-the-honest-2026-comparison-253p

**Cost control** [background]
- FrugalGPT: https://arxiv.org/abs/2305.05176
- RouteLLM: https://arxiv.org/abs/2406.18665
