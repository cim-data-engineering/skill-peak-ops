# Field mapping

Authoritative map of each table column to its MCP source path and any derivation rule. Use this when you need to confirm where a value comes from.

## Sources

- **AT** = `search_alert_tickets` result item
- **derived** = computed locally per the rule below

Every field this skill needs is now on the alert ticket itself. No equipment-side enrichment calls are required.

## Bucket and label rules (apply once, reuse across tables)

| Derived field | Rule |
|---|---|
| `time_in_fault_days_bucket` | `AT.time_in_fault_hours >= 720` → `"> 30 days"`; else `"< 30 days"`. If `time_in_fault_hours` is null/missing, bucket as `"< 30 days"` (don't drop the row — that would skew totals invisibly). |
| `priority_label` | `AT.priority` verbatim — wire already returns the label string (`"P1 Critical"`, `"P2 Urgent"`). The integer 1/2 form is **input-only** (filter param); never appears in responses. |
| `is_assigned` | `AT.assignee != null` (true even if `AT.assignee.name` is blank — someone's been assigned, just not properly named) |
| `assignee_label` | `AT.assignee.name` if non-null and non-empty; `"(unnamed)"` if `AT.assignee` exists but name is null/blank; `"—"` if `AT.assignee` itself is null |
| `impact` | `AT.impact` is an array of enum strings (e.g. `["reliability"]`, `["energy"]`, `[]`). Take `AT.impact[0]` and render: verbatim if already mixed-case, else title-cased (`"reliability"` → `"Reliability"`). `"—"` if the array is empty or missing. If the array has multiple values (rare), join with `", "` after applying the case rule per element. |
| `updated_relative` | relative time between `AT.updated_at` and now, formatted as `"X unit ago"`. Unit thresholds (use the largest unit that fits, floor the value): <br>• `< 60 sec` → `"just now"` <br>• `< 60 min` → `"N minute(s) ago"` <br>• `< 24 hr` → `"N hour(s) ago"` <br>• `< 30 days` → `"N day(s) ago"` <br>• `< 12 months` → `"N month(s) ago"` (1 month = 30 days) <br>• otherwise → `"N year(s) ago"` (1 year = 365 days) <br>Singular for `N == 1` (`"1 hour ago"`), plural otherwise (`"5 days ago"`). |
| `ticket_link_md` | `"[View](" + AT.alert_link + ")"` |
| `equipment_name_primary` | `AT.equipment_names[0]` |
| `equipment_type_primary` | `AT.equipment_types[0]` |
| `site_alert_search_url` | PEAK alert-search URL filtered to one site, scoped to the active priority and status filters: `https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=<AT.site_id>&<priority_pins>&<status_pins>&archived=<include_archived>`. Priority pins: repeat `priority_ids=N` once per active priority (P1-2 default → `priority_ids=1&priority_ids=2`; P1-3 override → `priority_ids=1&priority_ids=2&priority_ids=3`). Status pins: repeat `status_ids=<status>` once per active status (default → `status_ids=fault`; recovered override → `status_ids=fault&status_ids=recovered`). `archived=true` only when the user explicitly opts in. |
| `site_link_md` | `"[View](" + site_alert_search_url + ")"` — used in Table 1's trailing `Link` cell. Total row's `Link` cell stays empty. |
| `equipment_alert_search_url` | PEAK alert-search URL for one equipment-name group, scoped to the table's site and **all** alerts on that equipment (no priority/status pins — wider scope than `site_alert_search_url`): `https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=<site_id>&<equipment_pins>&archived=false`. Equipment pins: take the union of `AT.equipment_ids[0]` across the alert tickets in the group (i.e., the IDs that line up with `equipment_names[0]`), dedupe, and emit one `equipment_ids=<id>` query param per ID. `archived=false` always — opens the search live in the user's browser, where they can flip the filter themselves. |
| `equipment_link_md` | `"[View](" + equipment_alert_search_url + ")"` — used in Table 3's trailing `Link` cell. Total row's `Link` cell stays empty. |
| `equipment_type_alert_search_url` | PEAK alert-search URL for one equipment-type group, scoped to the table's site, the active priority/status filters, **and** the type: `https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=<site_id>&<priority_pins>&<status_pins>&metadata_type_ids=<type_id>&archived=<include_archived>`. Priority pins reflect the **table's** active priority set, not the global override — Table 2 P1-2 emits `priority_ids=1&priority_ids=2`; the P3-5 companion table (rendered for single-site scope) emits `priority_ids=3&priority_ids=4&priority_ids=5`. Status pins and `archived` follow the active filters (same rule as `site_alert_search_url`). `<type_id>` = lookup of `equipment_type_primary` in `references/equipment_type_ids.md`. If the name doesn't resolve, emit no link for that row (empty cell). |
| `equipment_type_link_md` | `"[View](" + equipment_type_alert_search_url + ")"` — used in Table 2's trailing `Link` cell. Empty on the Total row, and empty on any row whose type name doesn't resolve in `equipment_type_ids.md`. |

## Table 1 — by site

| Column | Source | Rule |
|---|---|---|
| `Site` | AT | `AT.site_name`, grouped by `AT.site_id` / `AT.site_name` |
| `> 30 days` | derived | count of alerts in group with bucket = `"> 30 days"` |
| `< 30 days` | derived | count of alerts in group with bucket = `"< 30 days"` |
| `Total` | derived | `> 30 days` + `< 30 days` |
| `Assigned` | derived | count of alerts in group with `is_assigned == true` |
| `% Assigned` | derived | `Assigned / Total`, rendered with no decimal place; `0%` if Total is 0 |
| `Link` | derived | `site_link_md` — markdown `[View](url)` to the PEAK alert search filtered to that site, priority, and status. Empty on the Total row. |

## Table 2 — by equipment type (per site)

| Column | Source | Rule |
|---|---|---|
| `Equipment type` | AT | `equipment_type_primary`, grouped |
| `> 30 days` / `< 30 days` / `Total` / `Assigned` / `% Assigned` | derived | same rules as Table 1 |
| `Link` | derived | `equipment_type_link_md` — markdown `[View](url)` to the PEAK alert search filtered to this site, the table's active priorities/status, and the equipment type's `metadata_type_ids`. Empty on the Total row, and empty on any row whose type name doesn't resolve in `equipment_type_ids.md`. |

## Table 3 — by equipment name (per site, optionally also per equipment type)

| Column | Source | Rule |
|---|---|---|
| `Equipment name` | AT | `equipment_name_primary`, grouped. Optional pre-filter by `equipment_type_primary` |
| `> 30 days` / `< 30 days` / `Total` / `Assigned` / `% Assigned` | derived | same rules as Table 1 |
| `Link` | derived | `equipment_link_md` — markdown `[View](url)` to the PEAK alert search filtered to this site and the equipment IDs in the group. Empty on the Total row. |

## Table 4 — alert list (per site)

| Column | Source | Rule |
|---|---|---|
| `Title` | AT | `AT.title` verbatim |
| `Equipment type` | AT | `equipment_type_primary` |
| `Priority` | derived | `priority_label` |
| `Impact` | derived | `impact` |
| `Assignee` | derived | `assignee_label` |
| `Updated` | derived | `updated_relative` |
| `Ticket link` | derived | `ticket_link_md` |

Sort: `AT.updated_at` desc.

## Edge cases

- **Empty `equipment_names` / `equipment_types`**: drop the alert ticket. Don't surface "Unknown" rows.
- **No alerts in a group**: skip that row entirely from the table — don't render zero-count rows.
- **Site with zero P1-2 faults**: still include in Table 1 with all zeros, so the user sees the site is healthy.
