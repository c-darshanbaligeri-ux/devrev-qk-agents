# DevRev Agents

AI-powered agents that plan, build, and test snap-ins, connectors, and dashboards on DevRev's AgentOS platform.

> **Stop building DevRev connectors and dashboards manually.** These agents handle the full pipeline — from gathering requirements to generating deployable code/configs to running end-to-end tests.

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

## Installation

### Claude Code

```bash
# Direct from GitHub
/plugin install --github QK-SnapIn/devrev-qk-agents

# Or clone locally
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
cd devrev-qk-agents

# Option 1: Install locally
claude /plugin install --path .

# Option 2: Session-only (no install)
claude --plugin-dir .
```

### Cursor

```bash
# Copy skills into your project
cp -r devrev-agents/skills/* /path/to/your/project/.cursor/skills/
```

## Usage

### Snap-in commands

```bash
# Plan a connector
/devrev:plan-snapin Build an AirSync connector for HubSpot

# Build the code
/devrev:build-snapin Implement the HubSpot connector from the approved TDD

# Test it
/devrev:test-snapin Write unit tests and run E2E test for the HubSpot connector
```

### Implementation commands

```bash
# Plan a dashboard
/devrev:plan-implementation I need a support dashboard showing ticket volume, SLA compliance, and agent productivity

# Build the widgets + dashboard
/devrev:build-implementation Build the support dashboard from the approved spec

# Test it
/devrev:test-implementation Validate the widget JSON and verify metrics against Notebook
```

### Natural language (skills auto-trigger)

```
You: "I need to sync Asana tasks into DevRev"
→ Snap-in PM agent activates, gathers requirements

You: "Generate the TypeScript code for the extractor"
→ Snap-in Architect agent activates, researches Asana API, builds code

You: "I need a dashboard showing ticket trends by priority"
→ Implementation PM agent activates, runs discovery rounds

You: "Build the widget JSON for the ticket dashboard"
→ Implementation Architect agent activates, generates widget configs

You: "Test the dashboard metrics against Notebook"
→ Implementation Tester agent activates, runs validation + verification
```

## How the pipelines work

### Snap-in pipeline

```
┌─────────────┐     Handoff      ┌──────────────┐     Handoff     ┌─────────────┐
│   Snap-in   │ ──────────────→  │   Snap-in    │ ─────────────→  │   Snap-in   │
│     PM      │   PRD + TDD      │  Architect   │  Code + deploy  │   Tester    │
│             │   + diagrams     │              │  + TECH_DEC.md  │             │
└─────────────┘                  └──────────────┘                 └─────────────┘
  Requirements                    Research →                       Unit tests
  Discovery                       15 decisions →                   UI automation
  Feasibility                     All files to disk                Ship/block
```

### Implementation pipeline

```
┌─────────────┐     Handoff      ┌──────────────┐     Handoff     ┌─────────────┐
│   Impl.     │ ──────────────→  │   Impl.      │ ─────────────→  │   Impl.     │
│     PM      │  Dashboard spec  │  Architect   │  Widget JSON +  │   Tester    │
│             │  + widget specs  │              │  dashboard JSON │             │
└─────────────┘                  └──────────────┘                 └─────────────┘
  Discovery rounds                Read 3 references                JSON validation
  Viz recommendations             Generate JSON                    UI deployment
  Fix specs (bug mode)            Fix mode: diff                   Notebook verify
```

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

The agents produce PRDs and TDDs matching actual DevRev team documents:
- Slack ADaaS TDD — channel import with OAuth scopes, data mapping diagrams
- Monday.com TDD — GraphQL API, 40+ column types, workspace/board mapping
- Planhat PRD — bidirectional sync with persona query examples
- Snowflake PRD — table-to-object mapping with JWT auth

## Contributing

1. Fork the repo
2. Test locally: `claude --plugin-dir .`
3. Make changes to skills/agents/commands
4. Submit a PR

## License

MIT
