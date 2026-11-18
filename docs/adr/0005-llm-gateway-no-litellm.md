# ADR-0005: Own LLM gateway on native provider clients; no LiteLLM

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-2, R-6; ADR-0004, ADR-0009

## Context
Every model call must go through one place that enforces tiers, priorities, per-key rate limits, fallbacks, metering and tenant key selection. LiteLLM offered this, but its PyPI package was compromised on 2026-03-24 (v1.82.7/1.82.8 shipped a credential stealer). The gateway is the component that holds every tenant's API keys.

## Decision
- Build our own **LLM gateway** (tier routing, priority admission, token buckets per key, cooldown on 429, fallback chains, usage metering, BYOK key resolution).
- Make provider calls through **Pydantic AI's native provider clients** (official SDKs underneath).
- **Supply-chain controls for all dependencies:** hash-pinned lockfiles, dependency and vulnerability scanning in CI, minimal dependency footprint for secret-holding services, and keys decrypted only at call time, never placed in process-wide env vars.

## Consequences
- More code to own in the gateway, but it is small and central to our cost model.
- New providers require a Pydantic AI-supported client (or a thin adapter).

## Revisit when
A provider-agnostic library with a strong security track record offers what the gateway needs.
