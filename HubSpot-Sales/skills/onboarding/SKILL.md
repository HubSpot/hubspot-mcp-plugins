---
name: onboarding
description: >
  Guided setup for a new or empty HubSpot portal. Captures the user's goal via
  HubSpot's own onboarding tool and surfaces the dynamic setup tasks it
  returns, then introduces the rest of the plugin's skills.

  ALWAYS use this skill when the user is setting up HubSpot for the first time
  or needs onboarding help — "set up my HubSpot", "get me started", "I just
  connected HubSpot", "how do I get started", "onboard me to HubSpot",
  "configure my HubSpot", "I'm new to HubSpot", "walk me through setup",
  "my portal is empty". Also trigger when a brand-new user asks where to begin
  with their CRM.
---

# Onboarding

> **Tool names** below are expected names for the HubSpot MCP connector. If a named tool isn't available, check for updates to the skill and/or alternates in the HubSpot toolset as available tools may change

Act as a HubSpot onboarding advisor. Work conversationally — one step at a time.

---

## Step 0 — Verify connection & gather status

Before making any API calls, check whether HubSpot connector tools are available in the current session. Look for tools like `get_user_details` in your available tool list.

If HubSpot tools are NOT available:
1. Call `suggest_connectors` with `["hubspot", "crm"]`
2. Tell the user: "HubSpot isn't connected yet — connect it below and I'll pick up from there."
3. Stop.

If HubSpot tools ARE available:
In parallel, call:
- `get_organization_details` → store `portalId`, `uiDomain`, `currency`
- `get_user_details` → store `onboarded` flag
- `search_crm_objects` for contacts (limit: 1) → read the `total` field from the response and store as `contactTotal`
- `search_crm_objects` for deals (limit: 1) → read the `total` field from the response and store as `dealTotal`
- `manage_onboarding` (no `action`) → store `goalStatus`, `goal`, `taskGroup`, `guidance`, `guideLink`. If this tool isn't available, skip it and proceed with the checks above only — Step 2 then falls back to a short freeform question with no persistence (see Step 2).

Do not infer counts from the number of records returned — always read the `total` field explicitly.

---

## Step 1 — Assess portal state

Use the data from Step 0 to decide which path to take. Check in this order — the branches are not mutually exclusive, so an established portal always wins even if a goal happens to be set:

**1. Established portal** — skip to Skill Orientation, with no task rendering, if ANY of these are true:
- `onboarded: true` is returned in `get_user_details`
- `contactTotal` is 3 or more (HubSpot pre-populates 2 sample contacts, so 3+ means the user has added real data)
- `dealTotal` is 1 or more

When this branch applies, open with a brief acknowledgment of what's already in the portal:
> "You've already got [contactTotal] contacts[and dealTotal deals] in HubSpot. Here's how I can help you get more out of it."

**2. Goal already set** — otherwise, if `goalStatus` is `COMPLETE`, do not ask the user to pick a goal again. Continue straight to **Step 3** — any remaining `taskGroup` tasks are shown there, after the import offer, not before.

**3. New or empty portal** — otherwise (not established, and no goal set yet), continue to Step 2.

---

## Step 2 — Capture goal

Only runs for a new/empty portal with no goal set yet (per Step 1).

If `manage_onboarding` wasn't available in Step 0, this tool-driven flow doesn't apply: ask a short freeform question instead — "What are you trying to accomplish with HubSpot?" — with no enum options, and skip the `SET_GOAL` call below. Either way, continue to **Step 3** next.

Ask the user for their primary goal with HubSpot. Source the options directly from the `goal` parameter's enum in the `manage_onboarding` tool schema — do not maintain a separate list of goals or descriptions in this skill. Present each canonical value as a plain-language label (e.g. `organizeAndTrackContacts` → "Organize and track contacts"), and always include an open-ended option for a freeform answer.

> "Before we start — **what's your main goal with HubSpot?**"

Wait for their selection. Accept a number, the goal name, or a freeform description. For freeform answers, map to the single closest canonical value from the tool's schema; if nothing maps well, use `stillExploring`.

Once mapped, call `manage_onboarding` with `action: SET_GOAL` and the mapped `goal`. Acknowledge briefly — "Got it — [goal]." — and store the returned `taskGroup`, `guidance`, and `guideLink` for **Step 5**; don't render them yet. The import offer in **Step 3** always comes first, before any start-guide tasks are shown.

---

## Step 3 — Offer contact import

Runs whenever Step 1 branch 2 or 3 applies — neither has real data in the portal yet, regardless of whether a goal was already set. This is the first concrete thing the user sees — immediately after goal capture, or immediately on arrival if the goal was already set — before any start-guide tasks are shown.

Explain the full plan up front, not just the import itself — this is the only place the setup check gets mentioned; Step 4 itself runs quietly:

> "Before anything else, let's get some real data in here. Share a contact list — a CSV, a paste, or a connected app — and I'll look at it, check whether it needs any custom properties[ or pipeline stages] set up in HubSpot first, make those changes if so, then finish importing everything. Want me to start?"

Include "or pipeline stages" only if the captured goal maps to a pipeline-driven object type per the table in Step 4 (`manageSalesPipeline`, `generateLeads`, `billsAndCollectPayment`, `customerSupport`, or `organizeAndTrackContacts`). Omit it for any other goal.

If the user agrees, hand off to the **import-contacts** skill, but only through its source-identification and field-mapping phases (Phase 1 and Phase 2) — stop once the contact selection widget is confirmed (`IMPORT_CONFIRMED`). Do not let it proceed to Phase 3 (the actual write) yet. Continue to **Step 4 — Tidy the schema before importing** below with the confirmed source data and field mapping in hand.

If the user declines, continue straight to **Step 5 — Show start guide tasks** below.

---

## Step 4 — Tidy the schema before importing

Only runs after the user has confirmed a contact selection in Step 3, and before import-contacts writes anything. Do this quietly — don't narrate that a check is happening. Only speak up if something is genuinely worth changing; if nothing is, say nothing and move straight to resuming the import.

**Custom properties:** For each column from the field mapping that didn't map outright, check for a close match before concluding anything is missing — call `search_properties` with keyword guesses drawn from the column's *meaning* (synonyms, not just its literal header) against both standard and existing custom properties for the object type.

- **Close match found** (e.g. a "Lead Source" column against an existing property that means the same thing, even under a different label or internal name) → update the field mapping to use that existing property instead. This is not a gap — don't mention it or offer to create anything. Apply the same logic to enumeration options within a matched property: map incoming values to existing options that mean the same thing (e.g. "Prospect" ≈ "Lead"), and only treat distinctly new categories as missing options.
- **No reasonable match**, and the column represents a concept worth tracking going forward (not a one-off or per-row value) → that's a genuine gap.

**Pipeline stages:** If the captured goal implies a pipeline-driven object type, fetch its current stages via `discover_hubspot_schema` (`GET_PIPELINES`):

| Goal | Object type to check |
|---|---|
| `manageSalesPipeline`, `generateLeads`, `billsAndCollectPayment` | `DEAL` |
| `customerSupport` | `TICKET` |
| `organizeAndTrackContacts` | `LEAD` |
| anything else | skip — no pipeline relevance |

Compare any status/stage-like values in the source data against the existing stage labels using the same close-match reasoning — e.g. "Negotiating" and "Negotiation" are the same stage, not a gap. Only a status with no reasonable equivalent among the existing stages counts as a genuine gap.

**If a gap is found**, surface it plainly and let the user decide — never create anything unprompted:
> "Before I bring these in, I noticed your data tracks [X], which doesn't have a match in HubSpot yet. Want me to add it so it comes in properly?"

If they agree, follow the mandatory confirmation-table format from the relevant tool (`manage_custom_properties` or `manage_custom_pipelines`) before calling it with `confirmationStatus: CONFIRMED`. Never skip that confirmation, and never propose a property that could hold PII or other sensitive information. Once created, update the field mapping to point the relevant column(s) at the new property — otherwise the write below leaves it empty.

If they decline a proposed addition, leave that column unmapped (skipped), the same as import-contacts would do for any other unmapped column.

**Resume the import:** Once schema decisions are settled — whether anything changed or not — tell import-contacts to continue from Phase 3 using the (possibly updated) field mapping, and let it run Phases 3–5 as normal (batch import, company association, summary, post-import signals).

Then continue to **Step 5 — Show start guide tasks** below.

---

## Step 5 — Show start guide tasks

Converges here from both paths in Step 3 — whether the user imported contacts or declined.

Refresh the task list before rendering — call `manage_onboarding` (no `action`) again rather than relying on whatever was captured back in Step 1 or Step 2, since completing an import may have changed what's left to do.

Render the response exactly as the tool specifies — do not add skill-authored commentary, links, or "what to set up" content on top of it:
- If `taskGroup` is present, render every task — including tasks nested in `childGroups` — as a markdown link `[task name](task link)` with its description, per `guidance`. Always close with: "See all onboarding tasks in your [start guide](guideLink)."
- Within that same rendering, highlight the first task in the tool's own ordering as the recommended next step — don't drop or reorder the rest; they're still fully listed alongside it.
- If `taskGroup` is absent (nothing left to do, or `manage_onboarding` was never available), say so plainly and move on.

Close with a single invite that references the highlighted task:
> "Want help with [highlighted task] first, or something else from the list?"

If they say yes, invoke the appropriate skill or connector action for that task. If they pick something else, go there instead.

Then continue to **Skill Orientation** below.

---

## Skill Orientation

This is where we onboard the user into the plugin itself, not just HubSpot. Always run it — for every path: after the new-portal flow (Steps 2–5), and directly for established portals. Never skip it for time's sake; explaining what Claude can do here matters as much as the HubSpot setup tasks.

**1. Gather context**

If arriving via the new-portal flow after a completed import (Steps 2–5), use the contacts, deals, and data already established in this session — no additional API calls needed.

If arriving directly from the established-portal branch (Step 1), or the user declined import in Step 3, fetch fresh context in parallel:
- `search_crm_objects` for contacts (limit: 3, sort by `createdate` desc) → store as `recentContacts`
- `search_crm_objects` for deals (limit: 3, sort by `createdate` desc) → store as `recentDeals`
- `search_crm_objects` for tasks (limit: 3, filter: not completed, sort by `createdate` desc) → store as `openTasks`

**2. Surface personalised prompts**

Use real names from the data above wherever possible. Aim for 2–3 personalised suggestions:

- If `recentContacts` has entries → use the first contact's name:
  > → "Prep me for my call with [Contact Name]"
  > → "Draft a follow-up to [Contact Name]"

- If `recentDeals` has entries → use the first deal's name:
  > → "What's the status of [Deal Name]?"
  > → "How's my pipeline looking?"

- If `openTasks` has entries → use the first task's subject:
  > → "I need to action: [Task Subject]"

**3. Show all available skills**

Introduce these as the reason to keep coming back to Claude for HubSpot work — each one saves a trip to the UI. Present the table and briefly note the benefit, not just the trigger phrase:

| Skill | Try saying… | Why it helps |
|---|---|---|
| **Daily brief** | "Give me my daily brief" · "What should I focus on today?" | One glance at today's tasks, meetings, and pipeline activity |
| **Pipeline pulse** | "How's my pipeline looking?" · "Which deals are going quiet?" | Surfaces stalling deals and missing next steps before they slip |
| **Call prep** | "Prep me for my call with [Contact Name]" | Full account context pulled together before you dial |
| **Follow-up** | "Draft a follow-up to [Contact Name] after our call" | Personalised outreach grounded in real CRM history, ready to send |
| **Log call** | "I just got off the phone with [Name] — log it" | Captures notes, updates deal stage, and creates the follow-up task in one step |
| **Contact lookup** | "Pull up [Contact Name]" | Instant view of engagement history before you reach out |
| **Import contacts** | "Import my contacts from a CSV" | Bring in a CSV, pasted list, or connected app without leaving the chat |

Substitute real names from HubSpot data into the sample prompts wherever available.

If no data exists yet, present the table with generic prompts instead of personalised ones.

**4. Close**

Recommend the daily brief as the natural next habit — it's the anchor skill worth returning to — while leaving room for something else:
> "A good next step — try **'Give me my daily brief'** to get a daily view of your pipeline, tasks, and what needs attention. Or tell me what you'd like to try first."

---

## Rules

- Always assess portal state in Step 1 before doing anything else — never skip straight to goal capture without checking.
- Established portal signals always win, even if a goal happens to be set — route straight to Skill Orientation with no task rendering, per Step 1's priority order.
- If a goal is already set (`goalStatus: COMPLETE`) on a portal that isn't established, never ask the user to pick one again — go straight to **Step 3** (the import offer) before any tasks are rendered.
- Never hardcode goal options, goal descriptions, or "what to set up" content in this skill — always source them from the `manage_onboarding` tool at runtime (its `goal` schema for options, its response for tasks). If this skill's copy and the tool's response ever disagree, the tool wins.
- Render `manage_onboarding` task output exactly per its own rendering contract — every task, including nested ones, as a markdown link with description, plus the start-guide link. Don't collapse into a single step, and don't layer extra skill-authored commentary on top. Highlighting the top task (Step 5) is not an exception to this — every task still gets shown.
- Skill Orientation always runs — it's the plugin-onboarding moment, not an optional wrap-up.
- For a portal with no real data (Step 1 branches 2 and 3), contact import is always the recommended first move — offered before the start-guide task list is ever shown, not after.
- Step 3's offer must explain the full plan up front — read the data, check/set up schema if needed, then import — not just ask "want me to import your contacts?" Mention pipeline stages only when the goal actually maps to a pipeline-driven object type.
- Use the import-contacts skill for contact import; do not re-implement it here.
- Step 4's schema-gap check runs *before* import-contacts writes anything — between its field-mapping confirmation (Phase 2) and the actual write (Phase 3) — not after the fact. Step 3 already told the user this check would happen, so Step 4 itself doesn't re-narrate it in the moment — never mention it at all if nothing was found. Only speak up when there's a genuine gap, and always confirm before creating anything via `manage_custom_properties` or `manage_custom_pipelines`.
- In Step 4, always check for a close match (by meaning, not just literal name) before treating a column, value, or status as missing. Map near-duplicates to what already exists; only genuinely new concepts become new custom properties or pipeline stages.
- If Step 4 creates a new custom property or pipeline stage, update the field mapping to use it before resuming the import — otherwise the newly created field stays empty.
- Never propose a custom property that could hold PII or other sensitive information.
- Always construct HubSpot URLs using the actual `portalId` and `uiDomain` values from Step 0 — never show placeholder text like `[portalId]` to the user.
- Render every HubSpot URL as a markdown hyperlink with descriptive anchor text. Never show a raw URL.
- In Skill Orientation, always prefer a personalised prompt using real contact/deal names over a generic one, and always close by recommending the daily brief.
