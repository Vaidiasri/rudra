# ADR-0001: Positioning: verified automation for engineering teams

- **Status:** Accepted
- **Date:** 2026-11-18
- **Related:** [Research synthesis](../02-research/00-synthesis.md) PD-1, PD-2, PD-6, PD-7

## Context
The original idea was a broad proactive AI assistant that observes everything and automates anything. Research shows general "AI does your work" agents are now bundled into Microsoft 365, Google, Anthropic and OpenAI subscriptions, and several independent assistant startups closed or pivoted in 2026 because of it. Meanwhile no end-user product verifies agent output against explicit success criteria using real system state, and users complain loudly about paying for failed agent runs.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| A. General proactive assistant | Big vision, wide market | Head-on competition with free bundles; no defensible wedge |
| B. Activity-discovery tool ("watches you, suggests automations") | Novel UX | Partly occupied (Cofia, Screenpipe, Highlight); discovery without reliable execution has weak retention |
| **C. Verified automation for engineering work** | Real gap; verifiable through APIs (Jira/GitHub/Slack/Gmail); clear ROI; aligns pricing with outcomes | Narrower market at first; must earn depth in a few workflows |

## Decision
Rudra is **"automation for engineering teams that proves it worked."** The core is success contracts, state-based verification, retry/escalation and earned autonomy, applied to a small set of high-frequency engineering ops workflows. Discovery (from tool data now, desktop activity later) feeds it. Voice is out of v1. Users never pay for failed or unverified runs.

## Consequences
- Every MVP workflow must be verifiable against real external state; workflows that can't be verified are deprioritized.
- We do not build our own PR reviewer (CodeRabbit et al. own it); we build PR-review *digests* and hygiene.
- Metering and pricing count only verified, committed runs.

## Revisit when
A major platform ships contract-based verified automation for engineering tools, or user research shows the niche is too narrow to sustain revenue.
