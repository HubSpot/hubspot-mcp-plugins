---
name: renewal-radar
description: >
  Read-only renewal reporting for a HubSpot Solutions Partner: surface which of
  the partner's clients have HubSpot subscription renewals coming up, ranked by
  urgency, with per-Hub renewal dates and the Contract Manager to coordinate
  with. Also flags renewal dates in the past that need verifying. Use for
  viewing, prioritizing, or drilling into upcoming or per-Hub client renewals —
  e.g. "walk me through my upcoming renewals", "who's renewing in the next 90
  days", or "renewal dates for [client]". Do not use for creating or editing
  renewal deals, updating records, or contacting Contract Managers or
  customers — those are write workflows, and this skill only reads and reports.
  The deep-dive renewals companion to book-pulse.
---

# Renewal Radar

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. It identifies approaching HubSpot renewals per Hub, ranks them by urgency, flags past renewal dates that need verifying, and surfaces the Contract Manager to coordinate with.

**Before anything else:** check for PARTNER_CLIENT read access. If more than one HubSpot connector is attached, run the check on each and use the one whose portal has PARTNER_CLIENT read access; name that portal (account ID) in your first line of output. If no connector has access, do not query — say the partner client data isn't accessible in this connection, ask the user to connect a Solutions Partner portal or grant the required scope, and stop.

## Phase 1 — Read the active book

Read the partner's **active** clients — the PARTNER_CLIENT records where `hs_is_active` is true. The radar ranks and counts across the **whole active book** (only the *rendered* lists are capped, see Phase 3), so cover every active client, not a sample or a single page.

If a tool to query CRM data via SQL is available, use this query:

```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_hubs, hs_renewal_mrr,
       hs_marketing_hub_renewal_date, hs_sales_hub_renewal_date,
       hs_service_hub_renewal_date, hs_content_hub_renewal_date,
       hs_operations_hub_renewal_date, hs_commerce_hub_renewal_date,
       hs_next_renewal_date,
       hs_contract_manager_name, hs_contract_manager_email,
       hs_next_cancellation_date, hs_cancellation_products, hs_unified_usage_score
FROM PARTNER_CLIENT
WHERE hs_is_active = true
ORDER BY hs_total_subscription_mrr DESC
```

Don't filter on `hs_next_renewal_date IS NOT NULL` — a client can carry per-Hub dates even when the consolidated field is stale or empty.

If you rank and report before the book is fully covered, **state the partial coverage explicitly** in the summary line (see Phase 3) — never present a partial scan as if it were the whole book, and never imply "no more urgent renewals" from an incomplete scan.

If the query tool supports `verbosityLevel`, set it to `MEDIUM` so each record comes back with its URL — the default can omit links, and the Phase 3 client links depend on it.

Inspect the pagination instructions in the tool response (it reports how many of the total were returned and how to fetch the rest) and follow them to continue retrieving until the whole book is covered.

If no SQL query tool is available, use whatever CRM read tools are available to fetch all active clients (objectType `0-145`, filter `hs_is_active = true`) with the same property set, paging until the data is exhausted, and use the record URL each tool returns.

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

### Next renewal — `hs_next_renewal_date` does not roll forward

`hs_next_renewal_date` holds the account's last *scheduled* renewal and is **not** updated after that date passes, so it can sit in the past even when the client has later Hub renewals. Treat it as a fallback, not the key. For each client compute:

- **`nextRenewal`** = the earliest per-Hub renewal date that is on or after today. If no per-Hub date is on or after today, use `hs_next_renewal_date` only if it is on or after today. Otherwise the client has **no known next renewal**.
- **`pastDates`** = every per-Hub date, plus `hs_next_renewal_date`, that is before today. These may never have been updated after the renewal happened.
- **`needsVerification`** = `pastDates` is non-empty.

Never describe a past date as overdue, lapsed, missed, or at risk. It means the date needs confirming.

## Phase 2 — Classify (one row per client per table)

Use the **current date from the runtime/system context** — never hardcode today's date. This is light arithmetic on a few hundred rows — do it **inline**; don't write or run scripts/code to process the results.

For each client with a `nextRenewal`, compute `daysUntil` = calendar days from today to `nextRenewal`, then classify:

- **Urgent** — `0 <= daysUntil < 30`.
- **Renewing soon** — `30 <= daysUntil < 90`.
- **Beyond radar** — `daysUntil >= 90`. Include only when the user asks for a broader horizon.

Separately, every client with `needsVerification` goes in the **Date needs verification** table, whatever its tier. A client can appear in both (e.g. Marketing Hub date past, Sales Hub renewing in 45 days). Unparseable dates go in the data-quality section.

The Hub(s) on a tier row are those whose per-Hub date equals `nextRenewal`; if `nextRenewal` came from the consolidated field with no matching per-Hub date, mark the Hub as `unspecified`. Later Hub renewals are not separate rows — they surface via the per-product breakdown (Phase 4).

Default horizon is the **next 90 days** (urgent + renewing soon), with the verification table always shown when non-empty. 90 days is the **renewal window** — the ~3 months before contract end when the deal moves to the Renewal Pipeline under a Contract Manager, which is why coordinating with the CM is the default action. Honor any explicit window the user asks for instead of the default; the urgency tiers keep their thresholds and empty ones are omitted. Window phrases:
- "this month" → today through the last day of the current calendar month.
- "next month" → the first through the last day of the next calendar month.
- "next N days/weeks" → today through today + N.

**Single-Hub requests.** When the user scopes to one Hub (e.g. "Sales Hub renewals"), key and classify each client by **that Hub's own** renewal date — not `nextRenewal` — so the Renews date, days-left, and tier all reflect the requested Hub. If that Hub's date is in the past, the client goes in the verification table instead. Include only clients that have that Hub's date, title the report for the Hub (e.g. "SALES HUB RENEWALS"), and set the Hub(s) column to that Hub. Show MRR as `account-level` — `hs_renewal_mrr` / `hs_total_subscription_mrr` are account-wide, never that Hub's renewal value.

**Optional enrichment** (only when the field is actually present — never fabricate): `hs_renewal_mrr`, `hs_total_subscription_mrr`, `hs_all_active_hubs`, `hs_next_cancellation_date` / `hs_cancellation_products`, `hs_unified_usage_score`. Both MRR fields are **account-level**. `hs_renewal_mrr` is the MRR at the consolidated `hs_next_renewal_date`, so it only describes `nextRenewal` when that consolidated date is on or after today and equals `nextRenewal`. A renewal date alone is **not** evidence a client is healthy or at risk — only label risk/health from these signals, and label them as available data.

## Phase 3 — The renewal report

Render a one-line summary, the grouped tables, any data-quality notes, then a short "what to do next". Formatting rules:
- Tables in order: **Urgent**, **Renewing soon**, **Date needs verification** (and **Beyond radar** only if asked). Omit a table with no rows.
- Within a tier, sort by `daysUntil` ascending, then client name. Sort the verification table by the oldest past date first, then MRR descending. Cap each table at 5 rows; if more, add a `+ N more` line below it (don't drop them from the counts).
- Render as real markdown (headings + tables) — **not** inside a code fence, so links and table borders render. The blocks below show layout only.

**1. Summary** — a bold header line. `[title]` is `RENEWAL RADAR` for the whole book or `[HUB] RENEWALS` when scoped to one Hub; `[window]` is the horizon actually applied — default `next 90 days`, or the requested one. Don't print `next 90 days` when a different window was asked for.

**[title] — [window] · [Day, Month D]** — [N] urgent · [N] renewing soon · [N] dates to verify[ · ⚠ scanned [X] of [Y], say "analyse more clients" for the rest]

**2. Tier tables** — put the tier's action as a one-line italic note under the heading, then the table:

### Urgent (N)
_Contact the Contract Manager now; confirm renewal status, scope, and changes._

| Client | Renews | Left | MRR now → renewal | Hub(s) | Contract Manager |
|---|---|---|---|---|---|
| [Client name](returned-url) | Mon D YYYY | Nd left | $current → $renewal/mo | Hub(s) | CM name · CM email |

Tier headings and their italic note:
- **Urgent (N)** → _Contact the Contract Manager now; confirm renewal status, scope, and changes._
- **Renewing soon (N)** → _Start renewal planning with the CM; validate product, tier, seat, or upsell changes._

**3. Verification table**

### Date needs verification (N)
_These renewal dates are in the past and may not have been updated after the renewal. Confirm with the Contract Manager whether each one renewed._

| Client | Past date(s) | Next known renewal | MRR now | Contract Manager |
|---|---|---|---|---|
| [Client name](returned-url) | Mktg · Sep 1 2026 (31d ago) | Sales, Service · Aug 2 2027 | $1,800/mo | CM name · CM email |

Show `none recorded` in "Next known renewal" when the client has no date on or after today.

Column rules:
- **Client**: markdown link using the URL the tool returned for that record (never build one, and never use `hs_portal_id` — it is the client's portal, not the partner's) (plain text if none).
- **Renews**: `Mon D YYYY`. **Left**: `Nd left`, or `today` at 0.
- **MRR now → renewal**: `$[hs_total_subscription_mrr] → $[hs_renewal_mrr]/mo` only when `hs_renewal_mrr` describes this row's date (see Phase 2 enrichment). Otherwise show current MRR only. Leave blank if neither is present. Thousands separators; cents optional. Both values are **account-level**.
- **Hub(s)**: abbreviations fine — Mktg, Ops, Revenue. If the client has other Hubs renewing on later dates, you may append `(+N later)`.
- **Contract Manager**: `[name] · [email]`; `No CM listed` when both are empty.

**4. Missing or incomplete data** — a short list, only if any apply (never silently omit):
- A renewal date but no usable client name. A name that is empty or matches `Hub ID: <number>` counts as missing — show the `hs_object_id` and mark identity incomplete.
- A renewal date but no Contract Manager.
- Active clients with no renewal date at all (count them; list up to 5).
- Unparseable or conflicting date values, noting what couldn't be interpreted.

**5. What to do next** — **3–4 short one-line bullets, never a prose paragraph**. Draw only from:
- Start with the urgent rows, then the dates to verify.
- Contact each client's listed Contract Manager by email — don't send anything automatically.
- For past dates, confirm whether the renewal happened and the date just wasn't updated.
- Confirm renewal scope, timing, seats/tiers, and upsells.
- Check for an existing renewal deal before creating a new one.

Then offer, when relevant:
- 📅 **"renewal dates for [client]"** — the client's full per-product renewal calendar.
- 🔎 **"tell me about [client]"** — the single-client view (book-pulse Phase 6).

## Phase 4 — Per-product renewal breakdown (on demand), "renewal dates for [client]"

When asked, list **every** non-null per-Hub renewal date for that client, sorted soonest-first — including ones past the 90-day horizon. Use the Phase 1 field → Hub mapping; compute days from the runtime date.

- [Hub] — [Mon D YYYY] ([N]d left)
- [Hub] — [Mon D YYYY] ([N]d ago — verify)

Then: `Next known renewal: [Mon D YYYY or none recorded] · Consolidated field: [Mon D YYYY] · CM: [name] — [email]`.

List only Hubs with a real date; omit Hubs the client doesn't have. If only the consolidated `hs_next_renewal_date` is set, say the per-Hub dates aren't recorded and show that one (marked "verify" if it is in the past).

## Business-role guidance

These boundaries govern how you describe who does what — follow them so the partner isn't misled:

- **Contract Managers** own and coordinate the HubSpot renewal deal during the renewal process. When one is listed, present that person as the partner's **renewal coordination contact** — not as proof that every commercial or service responsibility sits with them.
- **Partner-Owned Contracts:** the partner remains responsible for the customer renewal, upsells/tier changes, and late or missed payments. The Contract Manager and partner **coordinate** the pre-renewal workflow, eligibility checks, and contract changes.
- Partners should surface upcoming renewals early, confirm whether products/seats/tiers/upsells may change, and coordinate with the listed Contract Manager. Don't imply the Contract Manager replaces the partner's customer relationship or contractual responsibility.
- **Don't advise opening a new sourced deal just because a renewal is approaching.** First identify or coordinate with the existing renewal deal / Contract Manager, to avoid duplicate renewal and sourced deals.

## Safety and accuracy guardrails

- Read-only: never create or edit records or deals, and never contact anyone, even when write tools are available.
- Never state a renewal is complete, lost, saved, overdue, or at risk unless the data explicitly supports it. A past date alone supports none of these.

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **No renewal fields available / no records returned**: say so plainly and explain the limitation — don't render an empty-looking radar as if there were simply nothing to renew.
- **Fetch fails**: report the error and suggest checking the HubSpot connection.
