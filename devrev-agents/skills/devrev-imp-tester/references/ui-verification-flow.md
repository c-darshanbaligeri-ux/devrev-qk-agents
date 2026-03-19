# UI Verification Flow

Step-by-step browser automation flow for testing DevRev dashboards. Execute each phase in order.

---

## Phase A: Test SQL in Notebook

```
1. Navigate to DevRev → Explore → Notebook
2. Select the appropriate dataset
3. For each widget's base sql_query:
   a. Paste the sql_query into Notebook
   b. Click "Run"
   c. Verify:
      - Query executes without error
      - Returns data (not empty result set)
      - Row count is reasonable for the data
   d. Spot-check 3-5 values against known data
   e. Record: "SQL for Widget [X] returns [N] rows, sample values correct"
4. If query fails:
   - Record exact error message
   - Check SQL rules (SELECT *, unqualified columns, missing table prefix)
   - Report as Critical bug — do NOT proceed to widget deployment
```

## Phase B: Deploy Widgets via widget-preview

```
1. Navigate to: https://app.devrev.ai/<slug>/widget-preview
2. For each widget JSON file:
   a. Clear the editor (select all → delete)
   b. Paste the complete widget JSON
   c. Click "Create widget"
   d. Check for errors:
      - If RED error banner: screenshot + record exact error text → Critical bug
      - If JSON parse error: check for trailing commas, missing quotes
      - If "Referenced table not found": check sql_expression for aliases
   e. If success:
      - Copy the widget ID from the response
      - Record: Widget [Name] → ID: [widget_id]
   f. Verify the widget preview renders:
      - Chart type matches expected? (line shows as line, not bar)
      - Data appears? (not empty/blank)
      - Labels correct? (axis labels, legend)
      - Colors correct? (if specified)
      - Numbers in reasonable range?
3. Record all widget IDs for dashboard assembly
```

## Phase C: Build Dashboard via dashboard-preview

```
1. Navigate to: https://app.devrev.ai/<slug>/dashboard-preview
2. Update the dashboard layout JSON with real widget IDs from Phase B
3. Paste the complete dashboard layout JSON
4. Click "Create dashboard"
5. Verify:
   a. All widgets visible? (count matches expected)
   b. Layout correct? (widths, heights, positions match spec)
   c. No overlapping widgets?
   d. No large blank gaps?
6. Copy the dashboard ID
7. Navigate to: https://app.devrev.ai/<slug>/dashboard?dashboardId=<ID>
8. Verify the dashboard loads from the URL
9. Record: Dashboard ID: [dashboard_id]
```

## Phase D: Verify Metrics (Data Accuracy)

```
For each metric/measure displayed on the dashboard:
1. Note the exact value shown in the widget
2. Run the equivalent SQL in Notebook:
   - For COUNT metrics: run the COUNT query directly
   - For AVG/MEDIAN: run the aggregation query
   - For percentages: compute numerator and denominator separately
   - For time durations: run DATEDIFF query
3. Compare values:
   - Widget value == Notebook value → PASS
   - Mismatch → FAIL → record both values
4. Spot-check at least 3 data points per widget
5. For table widgets: verify 5 random rows match Notebook output
6. Record all comparisons in the test results table
```

## Phase E: Test Filters

```
1. Date filter test:
   a. Apply "Last 7 days" filter
   b. Verify ALL widgets update (not just some)
   c. Check: do numbers decrease vs. "Last 30 days"?
   d. Apply "Last 90 days" — numbers should increase
   e. Clear filter → verify original values return

2. Category filter test:
   a. Apply a category filter (e.g., Priority = "P0")
   b. Verify all widgets show filtered data
   c. Check: metric cards show reduced numbers
   d. Check: charts show only filtered category
   e. Clear filter → verify original values return

3. Edge case tests:
   a. Filter that returns 0 results → widgets show 0/empty, NOT error
   b. Multiple filters combined → data intersects correctly
   c. Filter → unfilter → values match pre-filter state exactly
```

## Phase F: Test Drill-Throughs (if configured)

```
1. Identify widgets with drill_throughs configured
2. For each drill-through:
   a. Click on the chart element (bar, line point, etc.)
   b. Verify navigation to the target dashboard
   c. Verify the target dashboard is filtered by the clicked element
   d. Navigate back → verify original dashboard state preserved
   e. Record: Source [widget] → Target [dashboard] → Filtered correctly: YES/NO
```

## Phase G: Regression Testing (for fixes only)

```
For each fix applied:
1. Verify the FIXED metric now shows the EXPECTED value
2. Run Notebook query to confirm the number matches
3. Check that OTHER metrics on the SAME widget are UNCHANGED
4. Check that OTHER widgets on the dashboard are UNCHANGED
5. If fix involved SQL changes: re-run ALL SQL queries in Notebook
6. Record:
   - Fix [N]: Correct value displayed: YES/NO
   - Other metrics unchanged: YES/NO
   - Other widgets unchanged: YES/NO
```

---

## Test Results Template

```markdown
# Dashboard Test Results: [Dashboard Name]

## Environment
- DevRev org: [slug]
- Dashboard ID: [ID]
- Test date: [date]

## UI Deployment
| Widget | Widget ID | Deployed | Renders | Notes |
|--------|-----------|----------|---------|-------|

## Data Accuracy
| Widget | Metric | Widget Value | Notebook Value | Match |
|--------|--------|-------------|----------------|-------|

## Filter Testing
| Filter | Applied | All Widgets Updated | Data Correct |
|--------|---------|-------------------|-------------|

## Bugs Found
| # | Severity | Widget | Description | Evidence |
|---|----------|--------|-------------|----------|

## Recommendation
- [ ] Ship
- [ ] Ship with known issues
- [ ] Block — return to architect
- [ ] Block — return to PM
```
