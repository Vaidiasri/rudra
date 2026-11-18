# ADR-0006: Durable execution behind an interface; Temporal vs DBOS by spike

- **Status:** Proposed (final choice pending spike)
- **Date:** 2026-11-18
- **Related:** Synthesis TD-3; AI architecture research §2

## Context
Workflow runs span minutes to days (approvals, schedules, retries). They must survive crashes and deploys and support human-in-the-loop signals. Constraints: a near-zero budget, an 8 GB dev machine, and the principle "one Postgres before many datastores".

## Options considered
| Option | Pros | Cons |
|---|---|---|
| Temporal | Most mature; GA AI integrations (2026-10-30); signals, timers, schedules | Extra system to run (or pay Temporal Cloud); determinism rules; 2 MB payload / 50 MB history limits |
| DBOS | Runs on our existing Postgres; no extra infrastructure; Python-native | Younger; smaller ecosystem |
| Plain Postgres queue + state machine | Simplest | We'd rebuild durability, timers and retries |

## Decision (current)
Workflow code targets an internal **`WorkflowEngine` interface** (start, signal, query, cancel, schedule). Run a time-boxed spike that implements the same reference workflow (stage → audit → approval signal → commit, with crash recovery) on **both Temporal and DBOS**. Choose by: correctness under failure, operational cost, developer experience, and local footprint. Either way: claim-check large payloads (store them in Postgres/object storage, pass references) and keep histories bounded.

## Revisit when
The spike is complete. This ADR is then updated to Accepted with the result.
