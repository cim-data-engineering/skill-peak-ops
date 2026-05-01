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

The work happens in four phases. Move through them in order; do not skip phase 1 — every later step depends on resolved `site_id`s.

Track progress with this checklist as you work:

- [ ] Phase 1: `site_id` resolved for every named site
- [ ] Phase 2: alert tickets pulled for both priorities, all pages, every site
- [ ] Phase 3: equipment batch-resolved
- [ ] Phase 4: per-alert records built and table rendered

### Phase 1 — Resolve site_ids

Call `search_sites` with `display_name` set to each named site. If the top result has `score >= 0.9`, take it; otherwise show the user the top 3 candidates and ask them to confirm.

### Phase 2 — Pull alert tickets (paginated)

For each site, run `search_alert_tickets` twice — once with `priority: 1`, once with `priority: 2` — passing `site_ids: [<id>]`, `status: "fault"`, `include_archived: false`, `limit: 500`, `start_index: 0`. After each call, read `pagination.has_more` and loop with an updated `start_index` until `has_more === false`. This guarantees complete counts. Concatenate the two priority result sets.

**Pagination safety cap**: if a single `(site, priority)` pair has made 20 paginated calls without `has_more` going false, stop, report the partial total to the user, and flag the site as incomplete. A misbehaving connector should not put you in an infinite loop.

The bucket rule (`hours_in_current_state >= 720` → `"> 30 days"`) is defined in Phase 4 — apply it as you go through alerts, but the canonical definition lives there.

### Phase 3 — Resolve equipment

Collect every `equipment_ids[0]` (the **first** equipment_id only — see "Multi-equipment alerts" below) into a single deduped list. Call `search_equipment` with `equipment_ids: [...]` in one batch. The response gives you `name`, `equipment_type_name`, `equipment_type_code`, `zone_name`, and `equipment_link` for each. You generally **don't need** `search_equipment_types` — `search_equipment` already returns the human-readable type name. Only fall back to `search_equipment_types` if you encounter an `equipment_type_id` that wasn't enriched.

If an `equipment_id` doesn't resolve (deleted, virtual, out of scope), **silently skip** that alert. Don't pollute the output with "Unknown" rows.

### Phase 4 — Aggregate and render

Build a per-alert record with these derived fields:

| Field | Source / rule |
|---|---|
| `site_name` | from `search_alert_tickets` |
| `equipment_id` | first item of `equipment_ids` array |
| `equipment_name` | resolved from Phase 3 |
| `equipment_type_name` | resolved from Phase 3 |
| `priority_label` | `1 → "P1 Critical"`, `2 → "P2 Urgent"` |
| `time_in_fault_days_bucket` | `hours_in_current_state >= 720 → "> 30 days"` else `"< 30 days"` (note the space — match column headers exactly) |
| `is_assigned` | `assignee != null` (today this is always false — forward-compatible) |
| `impact` | `"—"` placeholder (not yet wired through MCP) |
| `assignee_label` | `assignee` if non-null, else `"—"` |
| `updated_relative` | relative time ago between `alert_updated_at` and now (e.g. `"1 hour ago"`, `"5 days ago"`) — see `references/field_mapping.md` for the unit thresholds |
| `ticket_link_md` | `[View](alert_link)` |

Then render the requested table per `references/table_templates.md`.

**Sort orders** (these match the demo XLSX):

- Summary tables (by site / equipment type / equipment name): primary `> 30 days` count desc, tiebreak `Total` desc.
- Alert list: `alert_updated_at` desc (sort by the underlying timestamp, not the rendered relative string).

Always append a `Total` row to summary tables that sums `> 30 days`, `< 30 days`, `Total`, `Assigned`, and computes `% Assigned = Assigned / Total` (display `0%` if Total is 0).

After rendering a summary table, append this hint line so the user knows what to ask next:

> _Ask me to drill into a site by equipment type, equipment name, or the full alert list._

For the equipment-name drilldown, the user can scope to a site only or to a site + equipment type. Title the table accordingly:

- Site only: `[site_name]: P1-2 alerts in fault by equipment name`
- Site + type: `[site_name] / [equipment_type_name]: P1-2 alerts in fault by equipment name`

## Multi-equipment alerts

Alerts occasionally have multiple `equipment_ids` (e.g., `"CH-302,Common_CHWS-3 - Inspect Unit Fail"` carries `[721554505777, 721554505869]`). To keep the math simple and unambiguous, **use `equipment_ids[0]` only** for both the equipment-name and equipment-type tables. Each alert is counted exactly once. The displayed equipment name and type are those of the first equipment in the array.

## Output discipline

- Return **only** the requested table(s) plus the optional drilldown hint. No commentary, no recommendations, no executive summary.
- Below every alert list and below every summary table that involves Impact or Assignee, include this footnote (single line):
  > _Impact and Assignee are not yet wired through the PEAK MCP — placeholders shown until those fields are exposed._
- Numbers are integers; percentages are rendered with no decimal place (e.g., `26%`).
- The `#` column auto-numbers from 1 within each table. Don't include `#` on the Total row.

## Field reference

`references/field_mapping.md` lists every column in every table and the exact MCP path it maps from. Read it when you need to confirm a field.

## Tool sequence summary

```
search_sites              (Phase 1, once per named site)
search_alert_tickets      (Phase 2, twice per site: priority=1, priority=2; loop until has_more=false)
search_equipment          (Phase 3, once with batched equipment_ids)
search_equipment_types    (Phase 3 fallback, only if needed)
```

Avoid `execute_graphql_query` and `introspect_graphql_schema` — the four tools above cover every field this skill needs.
