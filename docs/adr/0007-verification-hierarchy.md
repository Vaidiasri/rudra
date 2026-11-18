# ADR-0007: Verification hierarchy: real state first, LLM judge second

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** Synthesis TD-4; ADR-0001; AI architecture research §3

## Context
Verified automation is Rudra's core promise. Research shows agents frequently report false success, and LLM judges detect these false completions poorly (AUROC ≤ 0.65 in a 2026 study). Judges also show self-preference and position biases.

## Decision
Every step's success contract is evaluated in this order. A step fails at the first failing layer.
1. **Security and provenance checks:** only declared tools were used, outbound arguments come from allowed sources (ADR-0008), recipients and targets are within scope.
2. **Deterministic checks against real external state:** re-read Jira/GitHub/Slack/Gmail through APIs and compare with the contract (e.g. all tickets covered, draft exists with the expected recipients, comment posted on the right PR).
3. **LLM judge, secondary signal only:** for qualities that can't be checked deterministically (tone, faithfulness). It uses a **different model family** from the worker, with rubric-based structured verdicts and calibration sets.
4. **Sampled human review:** a percentage of runs feeds judge calibration and measures false-success rates.

Trust-ladder promotion requires measured thresholds on audited pass rate, human override rate and false-success rate over N runs.

## Consequences
- Workflows whose success can't be checked against state are lower priority (ADR-0001).
- Connectors need read-back operations, not only write operations.
- The eval and labelling pipeline is part of the MVP, not an afterthought.

## Revisit when
Judge reliability measurably improves on our calibration sets.
