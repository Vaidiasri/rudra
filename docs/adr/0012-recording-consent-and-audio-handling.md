# ADR-0012: Recording consent model and audio handling

- **Status:** Accepted
- **Date:** 2026-11-19
- **Related:** [STT & consent research](../02-research/05-speech-to-text-and-consent.md) Part B, ADR-0011, ADR-0003

## Context
Capture is invisible to clients by design (ADR-0011). The research found:
- Covertly recording someone in an all-party-consent jurisdiction is likely unlawful: California, Florida, Washington, Illinois, Pennsylvania, Massachusetts, Maryland and others; Germany (§201 StGB, a crime); the UAE (a crime).
- Courts tend to apply the recorded party's local law.
- India is a grey area. Under the DPDP Act the user is likely the data fiduciary and Rudra the processor.
- Invisible note-takers face active litigation in 2026: Otter.ai (wiretap/CIPA claims proceeding, tied to training on recordings), *Chamberlain v. Granola* (covertness marketing cited), and Fireflies (BIPA voiceprints).

## Decision
1. **Consent once per client, outside the call.**
   - Rudra provides a contract-clause template ("calls may be transcribed using AI tools for accuracy and record-keeping") for the user's SOW/MSA.
   - Each client in Rudra has a **consent record**: status (`consented`, `not_consented`, `unknown`), basis, date, and an optional document reference.
   - Rudra stays invisible in calls; the lawful basis comes from the contract.
2. **Capture gating by consent and jurisdiction.**
   - `consented`: full capture (both tracks).
   - `not_consented` or `unknown` with a client in an all-party-consent jurisdiction: **notes-only mode**, which captures only the user's own mic plus voice memos.
   - Elsewhere, with `unknown` consent: a one-time prompt asks the user to confirm consent before full capture.
   - Per-client **never transcribe** overrides everything.
3. **Audio handling.**
   - Audio is **deleted after successful transcription** (bounded retry window, then deleted regardless).
   - Transcripts and derived notes have a configurable retention period, with a default to be set in the data design.
4. **No training, no biometrics.**
   - Customer audio and transcripts are never used to train models, ours or vendors'. Vendor no-retention / zero-data-retention options are enabled.
   - No voiceprints or speaker-identification biometrics are stored.
5. **Positioning.** We never market covertness. The promise is completeness and accuracy ("never miss a requirement").
6. **Terms.** The ToS and DPA place the consent obligation on the user, define Rudra as processor, and document the safeguards above.
7. **Legal review** of the clause template, jurisdiction list and DPDP position happens before public launch. The open questions are listed in the research doc.

## Consequences
- The data model needs clients, consent records and jurisdiction.
- Capture adapters must support a mic-only mode.
- Onboarding includes per-client consent setup.

## Revisit when
Legal review completes, or litigation outcomes (Otter, Granola) change the landscape.
