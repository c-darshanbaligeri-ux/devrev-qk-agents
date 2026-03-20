# DevRev Agents

AI-powered agents that plan, build, and test snap-ins, connectors, and dashboards on DevRev's AgentOS platform.

> **Stop building DevRev connectors and dashboards manually.** These agents handle the full pipeline — from gathering requirements to generating deployable code/configs to running end-to-end tests.

## Architecture

![DevRev Agents — Complete Architecture](devrev_agents_mindmap.png)

## What's inside

### Snap-in vertical (v1.0)

| Agent | What it does |
|-------|-------------|
| **Snap-in PM** | Gathers requirements, validates feasibility, creates PRDs and TDDs matching real DevRev document patterns, gets approval before development |
| **Snap-in Architect** | Researches external APIs (never guesses), makes 15 engineering decisions, generates complete deployable projects using ADaaS SDK |
| **Snap-in Tester** | Writes Jest unit tests, performs UI automation via browser to install snap-in, configure mapping, run sync, and verify imported data |

### Implementation vertical (v1.1)

| Agent | What it does |
|-------|-------------|
| **Implementation PM** | Gathers dashboard requirements in focused rounds (3-5 questions each), produces structured widget specs with data sources, visualizations, and filters. Handles both new dashboards and fix requests |
| **Implementation Architect** | Generates complete widget JSON for widget-preview, assembles dashboard layouts for dashboard-preview. Fix mode: fetches existing JSON, diagnoses SQL/reference issues, produces corrected JSON with diff |
| **Implementation Tester** | JSON validation checklist (catches 90% of errors before deployment) + UI verification via browser (widget-preview → dashboard-preview → Notebook cross-check for data accuracy) |

### Cross-cutting

| Agent | What it does |
|-------|-------------|
| **Skill Improver** | Diagnoses agent mistakes, traces errors to specific reference files, applies targeted fixes to prevent repeats |

## Quick Setup (one command)

### Claude Code

```bash
/plugin install --github QK-SnapIn/devrev-qk-agents
```

That's it. All 7 agents, 11 commands, and 7 skills are ready to use.

### MCP Server Setup (Required for snap-in commands)

The snap-in commands require the **Snap-in Builder MCP** server for guides, validation, and code templates.

**Claude Code:**

```bash
claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp
```

**Cursor** — add to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "snapin-builder": {
      "type": "streamable-http",
      "url": "https://snapin-builder-mcp.onrender.com/mcp"
    }
  }
}
```


### Update to latest

```bash
/devrev:update
```

### Local development

```bash
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
cd devrev-qk-agents
claude --plugin-dir .
```

### Cursor

```bash
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
cp -r devrev-qk-agents/devrev-agents/skills/* /path/to/your/project/.cursor/skills/
```

## Commands

| Command | What it does |
|---------|-------------|
| `/devrev:plan-snapin` | Plan a snap-in or AirSync connector (PM agent) |
| `/devrev:build-snapin` | Build complete deployable snap-in code (Architect agent) |
| `/devrev:test-snapin` | Test with unit tests + UI automation (Tester agent) |
| `/devrev:update-snapin` | Update an existing snap-in (add entity, fix pagination, switch auth) |
| `/devrev:generate-metadata` | Generate metadata + mapping JSON with validation |
| `/devrev:search-guide` | Quick reference lookup for AirSync patterns |
| `/devrev:plan-implementation` | Plan dashboards, widgets, analytics (PM agent) |
| `/devrev:build-implementation` | Generate widget JSON + dashboard layout (Architect agent) |
| `/devrev:test-implementation` | JSON validation + UI verification (Tester agent) |
| `/devrev:improve-skill` | Report mistakes, update agent skills (Self-learning agent) |
| `/devrev:update` | Update plugin to the latest version |

## Usage

```bash
# Plan a connector
/devrev:plan-snapin Build an AirSync connector for HubSpot

# Build the code
/devrev:build-snapin Implement the HubSpot connector from the approved TDD

# Test it
/devrev:test-snapin Write unit tests and run E2E test for the HubSpot connector
```

Or just use natural language:

```
You: "I need to sync Asana tasks into DevRev"
→ PM agent activates, gathers requirements

You: "Generate the TypeScript code for the extractor"
→ Architect agent activates, researches Asana API, builds code

You: "Test this connector end-to-end"
→ Tester agent activates, writes tests + drives UI
```

## How the pipelines work

See the [architecture diagram above](#architecture) for the complete flow. Both verticals follow the same pattern:

**PM** (plan) → **Architect** (build) → **Tester** (verify) — with bugs flowing back upstream:
- **Code bugs** (wrong API call, bad manifest, SDK error) → back to **Architect**
- **Requirements bugs** (missing field, wrong scope, unclear mapping) → back to **PM**
- **Systematic agent errors** (skill produces same mistake repeatedly) → **Skill Improver**

## Key technical facts

### Snap-in development
- Uses `processTask({ task, onTimeout })` — SDK manages timeout, not manual tracking
- `adapter.initializeRepos()` + `.push()` for automatic batching
- Platform provides `lastSuccessfulSyncStarted` — no webhook registration
- `chef-cli validate-metadata` + `configure-mappings` for AirSync
- Architect researches real API docs before writing code (never hallucinates)
- 15 engineering decisions documented before any code generation

### Dashboard development
- Widget JSON → widget-preview → widget ID → dashboard-preview → dashboard ID
- Widget structure: `data_sources` (oasis) → `dimensions` + `measures` → `sub_widgets` → `visualization`
- SQL rules: No `SELECT *`, no table aliases in `sql_expression`, list every column, fully qualify all references
- Visualization types: metric, line, column, bar, table, donut, pie, packed_bubble, heatmap
- Dashboard grid: 12 columns wide. Metric cards (height:2), Charts (height:4), Tables (height:6)
- Dashboard URL: `https://app.devrev.ai/<slug>/dashboard?dashboardId=<ID>`

## Real document examples

The agents produce PRDs and TDDs matching actual DevRev team documents. Examples in `examples/`:
- `example-slack-tdd.md` — Slack channel import with OAuth scopes, data mapping diagrams
- `example-monday-tdd.md` — Monday.com GraphQL API, 20 column types, workspace/board mapping
- `example-planhat-prd.md` — Planhat bidirectional sync with 10 object types
- `example-snowflake-prd.md` — Snowflake table-to-object mapping with JWT + OAuth auth
- `example-trello-prd.md` — Trello board/card import PRD
- `example-trello-tdd.md` — Trello AirSync connector TDD

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full guide — setup for Claude Code and Cursor, how to report skill issues, how to improve agent skills, and how to add new skills.

Quick version:
1. Fork the repo
2. Test locally: `claude --plugin-dir .`
3. Fix skill issues with `/devrev:improve-skill <what went wrong>`
4. Submit a PR

## License

MIT
