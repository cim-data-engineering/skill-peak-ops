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

## Table 1 — by site

| Column | Source | Rule |
|---|---|---|
| `Site` | AT | `AT.site_name`, grouped |
| `> 30 days` | derived | count of alerts in group with bucket = `"> 30 days"` |
| `< 30 days` | derived | count of alerts in group with bucket = `"< 30 days"` |
| `Total` | derived | `> 30 days` + `< 30 days` |
| `Assigned` | derived | count of alerts in group with `is_assigned == true` |
| `% Assigned` | derived | `Assigned / Total`, rendered with no decimal place; `0%` if Total is 0 |

## Table 2 — by equipment type (per site)

| Column | Source | Rule |
|---|---|---|
| `Equipment type` | AT | `equipment_type_primary`, grouped |
| Remaining columns | derived | same rules as Table 1 |

## Table 3 — by equipment name (per site, optionally also per equipment type)

| Column | Source | Rule |
|---|---|---|
| `Equipment name` | AT | `equipment_name_primary`, grouped. Optional pre-filter by `equipment_type_primary` |
| Remaining columns | derived | same rules as Table 1 |

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

- **Empty `equipment_names` / `equipment_types`**: drop the alert. Don't surface "Unknown" rows.
- **No alerts in a group**: skip that row entirely from the table — don't render zero-count rows.
- **Site with zero P1-2 faults**: still include in Table 1 with all zeros, so the user sees the site is healthy.
