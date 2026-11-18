# 05: Speech-to-Text Options and Call-Recording Consent

**Date:** 2026-11-18. **Method:** live web search plus official docs and pricing pages where they could be fetched. Anything not confirmed against a primary or official source is marked **[UNVERIFIED]**. Prices change often, so re-check them before committing.

> **Not legal advice.** Part B is a research map that tells a lawyer where to look. It is not a legal opinion.

---

## Executive summary

**STT**
- **Hinglish accuracy separates the vendors sharply.** The largest independent conversational Hinglish benchmark (Specific Labs, June 2026) reports these WERs:
  - ElevenLabs Scribe v2: 8.1%
  - Deepgram Nova-3: 19.3%
  - Gemini 3.5 Flash: 35.0%
  - Sarvam saarika:v2.5 (old model): 63.6%
  - Groq whisper-large-v3: 75.7%
  - OpenAI gpt-4o-transcribe: 91.3%

  On a second, independent Hinglish evaluation (Trelis, 2026), **Sarvam Saaras v3 in `codemix` mode** roughly matches Scribe v2. It wins on conversational and child-accent speech and loses on read speech.
- **Recommendation:**
  - **Primary (production):** ElevenLabs Scribe v2, batch, in multichannel mode.
  - **Fallback and default for development:** Deepgram Nova-3 Multilingual. It comes with $200 of non-expiring free credit (several hundred hours) and has the best streaming option.
  - **India-residency and regional-language option:** Sarvam Saaras v3 (`codemix`), at about ₹29–35/hr, with data processed in India.
  - **Avoid for Hinglish:** Groq Whisper and OpenAI. They are cheap, but their accuracy on code-mixed speech is poor.
- **Strategy:**
  - Use batch transcription after the call by default. It is cheaper and more accurate, gives better diarization, and lets Rudra delete the audio quickly.
  - Offer live streaming only as an opt-in feature.
  - Transcribe the mic and remote channels separately. The mic channel needs no diarization, and the remote channel needs it only when two or more client speakers are present.

**Legal**
- **India:** Recording a call you take part in is not expressly criminalised for private individuals. There are two problems, though:
  - Several High Courts have held that secret recording violates the Article 21 privacy right.
  - The **DPDP Act 2023** almost certainly applies to professional use, which is outside the "personal/domestic" exclusion. That makes the developer a **Data Fiduciary** and Rudra a **Data Processor**. The Act's main obligations take effect around **13 May 2027**.
- **Abroad:** A covert recording is likely **unlawful** when the client is in an all-party jurisdiction. That includes California, Florida, Washington, Illinois, Pennsylvania, Massachusetts, Maryland, Montana and New Hampshire, plus **Germany (§201 StGB, criminal)**, the **UAE (criminal)** and **NSW, Australia**. Courts generally apply the law of the place where the protected party is. For example, *Kearney v. Salomon Smith Barney* (Cal. 2006) applied California law to an out-of-state recorder.
- **Litigation is now aimed at invisible note-takers:**
  - *In re Otter.AI Privacy Litigation* (N.D. Cal.): the 13 Aug 2026 order let the Wiretap Act, CIPA §631 and BIPA claims proceed on a "third-party eavesdropper" theory, because the vendor used recordings for its own purposes.
  - ***Chamberlain v. Granola*** (N.D. Cal., filed 30 Jul 2026) targets a **bot-free** design specifically.
  - Fireflies faces Illinois BIPA (voiceprint) suits.
- **Conclusion:** "Invisible to the client" can be defended only if *someone* obtains consent where the law requires it. The consent does not have to come from a bot. The user can obtain it once per client, in the contract, the SOW or an email. Rudra should:
  - make that consent step easy and record it;
  - be jurisdiction-aware;
  - never train on customer audio;
  - never build voiceprints;
  - delete the audio after transcription.

---

## Part A: Cloud speech-to-text

### A.1 Hinglish / code-mixed accuracy evidence

| Benchmark | Data | Key results (WER, lower is better) | Caveats |
|---|---|---|---|
| **Specific Labs bench-1** (June 2026) | Natural, unscripted conversations between native speakers, with overlap and mid-sentence switches. Hindi is in Devanagari and English in Latin script. | ElevenLabs scribe_v2 **8.1**, Deepgram nova-3 **19.3**, gemini-3.5-flash 35.0, gpt-4o-transcribe-diarize 44.8, gemini-3.1-pro-preview 52.6, AssemblyAI (Universal-2 fallback, because U-3 Pro lacks Hindi) 53.3, gpt-4o-mini-transcribe 63.0, Sarvam **saarika:v2.5** 63.6, Groq whisper-large-v3 75.7, gpt-4o-transcribe 91.3, whisper-1 94.5 | Small public sample. Part of the Whisper-family gap comes from script convention (English written in Devanagari). Sarvam was tested on the older Saarika model, not Saaras v3 codemix. |
| **Trelis "Tara" model card** (2026) | CoSHE-500 (conversational), CS-FLEURS hi-en (read), HiACC (accented). Clips ≤30 s. | Sarvam Saaras-v3 codemix vs Scribe-v2: CoSHE **11.25 vs 12.40**, CS-FLEURS 16.47 vs **7.57**, HiACC adult 13.16 vs **12.87**, HiACC child **10.10** vs 11.66 | Published by a competing model vendor. The evaluation code is public. |
| AssemblyAI code-switching blog (2026) | 5 language pairs | Scribe v2 8.77, Nova-3 Multilingual 12.22, GPT-4o Transcribe 44.58 | Vendor-run, and not Hinglish-only. [UNVERIFIED detail] |
| Soket AI (older) | Hindi-English | Nova-2 15.7, GPT-4o-mini 42.3, Saarika 2.5 56.8, Scribe v1 71.3 | Old model generations. |
| Indic DiarBench (arXiv 2607.23808) | 108 h of multi-speaker audio across 22 Indian languages, for joint diarization and ASR | Baseline numbers not extracted [UNVERIFIED] | **Use this to run Rudra's own evaluation.** |

**Takeaway:** ElevenLabs Scribe v2 leads most consistently. Sarvam Saaras v3 (codemix) is competitive and leads on conversational Hinglish in one study. Deepgram Nova-3 is a solid second tier. Gemini is usable but weaker. Whisper and OpenAI are poor on Hinglish. **Rudra should benchmark on 1–2 hours of its own Indian-accent, poor-network call audio before locking in a vendor.**

### A.2 Comparison table

Prices are list pay-as-you-go rates per audio hour. ₹ is converted at about ₹88/$ [UNVERIFIED FX].

| Option | Hinglish quality | Diarization | Streaming | Word timestamps | Max file | Free tier / credits | Price / audio hr | Data use / training | India residency |
|---|---|---|---|---|---|---|---|---|---|
| **ElevenLabs Scribe v2 (batch) / v2–v3 Realtime** | **Best** (8.1% Specific) | Yes, up to 32 speakers (batch). Realtime diarization in Scribe v3 Live. **Multichannel up to 5 channels**, each channel labelled as its own speaker. | Yes (Realtime) | Yes | 5 GB / **15 h** | Small free plan allowance [UNVERIFIED] | **~$0.22** batch; ~$0.39 realtime (secondary sources; the official FAQ still says "from $0.40") [UNVERIFIED] | Zero-retention mode (`enable_logging=false`, per request, Enterprise) [UNVERIFIED tier] | **Yes: India data residency (Enterprise)** |
| **Deepgram Nova-3 Multilingual** | Good (19.3%) | Yes. Free on pre-recorded; +$0.0020/min when streaming | **Yes (best-in-class)** | Yes | Large [UNVERIFIED limit] | **$200 free credit, non-expiring, no card** | Pre-recorded **$0.31** ($0.0052/min); streaming $0.35 (+$0.12 diarization) | Model Improvement Program: opt out per request with `mip_opt_out=true`. Since a Mar 2026 changelog this no longer changes the price. | No India region [UNVERIFIED]. US/EU. |
| **Sarvam AI Saaras v3 / v4** (`codemix` mode) | Good to best on conversational Hinglish (Trelis). Covers 22 Indian languages. | Yes, batch only (up to 20 speakers) | Yes (WebSocket; REST ≤30 s) | [UNVERIFIED] | Batch 2 h per file, 20 files per job | **₹100 free credit** (about 3 h) | **₹29.40 (~$0.33)**; with diarization ₹35.40 (~$0.40) | Developer page: "API inputs are never used for model training"; deleted after processing by default. A third-party listing reads the terms differently [verify DPA]. | **Yes: India-only** (Trust Center) |
| **Google Gemini audio** (e.g. 3.5 Flash-Lite, 3.6/3.8 Flash) | Fair (gemini-3.5-flash 35%) | Prompted only (labels in structured output). Not a dedicated diarizer. | Not for this API (Google points to Cloud STT / Live) | Prompted MM:SS only. **Not reliable word timing.** | **9.5 h** per prompt (Files API >20 MB) | **Free tier exists, but free-tier content is used to improve Google products. Do not use it for client audio.** | 32 tokens/s, so about 115k input tokens/hr. At $0.30/1M (3.5 Flash-Lite), input is about **$0.035/hr** plus transcript output tokens [output price UNVERIFIED]. | Paid tier not used for training | Via Vertex AI regions [UNVERIFIED for these models] |
| **Google Cloud STT v2 (Chirp 3)** | Not benchmarked on Hinglish here | Yes, but **batch only** for Chirp 3 | Yes | Yes | Long-running batch [UNVERIFIED] | Free 60 min/mo plus GCP trial credit [UNVERIFIED] | ~$0.96 ($0.016/min); dynamic batch ~$0.18 ($0.003/min, 24 h turnaround) [secondary sources] | Data-logging opt-in discount [UNVERIFIED] | asia-south1 (Mumbai) is **Preview** for chirp_3 per release notes [UNVERIFIED for hi-IN + diarization] |
| **AssemblyAI** (STT-3.7 / Universal-2) | Weak for Hindi: STT-3.7 covers 32 languages and Hindi is routed to Universal-2 (53.3%) | +$0.02/hr batch; +$0.12/hr streaming | Yes | Yes | [UNVERIFIED] | **$50 credit** | U-2 **$0.15**; STT-3.7 $0.21; streaming $0.30 (billed on session time) | Model-improvement opt-out documented | EU residency; no India region |
| **OpenAI** (gpt-4o-transcribe, -diarize, mini, gpt-transcribe) | **Poor** (44.8–91.3%) | gpt-4o-transcribe-diarize | Yes (gpt-live-transcribe $0.017/min) | whisper-1 only [UNVERIFIED for 4o] | 25 MB per request [UNVERIFIED] | None | $0.18 (mini) to $0.36/hr | API data not used for training by default [UNVERIFIED here] | No India region [UNVERIFIED] |
| **Groq Whisper large-v3 / -turbo** | **Poor** (75.7%) | **No** | No (fast batch, 216x real time) | Yes (`verbose_json`) | 25 MB free tier / 100 MB dev tier, or by URL | **Free tier** (about 2,000 requests/day; audio-seconds caps) [limits UNVERIFIED by plan] | **$0.04** (turbo) / $0.111 (v3); 10 s minimum per request | Self-serve Zero Data Retention. Default retention is described inconsistently (up to 30 days). | No |
| **Soniox v6** | Claims mid-sentence switching across 60+ languages. **No independent Hinglish number.** | Included in price (up to 15 speakers, also when streaming) | Yes | Yes | [UNVERIFIED] | [UNVERIFIED] | **$0.10** async / $0.12 realtime (vendor) | [UNVERIFIED] | [UNVERIFIED] |
| **AI4Bharat / Bhashini (ULCA)** | IndicConformer models (MIT licence), 22 languages | [UNVERIFIED] | Yes (Bhashini pipeline) | [UNVERIFIED] | [UNVERIFIED] | Free API keys via the ULCA portal, but reportedly "**proof-of-concept only**". Production needs a paid agreement. | Not published [UNVERIFIED] | Government platform | India |

**Notes**
- Bhashini is not a realistic production dependency for a SaaS today. Self-hosting IndicConformer is ruled out by the "no local models" constraint, although a cloud GPU later could change that.
- Microsoft's mai-transcribe-1.5 scored 52.3% on Specific's Hinglish benchmark. Azure was not researched further.

### A.3 Recommendation

| Role | Choice | Why |
|---|---|---|
| **Primary (production, post-call batch)** | **ElevenLabs Scribe v2** (multichannel) | Best Hinglish evidence. Multichannel mode gives a speaker label per channel, and diarization handles multiple clients on the remote channel. 15 h file limit, word timestamps, India residency on Enterprise, roughly $0.22/hr. |
| **Fallback, development default and streaming** | **Deepgram Nova-3 Multilingual** | $200 free credit covers about 640 h of batch transcription, so development costs effectively nothing. Second-best Hinglish score, mature streaming, free diarization on pre-recorded audio, and a training opt-out at no extra cost. |
| **India-residency / regional languages** | **Sarvam Saaras v3/v4, `codemix`** | Data stays in India, strong conversational Hinglish, and covers Tamil, Telugu and other languages mixed with English. Offer it as a selectable engine, and as the default for users who need India-only processing. |
| **Not recommended for Hinglish** | Groq Whisper, OpenAI, AssemblyAI (for Hindi) | Low cost does not offset WERs of 50–90%. Groq remains acceptable for English-only calls in development. |
| **Gemini** | **Only as an analysis LLM**, not as the main STT | It works as a cheap bring-your-own-key fallback on the paid tier only. The free tier must never receive client audio. |

**Bring-your-own-key:** Abstract STT behind an interface (`transcribe(channelAudio, opts) → words[{text,start,end,speaker,channel}]`) with adapters for ElevenLabs, Deepgram and Sarvam. Send every request with no-retention / no-training flags: `enable_logging=false` for ElevenLabs and `mip_opt_out=true` for Deepgram.

**Streaming vs batch strategy**
1. **Default to batch after the call.** It costs less, is more accurate (more context), and gives better diarization. Scribe v2 diarization is batch-only, and so is Sarvam's and Chirp 3's.
2. **Upload in chunks while the call runs** (for example every 5–10 minutes per channel). The transcript is then ready seconds after hang-up, and long calls never become huge files.
3. **Offer live mode as opt-in** (Deepgram streaming or Scribe Realtime) only for a live-hints feature. It costs roughly 1.1–2x more, and the diarization is provisional.

**Using the two channels to reduce diarization**
- **Mic channel = the user.** Transcribe it with no diarization. Every word belongs to the user.
- **Remote channel = the clients.**
  - Enable diarization only when more than one client voice is expected, for example when calendar attendees number two or more. Otherwise label the whole channel "Client".
  - With ElevenLabs, send both as one multichannel file (up to 5 channels). Each channel gets its own speaker ID, and diarization within a channel still separates multiple clients [verify that multichannel and diarization combine].
- **Merge** the two transcripts by word timestamps into a single timeline.
- **Echo and bleed:** speaker output picked up by the mic duplicates client speech in the mic channel. Apply OS or WebRTC echo cancellation, and drop mic-channel words that time-overlap and fuzzy-match remote words.
- **Single-channel sources** (in-person meetings, and phone or WhatsApp calls where only a mixed stream is available) need full diarization. Expect lower quality.
- **Do not build persistent voiceprints** to name speakers across calls. That creates biometric (BIPA) risk; see Part B. Name speakers from calendar or CRM context instead.

---

## Part B: Call-recording consent law

### B.1 India

**1. Is it lawful for a participant to record secretly?** No statute expressly forbids a private participant from recording. The case law, however, pulls in different directions.

- **Telecommunications Act 2023:** s.42(2)(b) punishes anyone who "intercepts a message unlawfully" with up to 3 years' imprisonment and/or a fine of up to ₹2 crore. "Interception" is generally understood as third-party or state capture, not a party recording their own call [UNVERIFIED interpretation; lawyer]. The old Telegraph Act s.5(2) and the *PUCL* (1997) line of cases concern state tapping.
- **IT Act 2000:**
  - s.66E covers images, not audio.
  - s.72 applies only to persons acting under powers conferred by the Act.
  - **s.43A and the SPDI Rules 2011** are the more relevant provisions. The SPDI Rules define "biometrics" to include **voice patterns** [UNVERIFIED wording]. Those rules are due to give way to the DPDP regime.
- **Privacy (Article 21):**
  - *K.S. Puttaswamy v. Union of India* (2017) recognises privacy as a fundamental right, enforceable mainly against the State.
  - High Courts have nonetheless held that secretly recording another person's calls violates privacy: Delhi HC, Chhattisgarh HC (2023), and Telangana HC (spousal recordings inadmissible).
  - **Contra:** *Vibhor Garg v. Neha*, 2025 INSC 829 (SC, 14 Jul 2025) held that secretly recorded spousal calls are **admissible** in matrimonial proceedings, because privacy is not absolute. Commentators read it narrowly (Evidence Act s.122 context). It concerns admissibility, not legality.
  - *R.M. Malkani v. State of Maharashtra* (SC 1973) admitted a tape recording made by a participant [UNVERIFIED here].
- **Bottom line:** Covert recording by a participant is a grey area. Criminal exposure is low, but there is real civil and privacy-tort exposure, and evidence may be challenged. **[Lawyer]**

**2. DPDP Act 2023 and Rules 2025**
- **Commencement:** The Rules were notified 13–14 Nov 2025 (G.S.R. 846(E)). Consent-manager provisions apply from about Nov 2026. **Most obligations (notice, consent, security, retention, breach reporting) apply from about 13 May 2027.**
- **Personal/domestic exclusion:** The Act does not apply to "personal data processed by an individual for any personal or domestic purpose" (s.3(c)(i)). Recording *client* calls for a freelance or business engagement is **professional**, so it is very likely **outside** the exclusion. GDPR's *Ryneš* (CJEU C-212/13) reading of the same concept supports a narrow interpretation.
- **Roles:**
  - The **developer** (sole proprietor or company) decides purpose and means, so they are the **Data Fiduciary**.
  - **Rudra** processes on their behalf, so it is a **Data Processor** under s.2(k). That requires a contract (s.8(2)). The fiduciary stays liable (s.8(1)) and must ensure security safeguards (s.8(5)).
  - If Rudra uses transcripts for its own purposes (training, cross-customer analytics), it risks becoming a fiduciary itself.
- **Lawful ground:** DPDP has no "legitimate interests" basis. Processing needs **consent** (s.6), which must be free, specific, informed and unambiguous with a notice, or a narrow **legitimate use** (s.7). Section 7(a), covering data the principal voluntarily provided for a specified purpose, is arguable at best for covert capture **[Lawyer]**.
- **Territorial scope:** The Act applies to processing **within India** whatever the data principal's nationality. Foreign clients' data processed by an Indian developer or by Rudra is therefore in scope. The s.17(1)(d) exemption covers foreign-contract outsourcing processing, which is a different situation **[Lawyer]**.
- **Penalties:** Up to ₹250 crore for security failures (Schedule) [UNVERIFIED amount here].

### B.2 Jurisdiction table: recording a conversation you are part of

| Jurisdiction | Rule | Key citations | Notes |
|---|---|---|---|
| **India** | **Unclear** (no express bar; privacy case law against it) | Telecom Act 2023 s.42(2)(b); IT Act s.43A + SPDI Rules; *Puttaswamy* (2017); *Vibhor Garg* (2025); Chhattisgarh, Delhi and Telangana HCs | DPDP consent duties from May 2027 |
| **US federal** | One-party | 18 U.S.C. §2511(2)(d) (no criminal or tortious purpose) | Vendors face "third-party eavesdropper" claims (*Otter*, *Ambriz v. Google*) |
| **California** | **All-party** (confidential communications) | Cal. Penal Code §632 (and §631 wiretap); §637.2 $5,000 per violation | *Kearney* (2006): applies to out-of-state recorders of California callers |
| **Washington** | **All-party** | RCW 9.73.030 | Announcement must itself be recorded [UNVERIFIED] |
| **Illinois** | **All-party** (private conversations) + **BIPA** for voiceprints | 720 ILCS 5/14-2 (eavesdropping) [UNVERIFIED cite; one source cites 5/26-4]; 740 ILCS 14 (BIPA) | Fireflies and Otter BIPA claims |
| **Florida** | **All-party** | Fla. Stat. §934.03 | Plaintiff in *Chamberlain v. Granola* is in Florida |
| **Pennsylvania** | **All-party** | 18 Pa.C.S. §5703–5704 | |
| **Massachusetts** | **All-party** (bars "secret" recording, in person too) | M.G.L. c.272 §99 | |
| **Maryland** | **All-party** | Md. Cts. & Jud. Proc. §10-402 | |
| **Montana, New Hampshire** | **All-party** | MCA 45-8-213; RSA 570-A:2 [UNVERIFIED cites] | |
| **Connecticut** | All-party for phone calls (civil); one-party in person | Conn. Gen. Stat. §52-570d [UNVERIFIED] | Hybrid |
| **Delaware, Michigan, Nevada, Oregon** | **Unclear / hybrid** | Conflicting statutes and case law | Oregon: all-party in person. Treat all four as all-party. |
| **Other US states** | One-party | | |
| **UK** | One-party for an individual's own recording. **UK GDPR applies to business use.** | Investigatory Powers Act 2016 (formerly RIPA); Lawful Business Practice Regs; UK GDPR Art. 6, 13 | ICO reprimanded Surrey and Sussex Police (2023) over 200k covertly recorded calls. Transparency is expected. |
| **EU / GDPR** | Lawful basis + **transparency (Art. 13/14)** needed. Covert professional recording is hard to justify. | GDPR Art. 2(2)(c) (household, narrow, *Ryneš*), Art. 6(1)(a)/(f), Art. 13 | Member-state criminal law sits on top |
| **Germany** | **All-party, criminal** | **§201(1) StGB**: recording another's non-publicly spoken word without authorisation; up to 3 years or a fine; prosecuted on complaint (§205) | Business calls and video meetings are "non-public" [UNVERIFIED application]. Highest-risk EU state. |
| **Canada** | One-party (criminal law) | Criminal Code ss.183.1, 184 | PIPEDA / provincial privacy law applies to commercial recording (notice/purpose) [UNVERIFIED detail] |
| **Australia** | **Varies by state.** One-party participant exception in Qld, Vic and NT. **All-party in NSW** (and others). | Telecommunications (Interception and Access) Act 1979; Surveillance Devices Act 2007 (NSW); Invasion of Privacy Act 1971 (Qld); ALRC Report 123 | Treat as all-party |
| **UAE** | **All-party, criminal** (even for a participant, per commentary) | Federal Decree-Law 34/2021 (Cybercrime) Art. 44; Penal Code (Decree-Law 31/2021) Art. 431 / 433 | Fines reported from AED 150k–500k up to AED 1M [conflicting] |
| **Singapore** | No clear participant wiretap bar [UNVERIFIED]. **PDPA requires notice and consent** for organisations recording in the course of business. | PDPA ss.13–20; PDPC guidance (voice recordings are personal data) | Treat as notify/consent |

### B.3 Cross-border calls (caller in India, client in California or Germany)

- **Several laws can apply at once.** Wiretap and privacy statutes are usually applied by the **place where the protected person is or where the harm occurs**, in addition to the recorder's location.
  - *Kearney v. Salomon Smith Barney*, 39 Cal.4th 95 (2006): California's all-party rule applied to a Georgia firm recording calls with Californians.
  - German criminal law treats an offence as committed where its result occurs (§9 StGB) [UNVERIFIED application to §201]. Practical prosecution of an Indian individual is unlikely, but civil claims, GDPR complaints and reputational damage are not.
- **GDPR reach:**
  - An Indian developer recording an EU client is arguably not covered by Art. 3(2), unless they offer services to people in the EU or monitor their behaviour **[Lawyer]**.
  - **Rudra itself** is likely in scope if it markets to EU-based users.
- **The Indian DPDP Act still applies** to the processing in India.
- **Practical rule:** apply the **strictest law among the participants' locations**, which is the standard US interstate guidance.

### B.4 How existing botless note-takers handle this

- **Granola** (bot-free; captures system audio):
  - Says audio is cached only during transcription and then **deleted** from Granola and its subprocessors. Transcripts and notes are kept (AWS US).
  - Places consent on the user and recommends verbal consent plus calendar-invite disclosure.
  - Has since added an **optional meeting-chat notice and a video "watermark"** that admins can enforce. Granola itself says these "give notice, not agreement."
  - **Sued in *Chamberlain v. Granola, Inc.*** (N.D. Cal. 3:26-cv-07926, filed 30 Jul 2026; amended complaint reportedly Oct 2026). Claims include ECPA §2511, CIPA §§631/632, CDAFA and privacy torts. The allegations centre on invisible capture, marketing that boasted "other people in the room won't know it's there," and training on transcripts by default, with the opt-out held only by the account holder.
- **Otter.ai:** *In re Otter.AI Privacy Litigation*, No. 5:25-cv-06911-EKL (N.D. Cal.).
  - The motion to dismiss was **granted in part and denied in part on 13 Aug 2026**. The Wiretap Act, CIPA §631 and BIPA claims proceed on the theory that Otter is a **third-party eavesdropper because it uses recordings for its own purposes (training)**. The CFAA and CDAFA claims were reportedly dismissed.
  - Class certification motion is due 13 May 2027.
  - Exact claim-by-claim outcomes should be confirmed from the order (Dkt. 68) [partly UNVERIFIED].
- **Fireflies.ai:** Illinois **BIPA** suits (*Cruz*, filed Dec 2025; *Fricker*, Mar 2026), consolidated with a motion to dismiss pending (2026) [UNVERIFIED status].
- **Microsoft Teams:** BIPA suit over live transcription (W.D. Wash., Feb 2026) [UNVERIFIED].
- **Google (*Ambriz v. Google*, contact-centre AI):**
  - Feb 2025: CIPA claims survived under the "capability" test.
  - Later rulings, including the Ninth Circuit's *Popa*, narrow standing [UNVERIFIED final status].
- **Industry pattern:**
  - Terms of service make the **user responsible for consent**, but those terms cannot bind non-users.
  - Vendors delete audio and keep transcripts.
  - Vendors are retreating toward visible notice options.
  - No court has yet ruled on the merits.

### B.5 Responsible design for Rudra: safeguards that keep it invisible to the client but not lawless

**Core principle:** Rudra can be invisible as a tool, but the *recording* must not be covert where the law requires consent. A bot is not needed. The user can obtain consent once per client relationship, through a clause in the contract or SOW, an email, or a verbal agreement at kickoff. Rudra should make obtaining and recording that consent easy.

1. **Per-client consent register (the key feature)**
   - Every client record has a status: `consent: none | verbal | written (contract/email) | declined`, plus jurisdiction, date and evidence (a link or note).
   - Supply a ready-made **contract clause and email template**, for example: "Calls may be recorded and transcribed using AI tools for note-taking; recordings are deleted after transcription."
2. **Jurisdiction-aware gating**
   - Infer the client's location from the CRM or calendar timezone, or ask once.
   - For **all-party jurisdictions** (the CA/FL/WA/IL/PA/MA/MD/MT/NH group, DE/MI/NV/OR/CT treated as all-party, DE-Germany, UAE, NSW, EU generally):
     - block capture unless consent is recorded, **or** require an explicit per-call attestation;
     - default to **notes-only mode**, which transcribes only the user's mic channel, never the remote channel. This is a strong middle ground: the user's own speech plus manual notes can still feed the LLM.
   - For one-party jurisdictions, show a lighter warning.
3. **Per-client "never transcribe" list**, plus a one-tap pause and redact control for any call.
4. **Audio minimisation**
   - Stream or upload in chunks to STT, then **delete the audio immediately** after a successful transcription (Granola's model).
   - Never persist raw audio by default. Allow opt-in retention of up to N days for re-transcription.
5. **Retention limits on transcripts**
   - Set a default TTL, for example 90 days for raw transcripts.
   - Keep LLM-derived summaries and action items longer.
   - Support user-initiated deletion per client, which also fits DPDP "purpose served" erasure.
6. **No training, and no vendor "own use"**
   - Contractually and technically, never use customer audio or transcripts to train Rudra or third-party models. Set `mip_opt_out` and zero-retention flags with STT vendors and use paid tiers only (no Gemini free tier).
   - This directly counters the *Otter* "third-party eavesdropper" theory and keeps Rudra a DPDP **processor**.
7. **No biometrics.** No voiceprints or speaker embeddings stored across calls. This avoids BIPA and SPDI "voice pattern" issues.
8. **Terms that allocate responsibility**
   - The user warrants they have obtained all required consents and will comply with local recording laws.
   - Include an indemnity, a **DPDP-compliant processor agreement (DPA)** with the user, and a GDPR Art. 28 DPA for EU users.
   - Terms reduce Rudra's risk but **do not cure** an unlawful recording, and they do not bind clients.
9. **Honest marketing.** Never advertise covertness ("the client won't know"). *Granola*'s marketing copy is an exhibit in the complaint against it. Say "no bot in your meeting" instead.
10. **Data residency choice.** Offer an India-only pipeline (Sarvam STT plus an India-region LLM) for users who want it.
11. **Audit trail.** Log which consent basis applied to each capture. That evidence protects the user.

---

## Open questions for a lawyer

1. **India, participant recording:** Does a party's covert recording of a business call breach any statute (Telecom Act 2023 s.42(2)(b), IT Act) or expose them to a civil privacy claim after *Puttaswamy* and *Vibhor Garg*? Does the answer change for WhatsApp/VoIP calls compared with in-person meetings?
2. **DPDP roles:** Confirm the developer is a Data Fiduciary and Rudra a Data Processor. What must the processor contract contain? Does any LLM post-processing (summaries, insights) make Rudra a fiduciary?
3. **DPDP lawful ground:** Can covert capture rely on s.7 "legitimate uses", or is s.6 consent with notice mandatory? What notice language suffices when consent is obtained through a client contract clause?
4. **DPDP scope for foreign clients:** Does s.17(1)(d) (or any other exemption) apply to foreign clients' data processed in India? How do the transition timeline (May 2027) and SPDI Rules "voice pattern" biometrics apply until then?
5. **Choice of law:** What is the exposure of (a) the Indian user and (b) Rudra (an Indian company) when the client is in California, Germany or the UAE? Could Rudra be sued in the US as the "third-party eavesdropper" even though the user is the recorder?
6. **Contract-clause consent:** Is a one-time contract or SOW clause sufficient consent under CIPA §632, Fla. §934.03, RCW 9.73, §201 StGB and UAE Art. 44? Or is per-call notice needed, and must it reach every participant, including client colleagues who never signed?
7. **Notes-only mode:** Does transcribing only the user's own mic channel avoid recording-law issues entirely? Is incidental bleed of client audio into the mic a problem?
8. **GDPR:** Does Rudra fall under Art. 3(2)? Does it need an EU representative (Art. 27) and a DPIA? Is legitimate interest viable for transcription, or must consent be used?
9. **Terms and indemnity:** Are user warranties and indemnities enforceable against Indian sole proprietors? How should liability caps be structured?
10. **Product positioning:** Does the "invisible to the client" requirement itself create foreseeable-misuse or aiding-and-abetting risk for Rudra in all-party jurisdictions? What marketing language is safe?
11. **Vendors:** Are STT subprocessors (ElevenLabs, Deepgram, Sarvam) adequate under DPDP and GDPR transfer rules? Is zero retention contractually guaranteed at the tier used?

---

## Sources

**STT: pricing and docs**
- Groq pricing: https://groq.com/pricing ; STT docs: https://console.groq.com/docs/speech-to-text ; rate limits: https://console.groq.com/docs/rate-limits ; data: https://console.groq.com/docs/your-data
- Deepgram pricing: https://deepgram.com/pricing ; MIP: https://developers.deepgram.com/docs/the-deepgram-model-improvement-partnership-program ; changelog 2026-03-05: https://developers.deepgram.com/changelog/2026/3/5
- AssemblyAI pricing: https://www.assemblyai.com/pricing.md ; opt-out: https://assemblyai.com/docs/faq/how-to-opt-out-of-data-sharing-for-our-model-improvement-program
- OpenAI pricing: https://developers.openai.com/api/docs/pricing
- ElevenLabs STT docs: https://elevenlabs.io/docs/capabilities/speech-to-text ; India residency: https://elevenlabs.io/blog/introducing-india-data-residency ; residency docs: https://elevenlabs.io/docs/overview/administration/data-residency.md ; Scribe v2 price (secondary): https://www.stork.ai/blog/elevenlabs-speech-to-text-scribe-2026
- Sarvam pricing: https://docs.sarvam.ai/api/getting-started/pricing.md ; STT overview: https://docs.sarvam.ai/api/api-guides-tutorials/speech-to-text/overview ; batch: https://docs.sarvam.ai/api/speech-to-text/apis/batch ; trust: https://www.sarvam.ai/trust-center ; developers: https://sarvam.ai/developers
- Gemini audio: https://aistudio.google.com/docs/audio ; pricing: https://aistudio.google.com/docs/pricing
- Google Cloud STT pricing: https://cloud.google.com/speech-to-text/pricing ; Chirp 3: https://docs.cloud.google.com/speech-to-text/docs/models/chirp-3 ; release notes: https://docs.cloud.google.com/speech-to-text/docs/release-notes
- Soniox pricing: https://soniox.com/pricing ; diarization: https://soniox.com/speech-to-text/features/diarization
- Bhashini (secondary): https://dev.to/mr_manushukla/22-scheduled-languages-one-app-what-indic-localization-actually-costs-to-ship-in-2026-12ag

**STT: benchmarks**
- Specific Labs: https://www.withspecific.com/benchmarks/speech-to-text
- Trelis Tara: https://huggingface.co/Trelis/tara
- AssemblyAI accuracy blog: https://www.assemblyai.com/blog/how-accurate-speech-to-text
- arXiv code-switching (May 2026): https://arxiv.org/pdf/2605.19069
- Indic DiarBench: https://arxiv.org/abs/2607.23808
- Soket AI: https://docs.tensorstudio.ai/dhrith/evaluation-and-benchmarking

**Law: India**
- Telecommunications Act 2023 s.42: https://www.advocatekhoj.com/library/bareacts/telecommunications/42.php
- *Vibhor Garg v. Neha* (SC 2025): https://www.scobserver.in/supreme-court-observer-law-reports-scolr/evidentiary-value-of-secretly-recorded-phone-calls-between-spouses-in-marital-disputes-vibhor-garg-v-neha/ ; https://www.livelaw.in/amp/top-stories/secretly-recorded-telephonic-conversation-of-spouse-admissible-in-matrimonial-cases-supreme-court-297390
- High Courts: https://ksandk.com/litigation/recording-calls-without-consent-is-a-violation-of-article-21/ ; https://www.verdictum.in/telangana-high-court/recording-calls-without-consent-spouse-breach-privacy-article-21-1617565 ; https://www.dtnext.in/amp/story/news/national/recording-phone-call-secretly-violates-privacy-chhattisgarh-hc-742167
- DPDP Act s.3: https://dpdpa.com/dpdpa2023/chapter-1/section3.html ; DPDP Rules 2025: https://www.amsshardul.com/insight/enforcement-of-the-dpdp-act-and-notification-of-the-dpdp-rules/ ; https://www.lkslaw.com/insights/articles/digital-personal-data-protection-rules-2025-incremental-tweaks-and-the-way-ahead ; Bird & Bird: https://twobirds.com/en/insights/2023/global/decrypting-indias-new-data-protection-law-key-insights-and-lessons-learned

**Law: other jurisdictions**
- US state list (secondary): https://www.kixie.com/sales-blog/what-are-the-laws-governing-call-recordings ; https://www.layer3labs.io/guides/two-party-consent-states
- *Kearney v. Salomon Smith Barney*: https://scocal.stanford.edu/opinion/kearney-v-salomon-smith-barney-33656 ; https://www.dwt.com/insights/2006/07/california-supreme-court-applies-electronicrecordi
- §201 StGB: https://www.gesetze-im-internet.de/stgb/__201.html
- UK ICO reprimand: https://ico.org.uk/about-the-ico/media-centre/news-and-blogs/2023/04/ico-reprimands-surrey-police-and-sussex-police ; UK business recording: https://www.jdsupra.com/legalnews/monitoring-and-recording-telephone-calls-98592/
- *Ryneš* C-212/13: https://curia.europa.eu/jcms/jcms/P_152318
- Canada: https://www.mondaq.com/canada/telecoms-mobile-cable-communications/1569014/can-i-record-a-conversation-in-ontario ; https://www.torontodefencelawyers.com/crime-record-conversation
- Australia: https://www.alrc.gov.au/publication/serious-invasions-of-privacy-in-the-digital-era-alrc-report-123/14-surveillance-devices/participant-monitoring/ ; https://hamiltonlocke.com.au/recording-private-conversations-the-law-in-australia/
- UAE: https://www.mondaq.com/privacy-protection/1805484/can-you-record-a-call-in-the-uae-and-use-it-in-court ; https://thelawreporters.com/understanding-privacy-laws-and-technology-misuse-in-the-uae-a-focus-on-federal-decree-law-no-34-of-2021
- Singapore: https://www.pdpc.gov.sg/-/media/files/pdpc/pdf-files/dp-starter-kit---171017.pdf ; https://singaporelegaladvice.com/can-an-organisation-record-a-conversation-without-consent/

**Litigation and note-takers**
- Otter MTD order (Dkt. 68): https://storage.courtlistener.com/recap/gov.uscourts.cand.454675/gov.uscourts.cand.454675.68.0.pdf ; docket: https://www.courtlistener.com/docket/71118721/brewer-v-otterai-inc/ ; coverage: https://uctoday.com/productivity-automation/otter-ai-fails-to-dismiss-core-privacy-claims-in-u-s-court/ ; https://www.npr.org/2026/08/15/g-s1-83087/otter-ai-transcription-class-action-lawsuit
- *Chamberlain v. Granola* (secondary): https://basilai.app/articles/2026-08-05-chamberlain-v-granola-invisible-ai-notetaker-wiretap-lawsuit.html ; https://runtimewire.com/article/granola-sued-bot-free-meeting-capture-ai-training
- Granola consent and data docs: https://www.granola.ai/blog/ai-notetaker-participant-privacy-consent ; https://docs.granola.ai/help-center/consent-security-privacy/security-privacy-data-faqs.md ; https://www.granola.ai/docs/docs/FAQs/consent-granola
- Fireflies BIPA (secondary): https://scribbl.co/post/fireflies-ai-lawsuit ; overview: https://blog.buildbetter.ai/ai-notetaker-lawsuits-2026-how-to-record-meetings-legally/
- *Ambriz v. Google*: https://www.zwillgen.com/privacy/federal-judge-allows-google-customer-service-ai-class-action-to-proceed/ ; https://www.goodwinlaw.com/en/insights/publications/2025/02/alerts-practices-dpc-ftec-ai-voice-products-subject-to-california-invasion-of-privacy-claims
