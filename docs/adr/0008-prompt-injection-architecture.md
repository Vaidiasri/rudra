# ADR-0008: Prompt-injection defence by architecture

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-5; AI architecture research §4

## Context
Rudra's agents read untrusted content (emails, web pages, tickets, PR text) and can take actions (send mail, post to Slack, comment on PRs). Prompt-level defences alone are unreliable. Current best practice combines plan-then-execute, privilege separation (dual-LLM / CaMeL-style data-flow control) and Meta's "Rule of Two" (an agent should not simultaneously process untrusted input, access sensitive data, and change state or communicate externally).

## Decision
1. **Plan before reading:** the workflow plan and tool set are fixed before any untrusted content is read. Untrusted content cannot add steps or tools.
2. **Privilege separation:** agents that read untrusted content get **no side-effect tools**. They return typed data only.
3. **Provenance gate (CaMeL-lite):** values flowing into side-effecting tool arguments (recipients, URLs, repo/PR targets) carry a provenance tag. A code-level policy rejects arguments derived from untrusted sources unless explicitly allowed by the contract.
4. **Rule of Two at agent creation:** the Agent Factory refuses agent configs that combine all three of untrusted input, sensitive data access and external side effects.
5. **Stage → audit → commit** for every external write (ADR-0007 security layer runs first).
6. Untrusted content is always delimited and labelled in prompts (spotlighting), as defence in depth.

## Consequences
- The agent template and tool-registry design must express capabilities and data provenance.
- Some workflows need two agents (a reader and an actor) instead of one.

## Revisit when
New attack classes appear in red-team evals, or provenance tracking proves too coarse.
