# SQL Rules for DevRev Widgets

These rules are MANDATORY. Violating any one of them will break your widget.

---

## Rule 1: List EVERY column individually

```sql
-- CORRECT:
SELECT
    table1.column1,
    table1.column2,
    table2.column3
FROM system.table1
JOIN system.table2 ON table1.id = table2.id

-- BREAKS:
SELECT *
SELECT table.*
SELECT column1, column2    -- missing table prefix
SELECT table1.*, table2.column3
```

Why: The system wraps your query in a subquery. If you use `*`, the outer query can't resolve individual column names for aggregation.

## Rule 2: Fully qualify ALL references

```json
// WRONG:
"dimensions": ["some_dimension"]
"measures": ["some_measure"]

// CORRECT:
"dimensions": ["your_source_name.some_dimension"]
"measures": ["your_source_name.some_measure"]
```

This applies everywhere: query dimensions, query measures, order_by reference_name, visualization x/y reference_name.

## Rule 3: No table aliases in sql_expression

```json
// WRONG — alias "t2" won't be available in the outer query:
{
  "sql_query": "SELECT * FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id",
  "meerkat_schema": {
    "sql_expression": "t2.column_name"   // BREAKS
  }
}

// CORRECT — use raw column name:
{
  "sql_query": "SELECT t1.col1, t2.column_name FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id",
  "meerkat_schema": {
    "sql_expression": "column_name"      // Works
  }
}
```

Why: The system generates outer queries that reference your base query as a subquery. Aliases inside the subquery are not visible outside.

## Rule 4: order_by is required for time series

Any visualization that has a time dimension on the x-axis MUST include order_by:

```json
"query": {
  "dimensions": ["source.created_date"],
  "measures": ["source.count"],
  "order_by": [{
    "direction": "ascending",
    "reference_name": "source.created_date"
  }]
}
```

Without this, data points appear in random order and the line chart is meaningless.

## Rule 5: Column names must be unique when joining

If two tables have the same column name (e.g., both have `id`), alias them in the base query:

```sql
SELECT
    t.id AS ticket_id,
    c.id AS conversation_id,
    t.title,
    c.subject
FROM system.dim_ticket t
JOIN system.dim_conversation c ON t.conversation_id = c.id
```

Then use `ticket_id` and `conversation_id` in your dimensions.

## Rule 6: Use NULLIF to prevent division by zero

```sql
-- WRONG:
"sql_expression": "SUM(resolved) / SUM(total)"

-- CORRECT:
"sql_expression": "SUM(resolved) / NULLIF(SUM(total), 0)"
```

## Rule 7: CASE WHEN for conditional aggregations

```sql
-- Count only open tickets:
"sql_expression": "COUNT(DISTINCT CASE WHEN state != 'closed' THEN id END)"

-- Sum only high-priority amounts:
"sql_expression": "SUM(CASE WHEN priority = 'p0' OR priority = 'p1' THEN amount ELSE 0 END)"

-- Percentage of blocker tickets:
"sql_expression": "100.0 * (COUNT(DISTINCT CASE WHEN severity_name = 'Blocker' THEN id END) / NULLIF(COUNT(DISTINCT id), 0))"
```

## Rule 8: Timestamp functions

```sql
-- Date diff in minutes:
"DATEDIFF('minute', created_date, actual_close_date)"

-- Date diff in hours:
"DATEDIFF('hour', created_date, modified_date)"

-- Median resolution time:
"MEDIAN(DATEDIFF('minute', created_date, actual_close_date))"
```

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Parser Error: syntax error at or near ","` | SELECT * or unqualified columns | List every column individually |
| `Referenced table "X" not found!` | Table alias used in sql_expression | Remove alias, use raw column name |
| `Cannot find reference 'field_name'` | Unqualified reference in query/viz | Add `source_name.` prefix |
| Widget shows 0 or null | NULLIF missing, division by zero | Add NULLIF wrapper |
| Multiple rows for single date range | No aggregation on date | Add GROUP BY or check date dimension |
| Filters have no effect | `is_filterable` not set | Add `"is_filterable": true` to dimension |

## Pre-Flight Checklist

Before pasting JSON into widget-preview:
- [ ] Every column in SELECT is individually listed (no *)
- [ ] Every sql_expression uses raw column names (no aliases)
- [ ] Every reference in query/visualization is `source_name.field_name`
- [ ] Time series have order_by ascending
- [ ] Division operations use NULLIF
- [ ] Base query tested in Notebook first
