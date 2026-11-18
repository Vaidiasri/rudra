# ADR-0003: Observation: events and metadata only, user-owned data

- **Status:** Accepted (applies when the desktop observer ships, per ADR-0002)
- **Date:** 2026-11-18
- **Related:** Synthesis PD-4, PD-5; [Observation & privacy research](../02-research/03-observation-and-privacy.md)

## Context
"Full desktop + browser activity" capture conflicts with weak target machines (no local ML), EDR heuristics (keyboard hooks look like keyloggers), Chrome Web Store data policy, privacy expectations after Microsoft Recall, and workplace-monitoring law (GDPR, works councils, NY notice law, EU AI Act high-risk classification for worker monitoring/evaluation). Rule-based redaction alone cannot reliably remove PII; the safest data is data never captured.

## Decision
**Capture:** foreground window/app and title changes (`SetWinEventHook`), idle state, UI Automation reads limited to the focused element, and a Chrome extension restricted to an **allow-list of work sites**.

**Never capture:**
- continuous screenshots or OCR (allowed only in explicit, user-started demo recording),
- keystrokes or low-level keyboard/mouse hooks,
- password fields,
- excluded apps.

**Processing and storage:** process on-device; upload only abstracted, redacted episodes.

**User control:** opt-in, a visible indicator, pause, per-app exclusions, export and delete.

**Ownership:** the individual owns their observation data. In team workspaces managers see shared workflow outputs only, never individual activity, scores or comparisons. No feature evaluates worker performance.

## Consequences
- Pattern discovery must work from coarse event sequences (supported by routine-discovery research).
- Team product design excludes manager dashboards over individual activity.
- Lower compliance burden; a DPIA template is still prepared before team launch.

## Revisit when
Never relax without a new ADR plus legal review.
