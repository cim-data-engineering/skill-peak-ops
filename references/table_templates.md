# Table templates

Render exactly these markdown shapes. Replace `[bracketed]` placeholders with computed values. Keep column order, header text, and capitalization identical to what's shown — the user is comparing against a fixed XLSX layout.

---

## Table 1 — `P1-2 alerts in fault by site`

```
**P1-2 alerts in fault by site**

| # | Site | > 30 days | < 30 days | Total | Assigned | % Assigned | Link |
|---|------|-----------|-----------|-------|----------|------------|------|
| 1 | Charlestown Square         | 20 | 3 | 23 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=52&priority_ids=1&priority_ids=2&status_ids=fault&archived=false) |
| 2 | Rouse Hill Town Centre     | 11 | 3 | 14 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=154&priority_ids=1&priority_ids=2&status_ids=fault&archived=false) |
| 3 | Parkmore Shopping Centre   | 11 | 2 | 13 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=157&priority_ids=1&priority_ids=2&status_ids=fault&archived=false) |
| ... |
|   | **Total**                  | 69 | 16 | 85 | 0 | 0% |  |

_Ask me to drill into a site by equipment type, equipment name, or the full alert list._
```

Sort: `> 30 days` desc, then `Total` desc. Total row has no `#` and no `Link`.

`Link` cell is a markdown `[View](url)` pointing to the PEAK alert search filtered to that site:

```
https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=[site_id]&priority_ids=1&priority_ids=2&status_ids=fault&archived=false
```

- `[site_id]` = the site's numeric `site_id`.
- Priority pins: P1-2 → `priority_ids=1&priority_ids=2`; P3-5 → `priority_ids=3&priority_ids=4&priority_ids=5`.

---

## Table 2 — `[site_name]: P1-2 alerts in fault by equipment type`

```
**Parkmore Shopping Centre: P1-2 alerts in fault by equipment type**

| # | Equipment type | > 30 days | < 30 days | Total | Assigned | % Assigned | Link |
|---|----------------|-----------|-----------|-------|----------|------------|------|
| 1 | Exhaust Air Fans                | 4 | 0 | 4 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=157&priority_ids=1&priority_ids=2&status_ids=fault&metadata_type_ids=32&archived=false) |
| 2 | Packaged Air Conditioning Units | 3 | 0 | 3 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=157&priority_ids=1&priority_ids=2&status_ids=fault&metadata_type_ids=20&archived=false) |
| ... |
|   | **Total**                       | 11 | 2 | 13 | 0 | 0% |  |

_Ask me to drill into equipment names within a type, or the full alert list._
```

Title prefix is the site's `display_name` followed by `: `. Sort: `> 30 days` desc, then `Total` desc. Total row has no `#` and no `Link`.

`Link` cell is a markdown `[View](url)` pointing to the PEAK alert search filtered to that site, priority, status, **and** equipment type:

```
https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=[site_id]&<priority_pins>&<status_pins>&metadata_type_ids=[type_id]&archived=[include_archived]
```

- `[site_id]` = the table's site `site_id`.
- `[type_id]` = `metadata_type_ids` for `equipment_type_primary`, looked up in `references/equipment_type_ids.md`.
- Priority pins reflect the table's active priority set: P1-2 → `priority_ids=1&priority_ids=2`; P3-5 → `priority_ids=3&priority_ids=4&priority_ids=5`.
- Status / archived pins match the active filter set (same rule as Table 1's `site_alert_search_url`).
- If the type name doesn't resolve in `equipment_type_ids.md`, leave the `Link` cell empty for that row.

---

## Table 3 — `[site_name]: P1-2 alerts in fault by equipment name`

Two title variants depending on scope:

- Site only: `**[site_name]: P1-2 alerts in fault by equipment name**`
- Site + equipment type: `**[site_name] / [equipment_type_name]: P1-2 alerts in fault by equipment name**`

```
**Parkmore Shopping Centre: P1-2 alerts in fault by equipment name**

| # | Equipment name | > 30 days | < 30 days | Total | Assigned | % Assigned | Link |
|---|----------------|-----------|-----------|-------|----------|------------|------|
| 1 | PAC_New_R4    | 2 | 0 | 2 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=157&equipment_ids=313532612679&archived=false) |
| 2 | TEF-202       | 2 | 0 | 2 | 0 | 0% | [View](https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=157&equipment_ids=313532612676&archived=false) |
| ... |
|   | **Total**     | 11 | 2 | 13 | 0 | 0% |  |

_Ask me for the full alert list for this site._
```

Sort: `> 30 days` desc, then `Total` desc. Total row has no `#` and no `Link`.

`Link` cell is a markdown `[View](url)` pointing to the PEAK alert search for that equipment at this site (no priority/status pins — opens **all** non-archived alerts for the equipment, so the user can drill across priorities live):

```
https://ace.cimenviro.com/tickets/alerts/search?tickets_order_by=updated_at%20DESC&site_ids=[site_id]&equipment_ids=[id1]&equipment_ids=[id2]&archived=false
```

- `[site_id]` = the table's site `site_id`.
- `equipment_ids=…` repeats once per unique `AT.equipment_ids[0]` across the alert tickets grouped under this row (deduped). Most rows resolve to a single ID; multiple appear when the same display name maps to more than one physical equipment.

---

## Table 4 — `[site_name]: P1-2 alert list`

```
**Parkmore Shopping Centre: P1-2 alert list**

| # | Title | Equipment type | Priority | Impact | Assignee | Updated | Ticket link |
|---|-------|----------------|----------|--------|----------|---------|-------------|
| 1 | CH-101 - Inspect Unit Fail                       | Chiller                          | P1 Critical | Energy      | Alex Wong | 2 hours ago | [View](https://ace.cimenviro.com/tickets/alerts/157/<ticket-uuid>) |
| 2 | CH-302 - Inspect Unit Fail                       | Chiller                          | P1 Critical | Reliability | —         | 5 days ago  | [View](https://...) |
| ... |
```

- `Title` = `AT.title` verbatim.
- `Equipment type` = `AT.equipment_types[0]`.
- `Priority` = `AT.priority` verbatim — wire already returns `"P1 Critical"` / `"P2 Urgent"`.
- `Impact` = `AT.impact[0]` title-cased (e.g. `"energy"` → `"Energy"`); `"—"` if the array is empty or missing.
- `Assignee` = `AT.assignee.name` if non-null and non-empty; `"(unnamed)"` if `AT.assignee` exists but name is null/blank; `"—"` if `AT.assignee` itself is null.
- `Updated` = relative time between `AT.updated_at` and now (e.g. `"1 hour ago"`, `"5 days ago"`). See `field_mapping.md` for unit thresholds.
- `Ticket link` = `[View](AT.alert_link)`.
- Sort: `AT.updated_at` desc (newest first) — sort by the raw timestamp, not the rendered string. No Total row on the alert list.
