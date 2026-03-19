# DevRev System Tables Reference

Common system tables available for dashboard queries. Use `Explore > Notebook` to verify table availability and column names in your org.

---

## Ticket & Issue Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.dim_ticket` | Ticket dimension table | id, title, state, priority, severity, owned_by_id, created_date, modified_date, actual_close_date, applies_to_part_id |
| `system.support_insights_ticket_metrics_summary` | Pre-aggregated ticket metrics | id, record_date, severity_name, state, owned_by_id, created_date |
| `system.dim_work` | All work items (issues, tickets, tasks) | id, type, title, state, priority, owned_by_id, created_date |

## Conversation Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.dim_conversation` | Conversations | id, subject, state, created_date, owned_by_id |
| `system.dim_timeline_entry` | Timeline entries (messages, notes) | id, type, created_by_id, created_date, parent_id |

## User & Account Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.dim_dev_user` | Dev users (internal) | id, display_name, email, state |
| `system.dim_rev_user` | Rev users (customers) | id, display_name, email, rev_org_id |
| `system.dim_account` | Accounts/organizations | id, display_name, created_date |
| `system.dim_rev_org` | Revenue organizations | id, display_name |

## Sales & Revenue Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.dim_opportunity` | Sales opportunities | id, name, state, stage, amount, probability, close_date, owned_by_id |

## Part & Product Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.dim_part` | Parts (products, features, components) | id, name, type, owned_by_id |

## SLA Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `system.support_insights_sla_summary` | SLA metrics | id, sla_stage, policy_name, breach_time |

---

## Common Joins

```sql
-- Tickets with owner details
SELECT t.id, t.title, u.display_name as owner_name
FROM system.dim_ticket t
LEFT OUTER JOIN system.dim_dev_user u ON t.owned_by_id = u.id

-- Tickets with part details
SELECT t.id, t.title, p.name as part_name
FROM system.dim_ticket t
LEFT OUTER JOIN system.dim_part p ON t.applies_to_part_id = p.id

-- Ticket metrics with ticket details
SELECT m.record_date, m.severity_name, t.title, t.owned_by_id
FROM system.support_insights_ticket_metrics_summary m
LEFT OUTER JOIN system.dim_ticket t ON m.id = t.id
```

## Notes
- Table names and columns may vary by org configuration
- Always verify in Notebook before building widgets
- Custom fields appear as additional columns on base tables
- Use `SELECT * FROM system.<table> LIMIT 10` in Notebook to explore available columns
