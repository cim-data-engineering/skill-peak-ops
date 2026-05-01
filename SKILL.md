---
name: peak-ops
description: Portfolio operations dashboard over PEAK fault-detection alerts. Only invoke when the user explicitly runs the /peak-ops slash command. Do not auto-trigger on related keywords, topics, or PEAK-adjacent questions. Once invoked, remain active for the rest of the session to handle follow-up drilldowns (by site, equipment type, equipment name, or alert list) without requiring re-invocation.
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

The skill resolves a set of `site_id`s before pulling any alerts. There are three ways the user can specify scope:

**Specific sites named** — work over those. (e.g. "alerts at Building 12 and 123 Main St")

**Client / portfolio / customer named** — resolve the client, expand to its sites. Cue words: `portfolio`, `customer`, `client`, `across <X> sites`, `all of <X>'s sites`, `<X>'s sites`. (e.g. "What requires attention across GPT Office sites?")

**Bare "my portfolio" / "show me my sites" with no name** — discover all sites the user has access to. Call `search_sites()` with no filter and `limit: 51`. If 50 or fewer come back, proceed with all of them. If 51 come back (the user has access to more than 50), stop and ask the user to scope by client, region, or a specific site list — don't fetch further. Pulling P1+P2 alerts across a 50+ portfolio is slow and the resulting table is unreadable.

**Name ambiguity** — If a query doesn't carry a client cue but `search_sites(display_name=X)` returns no result with `score >= 0.5`, fall back to `search_clients(client_name=%X%)` before giving up. If both `search_sites` and `search_clients` return strong matches for the same bare name (e.g. "Westfield" is both a parent and a building), ask the user which they meant.

**Mixed scopes deferred** — queries like "GPT Office sites except Building 12" or "GPT Office plus 123 Main St" are out of scope for v1. If the user mixes the two, ask them to pick one form.

The skill defaults alert-ticket filters to:

- `include_archived: false`
- `status: fault`
- `priority: 1` **and** `priority: 2` (the connector accepts a single priority per call, so make one call per priority and union the results)

Treat these as defaults, not knobs to expose proactively. The user can override them only with an **explicit** ask — vague language like "show me everything" doesn't count; ask for confirmation. Cues that flip a default:

- `"P3"`, `"P1-3"`, `"all priorities"`, `"include P4"` → adjust the priority list (still one `search_alert_tickets` call per priority)
- `"recovered"`, `"status: recovered"`, `"in fault and recovered"` → change or expand the `status` filter (one call per status; union results the same way as priorities)
- `"include archived"`, `"with archived"` → set `include_archived: true`

When an override is active, reflect it in the **table title** so the user can see the scope at a glance — e.g. `P1-3 alerts in fault by site`, `P1-2 alerts (fault + recovered) by site`, `P1-2 alerts in fault by site (incl. archived)`. Defaults render as today (`P1-2 alerts in fault by site`) — no banner needed.

## Workflow

The work happens in three phases. Move through them in order; do not skip phase 1 — every later step depends on resolved `site_id`s.

Track progress with this checklist as you work:

- [ ] Phase 1: scope resolved to a concrete `site_id` set (named sites, client expansion, or accessible portfolio under the 50-site cap)
- [ ] Phase 2: alert tickets pulled for both priorities, all pages, every site
- [ ] Phase 3: per-alert records built and table rendered

### Phase 1 — Resolve site_ids

The path depends on what the user named.

**Site path (named sites).** Call `search_sites` with `display_name` set to each named site. If the top result has `score >= 0.9`, take it; otherwise show the user the top 3 candidates and ask them to confirm. If no result reaches `score >= 0.5` and the query was a single bare name with no client cue, fall back to the client path before giving up.

**Client path (named client / portfolio).** Call `search_clients(client_name="%<name>%")`. If exactly one result comes back, or one result is an exact case-insensitive match on the user's query, take it. Otherwise list the top 3 candidates back to the user and ask which one — `search_clients` does not return a relevance score, so don't bluff one. Then call `search_sites(client_ids: [<client_id>], limit: 51)`. Apply the same 50-site cap as the bare-portfolio case: if 51 come back, stop and ask the user to narrow further before pulling alert tickets. If `search_sites` returns zero, surface that plainly (e.g. *"GPT Office has no sites in PEAK — did you mean a different client, or a specific site name?"*) — do not auto-fall-back to a site-name lookup.

**Bare-portfolio path (no name given).** Call `search_sites(limit: 51)` with no filter. ≤50 → use all of them. >50 → ask the user to scope by client, region, or specific sites.

### Phase 2 — Pull alert tickets (paginated)

`search_alert_tickets` accepts `site_ids` as an integer **array** — pass multiple resolved site_ids in a single call. There is no `client_id` filter on the tickets endpoint; multi-site batching via `site_ids` is the optimization. Run one call per `(priority, status)` × `site_chunk` combination in the active filter set — defaults are `priority ∈ {1, 2}` and `status ∈ {"fault"}`. Pass `include_archived: <active-value>`, `limit: 500`, `start_index: 0`. After each call, read `pagination.has_more` and loop with an updated `start_index` until `has_more === false`. Concatenate all result sets.

**Chunk `site_ids` to stay under the response cap.** Each ticket record is ~800 bytes on the wire and the gateway caps responses around ~80KB / ~100 records. A single batched call across 10 sites at P2 fault was observed to trip the cap (97 records). Use these heuristics:

- P1 fault: chunk size **20** (alerts are sparse — usually safe to batch the whole portfolio).
- P2 fault, P3 fault, recovered, archived included: chunk size **5**.
- If a chunked call still trips the cap, halve the chunk and retry the affected chunk only.

All chunks are independent — issue them in parallel within a single tool-use batch.

**Parallelise.** All initial `search_alert_tickets` calls — every (priority, status, site_chunk) combination — are independent. Issue them in a single tool-use batch (one message with multiple tool calls), not sequentially. Subsequent pagination calls can also be batched per tuple as long as `has_more === true`.

**Pagination safety cap**: if a single tuple has made 20 paginated calls without `has_more` going false, stop, report the partial total to the user, and flag the result as incomplete. A misbehaving connector should not put you in an infinite loop.

**Recovery from a tripped response cap.** If a call fails with a transport error (5xx, timeout, truncated response — gateway notes call this out), the gateway will have **saved the raw JSON to disk** and surfaced the file path in the error. Two recovery options, in order of preference:

1. **Slim the saved file with `jq` instead of refetching.** The skill only needs ~12 fields per alert (`ticket_id`, `title`, `site_id`, `site_name`, `equipment_names[0]`, `equipment_types[0]`, `priority`, `impact[0]`, `assignee`, `time_in_fault_hours`, `updated_at`, `alert_link`). A `jq '[.results[] | {…}]'` projection turns ~83KB into ~10KB — no extra tool calls, full record set preserved. For very large files, run the `jq` step inside an `Explore` subagent so the raw blob never enters the main context.
2. **Refetch with smaller chunks** only if the saved file is missing or unusable. Halve the chunk size for the affected (priority, status) tuple and retry just those chunks.

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
| `priority_label` | `AT.priority` verbatim (wire returns `"P1 Critical"` / `"P2 Urgent"`; the integer form is input-filter-only) |
| `time_in_fault_days_bucket` | `AT.time_in_fault_hours >= 720 → "> 30 days"` else `"< 30 days"` (note the space — match column headers exactly). Null/missing → `"< 30 days"`, never drop the row |
| `is_assigned` | `AT.assignee != null` (true even if name is blank) |
| `impact` | `AT.impact` is an array; take `[0]` and title-case unless already mixed-case. Empty/missing array → `"—"` |
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
search_clients            (Phase 1, when the user names a client / portfolio)
search_sites              (Phase 1, to resolve named sites OR expand a client to its sites OR discover the user's accessible sites)
search_alert_tickets      (Phase 2, one call per (priority, status, site_chunk) tuple — chunk 20 for P1, 5 for P2/P3; loop each until has_more=false)
```

Avoid `execute_graphql_query` and `introspect_graphql_schema` — the three tools above cover every field this skill needs. Equipment names and types now come back inside each alert ticket, so `search_equipment` and `search_equipment_types` are not used.
