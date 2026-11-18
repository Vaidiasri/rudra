# ADR-0002: MVP surfaces and audience: web + Slack + connectors, individuals first

- **Status:** Accepted; partially superseded by [ADR-0011](0011-conversation-capture.md) (desktop call companion + Android app are now MVP)
- **Date:** 2026-11-18
- **Related:** Synthesis PD-3, PD-6, open question 1; ADR-0001, ADR-0003

## Context
The original plan shipped a Windows desktop observer, browser extension and voice in v1. Research found:
- Windows code signing for an India-based company needs an OV certificate on a cloud HSM (Azure Artifact Signing is unavailable).
- Corporate EDR may quarantine activity-capture agents.
- Employer-deployed observation is regulated workplace monitoring.
- Discovery from observation is not differentiating on its own.

The audience also had to be chosen: individuals first (bottom-up) or teams first.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| Desktop + web + Slack + voice in v1 | Full vision | Months of non-core work; signing cost; EDR and legal risk before value is proven |
| **Web + Slack + API connectors in v1; desktop observer in phase 2** | Fastest path to the core value (verified automation); no installer; low legal risk; discovery still possible from connected-tool data | Less "magic" discovery at launch |
| Teams first | Bigger contracts | Sales cycle, compliance (DPIA, works councils), admin features up front |
| **Individuals first, team sharing next** | Fast feedback, bottom-up adoption, lighter compliance | Lower initial revenue per account |

## Decision
- **v1 surfaces:** web app, Slack app, notifications (email/web/Slack) and API connectors (Jira, GitHub, Slack, Google Calendar, Gmail). Discovery in v1 uses connected-tool activity.
- **Phase 2:** Windows desktop observer + Chrome extension under ADR-0003 rules. Voice later still.
- **Audience:** individual engineers and tech leads first; team workspaces (shared workflows, shared outputs) next.

## Consequences
- Multi-tenancy is still designed in from day one (a personal workspace is a tenant of one).
- The data model and APIs must accept desktop episodes later without redesign.
- Pricing and onboarding are self-serve.

## Revisit when
Connector-based discovery proves too weak to find automations users accept, or a design partner team requires desktop discovery to buy.
