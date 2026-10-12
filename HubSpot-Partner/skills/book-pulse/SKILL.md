---
name: book-pulse
description: >
  Quick book-of-business snapshot for a HubSpot Solutions Partner — total
  clients, total MRR, which clients need attention (churn signals, declining
  usage, near-term renewals, renewal dates to verify, managed relationships
  about to lapse) and which carry an ML-detected revenue signal. The fast entry
  point: it gives the overview, then points you to the specialised partner
  skills for the deep work. Read-only: never creates or edits records or deals,
  and never contacts anyone.

  ALWAYS use this skill for any portfolio-level request — "how's my book of
  business", "show me my clients", "client overview", "portfolio summary",
  "book of business", "client snapshot", "which clients need attention", "how's
  my portfolio doing", "partner overview", "review my clients" — and for
  "tell me about [client]". This skill is a lightweight pulse only — it hands
  off to the other skills for detail.
---

# Book Pulse

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. This is a **pulse, not a deep analysis** — keep it short and route the partner onward (see Phase 4).

**Before anything else:** check for PARTNER_CLIENT read access. If more than one HubSpot connector is attached, run the check on each and use the one whose portal has PARTNER_CLIENT read access; name that portal (account ID) in your first line of output. If no connector has access, do not query — say the partner client data isn't accessible in this connection, ask the user to connect a Solutions Partner portal or grant the required scope, and stop.

## Phase 1 — Read the active book

Read the partner's **active** clients — the PARTNER_CLIENT records where `hs_is_active` is true. The pulse reports portfolio-level numbers, so compute them across the **whole active book** — not a sample or a single page. Don't cap the data you count (only the *rendered* lists are capped — see Phase 3).

If a tool to query CRM data via SQL is available, use the following queries:

Aggregate totals:
```sql
SELECT COUNT(*), SUM(hs_total_subscription_mrr)
FROM PARTNER_CLIENT WHERE hs_is_active = true
```

Rows to classify and render:
```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_hubs, hs_unified_usage_score, hs_last_4_weeks_usage_score_trend,
       hs_cancellation_products, hs_next_cancellation_date,
       hs_marketing_hub_renewal_date, hs_sales_hub_renewal_date,
       hs_service_hub_renewal_date, hs_content_hub_renewal_date,
       hs_operations_hub_renewal_date, hs_commerce_hub_renewal_date,
       hs_next_renewal_date,
       hs_managed_relationship_estimated_expiration_date, hs_revenue_signals
FROM PARTNER_CLIENT WHERE hs_is_active = true
ORDER BY hs_total_subscription_mrr DESC
```

If the query tool supports `verbosityLevel`, set it to `MEDIUM` so each record comes back with its URL — the default can omit links, and the Phase 3 client links depend on it.

Inspect the pagination instructions in the tool response (it reports how many of the total were returned and how to fetch the rest) and follow them to continue retrieving until the whole book is covered.

If no SQL query tool is available, use whatever CRM read tools are available to fetch all active clients with the same properties, paging until the data is exhausted, and use the record URL each tool returns.

If the rows weren't fully covered, **state the partial coverage** in the header line (see Phase 3) — never present a partial scan as the whole book.

> Don't fetch `hs_revenue_signal_explanation` or `hs_revenue_signal_positioning` here. `expansion-opportunities` pulls them on demand.

## Phase 2 — Classify each client (lightweight)

**Next renewal.** `hs_next_renewal_date` does **not** roll forward after the date passes, so use the same rule as renewal-radar:
- `nextRenewal` = the earliest per-Hub renewal date on or after today; if none, `hs_next_renewal_date` only if it is on or after today; otherwise no known next renewal.
- `renewalNeedsVerification` = any per-Hub date or `hs_next_renewal_date` is before today. Never call it overdue or lapsed.

For each client compute:
- `hasChurnSignal`: `hs_cancellation_products` non-empty OR `hs_next_cancellation_date` set OR `hs_unified_usage_score` < 30 (only when the score is present)
- `isDeclining`: `hs_last_4_weeks_usage_score_trend` < 0
- `renewingSoon`: `nextRenewal` is 0–89 days from today
- `managedLapsingSoon`: `hs_managed_relationship_estimated_expiration_date` is 0–15 days from today — the projected date the managed relationship lapses if nothing changes. This puts the client's **MRR at risk**, so it counts as needs-attention.
- `hasRevenueSignal`: `hs_revenue_signals` is non-empty (treat null or `""` as no signal). This is the same definition `expansion-opportunities` uses, so the counts match.
- `needsAttention`: `hasChurnSignal` OR `isDeclining` OR `renewingSoon` OR `renewalNeedsVerification` OR `managedLapsingSoon`
- `limitedData` (diagnostic, **not** needs-attention): `hs_total_subscription_mrr` AND `hs_unified_usage_score` are both empty, so the record isn't carrying real data.
- `noFlags`: not `needsAttention` and not `limitedData`. This means no flags were found in the available data — it is **not** a health verdict.

**Managed vs. populated:** `hs_is_managed` = true means the record's data is populated for the partner — *unless the client has opted out of partner data sharing*. `hs_is_managed` = false means the record is minimally populated by design. Phase 5 investigates this on request.

**Expansion scope:** `hasRevenueSignal` is only the ML `hs_revenue_signals` field. What it does **not** capture is **seat/credit/contact capacity**; this pulse doesn't assess that either. `expansion-opportunities` does, so treat the "with expansion signals" count here as a floor — always offer the expansion hand-off.

Take total active clients and total MRR from the Phase 1 aggregate query (on the fallback path, sum across all pages). Derive the other counts from the classified rows.

## Phase 3 — The overview

Render this layout as real markdown — **not inside a code fence**, so client links work. The block below shows layout only. Rules:
- Needs-attention first, then no-flags. Sort each group by MRR descending; clients without MRR sort last.
- **Omit any section with no items — no placeholders, no "N/A".**
- Cap each list at 5; if more, add `+ N more`.
- MRR as `$#,###/mo` (thousands separator, `/mo` suffix), e.g. `$4,250/mo`. If MRR is empty, show `MRR not reported`.
- For a `managedLapsingSoon` client, show the MRR at risk, e.g. `managed relationship lapses Nov 3 — $3,100/mo at risk`.
- For a `renewalNeedsVerification` client, show `renewal date [Mon D] passed — verify`.
- Products from `hs_all_active_hubs`, abbreviated (e.g. `Mktg, Sales, Service`); if more than 3, show the first 3 + `+N`. Omit the products line if empty.
- Show the `· [N] with limited data` header suffix and the limited-data line in "what next?" only when that count > 0.
- This is a pulse — do **not** render revenue-signal text or give per-client recommendations.
- Render every client name as a markdown link using the URL the tool returned for that record (never build one, and never use `hs_portal_id` — it is the client's portal, not the partner's) (plain text if none).

```
PARTNER BOOK PULSE — [Day, Month D] · portal [account ID]
[N] active clients · [total MRR]/mo · [N] need attention · [N] with expansion signals[ · [N] with limited data][ · ⚠ scanned [X] of [Y], say "analyse more clients" for the rest]

━━ NEEDS ATTENTION ([N])
• [Client name] — usage [score] · [MRR]/mo
  [products] · [churn signal / usage declining / renews in X days / renewal date [Mon D] passed — verify / managed relationship lapses [date] — [MRR]/mo at risk]
+ [N] more

━━ NO FLAGS ([N])
[names if 5 or fewer, otherwise just the count]

Top 3 by MRR: [name] [MRR] · [name] [MRR] · [name] [MRR]
```

## Phase 4 — What next?

Close with only the lines that match what the pulse found — these are handoffs into the other partner skills, not work to do here. The renewal and limited-data lines appear only when their count > 0. The **expansion** and **tell-me-about** lines always appear.

```
What next?
📅 "renewal radar" — [N] renewals in the next 90 days · [N] renewal dates to verify
📈 "expansion opportunities" — [N] flagged by ML signals; it also finds seat/credit/contact-maxed clients
🔧 "why the limited data?" — look into the [N] clients with limited data
🔎 "tell me about [client]" — drill into one
```

Drop either half of the renewal line when its count is 0.

## Phase 5 — Limited-data investigation (on request)

Run only if the partner asks why clients have limited data. This is a troubleshooting path — don't pull extra properties upfront.

For each `limitedData` client, check `hs_is_managed` (already fetched in Phase 1):
- **`hs_is_managed` = false** → that's the reason: the client isn't managed, so the record is only minimally populated. No further lookup needed.
- **`hs_is_managed` = true** → managed but data isn't flowing. Only now, pull `hs_available_property_groups` for that record and read which groups the client shares:
  - **Field empty, or some groups missing / restricted** → a **partner data-sharing opt-out**. For a managed client the field is only empty when the client has opted out, so treat an empty field the same as restricted groups. Explain that the client has limited what they share, and point the partner to the KB article: https://knowledge.hubspot.com/privacy-and-consent/manage-partner-data-sharing-settings
  - **All groups present / shared, yet the data is still empty** → this is *not* explained by an opt-out and isn't something the partner can fix. Encourage them to submit feedback (via the HubSpot feedback option, if available) so the gap can be investigated.

Summarise briefly, e.g. "3 of the 5 aren't managed; 1 has restricted data sharing; 1 is fully shared but empty — worth reporting." Use `hs_available_property_groups` on this path only.

## Phase 6 — Single-client drill-down ("tell me about [client]")

Query that one client by `hs_object_id`, or by `hs_client_name LIKE '%[name]%'` (ask the partner to pick if more than one matches), with the Phase 1 property set, read the same way as Phase 1. Show, omitting empty lines:

- [Client name](returned-url) — [MRR]/mo · [products] · managed: yes/no
- Usage [score], trend [+N / −N] over 4 weeks
- Next renewal [Mon D YYYY] ([N]d) · dates to verify: [Hub · Mon D YYYY] · CM [name] — [email]
- Managed relationship estimated expiration [Mon D YYYY]
- Churn signals: [cancellation products / next cancellation date]
- Revenue-signal categories: [hs_revenue_signals]

Then offer: 📅 "renewal dates for [client]" (renewal-radar) and 📈 "why is [client] flagged?" (expansion-opportunities). No recommendations here.

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **Fetch fails**: report the error and suggest checking the HubSpot connection.
