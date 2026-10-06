---
name: expansion-opportunities
description: >
  Read-only expansion reporting for a HubSpot Solutions Partner: surface which
  of the partner's clients are worth an upsell or cross-sell — including clients
  maxed out on seats, credits, or contacts, sitting on unactivated tools, or
  simply thriving with room to grow — ranked by value, plus (on request) the
  reasoning for a fit and suggested outreach copy. Use for viewing,
  prioritizing, or drilling into a partner's expansion, upsell, or cross-sell
  opportunities — e.g. "which clients can I sell more to", "who's maxed out on
  seats", "best Sales Hub opportunities", or "which clients should I approach
  about AI agents, and why". Do not use for creating or editing deals,
  updating records, or sending outreach or contacting customers — it drafts
  copy but only reads and reports. The deep-dive companion to book-pulse.
---

# Expansion Opportunities

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. It goes beyond the ML signal count to surface specific, actionable expansion opportunities per client.

**Before anything else:** check for PARTNER_CLIENT read access. If more than one HubSpot connector is attached, run the check on each and use the one whose portal has PARTNER_CLIENT read access; name that portal (account ID) in your first line of output. If no connector has access, do not query — say the partner client data isn't accessible in this connection, ask the user to connect a Solutions Partner portal or grant the required scope, and stop.

## Phase 1 — Read the active book

Read the partner's **active** clients, the PARTNER_CLIENT records where `hs_is_active` is true, with the full expansion property set below. The report classifies and ranks across the whole active book (only the *rendered* lists are capped, see Phase 3).

**Scalar fields for all active clients**:

```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_hubs,
       hs_marketing_hub_mrr, hs_sales_hub_mrr, hs_service_hub_mrr,
       hs_content_hub_mrr, hs_operations_hub_mrr, hs_commerce_hub_mrr,
       hs_unified_usage_score, hs_last_4_weeks_usage_score_trend,
       hs_marketing_hub_usage_score, hs_sales_hub_usage_score,
       hs_service_hub_usage_score, hs_commerce_hub_usage_score,
       hs_core_seats_assigned, hs_core_seats_limit,
       hs_sales_seats_assigned, hs_sales_seats_limit,
       hs_service_seats_assigned, hs_service_seats_limit,
       hs_commerce_hub_seats_assigned, hs_commerce_hub_seats_limit,
       hs_marketing_contacts_limit, hs_marketing_contacts_usage,
       hs_credits_limit, hs_credits_usage,
       hs_included_hubspot_credits_monthly_limit, hs_additional_hubspot_credits_monthly_limit,
       hs_credits_spent_monthly_limit, hs_credits_overage_setting,
       hs_marketing_hub_tools_to_activate, hs_sales_hub_tools_to_activate,
       hs_service_hub_tools_to_activate, hs_revenue_signals
FROM PARTNER_CLIENT WHERE hs_is_active = true
ORDER BY hs_total_subscription_mrr DESC
```

If you classify and report before the book is fully covered, **state the partial coverage explicitly** in the report header (see Phase 3) and offer "analyse more clients" in Phase 4 — never present a partial scan as if it were the whole book.

If the query tool supports `verbosityLevel`, set it to `MEDIUM` so each record comes back with its URL — the default can omit links, and the Phase 3 client links depend on it.

Inspect the pagination instructions in the tool response (it reports how many of the total were returned and how to fetch the rest) and follow them to continue retrieving until the whole book is covered.

If no SQL query tool is available, use whatever CRM read tools are available with the same property set and scoping, and use the record URL each tool returns.

> `hs_next_renewal_date` is not used here: it doesn't roll forward after a renewal passes, so it can't tell you how close a renewal is. Renewal timing belongs to renewal-radar.

### On-demand text

The two long fields are fetched **only when needed**, scoped to the specific clients in question by `hs_object_id`:

- Fetch `hs_revenue_signal_explanation` when the request asks for **reasoning** — *why* a client is a fit, *what* to sell, or *which* clients fit a given motion (e.g. "which clients should I approach about AI agents, and why?"). Also fetch it, without being asked, for clients whose only renewal-type signal needs a direction check (see Phase 2A).
- Fetch `hs_revenue_signal_positioning` **only** when the request asks for **outreach or messaging** — how to pitch, what to say, draft the email. Do not pull it just because you pulled the explanation.

Select only the field(s) needed, scoped by `hs_object_id` (e.g. `WHERE hs_object_id IN ('…', '…')`). If a fetched field comes back empty for a client, treat it as unavailable; never bulk-fetch to find out.

> `hs_revenue_signal_explanation` is ML-generated text (sometimes HTML) and can be long — it is the authoritative *why the signal fired* and *what the opportunity is*. Render it when a reasoning request pulls it.

> `hs_revenue_signal_positioning` is sales-enablement text, suggested outreach messaging. Render it only when an outreach request pulls it.

## Phase 2 — Classify opportunities

For each client, compute opportunity signals across five dimensions:

### A. ML Revenue Signals

Two fields carry the ML signal, and they play **different roles** — don't conflate them:

- **`hs_revenue_signals`** — a semicolon-separated list of signal *categories*. Use it for:
  1. **Existence** — `hasMLSignal` = `hs_revenue_signals` is present and non-empty (treat null or `""` as no signal). This is the only thing that tells you an ML signal fired, and it is the same definition book-pulse uses.
  2. **Category** — which *kind* of signal fired, for grouping and routing (see table).
- **`hs_revenue_signal_explanation`** — the **authoritative** source for the specific **hub**, **direction** (cross-sell vs. upsell vs. retention risk), and **tier movement** (e.g. "Pro → Enterprise"). It is **not** fetched for the standard report, except for the direction check below. Use it to refine the opportunity type **only after** it has been fetched; otherwise classify from the category labels alone.

The categories below are a **non-exhaustive guide** — HubSpot adds values over time, so surface any unlisted value as-is.

| Category | What it tells you |
|---|---|
| `Predicted Cross-Sell` / `Predicted Upsell` | Predicted expansion; the actual hub and direction come from the explanation |
| `AI SQL` | AI-identified Sales Qualified Lead (SQL) surfaced from a prior support-bot conversation |
| `Customer Agent` / `Data Agent` / `Prospecting Agent` | Breeze agent adoption signal |
| `Credits Limit` | Hitting the credits limit, needs more credits |
| `Upcoming HubSpot Renewal` / `Upcoming Competitor Renewal` | **Direction unknown** — can be expansion at renewal *or* retention risk. Needs the direction check below |
| `SFDC Compete Campaign` | Salesforce competitive campaign |
| `Seats Whitespace` | Seat expansion opportunity |

**Direction check for renewal signals.** For a client whose categories include `Upcoming HubSpot Renewal` or `Upcoming Competitor Renewal`, fetch `hs_revenue_signal_explanation` for those clients only before listing them as expansion. If it mentions retention risk, churn risk, declining usage, or low adoption, set `renewalRisk = true`: leave the client out of the ML REVENUE SIGNALS section and the opportunity counts, and list it under **RENEWAL RISK — SEE RENEWAL-RADAR** instead. Other signals or capacity findings for that client still count. If the explanation is empty, list the client under ML signals with the note `renewal signal — direction unconfirmed`.

By default, describe the opportunity type from the category labels alone (e.g. `Predicted Cross-Sell`, `Credits Limit`) — do not name a specific hub or tier the labels don't give you. Only when the explanation has been fetched for that client, refine the type with the hub, direction, and tier movement it states.

### B. Capacity Pressure (computed from seat/credit/contact properties)

Even when the ML signal is silent, capacity pressure signals an expansion opportunity. Only evaluate a comparison when the limit field is present and greater than zero, and when the usage field it needs is present. Otherwise treat that capacity dimension as **data unavailable** — do not flag it. The thresholds (≈90% seats/contacts, 80% credits, ≥70 usage) are heuristic defaults chosen to surface clients before they hit hard limits — they are not product limits. Honor a sensitivity the partner states (e.g. "flag anyone above 75%").

- `seatsMaxed`: for each hub (core, sales, service, commerce), when `seats_limit` > 0 and `seats_assigned` is present: flag if `seats_assigned >= seats_limit * 0.9` OR `seats_limit - seats_assigned <= 1`.
- `creditsMaxed`: when `hs_credits_limit` > 0 and `hs_credits_usage` is present: flag if `hs_credits_usage >= hs_credits_limit * 0.8`. Also flag if `hs_credits_spent_monthly_limit` is non-empty and `hs_additional_hubspot_credits_monthly_limit` > 0 (already buying additional credits, may need more).
- `contactsMaxed`: when `hs_marketing_contacts_limit` > 0 and `hs_marketing_contacts_usage` is present: flag if `usage >= limit * 0.9` OR `limit - usage <= 50`.
- `hasCapacitySignal`: `seatsMaxed` OR `creditsMaxed` OR `contactsMaxed`.

### C. Adoption Gaps (from `tools_to_activate`)

- `hasAdoptionGap`: any of `hs_marketing_hub_tools_to_activate`, `hs_sales_hub_tools_to_activate`, `hs_service_hub_tools_to_activate` is non-empty. These list tools the client has access to but hasn't activated — an adoption gap that may benefit from training (partner service opportunity) or a different edition (expansion).

### D. Usage Momentum (from usage score + trend)

A client actively getting value is a receptive expansion target:
- `hasUsageMomentum`: `hs_unified_usage_score` ≥ 70 OR `hs_last_4_weeks_usage_score_trend` > 0 OR **any single hub** usage score (`hs_marketing_hub_usage_score` / `hs_sales_hub_usage_score` / `hs_service_hub_usage_score` / `hs_commerce_hub_usage_score`) ≥ 70. Strongest when overall usage is high *and* rising.
- **Don't gate on the unified score alone** — a low overall score can mask a hub the client leans on heavily. That thriving hub is the expansion anchor. Note which hub is driving momentum.

### E. Opportunity classification

- `hasOpportunity`: `hasMLSignal` (excluding signals moved to renewal risk) OR `hasCapacitySignal` OR `hasAdoptionGap`.
- `isWarmOnly`: `hasUsageMomentum` and not `hasOpportunity`. A warm *proactive-outreach* candidate framed as "expand into a thriving account," not a specific gap. Warm-only clients are **not** counted in "clients with opportunities"; they are counted as "thriving."
- `opportunityTypes`: collect all matching types (e.g. `["ML: Predicted Cross-Sell", "Capacity: Seats Maxed (Sales)"]`); only sharpen ML types to a specific hub/tier when the explanation was fetched for that client.
- `priority`: rank by MRR descending; use `hasUsageMomentum` as an amplifier — a warm, growing account outranks a flat one at similar MRR. Clients without MRR sort last.

### F. Hub-specific requests ("best [hub] opportunities")

When the partner names a hub:
- **Cross-sell candidates** = clients whose `hs_all_active_hubs` does not include that hub. Rank by MRR, with momentum in any hub as the tiebreaker. Note any ML `Predicted Cross-Sell` signal (and, if fetched, whether its explanation names that hub).
- **Upsell candidates** = clients who have that hub and carry a `Predicted Upsell` signal, capacity pressure on that hub's seats, or an adoption gap in that hub.
- If no client lacks the hub, say so and show only the upsell list. If `hs_all_active_hubs` is empty for a client, leave it out and count it in a one-line "hub data missing for [N] clients" note.

## Phase 3 — The expansion report

**Scope to the request first.** The full report below is the default for an open-ended "show me expansion opportunities." If the request is narrower, answer in the smallest shape that fits and omit the rest:
- a specific motion or signal ("which clients for AI agents / Breeze", "who's maxed out on seats") → filter to those clients and render only the relevant section(s);
- a hub ("best Sales Hub opportunities") → Phase 2F, rendered as CROSS-SELL and UPSELL sections;
- one client ("tell me about [client]'s expansion fit") → just that client's opportunities and reasoning;
- a count or yes/no ("how many have cross-sell signals?") → answer directly, optionally with the list.

Rules:
- Render as real markdown — **not inside a code fence**, so client links work. The block below shows layout only.
- Sort by MRR descending within each group.
- **Omit any section with no items — no placeholders, no "N/A".**
- Cap each list at 5; if more, add `+ N more`.
- MRR as `$#,###/mo`. If MRR is empty, show `MRR not reported`.
- `[products]`: the client's active hubs from `hs_all_active_hubs`, abbreviated; if more than 3, show the first 3 + `+N`. Omit if empty.
- Render the explanation or positioning text **only when the current request pulled it** (the renewal direction check pulls it but doesn't render it — just use it to place the client). Otherwise add a light prompt: `💬 ask "why" for the reasoning`.
- For capacity signals, show the specific gap: e.g. `Sales seats: [assigned]/[limit] assigned ([N]%)`, `Credits: [used]/[limit] used ([N]%)`, `Marketing contacts: [used]/[limit] used ([N]%)`.
- For adoption gaps, list the specific tools: e.g. `Marketing: tracking code, ad retargeting not activated`.
- Tag usage momentum on any client that has it: `📈 usage [score], [+N/−N]`, or cite the driving hub when the overall score is low but a hub is thriving: `📈 [Hub] usage [score] (overall [score])`.
- Render each client name as a markdown link using the URL the tool returned for that record (never build one, and never use `hs_portal_id` — it is the client's portal, not the partner's). If no URL was returned, show the name as plain text.
- The bracketed examples are format only — always render the client's actual numbers.

```
EXPANSION OPPORTUNITIES — [Day, Month D] · portal [account ID]
[N] clients with opportunities · [N] with ML signals · [N] with capacity pressure · [N] with adoption gaps · [N] thriving
Current MRR represented by opportunities: $[sum of hs_total_subscription_mrr across the distinct clients with opportunities, each counted once; warm-only clients excluded]/mo
[Only when the book was not fully scanned: ⚠ Scanned [X] of [Y] clients — [Y−X] not yet checked. Say "analyse more clients" to continue.]

━━ ML REVENUE SIGNALS ([N])
• [Client name] — [MRR]/mo · [products]
  Categories: [signal categories from hs_revenue_signals]
  [If a reasoning request pulled it: Opportunity: [type + hub + direction] then the explanation]
  [If an outreach request pulled it: Suggested outreach: the positioning text]
  [If neither was pulled: 💬 ask "why" for the reasoning]
+ [N] more

━━ CAPACITY PRESSURE ([N])
• [Client name] — [MRR]/mo · [products]
  [specific capacity gap with numbers]
  [recommendation: "Consider upselling additional seats" / "Consider a credits pack" / "Consider a Marketing Hub contact tier upgrade"]
+ [N] more

━━ ADOPTION GAPS ([N])
• [Client name] — [MRR]/mo · [products]
  Unactivated: [tools from tools_to_activate]
  [recommendation: "Training opportunity" / "May benefit from a higher edition with these tools"]
+ [N] more

━━ WARM ACCOUNTS — THRIVING, NO SPECIFIC GAP ([N])
• [Client name] — [MRR]/mo · [products]
  📈 usage [score], [+N/−N] · strongest in [hub with highest usage score]
  Proactive expansion conversation — the account is succeeding and has room to grow
+ [N] more

━━ RENEWAL RISK — SEE RENEWAL-RADAR ([N])
• [Client name] — [MRR]/mo · renewal signal reads as retention risk, not expansion
```

### Overlap note

A client may appear in multiple sections (e.g. both ML signal and capacity pressure). That's expected — it strengthens the opportunity. Show the client in each relevant section and note the overlap, e.g. `⚠ Also has capacity pressure (see below)`. A client in RENEWAL RISK can still appear in CAPACITY PRESSURE or ADOPTION GAPS; add `⚠ renewal risk — handle the renewal first`.

## Phase 4 — What next?

Close with a few actionable handoffs. The list below is the palette, not a script to print in full. Choose the two or three that fit *this* partner right now:
- Offer a route only when it's relevant to what this run surfaced (e.g. "renewal radar" only when renewal-type signals or a RENEWAL RISK section appeared).
- **Don't route the partner back to a skill they just came from.**

"analyse more clients" is about clients you never scanned (a pagination gap); the reasoning/outreach prompts are about clients you *did* scan but whose on-demand text you haven't pulled. Don't conflate them.

```
📄 "analyse more clients" — ONLY when pagination did not reach X=Y: [X] of [Y] clients scanned, [Y−X] never checked
💬 "why these? / show the reasoning" — why the flagged clients fit and what to sell
✍ "draft outreach / how do I pitch" — suggested messaging for the clients in question
🎯 "best [hub] opportunities" — cross-sell and upsell candidates for one hub
🔎 "tell me about [client]" — the single-client view (book-pulse Phase 6)
📅 "renewal radar" — renewal timing and dates to verify for the flagged clients
📊 "book pulse" — zoom out to the whole-portfolio overview
📋 Export this list — copy the report for your team's outreach planning
```

### Outreach when no suggested copy exists

If `hs_revenue_signal_positioning` is empty for a client, say no suggested copy exists, then offer a short draft built only from the data already shown (signal category, capacity numbers, unactivated tools, usage). Label it `Draft — not sent`. Never invent figures, and never send or schedule anything.

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **No opportunities found**: say so plainly — don't render an empty report as if something were flagged.
- **Fetch fails**: report the error and suggest checking the HubSpot connection.