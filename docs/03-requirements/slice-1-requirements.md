# Slice 1 requirements: "Never miss a requirement"

- **Status:** Approved (2026-11-20)
- **Last updated:** 2026-11-20
- **Scope:** [PRD](../01-product/prd.md) §12 slice 1 · [W-1 design](../01-product/workflows/w1-client-conversations.md) stage 1 + sign-off
- **Sub-slices:** **1a** = upload/paste + intelligence + board (validates analysis quality) · **1b** = Windows desktop call companion (automatic capture)

Requirement IDs are stable; later slices append new IDs. Each FR traces to a PRD feature (F-xxx); each NFR is measurable.

---

## 1. Functional requirements

### 1.1 Account & workspace
| ID | Requirement | Trace |
|---|---|---|
| FR-001 | User signs in with Google or GitHub. | F-013 |
| FR-002 | Each user has a personal workspace (a tenant of one). All data is scoped to it. | ADR-0002 |
| FR-003 | User can export all their data (JSON + Markdown) and delete their account and all data. | ADR-0012 |

### 1.2 Clients & consent
| ID | Requirement | Trace |
|---|---|---|
| FR-010 | User can create, edit and archive clients: name, importance tier (High/Normal/Low), time zone, jurisdiction (country/state), contacts (names, emails, phone numbers), notes. | F-034 |
| FR-011 | Each client has a consent record: `consented` / `not_consented` / `unknown`, basis (e.g. contract clause), date, optional document reference. | ADR-0012 |
| FR-012 | Rudra provides a consent-clause template the user can copy into contracts. | ADR-0012 |
| FR-013 | Each client has a **never transcribe** flag that blocks all processing of that client's conversations. | ADR-0012 |
| FR-014 | Processing is gated: `consented` → full; all-party-consent jurisdiction without consent → **notes-only** (user's own speech + memos only); `unknown` elsewhere → one-time confirmation prompt before full processing. | ADR-0012 |

### 1.3 Conversation input (slice 1a)
| ID | Requirement | Trace |
|---|---|---|
| FR-020 | User can upload an audio recording (mp3, m4a, wav, ogg/opus, webm) up to 3 hours / 500 MB. | F-030 |
| FR-021 | User can paste a transcript or written notes instead of audio. | F-030 |
| FR-022 | User assigns the conversation to a client. Rudra suggests the client from names or terms mentioned when possible. | F-034 |
| FR-023 | User can record a short voice memo in the web app (browser mic) and treat it as a conversation. | F-030 |

### 1.4 Transcription
| ID | Requirement | Trace |
|---|---|---|
| FR-030 | Audio is transcribed through the `Transcriber` interface (ADR-0013) with word-level timestamps. | F-035 |
| FR-031 | Speakers are separated: by track when multi-track (user vs others), otherwise by diarization. The user can rename or correct speakers. | F-035 |
| FR-032 | Hindi/English code-mixed speech is supported. Analysis output is English; citations keep the original-language quote plus an English rendering. | ADR-0013 |
| FR-033 | Secrets and sensitive personal data (keys, passwords, card numbers, government IDs) are redacted from transcripts before any analysis. | ADR-0012 |
| FR-034 | Audio is deleted after successful transcription, and in any case within 24 hours of upload. | ADR-0012 |

### 1.5 Conversation intelligence
| ID | Requirement | Trace |
|---|---|---|
| FR-040 | Rudra extracts items of 8 types (requirement, change, bug, question, decision, constraint, risk, action). Each item has **at least one citation** (timestamp range + quote). | F-035, W-1 §3 |
| FR-041 | New items are merged into the client board: duplicates are linked, **changes** create a new version, and **contradictions** with earlier conversations are flagged with both citations. | F-038 |
| FR-042 | Each item is rated: complexity (XS–XL), effort per phase (dev/test/deploy, XS–XL), skills & tech tags, confidence, assumptions. | F-038 |
| FR-043 | Rudra lists **blind spots** (topics not discussed that should have been), each linked to the items it affects. | F-035 |
| FR-044 | Rudra lists **clarification questions**, each linked to items. | F-035 |
| FR-045 | Rudra extracts deadlines and commitments (by either side) with citations. | F-035 |
| FR-046 | Rudra produces a short conversation summary (what was discussed, decided, open). | F-035 |
| FR-047 | Notes-only conversations (FR-014) are analysed from the user's speech only, and the board shows that client speech was not processed. | ADR-0012 |

### 1.6 Verification
| ID | Requirement | Trace |
|---|---|---|
| FR-050 | Every analysis run is checked against the W-1 stage-1 success contract (citations exist in the transcript; coverage of client-ask segments; changes/contradictions link both sources; every item rated or explicitly "unknown"). | F-001, ADR-0007 |
| FR-051 | Qualitative checks (rating plausibility, blind-spot relevance) use an LLM judge from a **different model family** than the analyser, as a secondary signal. | ADR-0007 |
| FR-052 | A failed check triggers a retry with the failure as feedback (max 2). Then the run is marked **needs attention**, with the failed checks shown. | F-003 |
| FR-053 | Each conversation shows a **verification report:** every check, its result and the evidence. | F-002 |

### 1.7 Client board
| ID | Requirement | Trace |
|---|---|---|
| FR-060 | Board per client lists all items with filters (type, status, size, skills, deadline) and sorting. | F-038 |
| FR-061 | Item detail shows description, ratings, citations (quote + timestamp + conversation link), history, and linked questions and blind spots. | F-038 |
| FR-062 | User can edit, merge, split, change status and delete items. Every change is recorded in history. | F-038 |
| FR-063 | User can mark items **wrong** or add **missed items**. This feedback is stored as evaluation data (used for the slice-1 exit metric). | Slice-1 exit |
| FR-064 | Board shows conversation timeline per client (all conversations, with summaries). | F-038 |

### 1.8 Client sign-off summary
| ID | Requirement | Trace |
|---|---|---|
| FR-070 | User can generate a client-facing summary for selected items (requirements, decisions, open questions, deadlines) with no Rudra branding by default. | F-040 |
| FR-071 | Summary can be copied (text/Markdown) or exported as PDF. Rudra never sends it. | F-040 |
| FR-072 | User records the client's response: confirmed / corrections (corrections become board changes with a citation to the response). | F-040 |

### 1.9 Notifications (slice 1)
| ID | Requirement | Trace |
|---|---|---|
| FR-080 | When a conversation finishes processing, the user is notified in the web app and (optionally) by email, with counts: new items, changes, contradictions, questions. | F-010 |

### 1.10 Desktop call companion (slice 1b)
| ID | Requirement | Trace |
|---|---|---|
| FR-090 | Windows app detects call start/end for Google Meet (browser), Microsoft Teams, Zoom, Slack huddles and WhatsApp Desktop, and cellular calls via Phone Link (validated by test). | F-031, ADR-0011 |
| FR-091 | Captures call audio and the user's mic as **separate tracks**, invisible to other participants. | F-031 |
| FR-092 | On call start, identifies the client (from window title, contact name, or by asking the user via a private toast) and applies consent gating (FR-014) before capturing. | F-034 |
| FR-093 | Uploads audio in encrypted chunks during the call; buffers encrypted locally when offline and uploads later. | ADR-0011 |
| FR-094 | Tray app shows capture status, with pause/resume, "not a client call", and per-client never-transcribe. | F-031 |
| FR-095 | Chunks are transcribed and pre-analysed **during** the call (incremental analysis) so the board is ready shortly after the call ends. | NFR-001 |
| FR-096 | Signed installer and signed auto-update. | ADR-0011 |

---

## 2. Non-functional requirements

| ID | Category | Requirement (measurable) |
|---|---|---|
| NFR-001 | Latency | Desktop-captured calls ≤ 60 min: board updated ≤ **5 min** after call end (p90). Uploaded recordings: ≤ 15 min for a 60-min recording (p90). Dev on free tiers: measured, not gated. |
| NFR-002 | Analysis quality | On the founder's evaluation set (≥ 10 real calls): **item recall ≥ 90%** vs the founder's own list; **0 uncited items**; false-item rate ≤ 5%; blind spots/questions rated useful on ≥ 70% of calls. |
| NFR-003 | Transcription quality | Hinglish WER measured on our own eval set for each provider. Target ≤ 20% with the dev provider, ≤ 10% with the production provider. Item recall (NFR-002) is the gating metric. |
| NFR-004 | Privacy | Audio deleted ≤ 24 h after upload (verified by a daily job). Transcripts deleted after **90 days** (configurable). Vendor no-retention / no-training options on. No customer data used for training. Real client data never sent to providers whose terms allow training on it (ADR-0009). |
| NFR-005 | Security | TLS 1.2+ in transit; encryption at rest; tenant isolation enforced in DB (RLS) and app; secrets in a managed vault; target OWASP ASVS Level 2 for the web app; dependency scanning and hash-pinned lockfiles (ADR-0005). |
| NFR-006 | Availability | Slice 1 (dogfood): best effort, no data loss (uploads are durable before acknowledgement). Production target set before beta (proposed 99.5%). |
| NFR-007 | Cost | Dev: **₹0/month** (Deepgram credits for STT; Groq free tier for LLM, which doesn't train on data). Production target: AI cost ≤ **$5 per active user per month** at 10 call-hours/month, to be validated with a cost model in the HLD. |
| NFR-008 | Desktop footprint | On an 8 GB / 2-core i3 machine: ≤ 150 MB RAM, ≤ 2% CPU idle, ≤ 10% CPU while capturing. |
| NFR-009 | Capture reliability | ≥ 95% of the founder's laptop client calls captured end-to-end over 2 weeks (slice-1b exit). |
| NFR-010 | Observability | Every analysis run traced (inputs, model calls, checks, cost, latency) and viewable; cost per conversation recorded. |
| NFR-011 | Rate-limit resilience | Provider rate limits (429) never lose work: queued with backoff; the user sees "processing delayed" rather than a failure. |
| NFR-012 | Browser support | Latest two versions of Chrome and Edge (desktop). |
| NFR-013 | Data portability | Export (FR-003) completes within 10 minutes for a workspace with 1 year of data. |

---

## 3. Out of scope for slice 1
Tracker connectors and ticket creation (slices 2–3), daily plan, Slack, follow-ups, Android, planning session, trust ladder, billing, multi-user teams.

## 4. Open questions
1. ~~Free-tier rate limits in dev~~ **Resolved:** slower processing in dev is accepted.
2. ~~Evaluation-set consent~~ **Resolved:** only calls with consented clients; eval transcripts follow the same 90-day rule unless the client agrees otherwise.
