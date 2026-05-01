# Field mapping

Authoritative map of each table column to its MCP source path and any derivation rule. Use this when you need to confirm where a value comes from.

## Sources

- **AT** = `search_alert_tickets` result item
- **EQ** = `search_equipment` result item (joined on `equipment_id`)
- **ET** = `search_equipment_types` result item (rarely needed; only as fallback)
- **derived** = computed locally per the rule below

## Bucket and label rules (apply once, reuse across tables)

| Derived field | Rule |
|---|---|
| `time_in_fault_days_bucket` | `AT.hours_in_current_state >= 720` → `"> 30 days"`; else `"< 30 days"` |
| `priority_label` | `AT.priority_id == 1` → `"P1 Critical"`; `AT.priority_id == 2` → `"P2 Urgent"` |
| `is_assigned` | `AT.assignee != null` (forward-compatible; today always false) |
| `assignee_label` | `AT.assignee` if non-null else `"—"` |
| `impact` | `"—"` (placeholder; not yet wired through MCP) |
| `updated_relative` | relative time between `AT.alert_updated_at` and now, formatted as `"X unit ago"`. Unit thresholds (use the largest unit that fits, floor the value): <br>• `< 60 sec` → `"just now"` <br>• `< 60 min` → `"N minute(s) ago"` <br>• `< 24 hr` → `"N hour(s) ago"` <br>• `< 30 days` → `"N day(s) ago"` <br>• `< 12 months` → `"N month(s) ago"` (1 month = 30 days) <br>• otherwise → `"N year(s) ago"` (1 year = 365 days) <br>Singular for `N == 1` (`"1 hour ago"`), plural otherwise (`"5 days ago"`). |
| `ticket_link_md` | `"[View](" + AT.alert_link + ")"` |
| `equipment_id_primary` | `AT.equipment_ids[0]` |

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
| `Equipment type` | EQ | `EQ.equipment_type_name`, looked up via `equipment_id_primary`, then grouped |
| Remaining columns | derived | same rules as Table 1 |

## Table 3 — by equipment name (per site, optionally also per equipment type)

| Column | Source | Rule |
|---|---|---|
| `Equipment name` | EQ | `EQ.name`, looked up via `equipment_id_primary`, then grouped. Optional pre-filter by `EQ.equipment_type_name` |
| Remaining columns | derived | same rules as Table 1 |

## Table 4 — alert list (per site)

| Column | Source | Rule |
|---|---|---|
| `Title` | AT | `AT.alert_summary` verbatim |
| `Equipment type` | EQ | `EQ.equipment_type_name` looked up via `equipment_id_primary` |
| `Priority` | derived | `priority_label` |
| `Impact` | derived | `impact` placeholder |
| `Assignee` | derived | `assignee_label` |
| `Updated` | derived | `updated_relative` |
| `Ticket link` | derived | `ticket_link_md` |

Sort: `AT.alert_updated_at` desc.

## Edge cases

- **Equipment lookup miss**: If `equipment_id_primary` is missing from the `search_equipment` response, drop the alert. Don't surface "Unknown" rows.
- **Empty `equipment_ids`**: Treat as a lookup miss (skip).
- **No alerts in a group**: Skip that row entirely from the table — don't render zero-count rows.
- **Site with zero P1-2 faults**: Still include in Table 1 with all zeros, so the user sees the site is healthy.
