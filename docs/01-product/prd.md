# Rudra: Product Requirements Document (PRD)

- **Status:** Approved (2026-11-20). Persona, workflows W-1–W-3, scope and slice plan reviewed with the founder.
- **Version:** 1.0
- **Last updated:** 2026-11-18
- **Inputs:** [Research synthesis](../02-research/00-synthesis.md), [ADR-0001](../adr/0001-positioning-verified-automation.md), [ADR-0002](../adr/0002-mvp-surfaces-and-audience.md), [ADR-0003](../adr/0003-observation-data-minimization.md), [ADR-0007](../adr/0007-verification-hierarchy.md), [ADR-0009](../adr/0009-llm-usage-and-key-policy.md)

---

## 1. Summary

Rudra automates the coordination work of developers who juggle several client projects at once: turning client calls into requirements and tickets, planning the day across multiple trackers and deadlines, and handling follow-ups and help requests with teammates. Unlike general AI agents, **every Rudra run is verified against an explicit success contract by checking the real state of Jira, Linear, GitHub and Slack.** Failed runs are retried or escalated, never silently reported as done. Users only pay for verified work. Workflows earn autonomy as they prove themselves.

**One-line positioning:** *Verified automation for developers juggling multiple clients.* (A refinement of ADR-0001's "automation for engineering teams that proves it worked"; team features follow in phase 2.)

---

## 2. Problem

### 2.1 The work
Developers at software services companies often work for several clients at once, and each client brings its own tools: one uses Jira, another Linear, another GitHub Issues. Requirements arrive in calls on Google Meet, Microsoft Teams or the phone, and live only in the developer's memory or scattered notes. Every day the developer has to rebuild a picture of what is due, for whom and by when, across all of these tools, and then chase teammates on Slack for answers, reviews and help. The work is repetitive, error-prone (a missed requirement or a forgotten deadline lands on the developer) and spread across tools no single product unifies.

### 2.2 Why existing tools don't solve it
| Option | Why it falls short | Evidence |
|---|---|---|
| General agents (ChatGPT agent, Claude Cowork, Gemini Spark, Manus) | Unreliable for repeatable work; "fake done" reports; failed runs still cost credits; scheduled runs silently skipped | [Research 01 §3.1, §3.4](../02-research/01-market-and-competitors.md) |
| Workflow tools (Zapier, n8n, Make) | Exact but rigid; users must design every rule; poor at judgment tasks like summarising | [01 §2, §3.5] |
| Single-purpose tools (status-report bots, Linear triage) | Each covers one job inside one tool; no cross-tool verification | [01 §2] |
| Doing it by hand | Hours per week; inconsistent; forgotten when busy | Persona interviews (to be run, §11) |

### 2.3 The core insight
For developer coordination work, **success is checkable**. "Does every requirement from the call have a ticket?", "Does today's plan include every open item assigned to me in all three trackers?", "Did the follow-up land in the right thread, and did anyone reply?" are all API queries. Rudra makes those checks the definition of done.

---

## 3. Target users

ADR-0002: **individuals first**, team workspaces next. The founder is user #1 and matches PER-1; the product is dogfooded daily from the first alpha.

### 3.1 Personas
| ID | Persona | Context | Top pains (ranked) | Primary workflows |
|---|---|---|---|---|
| **PER-1 (primary)** | **Multi-client developer** at a software services company | Several client projects at once, each with its own tracker (**Jira, Linear, GitHub Issues**); code on **GitHub**; team chat on **Slack**; client calls on **Google Meet, Microsoft Teams, phone/WhatsApp**; Google Calendar | 1. Turning client calls into clear requirements and tickets · 2. Deciding daily priorities across three trackers and their deadlines · 3. Follow-ups and getting help from teammates | W-1, W-2, W-3 |
| PER-2 (later) | **Tech lead** of a multi-client team | Same stack; also reports upward and to clients | Status reports, team-wide ticket hygiene, stale PRs | W-5–W-8 (later) |
| PER-3 (later) | **Technical PM** at a services company | Tracker-heavy; client-facing | Client status reports, release communications | W-5, W-8 (later) |

### 3.2 Anti-personas (we do not design for them)
- Managers who want to **monitor or score individual employees** (ADR-0003: never).
- Non-technical ops teams needing general business automation (Zapier/Make territory).
- Enterprise automation CoEs buying task mining.
- People who want a general chat assistant.

---

## 4. Product principles

1. **Proven, not claimed.** Every run shows what was checked, against what evidence, and the result.
2. **Draft before act.** New workflows start by drafting; autonomy is earned (trust ladder).
3. **Quiet by default.** One daily digest; real-time pings only for approvals the user opted into and for failures that need them.
4. **The user owns their data.** Minimal collection, clear controls, export and delete. No surveillance features, ever.
5. **Pay for outcomes.** Failed or unverified runs are never billed.
6. **Fast time-to-value.** First verified run within 15 minutes of sign-up, using templates.

---

## 5. Core concepts (product vocabulary)

| Concept | Meaning for the user |
|---|---|
| **Workflow** | A recurring automation, e.g. "Weekly status report for Team Phoenix". Created from a template and customised. |
| **Run** | One execution of a workflow (scheduled, triggered by an event, or manual). |
| **Success contract** | The checklist that defines "done right" for a workflow, shown in plain language: "Includes every ticket that changed status this week", "Posted to #phoenix-updates", "Under 300 words". Users can view and adjust it. |
| **Verification report** | Per run: each contract check, pass/fail, and the evidence (links to the tickets, PRs and messages that were checked). |
| **Draft / approval** | Output staged for the user's review (approve / edit / reject) before anything is sent or changed. |
| **Trust level** | Per workflow: *Suggest → Draft → Supervised → Autonomous*. Promoted after consistent verified runs, demoted on failures or rejections. Always visible and user-controllable. |
| **Suggestion** | A proposed new workflow found by Rudra from connected-tool activity, e.g. "You post a summary in #team every Friday. Automate it?" |
| **Digest** | The daily summary: runs completed, drafts awaiting approval, suggestions, failures. |

---

## 6. Feature catalogue and admission

Admission test ([process §3](../00-architecture-process.md)): **P**ain (real, frequent), **O**utcome (measurable), **D**ifferentiating or **E**ssential, **V**erifiable/reliable, **C**ost-affordable. Scores: ✅ pass, ⚠️ partial, ❌ fail.

### 6.1 Platform features

| ID | Feature | P | O | D/E | V | C | Verdict |
|---|---|---|---|---|---|---|---|
| F-001 | **Workflow runtime with success contracts** | ✅ | ✅ | E+D | ✅ | ✅ | **MVP** |
| F-002 | **Verification report** per run with evidence links | ✅ | ✅ | D | ✅ | ✅ | **MVP** |
| F-003 | **Auto-retry with feedback, re-plan, escalation** | ✅ | ✅ | D | ✅ | ⚠️ cost of retries capped | **MVP** |
| F-004 | **Staged actions and approvals** (web + Slack, approve/edit/reject) | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-005 | **Trust ladder** per workflow, with user override | ✅ | ✅ | D | ✅ | ✅ | **MVP** |
| F-006 | **Connectors:** GitHub (code + Issues), Jira Cloud, Linear, Slack, Google Calendar | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-029 | **Unified work view across trackers:** everything assigned to me in Jira, Linear and GitHub Issues, normalised (client/project, deadline, status, priority) | ✅ | ✅ | D | ✅ | ✅ | **MVP** (foundation for W-2) |
| F-030 | **Manual call input:** paste notes or a transcript, or upload a recording (last-resort capture adapter) | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-031 | **Desktop call companion (Windows):** auto-detects calls (Meet, Teams, Zoom, Slack, WhatsApp Desktop, Phone Link cellular), captures call audio + mic as separate tracks, invisible to the client (ADR-0011) | ✅ | ✅ calls captured / client calls | D (USP) | ✅ | ✅ | **MVP** |
| F-032 | **Android app:** in-person recording, post-call voice memo, import of the phone's built-in call recordings, approvals on the go | ✅ | ✅ | E | ✅ | ✅ | **MVP** (iOS later) |
| F-033 | **Coverage guard:** detects likely client conversations not captured and asks for a 30-second memo | ✅ | ✅ coverage % | D | ✅ | ✅ | **MVP** |
| F-034 | **Client registry and consent records:** clients, their trackers/projects, **importance tier (High / Normal / Low)**, time zone, jurisdiction, consent status, never-transcribe flag (ADR-0012) | ✅ | ✅ | E (legal) | ✅ | ✅ | **MVP** |
| F-035 | **Conversation intelligence:** transcript (Hinglish-capable, ADR-0013) → requirements, build plan, blind spots, clarification questions, contradictions with earlier calls, change detection against existing tickets | ✅ | ✅ | D | ⚠️ judgement parts verified via traceability (every item cites the transcript) | ⚠️ STT + LLM cost | **MVP** |
| F-038 | **Client board:** per-client workspace inside Rudra; every item from every conversation, rated (T-shirt complexity, dev/test/deploy effort, skills & tech), with citations, history, changes and contradictions ([W-1 design](workflows/w1-client-conversations.md)) | ✅ | ✅ | D | ✅ citations | ✅ | **MVP** |
| F-039 | **Planning session:** interactive prioritisation and delivery planning with Rudra (approach, good practices, testing, deployment mode, skills); produces a delivery plan | ✅ | ✅ items triaged | D | ✅ every item triaged | ⚠️ LLM cost | **MVP** |
| F-040 | **Client sign-off summary:** shareable "requirements we discussed" document the user sends to the client; confirmations and corrections recorded on the board | ✅ scope disputes | ✅ % items confirmed | D | ✅ | ✅ | **MVP** |
| F-041 | **Deadline & commitment detection** in Slack messages and WhatsApp notifications (shadow deadlines with evidence; offer to write them into tickets) ([W-2 design](workflows/w2-daily-plan.md)) | ✅ | ✅ | D | ✅ evidence-cited | ⚠️ LLM cost (cheap tier) | **MVP** |
| F-042 | **Android notification reader:** reads WhatsApp message notifications from client contacts only (on-device filter), plus share-to-Rudra | ✅ | ✅ | D | ✅ | ✅ | **MVP** |
| F-043 | **Open-loop tracker:** every follow-up tracked from draft to answer; urgency-based nudges; answers fed back into board/tickets ([W-3 design](workflows/w3-follow-ups-and-help.md)) | ✅ | ✅ loops closed | D | ✅ read-back | ✅ | **MVP** |
| F-044 | **Expert finder:** who can help, from git/CODEOWNERS, ticket and Slack history, availability; evidence shown | ✅ | ✅ | D | ✅ | ✅ | **MVP** |
| F-045 | **Stuck detection:** gentle, opt-out offer of help when an in-progress item stalls; never reported to anyone else | ✅ | ✅ | D | ✅ | ✅ | **MVP** |
| F-036 | Later capture adapters: WhatsApp Business number, cloud business phone line, Plaud import, Meet/Teams org transcripts, iOS app | ✅ | ✅ | D | ✅ | ⚠️ | **Later** (ADR-0011 order) |
| F-037 | Rudra Bluetooth bridge (hardware: capture any call on any phone) | ✅ | ✅ | D (moat) | ⚠️ prototype | ⚠️ hardware | **R&D track** |
| F-007 | **Workflow templates** (the W-series below) with parameters | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-008 | **Natural-language customisation** of templates ("only include tickets labelled backend", "post on Thursdays") with contract preview | ✅ | ✅ | D | ✅ validated spec | ✅ | **MVP** |
| F-009 | **Dry run / preview** before enabling a workflow | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-010 | **Daily digest** (Slack DM or email) with notification cap | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-011 | Discovery from connected-tool activity (suggest new automations) | ⚠️ unproven precision | ✅ acceptance rate | D | ⚠️ | ✅ | **Later** (parked 2026-11-20: doesn't serve PER-1's ranked pains) |
| F-012 | **Style learning from edits** (drafts adapt to the user's tone and format) | ✅ | ✅ edit rate | D | ✅ edit-distance metric | ✅ | **MVP** |
| F-013 | **Run history and audit log** (every action Rudra took, with who approved) | ✅ | ✅ | E | ✅ | ✅ | **MVP** |
| F-014 | **API key management (BYOK)** with tier detection and free-key warnings | ✅ | ✅ | E (ADR-0009) | ✅ | ✅ | **MVP** |
| F-015 | **Usage metering of verified runs** + plan limits | ✅ | ✅ | E (revenue) | ✅ | ✅ | **MVP** (billing integration at public launch) |
| F-016 | Fully free-form workflow creation from a prompt (no template) | ⚠️ | ⚠️ | D | ⚠️ planner reliability | ⚠️ | **Later** (after template-based quality is proven) |
| F-017 | Team workspaces: shared workflows, shared outputs, roles | ✅ | ✅ | E for teams | ✅ | ✅ | **Phase 2** |
| F-018 | Desktop *activity* observer + Chrome extension (ADR-0003 rules); builds on the F-031 desktop app | ⚠️ | ⚠️ | D | ⚠️ | ✅ app already exists | **Phase 2** |
| F-019 | ~~Linear connector~~ | | | | | | Moved into F-006 (MVP) |
| F-020 | Gmail connector (read/send) | ⚠️ | ⚠️ | ❌ commodity | ✅ | ❌ restricted-scope security assessment | **Later**; re-evaluate scope cost |
| F-021 | Confluence / Google Docs publishing | ✅ | ✅ | E for some | ✅ | ✅ | **Phase 2** |
| F-022 | Workflow graduation (stable steps compiled to code) | — | ✅ cost | D | ✅ | ✅ saves cost | **Later** (needs run data) |
| F-023 | ~~Mobile app~~ | | | | | | Android app moved into MVP as F-032; iOS in F-036 |
| F-024 | Voice interface | ❌ | ❌ | ❌ | — | ❌ | **Parking lot** (ADR-0001) |
| F-025 | Own AI PR reviewer | ✅ | ✅ | ❌ CodeRabbit et al. | — | — | **Parking lot**: integrate, don't compete |
| F-026 | Inbox triage / email agent | ⚠️ | ⚠️ | ❌ commodity (Notion Mail lesson) | ⚠️ | ❌ | **Parking lot** |
| F-027 | General computer-use / browser agent | ⚠️ | ❌ | ❌ | ❌ brittle | ❌ | **Parking lot** |
| F-028 | Manager dashboards of individual activity or productivity | — | — | — | — | — | **Rejected permanently** (ADR-0003) |

### 6.2 MVP workflow templates (the W-series)

Selection criteria: ranked pains of PER-1 (founder input, 2026-11-18), outcome checkable via API (ADR-0007), clear time saved. Each MVP workflow gets a detailed design (inputs, outputs, success contract, edge cases) before the requirements phase. **Validated further with similar developers (§11) before MVP freeze.**

| ID | Workflow | Trigger | Output | Key contract checks (state-based) | Verdict |
|---|---|---|---|---|---|
| **W-1** | **Client conversations → board → plan → tickets** ([detailed design](workflows/w1-client-conversations.md)) | **Automatic** when a captured client conversation ends (F-031/F-032); memo or upload as fallback (F-030, F-033) | Stage 1 (auto): items added to the **client board** (F-038), rated by T-shirt complexity, dev/test/deploy effort, skills & tech; blind spots, clarification questions, changes and contradictions. Stage 2 (with user): **planning session** (F-039): priority, delivery approach, good practices, deployment mode. Stage 3 (on approval): tickets in the client's tracker; questions → W-3 follow-ups | Every requirement in the note maps to a ticket or is explicitly marked "no ticket"; created tickets exist in the right project with required fields; each ticket links back to the requirements note; deadlines mentioned in the call appear on tickets; no invented requirements (each traces to a source passage) | **MVP: priority 1** |
| **W-2** | **Daily plan across trackers** ([detailed design](workflows/w2-daily-plan.md)) | Schedule (each morning) + on demand; re-plan proposals on urgent changes | Prioritised plan for today across trackers, client boards and shadow deadlines from Slack/WhatsApp; ranked by deadline × client tier (deterministic, explainable); fitted into calendar free time; flags overdue, blocked, at-risk and unticketed requests; delivered via Slack / Android / web / email | Every open item assigned to me in all connected trackers is considered (included or explicitly deferred with reason); ordering respects deadlines; planned work fits available calendar time; no item listed that is closed or doesn't exist | **MVP: priority 2** |
| **W-3** | **Follow-ups, open loops and help requests** ([detailed design](workflows/w3-follow-ups-and-help.md)) | W-1 questions, W-2 blocked/at-risk items, manual, stuck detection | Open loops tracked end to end: Slack messages sent as the user after approval; client WhatsApp/email pre-filled for the user to send; replies detected and answers fed back to the board/tickets; urgency-based nudges; expert finder + complete help requests | Message posted to the intended person/thread; includes the referenced links; reply detected or a single nudge sent; never messages anyone outside the workspace or outside the allow-list | **MVP: priority 3** |
| W-4 | Estimate assistant (effort for dev/test/deploy from my history) | On demand | Estimate breakdown with confidence | Checkable only later (estimate vs actual); accuracy tracked over time | **Later** |
| W-5 | PR review queue digest | Schedule (daily) | Slack DM of PRs awaiting me / waiting on others | Every open PR requesting my review included | **Later** (could fold into W-2) |
| W-6 | Standup prep | Schedule | Yesterday / today / blockers draft | Every item I touched considered | **Later** (could derive from W-2) |
| W-7 | Weekly status / client report | Schedule | Report draft → Slack or email | Every changed ticket covered | **Phase 2** (PER-2/PER-3) |
| W-8 | Release notes | On tag/milestone | Grouped release notes | Every merged PR in range included | **Phase 2** |
| W-9 | Research summary with citations | Manual | Brief with sources | Citations resolve and support claims | **Later** (verification weaker) |

---

## 7. Key user journeys

### J-1 Onboarding to first verified run (target ≤ 15 min)
1. Sign up (Google or GitHub sign-in).
2. Connect the trackers used for each client (Jira, Linear, GitHub Issues), GitHub, Slack and Google Calendar (OAuth). Map each connected project to a client.
3. Add a model API key (BYOK) or use the trial allowance. Free-tier keys show a data-use warning (ADR-0009).
4. Pick a template (e.g. W-2 daily plan), set parameters (clients, time, delivery channel).
5. Review the plain-language success contract.
6. **Dry run:** Rudra executes, shows the output and the verification report. Nothing is sent.
7. Enable at trust level *Draft*.

### J-2 Daily use
1. Digest arrives in Slack once a day: completed runs (✅ verified), drafts awaiting approval, any failures needing attention, at most one new suggestion.
2. User approves, edits or rejects drafts from Slack or web.
3. Edits feed style learning (F-012).

### J-3 Failure and escalation
1. Verification fails (e.g. two tickets missing from the report).
2. Rudra retries with the failure as feedback (bounded retries).
3. If still failing, the run is marked **Needs attention** with the failed checks and evidence. The user can fix inputs, adjust the contract, or run it manually.
4. Nothing is sent, and the run is not billed.

### J-4 Earning autonomy
1. After N consecutive verified runs approved without edits, Rudra offers to promote the workflow (*Draft → Supervised → Autonomous*).
2. The user accepts or declines. Any failure or rejection demotes the workflow one level and says why.

### J-5 Suggestion
1. A detector notices a repeated pattern (e.g. a weekly update the user posts manually).
2. The digest shows one suggestion with the evidence ("you posted similar summaries on these 4 Fridays").
3. Accept → pre-filled template → dry run (J-1 steps 5–7).

---

## 8. Success metrics

| Type | Metric | MVP target (beta) |
|---|---|---|
| **North star** | **Verified runs per weekly active user** | ≥ 8 / week |
| Activation | Sign-up → first verified dry run | ≥ 60% within 24 h; median ≤ 15 min |
| Value | Self-reported time saved per user per week | ≥ 1 hour |
| Quality | Verified success rate (runs passing contract) | ≥ 90% |
| Quality | **False-success rate** (passed verification but judged wrong in sampled human review) | ≤ 2% |
| Quality | Drafts approved without edits, by the 4th run of a workflow | ≥ 70% |
| Trust | Workflows at *Supervised* or higher after 4 weeks | ≥ 30% |
| Proactivity | Suggestion acceptance rate | ≥ 25% |
| Noise | Notifications per user per day (excluding opted-in approvals) | ≤ 2 |
| Retention | Week-4 retention of activated users | ≥ 40% |
| Cost | LLM cost per active user per month | Set in NFRs (phase 3) |

---

## 9. Pricing hypothesis (to validate)

| Plan | Price (hypothesis) | Includes |
|---|---|---|
| Free | $0 | 2 workflows, Draft level only, limited verified runs/month, BYOK required |
| Pro | ~$24–30 / month | Unlimited workflows, all trust levels, more verified runs, platform model allowance or BYOK |
| Team (phase 2) | ~$30–40 / seat / month | Shared workflows and outputs, admin, SSO later |

**Rules:** failed and unverified runs are never counted. No surprise credit burn: usage shown in advance. Validate price points in interviews and beta.

---

## 10. Non-goals (v1)

- General-purpose assistant or chat product.
- Computer-use or browser-automation agents.
- Our own AI code reviewer.
- Email inbox management.
- Desktop/browser *activity* capture (phase 2, under ADR-0003). Call capture is in scope (ADR-0011).
- iOS / iPadOS app (later).
- Meeting bots or anything visible to the client in a call.
- Voice interface.
- Employee monitoring, productivity scoring, or manager views of individual activity (never).
- On-prem / self-hosted deployment.
- Non-engineering business workflows (sales, HR, finance).

---

## 11. Validation plan (gate before MVP scope freeze)

The workflow list and pricing are hypotheses until tested:
1. **Dogfooding:** the founder uses W-1 to W-3 on real client work during development.
2. **Interviews:** 10–15 developers at services companies who work across multiple clients (India first, then global). Validate frequency and time spent per workflow, current workarounds, trust concerns, willingness to pay, connector priorities.
3. **Concierge test:** for 3–5 interviewees, produce W-1/W-2 outputs semi-manually for 2 weeks and measure edits and perceived value.
4. **Decision:** confirm or replace each W-series workflow; confirm Jira vs Linear priority; confirm pricing range.

---

## 12. Release plan

The MVP ships as **vertical slices**. Each slice is usable on its own and dogfooded daily by the founder before the next starts (approved 2026-11-20).

| Slice | Theme | Scope | Exit criteria |
|---|---|---|---|
| **1** | **Never miss a requirement** (W-1 core, USP) | Web app; client registry + consent (F-034); conversation intelligence (F-035); client board (F-038); sign-off summary (F-040); verification engine (F-001, F-002); manual upload (F-030) **first, to validate analysis quality on real Hinglish calls**; then the Windows desktop call companion (F-031) | On ≥ 10 real client calls: item recall ≥ 90% vs founder's own list, zero uncited items, founder rates blind spots/questions useful on ≥ 70% of calls; then desktop capture works on ≥ 95% of laptop calls |
| **2** | **Know what to do today** (W-2) | Jira, Linear, GitHub, Calendar connectors (F-006); unified view (F-029); daily plan; shadow deadlines from Slack (F-041); delivery via Slack/web/email; digest (F-010) | Plan completeness verified daily for 2 weeks; founder follows the plan on ≥ 70% of days |
| **3** | **Close every loop** (W-3) | Open loops (F-043), Slack sending, nudges, expert finder (F-044), stuck detection (F-045), board → tickets (W-1 stage 3), approvals (F-004), retries (F-003) | No dropped loops over 2 weeks; ≥ 80% of loops closed with answers fed back |
| **4** | **Phone coverage** | Android app (F-032): memos, in-person recording, OEM call-recording import, WhatsApp notification reader (F-042); coverage guard (F-033) | ≥ 80% of the founder's client conversations captured or memo'd |
| **5** | **Ready for others** | Planning session (F-039), trust ladder (F-005), style learning (F-012), BYOK (F-014), metering/billing (F-015), onboarding, templates (F-007–F-009) | Private beta with 20–50 multi-client developers; metrics in §8 trending to target |
| Later | Teams & expansion | Team workspaces (F-017), desktop activity observer (F-018), later capture adapters (F-036), Bluetooth bridge R&D (F-037), iOS | Per later PRD |

---

## 13. Risks and assumptions

| ID | Risk / assumption | Mitigation |
|---|---|---|
| A-1 | Assumption: these workflows are frequent and painful enough to pay for | Validation plan §11 |
| A-2 | Assumption: state-based contracts catch most failures | Measure false-success via sampled review from alpha onward |
| R-1 | Platforms add verified automation | Depth in engineering workflows; speed |
| R-2 | Jira/GitHub/Slack API rate limits or permission scopes block workflows | Connector research in HLD; minimal scopes; caching |
| R-3 | Users won't bring their own API keys | Trial allowance on platform keys with caps; measure drop-off at key step |
| R-4 | LLM costs exceed price | Cost NFR, cheap-model routing, deterministic-first verification |
| R-5 | Slack app review / Marketplace requirements delay launch | Start Slack app review early; distribute unlisted during beta |

---

## 14. Open questions
1. ~~Jira first or Jira + Linear together?~~ Resolved: Jira, Linear and GitHub Issues together (PER-1 uses all three).
2. Trial allowance on platform keys: how large, and how is abuse prevented?
3. Where do status reports get published besides Slack in v1 (Confluence/Docs are phase 2)?
4. Exact trust-ladder thresholds (N runs, edit rate). Set in the AI design phase.
5. Data retention defaults for run outputs and evidence.
