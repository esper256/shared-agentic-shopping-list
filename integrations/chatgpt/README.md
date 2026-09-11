# ChatGPT on Android — Household Access Plan

**Status:** Research / recommended plan  
**Date:** 2026-09-11  
**Audience:** The household member who already has Grok Bot working via a Google service account, and wants a ChatGPT Pro user on Android to share the same Shopping Database.

This is the ChatGPT-specific slice of Development Plan Phase 3 / Milestone 0.5. It is not yet a wife-facing setup tutorial.

---

## Recommendation in one paragraph

Do **not** try to copy the Grok Bot pattern (ChatGPT talking directly to the Sheets API with a service account). ChatGPT Pro cannot hold a Google service account, cannot create new Custom GPTs, and cannot yet use the Drive-in-Library editor from the Android app. The most trouble-free path that still fits a personal Pro subscription is a **ChatGPT Project** whose instructions are the shopping steward protocol, with the Google Drive plugin connected to a Google account that already has Editor access to the workbook. Validate that path on her phone this week. If Android cannot persist writes, fall back to ChatGPT’s **mobile website / home-screen shortcut** (the Drive editor is web-first). Only if that is too clumsy should we add a tiny Cloud Run façade in front of the existing service account — and even then, consuming it from the Android ChatGPT **app** currently has no clean Pro-personal hook.

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

### 3. Household Sheets façade + ChatGPT Business Custom GPT — **best Grok-equivalent, higher cost**

This is the architecture that actually mirrors Grok:

```text
Same Google service account already shared on the sheet
        ▲
        │ Sheets API (spreadsheets.values.batchGet / append / update)
Cloud Run or Cloud Function in the existing GCP project
        ▲
        │ HTTPS + API key (or OAuth)
ChatGPT Custom GPT Action  (Business/Edu/Enterprise can still create GPTs)
        ▲
        │
  Android ChatGPT app  — pin the GPT, talk normally
```

Expose **protocol tools**, not the raw Sheets API:

```text
get_snapshot()           → Config + Items
append_events([...])     → Events rows
update_items([...])      → narrow Item field updates
get_recent_events(...)   → cold path only
```

That is the performant design: one read of the hot path, small JSON, timestamps serialized in the façade so ChatGPT cannot emit `9/10/26 2:15 PM`.

**Do not** point a GPT Action at:

- `https://sheets.googleapis.com` with her Google OAuth (`spreadsheets` scope is huge and LLM-hostile);
- a Google Apps Script `/exec` URL (ChatGPT historically eats the 302 HTML interstitial and reports `ResponseTooLarge`).

**Cost / friction:** ChatGPT Business is a workspace plan (two-seat minimum, currently advertised around $20–25/user/month). That is a plan change, not a toggle on Pro. Only justify this if native Drive-on-Android is a dead end **and** she refuses the web/PWA fallback.

If she already has a **pre-August-2026 Custom GPT** on the Pro account, you may still be able to **edit** it and add Actions without Business. Ask that before paying for seats.

### 4. Custom MCP / Apps SDK on personal Pro — **not for her phone**

A remote MCP server using the service account is an excellent *engine*, and it would be the right long-term adapter if ChatGPT mobile ever hosts custom apps. Today:

- add custom MCP via Developer Mode **on the web**;
- official help: MCP apps **not on mobile**;
- write confirmations and tool-picking are developer-oriented;
- Android Apps SDK threads report failures before `tools/call`.

Build this only as a shared backend **after** there is a mobile client that can call it (Business GPT Actions, a published plugin, or future mobile MCP).

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

## What I would not do yet

- Do not add Apps Script “for ChatGPT” as the HTTP endpoint.
- Do not publish a public GPT Store / plugin just so two people can buy milk.
- Do not move the canonical store off Sheets because ChatGPT’s client is awkward. Sheets is still the right household database; ChatGPT is the awkward client.
- Do not ask her to enable Developer Mode.

---

## Open questions

These change the recommendation if answered “yes”:

1. Does she already have a **Custom GPT** created before ~16 August 2026 that we can still edit (and add Actions to)?
2. Is the workbook already shared with her as a **human** Editor, or only with the Grok service account?
3. Is she willing to complete a **Google OAuth** screen inside ChatGPT? If not, only a façade works.
4. Is a Chrome/PWA shortcut acceptable if the Android app cannot write?
5. Would you ever consider **ChatGPT Business** (two seats) to regain Custom GPTs, or is personal Pro a hard constraint?
6. Which Google account should ChatGPT use: her personal Gmail, or a household-only account?
7. How do confirmation dialogs feel to her — deal-breaker, or fine?

---

## Suggested next step

Run the Android experiment above on her phone before implementing anything. That is a one-session test and it tells us whether this household’s ChatGPT client is “Project + Drive,” “web shortcut + Drive,” or “we need a façade.”

If the experiment passes, the follow-up documentation in this repo should be a short `SETUP.md` ChatGPT section (create Project, connect Drive, pin on Android, what to say). If it fails, the follow-up is a Cloud Run snapshot/append/update API using the existing service account — still behind whatever ChatGPT client can actually call it from her phone.
