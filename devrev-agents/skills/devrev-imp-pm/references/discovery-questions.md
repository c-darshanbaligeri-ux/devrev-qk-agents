# Discovery Questions for Dashboard Planning

Organized question bank for gathering dashboard requirements. Ask in focused rounds (3-5 questions each).

---

## Round 1 — Purpose & Audience

1. Who will use this dashboard? (support leads, PMs, executives, agents, customer success)
2. What decisions will they make from it? (staffing, prioritization, escalation, forecasting)
3. How often will they check it? (daily standup, weekly review, real-time monitoring, monthly reporting)
4. Is this a new dashboard or updating/fixing an existing one?
5. Any existing dashboards or reports this replaces?

## Round 2 — Data & Metrics

1. What are the top 3-5 KPIs to show?
2. What data source tables hold this data? (If unknown, describe the objects involved — tickets, conversations, accounts, opportunities, etc.)
3. What time granularity? (daily, weekly, monthly, quarterly)
4. Any existing Notebook SQL queries or reports the user already has?
5. What time range should be the default? (last 7 days, last 30 days, last quarter)

## Round 3 — Filters & Breakdowns

1. What filters should be available? (date range, owner, priority, team, account, status, severity)
2. Any group-by breakdowns? (by status, by priority, by owner, by time period, by team)
3. Any drill-through needed? (click a bar → see detail dashboard)
4. Should filters apply to all widgets or only specific ones?
5. Any cross-filter behavior? (clicking one chart filters another)

## Round 4 — Visualization Preferences

1. Metric cards at the top? (for KPI numbers with period-over-period comparison)
2. Trends over time? (line charts)
3. Breakdowns by category? (bar/column/pie/donut)
4. Detail table at the bottom? (sortable, filterable)
5. Any specific chart types requested? (heatmap, packed bubble, stacked bar)

## Round 5 — Layout & UX

1. How many widgets total? (guideline: 6-12 per dashboard)
2. Any tab organization? (e.g., Overview tab + Detail tab)
3. Dashboard title and description?
4. Any branding or color preferences?
5. Target device? (desktop-only or responsive)

---

## Follow-up Questions by Domain

### Support Dashboards
- Which ticket states to include? (open, in-progress, waiting, resolved, closed)
- SLA tracking needed? (breach rates, response time, resolution time)
- Agent performance comparison?
- Customer satisfaction (CSAT) scores available?

### Sales/Revenue Dashboards
- Which pipeline stages to track?
- Forecast categories? (commit, best-case, pipeline)
- Win rate calculation method? (by count or by amount)
- Currency handling? (single currency or multi-currency)

### Agent Productivity Dashboards
- What counts as "productivity"? (tickets resolved, messages sent, response time)
- How to distinguish agent actions from system actions?
- Time-based vs volume-based metrics?
- Team-level vs individual-level breakdowns?

### Executive Dashboards
- Period-over-period comparisons needed? (MoM, QoQ, YoY)
- Target/goal lines on charts?
- Top-N analysis? (top accounts, top issues)
- Anomaly/threshold highlighting?
