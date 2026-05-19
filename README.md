# n8n Workflows — Datarails

This repo documents n8n workflows running on Datarails' internal n8n instance.

---

## Upsell Agent

A HubSpot-triggered workflow that analyzes a Datarails customer, scores their likelihood of being open to an upsell, and posts a structured recommendation to a Slack channel for the CSM to action.

### What it does (in plain English)

1. **HubSpot sends a `company_id`** to an n8n webhook when its internal workflow flags a customer as worth evaluating.
2. **n8n pulls everything it knows** about that company from HubSpot — the company record, top contacts, and open/recent deals.
3. **A JavaScript scoring engine** combines intent signals (G2, Warmly, HubSpot tracked page views), renewal proximity, recent senior hires, and open Salesforce/HubSpot opportunities into a heat score.
4. **Hard-blocked accounts are dropped** (Inactive, Voided, On Hold). Scores below 40 are skipped silently.
5. **An AI Agent (Gemini 2.5 Flash)** reviews the full customer picture, the heat-score breakdown, and the company's previous run, then returns a structured JSON recommendation: which product to pitch, why now, the leading "4C" theme, the best contact to approach, a one-line opener, and a confidence level.
6. **A Slack signal lands in `#upsell-agent`** with the recommendation rendered as a Block Kit message, plus 👍 / 👎 feedback buttons.
7. **HubSpot is updated** so the company record reflects when it was last signaled, what was pitched, and a one-line outcome summary.

### Architecture

```
                                    REAL-TIME PATH (any time)
                                    --------------------------

HubSpot Workflow ──► Webhook (HubSpot) ──► Validate Payload ──► Has companyId?
                                                                        │
                       ┌───────────────────────────────────────────────┘
                       ▼
       HubSpot: Get Company → Search Contacts → Search Deals
                       │
                       ▼
       Build Score Input → Score Engine (JS) → Score ≥ 40? (gate)
                       │
                       ▼
       Build Agent Bundle → AI Agent ◄── Gemini Chat Model (OpenAI compat)
                                    ◄── Datarails_Fetch_URL tool
                       │
                       ▼
       Parse Agent Output → Skip / Low-confidence? (gate)
                       │
                       ▼
       Store in Queue (Code, $workflow.staticData)  ◄── webhook path
                                                       TERMINATES here.
                                                       No Slack, no writeback.
                                                       Queue grows during week.


                                    MONDAY 9AM ET PATH
                                    --------------------

Schedule Trigger (cron 0 9 * * 1) ──► Read Queue & Top 30 (Code)
                                              │
                                              │ sorts pending by score desc,
                                              │ slices top 30,
                                              │ clears those entries,
                                              │ emits 0–30 items.
                                              ▼
                                      Format Slack Blocks ──► Slack: Post  ──► #upsell-agent
                                                                       │
                                                                       ▼
                                                            HubSpot: Update Company
                                                                  (writeback)
```

### Why two paths

The workflow runs the **full scoring + agent reasoning real-time** whenever HubSpot's internal Workflow posts a `company_id`. But instead of posting to Slack immediately and spamming CSMs throughout the week, the result is **queued** in `$workflow.staticData`. Every Monday at 9am ET, a Schedule trigger reads the queue, picks the top 20–30 by heat score, posts those to Slack as a digest review for AMs, and writes back to HubSpot. Old queue entries auto-expire after 7 days.

This split gives CSMs a **predictable Monday-morning review session** rather than unpredictable real-time pings, while still using fresh agent reasoning (done at signal time, not on Monday).

### Stages explained

**Trigger & validation (real-time path).** HubSpot posts `{ "company_id": "<id>" }` to the webhook. `Validate Payload` extracts the ID; `Has companyId?` routes empty payloads to a no-op.

**Data pull (HubSpot only).** Three HTTP calls in series, all using the shared HubSpot App Token credential:
- `HubSpot: Get Company` — ~50 properties including journey stage, status fields, G2 buyer-intent fields, Warmly fields, HubSpot intent fields, owned-product flags (`fp_a_active`, `month_end_close_active`, etc.), renewal date, and the upsell-history properties (`last_upsell_alert_*`, `upsell_alert_count_lifetime`).
- `HubSpot: Search Contacts` — filtered by `associatedcompanyid = companyId`, returns up to 7 contacts with title, demo flags, pricing-visit flag, and engagement history.
- `HubSpot: Search Deals` — filtered by `associations.company = companyId`, returns up to 50 deals with closed/won state, dates, and dealnames.

**Scoring (JavaScript).** `Build Score Input` merges company + contacts + deals into a single bundle. `Score Engine` runs a weights-table-driven scorer (G2 stage, Warmly recency/touched/intent-keyword/sessions/visitor-count/pages, HubSpot intent pageviews/visitors, renewal window, new senior hire, open SF opps, recent closed-won penalty) and applies a customer-health multiplier (1.2 for Active+Nurture, 0.7 for Trial/Kickoff, 1.0 otherwise). Hard-block check on Inactive / Voided / On-Hold runs first. Output: `{ score, hard_blocked, owned_products, missing_products, top_contacts, contacts_text, sf_blocked_product, sf_open_opp_count, sf_recent_won_count, breakdown, health_multiplier }`.

`Score ≥ 40?` is a simple IF gate. Below threshold → silent no-op. At or above → continue.

**AI reasoning (Gemini).** `Build Agent Bundle` is a Set node that re-shapes the scored data into the field names the agent prompt expects. The `AI Agent` node (n8n LangChain) is wired to:
- A **chat model** (`Gemini Chat Model`) — actually an `OpenAI Chat Model` node pointing at Google's OpenAI-compatibility endpoint (see "Why Gemini via OpenAI Chat Model" below).
- A **tool** (`Datarails_Fetch_URL`) — an HTTP Request Tool that the agent can invoke mid-reasoning to fetch a datarails.com page via Jina Reader (`r.jina.ai/<url>`) and read it as clean markdown. The agent decides whether and when to call it.

The agent's prompt has three sections:
- A **system message** with the Datarails product catalog, the 4C framework (Consolidation, Context, Consistency, Control), upsell heuristics, customer-health rules, tool-usage rules, and a strict JSON output spec.
- A **user message** with the full account dump, scoring breakdown, opportunity list, last-run memory, and the top-5 contacts.
- The model returns JSON only, with `responseMimeType: application/json` enforced server-side.

**Parse + queue (real-time path).** `Parse Agent Output` is a JS Code node that extracts the JSON from the agent's response, validates the keys, builds a one-line `last_alert_summary` for the writeback, and computes `new_alert_count` (existing count + 1). `Skip / Low-confidence?` drops runs where the agent set `skip_reason` or returned `confidence: low`. Surviving runs go to `Store in Queue`, a Code node that pushes the full parsed bundle (companyId, company, score, agent JSON, and all derived fields) into `$workflow.staticData.pending` with a timestamp. Old entries (older than 7 days) are auto-trimmed on each write. The webhook path **ends here** — no Slack post, no HubSpot writeback in real time.

**Monday digest (schedule path).** Every Monday at 9am ET, the `Schedule (Mon 9am ET)` trigger fires (cron `0 9 * * 1`, timezone `America/New_York`). `Read Queue & Top 30` reads `$workflow.staticData.pending`, applies the 7-day TTL filter, sorts by `score` descending, slices the top 30, and removes those entries from the queue. It then emits one item per surviving company in the same shape `Parse Agent Output` produces. Each item flows through `Format Slack Blocks` → `Slack: Post message` → `HubSpot: Update Company`.

**Slack format.** `Format Slack Blocks` builds a Block Kit payload with a header, fields (motion, confidence, product, score), the why-now line, the best-contact section, the reasoning paragraph, the suggested opener, and three action buttons (👍 / 👎 / Open in HubSpot). `Slack: Post message` POSTs to `chat.postMessage`.

**Writeback.** `HubSpot: Update Company` PATCHes four properties on the company record: `last_upsell_alert_sent` (timestamp), `last_upsell_alert_product` (the recommended product), `last_upsell_alert_outcome` (one-line summary, e.g. `cross-sell MEC · medium confidence · 2026-05-19`), and `upsell_alert_count_lifetime` (incremented).

### Why staticData instead of a HubSpot property for the queue

The queue lives in n8n's `$workflow.staticData` — a workflow-scoped persistent object that survives executions and restarts. Two reasons:

- **No new HubSpot property required.** Avoids adding another custom property to the Company object just for queueing.
- **Self-contained.** The queue is fully owned by this workflow. Easy to inspect via the n8n UI (Code node output panels) and reason about.

Trade-off: the queue isn't visible to CSMs in HubSpot mid-week. If pre-Monday visibility is needed, swap to a multi-line text property like `pending_upsell_signal_json` — same logic, persistent on the company record.

### Memory model — 1 run back, lives in HubSpot

The agent sees the company's previous recommendation in its prompt under `# PREVIOUS RUN ON THIS COMPANY` — last product pitched, when, the outcome summary, and the lifetime count. If the agent is about to pitch the same product within 14 days, the prompt explicitly nudges it to either escalate (expand) or pivot to a different angle.

Memory lives on the HubSpot company record itself in four properties: `last_upsell_alert_sent`, `last_upsell_alert_product`, `last_upsell_alert_outcome`, `upsell_alert_count_lifetime`. This is portable, visible to CSMs in HubSpot, survives any n8n change, and doesn't depend on the LangChain `Simple Memory` node (which isn't available on the running n8n version). A future upgrade to multi-line text storage (last 15 runs as JSON) is straightforward if richer memory is needed.

### Why Gemini via OpenAI Chat Model

The running n8n instance is on a version that pre-dates the `Google Gemini Chat Model` LangChain node — it shows "Install this node" in the editor. Rather than upgrade n8n (which is the right long-term fix), the workflow uses n8n's `OpenAI Chat Model` node with a credential that contains:
- **API Key**: a Gemini API key
- **Base URL**: `https://generativelanguage.googleapis.com/v1beta/openai`

Google publishes an OpenAI-protocol-compatible endpoint at that URL. n8n speaks OpenAI's protocol, Google's server translates internally and routes to Gemini. No OpenAI account, no OpenAI billing, no OpenAI servers in the call path — just Google. The node is labeled "Gemini Chat Model" in the canvas for clarity; the underlying type is `lmChatOpenAi`.

When n8n is upgraded to a version that ships the native Gemini Chat Model node, this becomes a one-node swap with no other workflow changes.

### Why JavaScript for the scoring engine

The instance's Python sandbox is configured with zero allowed stdlib modules (`Allowed stdlib modules: none`), which makes even `datetime` unusable. The scorer was rewritten in JavaScript using `Date.parse` for all date math. Same weights, same hard-block logic, same output keys. The JS version is portable across any n8n setup.

### Output JSON schema (agent)

```json
{
  "motion": "cross-sell | expansion",
  "recommended_product": "FP&A | MEC | Cash | Connect | FinanceOS",
  "why_now": "1–2 sentences citing specific signals",
  "reasoning": "3–5 sentence analytical paragraph",
  "leading_c": "Consolidation | Context | Consistency | Control",
  "best_contact_email": "...",
  "best_contact_name": "First Last",
  "best_contact_title": "...",
  "best_contact_reason": "1 sentence on why this contact",
  "suggested_opener": "1–2 sentence personalized line",
  "confidence": "low | medium | high",
  "skip_reason": null
}
```

If `skip_reason` is set or `confidence` is `low`, the workflow short-circuits before Slack — the agent's own veto is respected.

### What's deferred

- **LinkedIn research tool.** Datarails doesn't have approved API access to Tavily/SerpAPI/Apollo for company research; live LinkedIn fetching via Jina was unreliable. Deferred until a vendor is approved.
- **Salesforce-backed opportunities.** The original design pulled SF Account Owner + Opportunities via SOQL. Currently uses HubSpot deals as a stand-in. Swap to a Salesforce HTTP Request node when SF OAuth credentials are wired.
- **15-run memory.** Current model stores 1 run back in standard HubSpot properties. Extending to 15 runs requires a new `recent_upsell_signals` multi-line text property on the Company object.
- **Dedup in queue.** `Store in Queue` currently appends — if a company gets 5 webhook fires in a week, all 5 entries land in the queue. Add a step that replaces the existing entry for the same `companyId` with the latest one.
- **Quiet-Monday digest.** If the queue is empty on Monday, the schedule path emits 0 items and nothing posts. Optional: post a "0 signals this week" confirmation so the channel knows the sweep ran.
- **Continue On Fail.** Slack and HubSpot writeback nodes don't have error handling — a single 4xx mid-batch halts the rest of the Monday run. Add `onError: continueRegularOutput` for resilience.
- **AM-side email drafting (Phase 2).** Auto-draft an outreach email per recommendation, sent on behalf of the assigned AM. Separate workflow, separate planning.
- **Feedback handler workflow.** The 👍 / 👎 buttons in Slack post to Slack's interactivity endpoint; a separate workflow needs to receive those, write the verdict to the `upsell_signal_feedback` HubSpot property, and (optionally) feed the verdict back into future agent prompts as ground truth.

### Credentials this workflow needs

| Node | Credential type | Notes |
|---|---|---|
| HubSpot: Get Company / Search Contacts / Search Deals / Update Company | HubSpot App Token | Single shared private-app credential |
| Slack: Post message | Slack OAuth | Bot token with `chat:write` |
| Gemini Chat Model | OpenAI API (in n8n's type system) | API key field holds the Gemini key; Base URL set to Google's OpenAI-compat endpoint |
| Webhook (HubSpot) | optional Header Auth | Shared-token header to gate incoming webhook calls |

No new credentials are created by the workflow — all reference existing n8n credential records.

---

## Repo conventions

This repo holds n8n workflow documentation. Each workflow gets a section in this README. The workflow JSON itself lives in n8n (not in git) so the live version is always source of truth.
