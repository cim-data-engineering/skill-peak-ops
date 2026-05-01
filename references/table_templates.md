# Table templates

Render exactly these markdown shapes. Replace `[bracketed]` placeholders with computed values. Keep column order, header text, and capitalization identical to what's shown — the user is comparing against a fixed XLSX layout.

---

## Table 1 — `P1-2 alerts in fault by site`

```
**P1-2 alerts in fault by site**

| # | Site | > 30 days | < 30 days | Total | Assigned | % Assigned |
|---|------|-----------|-----------|-------|----------|------------|
| 1 | Charlestown Square         | 20 | 3 | 23 | 0 | 0% |
| 2 | Rouse Hill Town Centre     | 11 | 3 | 14 | 0 | 0% |
| 3 | Parkmore Shopping Centre   | 11 | 2 | 13 | 0 | 0% |
| ... |
|   | **Total**                  | 69 | 16 | 85 | 0 | 0% |

_Ask me to drill into a site by equipment type, equipment name, or the full alert list._
```

Sort: `> 30 days` desc, then `Total` desc. Total row has no `#`.

---

## Table 2 — `[site_name]: P1-2 alerts in fault by equipment type`

```
**Parkmore Shopping Centre: P1-2 alerts in fault by equipment type**

| # | Equipment type | > 30 days | < 30 days | Total | Assigned | % Assigned |
|---|----------------|-----------|-----------|-------|----------|------------|
| 1 | Exhaust Air Fans                | 4 | 0 | 4 | 0 | 0% |
| 2 | Packaged Air Conditioning Units | 3 | 0 | 3 | 0 | 0% |
| ... |
|   | **Total**                       | 11 | 2 | 13 | 0 | 0% |

_Ask me to drill into equipment names within a type, or the full alert list._
```

Title prefix is the site's `display_name` followed by `: `. Sort: `> 30 days` desc, then `Total` desc.

---

## Table 3 — `[site_name]: P1-2 alerts in fault by equipment name`

Two title variants depending on scope:

- Site only: `**[site_name]: P1-2 alerts in fault by equipment name**`
- Site + equipment type: `**[site_name] / [equipment_type_name]: P1-2 alerts in fault by equipment name**`

```
**Parkmore Shopping Centre: P1-2 alerts in fault by equipment name**

| # | Equipment name | > 30 days | < 30 days | Total | Assigned | % Assigned |
|---|----------------|-----------|-----------|-------|----------|------------|
| 1 | PAC_New_R4    | 2 | 0 | 2 | 0 | 0% |
| 2 | TEF-202       | 2 | 0 | 2 | 0 | 0% |
| ... |
|   | **Total**     | 11 | 2 | 13 | 0 | 0% |

_Ask me for the full alert list for this site._
```

Sort: `> 30 days` desc, then `Total` desc.

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
- `Priority` = `"P1 Critical"` for `priority=1`, `"P2 Urgent"` for `priority=2`.
- `Impact` = `AT.impact` verbatim if it already contains an uppercase letter, else title-cased (e.g. `"energy"` → `"Energy"`); `"—"` if null.
- `Assignee` = `AT.assignee.name` if non-null and non-empty; `"(unnamed)"` if `AT.assignee` exists but name is null/blank; `"—"` if `AT.assignee` itself is null.
- `Updated` = relative time between `AT.updated_at` and now (e.g. `"1 hour ago"`, `"5 days ago"`). See `field_mapping.md` for unit thresholds.
- `Ticket link` = `[View](AT.alert_link)`.
- Sort: `AT.updated_at` desc (newest first) — sort by the raw timestamp, not the rendered string. No Total row on the alert list.
