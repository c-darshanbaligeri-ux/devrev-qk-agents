# JSON Validation Checklist

Run through every check for each widget JSON before deployment. This catches 90% of errors before they hit the UI.

---

## A. Structure Checks

- [ ] Top-level has `data_sources` (array), `sub_widgets` (array), and `title` (string)
- [ ] Each data_source has: `type: "oasis"`, `reference_name`, `oasis`, `dimensions`, `measures`
- [ ] `oasis` has: `datasets` (array of table names), `sql_query` (string)
- [ ] Each sub_widget has: `query`, `visualization`
- [ ] `query` has: `dimensions` (array), `measures` (array)
- [ ] Visualization `type` matches the child key (e.g., `type: "line"` → has `"line": {}`)

## B. SQL Checks

- [ ] `sql_query` lists every column individually — NO `SELECT *`, NO `table.*`
- [ ] Every column in SELECT has a table prefix (e.g., `t.id`, not just `id`)
- [ ] If JOINs: all ambiguous columns are aliased in SELECT
- [ ] All tables used in the query are listed in `datasets` array
- [ ] No syntax errors — test in Notebook first

## C. Reference Naming Checks

- [ ] Every `reference_name` in dimensions/measures is unique within its data source
- [ ] Every reference in `query.dimensions` is fully qualified: `source_name.dim_name`
- [ ] Every reference in `query.measures` is fully qualified: `source_name.measure_name`
- [ ] Every reference in `query.order_by` is fully qualified: `source_name.field_name`
- [ ] Every reference in `visualization` x/y/z is fully qualified: `source_name.field_name`
- [ ] References used in sub_widgets actually exist in data_sources dimensions/measures
- [ ] No typos in reference names (exact match required)

## D. Visualization-Specific Checks

### Line Chart
- [ ] Uses `"line": {}` — NOT `"settings": {}`
- [ ] Has `order_by` ascending on time dimension
- [ ] x array has time dimension first

### Column/Bar Chart
- [ ] `is_stacked` is explicitly set (true or false)
- [ ] Second x item (if any) is the categorical split dimension

### Donut/Pie
- [ ] `y` is a single object — NOT an array
- [ ] x is an array with one dimension

### Heatmap
- [ ] `x`, `y`, `z` are all single objects — NOT arrays
- [ ] Exactly 2 dimensions + 1 measure in the query
- [ ] x is category/time, y is category, z is measure

### Metric
- [ ] No ID-type fields in y (use display_name or text instead)
- [ ] PVP: `pvp.enabled: true` present if period comparison needed

### Table
- [ ] All `columns` have `label` and `reference_name`
- [ ] Column `reference_name` values match data_source dimensions/measures
- [ ] `order` values are sequential (1, 2, 3...)

## E. Dimension/Measure Checks

- [ ] Each dimension has both `devrev_schema` and `meerkat_schema`
- [ ] `devrev_schema.field_type` is valid: enum, id, timestamp, int, double, text, bool, tokens
- [ ] Filterable dimensions have `"is_filterable": true` in devrev_schema
- [ ] Each measure has `sql_expression` in meerkat_schema
- [ ] Division operations use `NULLIF` to prevent divide-by-zero
- [ ] meerkat_schema `type` matches: timestamps → `"time"`, numbers → `"number"`, text → `"string"`
- [ ] ID dimensions have `id_type` array (e.g., `["work"]`, `["dev_user"]`)

## F. Cross-Widget Consistency

- [ ] All widgets sharing a filter use the same dimension name and reference_name
- [ ] Dashboard-level date filter dimension exists in every widget's data_source
- [ ] Consistent naming conventions across all widgets

## G. Color Checks

- [ ] Colors use valid catalog values (e.g., `chart-category-1-base`, `chart-alert-base`)
- [ ] `key_lookup` color entries match actual dimension values
- [ ] No hex codes or RGB values (use catalog names only)

---

## Validation Report Format

```markdown
## JSON Validation Report

### Widget: [Name]
| Check | Status | Details |
|-------|--------|---------|
| Structure | PASS/FAIL | [issue if any] |
| SQL | PASS/FAIL | |
| References | PASS/FAIL | |
| Visualization | PASS/FAIL | |
| Dimensions/Measures | PASS/FAIL | |

### Issues Found
| # | Severity | Widget | Description | Fix |
|---|----------|--------|-------------|-----|
| 1 | Critical | [name] | [description] | [exact fix] |
```
