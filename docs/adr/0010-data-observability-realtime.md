# ADR-0010: Postgres + pgvector, Langfuse via OpenTelemetry, SSE for realtime

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-6, TD-8

## Context
We need relational data with tenant isolation, vector search for memory and retrieval, LLM tracing and evals, and server-to-client realtime updates. All of this on minimal infrastructure.

## Decision
- **Database:** PostgreSQL with **pgvector** (Supabase-managed). Tenant isolation enforced in the database (RLS) as well as in the app. No graph or vector DB until a measured need.
- **Observability:** OpenTelemetry instrumentation everywhere. LLM traces exported to **Langfuse Cloud** (free plan), so the tracing backend is swappable. No self-hosted Langfuse on dev machines (it needs ClickHouse, Redis and S3).
- **Realtime:** **SSE** for server → client (one multiplexed stream per client, `Last-Event-ID` replay from an `events` table, Postgres `LISTEN/NOTIFY` fan-out); **REST** for client → server. No WebSockets.

## Consequences
- One primary datastore to operate, back up and secure.
- Event replay retention and HTTP/2 serving are required.

## Revisit when
Vector or graph queries become a measured bottleneck, or the Langfuse free plan limits are exceeded.
