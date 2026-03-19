# Dashboard Spec Template

Use this template to document dashboard requirements after discovery. The architect builds directly from this spec.

---

## Dashboard: [Name]

### Overview
- **Purpose**: [What decisions does this dashboard support?]
- **Audience**: [Who uses it and how often?]
- **Data sources**: [Which system tables are involved?]
- **Default date range**: [Last 7 days / Last 30 days / etc.]

### Global Filters
| Filter | Type | Source Field | Default |
|--------|------|-------------|---------|
| Date Range | timestamp | created_date | Last 30 days |
| Owner | id (dev_user) | owned_by_id | All |
| Priority | enum | priority | All |

---

### Widget Layout

```
Row 1: [Metric] [Metric] [Metric] [Metric]     (4 × width:3, height:2)
Row 2: [Line Chart]        [Column Chart]        (2 × width:6, height:4)
Row 3: [Detail Table - full width]               (1 × width:12, height:6)
```

---

### Widget Specifications

#### Widget 1: [Name]
- **Purpose**: What question does this answer?
- **Type**: metric | line | column | bar | table | pie | donut | heatmap | packed_bubble
- **Data source**: system.[table_name] (+ joins if needed)
- **SQL sketch**: `SELECT ... FROM system.table WHERE ...`
- **Dimensions**: [field names with types — what the user filters/groups by]
- **Measures**: [aggregation description — COUNT of X, AVG of Y, SUM of Z]
- **Filters**: [which dimensions are filterable from the dashboard]
- **Notes**: [any special logic — conditional counts, percentage calculations, PVP comparison, etc.]

#### Widget 2: [Name]
- **Purpose**: ...
- **Type**: ...
- **Data source**: ...
- **SQL sketch**: ...
- **Dimensions**: ...
- **Measures**: ...
- **Filters**: ...
- **Notes**: ...

[Repeat for each widget]

---

### Handoff to Architect

**Summary**: [N] widgets across [M] rows. Data from [tables]. Global filters: [list].

**Priority order for implementation**:
1. [Most critical widget first]
2. [Second priority]
3. ...

**Known limitations / deferred items**:
- [Anything explicitly out of scope for V1]

**Open questions**:
- [Anything that needs stakeholder clarification]
