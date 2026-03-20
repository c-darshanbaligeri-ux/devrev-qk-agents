# DevRev Agents

AI-powered agents that plan, build, and test snap-ins and connectors on DevRev's AgentOS platform.

> **Stop building DevRev connectors manually.** These agents handle the full pipeline — from gathering requirements to generating deployable TypeScript code to running end-to-end tests.

## Quick Setup

### Claude Code

```bash
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
claude --plugin-dir ./devrev-qk-agents
```

All 7 agents, 11 commands, and 7 skills are ready to use.

To update to the latest version, pull the repo and restart Claude Code — or run `/devrev:update` from within a session.

### Cursor

```bash
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
cp -r devrev-qk-agents/devrev-agents/skills/* /path/to/your/project/.cursor/skills/
```

### MCP Server Setup (Required for snap-in commands)

The snap-in commands (`/devrev:build-snapin`, `/devrev:update-snapin`, `/devrev:generate-metadata`, `/devrev:search-guide`) require the **Snap-in Builder MCP** server for guides, validation, and code templates.

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
| **Implementation PM** | Gathers dashboard requirements in focused rounds, produces structured widget specs, handles both new dashboards and fix requests |
| **Implementation Architect** | Generates complete widget JSON for widget-preview, assembles dashboard layouts, fixes existing widgets by diagnosing SQL/reference issues |
| **Implementation Tester** | JSON validation checklist (catches 90% of errors before deployment) + UI verification via browser (widget-preview → dashboard-preview → Notebook cross-check) |

### Cross-cutting

| Agent | What it does |
|-------|-------------|
| **Skill Improver** | Diagnoses agent mistakes, traces errors to specific reference files, applies targeted fixes to prevent repeats |

## Commands

| Command | What it does |
|---------|-------------|
| `/devrev:plan-snapin` | Plan a snap-in or AirSync connector (PM agent) |
| `/devrev:build-snapin` | Build complete deployable snap-in code (Architect agent) |
| `/devrev:test-snapin` | Test with unit tests + UI automation (Tester agent) |
| `/devrev:plan-implementation` | Plan dashboards, widgets, analytics (PM agent) |
| `/devrev:build-implementation` | Generate widget JSON + dashboard layout (Architect agent) |
| `/devrev:test-implementation` | JSON validation + UI verification (Tester agent) |
| `/devrev:update-snapin` | Update an existing snap-in (add entity, fix pagination, switch auth) |
| `/devrev:generate-metadata` | Generate metadata + mapping JSON with validation |
| `/devrev:search-guide` | Quick reference lookup for AirSync patterns |
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

## How the pipeline works

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

Bugs flow back upstream:
- **Code bugs** (wrong API call, bad manifest, SDK error) → back to **Architect**
- **Requirements bugs** (missing field, wrong scope, unclear mapping) → back to **PM**
- **Systematic agent errors** (skill produces same mistake repeatedly) → **Skill Improver**

## Real document examples

The agents produce PRDs and TDDs matching actual DevRev team documents. Examples in `devrev-agents/examples/`:
- `example-slack-tdd.md` — Slack channel import with OAuth scopes, data mapping diagrams
- `example-monday-tdd.md` — Monday.com GraphQL API, 20 column types, workspace/board mapping
- `example-planhat-prd.md` — Planhat bidirectional sync with 10 object types
- `example-snowflake-prd.md` — Snowflake table-to-object mapping with JWT + OAuth auth
- `example-trello-prd.md` — Trello board/card import PRD
- `example-trello-tdd.md` — Trello AirSync connector TDD

## Contributing

1. Fork the repo
2. Test locally: `claude --plugin-dir .`
3. Make changes to skills/agents/commands
4. Fix skill issues with `/devrev:improve-skill <what went wrong>`
5. Submit a PR

See [CONTRIBUTING.md](devrev-agents/CONTRIBUTING.md) for the full guide.

## License

MIT
