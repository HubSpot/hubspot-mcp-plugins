---
name: expansion-opportunities
description: >
  Read-only expansion reporting for a HubSpot Solutions Partner: surface which
  of the partner's clients are worth an upsell or cross-sell — including clients
  maxed out on seats or credits, sitting on unactivated tools, or simply
  thriving with room to grow — ranked by value, plus (on request) the reasoning
  for a fit and suggested outreach copy. Use for viewing, prioritizing, or drilling into a
  partner's expansion, upsell, or cross-sell opportunities — e.g. "which clients
  can I sell more to", "who's maxed out on seats", or "which clients should I
  approach about AI agents, and why". Do not use for creating or editing deals, updating records, or sending outreach or contacting customers 
  — it drafts copy but only reads and reports. The deep-dive companion to book-pulse.
---

# Expansion Opportunities

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. It goes beyond the ML signal count to surface specific, actionable expansion opportunities per client.

**Before anything else:** check for PARTNER_CLIENT read access. If the object or CRM read tools aren't available, do not query it — say the partner client data isn't accessible in this connection, ask the user to connect a Solutions Partner portal or grant the required scope, and stop.

## Phase 1, Read the active book

Read the partner's **active** clients, the PARTNER_CLIENT records where `hs_is_active` is true with the full expansion property set below. The report classifies and ranks across the whole active book (only the *rendered* lists are capped, see Phase 3). If the tool response returns a URL for a record, use it to link that client in Phase 3 — don't construct or alter record URLs yourself.

If a tool to query CRM data via SQL is available, fetch all active clients with the property set below:

**Scalar fields for all active clients:**

```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_products, hs_all_active_hubs,
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
       hs_service_hub_tools_to_activate, hs_next_renewal_date,
       hs_revenue_signals
FROM PARTNER_CLIENT WHERE hs_is_active = true
ORDER BY hs_total_subscription_mrr DESC
```

Inspect the pagination instructions in the tool response (it reports how many of the total were returned and how to fetch the rest) and follow them to continue retrieving until the book is covered.

If you classify and report before the book is fully covered, **state the partial coverage explicitly** in the report header (see Phase 3) and offer "analyse more clients" in Phase 4 — never present a partial scan as if it were the whole book.

### On-demand text

The two long fields are fetched **only when the request needs them**, scoped to the specific clients in question by `hs_object_id` (which you have from the scalar read):

- Fetch `hs_revenue_signal_explanation` when the request asks for **reasoning** — *why* a client is a fit, *what* to sell, or *which* clients fit a given motion (e.g. "which clients should I approach about AI agents, and why?").
- Fetch `hs_revenue_signal_positioning` **only** when the request asks for **outreach or messaging** — how to pitch, what to say, draft the email. Do not pull it just because you pulled the explanation.

Select only the field(s) the request calls for, scoped to the in-scope clients by `hs_object_id`. If a fetched field comes back empty for a client, treat it as unavailable — fall back to the category labels for reasoning, or say no suggested copy exists for outreach; never bulk-fetch to find out. If the tool has no SQL option, use the available CRM read tools with the same scoping.

> `hs_revenue_signal_explanation` is ML-generated **HTML** and can be long — it is the authoritative *why the signal fired* and *what the opportunity is*. Render it when a reasoning request pulls it.

> `hs_revenue_signal_positioning` is sales-enablement text, suggested outreach messaging. Render it only when an outreach request pulls it.

## Phase 2, Classify opportunities

For each client, compute opportunity signals across five dimensions:

### A. ML Revenue Signals

Two fields carry the ML signal, and they play **different roles**, don't conflate them:

- **`hs_revenue_signals`**, a semicolon-separated list of signal *categories*. Use it for:
  1. **Existence**, `hasMLSignal = hs_revenue_signals` is present and non-empty (treat null or an empty string `""` as no signal). This is the only thing that tells you an ML signal fired.
  2. **Category**, which *kind* of signal fired, for grouping and routing (see table).
- **`hs_revenue_signal_explanation`**, ML-generated HTML that is the **authoritative** source for the specific **hub**, **direction** (cross-sell to a new hub vs. upsell to a higher tier), and **tier movement** (e.g. "Starter → Pro"). It is **not** fetched for the standard report — it's pulled on demand (Phase 1 "On-demand text"). Use it to refine the opportunity type and build talking points **only after** it has been fetched for the client; otherwise classify from the category labels alone.

The categories below are a **non-exhaustive guide**, HubSpot adds values over time, so surface any unlisted value as-is; read its finer meaning from `hs_revenue_signal_explanation` only if that field was fetched on demand.

| Category | What it tells you |
|---|---|
| `Predicted Cross-Sell` / `Predicted Upsell` | Predicted expansion; the actual hub and direction come from the explanation, available only if it was fetched on demand |
| `AI SQL` | AI-identified Sales Qualified Lead (SQL) surfaced from a prior support-bot conversation |
| `Customer Agent` / `Data Agent` / `Prospecting Agent` | Breeze agent adoption signal |
| `Credits Limit` | Hitting the credits limit, needs more credits |
| `Upcoming HubSpot Renewal` / `Upcoming Competitor Renewal` | Renewal-time expansion (competitor threat if the latter) |
| `SFDC Compete Campaign` | Salesforce competitive campaign |
| `Seats Whitespace` | Seat expansion opportunity |

For each client whose `hs_revenue_signals` is non-empty: `hasMLSignal = true`. By default, describe the opportunity type from the category labels alone (e.g. `Predicted Cross-Sell`, `Predicted Upsell`, `Credits Limit`) — do not name a specific hub or tier the labels don't give you. Only when `hs_revenue_signal_explanation` has been fetched on demand for that client, refine the type with the hub, direction, and tier movement it states. Never fetch the explanation just to classify.

### B. Capacity Pressure (computed from seat/credit/contact properties)

Even when the ML signal is silent, capacity pressure signals an expansion opportunity:

All utilization checks must be guarded: only evaluate the comparison when the limit field is present and greater than zero. If a limit is missing, zero, or blank, treat that capacity dimension as **data unavailable** — do not flag it as pressure. The thresholds this skill uses (≈90% seats/contacts, 80% credits, ≥70 usage) are heuristic defaults chosen to surface clients before they hit hard limits — they are not product limits. Honor a sensitivity the partner states (e.g. "flag anyone above 75%").

- `seatsMaxed`: for each hub (core, sales, service, commerce) — only if `seats_limit` is present and `> 0`: flag if `seats_assigned >= seats_limit * 0.9` (90%+ utilization) OR, when `seats_assigned` is present, `seats_limit - seats_assigned <= 1` (at most one seat left; skip this second comparison if `seats_assigned` is missing or blank).
- `creditsMaxed`: only if `hs_credits_limit` is present and `> 0`: flag if `hs_credits_usage >= hs_credits_limit * 0.8` (80%+ utilization). Also flag if `hs_credits_spent_monthly_limit` is non-empty and `hs_additional_hubspot_credits_monthly_limit` is present and `> 0` (already buying additional credits, may need more).
- `contactsMaxed`: only if `hs_marketing_contacts_limit` is present and `> 0`: flag if `hs_marketing_contacts_usage >= hs_marketing_contacts_limit * 0.9` (90%+ utilization) OR `hs_marketing_contacts_limit - hs_marketing_contacts_usage <= 50` (50 or fewer contacts remaining).
- `hasCapacitySignal`: `seatsMaxed` OR `creditsMaxed` OR `contactsMaxed`.

### C. Adoption Gaps (from `tools_to_activate`)

- `hasAdoptionGap`: any of `hs_marketing_hub_tools_to_activate`, `hs_sales_hub_tools_to_activate`, `hs_service_hub_tools_to_activate` is non-empty. These list tools the client has access to but hasn't activated, an adoption gap that may benefit from training (partner service opportunity) or a different edition (expansion).

### D. Usage Momentum (from usage score + trend)

A client actively getting value is a receptive expansion target, high or rising usage means appetite and room to grow, even with no specific gap:
- `hasUsageMomentum`: `hs_unified_usage_score` is high (≥ 70) OR `hs_last_4_weeks_usage_score_trend` is upward OR **any single hub** (`hs_marketing_hub_usage_score` / `hs_sales_hub_usage_score` / `hs_service_hub_usage_score` / `hs_commerce_hub_usage_score`) is high (≥ 70). Strongest when overall usage is high *and* rising.
- **Don't gate on the unified score alone**, a low overall UUS can mask a hub the client leans on heavily. That thriving hub is the expansion anchor: deepen it, or cross-sell an adjacent hub off its strength. Note which hub is driving momentum.

### E. Opportunity classification

- `hasOpportunity`: `hasMLSignal` OR `hasCapacitySignal` OR `hasAdoptionGap`. A client with **only** `hasUsageMomentum` still qualifies, as a warm *proactive-outreach* candidate framed as "expand into a thriving account," not a specific gap.
- `opportunityTypes`: collect all matching types. For ML signals, describe the opportunity from the category label by default (e.g. `["ML: Predicted Cross-Sell", "Capacity: Seats Maxed (Sales)"]`); only sharpen it to a specific hub/tier (e.g. `"ML: Cross-sell Service Hub"`) when `hs_revenue_signal_explanation` was fetched on demand for that client.
- `priority`: rank by MRR descending (higher MRR = higher expansion value); use `hasUsageMomentum` as an amplifier, a warm, growing account outranks a flat one at similar MRR.

## Phase 3, The expansion report

**Scope to the request first.** The full four-section report below is the default for an open-ended "show me expansion opportunities." If the request is narrower, answer in the smallest shape that fits and omit the rest:
- a specific motion or signal ("which clients for AI agents / Breeze", "who's maxed out on seats") → filter to those clients and render only the relevant section(s);
- one client ("tell me about [client]'s expansion fit") → just that client's opportunities and reasoning;
- a count or yes/no ("how many have cross-sell signals?") → answer directly, optionally with the list.

Otherwise, render the default report using this template (adapt sections as needed). Rules:
- Sort by MRR descending within each group.
- **Omit any section with no items, no placeholders, no "N/A".**
- Cap each list at 5; if more, add `+ N more`.
- MRR as `$#,###/mo` (thousands separator, `/mo` suffix).
- `[products]`: the client's active hubs from `hs_all_active_hubs`, abbreviated; if more than 3, show the first 3 + `+N`.
- For ML signals, the report classifies from `hs_revenue_signals` (the category list). Render `hs_revenue_signal_explanation` / `hs_revenue_signal_positioning` **only when the current request pulled them** on demand (see Phase 1 "On-demand text") — render them (HTML) for the in-scope clients. Otherwise don't render the text; add a light prompt that it's available on request: `💬 ask "why" for the reasoning`.
- For capacity signals, show the specific gap: e.g. `Sales seats: [assigned]/[limit] assigned ([N]%)` or `Credits: [used]/[limit] used ([N]%)`.
- For adoption gaps, list the specific tools: e.g. `Marketing: tracking code, ad retargeting not activated`.
- Tag usage momentum on any client that has it: `📈 usage [score], [trend]`, or cite the driving hub when the overall score is low but a hub is thriving: `📈 [Hub] usage [score] (overall [score])`. It marks readiness and is why the client ranks where it does. A client whose *only* signal is momentum goes in the WARM ACCOUNTS section.
- Render each client name as a **clickable markdown link** using the URL the tool returned for that record; don't construct or alter it. If no URL was returned, show the name as plain text.
- The bracketed examples above are format only — always render the client's actual numbers, never the example values.

```
EXPANSION OPPORTUNITIES, [Day, Month D]
[N] clients with opportunities · [N] with ML signals · [N] with capacity pressure · [N] with adoption gaps · [N] thriving
Current MRR represented by opportunities: $[sum of hs_total_subscription_mrr across the distinct clients with opportunities — count each client once even if it appears in multiple sections]/mo
[Only when the book was not fully scanned: ⚠ Scanned [X] of [Y] clients — [Y−X] not yet checked. Say "analyse more clients" to continue.]

━━ ML REVENUE SIGNALS ([N])
• [Client name], [MRR]/mo · [products]
  Categories: [signal categories from hs_revenue_signals]
  [If a reasoning request pulled it: Opportunity: [type + hub + direction, read from the explanation] then hs_revenue_signal_explanation, rendered HTML]
  [If an outreach request pulled it: Suggested outreach: hs_revenue_signal_positioning]
  [If neither was pulled: 💬 ask "why" for the reasoning]
+ [N] more scanned clients with ML signals

━━ CAPACITY PRESSURE ([N])
• [Client name], [MRR]/mo · [products]
  [specific capacity gap: seats / credits / contacts with numbers]
  [recommendation: "Consider upselling additional seats" / "Consider credits pack" / "Consider Marketing Hub tier upgrade"]
+ [N] more

━━ ADOPTION GAPS ([N])
• [Client name], [MRR]/mo · [products]
  Unactivated: [tools from tools_to_activate]
  [recommendation: "Training opportunity" / "May benefit from a higher edition with these tools"]
+ [N] more

━━ WARM ACCOUNTS, THRIVING, NO SPECIFIC GAP ([N])
• [Client name], [MRR]/mo · [products]
  📈 usage [score], [trend] · strongest in [hub with highest usage score]
  Proactive expansion conversation, the account is succeeding and has room to grow
+ [N] more
```

### Overlap note

A client may appear in multiple sections (e.g. both ML signal and capacity pressure). That's expected, it strengthens the opportunity. In the report, show the client in each relevant section. If a client has both an ML signal and capacity pressure, note the overlap: e.g. `⚠ Also has capacity pressure (see below)`.

## Phase 4, What next?

Close with a few actionable handoffs. These are **context-dependent**, the list below is the palette to draw from, not a script to print in full. The quoted strings below are suggested prompts — adapt the wording to the conversation; the action behind each is what matters. Choose the two or three that fit *this* partner right now:
- Offer a route only when it's relevant to what this run surfaced or where the partner is likely to go next (e.g. only mention "renewal radar" when there are clients with `Upcoming HubSpot Renewal` / `Upcoming Competitor Renewal` signals).
- **Don't route the partner back to a skill they just came from**

"analyse more clients" is about clients you never scanned (an exceptional pagination gap); the reasoning/outreach prompts below are about clients you *did* scan but whose on-demand text you haven't pulled. Don't conflate them.

```
📄 "analyse more clients" — [ONLY in the exceptional case that pagination did not reach X=Y] [X] of [Y] clients scanned, [Y−X] never checked. Do not show this when the full book was scanned.
💬 "why these? / show the reasoning" — why the flagged clients fit and what to sell
✍ "draft outreach / how do I pitch" — suggested messaging for the clients in question
🎯 "best [hub] opportunities", rank the whole book for a specific hub cross-sell
🔎 "tell me about [client]", drill into one client's full record
📅 "renewal radar", the [N] clients with upcoming renewals (expand at renewal)
📊 "book pulse", zoom out to the whole-portfolio overview
📋 Export this list, copy the report for your team's outreach planning
```

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **No opportunities found**: say so plainly — don't render an empty report as if something were flagged.
- **Fetch fails**: report the error and suggest checking the HubSpot connection.