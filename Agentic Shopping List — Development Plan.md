# Agentic Shopping List

A shared household shopping and inventory system designed primarily for interaction through AI assistants.

The canonical state is stored in a shared Google Sheet. Humans normally interact with the system conversationally through agents such as ChatGPT, Gemini, Grok, Claude, or future assistants rather than manually editing a traditional shopping-list UI.

The core design principle is that household inventory is uncertain. The system should preserve observations, confidence, preferences, and intent rather than collapsing every statement into a binary checked/unchecked shopping-list item.

Examples:

- "We're out of Rice Krispies." → strong evidence that Rice Krispies should be purchased.
- "I used up a bottle of ketchup." → evidence that ketchup inventory decreased, but not necessarily evidence that the household is out.
- "We usually buy ketchup at FoodMaxx." → persistent store preference.
- "We have tons of detergent." → evidence against buying detergent even if it appears on a shopping list.
- "Detergent is unusually cheap at Costco this week." → possible reason to increase stock even though it is not currently needed.

The AI is therefore both the primary input interface and the interpreter of household state.

---

## Phase 1 — Define the Agent Contract

Develop a canonical introductory instruction that can be given to any compatible AI assistant.

This should describe:

- what the shopping sheet represents;
- how the agent should read it;
- how and when the agent may modify it;
- how observations differ from facts;
- how uncertainty should be represented;
- how explicit human intent outranks inference;
- how aliases and duplicate products should be handled;
- how store preferences should be maintained;
- how conflicting observations should be resolved;
- what data must never be silently deleted;
- how agents should behave when they are unsure.

The contract should strongly favor **preserving information over prematurely interpreting it**.

For example, the agent must not translate:

> "I used up a bottle of ketchup."

into:

> "Ketchup: OUT"

unless other evidence supports that conclusion.

Likewise, it should not convert:

> "We might need cereal."

into a definite purchase requirement.

### Data-health rules

Initial rules should include:

1. Never invent quantities, dates, purchases, consumption, or preferences.
2. Distinguish explicit human statements from agent inference.
3. Prefer uncertainty over false precision.
4. Do not silently delete historical observations.
5. Preserve the original human message for important state-changing observations.
6. Prefer updating an existing canonical item over creating a duplicate.
7. Never merge ambiguous products without sufficient evidence.
8. Treat explicit corrections from humans as authoritative.
9. Do not overwrite stronger evidence with weaker inference.
10. Keep the Sheet understandable and repairable by a human without requiring the AI.

### Deliverables

- `AGENT_INSTRUCTIONS.md`
- Initial Google Sheet schema
- A small corpus of example conversations and expected mutations
- A data-health test suite consisting of adversarial/ambiguous statements

Example test:

```text
Existing state:
Ketchup: probably 2 bottles

Human:
"I finished the ketchup."

Bad:
inventory = 0

Acceptable:
record an observation that one currently-used ketchup container was exhausted;
re-evaluate inventory based on existing evidence;
do not claim zero remaining unless supported.
```

---

## Phase 2 — Design the Google Sheets Data Model

Start deliberately simple.

A likely initial workbook contains two principal sheets.

### `Items`

One row per canonical household item.

Possible columns:

```text
item_id
canonical_name
aliases
preferred_stores
inventory_state
inventory_confidence
estimated_quantity
quantity_unit
desired_minimum
desired_normal
explicitly_requested
last_observation
last_observation_at
last_purchase_at
notes
updated_at
updated_by
```

Not every field needs to be populated.

The system should explicitly allow:

```text
estimated_quantity = blank
inventory_state = probably_low
inventory_confidence = 0.6
```

rather than forcing an artificial quantity.

### `Events`

Append-only household observations.

```text
event_id
timestamp
actor
item_id
event_type
quantity
unit
raw_message
agent_interpretation
source
```

Possible events include:

```text
consumed
purchased
opened
ran_out
observed_low
observed_plenty
requested
request_cancelled
store_preference
correction
deal_observed
```

The event log should be considered more authoritative than the derived `Items` state.

`Items` is effectively a materialized view maintained by agents for convenience.

This gives the system a recovery mechanism when an agent makes a bad inference: later agents can inspect the underlying observations and reconstruct current state.

### Design goal

The schema should contain **just enough structure to make multi-agent cooperation reliable** without attempting to encode every piece of household knowledge into relational fields.

Natural-language fields are acceptable because another agent will ultimately interpret the data.

---

## Phase 3 — Establish the Sharing and Authorization Model

Determine how a household creates one shared workbook and safely exposes it to multiple independently operated AI assistants.

Research and document at least:

- ChatGPT + Google Sheets/Drive
- Gemini + Google Sheets/Drive
- Grok + Google Sheets/Drive
- Claude + Google Sheets/Drive, if practical
- generic MCP/API-based agents

Questions that need definitive answers:

- Can an agent be authorized to only one Sheet?
- If not, what Google OAuth scopes are required?
- Can multiple Google accounts be connected to one assistant?
- Can a dedicated household Google identity provide useful isolation?
- Which assistants can mutate Sheets versus merely read them?
- How are concurrent writes handled?
- How are credentials revoked?
- What happens if one household member changes AI providers?
- Can a shared Sheet be addressed reliably by URL, ID, name, or folder?
- Do providers retain cached copies of Sheet contents, and under what policies?

The preferred architecture should avoid requiring every household member to expose an entire personal Google Drive merely to access the shopping list.

One candidate model:

```text
Household Google identity
          │
          │ owns
          ▼
  Shopping workbook
      ▲          ▲
      │          │
 Household    Household
 member A     member B
      │          │
  Agent A      Agent B
```

Another candidate:

```text
Member A Google account ──┐
                          ├── Shopping workbook
Member B Google account ──┘
        ▲                         ▲
        │                         │
     Agent A                   Agent B
```

Document the security and convenience tradeoffs before choosing one.

### Deliverable

`SETUP.md`

It should allow a technically ordinary household to create and share the system from scratch without understanding OAuth terminology.

---

## Phase 4 — Create the Human Interaction Tutorial

The system should require essentially no command syntax.

Create a short tutorial demonstrating the kinds of statements agents are expected to understand.

### Inventory observations

```text
"We're out of Rice Krispies."

"I opened the last toothpaste."

"We have plenty of toilet paper."

"I used up one bottle of ketchup."

"I think we're getting low on milk."
```

### Shopping intent

```text
"Add dishwasher detergent."

"Don't buy bananas this week."

"We need cereal, but it isn't urgent."

"Get two ketchup bottles next time."
```

### Preferences

```text
"We usually get ketchup at FoodMaxx."

"Buy paper towels at Costco."

"Don't buy the Costco salsa again."

"I prefer Rao's if it isn't much more expensive."
```

### Purchases

```text
"I bought two gallons of milk."

"I got three boxes of cereal at FoodMaxx."

"I bought the detergent."
```

### Planning

```text
"I'm heading to FoodMaxx. What should I know?"

"Anything we definitely need at Costco?"

"What should I check before I leave?"

"What are we probably getting low on?"

"Anything worth stocking up on?"
```

The tutorial should emphasize that users do **not** need to maintain the spreadsheet or phrase statements in a particular syntax.

### Deliverable

`USAGE.md`

Keep the initial tutorial short enough that a household member will actually read it.

---

## Phase 5 — Build a Behavioral Test Harness

Before adding integrations, establish whether different agents interpret the same household state compatibly.

Maintain example Sheet snapshots plus prompts such as:

```text
"We used the last open ketchup bottle."
```

Expected behavior should describe constraints rather than exact wording:

```text
MUST:
- append an event
- preserve the original observation
- reconsider ketchup state

MUST NOT:
- claim the household has zero ketchup unless prior state supports it
- remove previous purchase history
- create another "Ketchup" item if one already exists
```

Test against every supported agent.

This becomes especially important because the system intentionally relies on heuristic interpretation rather than deterministic application logic.

Track results by:

```text
agent
model
date
test
pass/fail
notes
```

Agents that cannot maintain data health should be documented as unsupported.

---

## Phase 6 — Make Store Trip Briefings Excellent

The first major user-facing feature should be:

> "I'm heading to FoodMaxx. What do I need to know?"

The agent should classify relevant items into concepts such as:

```text
Definitely buy

Probably buy

Check before leaving

Optional / stock-up

Do not buy
```

The response should combine:

- explicit requests;
- known inventory;
- probable shortages;
- store preferences;
- recent purchases;
- uncertainty;
- household consumption patterns;
- later, pricing and promotions.

An important principle is that the agent should explain useful uncertainty rather than hide it.

Example:

```text
Definitely:
- Rice Krispies — explicitly out.

Check first:
- Ketchup — one bottle was recently finished, but remaining stock is unknown.

Probably skip:
- Paper towels — recently reported as plentiful.
```

That output is more useful than pretending every item belongs on a deterministic shopping list.

---

## Phase 7 — Add Purchase and Consumption Learning

Over time, allow the system to infer rough household consumption patterns.

For example:

```text
Milk
typical purchase interval: ~5 days

Dishwasher detergent
typical purchase interval: ~75 days

Ketchup
irregular / insufficient evidence
```

These should remain probabilistic observations rather than hard schedules.

A user asking:

> "Anything else we might need?"

could then receive:

```text
You haven't said you're low on milk, but you're around the point when you
normally buy it. Worth checking before leaving.
```

Agents must distinguish this from:

```text
Milk is out.
```

Explicit observations always take precedence.

---

## Phase 8 — External Deal Sources

Once household inventory handling is reliable, begin adding external pricing information.

External systems should preferably write **deal observations**, not mutate household inventory directly.

For example:

```text
event_type = deal_observed
item = Kirkland Dishwasher Tablets
store = Costco
regular_price = 19.99
current_price = 14.99
discount = 5.00
valid_until = ...
source = Costco
```

Then the agent decides whether the deal matters.

This separation is important:

```text
External scraper:
"detergent is cheap"

Household state:
"we probably have enough detergent for another month"

Agent:
"This isn't needed yet, but it's unusually cheap and you regularly use it.
Consider stocking up."
```

### Costco

Initial Costco integration should investigate:

- warehouse inventory;
- current warehouse price;
- Instant Savings/promotions;
- receipt history;
- historical household purchase prices;
- preferred warehouse;
- warehouse-specific availability.

Eventually it should support:

```text
"I'm going to Costco."
```

and answer based on both necessity and opportunity.

Possible reasoning:

```text
Paper towels:
inventory uncertain
frequently purchased
$5 below usual price
→ stock-up candidate

Rice:
known plenty
$3 discount
→ suppress

Dishwasher detergent:
probably low
$6 discount
→ high priority
```

### Other stores

Design the deal interface so Costco is merely one provider.

Potential future sources:

```text
FoodMaxx
Safeway
Target
Walmart
Trader Joe's
Amazon
local grocery chains
```

Each source should normalize observations into a common promotion format rather than teaching household agents store-specific schemas.

---

## Phase 9 — Introduce a Normalized Deal Interface

Once multiple sources exist, define a simple canonical representation:

```text
deal_id
item
store
location
current_price
normal_price
discount_amount
discount_percent
starts_at
ends_at
availability
source
source_confidence
observed_at
```

Deals should have expiration.

Unlike household observations, promotion data should normally be replaceable/cacheable rather than permanently accumulated in the primary event history.

---

## Phase 10 — Recommendation Heuristics

Do not initially attempt to build a deterministic recommendation engine.

Document the factors agents should consider:

```text
explicit purchase request
known stockout
probability inventory is low
usual consumption interval
preferred store
current promotion
magnitude of discount
historical purchase price
normal household purchase quantity
recent purchase
known excess inventory
explicit "don't buy"
```

A conceptual model may be useful:

```text
purchase desirability =
    need
  + expected near-term consumption
  + deal quality
  + store suitability
  - known inventory
  - recent purchasing
  - explicit suppression
```

But agents should not be required to expose or literally calculate such a score.

The desired behavior is good judgment rather than false mathematical precision.

---

## Phase 11 — Only Build a Backend If Sheets Actually Becomes the Limitation

Google Sheets should remain the canonical backend until concrete problems emerge.

Possible reasons to eventually replace or augment it:

- unacceptable concurrent-write conflicts;
- API quotas;
- latency;
- inability to enforce data integrity;
- agent integrations behaving unreliably;
- excessive event volume;
- need for server-side computation;
- need for fine-grained permissions.

Do not build infrastructure merely because a traditional software architecture would normally contain a database.

If Sheets works, its advantages are significant:

- managed durability;
- automatic version history;
- backup/recovery;
- authentication;
- ACLs;
- collaborative editing;
- human-readable data;
- mobile UI;
- no server maintenance;
- wide AI-provider support.

---

# Repository Structure

A reasonable initial repository:

```text
agentic-shopping/
│
├── README.md
├── DEVELOPMENT.md
├── AGENT_INSTRUCTIONS.md
├── SETUP.md
├── USAGE.md
│
├── schema/
│   ├── items.md
│   ├── events.md
│   └── deals.md
│
├── tests/
│   ├── README.md
│   ├── inventory-observations.md
│   ├── ambiguous-language.md
│   ├── corrections.md
│   ├── conflicting-agents.md
│   └── shopping-briefings.md
│
└── integrations/
    ├── costco/
    │   └── README.md
    └── README.md
```

`README.md` should explain the product.

`DEVELOPMENT.md` should contain this roadmap.

`AGENT_INSTRUCTIONS.md` is effectively the executable behavioral specification of the application.

That file may ultimately be the **most important source file in the repository**.

---

# Initial MVP

The first usable release should deliberately contain no deal scraping and almost no application code.

A successful MVP is:

1. A Google Sheet containing `Items` and `Events`.
2. A mature `AGENT_INSTRUCTIONS.md`.
3. Two humans can give two independent AI agents access to the same Sheet.
4. Either person can casually say things like "we're almost out of ketchup."
5. Either person can later ask "I'm going to FoodMaxx, what should I know?"
6. Both agents produce substantially equivalent and useful behavior.
7. Neither agent corrupts or unnecessarily duplicates shared state.
8. A human can open the Sheet and understand or repair everything.
9. Google Sheets version history can recover from a destructive agent mistake.

If those nine things work, the central idea has been validated.

Everything after that—Costco prices, receipts, sale feeds, household consumption estimation, recommendation scoring—is enhancement rather than prerequisite.

---

# Immediate Development Order

The next work should happen in this order:

**Milestone 0.1 — Schema**

Define `Items` and `Events` precisely enough to create the first Sheet.

**Milestone 0.2 — Agent instructions**

Write the first complete agent contract, including ambiguous-input and data-integrity rules.

**Milestone 0.3 — Manual single-agent experiment**

Use one real household for several days through one AI provider. Record failure modes rather than trying to anticipate all of them.

**Milestone 0.4 — Multi-agent experiment**

Have a second independent agent operate on the same Sheet and determine where conventions break down.

**Milestone 0.5 — Setup documentation**

Document working ACL/OAuth setup for ChatGPT, Gemini, Grok, and any other supported agents.

**Milestone 0.6 — Store briefing**

Make "I'm going to X" consistently produce an excellent actionable summary.

**Milestone 0.7 — External data**

Only after the conversational inventory model is trustworthy, begin the Costco integration.

The project should optimize first for **frictionless capture and trustworthy shared state**. Deal intelligence becomes extremely valuable only after those two things work.