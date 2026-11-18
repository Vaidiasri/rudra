# ADR-0013: Speech-to-text: ElevenLabs Scribe primary, Deepgram fallback, behind an interface

- **Status:** Accepted
- **Date:** 2026-11-19
- **Related:** [STT & consent research](../02-research/05-speech-to-text-and-consent.md) Part A, ADR-0011, ADR-0012

## Context
Client calls mix Hindi and English (Hinglish), with Indian accents and variable audio quality. No local models are possible. Independent benchmarks (Specific Labs, June 2026) report Hinglish word error rates of:

| Provider | WER |
|---|---|
| ElevenLabs Scribe v2 | 8.1% |
| Deepgram Nova-3 | 19.3% |
| Gemini 3.5 Flash | 35% |
| Groq Whisper | 75.7% |
| OpenAI gpt-4o-transcribe | 91.3% |

A second test puts Sarvam Saaras v3 (code-mix mode) on par with Scribe v2.

## Decision
- An internal **`Transcriber` interface** (multi-track input → word-timestamped, speaker-attributed transcript).
- **Primary:** ElevenLabs Scribe v2, batch, multichannel mode.
- **Fallback, and default in development:** Deepgram Nova-3 Multilingual (free credits cover development).
- **India-resident option:** Sarvam Saaras v3, for users or clients needing data processed in India.
- **Excluded for Hinglish:** Groq Whisper, OpenAI transcription. Gemini free tier is excluded entirely (it trains on data).
- **Strategy:**
  - Upload audio in 5–10 minute chunks during the call, and produce the final transcript right after it ends. Live streaming is opt-in later.
  - The mic track is always the user, so no diarization is needed for it. Diarize the remote track only when needed.
  - Merge the tracks by word timestamps and remove echo bleed.
- Vendor no-retention settings are on (ADR-0012).

## Consequences
- Transcription is a metered cost per audio hour (~$0.22–0.40/hr primary, prices to confirm). It goes into the cost-per-user NFR.
- We maintain our own Hinglish evaluation set (with consented or synthetic recordings) to re-benchmark providers.

## Revisit when
Our own eval set shows another provider winning, or pricing or terms change.
