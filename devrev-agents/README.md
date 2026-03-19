# DevRev Agents

AI-powered agents that plan, build, and test snap-ins and connectors on DevRev's AgentOS platform.

> **Stop building DevRev connectors manually.** These agents handle the full pipeline — from gathering requirements to generating deployable TypeScript code to running end-to-end tests.

## What's inside

### Snap-in vertical (v1.0 — ready)

| Agent | What it does |
|-------|-------------|
| **Snap-in PM** | Gathers requirements, validates feasibility, creates PRDs and TDDs matching real DevRev document patterns, gets approval before development |
| **Snap-in Architect** | Researches external APIs (never guesses), makes 15 engineering decisions, generates complete deployable projects using ADaaS SDK |
| **Snap-in Tester** | Writes Jest unit tests, performs UI automation via browser to install snap-in, configure mapping, run sync, and verify imported data |

### Implementation vertical (coming soon)

| Agent | What it will do |
|-------|----------------|
| Implementation PM | Plan dashboards, workflows, AI agents, PLuG configurations |
| Implementation Architect | Build via DevRev APIs |
| Implementation Tester | Verify dashboard data, workflow triggers, agent responses |

## Installation

### Claude Code

```bash
# Direct from GitHub
/plugin install --github <your-org>/devrev-agents

# Or from marketplace (after publishing)
/plugin marketplace add <your-org>/devrev-agents-marketplace
/plugin install devrev-agents@devrev-agents-marketplace
```

### Local testing

```bash
git clone https://github.com/<your-org>/devrev-agents.git
cd devrev-agents

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

### Commands (explicit invoke)

```bash
# Plan a connector
/devrev:plan-snapin Build an AirSync connector for HubSpot

# Build the code
/devrev:build-snapin Implement the HubSpot connector from the approved TDD

# Test it
/devrev:test-snapin Write unit tests and run E2E test for the HubSpot connector
```

### Natural language (skills auto-trigger)

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

## Real document examples

The agents produce PRDs and TDDs matching actual DevRev team documents:
- Slack ADaaS TDD — channel import with OAuth scopes, data mapping diagrams
- Monday.com TDD — GraphQL API, 40+ column types, workspace/board mapping
- Planhat PRD — bidirectional sync with persona query examples
- Snowflake PRD — table-to-object mapping with JWT auth

## Key technical decisions built into the agents

- Uses `processTask({ task, onTimeout })` — SDK manages timeout, not manual tracking
- `adapter.initializeRepos()` + `.push()` for automatic batching
- Platform provides `lastSuccessfulSyncStarted` — no webhook registration
- `chef-cli validate-metadata` + `configure-mappings` for AirSync
- Architect researches real API docs before writing code (never hallucinates)
- 15 engineering decisions documented before any code generation

## Contributing

1. Fork the repo
2. Test locally: `claude --plugin-dir .`
3. Make changes to skills/agents/commands
4. Submit a PR

## License

MIT
