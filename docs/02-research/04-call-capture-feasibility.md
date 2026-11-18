# 04: Call-Capture Feasibility by Channel and Device

**Date:** 2026-11-18. **Method:** live web search plus official platform docs (Microsoft Learn, Apple Developer, Android Developers/AOSP, Google Play Console Help, Google Workspace, Meta for Developers, Espressif, vendor developer docs). Anything not confirmed against a primary or official source is marked **[UNVERIFIED]**. Prices change often, so re-check them before committing.

> Consent and recording law (India and client jurisdictions) is covered in `05-speech-to-text-and-consent.md`. This document covers only what is **technically** possible. "Invisible to client" here means *no bot, no in-meeting notice, no audible announcement*. It does **not** mean that recording without consent is lawful.

---

## Executive summary

1. **The laptop is the only device where botless, automatic, invisible capture works for every VoIP channel.** On Windows, WASAPI loopback (system mix) and process loopback (per-app, Windows 10 build 20348+) work alongside a separate mic stream. On macOS, Core Audio taps (14.2+) and ScreenCaptureKit (13+, mic from 15+) do the same. Granola already ships this design on both OSes. Meet, Teams, Zoom, Slack and WhatsApp Desktop are all covered **when the call is taken on the laptop**.
2. **Phones are effectively closed to third-party call capture.**
   - **Android:** `VOICE_CALL`/`VOICE_UPLINK`/`VOICE_DOWNLINK` need `CAPTURE_AUDIO_OUTPUT`, which is "not available to third-party applications". AudioPlaybackCapture only captures `USAGE_MEDIA`/`GAME`/`UNKNOWN`, so VoIP voice is excluded. During a cellular or VoIP call, an ordinary app's mic capture is silenced. Play policy lists "call recorder" as an invalid use of call-log permissions, and the Accessibility API is banned for call recording.
   - **iOS:** Apple DTS says there is "no public API" to capture the mic during a cellular call.
3. **Phone workarounds, each with a cost:**
   - **(a) Import OEM-recorder files** (Android only). Samsung and Xiaomi in India can record without an announcement [partly UNVERIFIED]. The Google Phone app always announces. Not real-time.
   - **(b) Cloud telephony** via an Indian virtual number (Exotel/Knowlarity/Plivo-India). The client sees the **virtual number, not the developer's own number**, on outbound calls, and Indian rules forbid caller-ID spoofing. Twilio blocks India-to-India calls.
   - **(c) A clip-on hardware recorder** (Plaud Note Pro, ₹20,990 in India) that picks up call audio through vibration. Works for any app, not automatic per call, and has a partner SDK that returns audio.
   - **(d) A Bluetooth "headset bridge"** (Q9). Technically the most universal phone path: it gets both directions of any call the phone routes to a headset, with no OS sandbox involved. But it is custom hardware, ESP32 Classic BT can't also relay to wireless earbuds, and it is Rudra-built.
4. **Native platform recordings are not invisible.**
   - iPhone call recording announces to both parties and repeats periodically.
   - Google Meet transcripts show an icon to everyone, and only the host's domain can start them.
   - Teams shows "being transcribed" to all, and a person from another tenant cannot view or download the transcript afterwards.
   - The Meet Media API (real-time) is being shut down on 2027-01-15.
5. **WhatsApp personal calls cannot be captured by any third party.** The only route is the WhatsApp Business Calling API on a **separate API-only business number**. The client must call that number. Media terminates at Rudra's WebRTC/SIP endpoint, so audio is fully available. User-initiated calls are free.
6. **In-person meetings are easy.** Any phone or laptop mic works, with a foreground service on Android and an active audio session on iOS. Quality is good at a table and worse across a room.
7. **A coverage guard is feasible on each platform:**
   - Windows: audio-session enumeration plus process names.
   - macOS: Core Audio process objects plus `IsRunningSomewhere`.
   - Android: `READ_PHONE_STATE`, or call-screening role or call log (policy permitting).
   - iOS: CallKit `CXCallObserver` [UNVERIFIED that it is usable for this purpose under App Review].

   On iPhone, missed-call **detection** is possible but **capture** is not.

---

## Coverage matrix

Legend: **Invisible** = no bot, no in-call notice or announcement to the client. **Auto** = starts without the developer tapping anything per call. Feasibility: **works / limited / impossible**.

| Channel × device | Capture method | Invisible? | Auto? | Feasibility | Cost | Notes |
|---|---|---|---|---|---|---|
| Google Meet on Windows laptop (browser) | WASAPI loopback (system) or process loopback (browser PID) + mic stream | Y | Y | **works** | free (STT only) | Process loopback isolates the browser tree; Granola uses the system mix. |
| Google Meet on Mac | Core Audio tap (14.2+) or ScreenCaptureKit audio + mic | Y | Y | **works** | free | Needs the "system audio recording" TCC permission. |
| Teams on Windows laptop (desktop app) | Process loopback on `ms-teams.exe` + mic | Y | Y | **works** | free | Disable "Audio enhancements" if capture is silent (Granola guidance). |
| Teams / Meet / Zoom / Slack huddle on Mac | Core Audio tap per process + mic | Y | Y | **works** | free | |
| WhatsApp Desktop call on Windows/Mac | Loopback/tap + mic | Y | Y | **works** | free | Only if the call is taken on the laptop. |
| Meet / Teams via native transcript (any device) | Meet REST API / Graph | **N** (icon or banner to all) | Y for own org | **limited** | Workspace Business Standard+ / M365 | External-tenant transcripts are not accessible (Teams). Host's domain must start it (Meet). |
| Meet on iPad / iPhone | none (mic blocked during the call); Bluetooth bridge (Q9) | Y (bridge) | N | **impossible** natively / limited via bridge | bridge HW | iOS gives no third-party mic or loopback during a call. |
| Teams / Meet on Android phone or tablet | AudioPlaybackCapture excludes voice; mic silenced; Bluetooth bridge | Y (bridge) | N | **impossible** natively / limited via bridge | bridge HW | |
| WhatsApp call on Android | none in software; Plaud call mode; Bluetooth bridge; ambient 2nd device on speaker | Y | N | **limited** | ₹17–21k (Plaud) / bridge | No API or file for personal WhatsApp calls. Samsung's recorder doesn't record VoIP [UNVERIFIED]. |
| WhatsApp call on iPhone | Plaud call mode; Bluetooth bridge; ambient 2nd device | Y | N | **limited** | as above | |
| WhatsApp call to a Business API number (any device for the client) | WhatsApp Business Calling API (WebRTC/SIP) answered in the Rudra app | Y (no notice by API) | Y | **works** (if the client calls that number) | inbound free; outbound per-minute | Separate number, not the dev's personal WhatsApp. Not on coexistence numbers [UNVERIFIED]. |
| Cellular call on Android (Samsung/Xiaomi, India) | Import native recorder files (`READ_MEDIA_AUDIO` / SAF) after the call | Y on Samsung/Xiaomi India [UNVERIFIED]; **N** on Google Phone app | Y (auto-record) | **limited** | free | Post-call only. OEM paths vary. Google Dialer phones (Pixel, Motorola, OnePlus since ~2021) announce. |
| Cellular call on Android, other OEMs | Cloud telephony; Plaud; Bluetooth bridge; Phone Link on PC | varies | varies | **limited** | see Q4 | |
| Cellular call on iPhone | Apple call recording (announces); cloud telephony; TapeACall-style merge; Plaud; Bluetooth bridge; Phone Link on PC | Apple: **N**; others Y/partly | N (except telephony inbound) | **limited** | | No third-party access to Apple's recordings [UNVERIFIED: no API exists]. |
| Cellular call via cloud number (forward / virtual number) | Exotel / Knowlarity / Plivo-India bridge with `Record=true` | Inbound Y; outbound: client sees the virtual number | Y | **limited** | ~₹0.4–1.2/min per leg + number rental [UNVERIFIED ranges] | Dev's own number is kept only for **inbound** (via forwarding). |
| Cellular call relayed to Windows PC via Phone Link (Android or iPhone) | PC acts as Bluetooth HFP hands-free; capture PC render via loopback + PC mic | Y | Y (if the call is answered on the PC) | **limited** | free | Narrowband/wideband HFP. Not relayable to a headset. Echo, so a wired headset is advised. VoIP not reliably relayed. |
| Any call on any phone via Rudra Bluetooth bridge | Device pairs as HFP headset; captures SCO Rx + its own mic Tx | Y | Y (per call routed to it) | **limited** (prototype-grade) | ~US$15–40 BOM [UNVERIFIED] | 8/16 kHz mono. Dev must use the bridge's earpiece or wired earphones. Dual-role relay to BT earbuds isn't supported on ESP32. |
| Any call, phone on speaker, 2nd device records the room | Laptop/phone mic ambient | Y | Partly (coverage guard can prompt) | **limited** | free | Reverb and low far-end level. One mixed channel. Worse diarization. |
| In-person meeting | Phone or laptop mic; Plaud NotePin/Note | Y | Partly (calendar-triggered) | **works** | free / device | Android foreground service `microphone` type. iOS orange dot (user only). |

---

## Q1. Windows and macOS laptop companion

### Capture APIs

| Need | Windows | macOS |
|---|---|---|
| System/call audio | **WASAPI loopback**: open a shared-mode capture stream on a *render* endpoint with `AUDCLNT_STREAMFLAGS_LOOPBACK`. Captures the full mix. Works without a "Stereo Mix" device. Event-driven since Win10 1703. DRM-protected streams are excluded. | **ScreenCaptureKit** `SCStreamConfiguration.capturesAudio` (macOS 13+); `captureMicrophone` (**macOS 15.0+**). **Core Audio taps**: `CATapDescription` + `AudioHardwareCreateProcessTap`, added to an aggregate device. Sample requires **macOS 14.2+**. |
| Per-app isolation | **Process loopback**: `AUDIOCLIENT_PROCESS_LOOPBACK_PARAMS` (include or exclude a PID **and its child process tree**) via `ActivateAudioInterfaceAsync`. Min client **Windows 10 build 20348**, so in practice Windows 11. | Taps can target specific process objects or groups, can be private, and can mute the process output. |
| Mic as a separate stream | Normal WASAPI capture on the mic endpoint. Shared mode allows concurrent capture with Teams/Meet. | AVAudioEngine / Core Audio input, or SCK `captureMicrophone`. |
| Permission | No consent prompt for loopback. Mic follows the Settings > Privacy > Microphone toggle. | TCC prompt. `NSAudioCaptureUsageDescription` for taps ("system audio recording" permission). Screen & System Audio Recording for SCK. Mic permission. |

**Design implication:** record **two tracks**: (1) remote = loopback of the call app's process tree, (2) local = mic. That gives free speaker separation (developer vs. client side) before diarization. Echo: if the user is on laptop speakers, the client voice leaks into the mic track, though the meeting app's AEC only cleans *its* send path. Recommend headphones, or run AEC using the loopback track as reference [standard practice; not platform-documented].

### Call-start detection (coverage guard)

- **Windows**
  - `IAudioSessionManager2` lets an app "enumerate sessions on the audio device" and `RegisterSessionNotification` on session creation (Windows 7+). Activate it on the **capture** endpoint to see which processes have open mic sessions, and on the render endpoint for playback sessions. Map PID to exe name (`ms-teams.exe`, `Zoom.exe`, `slack.exe`, `WhatsApp.exe`, `chrome.exe`/`msedge.exe`) [applying it to capture endpoints is standard practice; the doc example uses render].
  - The `CapabilityAccessManager\ConsentStore\microphone` registry keys (`LastUsedTimeStart/Stop`) are a **history**, not a reliable live signal. A user reports no update when Teams calls start [UNVERIFIED]. Use them only as a fallback.
  - For Meet in a browser, a browser mic session plus the active window/tab title containing "Meet" is a heuristic. Tab URLs need a browser extension [UNVERIFIED approach].
- **macOS:**
  - `kAudioDevicePropertyDeviceIsRunningSomewhere` gives a device-level "in use" signal. It doesn't separate input from output and must be checked per device.
  - Newer Core Audio **process objects** expose `kAudioProcessPropertyIsRunningInput`, `…BundleID` and `…PID` (documented symbols; min OS version not shown in the docs; believed macOS 14.2+ [UNVERIFIED]). These map mic use to an app bundle (`com.microsoft.teams2`, `us.zoom.xos`, `com.tinyspeck.slackmacgap`, `net.whatsapp.WhatsApp`, browsers).
- **Calendar join** (Google/Microsoft calendar events with Meet/Teams links) is a strong second signal for the coverage guard.

### OS indicators (visible to the user only, never to the client)

- **Windows 11:** mic-in-use icon in the taskbar, and Settings > Privacy > Microphone lists apps using it. Loopback shows **no** indicator [UNVERIFIED that none appears].
- **macOS:** orange mic dot and the Control Center app list (13.3+). Screen/system-audio capture shows the purple recording indicator. macOS 15 periodically re-prompts for screen-recording permission [UNVERIFIED cadence]. Core Audio taps avoid the screen-recording entitlement [UNVERIFIED].
- None of these signals reach meeting participants. Meet, Teams and Zoom do not detect OS-level loopback [no documented detection mechanism].

### What Granola does (official docs)

- "On desktop, Granola captures audio from your device, so no bot joins the call." It "cannot isolate audio from individual applications — it captures the combined audio stream" (system output + mic). It does "not record or save audio" and streams to a cloud STT provider. Speaker tags exist for Meet/Zoom/Teams on macOS and Windows.
- **macOS permissions:** Microphone + "Screen & System Audio Recording".
- **Mobile:** the iPhone app "cannot capture audio from virtual meeting apps". Phone calls are only captured via **Granola's in-app dialer** (outbound, one-to-one). Granola's own blog says this is because "Apple doesn't allow third-party apps to access the microphone during cellular calls". The Android app (launched 2026-07-01) is for in-person only. Inbound calls: "neither platform offers a clean third-party solution."

### Rust / Tauri libraries

| Crate | Status (crates.io, Nov 2026) | Use |
|---|---|---|
| `wasapi` (HEnquist/wasapi-rs) | 0.27.0, updated 2026-11-06 | README lists "Loopback capture" and a `record_application` example ("records audio from a single application") = process loopback. |
| `cpal` | 0.18.2, 2026-08-16 | Cross-platform mic capture. Loopback support on Windows is limited [UNVERIFIED for 0.18]. |
| `screencapturekit` (doom-fish/screencapturekit-rs) | 11.1.2, 2026-11-02 | SCK screen + audio capture on macOS. |
| Core Audio taps | No mature dedicated crate found [UNVERIFIED]. Use `objc2-core-audio` / `coreaudio-sys` FFI or a small Swift sidecar. | |
| `windows` (microsoft/windows-rs) | official | `IAudioSessionManager2`, `ActivateAudioInterfaceAsync`, process enumeration. |

---

## Q2. Android

### Capturing call audio: **not possible for a Play-distributed app**

- **VOICE_CALL / VOICE_UPLINK / VOICE_DOWNLINK** "requires the `CAPTURE_AUDIO_OUTPUT` permission. This permission is reserved for use by system components and is not available to third-party applications." `REMOTE_SUBMIX` is the same.
- **AudioPlaybackCapture** (Android 10+, MediaProjection consent prompt) only captures players whose usage is `USAGE_MEDIA`, `USAGE_GAME` or `USAGE_UNKNOWN`, with capture policy `ALLOW_CAPTURE_BY_ALL`. VoIP apps play call audio as `USAGE_VOICE_COMMUNICATION`, so it is **not capturable**. WhatsApp, Meet and Teams voice are excluded. Meeting apps may also set `ALLOW_CAPTURE_BY_NONE/SYSTEM` [UNVERIFIED per app].
- **Mic during a call** ("Voice call + ordinary app", Android 10+ sharing rules): when `AudioManager.getMode()` is `MODE_IN_CALL` (cellular) **or `MODE_IN_COMMUNICATION` (VoIP)**, "the call always receives audio". Another app can capture only if it is an **accessibility service** or a privileged pre-installed app with `CAPTURE_AUDIO_OUTPUT`. Ordinary apps get silence (`isClientSilenced()`). `VOICE_COMMUNICATION` is privacy-sensitive by default.
- **Accessibility loophole is closed by policy.** Google Play's 2022 policy says "The Accessibility API is not designed and cannot be requested for remote call audio recording" (widely reported; the current Accessibility policy page fetched today doesn't repeat that sentence, but the call-log policy below still bans call recorders).
- **Call-log policy** (Play Console Help, fetched 2026-11-18): `READ_CALL_LOG` is allowed for the default Phone/Assistant handler, or by exception. Exceptions include **"Enterprise CRM and business use"**, caller ID/spam, backup and cross-device sync. **"Call recorder" is listed as an invalid use case.** Account verification via phone call stops being permitted on **2027-01-27**.

### Importing OEM recorder files

| OEM dialer (India) | Native recording | Announces to other party? | Source quality |
|---|---|---|---|
| **Google Phone app** (Pixel 6+/Android 14+, and Google-Dialer OEMs such as Motorola, Nokia, recent OnePlus) | Yes, in supported regions; auto-record available in India per community reports | **Yes.** "Both parties are notified with a disclosure", again on stop. Cannot be disabled. | Official help (support.google.com/phoneapp/answer/9803950) |
| **Samsung One UI** | Yes, auto-record all, unsaved or specific numbers. "India has a built-in recorder on all models." | Conflicting. Beebom says Samsung India records **without** an announcement. Salestrail says the other party hears a notice "in some regions". | [UNVERIFIED for India; test on device] |
| **Xiaomi / Redmi / POCO (HyperOS, own dialer)** | Yes on many India models | Reported no announcement on own dialer | [UNVERIFIED] |
| **OnePlus** | Mixed: Google Dialer since ~2021 (announces); recorder availability changed across OxygenOS versions | Google Dialer: yes | [UNVERIFIED] |

**Access:** recordings are audio files in shared storage (`Recordings/` is indexed into `MediaStore.Audio` on Android 12+). A third-party app can read other apps' audio files with the runtime **`READ_MEDIA_AUDIO`** permission. Play's 2025 photo/video permission restrictions do **not** cover audio. Folder names differ by OEM and version [UNVERIFIED paths]. Alternatively, the user picks the folder once via SAF `ACTION_OPEN_DOCUMENT_TREE`. Salestrail (a CRM) ships exactly this "use built-in recorder" pattern on Samsung One UI 8. Files carry contact name/number and timestamp in the filename (Samsung) [UNVERIFIED format], which lets the coverage guard match them to calls.

### Detecting that a call happened

- **`READ_PHONE_STATE`** + `TelephonyCallback.CallStateListener`: call state (idle/ringing/off-hook). This is a dangerous runtime permission, **not** a Play-restricted SMS/Call-Log permission.
- **`READ_CALL_LOG`**: numbers and durations. Play-restricted. A plausible declaration is "Enterprise CRM and business use", but approval is not guaranteed.
- **`CallScreeningService`** (call-screening role) receives incoming call details without the call log [standard Android API; role-holder UX applies].
- **VoIP calls:** `AudioManager.getMode() == MODE_IN_COMMUNICATION` plus `getActiveRecordingConfigurations()` show that *some* app holds a communication session. The app identity is not exposed to ordinary apps [UNVERIFIED].

---

## Q3. iOS and iPadOS

- **Third-party mic during a cellular call: no.** Apple DTS (forum thread 856047): "There is no public API that allows a third-party app to capture microphone input (AVAudioEngine, AVAudioRecorder, SFSpeechRecognizer, or AudioUnit input) while a regular cellular call is active." Granola cites the same restriction.
- **During a VoIP/FaceTime call:** developers report `setActive` failing (`OSStatus 561017449`, "insufficient priority") and recordings being interrupted. There is no system-audio or loopback API on iOS (ReplayKit broadcast excludes call audio [UNVERIFIED]). Treat it as **not possible**.
- **Apple's native call recording** (iOS 18.1+; iPadOS 26 and macOS Tahoe Phone app):
  - "Both participants receive an audio notice that the call is being recorded." "The audio notice replays periodically during the call."
  - Recordings and transcripts are saved to the **Notes** app ("Call Recordings" folder). Transcripts are available in select languages and regions.
  - "Available only in selected regions and languages", and only the initiator needs it. India availability: [UNVERIFIED]; believed available.
  - **No API** gives a third-party app access to the Notes recordings or transcripts [UNVERIFIED: no such API found]. The user can manually share or export the audio from Notes to Rudra via the share sheet [UNVERIFIED for audio export].
  - **Not invisible.**
- **TapeACall-style conference merge:** the user calls the recording line, taps *Add Call*, then *Merge*. The service "records the conversation as one of the three parties". It depends on carrier three-way calling. A third-party table says Jio and Airtel support it [UNVERIFIED]. Some carriers insert beeps or hold music the other party can hear. TapeACall's India access-number support: **[UNVERIFIED, not found]**. Under India's media-anchoring rule, a domestic recording line must be an Indian provider number (see Q4). This works manually, isn't automatic, and is semi-invisible.
- **Coverage guard on iOS:** `CXCallObserver` (CallKit) reports call start/end for cellular and CallKit-VoIP calls (WhatsApp uses CallKit) without content [UNVERIFIED: no direct doc fetched; well-known API]. It doesn't run reliably in the background without a reason to be running [UNVERIFIED].

---

## Q4. Cloud telephony for regular phone calls (India)

**Architectures**

1. **Inbound via forwarding.** The developer sets unconditional call forwarding (CFU) on their own SIM to a virtual number (ExoPhone, Knowlarity or Plivo-India number). The provider records and bridges to a **second** endpoint: a second SIM, the provider's agent app, or a SIP/WebRTC client in the Rudra app. The developer's number stays published. Busy/no-answer forwarding doesn't help, because the answered call stays on the handset.
2. **Outbound via click-to-call.** The provider calls the developer first, then the client. **The client sees the ExoPhone/virtual number**, and Exotel says "CallerId must be an ExoPhone". Exotel and Plivo say presenting an arbitrary mobile number violates TRAI rules. Custom caller ID is limited to authorised numbers on Exotel Growth/Enterprise plans.
3. **Conference-merge to a recording number** (TapeACall pattern): manual, not automatic.

**Rules and constraints**

- **USSD forwarding suspended:** DoT order of 28 Mar 2024 told all licensees to discontinue USSD-based call forwarding (`*401#`) from 15 Apr 2024 "till further notice" (fraud). Forwarding must be set via other operator channels: GSM MMI codes in the dialer, the operator app, or customer care. Current state per operator: [UNVERIFIED].
- **Media anchoring:** Plivo: "both legs of a call must originate and terminate within India", or the call fails with `violates_media_anchoring`. Only India-registered businesses (KYC) can rent Indian numbers and make domestic calls. Rudra (or each developer) needs Indian KYC.
- **Twilio:** domestic India calling is not supported (calls +91→+91 have failed since 1 Aug 2024 per Twilio's guidelines). Twilio is unusable here.
- **OSP/DoT 2021 guidelines** relaxed registration, but cloud EPABX outside India and PSTN–internet interconnection remain constrained. Use a licensed Indian provider rather than building a SIP trunk [law-firm commentary; text not re-verified].
- **Telemarketing/UCC** (TCCCPR) applies to *commercial* calls. Developer-client project calls are transactional/service calls [UNVERIFIED classification].

**Providers and cost (indicative)**

| Provider | India domestic | Recording | Indicative cost |
|---|---|---|---|
| **Exotel** | Yes (ExoPhones on licensed operators) | `Record=true` returns `RecordingUrl`; "no extra charge" | Per-pulse, sales-led. Two legs billed per bridged call. Number rental + activation [rates not published] |
| **Knowlarity** | Yes | Included; 3–12-month retention by plan | Plans ₹16,800–60,000/yr incl. 1,200–10,000 min/month; overage 20–30 paise per 30 s (≈₹0.4–0.6/min per leg) [third-party listings, UNVERIFIED] |
| **Plivo (India region account)** | Yes for India-registered businesses | Yes | Not published on the India page [UNVERIFIED] |
| **Twilio** | **No** (domestic blocked) | n/a | n/a |

**Realistic all-in cost:** about ₹1–2.5 per conversation-minute (two legs + carrier forwarding charges + STT), plus about ₹500–1,500/month per number [UNVERIFIED estimate].

**Can the client detect it?**
- **Inbound forwarded:** the client dials the usual number and hears ringback, possibly with a slightly longer setup. No announcement unless configured. Effectively invisible.
- **Outbound:** the client sees an unfamiliar virtual number, which is **visible**.
- **Merge:** possible beeps.

---

## Q5. WhatsApp

- **Personal WhatsApp calls (consumer or Business *app*):** end-to-end encrypted, no API. On Android the audio is `USAGE_VOICE_COMMUNICATION` (not capturable) and the mic is silenced for other apps. On iOS the mic is unavailable. **No third-party capture is possible.** Only physical routes work: Plaud call mode, the Bluetooth bridge (Q9), ambient speakerphone, or the desktop app with loopback when the call is on a laptop.
- **WhatsApp Business Calling API** (Cloud API; Meta docs):
  - Voice VoIP calls. Video and screen share are "planned/in development".
  - **User-initiated** calls are available wherever Cloud API is. **Business-initiated** calls need an approved call permission from the user. They are excluded only in US, Canada, Egypt, Vietnam and Nigeria, so **India is allowed** (Exotel, Bird and Infobip list India).
  - **Media:** Graph API + webhooks with **WebRTC** (ICE/DTLS/SRTP), or **SIP** (TLS) with WebRTC or SDES-SRTP. Opus/PCMA/PCMU. With SIP "the SIP server handles the media". Media terminates at the business's endpoint, so **the business has the raw call audio**.
  - **Requirements:** the number must be on Cloud API (**not the WhatsApp Business app**), messaging limit ≥ 2,000 unique recipients/day (sandbox exempt), calling enabled on the number. Business-initiated limit: 100 calls/user/day. Calling is reportedly **not available on coexistence (app + API) numbers** [UNVERIFIED, third-party].
  - **Pricing:** "All user-initiated calls are free." Business-initiated calls are billed per 6-second pulse by country and volume tier. The INR rate card wasn't readable. One source cites about ₹0.50/min to India [UNVERIFIED]. BSP markup is about ₹0.40–0.60/min [UNVERIFIED, single vendor].
- **Realistic for a developer?** Partly. The developer would publish a separate "business" WhatsApp number (API-only, so they can't also use it in their phone's WhatsApp app). Calls are answered in Rudra's app or web client, and Rudra receives the audio directly. **Invisible:** the API itself adds no notice. **The catch:** existing clients already call the developer's personal WhatsApp, so the developer must move them to the new number. The 2,000-recipient messaging tier requirement also means a new, unverified business must first build reputation or go through a BSP [UNVERIFIED how BSPs handle this for new numbers].

---

## Q6. Google Meet and Microsoft Teams platform access (no bot)

**Google Meet**
- **Meet REST API:** `conferenceRecords.recordings/transcripts/transcripts.entries/smartNotes` `get`/`list`. "If you're a meeting space owner or participant, you can call get and list." Artifacts live in the **organizer's Drive**/Docs. Transcript entries are deleted 30 days after the conference.
- **Transcripts require** Business Standard, Business Plus, Enterprise Starter/Standard/Plus, Education Plus, Teaching & Learning Upgrade or Workspace Individual. **Who can start:** only the host's domain (or host/co-hosts with host management on). Automatic transcription is Business Plus+ [per comparison table].
- **Visibility:** "a Transcripts icon appears … for everyone in the meeting". Admins may require explicit participant consent. **Not invisible.** If the client organizes, the developer (external) usually can't start it, and the transcript is emailed only to host, co-hosts and the person who started it.
- **Meet Media API (real-time):** Developer Preview only, **closed to new signups, endpoints shut down 2027-01-15**. It required all participants to be enrolled in the preview. **Dead end.**

**Microsoft Teams**
- **Graph** `onlineMeetings/{id}/transcripts`, `getAllTranscripts`, `adhocCalls` (incl. PSTN). Needs app permissions with an **application access policy** granted by the tenant admin, or meeting-scoped RSC. Tenant admins can disable Graph transcript access (`GraphAccessToTranscriptsDisabled`). This has been unmetered since 2025-08-25.
- **Client-tenant meetings:** Microsoft's support table states that a **person from another tenant can view the real-time transcription but cannot view or download the transcript after the meeting**. The developer's own-tenant Graph app has no rights in the client's tenant. **Not accessible.**
- **Visibility:** "all participants see a notification that the meeting is being transcribed". **Not invisible.**
- **Real-time media:** application-hosted media bots join as a participant, so they are not botless. There is no botless real-time API.

**Conclusion:** platform APIs help only for meetings **the developer's own Workspace/M365 tenant organizes**, and only with a visible transcript indicator. They are an optional enrichment, not a capture adapter.

---

## Q7. Hardware recorders

| Device | Phone-call capture | Export / API | Price | Status |
|---|---|---|---|---|
| **Plaud Note Pro** | Magnetic case/ring on the phone's back. A vibration/voice pickup sensor (VCS/VPU) captures **both sides of any call app** (cellular, WhatsApp, Teams). Auto-switches call vs. in-person (one review says manual switching) | **Plaud Embedded:** Device SDK (BLE/Wi-Fi; iOS 14+, Android 5+) to start/stop/monitor recordings, plus a Transcription API that "returns audio and speaker-labeled transcripts". Partner programme ("Become a partner"), priced on usage + devices. Consumer route: Zapier triggers (transcript/summary ready) | US$179–189. **India ₹20,990 MRP** (₹17,990 promo), Amazon/Blinkit. India launch press release dated Aug 2026 | Shipping |
| **Plaud NotePin S** | **No** call sensor; in-person only | same | US$179 | Shipping |
| **Plaud Note (original)** | Vibration conduction sensor call mode | same | ~US$159 [UNVERIFIED current] | |
| **Limitless Pendant** | Mic only | Data export | n/a | **Acquired by Meta (Dec 2025). Sales to new customers halted.** Existing users supported ≥ 1 year. |
| **Bee (Amazon)** | Wrist mic, ambient | API: [UNVERIFIED] | Launched at US$49.99 + optional sub | Amazon acquisition announced Jul 2025. Post-acquisition and India status [UNVERIFIED] |
| **RecorderGear PR200** | Pairs as a Bluetooth headset and records both sides (Q9) | SD/USB export | ~US$60–100 [UNVERIFIED] | Mixed reviews |

**Implication:** Plaud's Embedded SDK is the most credible hardware partner path. Rudra could pair a Plaud Note Pro directly and pull audio, skipping Plaud's app subscription. Call capture is still physically "clip on the phone", and the user must have the device on them. Transcription quality via vibration pickup is lower than a direct digital feed [UNVERIFIED, no benchmark found].

---

## Q8. Ambient capture by a second device

- A speakerphone call recorded by another device's mic is a "second-generation signal": phone codec (often 8 kHz narrowband on cellular), then loudspeaker, then room reverb, then the second mic. Published evidence is qualitative. One benchmark write-up reports far-field WER **2.5–4×** worse than close-talk. No controlled study of speakerphone re-recording WER was found [UNVERIFIED magnitude].
- **Echo:** the *phone's* AEC removes far-end audio from its own uplink, but the second device hears both parties raw. That's fine for transcription, but there is no channel separation, so diarization must split one mono mix. Interspeech 2025 work shows AEC/echo-aware training cuts WER by over 70% in loopback-playback scenarios, which is a loose analogy.
- **Practical guidance:** put the phone flat near the recording device, at moderate volume, in a quiet room. Expect usable but noticeably worse transcripts than a digital capture. Microsoft also warns of bad echo on Phone Link calls without a wired headset.
- **Laptop as second device:** the coverage guard can detect phone-call state (Android `READ_PHONE_STATE` → push to laptop) and auto-start laptop mic recording, giving **semi-automatic** capture. The developer must still switch the phone to speaker.

---

## Q9. Capture at the Bluetooth audio transport layer

**Premise check:** when a phone routes a call to a Bluetooth HFP device, the downlink (client voice) is sent to the headset over SCO/eSCO. The uplink (developer voice) comes **from the headset's mic**. A device in the headset role therefore holds both directions in digital form, for any app the phone routes to Bluetooth, with no OS sandbox involved. This is the only device-independent software-free path that exists today.

### (a) Windows Phone Link calling

- **Mechanism:** the PC is paired over Bluetooth and acts as the phone's hands-free unit. "Calling requires Bluetooth capability on both your PC and on your Android device." For iPhone the same applies: "As long as it's within Bluetooth range, you can make and take calls with your Windows PC." Requirements: Android 10+ (calls page) and iOS 16+ with a BLE-capable PC (requirements page; Microsoft pages conflict on minimums). The PC's Bluetooth adapter must expose the Hands-Free Telephony service [community answers]. Windows exposes `Windows.ApplicationModel.Calls.PhoneLineTransportDevice` ("currently only supported for use with bluetooth devices", Win10 1903+) with `AudioRoutingStatus`.
- **VoIP on the phone:** Phone Link is designed around cellular calls. A Microsoft Insider note says that if a VoIP call is answered on an iPhone connected over Bluetooth, the call can appear in Phone Link. **No documented, reliable VoIP relay** [UNVERIFIED]. For WhatsApp, Meet and Teams, do it on the laptop directly instead (Q1).
- **Third-party capture:** call audio plays on the PC's output device and the PC mic sends the uplink. So WASAPI **system loopback** + mic capture should get both sides. Which process renders the audio (Phone Link vs. a Bluetooth audio service), and therefore whether *process* loopback works, is **[UNVERIFIED, must test]**.
- **Quality:** HFP narrowband 8 kHz or wideband 16 kHz mSBC (Windows 10 1703+ / Windows 11). Some radios fall back to narrowband. LE Audio super-wideband (LC3) on Windows 11 24H2 applies to LE Audio *headsets*, not phone-to-PC calling [UNVERIFIED]. In practice, assume **8–16 kHz mono**.
- **Limits:**
  - "Phone Link does not currently support relaying a phone call … to a headset over Bluetooth" (Android and iPhone). The developer must use PC speakers/mic or a **wired** headset (echo warning).
  - One active phone connection [UNVERIFIED].
  - Not an emergency-call replacement.
  - India: no documented regional restriction [UNVERIFIED].
  - The developer must be at the PC.
- **Verdict:** a cheap, useful **"desk mode"** adapter for cellular calls on both Android and iPhone. Automatic when the call is taken on the PC, invisible to the client.

### (b) Can a Windows or macOS app be the HF unit without Phone Link?

- **Windows:** HF role support exists in the OS (Phone Link uses it). The public WinRT `Windows.ApplicationModel.Calls` (PhoneLine, PhoneLineTransportDevice) allows registering for and controlling a Bluetooth phone line, but the API was built for the Windows Phone/Your Phone dialer. Whether a third-party desktop app can do call control plus audio without Phone Link is **[UNVERIFIED]**. There is **no documented user-mode API to open raw SCO audio**. Third-party Bluetooth stacks need a dedicated USB dongle with its own driver, e.g. a WinUSB-bound adapter driven by BTstack, which supports HFP-HF [UNVERIFIED on Windows in 2026].
- **macOS:** IOBluetooth has `IOBluetoothHandsFreeDevice` ("manage phone calls on a connected Bluetooth hands-free phone or headset", macOS 10.7+). The Mac can act as HF toward a phone. 2012–2013 mailing-list reports show it working, but a recent Apple forum thread reports connect failures. Reliability on current macOS is **[UNVERIFIED]**.
- **Verdict:** possible in principle on macOS, fragile on Windows. Phone Link (a) is the practical laptop route.

### (c) Dedicated hardware bridge (ESP32-class)

- **Stack:** ESP-IDF provides an **HFP Client (HF)** API.
  - `esp_hf_client_register_audio_data_callback()` delivers **incoming** SCO payload (client voice).
  - `esp_hf_client_audio_data_send()` sends **outgoing** audio (developer mic) when Voice-over-HCI is enabled.
  - States exist for CVSD ("standard"), **mSBC** and **LC3-SWB**. There is a PCM resampler to 8 kHz.
  - Both directions pass through the firmware, so capturing both is straightforward.
- **Relay to the user's ears:**
  - **Wired** earphones/headset via an I2S codec are simple and low-latency (a few ms beyond the SCO path) [UNVERIFIED figure].
  - **Wireless earbuds** need the bridge to be an *audio source* at the same time. Espressif's example docs say A2DP sink and source can't run simultaneously. Community projects use **two radios** (two ESP32s, or ESP32 + a second BT SoC). That adds latency (~100–200 ms for A2DP) and complexity [UNVERIFIED figures]. LE Audio broadcast/unicast relay needs a newer SoC (nRF5340 or similar) [UNVERIFIED].
  - Only the original ESP32 (not S3/C3/C6) has Bluetooth Classic [UNVERIFIED in fetched docs; per Espressif product line].
- **Upload:** buffer to flash/SD, then upload via Wi-Fi or BLE to the Rudra phone app. Wi-Fi + Classic BT SCO coexistence on one ESP32 is tight, so upload after the call [UNVERIFIED].
- **Open source:**
  - ESP-IDF `bluedroid/classic_bt/hfp_hf` example.
  - `walinsky/a2dpsinkhfpclient` (A2DP sink + HFP client).
  - `pschatzmann/ESP32-A2DP` (no HFP).
  - No turnkey open-source "HFP call-recording bridge" was found.
- **BOM (rough):** ESP32-WROOM module ~US$3–5, audio codec + mic + amp ~US$3–6, microSD ~US$1–3, LiPo + charger ~US$3–5, PCB/enclosure ~US$5–10. **About US$15–30 at volume** (₹1,300–2,500), excluding certification (BIS/WPC in India, FCC/CE) [UNVERIFIED estimates].
- **Existing products:** **RecorderGear PR200**, a Bluetooth "call recording device" that pairs as a headset and "records both sides … on any Bluetooth compatible mobile phone". The user holds it to their ear. Reviews are mixed. **Plaud call mode** is *not* Bluetooth. It uses vibration pickup on the phone's back (Q7).
- **Verdict:** technically the most universal phone adapter (cellular + any VoIP app on Android, iPhone or iPad). Product risk is high: custom hardware, the user must wear or hold it or use wired earphones, it breaks the user's AirPods/earbuds habit, and it needs certification. Treat it as a **later-phase R&D bet**, possibly via a partner device rather than in-house.

### (d) iOS/iPadOS routing of VoIP to Bluetooth HFP

- iOS routes phone and FaceTime call audio to a connected Bluetooth headset automatically. Settings > Accessibility > Touch > **Call Audio Routing** (Automatic / Bluetooth Headset / Speaker) controls "the audio of phone or FaceTime calls".
- **Third-party VoIP** apps get HFP only if they set `AVAudioSession.CategoryOptions.allowBluetoothHFP` ("makes Bluetooth Hands-Free Profile (HFP) devices available for audio input"; record/playAndRecord only). WhatsApp, Teams and Meet do use Bluetooth headsets in practice, with user reports of per-app glitches [UNVERIFIED per app; no official app docs found]. CallKit-integrated apps (WhatsApp) follow the system call route [UNVERIFIED].
- **Restriction:** iOS picks one HFP route. The bridge must be the *active* audio device, so the user cannot simultaneously use AirPods for that call.

### (e) Does routing to Bluetooth notify the other party?

- **No mechanism found.** HFP is a local link between the phone (Audio Gateway) and the accessory. Nothing in Android's, Apple's or Microsoft's documentation describes any network-side signal or announcement when audio is routed to a Bluetooth device. Announcements exist only for *recording features* (Google Phone app, Apple call recording) [negative finding; UNVERIFIED in the sense that no source states it explicitly].

---

## Recommended adapter set

### MVP (ship first)

1. **Desktop companion (Windows first, then macOS), Tauri + Rust.**
   - **Capture:** two-track capture (process loopback of the call app + mic), using `wasapi` / Core Audio taps (Swift sidecar if needed).
   - **Auto-start:** on mic-session detection for known processes plus calendar events.
   - **Covers:** Meet, Teams, Zoom, Slack, WhatsApp Desktop, and **Phone Link cellular calls** (system loopback). Invisible, automatic, free.
   - *Rationale:* it covers most client calls for developers (video meetings) at zero marginal cost, and it is proven by Granola.
2. **Mobile in-person recorder** (Android + iOS app): mic recording with a foreground service, auto-prompt from calendar or location. *Rationale:* trivial, needed anyway.
3. **Android OEM-recording importer** (Samsung/Xiaomi in India first): `READ_MEDIA_AUDIO`/SAF folder watch, matching files to `READ_PHONE_STATE` call events. *Rationale:* covers cellular calls on the most common Indian phones, invisibly, at zero cost, if the OEM doesn't announce. Must verify the announcement behaviour per model.
4. **Coverage guard:**
   - Android: call-state listener (cellular), plus `MODE_IN_COMMUNICATION` (VoIP), plus "no recording arrived".
   - Desktop: session detection.
   - iOS: CallKit observer [UNVERIFIED].
   - Each prompts: "Call with +91… at 14:05 wasn't captured. Add notes or upload audio?"
   - *Rationale:* honesty about gaps is cheaper than pretending to full capture.
5. **Manual upload / share-sheet import** for anything else: iPhone Notes call recordings, Plaud exports, WhatsApp voice notes.

### Later

6. **Cloud-telephony "business line"** (Exotel or Knowlarity) for developers who want automatic cellular capture on iPhone or non-recording Android. Inbound via CFU forwarding to an ExoPhone, answered in the Rudra app (WebRTC/SIP) or a second SIM. *Trade-off:* outbound shows the virtual number. India KYC and per-minute cost apply.
7. **WhatsApp Business Calling adapter** for developers willing to move clients to an API-only business number. Full audio, free inbound.
8. **Plaud Embedded integration** (partner SDK) as the "any call on any phone" accessory for heavy phone users.
9. **Bluetooth bridge R&D** (Q9c). Prototype on ESP32 with wired earphones. Evaluate a partner OEM before building.
10. **Meet/Teams native transcript enrichment** for meetings the developer's own tenant organizes (visible to participants, so opt-in).

---

## Hard limits (genuinely impossible today)

1. **No third-party app on Android can capture cellular or VoIP call audio.** The `VOICE_*` sources need the system-only `CAPTURE_AUDIO_OUTPUT`. Playback capture excludes voice-communication usage. The mic is silenced for ordinary apps during calls. Play bans call recorders and accessibility-based recording. (Only root/sideload or OEM-privileged apps can, and those are off-Play.)
2. **No third-party app on iPhone/iPad can capture call audio or use the mic during a cellular call** (Apple DTS). There is no system-audio loopback on iOS.
3. **Apple's native call recording always announces to both parties**, periodically. No API exposes its recordings.
4. **Personal WhatsApp calls cannot be received or captured by any third party** (E2EE, no API). Only a separate Cloud-API business number gives audio.
5. **No botless real-time media API** for Meet (Media API shutting down 2027-01-15) or Teams. Native transcripts are always visible to participants, and **client-organized (other-tenant) Teams transcripts are inaccessible** to the developer.
6. **Cloud telephony in India cannot show the developer's own mobile number on outbound calls** (caller ID must be a provider/virtual number; spoofing violates TRAI rules). Twilio cannot carry India-domestic calls. Calls must be anchored in India.
7. **Physical capture still needs a physical act:** clip a Plaud on, use a headset bridge, or put the phone on speaker. None is automatic per call, and none works if the developer forgets.

---

## Sources

**Windows / macOS**
- WASAPI loopback recording: https://learn.microsoft.com/en-us/windows/win32/coreaudio/loopback-recording
- AUDIOCLIENT_PROCESS_LOOPBACK_PARAMS (Win10 build 20348): https://learn.microsoft.com/en-us/windows/win32/api/audioclientactivationparams/ns-audioclientactivationparams-audioclient_process_loopback_params
- IAudioSessionManager2: https://learn.microsoft.com/en-us/windows/win32/api/audiopolicy/nn-audiopolicy-iaudiosessionmanager2
- Bluetooth Classic audio on Windows (HFP narrowband/wideband): https://learn.microsoft.com/en-us/windows-hardware/drivers/bluetooth/bluetooth-classic-audio
- PhoneLineTransportDevice: https://learn.microsoft.com/en-us/uwp/api/windows.applicationmodel.calls.phonelinetransportdevice
- Core Audio taps: https://developer.apple.com/documentation/coreaudio/capturing-system-audio-with-core-audio-taps
- SCStreamConfiguration.captureMicrophone: https://developer.apple.com/documentation/screencapturekit/scstreamconfiguration/capturemicrophone
- kAudioProcessPropertyIsRunningInput: https://developer.apple.com/documentation/coreaudio/kaudioprocesspropertyisrunninginput
- Mic detection via IsRunningSomewhere (forum): https://developer.apple.com/forums/thread/741026
- IOBluetoothHandsFreeDevice: https://developer.apple.com/documentation/iobluetooth/iobluetoothhandsfreedevice
- ConsentStore registry (community): https://www.elevenforum.com/t/clear-microphone-last-accessed-history-in-windows-11.8831/
- Granola transcription docs: https://docs.granola.ai/help-center/taking-notes/transcription
- Granola phone calls blog: https://www.granola.ai/blog/best-ai-note-taker-for-phone-calls-on-android-and-iphone
- Granola for Android: https://www.granola.ai/blog/granola-for-android-ai-meeting-notes-for-mixed-device-teams
- Crates: https://crates.io/crates/wasapi, https://github.com/HEnquist/wasapi-rs, https://crates.io/crates/cpal, https://crates.io/crates/screencapturekit

**Android**
- AudioPlaybackCapture: https://developer.android.com/media/platform/av-capture
- MediaRecorder.AudioSource: https://developer.android.com/reference/android/media/MediaRecorder.AudioSource
- Sharing audio input (voice call + ordinary app): https://developer.android.com/media/platform/sharing-audio-input
- AOSP concurrent capture: https://source.android.com/docs/core/audio/concurrent
- Play SMS/Call Log policy: https://support.google.com/googleplay/android-developer/answer/10208820
- Play Accessibility API policy: https://support.google.com/googleplay/android-developer/answer/10964491
- 2022 accessibility call-recording ban (press): https://www.theregister.com/2022/04/22/google_banning_thirdparty_callrecording_apps/
- Shared media storage / READ_MEDIA_AUDIO: https://developer.android.com/training/data-storage/shared/media
- Google Phone app call recording: https://support.google.com/phoneapp/answer/9803950
- Salestrail Samsung One UI 8 built-in recorder: https://www.salestrail.io/knowledge-base/sim/gsm-call-recording-on-samsung-devices-running-one-ui-8-android-16
- Beebom Android call recording guide: https://gadgets.beebom.com/guides/how-to-record-a-phone-call-on-android
- OEM dialer reports: https://www.tech2.com/technology/5-android-phones-that-come-with-built-in-call-recorder-and-how-it-works-text-ws-el-10251242.html, https://piunikaweb.com/blog/automatic-call-recording-finally-comes-to-oneplus-devices-albeit-with-a-catch

**iOS**
- Apple DTS answer on mic during cellular calls: https://developer.apple.com/forums/thread/856047
- Record and transcribe a call (iPhone guide): https://support.apple.com/guide/iphone/record-and-transcribe-a-call-iph57c6590e9/ios
- Personal Safety guide (notice, periodic replay, regions): https://support.apple.com/en-gb/guide/personal-safety/ipsfa513e1a6/web
- Apple support 121583: https://support.apple.com/en-us/121583
- Call Audio Routing: https://support.apple.com/guide/iphone/auto-answer-calls-iph29145acf1/ios
- allowBluetoothHFP: https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/allowbluetoothhfp
- TapeACall 3-way calling: https://www.tapeacall.com/blog/3-way-conference-calling-on-iphone-everything-you-need-to-know
- India carriers 3-way (third party): https://www.airdroid.com/remote-support/how-do-3-way-call-on-iphone

**Telephony (India)**
- Twilio India voice guidelines: https://www.twilio.com/en-us/guidelines/in/voice
- Plivo India calling: https://www.plivo.com/docs/voice/concepts/india-calling
- Exotel voice API India guide: https://exotel.com/blog/voice-call-api-in-india/
- Exotel Connect Two Numbers: https://developer.exotel.com/docs/voice-v1/api-reference/connect-two-numbers
- Exotel custom caller ID: https://developer.exotel.com/docs/advanced-config/custom-caller-id
- Exotel pricing model: https://developer.exotel.com/docs/billing/pricing-model
- Knowlarity pricing (third party): https://www.softwaresuggest.com/knowlarity-smartivr/pricing
- DoT USSD forwarding order (28 Mar 2024): https://dot.gov.in/sites/default/files/28032024%20ussd.pdf
- OSP 2021 commentary: https://khaitanco.com/thought-leaderships/DoT-Re-vision-to-the-OSP-framework

**WhatsApp**
- Calling API overview: https://developers.facebook.com/docs/whatsapp/cloud-api/calling
- Calling SIP: https://developers.facebook.com/docs/whatsapp/cloud-api/calling/sip
- Calling pricing: https://developers.facebook.com/docs/whatsapp/cloud-api/calling/pricing
- India availability (vendors): https://exotel.com/blog/introducing-whatsapp-business-calling-api-beta/, https://support.infobip.com/docs/whatsapp/whatsapp-business-calling/business-initiated-calling
- Coexistence and calling (third party): https://wassenger.com/help/coexistence

**Meet / Teams**
- Meet artifacts: https://developers.google.com/workspace/meet/api/guides/artifacts
- Meet transcripts help: https://support.google.com/meet/answer/12849897
- Meet REST API release notes (Media API shutdown): https://developers.google.com/workspace/meet/release-notes
- Teams transcripts via Graph: https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/meeting-transcripts/overview-transcripts
- Get callTranscript: https://learn.microsoft.com/en-us/graph/api/calltranscript-get
- Teams live transcripts (external tenant can't view after meeting): https://support.microsoft.com/en-us/teams/meetings/start-stop-and-download-live-transcripts-in-microsoft-teams-meetings

**Hardware / Bluetooth**
- Plaud NotePin vs NotePin S: https://www.plaud.ai/blogs/articles/plaud-notepin-vs-plaud-notepin-s
- Plaud India launch: https://in.plaud.ai/blogs/news/plaud-launches-in-india-bringing-conversational-intelligence-to-a-new-generation-of-india-s-ambitious-professionals
- Plaud Note Pro India: https://in.plaud.ai/pages/plaud-note-pro
- Plaud developer platform: https://dev.plaud.ai/
- Plaud Zapier: https://support.plaud.ai/hc/en-us/articles/12200669941647-Zapier-integration
- Limitless/Meta: https://betanews.com/2025/12/08/meta-has-acquired-limitless-a-startup-that-produces-ai-powered-wearables/
- Bee/Amazon: https://siliconangle.com/2025/07/22/amazon-buys-wearable-ai-device-maker-bee-undisclosed-sum/
- Phone Link calls: https://support.microsoft.com/en-us/windows/apps/make-and-receive-phone-calls-from-your-pc
- Phone Link requirements: https://support.microsoft.com/en-us/windows/apps/phonelink/phone-link-requirements-and-setup
- Phone Link call troubleshooting (no headset relay, echo): https://support.microsoft.com/en-US/Windows/Apps/PhoneLink/troubleshooting-calls-in-the-Phone-Link
- ESP-IDF HFP Client API: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/bluetooth/esp_hf_client.html
- ESP32 A2DP library: https://github.com/pschatzmann/ESP32-A2DP
- a2dpSinkHfpClient component: https://components.espressif.com/components/walinsky/a2dpsinkhfpclient/versions/0.2.2/readme
- RecorderGear PR200: https://www.newegg.com/p/9SIA58A4SA8593
- Windows LE Audio super wideband: https://techcommunity.microsoft.com/blog/windowsosplatform/cutting-the-wire-without-cutting-the-audio-quality/4447942

**Ambient**
- Speakerphone transcription degradation (vendor explainer): https://shop.zimaspace.com/blogs/tech-ai-hub/transcription-less-accurate-speakerphone-audio
- Real-world far-field WER: https://embeddedcomputing.com/technology/ai-machine-learning/computer-vision-speech-processing/beyond-the-lab-why-voice-ai-must-be-benchmarked-under-real-world-conditions
- Interspeech 2025 echo-aware ASR: https://www.isca-archive.org/interspeech_2025/lin25b_interspeech.pdf
