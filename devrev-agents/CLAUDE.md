# DevRev Agents

AI-powered agents for building on DevRev's AgentOS platform. Plan, build, and test snap-ins and connectors without manual work.

## Available Commands

### Snap-in vertical
- `/devrev:plan-snapin` — Plan a snap-in or AirSync connector (PM agent)
- `/devrev:build-snapin` — Build complete deployable snap-in code (Architect agent)
- `/devrev:test-snapin` — Test with unit tests + UI automation (Tester agent)
- `/devrev:update-snapin` — Update an existing snap-in (add entity, fix pagination, switch auth)
- `/devrev:generate-metadata` — Generate metadata + mapping JSON with validation
- `/devrev:search-guide` — Quick reference lookup for AirSync patterns

### Implementation vertical
- `/devrev:plan-implementation` — Plan dashboards, widgets, analytics (PM agent)
- `/devrev:build-implementation` — Generate widget JSON + dashboard layout (Architect agent)
- `/devrev:test-implementation` — JSON validation + UI verification (Tester agent)

### Cross-cutting
- `/devrev:improve-skill` — Report mistakes, update agent skills (Self-learning agent)
- `/devrev:update` — Update plugin to the latest version from GitHub

## Agent Pipeline

```
User request → PM (plan) → Architect (build) → Tester (verify)
                  ↑                                    |
                  └──── bugs back to architect/PM ─────┘
```

Each agent hands off to the next with a structured brief. Bugs flow back upstream:
- **Code bugs** (wrong API call, bad manifest, SDK error) → back to **Architect**
- **Requirements bugs** (missing field, wrong scope, unclear mapping) → back to **PM**
- **Systematic agent errors** (skill produces same mistake repeatedly) → **Skill Improver** (`/devrev:improve-skill`)

## Key DevRev Technical Facts

- **Two snap-in types**: Simple (automations, commands, webhooks) and AirSync (data sync connectors)
- **Simple snap-ins** use `@devrev/typescript-sdk`
- **AirSync snap-ins** use `@devrev/ts-adaas` SDK (+ typescript-sdk for DevRev API calls)
- **Timeout**: `processTask({ task, onTimeout })` — SDK handles 10-min soft / 13-min hard timeout
- **Sync model**: DevRev Airdrop platform triggers periodic sync and provides `lastSuccessfulSyncStarted` — snap-ins do NOT register webhooks
- **Batching**: `adapter.initializeRepos(repos)` + `adapter.getRepo("type").push(items)` — SDK batches automatically
- **Mapping**: `chef-cli configure-mappings` generates `initial_domain_mapping.json`
- **Metadata**: `chef-cli validate-metadata` validates `external_domain_metadata.json`
- **Snap-in Builder MCP**: Required for snap-in commands. Provides `validate_metadata`, `get_code_template` (18 patterns), `get_decision_guide`, `scaffold_snapin`, `get_devrev_object_schema`, and guide search. Setup: `claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp`
- **Dashboard creation**: Widget JSON → widget-preview → widget ID → dashboard-preview → dashboard ID
- **Widget JSON**: data_sources (oasis) → dimensions + measures → sub_widgets → visualization
- **SQL rules**: No SELECT *, no table aliases in sql_expression, list every column, fully qualify all references
- **Visualization types**: metric, line, column, bar, table, donut, pie, packed_bubble, heatmap
- **Dashboard URL**: `https://app.devrev.ai/<slug>/dashboard?dashboardId=<ID>`

## Real Examples

PRD and TDD examples from actual DevRev projects are in `examples/`:
- `example-slack-tdd.md` — Slack channel import (OAuth, cursor pagination, mention conversion)
- `example-monday-tdd.md` — Monday.com board/item import (GraphQL, 20 column types)
- `example-planhat-prd.md` — Planhat bidirectional sync (10 object types, API key auth)
- `example-snowflake-prd.md` — Snowflake table import (JWT + OAuth, Streams for incremental)
- `example-trello-prd.md` — Trello board/card import PRD
- `example-trello-tdd.md` — Trello AirSync connector TDD
