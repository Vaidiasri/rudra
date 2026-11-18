# ADR-0004: Agent runtime: custom orchestrator + Pydantic AI

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-1; [AI architecture research](../02-research/02-ai-architecture.md) §1

## Context
Rudra's core (planner, DAG execution, auditor, staging, trust ladder) is its differentiator and must be fully under our control. Agents themselves need: multi-provider model access, typed structured output, tool calling, MCP, tracing, and compatibility with a durable-execution engine. The initial choice was Google ADK.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| Google ADK 2.x | Multi-agent primitives; Temporal integration | Runtime rewrite in 2.0 plus breaking changes in 2.9/2.13/2.14; Gemini/GCP-first; non-Gemini models only via the LiteLLM connector |
| LangGraph | Mature graph runtime, popular | Its own orchestration model overlaps with ours; heavier abstractions |
| OpenAI Agents SDK | Clean API | OpenAI-centric |
| **Custom orchestrator + Pydantic AI for agents** | Native clients for all our providers; typed outputs; MCP; OpenTelemetry; integrations with durable-execution engines; thin, so our orchestrator stays ours | We own more orchestration code (intended) |

## Decision
Build the orchestrator ourselves. Run each agent on **Pydantic AI**, behind a small internal `AgentRuntime` interface so the runtime can be replaced. Google ADK is the documented fallback.

## Consequences
- The orchestrator, auditor and trust ladder are first-class, tested domain code.
- Pin Pydantic AI versions; upgrade deliberately with eval runs.
- The M0 spike (ADK/LiteLLM-oriented) is not carried forward as-is.

## Revisit when
Pydantic AI shows breaking churn or drops a provider we need, or a framework ships verification/contract features we'd otherwise build.
