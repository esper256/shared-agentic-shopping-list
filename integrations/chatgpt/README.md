# ChatGPT on Android — Household Access Plan

**Status:** Research / recommended plan  
**Date:** 2026-09-11  
**Audience:** The household member who already has Grok Bot working via a Google service account, and wants a ChatGPT Pro user on Android to share the same Shopping Database.

This is the ChatGPT-specific slice of Development Plan Phase 3 / Milestone 0.5. It is not yet a wife-facing setup tutorial.

---

## Recommendation in one paragraph

ChatGPT cannot hold the Grok service-account key itself, so the closest copy of the Grok pattern is a **tiny Cloud Run adapter** in the same GCP project: it uses that service account against Sheets and speaks **MCP** to ChatGPT. Household traffic fits Cloud Run’s always-free compute quota with room to spare, but it is **not a no-billing-account product** — Google still requires a billing account and card, and a few misconfigurations (minimum instances, leftover container images) are how people get surprise invoices. The harder problem is the ChatGPT **client**: custom MCP is Developer Mode, officially **web-not-mobile**, with conflicting docs on whether personal Pro can **write**, and Android bugs that approve a tool then never send `tools/call`. Build the adapter only if you accept a GCP billing relationship kept at $0 by quotas and a budget alert, **and** you prove on her Pro account that ChatGPT will actually invoke write tools from a client she will use.

---

## What already works (Grok) and why ChatGPT is different

Grok Bot is a **robot identity**:

```text
Shopping workbook
        ▲
        │ shared as Editor
        │
Google service account  ← Sheets API credentials
        ▲
        │
     Grok Bot
```

ChatGPT has no equivalent “give this bot a JSON key” path. Native ChatGPT Google access is **the human’s Google OAuth**, through the Google Drive plugin:

```text
Shopping workbook
        ▲
        │ shared as Editor with her Google account
        │
  Her Google account
        ▲
        │ OAuth (Drive / Docs / Sheets / Slides)
        │
ChatGPT Google Drive plugin
        ▲
        │
  Her ChatGPT Pro account (Android)
```

That is a different trust model, a different tool surface, and a different mobile story. The rest of this document is about picking the least-fragile version of that second diagram.

---

## Constraints that decide the architecture

These are current as of 11 September 2026. OpenAI has been moving this surface quickly; re-check before building anything.

| Constraint | Why it matters |
|---|---|
| **No new Custom GPTs on personal Pro** | OpenAI help center: personal accounts (Free, Go, Plus, Pro) cannot create or publish new GPTs. Existing GPTs remain usable/editable. The old “make a Household Shopping GPT with Actions” recipe is closed unless she already has a GPT, or the household moves to ChatGPT Business. |
| **Drive-in-Library editing is web-first** | August 2026 release notes: Plus/Pro can open Docs/Sheets/Slides beside chat and, when authorized, write back to the original file. Official wording: rolling out **on the web**; “Mobile support will follow.” |
| **Developer Mode / custom MCP is web-only** | Custom MCP apps are not available on the ChatGPT mobile apps. Developer Mode is a developer UX (confirm writes, pick tools), not a wife UX. Some help articles also limit Pro custom MCP to read/fetch outside Business/Enterprise/Edu. |
| **ChatGPT cannot use the Grok service account directly** | There is no “paste this JSON key into ChatGPT.” A service-account path requires middleware ChatGPT can call. |
| **Generic Sheets tools are a worse fit than Grok’s API access** | The schema’s hot path is `read Config+Items → append Event → update Item`. A generic Drive/Sheets connector may dump tabs, rewrite timestamps, or confirm every write. |
| **Android Custom GPT Actions have a long bug history** | Even before new GPT creation was removed, Actions often failed to invoke from the Android app while working on desktop web. |
| **The product vision is “talk to your AI,” not “open the spreadsheet”** | The ChatGPT-for-Google-Sheets **sidebar add-on** lives inside Sheets, has a separate chat history, and historically does not work in the Google Sheets Android app. It is the wrong primary UX. |

Schema v0.1 explicitly deferred a custom backend. That is still right **until** a connector cannot do ordinary range reads/writes reliably. ChatGPT-on-Android is the first case that may trip that trigger (`GOOGLE_SHEETS_SCHEMA.md` §15: unreliable agent integrations, context bloat, missing transactional operations).

---

## Options, ranked for *her phone*

### 1. ChatGPT Project + Google Drive plugin — **try this first**

**Wife UX:** Open the pinned Project in the Android ChatGPT app. Say “we’re out of ketchup” or “I’m heading to FoodMaxx.”

**Setup (once, probably on the web together):**

1. Share the Shopping Database with her Google account as **Editor** (human share, not the service account).
2. On ChatGPT web, signed into **her** Pro account: Plugins / Apps → connect **Google Drive**. Review the Google consent screen; this is the privacy tradeoff (below).
3. Create a **Project** named something she will actually tap, e.g. `Shopping`.
4. Put `AGENT_RUNTIME_INSTRUCTIONS.md` in Project instructions, plus:
   - the workbook URL and spreadsheet ID;
   - that the Drive file named whatever you called it **is** the Shopping Database;
   - `written_by = ChatGPT`, `actor =` her name;
   - persist routine observations without being asked;
   - ISO-8601 timestamps as Plain text; never Sheets date serials.
5. Add the workbook as a Project source if the UI offers “paste a Google Drive link.”
6. On her phone: pin the Project; optionally enable Voice in that Project.

**Why this is the default:**

- Projects exist on Android today.
- Project sharing exists for Pro on web/iOS/Android, so you can draft instructions and she can use them.
- No server to run.
- Matches the MVP: two humans, two agents, one sheet.
- Project instructions are the consumer replacement for “a Custom GPT that only does shopping.”

**Risks:**

- Drive-in-Library’s **visual editor** is web-only. The unknown is whether the Android app can still **call Drive/Sheets actions** (search file, read ranges, write ranges) without that panel. That is the experiment in the next section.
- Google’s ChatGPT OAuth is not “this one spreadsheet.” ChatGPT can reach files that Google account can already access. Mitigation: connect a **household Google account** that only exists to own/share this workbook, not her personal Gmail Drive.
- ChatGPT may ask her to confirm writes. If every “we’re out of milk” needs a tap, kitchen use will fail. Test this explicitly.
- Generic spreadsheet edits can violate `INV-01` / timestamp rules. Instructions help; they do not enforce.
- Concurrent writes with Grok are the same as any other second agent: event-first, no shared `next_event_id` counter.

### 2. ChatGPT in Chrome / home-screen shortcut — **best fallback if the app cannot write**

If the Android **app** cannot mutate the sheet, the **website** is the officially supported Drive editor surface.

**Wife UX:** Home-screen icon that opens `chatgpt.com` (or the Project URL) in Chrome / a PWA, already signed in.

Worse than the native app (no Advanced Voice in the same way, easier to wander out of the Project). Still zero infrastructure, and it uses the same Drive plugin she already connected. Prefer this over building a backend if she will tolerate it.

### 3. Cloud Run MCP adapter on the Grok service account — **closest to Grok; acceptable if GCP stays at $0**

This is the architecture ChatGPT is recommending, and it is the one that actually mirrors Grok:

```text
Shopping workbook
        ▲
        │ already shared with the Grok service account
        │
Same Google service account
        ▲
        │ Sheets API (batchGet Config+Items, append Events, update Items)
Cloud Run  (scale-to-zero, streamable HTTP MCP at /mcp)
        ▲
        │ HTTPS + bearer token (not raw unauthenticated)
ChatGPT Developer Mode connector
        ▲
        │
  ChatGPT Pro  — web today; Android is the open risk
```

Expose **protocol tools**, not the raw Sheets API:

```text
get_snapshot()           → Config + Items     readOnlyHint: true
append_events([...])     → Events rows
update_items([...])      → narrow Item field updates
get_recent_events(...)   → cold path only     readOnlyHint: true
```

The façade owns ISO-8601 timestamps so ChatGPT cannot emit `9/10/26 2:15 PM`. It hard-codes one `spreadsheetId`. ChatGPT never sees Google.

**Do not** point ChatGPT at:

- `https://sheets.googleapis.com` with her Google OAuth (`spreadsheets` scope is huge and LLM-hostile);
- a Google Apps Script `/exec` URL (ChatGPT historically eats the 302 HTML interstitial and reports `ResponseTooLarge`). Apps Script is also a poor MCP host (no streamable HTTP/SSE).

Details on staying at $0, and on whether ChatGPT will actually *call* this from her phone, are in [GCP adapter: free tier vs billing headache](#gcp-adapter-free-tier-vs-billing-headache) and [Will ChatGPT Pro call this MCP?](#will-chatgpt-pro-call-this-mcp).

### 4. ChatGPT Business Custom GPT Actions — **only if MCP writes are blocked on Pro**

If Developer Mode on personal Pro turns out to be read/fetch-only (OpenAI’s help center and developer docs currently disagree), the same Cloud Run process can speak **OpenAPI** instead of MCP and be consumed as Custom GPT Actions. That still requires ChatGPT Business (or an existing pre-August GPT you can still edit). Do not buy Business seats to solve hosting; buy them only if Pro cannot write.

### 5. Rejected as primary UX

| Approach | Why not |
|---|---|
| ChatGPT for Google Sheets add-on | Sidebar inside desktop Sheets; separate history; no Memory; not “talk to ChatGPT.” |
| Raw Sheets API as a Custom GPT Action | Can’t create the GPT on Pro; OAuth scopes too broad; API too general; Android Actions flaky. |
| Zapier / Make | Extra failure domain; not protocol-aware; confirmation and latency worse. |
| Upload a CSV of the sheet into a Project | Snapshot, not live shared state. Violates “the database is durable; conversation memory is not.” |
| Switching her to Grok | She uses ChatGPT. The whole point of this project is mixed-agent households. |

---

## The experiment to run before writing any code

Do this on **her** Android phone, official ChatGPT app, Pro account. You can prepare the Project on the web.

**Prep**

- [ ] Sheet shared with her Google user as Editor.
- [ ] Google Drive plugin connected on her ChatGPT (web is fine for the OAuth).
- [ ] Project created with runtime instructions + spreadsheet URL.
- [ ] A throwaway item name reserved for the test, e.g. `ZZZ ChatGPT Probe`.

**On the Android app, inside that Project**

1. **Read:** “What do we currently believe about milk?”  
   Pass = it used live `Items` (names/states that match the sheet, not a hallucinated list).
2. **Write:** “We’re out of ZZZ ChatGPT Probe.”  
   Pass = new Events row + Items row/fields in the actual Google Sheet, with `written_by` identifiable as ChatGPT, ISO-8601 timestamps, no invented quantity.
3. **Briefing:** “I’m heading to FoodMaxx. What do I need to know?”  
   Pass = uses current `Items`, does not dump `Events`.
4. **Friction:** Count confirmation taps. Kitchen-viable is zero or one per message, not per cell.
5. **Voice (optional):** Same write via Voice. Pass = she would actually do this while cooking.

**If 1–3 pass in the app:** ship this. Write `SETUP.md` for ChatGPT around the Project. Treat Drive-scope isolation (household Google account) as a follow-up, not a blocker.

**If reads work and writes do not:** repeat 2 in Chrome on the same phone.  
- Chrome works, app doesn’t → home-screen web shortcut as the ChatGPT client.  
- Neither works → façade / Business GPT, or wait for mobile Drive.

**If it writes but corrupts timestamps or duplicates items:** keep the Project, tighten instructions, and consider a façade that owns serialization.

Record the ChatGPT model name, app version, and date. That is Milestone 0.4 evidence, not just setup.

---

## Privacy: do not connect her whole personal Drive if you can avoid it

The Development Plan’s preferred isolation is a household Google identity that owns the workbook. ChatGPT’s Drive plugin cannot be scoped to one file the way a service account can.

Practical options, best first:

1. **Household Google account** used only for this workbook (and maybe a shared calendar later). She signs into ChatGPT’s Google Drive plugin with *that* account, not her personal Gmail. The shopping sheet lives in that Drive; her personal files never enter the OAuth grant.
2. **Share the sheet with her personal Google account**, accept that ChatGPT’s grant follows whatever Google shows on the consent screen for that account. Revoke in Google Account → Third-party access if you abandon ChatGPT.
3. **Service-account façade** (option 3 above): ChatGPT never talks to Google. Closest to Grok; needs a host ChatGPT can call from Android.

Grok’s service account should stay. Do not replace it with her OAuth; the two identities can coexist as Editors on the same workbook.

---

## Performance notes (why a façade eventually wins)

The schema is already designed for cheap agent turns. A good ChatGPT connector does:

```text
READ   Config + Items     (one batch)
WRITE  append Event(s)
WRITE  update affected Item fields
```

A generic Drive/Sheets plugin often does extra `list` / `get metadata` / whole-tab reads, then writes cells with the wrong type. That costs latency, tokens, and data health.

For a household `Items` sheet of a few dozen rows, native Drive is still probably *fast enough* if it writes correctly. Optimize for **reliability and zero taps**, not API quota. Google’s 60 writes/minute/user is irrelevant here.

If we do build a façade, put it in the **same GCP project as the Grok service account**, lock it to one `spreadsheetId`, and keep credentials off the phone.

---

## GCP adapter: free tier vs billing headache

Household shopping will not exhaust Cloud Run compute. The billing headache is **having a Google Cloud billing relationship at all**, plus a short list of knobs that take you off the free tier.

### What “free” actually means

Official [Google Cloud Free Program](https://docs.cloud.google.com/free/docs/free-cloud-features) (checked 2026-09-11):

- A **billing account is required**, even to use Always Free. New accounts usually attach a card via Free Trial ($300 credit), then convert to a paid billing account. Linking a card is not the same as being charged, but it **is** a billing relationship: overages invoice automatically.
- Cloud Run Always Free (request-based billing, quoted against `us-central1` Tier 1): **2 million requests / month**, **180,000 vCPU-seconds**, **360,000 GiB-seconds** of memory, **1 GB** North America egress. Shared across every project on that billing account.
- Idle scale-to-zero with **minimum instances = 0** and **request-based billing** (CPU only while serving) incurs **no compute charge**.

A kitchen-and-store-trip household might generate a few dozen MCP calls a day. That is thousands of requests a month, not millions. A 128–512 MiB service that runs for a second or two per call is nowhere near the CPU/memory seconds. Snapshot JSON is tens of kilobytes, not the 1 GB egress cap.

So: **yes, this workload fits the free compute quota.** It does not fit a “I refuse to give Google a card” constraint.

### How people accidentally pay

These are the real invoice paths, not request volume:

| Misconfiguration | Why it bills |
|---|---|
| **Minimum instances ≥ 1** | Instance never scales to zero; you pay 24/7. |
| **Instance-based billing** (“CPU always allocated”) | Charged for idle time after a request, not only while serving. |
| **Leaving old images in Artifact Registry** | Cloud Run source deploys store images. Always Free storage is **0.5 GiB-month per billing account**. Repeated `gcloud run deploy --source` without deleting old tags is the classic surprise. |
| **Cloud Build overage** | Source deploy uses Cloud Build. **2,500 free build-minutes / month** (promotional, default pool). Fine for rare deploys; not for a CI firehose. |
| **Wrong region** | Free-tier discount is applied at `us-central1` Tier 1 rates. Deploy there. |
| **Public unauthenticated URL** | Bots hitting `/mcp` still count as requests. Unlikely to hit 2M, but it is a reason to require a bearer token. |
| **Other products in the same billing account** | Always Free is **per billing account**, not per project. A forgotten VM elsewhere eats the same pool. |

Cloud Run’s own pricing page is explicit: Build and Artifact Registry are **not** included in Cloud Run’s free tier.

### Guardrails if we deploy this

Keep the service boring:

```text
region:            us-central1
memory:            512Mi or less
cpu:               1
min-instances:     0
max-instances:     1
billing:           request-based (CPU throttled when idle)
auth:              public URL + application bearer token
                   (ChatGPT cannot mint Google identity tokens)
service account:   the existing Grok Sheets identity, or a clone with
                   only spreadsheets scope on this one workbook
```

Also:

1. Create a **budget of $1** (or $0 if the UI allows) with email alerts at 1%, 50%, 90%. That is the anti-headache control. Google will not refuse charges for you unless you add extra automation to disable billing.
2. After each deploy, delete old Artifact Registry images in `cloud-run-source-deploy` so storage stays under 0.5 GiB.
3. Do not turn on Cloud SQL, Load Balancing, a custom domain on a forwarding rule, or minimum instances “to avoid cold start.” A 1–3s cold start on the first shopping message of the day is fine.
4. Log at warning, not debug-of-every-Sheets-payload, so Logging stays trivial.

There is **no** Cloud Run path that avoids a billing account. Firebase Spark also will not host this: Cloud Functions need Blaze (a billing account) the same way.

Apps Script would avoid GCP billing, but it is the wrong protocol host for MCP and a known-bad ChatGPT Action endpoint. Skip it.

### What I would still verify before writing code

1. The GCP project that already owns the Grok service account: does it **already** have a billing account? If Grok only needed a service account + Sheets API, it may not. Enabling Cloud Run is what forces the card.
2. On **her** ChatGPT Pro web session: Developer Mode → add a hello-world MCP → can a **write** tool run, or is Pro limited to `search`/`fetch` as the help center claims?
3. Same connector from the **Android app** and from **Chrome on the phone**. Community reports: Android constructs the approval UI, she taps Allow, `tools/call` never hits the server.

If (2) is read-only, a free Cloud Run box does not help her update the list. If (3) fails and she will not use ChatGPT in Chrome, same conclusion.

---

## Will ChatGPT Pro call this MCP?

OpenAI currently publishes two stories:

- [Developer Mode docs](https://developers.openai.com/api/docs/guides/developer-mode): Plus and Pro, **on the web**, full MCP client, **read and write**, write tools confirm unless `readOnlyHint` is set. Auth: OAuth, none, or mixed; token auth is also described in connector setup.
- [Help center](https://help.openai.com/en/articles/12584461-developer-mode-and-full-mcp-apps-in-chatgpt-beta): MCP apps **not on mobile**. “Full MCP is only available to Business and Enterprise/Edu… Pro users can connect MCPs with **read/fetch** permissions in developer mode.”

Treat writes-on-Pro as **unproven until tested on her account**. Treat Android as **unproven and currently hostile** (separate Apps SDK threads: `initialize` / `tools/list` succeed, `tools/call` never leaves the phone).

Wife UX if it only works on web: Developer Mode from the `+` menu, connector selected, confirm writes (once per conversation if she taps “remember”). That is closer to Grok’s identity model than Drive OAuth, and worse than Grok’s “just talk” UX. A Chrome home-screen shortcut to ChatGPT web is the realistic phone client until OpenAI ships custom MCP on the Android app.

Do not ask her to live in Developer Mode if native Drive-in-Project already writes from the Android app. The adapter is for **Grok-parity access control and protocol tools**, not for its own sake.

---

## What I would not do yet

- Do not add Apps Script “for ChatGPT” as the HTTP or MCP endpoint.
- Do not publish a public GPT Store / plugin just so two people can buy milk.
- Do not move the canonical store off Sheets because ChatGPT’s client is awkward. Sheets is still the right household database; ChatGPT is the awkward client.
- Do not set Cloud Run minimum instances to dodge cold start.
- Do not enable Developer Mode on her account until a write-capable MCP hello-world has been proven on **your** Pro/web session, or you have accepted Chrome-on-phone as her client.

---

## Open questions

These change the recommendation:

1. Does the GCP project that owns the Grok service account **already have a billing account**? Enabling Cloud Run is what forces a card if it does not.
2. Is attaching a Google Cloud billing account with a **$1 budget alert** an acceptable “billing relationship,” or is any card-on-file a deal-breaker?
3. On her ChatGPT Pro **web** session, can Developer Mode actually **write**, or only `search`/`fetch`?
4. Will she use ChatGPT in **Chrome on the phone** if the Android app will not send MCP `tools/call`?
5. Does she already have a **Custom GPT** created before ~16 August 2026 that we can still edit?
6. How do write-confirmation dialogs feel to her — deal-breaker, or fine once per conversation?

---

## Suggested next step

Two proofs, in this order, before implementing the adapter:

1. **ChatGPT Pro write test (no GCP).** On web, enable Developer Mode, add any trivial remote MCP (even a public echo server or a local tunnel), and try a tool *without* `readOnlyHint`. If that is blocked, Cloud Run will not help her update the list on personal Pro.
2. **Android vs Chrome.** Repeat that write from the official Android app and from Chrome on the same phone.

If writes work on a client she will actually open, Cloud Run on Always Free with the guardrails above is a reasonable Grok-shaped adapter: same service account, protocol tools, no personal Drive grant. If writes only work on desktop web, decide whether a Chrome home-screen shortcut is good enough before spending time on the service.

If the write test fails on Pro entirely, stop. The next paid lever is ChatGPT Business for Custom GPT Actions, not a bigger GCP bill.
