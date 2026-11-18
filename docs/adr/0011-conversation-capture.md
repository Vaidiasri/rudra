# ADR-0011: Conversation capture: transport-level adapters, laptop + Android first

- **Status:** Accepted
- **Date:** 2026-11-19
- **Supersedes:** the "desktop app in phase 2" part of [ADR-0002](0002-mvp-surfaces-and-audience.md) (a desktop **call companion** moves into the MVP; full activity observation under [ADR-0003](0003-observation-data-minimization.md) stays in phase 2)
- **Related:** [Call-capture feasibility research](../02-research/04-call-capture-feasibility.md), [STT & consent research](../02-research/05-speech-to-text-and-consent.md), ADR-0012, ADR-0013, PRD W-1

## Context
Workflow W-1 (client call → requirements → tickets) is only automation if Rudra captures client conversations by itself. The founder's requirements:
- capture is **invisible to the client** (no bot participant, nothing posted in the meeting),
- capture is **not tied to one device**,
- calls happen on Google Meet, Microsoft Teams, WhatsApp, cellular and in person.

Research established hard OS limits:
- Android and iOS give third-party apps no access to call audio (cellular or VoIP), and Google Play bans call recorders.
- Apple's native call recording always announces itself.
- Personal WhatsApp calls have no API.
- There is no botless real-time API for Meet or Teams.
- Indian rules forbid showing the user's own number on cloud-routed outbound calls.

On laptops (Windows 10 build 20348+, macOS 14.2+), per-app call audio and the mic can be captured as separate tracks, invisibly. This is the proven Granola design.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| Meeting bot | Works across Meet/Teams on any device | Visible to the client, which violates the founder's requirement |
| Platform transcripts | No client-side software | Visible to all participants; inaccessible for client-organised Teams meetings; Meet media API ends 2027-01-15 |
| Desktop-only capture | Works fully for laptop calls | Misses phone calls entirely: a patch, not a solution |
| **Pluggable capture adapters → one conversation pipeline → coverage guard** | Device-independent architecture; each channel gets the best legitimate path; new channels are new adapters, not redesigns | Several adapters to build and maintain; some phone scenarios remain limited until hardware or business-line adapters ship |

## Decision
1. **Architecture:** a **Capture Adapter** interface. Every adapter produces a normalised *conversation recording* (audio tracks + metadata: source, device, start/end, channel layout, participants hint). All of these feed one **conversation pipeline** (transcribe → attribute speakers → link to client → analyse → tickets → verify).
2. **Capture at the transport layer, never by bypassing OS security.** We do not exploit, jailbreak, or use banned APIs.
3. **MVP adapters:**
   - **Desktop call companion** (Windows first, macOS next): detects call start/end (mic in use + known call apps/browser tabs + calendar); captures call audio and mic as separate tracks; covers Meet, Teams, Zoom, Slack huddles and WhatsApp Desktop on the laptop, and cellular calls routed to the laptop through Windows Phone Link (to be validated by a hands-on test).
   - **Android app:** in-person recording; post-call voice memo; import of the phone's built-in call recordings where the OEM provides them.
   - **Coverage guard:** detects likely client conversations that weren't captured (calendar events, Phone Link / OEM call records where available) and asks the user for a 30-second memo.
   - **Manual upload / paste,** as a last-resort adapter.
4. **Later adapters, in order:**
   - WhatsApp Business Calling API (a dedicated client-facing business number),
   - cloud business phone line (Exotel / Knowlarity),
   - Plaud partner SDK import,
   - Meet/Teams transcripts for meetings the user's own organisation runs,
   - iOS app.
5. **R&D track:** the Rudra Bluetooth bridge (headset-path capture of any call on any phone). It's the long-term answer for phone-based VoIP calls.

## Consequences
- The MVP includes a Windows desktop app (Tauri) and an Android app. Code signing (OV certificate on a cloud HSM) and Play Store compliance (no call-recording APIs; clear in-app disclosure) are MVP work.
- Phone-based WhatsApp/Meet/Teams calls are covered only via the coverage guard and memos until later adapters ship. This limit is stated honestly in onboarding.
- The pipeline must handle multi-track audio, partial captures, stitching of dropped calls, and late uploads (offline buffering).

## Revisit when
- The Phone Link test fails or succeeds.
- The WhatsApp Business Calling API proves usable for client calls.
- The Bluetooth bridge prototype works.
- An OS vendor opens a call-audio API.
