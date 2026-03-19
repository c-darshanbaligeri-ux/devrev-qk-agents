# DevRev Agents

AI-powered agents for building on DevRev's AgentOS platform. Plan, build, and test snap-ins and connectors without manual work.

## Available Commands

### Snap-in vertical
- `/devrev:plan-snapin` — Plan a snap-in or AirSync connector (PM agent)
- `/devrev:build-snapin` — Build complete deployable snap-in code (Architect agent)
- `/devrev:test-snapin` — Test with unit tests + UI automation (Tester agent)

### Implementation vertical
- `/devrev:plan-implementation` — Plan dashboards, widgets, analytics (PM agent)
- `/devrev:build-implementation` — Generate widget JSON + dashboard layout (Architect agent)
- `/devrev:test-implementation` — JSON validation + UI verification (Tester agent)

### Cross-cutting
- `/devrev:improve-skill` — Report mistakes, update agent skills (Self-learning agent)

## Agent Pipeline

```
User request → PM (plan) → Architect (build) → Tester (verify)
                  ↑                                    |
                  └──── bugs back to architect/PM ─────┘
```

Each agent hands off to the next with a structured brief. Bugs flow back upstream.

## Key DevRev Technical Facts

- **Two snap-in types**: Simple (automations, commands, webhooks) and AirSync (data sync connectors)
- **Simple snap-ins** use `@devrev/typescript-sdk`
- **AirSync snap-ins** use `@devrev/ts-adaas` SDK (+ typescript-sdk for DevRev API calls)
- **Timeout**: `processTask({ task, onTimeout })` — SDK handles 10-min soft / 13-min hard timeout
- **Sync model**: DevRev Airdrop platform triggers periodic sync and provides `lastSuccessfulSyncStarted` — snap-ins do NOT register webhooks
- **Batching**: `adapter.initializeRepos(repos)` + `adapter.getRepo("type").push(items)` — SDK batches automatically
- **Mapping**: `chef-cli configure-mappings` generates `initial_domain_mapping.json`
- **Metadata**: `chef-cli validate-metadata` validates `external_domain_metadata.json`
- **Dashboard creation**: Widget JSON → widget-preview → widget ID → dashboard-preview → dashboard ID
- **Widget JSON**: data_sources (oasis) → dimensions + measures → sub_widgets → visualization
- **SQL rules**: No SELECT *, no table aliases in sql_expression, list every column, fully qualify all references
- **Visualization types**: metric, line, column, bar, table, donut, pie, packed_bubble, heatmap
- **Dashboard URL**: `https://app.devrev.ai/<slug>/dashboard?dashboardId=<ID>`

## Real Examples

PRD and TDD examples from actual DevRev projects are in `/examples/`:
- Slack ADaaS TDD — channel import connector
- Monday.com TDD — board/item import connector
- Planhat PRD — bidirectional customer success sync
- Snowflake PRD — one-way table import
