---
name: renewal-radar
description: >
  Read-only renewal reporting for a HubSpot Solutions Partner: surface which of
  the partner's clients have HubSpot subscription renewals coming up, ranked by
  urgency, with per-Hub renewal dates and the Contract Manager to coordinate
  with. Use for viewing, prioritizing, or drilling into upcoming, overdue, or
  per-Hub client renewals — e.g. "walk me through my upcoming renewals",
  "who's renewing in the next 90 days", or "renewal dates for [client]".
  Do not use for creating or editing renewal deals, updating records, or
  contacting Contract Managers or customers — those are write workflows, and
  this skill only reads and reports. The deep-dive renewals companion to
  book-pulse.
---

# Renewal Radar

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. It identifies approaching HubSpot renewals per Hub, ranks them by urgency, and surfaces the Contract Manager to coordinate with.

**Before anything else:** check for PARTNER_CLIENT read access. If the object or CRM read tools aren't available, do not query it — say the partner client data isn't accessible in this connection, ask the user to connect a Solutions Partner portal or grant the required scope, and stop.

## Phase 1 — Read the active book

Read the partner's **active** clients — the PARTNER_CLIENT records where `hs_is_active` is true. The radar ranks and counts across the **whole active book** (only the *rendered* lists are capped, see Phase 3), so cover every active client, not a sample or a single page.

If a tool to query CRM data via SQL is available, use this query:

```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_products, hs_renewal_mrr,
       hs_marketing_hub_renewal_date, hs_sales_hub_renewal_date,
       hs_service_hub_renewal_date, hs_content_hub_renewal_date,
       hs_operations_hub_renewal_date, hs_commerce_hub_renewal_date,
       hs_next_renewal_date,
       hs_contract_manager_name, hs_contract_manager_email,
       hs_next_cancellation_date, hs_cancellation_products, hs_unified_usage_score
FROM PARTNER_CLIENT
WHERE hs_is_active = true
  AND hs_next_renewal_date IS NOT NULL
ORDER BY hs_next_renewal_date ASC
```

Inspect the pagination instructions in the tool response (it reports how many of the total were returned and how to fetch the rest) and follow them to continue retrieving until the whole book is covered. If you rank and report before the book is fully covered, **state the partial coverage explicitly** in the summary line (see Phase 3, which already carries the "analyse more clients" offer) — never present a partial scan as if it were the whole book, and never imply "no more urgent renewals" from an incomplete scan.

If no SQL query tool is available, use whatever CRM read tools are available to fetch all active clients (objectType `0-145`, filter `hs_is_active = true`) with the same property set, paging until the data is exhausted.

If the tool response returns a URL for a record, always use it to link that client in Phase 3 — don't construct or alter record URLs yourself.

### Renewal-date fields → Hub display names

Each field is a separate Hub's renewal date and is populated only if the client has that Hub. Map field → display name:

| Field | Hub |
|---|---|
| `hs_marketing_hub_renewal_date` | Marketing Hub |
| `hs_sales_hub_renewal_date` | Sales Hub |
| `hs_service_hub_renewal_date` | Service Hub |
| `hs_content_hub_renewal_date` | Content Hub |
| `hs_operations_hub_renewal_date` | Operations Hub |
| `hs_commerce_hub_renewal_date` | Revenue Hub |

`hs_next_renewal_date` is the account's consolidated next renewal (the soonest across its Hubs) and is the **primary key for the radar**: one row per client, classified and dated by this field. The per-Hub dates identify *which* Hub(s) renew on that date (for the Hub(s) column) and power the on-demand per-product breakdown (Phase 4). If only the consolidated date is present with no per-Hub date, mark the Hub as "unspecified".

## Phase 2 — Classify urgency (one row per client)

Use the **current date from the runtime/system context** — never hardcode today's date. This is light arithmetic on a few hundred rows — do it **inline**; don't write or run scripts/code to process the results.

**One row per client, keyed on the consolidated next renewal.** Classify each client by `hs_next_renewal_date` (its soonest renewal) — that single date sets the client's tier and is the row's date. The Hub(s) shown are those whose per-Hub renewal date equals that consolidated date. Do **not** split a client into multiple rows for Hubs that renew on other dates — those later Hub renewals are surfaced only on request via the per-product breakdown (Phase 4). This keeps the radar one row per client and the account-level MRR unambiguous.

For each client, compute `daysUntil` = calendar days from today to its `hs_next_renewal_date`, then classify:

- **Urgent** — `0 <= daysUntil < 30` (renews today through 29 days out).
- **Renewing soon** — `30 <= daysUntil < 90`.
- **Beyond radar** — `daysUntil >= 90`. Include only when the user asks for a broader horizon.
- **Overdue / needs verification** — `daysUntil < 0` (date is in the past). Surface **separately** — never silently drop it.
- **Invalid / ambiguous** — the date can't be parsed or is contradictory. Put in the data-quality section with a note on what couldn't be interpreted.

Default horizon is the **next 90 days** (urgent + renewing soon), with overdue surfaced separately. 90 days is the **renewal window** — the ~3 months before contract end when the deal moves to the Renewal Pipeline under a Contract Manager, which is why coordinating with the CM is the default action. Honor any explicit window the user asks for instead of the default (e.g. "renewals this month" → next 30 days); the urgency tiers keep their thresholds and empty ones are omitted. For a **single-Hub request**, switch the radar to that Hub — see below.

**Single-Hub requests.** When the user scopes to one Hub (e.g. "Sales Hub renewals"), key and classify each client by **that Hub's own** renewal date (`hs_sales_hub_renewal_date`, `hs_marketing_hub_renewal_date`, etc.) — **not** the consolidated `hs_next_renewal_date` — so the Renews date, days-left, and tier all reflect the requested Hub. Include only clients that have that Hub's date, title the report for the Hub (e.g. "SALES HUB RENEWALS"), and set the Hub(s) column to that Hub. Leave the MRR column blank or mark it `account-level` — `hs_renewal_mrr` / `hs_total_subscription_mrr` are account-wide, not specific to one Hub, so never present them as that Hub's renewal value.

**Optional enrichment** (only when the field is actually present — never fabricate): `hs_renewal_mrr` (**account-level** MRR at the account's next renewal — not per-Hub), `hs_total_subscription_mrr` (**account-level** current MRR; the delta `hs_renewal_mrr − hs_total_subscription_mrr` is the projected change at that next renewal), `hs_all_active_products`, `hs_next_cancellation_date` / `hs_cancellation_products`, `hs_unified_usage_score`. Because both MRR fields are account-level, tie them to the account's consolidated next-renewal row only (see Phase 3), never to individual later Hub rows. A renewal date alone is **not** evidence a client is healthy or at risk — only label risk/health from these signals, and label them as available data.

## Phase 3 — The renewal report

Render three parts: a one-line summary, the grouped renewal **tables**, then a short "what to do next". Formatting rules:
- One table per urgency tier, in order: **Urgent**, **Renewing soon**, **Overdue / needs verification**. Omit a tier with no rows.
- Within a tier, sort by `daysUntil` ascending, then client name. Cap each table at 5 rows; if more, add a `+ N more` line below it (don't drop them from the counts).
- One row per client, at its consolidated next-renewal date; list the Hub(s) renewing on that date. Hubs renewing on later dates are **not** separate rows here — they surface only via the on-demand per-product breakdown (Phase 4). (For a single-Hub request, key on that Hub's date instead — see Phase 2 "Single-Hub requests".)
- Render as real markdown (headings + tables) — **not** inside a code fence, so links and table borders render.

**1. Summary** — a bold header line. Parameterize `[title]` and `[window]`: `[title]` is `RENEWAL RADAR` for the whole book or `[HUB] RENEWALS` when scoped to one Hub (Phase 2); `[window]` is the horizon actually applied — default `next 90 days`, or the requested one (`this month`, a date range, …). Don't print `next 90 days` when a different window was asked for.

**[title] — [window] · [Day, Month D]** — [N] urgent · [N] renewing soon · [N] overdue[ · ⚠ scanned [X] of [Y], say "analyse more clients" for the rest]

**2. Renewal tables** — one per tier. Put the tier's action as a one-line italic note under the heading, then the table:

### Urgent (N)
_Contact the Contract Manager now; confirm renewal status, scope, and changes._

| Client | Renews | Left | MRR now → renewal | Hub(s) | Contract Manager |
|---|---|---|---|---|---|
| [\[Client name\]](record-url) | [Mon D YYYY] | [N]d left | $[current] → $[renewal]/mo | [Hub(s)] | [CM name] · [CM email] |

`+ N more`

Tier headings and their italic note:
- **Urgent (N)** → _Contact the Contract Manager now; confirm renewal status, scope, and changes._
- **Renewing soon (N)** → _Start renewal planning with the CM; validate product, tier, seat, or upsell changes._
- **Overdue / needs verification (N)** → _Verify status immediately; escalate via the listed CM._

Column rules:
- **Client**: markdown link using the URL the tool returned for that record (plain text if none).
- **Renews**: `Mon D YYYY`. **Left**: `Nd left` upcoming, `today` at 0, `Nd ago` overdue.
- **MRR now → renewal**: `$[hs_total_subscription_mrr] → $[hs_renewal_mrr]/mo` — both **account-level** (`hs_renewal_mrr` is the MRR at the account's next renewal, which is this row's date, so it applies cleanly to the one-row-per-client radar). Show just one value (no arrow) if only one is present; leave blank if neither. Thousands separators; cents optional.
- **Hub(s)**: the Hub(s) whose per-Hub renewal date equals the client's consolidated next-renewal date (Phase 1 mapping; abbreviations fine — Mktg, Ops, Revenue). `unspecified` if the client has only the consolidated date and no per-Hub date. If the client has other Hubs renewing on later dates, you may append `(+N later)` as a hint to use the per-product breakdown.
- **Contract Manager**: `[name] · [email]`; `No CM listed` when both are empty.

**3. Missing or incomplete data** — a short list, only if any apply (never silently omit):
- A renewal date but no client identity → show the `hs_object_id`, mark identity incomplete.
- A renewal date but no Contract Manager.
- Invalid or conflicting date values, noting what couldn't be interpreted.

**4. What to do next** — **3–4 short one-line bullets, never a prose paragraph** (don't fold them into a sentence). Draw only from:
- Start with the overdue, then the urgent, rows above.
- Contact each client's listed Contract Manager by email — don't send anything automatically.
- Confirm renewal scope, timing, seats/tiers, and upsells.
- Check for an existing renewal deal before creating a new one.

Then offer, when relevant:
- 📅 **"renewal dates for [client]"** — the client's full per-product renewal calendar, including Hubs renewing beyond the 90-day radar (see below). Especially useful when a client's Hubs renew on different dates.

## Phase 4 — Per-product renewal breakdown (on demand), "renewal dates for [client]"

When asked, list **every** non-null per-Hub renewal date for that client, sorted soonest-first, so the partner sees exactly which product renews when — including ones past the 90-day horizon that the radar omits. Use the Phase 1 field → Hub mapping; compute `daysUntil` from the runtime date.

```
[Client name] — renewal calendar
• [Hub] — [Mon D YYYY] ([N]d left)
• [Hub] — [Mon D YYYY] ([N]d ago, overdue)
Consolidated next renewal: [Mon D YYYY] · CM: [name] — [email]
```

List only Hubs with a real date; omit Hubs the client doesn't have. If only the consolidated `hs_next_renewal_date` is set, say the per-Hub dates aren't recorded and show that one.

## Business-role guidance

These boundaries govern how you describe who does what — follow them so the partner isn't misled:

- **Contract Managers** own and coordinate the HubSpot renewal deal during the renewal process. When one is listed, present that person as the partner's **renewal coordination contact** — not as proof that every commercial or service responsibility sits with them.
- **Partner-Owned Contracts:** the partner remains responsible for the customer renewal, upsells/tier changes, and late or missed payments. The Contract Manager and partner **coordinate** the pre-renewal workflow, eligibility checks, and contract changes.
- Partners should surface upcoming renewals early, confirm whether products/seats/tiers/upsells may change, and coordinate with the listed Contract Manager. Don't imply the Contract Manager replaces the partner's customer relationship or contractual responsibility.
- **Don't advise opening a new sourced deal just because a renewal is approaching.** First identify or coordinate with the existing renewal deal / Contract Manager, to avoid duplicate renewal and sourced deals.

## Safety and accuracy guardrails

- Never state a renewal is complete, lost, saved, or at risk unless the data explicitly supports it.

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **No renewal fields available / no records returned**: say so plainly and explain the limitation — don't render an empty-looking radar as if there were simply nothing to renew.
- **Fetch fails**: report the error and suggest checking the HubSpot connection.
