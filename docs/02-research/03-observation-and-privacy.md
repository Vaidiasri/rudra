# 03 — Observation, Pattern Discovery, Privacy & Compliance Research

**Project:** Rudra (proactive AI ops assistant, Windows-first Tauri 2 desktop agent + Chrome MV3 extension + server-side LLMs)
**Date of research:** 2026-11-18
**Method:** Live web search and official docs where they could be reached. Items marked **[UNVERIFIED]** could not be confirmed against a primary source in this session. Items marked **[MEMORY]** come from prior engineering knowledge of stable, long-documented Windows/Chrome APIs and were not re-fetched; they should be spot-checked before implementation.
**Not legal advice.** Sections flagged **⚖ Lawyer** need review by qualified counsel (India and EU/US) before launch.

---

## Executive summary

1. **Full desktop observation is technically feasible on 8 GB machines without local ML, but only if it is event-driven and metadata-first.** The cheap, reliable core is: foreground-window and title events (`SetWinEventHook`), idle detection, process/exe identity, URL + navigation from the browser extension, and *targeted* UI Automation (UIA) reads of the focused element. Continuous screenshots + OCR is the expensive, privacy-toxic tier; Rudra should not do it by default.
2. **Global keyboard/mouse hooks (`WH_KEYBOARD_LL`) are the single biggest EDR/antivirus risk.** Commercial EDRs have flagged known-good software for calling `SetWindowsHookEx` (Carbon Black, 2025). Rudra should **never install a low-level keyboard hook** and never log keystrokes. Use WinEvents and UIA focus/invoke events instead.
3. **Chrome's 2026 policy tightening matters.** Effective **1 Aug 2026**, Chrome Web Store "Limited Use" requires data collection be *strictly necessary* for the extension's single disclosed purpose and *all* collection be prominently disclosed in-product, not just unexpected collection. Browsing-activity collection is allowed only for a prominently described user-facing feature. Rudra's extension must be designed around this.
4. **Microsoft Recall is the template for trust**, and it got there the hard way: plaintext-database backlash (2024) → opt-in, Windows Hello gating, per-snapshot encryption in a VBS enclave, sensitive-info filtering on by default, removable, and **disabled by default on managed (enterprise) devices**. Rewind, the other reference product, was acquired by Meta (Dec 2025) and its capture shut down (19 Dec 2025), a reminder that "local-first" promises die with the company.
5. **Pattern discovery can run cheaply server-side.** The academic Robotic Process Mining pipeline (Leno, Dumas et al.) is: normalise UI events → build a directly-follows graph → cut segments at loop back-edges → closed sequential pattern mining (CloFast) → rank by frequency/length/coverage/cohesion. All classic CPU algorithms; no GPU needed. LLMs are best used *after* mining, to name and explain patterns and draft automations.
6. **Rule-based on-device PII redaction is doable but leaky.** Regex + checksum recognisers (Presidio-style) and gitleaks-style secret rules catch structured identifiers; they **do not** catch names, free-text health/financial content, or secrets in unusual formats. The defence must be *data minimisation by design* (don't capture text content by default), not redaction alone.
7. **India DPDP:** Rules notified 13 Nov 2025; consent-manager rule (Rule 4) from ~13 Nov 2026; core obligations (notice, consent, security, breach notification, rights, cross-border) from **~13 May 2027**. MeitY floated shortening to 12 months in early 2026; **no final notification confirmed [UNVERIFIED]**. Cross-border is a negative-list model, with no list published yet.
8. **Team/enterprise deployment changes the legal picture sharply.** When an employer deploys Rudra, it becomes *workplace monitoring*: GDPR DPIA almost certainly required, consent is generally not a valid lawful basis for employees, German works councils have co-determination rights (BetrVG §87(1) no. 6), NY requires written notice + acknowledgment, California **AB 1331 was chaptered 30 Sep 2026** (verify final text), and the **EU AI Act** treats AI used to "monitor and evaluate performance and behaviour" of workers as high-risk (application deferred to **2 Dec 2027** by the Digital Omnibus). **Design the team product so it does not produce per-employee performance outputs for managers.**
9. **Code signing:** EV certificates **no longer give instant SmartScreen reputation** (since 2024). Since 1 Mar 2026, public code-signing certs max out at **460 days**. Azure **Artifact Signing** (formerly Trusted Signing) is the cheapest good option but Public Trust is available only to organisations in the USA, Canada, EU, UK and individuals in the USA/Canada. **An Indian company cannot use it directly today.** Budget for an OV certificate on a cloud HSM.

**Biggest risk to the "full desktop observation" idea:** not technology but *trust + distribution*. Specifically: (a) EDR/AV quarantining the agent in corporate fleets, (b) Chrome Web Store rejection under the 2026 Limited Use policy, (c) employers' legal exposure (works councils, AI Act high-risk, US state laws) making "observe everything" unsellable to teams, and (d) consumer backlash of the Recall kind if any capture is opt-out, unencrypted, or includes screenshots/keystrokes.

---

## 1. Windows activity-capture techniques

### 1.1 Technique comparison

CPU/RAM figures are engineering estimates for a Rust implementation on a typical 4-core laptop unless sourced. They have not been measured for Rudra; **benchmark in a spike**.

| Technique | Data it gives | CPU / RAM cost (est.) | Reliability across apps | Permissions | AV/EDR false-positive risk |
|---|---|---|---|---|---|
| **Foreground window + title** (`SetWinEventHook(EVENT_SYSTEM_FOREGROUND, EVENT_OBJECT_NAMECHANGE)`, `GetForegroundWindow`, `GetWindowThreadProcessId`, `QueryFullProcessImageNameW`) [MEMORY] | Active app (exe path, product name), window title, focus switches, timestamps | Negligible; event-driven, <1% CPU, few MB | Universal. Titles are rich for browsers (page title), Office (doc name), IDEs (file/project), Teams (chat/meeting name). Elevated windows can't be queried from a non-elevated process (ActivityWatch falls back to WMI for this) | None (standard user) | Very low; used by every time tracker |
| **Idle/AFK detection** (`GetLastInputInfo`) [MEMORY] | Seconds since last input (no content) | Negligible (poll every few seconds) | Universal | None | Very low |
| **UI Automation (UIA)** via COM (`IUIAutomation`), focus/structure/invoke events | Element tree: control type, name, value (text field contents!), automation ID, selected item, button invoked, URL bar value, `IsPassword` flag | Low if limited to *focused element + ancestors on events*; high (100s ms, CPU spikes, tens of MB) if you walk full trees or subscribe to all property-change events. Screenpipe caps tree walks at ~200 ms [UNVERIFIED snippet] | **Win32/WPF/WinForms/UWP: good. Office: good. Chromium browsers: native UIA default since Chrome 138** (rollout began 126). Chromium/Electron (VS Code, Slack, Teams-new/WebView2) build the accessibility tree *on demand* when an AT client is detected, which adds memory/CPU *in the target app*. Java/Qt/custom-drawn/games/remote desktop: poor | None for standard apps; can't read elevated processes from non-elevated | Low–medium. Generally benign (screen readers use it), but enumerating everything + exfiltration can trip behavioural rules |
| **MSAA / IAccessible2** (legacy accessibility) | Similar to UIA, older | Similar | Chromium keeps MSAA/IA2 alongside UIA | None | Low |
| **Low-level hooks** `SetWindowsHookEx(WH_KEYBOARD_LL / WH_MOUSE_LL)` | Every keystroke / click with coordinates | Low CPU, but hook latency affects the whole desktop; Windows silently unhooks slow hooks [MEMORY] | Universal (except elevated/secure desktop) | None | **High.** Classic keylogger signature (MITRE T1056.001). Carbon Black raised sev-7 alerts on known-good software calling `SetWindowsHookEx` (2025). **Do not use for keyboard.** Mouse-only LL hook is lower risk but still flagged by some heuristics |
| **ETW (Event Tracing for Windows)** | Process start/stop, file I/O, network, registry, image loads | Kernel providers: moderate; can be noisy | System-wide, app-agnostic, but *no UI semantics* | Most useful kernel/security providers need **Administrator** (or Performance Log Users group) [MEMORY] | Medium; legit (Sysmon-like), but an always-on admin service tracing file+network looks like an EDR/spyware |
| **Clipboard listener** (`AddClipboardFormatListener` → `WM_CLIPBOARDUPDATE`) [MEMORY] | Notification of clipboard change; reading content gives copied text/images/files | Negligible | Universal | None | Low–medium. Reading clipboard content is an infostealer behaviour; reading only *format + source app + length* is benign. **Clipboard frequently contains passwords/OTPs/keys** |
| **Windows.Media.Ocr** (WinRT `OcrEngine`) | Text + bounding boxes from a bitmap | Moderate: tens–hundreds of ms per full-screen frame on CPU; RAM tens of MB. Microsoft calls it "legacy"; newer Windows App SDK `TextRecognizer` is faster but **experimental, not supported for production** | Works on anything visible (fallback for apps with poor UIA) | Language packs must be installed; docs reference MSIX-packaged context, but unpackaged desktop use is documented separately | Low |
| **Windows Graphics Capture (WGC)** (`GraphicsCaptureItem`, Direct3D frame pool) or DXGI Desktop Duplication | Pixel frames of a window/monitor | GPU-accelerated capture is cheap; *encoding/storing/OCR* is what costs. Continuous capture at 1 fps + OCR is the heaviest tier | Universal for visible content; DRM-protected/`WDA_EXCLUDEFROMCAPTURE` windows come out black [MEMORY] | No permission prompt for desktop apps on Win10/11; Windows draws a yellow capture border (Win11 lets apps request no border via `IsBorderRequired` with access request) [MEMORY] | Medium; screen capture + upload is an infostealer pattern |

### 1.2 Reliability notes by app family

- **Browsers:** the extension is more reliable than UIA for URL/tab/navigation. UIA on Chrome now works natively (Chrome 138+), but forcing Chrome's accessibility tree on costs the browser RAM. Prefer extension data, and use UIA only as fallback for non-Chrome browsers.
- **Electron (VS Code, Slack, Discord, Notion, Teams-new via WebView2):** title is reliable; UIA works once the accessibility tree is activated, which is costly for the target process. Treat deep UIA on Electron as opt-in per app. [Electron specifics UNVERIFIED in this session]
- **Office:** Titles reliably give document names; UIA exposes ribbon commands invoked and cell/field focus. Excel cell *values* via UIA are possible but are content capture.
- **IDEs:** Titles give file + project (VS Code, JetBrains); JetBrains (Java/Swing-ish) has weaker UIA. Better: optional IDE plugins later.
- **Teams/Zoom/Meet:** title reveals meeting/chat names (often PII). Meeting presence is detectable via title/process. Don't capture audio or video.

### 1.3 What reference products actually use

| Product | Capture approach | Notes |
|---|---|---|
| **Screenpipe** | Event-driven screenshots (on app switch/click, not fixed fps); Windows input listener via `SetWindowsHookEx`; capture via DXGI/native APIs; **accessibility tree first, Windows native OCR as fallback**; local storage; claims ~5–10% CPU (vendor, some say 5–20%) | Rust. Licence described inconsistently (MIT vs source-available commercial licence); **check LICENSE before reusing code** |
| **Microsoft Recall** | Periodic screenshots on Copilot+ PCs; on-device NPU OCR + semantic index; Microsoft Classification Engine sensitive-info filter; encrypted per-snapshot in VBS enclave | Requires Copilot+ (NPU) hardware. Not reproducible on 8 GB machines without ML |
| **ActivityWatch** | Polls `GetForegroundWindow` + window title + process name (pywin32, WMI fallback for elevated processes); AFK watcher based on last input time (180 s default); separate browser extension for URLs; local-only server | Open source (MPL-2.0 [MEMORY]); good reference for the minimal tier |
| **RescueTime** | Background app collecting **window titles** of apps/sites; browser detection via window titles; Firefox extension records app name, URL, title, start/end and sends via WebSocket to the desktop app; vendor states no keystrokes or page content | Third-party claims of page-content reading conflict with vendor FAQ [UNVERIFIED] |
| **Rewind (macOS)** | Continuous screen recording + on-device OCR + audio; local-first | Acquired by Meta, Dec 2025; screen/audio capture ended 19 Dec 2025 |

### 1.4 Rust crates/libraries usable from Tauri

| Need | Crate | Notes |
|---|---|---|
| All Win32/WinRT APIs (WinEvents, UIA COM, DPAPI, OCR, WGC) | `windows` (windows-rs, Microsoft) | Feature-gated bindings; the canonical choice |
| UIA ergonomic wrapper | `uiautomation` | Walk tree, read properties, handle events |
| Screen capture | `windows-capture` (v2.x) | WGC + DXGI Desktop Duplication |
| Active window (cross-platform) | `active-win-pos-rs`, `x-win` [UNVERIFIED current maintenance] | Useful for later Mac port; on Windows, direct `windows` calls are simpler |
| Credentials | `keyring` (Windows Credential Manager backend) [MEMORY] | For refresh tokens |
| Local DB | `rusqlite` (+ SQLCipher feature) [MEMORY] | Encrypted local episode buffer |
| Regex for redaction | `regex` (linear-time, no catastrophic backtracking) | Port Presidio/gitleaks patterns; some lookarounds need `fancy-regex` |
| Updater | `tauri-plugin-updater` | Minisign/Ed25519 signatures (see §7) |

Run capture in a dedicated Rust thread with its own message loop (WinEvent hooks require one), separate from the Tauri webview.

---

## 2. Browser-extension observation under Manifest V3

### 2.1 What's possible

| Capability | API | Permission | Notes |
|---|---|---|---|
| Active tab, URL, title, tab switches, window focus | `chrome.tabs.onActivated/onUpdated`, `chrome.windows.onFocusChanged` | `tabs` (URL/title) or `activeTab` | `tabs` triggers "read your browsing history" warning [MEMORY] |
| Navigation lifecycle, SPA route changes | `chrome.webNavigation.*` (`onCommitted`, `onHistoryStateUpdated`) | `webNavigation` | Good for detecting web-app "screens" |
| DOM events (clicks, form submits, field focus) | Content scripts (`content_scripts` / `scripting.executeScript`) | host permissions (`<all_urls>` or listed) | Highest-value *and* highest-risk; broad host permissions trigger extra CWS review. Prefer **allow-listed domains** (Jira, GitHub, AWS console, Grafana, etc.) |
| Idle state | `chrome.idle` | `idle` | |
| Native app link | `chrome.runtime.connectNative` / `sendNativeMessage` | `nativeMessaging` | See 2.3 |

Network-request *bodies* are not readable in MV3 (`webRequest` blocking/body access is restricted; `declarativeNetRequest` doesn't expose content) [MEMORY]. That's fine for Rudra.

### 2.2 Service-worker lifetime limits (official docs)

- Terminated after **30 s idle**. Events and extension API calls reset the timer (Chrome 110+).
- Single event/API call > **5 min** → terminated; `fetch()` response > **30 s** → terminated.
- **Kept alive by:** an open **native-messaging port** (Chrome 105+, reconnect in `onDisconnect`), messages over long-lived ports (114+), active WebSockets (116+), alarms down to 30 s (120+).
- Globals are lost on termination; persist state in `chrome.storage`. Design the worker as stateless and event-driven, with content scripts batching events.

**Implication:** a persistent `connectNative` port to the Rudra desktop app both delivers events and keeps the worker alive. That is the intended pattern.

### 2.3 Native messaging [MEMORY, verify against current docs]

- Host registered via a JSON manifest. On Windows, the registry key is `HKCU\Software\Google\Chrome\NativeMessagingHosts\<name>` (Edge: `HKCU\Software\Microsoft\Edge\NativeMessagingHosts\<name>`), with `allowed_origins` listing the extension ID.
- Chrome launches the host exe as a child process over stdio; messages are length-prefixed JSON (≈1 MB max host→extension, larger extension→host).
- The host should be a tiny Rust binary that relays to the main Tauri app over a local named pipe with ACLs restricted to the current user. **Do not** open a localhost HTTP/WebSocket port, which any local process or web page can probe.
- Validate the caller origin (`chrome-extension://<id>`) passed as an argument, and pin the extension ID.

### 2.4 Chrome Web Store policy (critical)

- **Limited Use:** data collected/used/transferred only for the single disclosed purpose; no ads, no data brokers, no creditworthiness; human reading only with explicit consent, for security, for law, or aggregated/anonymised. This applies to derived and anonymised data too. Must publish an affirmative compliance statement on the developer website.
- **Web browsing activity:** prohibited *except* for a user-facing feature prominently described on the store page **and** in the product UI.
- **2026 update (effective 1 Aug 2026, reported by CyberInsider; confirm in the live policy text):** collection only where **"strictly necessary"** for the single purpose; analytics/"future features" collection not allowed; **all** data collection must be prominently disclosed in-product even if closely related; users must be notified of post-install changes in data handling.
- **Design consequences for Rudra:**
  - Single purpose: "Detect repetitive work patterns in your browser and suggest automations."
  - An in-extension consent screen (before any collection) listing exactly: URLs/domains, page titles, navigation, and (optional, per-site) click/field-focus events. **Never** form-field values.
  - Product analytics telemetry about the extension itself should be minimal or absent.
  - Expect manual review because of `<all_urls>`; start with `activeTab` + an allow-list.

---

## 3. Task mining and pattern discovery

### 3.1 Commercial tools

| Tool | Capture | Event→task method | Status/notes |
|---|---|---|---|
| **Celonis Task Mining** | Windows-only client: clicks, keyboard shortcuts, copy/paste, app time, optional screenshots; allow-listing of apps | Tasks defined by **start/end events** (e.g. click "Save Opportunity"); data joined into a Process Data Model and **case-matched** to system-of-record process data; "AI clustering" of topics per 2019 press release | Premium add-on licence |
| **UiPath Task Mining** | Recorder captures **screenshots + metadata per keystroke/click**, PII masking before AI analysis | ML clusters screenshots into tasks; flags frequently repeated tasks as automation candidates; shows frequent path and variants | **Unassisted Task Mining deprecated, removal 1 Dec 2025** (UiPath ES docs). Confirm current status |
| **Power Automate Process Mining (ex-Process Advisor)** | Power Automate for desktop *records* user-initiated sessions (assisted, not passive) | Recordings grouped into processes; activity grouping, variants, time analysis | Task mining recording GA and licensed; process mining parts in preview per MS Learn page |
| **Soroco Scout** | Interaction data across apps incl. VDI/mainframes; anonymised; **user opt-in**; app/URL allow-list; storage on-prem or chosen jurisdiction | Builds a **"work graph"** (graph of people↔apps↔actions); deviation/bottleneck analysis | Vendor claims unverified |
| **Skan.ai** | **Computer vision on screen** interactions; redaction at source; app include/exclude lists | ML derives process model ("digital twin") | Vendor claims unverified |

Common pattern: **allow-listed apps → event stream → normalisation into (app, screen, element, action) → segmentation into cases/tasks → clustering/variant analysis → frequency × duration ranking**. Every enterprise tool advertises redaction, allow-lists and opt-in, because buyers require them.

### 3.2 Academic work: Robotic Process Mining (Leno, Dumas, La Rosa, Maggi, Polyvyanyy)

- **"Identifying candidate routines for RPA from unsegmented UI logs"** (ICPM 2020, arXiv 2008.05782):
  1. **Normalise** UI events (abstract away values; keep app/control/action).
  2. **Segmentation:** build a control-flow (directly-follows) graph of normalised UIs; compute **dominator tree** and non-trivial **SCCs** (Kosaraju); back-edges into SCC headers mark routine boundaries (DFS heuristic for irreducible SCCs); back-edge targets = segment starts, sources = segment ends; events outside segments are discarded as noise.
  3. **Routine discovery:** **CloFast closed sequential pattern mining** (gaps allowed) over segments, min support e.g. 0.1; greedy selection of the top pattern, remove its occurrences, repeat.
  4. **Ranking:** frequency, length, coverage, **cohesion** (length minus median gap count); cohesion/length performed best.
- **"Discovering executable routine specifications from user interaction logs"** (arXiv 2106.13446): extends the above to check **automatability** and synthesise executable specs (data-transformation discovery between copy/paste fields).
- **Noise-tolerant routine-log extraction** (arXiv 2510.08118, 2025): clusters segmented executions by type, then process discovery per cluster [authorship UNVERIFIED].
- Related lines of work [MEMORY]: Agostinelli et al. (segmentation of UI logs), Bosco et al. (discovering automatable routines), Rebmann et al. (event abstraction).

### 3.3 Algorithm menu and server cost

| Stage | Algorithms | Cost | Rudra fit |
|---|---|---|---|
| Event abstraction | Rule-based mapping (app + URL pattern + control) → "activity"; optional LLM labelling of *new* abstract activities, cached | O(n), trivial | **On-device rules** produce the episodes; server uses LLM only for naming |
| Segmentation | Idle gaps / app-switch boundaries (simple); start/end events; **DFG back-edge** (Leno); change-point detection | O(n) to O(V+E) per user-day | Use idle-gap + context-switch on device; Leno-style on server per user |
| Pattern mining | **PrefixSpan**, **CloFast/CloSpan/BIDE** (closed patterns), SPADE; n-gram counting with suffix arrays for contiguous runs | Linear-ish to exponential in worst case; bounded by min-support, max length (≤20), per-user daily windows | Cheap: per-user logs are small (10³–10⁴ events/day). Batch nightly per user |
| Clustering of executions/variants | Edit distance / Levenshtein on activity sequences + hierarchical/DBSCAN; MinHash/LSH for scale | O(k²) on candidate instances; fine at per-user scale | Server, batch |
| Automatability | Copy→paste field pairs, deterministic transformations (Foofah/FlashFill-style), repeated identical inputs | Moderate | Server; LLM drafts the automation |
| Ranking | frequency × avg duration × cohesion × automatability; "time-saved" estimate | Trivial | Drives proposal UX |

**Recommendation:** an n-gram/PrefixSpan pass on abstracted episodes per user per day (Rust or Python worker) costs fractions of a CPU-second per user. LLM calls only for top-K candidates → low Gemini/Groq bill. Cross-user (team) mining should be **opt-in and aggregate**.

---

## 4. Lessons from privacy backlash

| Case | What went wrong | What rebuilt (or failed to rebuild) trust |
|---|---|---|
| **Microsoft Recall (May 2024)** | Announced on by default; snapshots + OCR text in a readable local SQLite DB; researchers built extraction tools ("TotalRecall") within days [MEMORY]; captured passwords, banking, private messages | Sept 2024 redesign: **opt-in only**; **Windows Hello** (biometric) required to view; **per-snapshot keys protected by TPM, used only inside a VBS enclave**; search runs inside the enclave; UI treated as untrusted; **sensitive-info filtering on by default** (Purview's classification engine; snapshot not saved when detected); per-app/site exclusions; **uninstallable**; third-party pen test; **disabled and removed by default on managed devices**, users can't self-enable there; GA April 2025 (Copilot+), EEA later in 2025 with export to third parties. Residual: claimed bypasses after Hello auth (disputed), filter misses content in PDFs/notes |
| **Rewind / Limitless** | Trust rested on "local-first"; after Meta acquisition (Dec 2025) capture ended and remaining users had to accept Meta's terms | Lesson: **contractual + technical guarantees must survive change of control**: export/delete tools, no-sale-of-data commitments, data portability |
| **Bossware (Hubstaff/Time Doctor-style screenshots, keystroke counts, "productivity scores"; Microsoft 365 "Productivity Score" per-user data in 2020)** | Covert or per-person surveillance, scoring individuals; regulators and unions pushed back. Microsoft removed user names from Productivity Score after criticism [MEMORY]. Laws followed (NY 52-c, CA AB 1331, EU AI Act high-risk) | Trust signals: transparency to the observed person, **the individual sees everything first**, aggregation/anonymisation for managers, no keystroke/screenshot capture, no scoring |

**Design choices that demonstrably rebuilt trust (to copy):**
1. Opt-in with a clear capture preview ("here's exactly what Rudra saw in the last hour").
2. A visible always-on indicator plus one-click pause / incognito.
3. Encryption at rest with OS-bound keys; local data viewable only after re-auth (Windows Hello via `UserConsentVerifier`).
4. Default exclusions (password managers, banking, health, private/incognito windows) plus user-editable deny-lists.
5. Sensitive-content filter that *drops* data rather than redacting it.
6. Easy uninstall and full delete, local and server-side.
7. Independent security review published.
8. In team mode: **employee-owned data**, aggregate-only manager views, no individual ranking.

---

## 5. On-device PII redaction without ML models

### 5.1 Approaches

| Layer | Technique | Catches | Misses |
|---|---|---|---|
| **Don't collect** | Capture metadata (app, control type, action), not text values; never read fields where `UIA_IsPasswordPropertyId` is true or `type=password` in DOM; skip `WDA_EXCLUDEFROMCAPTURE` windows; deny-list apps (1Password, Bitwarden, KeePass, banking) and incognito windows | Most sensitive content, by construction | Sensitive text inside titles (email subjects, doc names, chat names) |
| **Structured PII regex + validators** (Presidio-style) | Email, phone (E.164 + Indian formats), credit card (Luhn), IBAN (mod-97), IP, URLs with tokens/query params, **Aadhaar (12 digits + Verhoeff)**, **PAN (`[A-Z]{5}[0-9]{4}[A-Z]`)**, GSTIN, US SSN, UK NINO, passport patterns | High precision on structured IDs with checksums | False positives on bare digit runs without checksums |
| **Secret detection** (gitleaks/trufflehog rules) | Provider-prefixed tokens (`AKIA…`, `ghp_…`, `xox[baprs]-…`, `sk-…`, `AIza…`), JWTs (`eyJ…`), PEM private-key headers, connection strings (`postgres://user:pass@`), generic `password=`/`secret=` key-value with **Shannon-entropy threshold** | Most well-known cloud/SaaS keys (gitleaks ships a large TOML rule set; trufflehog 700–800+ detectors) | Custom/internal secrets, low-entropy passwords, OTPs. Never use trufflehog-style *verification* (live API calls) on user data |
| **Generalisation** | Replace numbers with `<NUM>`, emails with `<EMAIL@domain>`, file paths → `<USER>\…`, URL → `scheme://host/<path-template>` (keep route shape, drop IDs/query values) | Makes episodes pattern-mineable *and* less identifying | Rare tokens can still re-identify |
| **Dictionary** | Local contact names / OS username / machine name redaction; user-supplied sensitive terms | Known names | Unknown names |

### 5.2 Presidio without spaCy

- Presidio's `AnalyzerEngine` creates a spaCy NLP engine **by default** if none is supplied. Its `PatternRecognizer`s treat NLP artifacts as optional, so a regex-only registry is plausible, but **no doc confirms a supported no-NLP mode [UNVERIFIED]**. It also requires Python, which shouldn't be shipped in the Tauri agent.
- **Recommendation:** don't embed Presidio. **Port its regex + checksum recognisers and context-word lists to Rust** (`regex` crate) and gitleaks' rule TOML (MIT [MEMORY], check licence) into a compiled rule pack, versioned and updatable via signed config. Use Presidio server-side (with spaCy) as a *second-pass audit* on uploaded episodes, as defence in depth.

### 5.3 Limits (must be stated in the DPIA and privacy notice)

- Names, addresses, health/financial narrative in free text are **not reliably detected without NER**.
- Window titles leak a lot (email subject lines, Slack DM partner, doc names like "Termination – Priya.docx").
- Regex on OCR text suffers OCR errors, which break checksums.
- Redaction failure is silent; mitigate with **minimisation**, server-side second pass, retention limits and user review of uploads.

---

## 6. Legal and compliance (⚖ Lawyer for all of this section)

### 6.1 India — DPDP Act 2023 + DPDP Rules 2025

| Item | Status (as researched 2026-11-18) |
|---|---|
| Rules notified | **13 Nov 2025**, G.S.R. 846(E) (some sources say 14 Nov) |
| In force immediately | Rules 1, 2, 17–21 (Data Protection Board) |
| Consent managers (Rule 4) | **~13 Nov 2026** (12 months). Must be Indian company, net worth ≥ ₹2 crore, registered with the Board. One source says registration awaits a further notification [UNVERIFIED] |
| Core obligations (Rules 3, 5–16, 22, 23): notice, consent, security safeguards, breach notification, retention/erasure, children, rights, SDF duties, cross-border | **~13 May 2027** (18 months; exact day varies by source) |
| Proposed acceleration | MeitY consulted (Jan–Feb 2026) on cutting to **12 months** and/or accelerating SDF duties; **no final amendment confirmed [UNVERIFIED]**. A 2026 Removal of Difficulties Order made textual amendments; sources conflict on its dates. **Check the Gazette** |
| Cross-border (s.16, Rule 15) | **Negative-list** model: transfers allowed unless the government restricts a country; **no list published**. SDFs may face localisation for specified data. |
| Employment | s.7(i) "legitimate use" for employment purposes lets employers process without consent, but scope is debated (unions argue performance surveillance is outside it) |

**Obligations relevant to Rudra (as Data Fiduciary for individual users; as Data Processor for team customers):** itemised plain-language notice (in English + Eighth Schedule languages on request), free/specific/informed/unambiguous consent with easy withdrawal, reasonable security safeguards (encryption, access control, logging retained ≥1 year per Rules [MEMORY, verify]), **breach notice to the Board and affected users** (72-hour detailed report to Board [MEMORY, verify]), erasure when purpose ends, grievance officer contact, verifiable parental consent for under-18s (Rudra should exclude minors). Penalties up to ₹250 crore per breach category.

### 6.2 GDPR (EU/UK users, individuals)

- **Applicability:** Art 3(2) applies to a non-EU company offering services to people in the EU or monitoring their behaviour. Rudra literally monitors behaviour. Needs an **Art 27 EU representative** (and UK representative under UK GDPR) unless an exemption applies ⚖.
- **Lawful basis (individual users):** contract (Art 6(1)(b)) for core capture the user asked for; **consent** for optional/deeper capture (content, screenshots, clipboard) and for any model improvement. Special-category data (health, union, sex life) will incidentally appear in screens/titles, so Art 9 issues arise; minimise and filter.
- **DPIA:** EDPB/WP29 WP248 criteria include *systematic monitoring*, *innovative technology*, *data concerning vulnerable subjects (employees)*, *large scale*. Rudra hits ≥2 → **DPIA required**, certainly for the team product, and prudent for individual. [WP248 criteria from MEMORY; verify]
- **International transfers:** India has no EU adequacy decision → **SCCs + transfer impact assessment**; the sub-processors (Google Gemini, Groq: US) need SCCs/DPF.
- **LLM processors:** use only **paid API tiers whose terms exclude training on customer data** (Gemini API free tier may use data to improve products [MEMORY, verify current terms]); document zero/short retention.

### 6.3 Workplace monitoring (team/company deployments)

| Jurisdiction | Key rules | Implication for Rudra |
|---|---|---|
| **EU (GDPR Art 88 + national law)** | Employee consent generally not "freely given" → employer relies on legitimate interests/contract with balancing test; DPIA; transparency; proportionality (CJEU/ECtHR *Bărbulescu*-style notice) [MEMORY] | Customer is controller, Rudra is processor; ship a **DPIA template + DPA** |
| **Germany** | **BetrVG §87(1) no. 6:** works council co-determination over technical devices *suitable* to monitor behaviour/performance (BAG reads broadly; virtually all such software). Recent rulings: no co-determination over pure data-protection matters, but monitoring tech still is | Sales cycle requires a **works agreement (Betriebsvereinbarung)**; provide a template + technical fact sheet. Similar regimes: France (CSE consultation), Netherlands (OR consent), Austria [MEMORY] |
| **EU AI Act** | AI used to **monitor and evaluate performance and behaviour** of workers / allocate tasks = **Annex III high-risk**; Digital Omnibus defers application to **2 Dec 2027** (adopted by EP 16 Jun 2026, Council 29 Jun 2026; OJ publication to confirm). **Emotion recognition in the workplace is already prohibited** (since 2 Feb 2025) [MEMORY] | Keep team features **assistive, not evaluative**: no per-employee productivity scores, no manager dashboards of individuals. Otherwise Rudra becomes a high-risk AI provider (conformity assessment, risk management, logging, human oversight) |
| **US — New York** | Civ. Rights Law §52-c (since 7 May 2022): prior written notice at hiring, written/electronic acknowledgment, conspicuous posting; AG enforcement; $500/$1,000/$3,000 penalties | Provide notice templates; in-app employee acknowledgment flow |
| **US — Connecticut, Delaware** | Prior notice statutes for electronic monitoring [MEMORY] | Same templates |
| **US — California** | CCPA/CPRA covers employee data (since 2023); **AB 1331 chaptered 30 Sep 2026 (Ch. 851)**: restrictions on surveillance tools in employee-only areas/off-duty, advance notice, $500/employee/violation (text read from Digital Democracy; **confirm final text and effective date**). AB 1221 failed | Off-duty capture must be impossible in team mode (auto-pause outside configured hours, employee-controlled pause) |
| **US — federal** | ECPA (consent/business-use exceptions); NLRB GC memo 2022 on surveillance chilling protected activity (status under later NLRB uncertain) [MEMORY] | Avoid capturing personal chats/union activity; allow-list work apps |
| **India** | No dedicated monitoring statute; DPDP s.7(i) legitimate use; IT Act s.43A/SPDI Rules until DPDP fully operative; employment contracts/policies typically include monitoring clauses [MEMORY] | Employer notice + policy template; DPDP notice still advisable |

### 6.4 What enterprise buyers will demand, in realistic order

| Order | Item | Typical timing/cost (vendor-blog benchmarks, rough) |
|---|---|---|
| 1 | Privacy policy, ToS, **DPA** (GDPR Art 28 + DPDP processor terms), **sub-processor list**, SCCs, security whitepaper, filled questionnaires (CAIQ-lite/SIG-lite) | Weeks; legal fees |
| 2 | Pen test of agent + API by a reputable firm, published summary | US$5–20k [estimate] |
| 3 | **SOC 2 Type I**, then **Type II** (3-month first window minimum, 6–12 typical) | Type II audit ~$12k–$100k+; first Type II 6–15 months from start; plus GRC tool (Vanta/Drata) cost |
| 4 | SSO (SAML/OIDC), SCIM, audit logs, admin policy controls, data export/delete APIs | Engineering |
| 5 | **Data residency** (EU region; India region for Indian enterprises/BFSI) | Infra cost; choose LLM endpoints in-region (Vertex AI regional endpoints) |
| 6 | **ISO 27001** (+ **ISO 27701** for privacy). European and Indian buyers often prefer ISO over SOC 2 [MEMORY] | 6–12 months; reuse SOC 2 controls |
| 7 | AI governance: ISO 42001, EU AI Act documentation (if any high-risk use) | Later |

---

## 7. Security of the desktop agent

### 7.1 Code signing on Windows

| Option | Cost | SmartScreen | Eligibility for Rudra (Indian company) |
|---|---|---|---|
| **EV certificate** (HSM/token or cloud HSM) | ~$400+/yr | **No longer bypasses SmartScreen** (EV OIDs removed from MS Trusted Root program Aug 2024; Microsoft docs: "same as OV since 2024") | Yes, but poor value |
| **OV certificate** on cloud HSM (DigiCert KeyLocker, SSL.com eSigner, GlobalSign, Sectigo; keys must be in FIPS HSM since 2023) | ~$150–300/yr + cloud signing fees | Reputation built per file hash + publisher over downloads; warnings early | Yes |
| **Azure Artifact Signing** (ex-Trusted Signing) | Basic SKU (5,000 sigs/mo) / Premium (100,000); historically ~$9.99/mo Basic [MEMORY; current price not shown on page] | Microsoft-managed short-lived certs; reputation behaves like OV | **Public Trust only for orgs in USA/Canada/EU/UK and individuals in USA/Canada.** Not available to an India-only entity today. Option: sign via a US/EU subsidiary later |
| **Microsoft Store (MSIX)** | Store signs the package | No SmartScreen issue for Store installs | Yes; but Store sandboxing/MSIX may complicate global hooks, native messaging host registration and auto-start [UNVERIFIED] |

- Since **1 Mar 2026**, public code-signing certificate validity max is **460 days** (CA/B Forum ballot CSC-31), so plan for annual renewal automation. Reports say reputation may reset after renewal [UNVERIFIED].
- SmartScreen reputation is per-hash; each release starts fresh but publisher reputation helps. Mitigate by fewer, larger releases and delta updates signed by the same identity.
- **EDR friendliness:** sign *every* binary (main exe, native-messaging host, updater, DLLs); stable install path under `%LOCALAPPDATA%\Programs`; no code injection, no DLL hooks into other processes, no LL keyboard hooks, no screenshotting by default; publish an **"IT admin guide"** with hashes/publisher for allow-listing; submit binaries to Microsoft Defender's false-positive portal before releases.

### 7.2 Auto-update security (Tauri 2)

- `tauri-plugin-updater` verifies **Minisign (Ed25519) signatures**; `pubkey` embedded in `tauri.conf.json`; build signs with `TAURI_SIGNING_PRIVATE_KEY` (+ `_PASSWORD`); `createUpdaterArtifacts: true`. Old `TAURI_PRIVATE_KEY*` env names are deprecated.
- Hardening: keep the updater private key **offline/HSM or CI secret with restricted environment approvals**; **losing the key bricks updates**, so back it up in two secured places; serve `latest.json` over HTTPS from your own domain; consider key-rotation plan (ship next pubkey in advance); Authenticode-sign the update payload too (Windows checks it; SmartScreen); staged rollouts; prevent downgrade attacks (version monotonicity check) [UNVERIFIED whether the plugin enforces this by default].

### 7.3 Secure local storage

- **DPAPI** (`CryptProtectData`, user scope) [MEMORY]: wrap a random 256-bit data key; use it for **SQLCipher** episode buffer / AES-GCM files. Protects against offline disk theft and other users, **not** malware running as the same user (Recall's lesson). Optionally gate viewing of local history with **Windows Hello** (`UserConsentVerifier`).
- Tokens: store refresh token in **Windows Credential Manager** (`keyring` crate) or DPAPI blob; keep access tokens in memory only; short-lived access tokens (≤15 min), rotating refresh tokens bound to a device key (generate a non-exportable key in TPM via CNG/Platform Crypto Provider for **DPoP-style proof-of-possession**) [design recommendation]. **Never** store tokens in the webview's `localStorage`.
- Tauri: restrict capabilities/permissions per window (Tauri 2 capability files), strict CSP, no remote content in the privileged webview, IPC commands validate inputs. The Stronghold plugin is reportedly slated for deprecation [UNVERIFIED]; prefer OS keystore.
- Local IPC (native-messaging host ↔ app): named pipe with DACL limited to current user SID, reject remote clients (`PIPE_REJECT_REMOTE_CLIENTS`).

---

## Recommended observation architecture for Rudra

### Tiers (default = Tier 0 + 1; everything else opt-in per app/site)

| Tier | What | How | Cost (est.) | Default |
|---|---|---|---|---|
| **0 Context** | Foreground app (exe, product name), window title, focus switches, idle/active, session lock/unlock | `SetWinEventHook` (foreground, name-change) on a dedicated thread; `GetLastInputInfo` poll 5 s; `WTSRegisterSessionNotification` | <1% CPU, <30 MB | ON (after onboarding consent) |
| **1 Browser** | Active tab domain + URL path template, title, navigation events, tab switches | MV3 extension → native-messaging host → named pipe | Negligible | ON (with in-extension disclosure) |
| **2 Interaction structure** | For **allow-listed apps/sites**: control type + accessible name of invoked element ("Button: Create issue"), field *focus* (not value), menu/ribbon commands, copy/paste *events* with source/dest app and data type/length | UIA focus-changed + invoke events (no full-tree walks); content scripts on allow-listed domains (click target role/label, form submit) | 1–3% CPU bursts | OPT-IN per app |
| **3 Content snippets** | Selected text values in specific fields for automation (e.g. ticket IDs) after redaction | UIA ValuePattern on specific elements; DOM reads in specific selectors | Low | OPT-IN, per field rule, user-reviewed |
| **4 Visual fallback** | OCR of a *single window* on demand when user says "teach Rudra this task" | WGC window capture → Windows.Media.Ocr → redaction → discard image | 100s ms per capture | **Explicit, per-session only ("record a demo")**; never continuous |

### On-device pipeline

`raw events (memory only, ring buffer ≤ 10 min)` → **deny-list filter** (password managers, banking, health, incognito, `IsPassword`, excluded windows, user pause) → **abstraction** (app + URL template + control role/name + action) → **redaction** (Rust regex/secret pack + generalisation) → **episode builder** (segment on idle > N min / context switch; compute durations, counts) → **encrypted local store** (SQLCipher + DPAPI key, 7–30 days) → **user-visible "what will be uploaded" view** → batched upload (TLS, device-bound token) of *episodes only*.

### Server pipeline

Episodes → per-user nightly mining (n-gram/PrefixSpan + Leno-style segmentation, variant clustering) → rank (frequency × duration × cohesion × automatability) → top-K to LLM (Gemini/Groq paid tiers, no training) for naming + automation draft → proposal to user. Raw episodes retained ≤ 90 days; derived patterns until user deletes.

### Never capture

- Keystrokes / keystroke counts (no `WH_KEYBOARD_LL`)
- Password fields, password-manager windows, OTP/2FA apps
- Continuous screenshots, video, audio, webcam, microphone
- Clipboard *contents* by default (events and metadata only)
- Incognito/private browsing; browser form values
- Content of personal messaging/email bodies (titles generalised: subjects stripped)
- Anything outside working hours / when paused (team mode: employee-controlled)
- Biometric data; emotion inference (prohibited in workplaces under EU AI Act)

### Trust UX (Recall lessons)

Opt-in onboarding with live preview; tray indicator + global pause hotkey; per-app/site allow/deny lists; "Rudra's view of your last hour" timeline; delete hour/day/all; export; Windows Hello to open history; published security architecture and third-party pen test; in team mode, data owned and reviewed by the employee first, with managers seeing only aggregate, opt-in-shared automations.

---

## Compliance roadmap

### Launch (individual users, India + global self-serve)

- Privacy policy and ToS reflecting DPDP + GDPR; **DPDP-style itemised notice + consent flow** built now (mandatory ~May 2027).
- GDPR: lawful-basis mapping, records of processing, **DPIA** for the product, **Art 27 EU/UK representatives** ⚖, SCCs with Gemini/Groq/cloud host, transfer impact assessment.
- Chrome Web Store: single-purpose statement, in-product prominent disclosure, Limited Use affirmative statement on website.
- Security baseline: signed binaries (OV on cloud HSM), signed updater with key ceremony, encryption at rest/in transit, breach-response runbook (DPDP Board + GDPR 72 h), vulnerability disclosure policy.
- Age gate (18+). Grievance officer/contact (DPDP).
- Data retention schedule; deletion endpoint.

### First team customers

- **DPA** (Rudra as processor), sub-processor list with change notice, SCCs.
- Customer-facing **DPIA template**, employee notice templates (NY 52-c, CT, DE; India policy clause), **works-council fact sheet + Betriebsvereinbarung template** (DE).
- Product guardrails: no individual scoring; employee-controlled pause; working-hours window; aggregate-only admin views; per-org allow-lists; admin cannot see raw episodes.
- Admin controls: SSO (Google/Microsoft), org-level retention settings, audit logs.
- Pen test + security whitepaper; start **SOC 2 Type I → Type II** (start observation window early, ~3 months first period).
- AI Act scoping memo ⚖ confirming Rudra is not an Annex III high-risk system as designed.

### Enterprise

- **SOC 2 Type II** report; **ISO 27001 + 27701** (especially EU/India buyers).
- Data residency: EU and India regions, regional LLM endpoints; customer-managed keys (later).
- SCIM, MDM/Intune deployment package (MSI), group-policy configuration, EDR allow-listing guide, signed hashes.
- Contractual: liability caps, AI-use terms, no-training commitments, change-of-control data clause (Rewind lesson).
- If any evaluative features are ever added: EU AI Act high-risk compliance by **2 Dec 2027** (risk management, technical docs, logging, human oversight, conformity assessment) ⚖.
- DPDP: if Rudra is notified as a Significant Data Fiduciary (unlikely early): DPO in India, annual DPIA and audit.

---

## Open questions

1. **DPDP timeline:** was the proposed 12-month compression or any acceleration formally notified? Check the Gazette/MeitY. ⚖
2. **Is Rudra (for individuals) a Data Fiduciary needing a consent manager integration?** Consent managers are optional for fiduciaries as far as reviewed; confirm. ⚖
3. **Chrome policy:** confirm the 1 Aug 2026 Limited Use text on the live Program Policies page and whether `<all_urls>` + content scripts for "pattern detection" passes review. Consider a pre-submission test listing.
4. **Code signing entity:** set up a US/EU entity to use Artifact Signing, or OV via cloud HSM from India? Measure SmartScreen warm-up time with real downloads.
5. **EDR behaviour:** test the agent against Defender for Endpoint, CrowdStrike, SentinelOne, Carbon Black in a lab (WinEvent hooks + UIA + named pipe + periodic upload).
6. **UIA cost on Electron/Chromium:** measure RAM/CPU impact on VS Code/Slack/Teams when Rudra's UIA client triggers accessibility mode; decide per-app defaults.
7. **Windows.Media.Ocr in unpackaged Tauri apps:** confirm it works without MSIX/package identity on Win10/11; check language-pack availability on Indian SKUs.
8. **Gemini and Groq data terms:** confirm current no-training/retention terms on paid tiers and regional processing options (EU/India). ⚖
9. **California AB 1331:** read the chaptered text; effective date; whether "workplace surveillance tool" covers Rudra in team mode. ⚖
10. **EU AI Act:** confirm Digital Omnibus OJ publication and whether "task allocation" suggestions in team mode could fall under Annex III 4(b). ⚖
11. **Screenpipe licence:** confirm before borrowing code.
12. **Mac port:** Accessibility + Screen Recording TCC permissions; separate research.

---

## Sources

**Windows capture / reference products**
- Screenpipe architecture: https://mintlify.com/screenpipe/screenpipe/architecture ; https://screenpi.pe/about ; https://mintlify.com/screenpipe/screenpipe/introduction
- ActivityWatch window watcher: https://codeclimate.com/github/ActivityWatch/aw-watcher-window/aw_watcher_window/windows.py/source ; https://docs.activitywatch.net/en/latest/watchers.html
- RescueTime: https://help.rescuetime.com/article/45-monitoring-options ; https://addons.mozilla.org/de/firefox/addon/rescuetime-for-firefox/privacy ; https://www.rescuetime.com/features
- Rewind/Limitless → Meta: https://9to5mac.com/2025/12/05/rewind-limitless-meta-acquisition/ ; https://www.outlookbusiness.com/corporate/meta-acquires-ai-wearables-start-up-limitless-as-it-doubles-down-on-personal-superintelligence ; https://techinformed.com/?p=37614
- Chromium UIA: https://developer.chrome.com/blog/windows-uia-support ; https://developer.chrome.com/blog/windows-uia-support-update?hl=pl ; https://chromium.googlesource.com/chromium/src/+/HEAD/docs/accessibility/overview.md
- Windows OCR: https://learn.microsoft.com/uwp/api/windows.media.ocr ; https://learn.microsoft.com/uwp/api/windows.media.ocr.ocrengine ; https://learn.microsoft.com/cs-cz/windows/ai/apis/text-recognition
- Rust crates: https://docs.rs/windows-capture ; https://docs.rs/crate/uiautomation/0.6.3
- EDR / hooks: https://knowledge.broadcom.com/external/article/403731/severity-7-alerts-seen-indicating-called.html ; https://www.elastic.co/security-labs/blog/protecting-your-devices-from-information-theft-keylogger-protection ; https://www.elastic.co/guide/en/security/current/powershell-keylogging-script.html

**Browser extension**
- Service worker lifecycle: https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle
- Limited Use: https://developer.chrome.com/docs/webstore/program-policies/limited-use
- User data FAQ / policies: https://developer.chrome.com/docs/webstore/program-policies/user-data-faq.md.txt ; https://developer.chrome.com/docs/webstore/user_data/
- 2026 policy update report: https://cyberinsider.com/?p=327769
- Native messaging (reference, not re-fetched): https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging

**Task mining / RPM**
- Leno et al. 2020: https://arxiv.org/abs/2008.05782 (HTML: https://arxiv.org/html/2008.05782)
- Executable routine specs: https://arxiv.org/pdf/2106.13446
- Noise-tolerant routine extraction: https://arxiv.org/pdf/2510.08118
- Celonis: https://docs.celonis.com/en/task-mining.html ; https://www.celonis.com/process-mining/what-is-task-mining ; https://www.ciodive.com/press-release/20191009-celonis-expands-intelligent-business-cloud-platform-to-add-task-mining-to
- UiPath: https://docs.uipath.com/task-mining/docs/unassisted-task-minining-introduction ; https://docs.uipath.com/es/task-mining/automation-cloud/latest/user-guide/unassisted-task-mining
- Power Automate: https://learn.microsoft.com/en-us/power-automate/process-advisor-overview ; https://learn.microsoft.com/en-us/power-automate/process-advisor-processes
- Soroco: https://soroco.com/work-graph/
- Skan: https://www.skan.ai/process-mining-insights/observation ; https://venturebeat.com/business/skan-raises-14-million-to-automate-repetitive-business-processes-with-computer-vision

**Recall / privacy backlash**
- https://blogs.windows.com/windowsexperience/2024/09/27/update-on-recall-security-and-privacy-architecture/
- https://learn.microsoft.com/en-us/windows/client-management/manage-recall
- https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-windowsai
- https://techcommunity.microsoft.com/blog/windows-itpro-blog/ai-innovations-grounded-in-transparency-and-control/4405797
- https://tomshardware.com/software/windows/microsoft-launches-recall-to-windows-11-general-availability-click-to-do-and-improved-search-also-coming
- https://itdaily.com/news/software/windows-recall-available-europe
- https://www.itnews.com.au/news/microsoft-says-new-windows-recall-bypass-isnt-a-vulnerability-624918

**PII / secrets**
- Presidio: https://microsoft.github.io/presidio/api/analyzer_python ; https://pypi.org/project/presidio-analyzer/
- gitleaks: https://github.com/gitleaks/gitleaks
- trufflehog verification: https://docs.trufflesecurity.com/docs/configuration/detector-specific-verification/

**Legal**
- DPDP Rules: https://www.mondaq.com/india/data-protection/1708164/digital-personal-data-protection-rules-2025-notified ; https://www.barandbench.com/view-point/meity-notifies-final-digital-personal-data-protection-rules-2025 ; https://yourstory.com/2025/11/digital-personal-data-protection-dpdp-rules-2025-rollout-over-18-months ; https://ssrana.in/articles/meity-notifies-final-digital-personal-data-protection-rules-2025/
- DPDP timeline compression proposal: https://www.mondaq.com/india/data-protection/1773554/meity-plans-to-cut-short-dpdp-compliance-timeline-and-notify-cross-border-restrictions-for-sdfs ; https://www.storyboard18.com/amp/digital/meity-seeks-industry-views-on-fast-tracking-dpdp-act-rollout-proposes-12-month-compliance-timeline-88332.htm ; https://www.india-briefing.com/news/dpdp-removal-of-difficulties-order-2026-consent-data-audit-47274.html/
- DPDP cross-border / employment: https://www.barandbench.com/columns/the-dpdp-cross-border-transfer-rules-arent-live-yet-so-why-are-contracts-being-redrafted-as-if-they-are ; https://chambers.com/articles/navigating-the-legitimate-use-exemption-for-employee-data-under-the-digital-personal-data-protection ; https://elplaw.in/wp-content/uploads/2026/05/Processing-employee-related-data-Where-are-the-potential-boundaries.pdf
- Germany works councils: https://www.luther-lawfirm.com/en/newsroom/blog/detail/dauerbrenner-software-vs-mitbestimmung-87-abs-1-nr-6-betrvg ; https://www.goerg.de/en/insights/publications/05-11-2025/no-mandatory-co-determination-in-relation-to-data-protection
- New York §52-c: https://newyork.public.law/laws/n.y._civil_rights_law_section_52-c*2 ; https://www.mofo.com/resources/insights/211115-new-york-enacts-employee-monitoring-notification-law
- California AB 1331 / AB 1221: https://calmatters.digitaldemocracy.org/bills/ca_202520260ab1331 ; https://legiscan.com/CA/bill/AB1221/
- EU AI Act Digital Omnibus: https://knowledge.dlapiper.com/dlapiperknowledge/globalemploymentlatestdevelopments/2026/The-Digital-AI-Omnibus-Proposed-deferral-of-high-risk-AI-obligations-under-the-AI-Act ; https://cdp.cooley.com/digital-ai-omnibus-delays-key-deadlines-introduces-new-rules/ ; https://natlawreview.com/article/eu-nears-approval-agreement-delay-rules-ai-use-employment-decisions
- GDPR DPIA guidelines WP248 (not re-fetched): https://ec.europa.eu/newsroom/article29/items/611236
- SOC 2 timelines/costs: https://drata.com/grc-central/soc-2/type-1-vs-type-2 ; https://blog.getagency.com/articles/soc-2-statistics-definitive-roundup ; https://iomergent.com/soc-2-for-startups/

**Code signing / updater**
- Artifact Signing: https://learn.microsoft.com/en-us/azure/artifact-signing/overview ; https://azure.microsoft.com/en-us/pricing/details/artifact-signing/ ; https://learn.microsoft.com/azure/artifact-signing/faq ; https://learn.microsoft.com/en-us/answers/questions/2243504/trusted-signing-for-other-countries
- SmartScreen / EV change: https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options ; https://www.todesktop.com/blog/posts/windows-apps-psa-ev-certs-do-not-grant-immediate-reputation-anymore ; https://learn.microsoft.com/en-us/answers/questions/1846647/program-requirements-microsoft-trusted-root-progra
- 460-day validity: https://www.digicert.com/blog/understanding-the-new-code-signing-certificate-validity-change ; https://sslinsights.com/code-signing-certificate-validity-reduced-460-days/
- Tauri updater: https://v2.tauri.app/plugin/updater/ ; https://docs.rs/crate/tauri-cli/latest/source/src/signer/sign.rs
