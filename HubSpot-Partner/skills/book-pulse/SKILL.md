---
name: book-pulse
description: >
  Quick book-of-business snapshot for a HubSpot Solutions Partner — total
  clients, total MRR, a health breakdown, and which clients need attention
  (churn signals, declining usage, near-term renewals) or carry an ML-detected
  revenue signal. The fast entry point: it gives the overview, then points you
  to the specialised partner skills for the deep work.

  ALWAYS use this skill for any portfolio-level request — "how's my book of
  business", "show me my clients", "client overview", "portfolio summary",
  "book of business", "client snapshot", "which clients need attention", "how's
  my portfolio doing", "partner overview", "review my clients". This skill is a
  lightweight pulse only — it hands off to the other skills for detail.
---

# Book Pulse

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change.

Reads the **PARTNER_CLIENT** CRM object (`0-145`), scoped to the logged-in partner's portal. This is a **pulse, not a deep analysis** — keep it short and route the partner onward (see Phase 4).

**Before anything else:** check the available HubSpot permissions/scopes. If the user has read access to PARTNER_CLIENT, continue. If not, this isn't a Solutions Partner portal — unload this skill, say so, and stop.

## Phase 1 — Read the active book

Read the partner's **active** clients — the PARTNER_CLIENT records where `hs_is_active` is true. The pulse reports portfolio-level numbers (client count, total MRR, needs-attention counts), so compute them across the **whole active book** — not a sample or a single page. Don't cap the data you count (only the *rendered* lists are capped — see Phase 3).

If a tool to query CRM data via SQL is available, use the following queries to perform the aggregation and analysis:

Aggregate totals:
```sql
SELECT COUNT(*), SUM(hs_total_subscription_mrr)
FROM PARTNER_CLIENT WHERE hs_is_active = true
```

Rows to classify and render — active clients ordered by MRR:
```sql
SELECT hs_object_id, hs_client_name, hs_is_managed, hs_total_subscription_mrr,
       hs_all_active_products, hs_unified_usage_score, hs_last_4_weeks_usage_score_trend,
       hs_cancellation_products, hs_next_cancellation_date, hs_next_renewal_date,
       hs_managed_relationship_estimated_expiration_date, hs_revenue_signals,
       hs_revenue_signal_explanation
FROM PARTNER_CLIENT WHERE hs_is_active = true
ORDER BY hs_total_subscription_mrr DESC
```

If no SQL query tool is available, use whatever CRM read tools are available to fetch all active clients with the same properties. Also note the portal's app domain and id (for the record links in Phase 3).

> `hs_revenue_signal_explanation` is ML-generated **HTML** and can be long. In this skill, only test whether it is non-empty — **do not render it here**. The full signal (and its `hs_revenue_signal_positioning` sales-enablement companion) is surfaced by `expansion-opportunities`.

## Phase 2 — Classify each client (lightweight)

For each client compute:
- `hasChurnSignal`: `hs_cancellation_products` non-empty OR `hs_next_cancellation_date` set OR `hs_unified_usage_score` < 30
- `isDeclining`: `hs_last_4_weeks_usage_score_trend` indicates a downward trend
- `renewingSoon`: `hs_next_renewal_date` within 90 days
- `managedLapsingSoon`: `hs_managed_relationship_estimated_expiration_date` within 15 days — the projected date the managed relationship lapses if nothing changes. This puts the client's **MRR at risk**, so it counts as needs-attention.
- `hasRevenueSignal`: `hs_revenue_signals` present OR `hs_revenue_signal_explanation` non-empty (treat an empty string `""` as no signal). This reflects **only** the ML signal (which already covers cross-sell to some extent) — do not evaluate seat/credit capacity here (see the expansion-scope note below).
- `needsAttention`: `hasChurnSignal` OR `isDeclining` OR `renewingSoon` OR `managedLapsingSoon`
- `limitedData` (diagnostic, **not** needs-attention): the client's core fields — `hs_total_subscription_mrr`, `hs_unified_usage_score`, `hs_all_active_products` — are empty/null, so the record isn't carrying real data.

**Managed vs. populated:** `hs_is_managed` = true means the record's data is populated for the partner — *unless the client has opted out of partner data sharing*. `hs_is_managed` = false means the record is minimally populated by design. This is what explains `limitedData`; Phase 5 investigates it on request.

**Expansion scope:** `hasRevenueSignal` is only the ML `hs_revenue_signals` / explanation — which already covers cross-sell and product-similarity fits. What the ML signal does **not** capture is **seat/credit capacity** (e.g. a client maxed out on seats or credits, a good fit to buy more); this pulse doesn't assess that either. `expansion-opportunities` does, so it can surface a seat/credit chance **even when the ML signal is silent**. Treat the "with expansion signals" count here as a floor — always offer the expansion hand-off.

Take total active clients and total MRR from the Phase 1 aggregate query (on the fallback path, sum across all pages). Derive the needs-attention / healthy / has-revenue-signal / limited-data counts from the classified rows.

## Phase 3 — The overview

Render exactly this template. Rules:
- Needs-attention first, then healthy. Sort each group by MRR descending.
- **Omit any section with no items — no placeholders, no "N/A".**
- Cap each list at 5; if more, add `+ N more`.
- MRR as `$#,###/mo` (thousands separator, `/mo` suffix), e.g. `$4,250/mo`.
- For a `managedLapsingSoon` client, show the MRR at risk using the client's MRR, e.g. `managed relationship lapses Nov 3 — $3,100/mo at risk`.
- Products from `hs_all_active_products` (everything the client subscribes to): abbreviate names (e.g. `Mktg Pro, Sales Ent`); if more than 3, show the first 3 + `+N`.
- Show the `· [N] with limited data` header suffix and the limited-data line in "what next?" only when that count > 0.
- This is a pulse — do **not** render revenue-signal HTML or give per-client recommendations.
- Whenever you list or reference a client, render its name as a **clickable markdown link** to its record: `[Client name](https://[uiDomain]/contacts/[portalId]/record/0-145/[objectId])`. Use the `[uiDomain]` and `[portalId]` captured in Phase 1 (`uiDomain` is account-specific, e.g. `app.hubspot.com` or `app.hubspotqa.com` — never hardcode it). This is the only URL you may construct; don't invent other formats.

```
PARTNER BOOK PULSE — [Day, Month D]
[N] active clients · [total MRR]/mo · [N] need attention · [N] with expansion signals[ · [N] with limited data]

━━ NEEDS ATTENTION ([N])
• [Client name] — usage [score] · [MRR]/mo
  [products] · [churn signal / usage declining / renews in X days / managed relationship lapses [date] — [MRR]/mo at risk]
+ [N] more

━━ HEALTHY ([N])
[names if 5 or fewer, otherwise just the count]

Top 3 by MRR: [name] [MRR] · [name] [MRR] · [name] [MRR]
```

## Phase 4 — What next?

Close with only the lines that match what the pulse found — these are handoffs into the other partner skills, not work to do here. Health, renewal, and limited-data lines appear only when their count > 0. The **expansion** and **tell-me-about** lines always appear — expansion because the deeper skill checks seat/credit capacity that the ML signal (and this pulse) never looks at, so there may be opportunities beyond the ML count.

```
What next?
🔴 "health scan" — churn analysis for the [N] at-risk clients
📅 "renewal radar" — the [N] renewals in the next 90 days
📈 "expansion opportunities" — [N] flagged by ML signals (incl. cross-sell); it also finds seat/credit-maxed
🔧 "why the limited data?" — look into the [N] clients with limited data
🔎 "tell me about [client]" — drill into one
```

## Phase 5 — Limited-data investigation (on request)

Run only if the partner asks why clients have limited data. This is a troubleshooting path — don't pull extra properties upfront.

For each `limitedData` client, check `hs_is_managed` (already fetched in Phase 1):
- **`hs_is_managed` = false** → that's the reason: the client isn't managed, so the record is only minimally populated. No further lookup needed.
- **`hs_is_managed` = true** → managed but data isn't flowing. Only now, pull `hs_available_property_groups` for that record and read which groups the client shares:
  - **Some groups missing / restricted** → a **partner data-sharing opt-out**. Explain that the client has limited what they share, and point the partner to the KB article: https://knowledge.hubspot.com/privacy-and-consent/manage-partner-data-sharing-settings
  - **All groups present / shared, yet the data is still empty** → this is *not* explained by an opt-out and isn't something the partner can fix. Encourage them to submit feedback (via the HubSpot feedback option, if available) so the gap can be investigated.

Summarise briefly, e.g. "3 of the 5 aren't managed; 1 has restricted data sharing; 1 is fully shared but empty — worth reporting." Use `hs_available_property_groups` on this path only.

## Error handling

- **No active clients**: "No active PARTNER_CLIENT records in this portal. Make sure you're connected to your partner portal."
- **Fetch fails**: report the error and suggest checking the HubSpot connection.
