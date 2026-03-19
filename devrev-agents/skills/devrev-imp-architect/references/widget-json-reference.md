# Widget JSON Reference

Complete reference for building DevRev widget configurations. This merges the widget building guide, dashboard cheat sheet, and visualization types catalog.

---

## Top-Level Structure

Every widget JSON must follow this structure:

```json
{
  "data_sources": [
    {
      "type": "oasis",
      "reference_name": "your_source_name",
      "oasis": {
        "datasets": ["system.table_name"],
        "sql_query": "SELECT table.col1, table.col2 FROM system.table_name"
      },
      "dimensions": [],
      "measures": []
    }
  ],
  "sub_widgets": [
    {
      "query": {
        "dimensions": ["your_source_name.dimension_name"],
        "measures": ["your_source_name.measure_name"],
        "order_by": []
      },
      "visualization": {}
    }
  ],
  "title": "Widget Title"
}
```

---

## Data Sources

### Oasis Configuration

```json
{
  "type": "oasis",
  "reference_name": "ticket_metrics",
  "oasis": {
    "datasets": [
      "system.support_insights_ticket_metrics_summary",
      "system.dim_ticket"
    ],
    "sql_query": "SELECT t.id, t.title, t.state, t.priority, t.created_date, t.owned_by_id, m.record_date, m.severity_name FROM system.support_insights_ticket_metrics_summary m LEFT OUTER JOIN system.dim_ticket t ON m.id = t.id"
  }
}
```

Key rules:
- `datasets`: list ALL tables used in the query
- `sql_query`: the base SQL — MUST list every column individually (see SQL Rules)
- `reference_name`: unique ID used to prefix all dimension/measure references

---

## Dimensions

Dimensions define filterable/groupable fields. Each dimension has `devrev_schema` (field config) + `meerkat_schema` (SQL config) + `reference_name`.

### Enum dimension
```json
{
  "devrev_schema": {
    "allowed_values": ["High", "Medium", "Low", "Blocker"],
    "field_type": "enum",
    "is_filterable": true,
    "name": "severity_name",
    "ui": { "display_name": "Severity" }
  },
  "meerkat_schema": {
    "sql_expression": "severity_name",
    "type": "string"
  },
  "reference_name": "severity_name"
}
```

### ID dimension (references a DevRev object)
```json
{
  "devrev_schema": {
    "field_type": "id",
    "id_type": ["work"],
    "is_filterable": true,
    "name": "id",
    "ui": { "display_name": "Id" }
  },
  "meerkat_schema": {
    "sql_expression": "id",
    "type": "string"
  },
  "reference_name": "id"
}
```

Valid id_types: `work`, `dev_user`, `rev_user`, `account`, `rev_org`, `part`, `tag`, `conversation`, `opportunity`, `custom_object.*`

### Timestamp dimension
```json
{
  "devrev_schema": {
    "field_type": "timestamp",
    "is_filterable": true,
    "name": "created_date",
    "ui": { "display_name": "Created Date" }
  },
  "meerkat_schema": {
    "sql_expression": "created_date",
    "type": "time"
  },
  "reference_name": "created_date"
}
```

### Text/String dimension
```json
{
  "devrev_schema": {
    "field_type": "text",
    "is_filterable": true,
    "name": "severity_name",
    "ui": { "display_name": "Severity" }
  },
  "meerkat_schema": {
    "sql_expression": "severity_name",
    "type": "string"
  },
  "reference_name": "severity_name"
}
```

### Bool dimension
```json
{
  "devrev_schema": {
    "field_type": "bool",
    "is_filterable": true,
    "name": "is_active",
    "ui": { "display_name": "Active" }
  },
  "meerkat_schema": {
    "sql_expression": "is_active",
    "type": "boolean"
  },
  "reference_name": "is_active"
}
```

---

## Measures

Measures define aggregations. The `sql_expression` in meerkat_schema contains the aggregation function.

### Count (int)
```json
{
  "devrev_schema": {
    "field_type": "int",
    "is_filterable": true,
    "name": "unique_ids_count",
    "ui": { "display_name": "Tickets" }
  },
  "meerkat_schema": {
    "sql_expression": "COUNT(DISTINCT id)",
    "type": "number"
  },
  "reference_name": "unique_ids_count"
}
```

### Sum/Average (double)
```json
{
  "devrev_schema": {
    "field_type": "double",
    "is_filterable": false,
    "name": "median_age",
    "ui": { "display_name": "Median Age" }
  },
  "meerkat_schema": {
    "sql_expression": "MEDIAN(age)",
    "type": "number"
  },
  "reference_name": "median_age"
}
```

### Percentage
```json
{
  "devrev_schema": {
    "field_type": "double",
    "name": "blocker_pct",
    "ui": {
      "display_name": "Blocker %",
      "unit": "percentage"
    }
  },
  "meerkat_schema": {
    "sql_expression": "100.0 * (COUNT(DISTINCT CASE WHEN severity_name = 'Blocker' AND state != 'closed' THEN id END) / NULLIF(COUNT(DISTINCT CASE WHEN state != 'closed' THEN id END), 0))",
    "type": "number"
  },
  "reference_name": "blocker_pct"
}
```

### Currency
```json
{
  "devrev_schema": {
    "field_type": "double",
    "name": "pipeline_amount",
    "ui": {
      "display_name": "Pipeline Amount",
      "is_currency_field": true,
      "unit": "USD"
    }
  },
  "meerkat_schema": {
    "sql_expression": "SUM(CASE WHEN state = 'open' OR state = 'in_progress' THEN (amount*fprobability/100) ELSE 0 END)",
    "type": "number"
  },
  "reference_name": "pipeline_amount"
}
```

### Time Duration
```json
{
  "devrev_schema": {
    "field_type": "double",
    "name": "median_resolution_time",
    "ui": {
      "display_name": "Median Resolution Time",
      "unit": "minutes"
    }
  },
  "meerkat_schema": {
    "sql_expression": "MEDIAN(DATEDIFF('minute', created_date, actual_close_date))",
    "type": "number"
  },
  "reference_name": "median_resolution_time"
}
```
Unit options: `milliseconds`, `seconds`, `minutes`, `hours`, `days`

### Compact Int (100K, 1M display)
```json
{
  "devrev_schema": {
    "field_type": "int",
    "name": "total_count",
    "ui": {
      "display_name": "Total",
      "is_compact": true
    }
  },
  "meerkat_schema": {
    "sql_expression": "COUNT(DISTINCT id)",
    "type": "number"
  },
  "reference_name": "total_count"
}
```

### Double with Precision
```json
{
  "devrev_schema": {
    "field_type": "double",
    "name": "avg_score",
    "ui": {
      "display_name": "Avg Score",
      "precision": 2
    }
  },
  "meerkat_schema": {
    "sql_expression": "AVG(score)",
    "type": "number"
  },
  "reference_name": "avg_score"
}
```

---

## Supported Field Types

| Field Type | Use For | meerkat type |
|-----------|---------|-------------|
| `int` | Counts, whole numbers | `number` |
| `double` | Averages, decimals, percentages | `number` |
| `decimal` | Precise decimal numbers | `number` |
| `currency` | Monetary values | `number` |
| `bool` | True/false | `boolean` |
| `text` | Basic strings | `string` |
| `string` | Basic strings | `string` |
| `tokens` | Tokenized text for search | `string` |
| `timestamp` | Date/time | `time` |
| `id` | DevRev object references | `string` |
| `enum` | Enumerated values (with allowed_values) | `string` |
| `overridable_enum` | Overridable enums | `string` |
| `uenum` | User-defined enums | `string` |

---

## UI Formatting Options

| Option | Where | Effect |
|--------|-------|--------|
| `"unit": "percentage"` | measure ui | Shows % symbol |
| `"unit": "USD"` | measure ui | Shows $ with currency |
| `"is_currency_field": true` | measure ui | Enables currency formatting |
| `"unit": "minutes"` | measure ui | Formats as time duration |
| `"is_compact": true` | measure ui | Shows 100K, 1M, etc. |
| `"precision": 2` | measure ui | Decimal places |
| `"is_filterable": true` | dimension devrev_schema | Enables filtering |
| `"is_groupable": true` | dimension ui | Enables table group-by |
| `"order": 1` | dimension/measure ui | Column order in tables |
| `"is_hidden": true` | visualization column | Hides column by default |
| `"max_width": 200` | visualization column | Max column width |
| `"min_width": 100` | visualization column | Min column width |

---

## Query Generation Layers

Understanding how the system generates SQL is critical for debugging.

The system wraps your base query in layers:

**Layer 1 — Your base query:**
```sql
SELECT t.id, t.title, t.state FROM system.dim_ticket t
```

**Layer 2 — System adds measures:**
```sql
SELECT COUNT(DISTINCT id) as ticket_metrics_unique_ids_count
FROM (
  SELECT t.id, t.title, t.state FROM system.dim_ticket t
) AS ticket_metrics
```

**Layer 3 — System adds filters + grouping:**
```sql
SELECT * FROM (
  -- Layer 2 query
) AS filtered_query
WHERE created_date >= '2024-01-01'
GROUP BY state
```

**Key implications:**
- Table aliases from Layer 1 are NOT available in Layer 2+ (the outer queries)
- Column names in sql_expression must be raw names, not aliased
- SELECT * in Layer 1 breaks because Layer 2 can't resolve columns
