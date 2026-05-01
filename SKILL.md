---
name: peak-portfolio-ops
description: Portfolio operations dashboard over PEAK fault-detection alerts. Surfaces what's broken, who's on it, and how long it's taking across a portfolio manager's sites. Trigger this skill whenever the user asks about open faults, alerts in fault, what requires attention, what's broken, what's outstanding, P1-2 priority alerts, or aged faults across one or more PEAK sites — and especially when they name multiple sites (e.g. "Charlestown Square", "Parkmore Shopping Centre", "Rouse Hill") or use phrases like "across my portfolio", "what's going on at...", "site health", or ask to drill into equipment types, equipment names, or the alert list. Also trigger when the user asks for a portfolio-ops summary, weekly readout, morning check-in, or any rollup over PEAK alert tickets.
---

# PEAK Portfolio Operations

## Purpose

Help a portfolio operations manager see, at a glance, **what's broken, who's on it, and how long it's taking** across all their PEAK-monitored sites. The skill returns markdown tables sourced from live PEAK MCP data — no opinions, no recommendations, just the numbers and the links.

The four output tables are:

1. `P1-2 alerts in fault by site` — portfolio rollup
2. `[site_name]: P1-2 alerts in fault by equipment type` — drilldown
3. `[site_name]: P1-2 alerts in fault by equipment name` — drilldown (optionally scoped to one equipment type)
4. `[site_name]: P1-2 alert list` — line-level detail

Exact markdown layouts are in `references/table_templates.md`. Read it before rendering any table.

## Triggering & scope

When the user names sites, work over those. When the user asks portfolio-wide ("across my portfolio", "all my sites") **without naming any**, ask which sites to include before running — the customer may have many sites and a silent default would be misleading.

The skill always filters alert tickets to:

- `archived: false`
- `status: fault`
- `priority: 1` **and** `priority: 2` (the connector accepts a single priority per call, so make two calls and union the results)

These are non-negotiable filters baked into every query. Don't expose them as configuration.

## Workflow

The work happens in three phases. Move through them in order; do not skip phase 1 — every later step depends on resolved `site_id`s.

Track progress with this checklist as you work:

- [ ] Phase 1: `site_id` resolved for every named site
- [ ] Phase 2: alert tickets pulled for both priorities, all pages, every site
- [ ] Phase 3: per-alert records built and table rendered

### Phase 1 — Resolve site_ids

Call `search_sites` with `display_name` set to each named site. If the top result has `score >= 0.9`, take it; otherwise show the user the top 3 candidates and ask them to confirm.

### Phase 2 — Pull alert tickets (paginated)

For each site, run `search_alert_tickets` twice — once with `priority: 1`, once with `priority: 2` — passing `site_ids: [<id>]`, `status: "fault"`, `include_archived: false`, `limit: 500`, `start_index: 0`. After each call, read `pagination.has_more` and loop with an updated `start_index` until `has_more === false`. This guarantees complete counts. Concatenate the two priority result sets.

**Parallelise.** All initial `search_alert_tickets` calls — both priorities, every site — are independent. Issue them in a single tool-use batch (one message with multiple tool calls), not sequentially. Same tool count, same tokens, but the wall-clock cost collapses from `2 × N_sites × per_call_latency` to roughly `1 × per_call_latency`. Subsequent pagination calls can also be batched per (site, priority) pair as long as `has_more === true`.

**Pagination safety cap**: if a single `(site, priority)` pair has made 20 paginated calls without `has_more` going false, stop, report the partial total to the user, and flag the site as incomplete. A misbehaving connector should not put you in an infinite loop.

Each ticket already carries `equipment_names` and `equipment_types` as string arrays, plus `impact`, `assignee`, and `time_in_fault_hours` — no follow-up enrichment calls are required.

### Reuse across drilldowns

Each `search_alert_tickets` result already carries every field the four tables need (`title`, `equipment_names`, `equipment_types`, `priority`, `impact`, `assignee`, `time_in_fault_hours`, `updated_at`, `alert_link`). When the user asks for a drilldown (Table 2, 3, or 4) on a site **already pulled earlier in the same conversation**, reuse the in-context result set — do **not** re-issue `search_alert_tickets`.

Re-fetch only if: the user asks for fresh data, the conversation has clearly moved on to other work and come back, or the drilldown is for a site that wasn't in the prior pull. Don't try to track wall-clock time — read the conversation context.

### Phase 3 — Aggregate and render

Build a per-alert record with these derived fields:

| Field | Source / rule |
|---|---|
| `site_name` | `AT.site_name` |
| `equipment_name` | `AT.equipment_names[0]` |
| `equipment_type_name` | `AT.equipment_types[0]` |
| `priority_label` | `1 → "P1 Critical"`, `2 → "P2 Urgent"` |
| `time_in_fault_days_bucket` | `AT.time_in_fault_hours >= 720 → "> 30 days"` else `"< 30 days"` (note the space — match column headers exactly). Null/missing → `"< 30 days"`, never drop the row |
| `is_assigned` | `AT.assignee != null` (true even if name is blank) |
| `impact` | `AT.impact` verbatim if mixed-case, else title-cased; `"—"` if null |
| `assignee_label` | `AT.assignee.name` if non-null/non-empty; `"(unnamed)"` if assignee exists but name is blank; `"—"` if assignee is null |
| `updated_relative` | relative time ago between `AT.updated_at` and now (e.g. `"1 hour ago"`, `"5 days ago"`) — see `references/field_mapping.md` for the unit thresholds |
| `ticket_link_md` | `[View](AT.alert_link)` |

Then render the requested table per `references/table_templates.md`.

**Sort orders** (these match the demo XLSX):

- Summary tables (by site / equipment type / equipment name): primary `> 30 days` count desc, tiebreak `Total` desc.
- Alert list: `AT.updated_at` desc (sort by the underlying timestamp, not the rendered relative string).

Always append a `Total` row to summary tables that sums `> 30 days`, `< 30 days`, `Total`, `Assigned`, and computes `% Assigned = Assigned / Total` (display `0%` if Total is 0).

After rendering a summary table, append this hint line so the user knows what to ask next:

> _Ask me to drill into a site by equipment type, equipment name, or the full alert list._

For the equipment-name drilldown, the user can scope to a site only or to a site + equipment type. Title the table accordingly:

- Site only: `[site_name]: P1-2 alerts in fault by equipment name`
- Site + type: `[site_name] / [equipment_type_name]: P1-2 alerts in fault by equipment name`

## Multi-equipment alerts

Alerts occasionally have multiple entries in `equipment_names` / `equipment_types` (e.g., `"CH-302,Common_CHWS-3 - Inspect Unit Fail"`). To keep the math simple and unambiguous, **use index `[0]` only** for both the equipment-name and equipment-type tables. Each alert is counted exactly once.

## Output discipline

- Return **only** the requested table(s) plus the optional drilldown hint. No commentary, no recommendations, no executive summary.
- Numbers are integers; percentages are rendered with no decimal place (e.g., `26%`).
- The `#` column auto-numbers from 1 within each table. Don't include `#` on the Total row.
- If `equipment_names` is empty for an alert, **silently skip** that alert. Don't pollute the output with "Unknown" rows.

## Field reference

`references/field_mapping.md` lists every column in every table and the exact MCP path it maps from. Read it when you need to confirm a field.

## Tool sequence summary

```
search_sites              (Phase 1, once per named site)
search_alert_tickets      (Phase 2, twice per site: priority=1, priority=2; loop until has_more=false)
```

Avoid `execute_graphql_query` and `introspect_graphql_schema` — the two tools above cover every field this skill needs. Equipment names and types now come back inside each alert ticket, so `search_equipment` and `search_equipment_types` are not used.
