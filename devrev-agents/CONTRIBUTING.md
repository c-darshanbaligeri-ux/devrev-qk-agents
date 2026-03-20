# Contributing to DevRev Agents

Thanks for helping improve the DevRev agents. This guide covers setup, reporting issues, fixing skills, and the overall improvement workflow.

## Quick start

### Claude Code

```bash
# Clone the repo
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git
cd devrev-qk-agents

# Option 1: Install as a plugin
claude /plugin install --path .

# Option 2: Session-only (no install, good for testing changes)
claude --plugin-dir .
```

### Cursor

```bash
# Clone the repo
git clone https://github.com/QK-SnapIn/devrev-qk-agents.git

# Copy skills into your Cursor project
cp -r devrev-agents/skills/* /path/to/your/project/.cursor/skills/

# Copy agent definitions
cp -r devrev-agents/agents/* /path/to/your/project/.cursor/agents/
```

After copying, Cursor will pick up the skills from the `.cursor/skills/` directory. Each skill folder contains a `SKILL.md` (the main instructions) and a `references/` folder (knowledge the agent reads before acting).

To update skills in Cursor, re-copy the changed files from the repo.

## Reporting issues

### When code generation fails or produces wrong output

1. **Note what went wrong** — wrong API call, bad manifest syntax, missing field, incorrect SDK pattern
2. **Identify which agent failed** — PM (planning), Architect (code generation), or Tester (testing)
3. **Use the built-in skill improver**:

```bash
/devrev:improve-skill The architect generated a manifest with `type: "webhook"` but it should be `type: "flow"`
```

The skill improver agent will:
- Diagnose the root cause
- Adding more examples
- Find the exact reference file that needs fixing
- Produce a minimal patch
- Ask for your confirmation before applying
- Log the change in `docs/CHANGELOG.md`

### When a skill is missing knowledge

```bash
/devrev:improve-skill DevRev now supports a new visualization type "treemap" — add it to the widget catalog
```

### When you want to file a GitHub issue

Open an issue at [github.com/QK-SnapIn/devrev-qk-agents/issues](https://github.com/QK-SnapIn/devrev-qk-agents/issues) with:

- **Title**: `[agent-name] short description` (e.g. `[architect] wrong OAuth scope format`)
- **Body**: What you asked, what the agent produced, what the correct output should be
- **Label**: `skill-bug`, `missing-knowledge`, or `enhancement`

## How agent skills work (for contributors)

Each agent is built from three layers:

```
commands/          → Entry points (what the user invokes)
agents/            → Agent definitions (which skill + references to load)
skills/
  └── <skill>/
      ├── SKILL.md        → Main instructions (the "brain")
      └── references/     → Knowledge files the agent reads before acting
```

When you run `/devrev:build-snapin`, it loads:
1. `commands/build-snapin.md` — routes to the architect agent
2. `agents/snapin-architect.md` — tells Claude which skill and references to read
3. `skills/devrev-snapin-architect/SKILL.md` — the full architect workflow
4. `skills/devrev-snapin-architect/references/*.md` — SDK patterns, CLI commands, templates

**The reference files are where most fixes happen.** If the architect generates wrong code, it's almost always because a reference file has a wrong pattern or is missing information.

## Key steps to improve agent skills

### 1. Identify the root cause

Don't just fix the output — trace back to *why* the agent got it wrong:

| Symptom | Root cause location |
|---------|-------------------|
| Wrong API endpoint / request format | `references/airsync-template.md` or `references/simple-snapin.md` |
| Wrong manifest syntax | `references/simple-snapin.md` or `references/airsync-template.md` |
| Wrong SDK method or import | `references/airsync-template.md` |
| Wrong CLI command | `references/cli-workflow.md` |
| Bad PRD/TDD structure | `references/prd-template.md` or `references/tdd-template.md` |
| Missing discovery question | `references/discovery-questions.md` |
| Wrong widget JSON structure | `references/widget-json-reference.md` |
| Wrong SQL in widget | `references/sql-rules.md` |
| Wrong visualization config | `references/visualization-catalog.md` |
| Wrong test pattern | `references/unit-testing.md` |
| Wrong UI automation step | `references/ui-automation-flow.md` |

### 2. Make minimal, precise edits

- Change only the lines that are wrong
- Don't rewrite entire files for one fix
- Keep the same formatting and structure
- Add comments only if the fix isn't obvious

### 3. Test your change

After editing a reference file, re-run the scenario that failed:

```bash
# Example: you fixed a widget JSON pattern
/devrev:build-implementation Build a metric widget showing total open tickets

# Verify the output uses your corrected pattern
```

### 4. Check for ripple effects

If a pattern was wrong in one file, search for the same pattern in other files:

- A wrong SDK import might appear in both `airsync-template.md` and `SKILL.md`
- A wrong SQL rule might appear in both `sql-rules.md` and `widget-json-reference.md`

### 5. Log every update

Add an entry to `docs/CHANGELOG.md`:

```markdown
### [Date] — [Short description]
- **Trigger**: What went wrong
- **File changed**: Path to the updated file
- **Change**: One-line summary
```

## Adding a new skill

1. Create a folder under `skills/` with a `SKILL.md` and optional `references/` folder
2. Create an agent definition in `agents/` that points to the skill
3. Create a command in `commands/` that invokes the agent
4. Add the command path to both `plugin.json` files (root and `.claude-plugin/`)
5. Update `CLAUDE.md` with the new command
6. Test with `claude --plugin-dir .`

## Project structure

```
devrev-agents/
├── CLAUDE.md                          # Root context for all agents
├── README.md                          # Project overview + architecture diagram
├── CONTRIBUTING.md                    # This file
├── agents/                            # Agent definitions (7 agents)
│   ├── snapin-pm.md
│   ├── snapin-architect.md
│   ├── snapin-tester.md
│   ├── implementation-pm.md
│   ├── implementation-architect.md
│   ├── implementation-tester.md
│   └── skill-improver.md
├── commands/                          # Slash command entry points (7 commands)
│   ├── plan-snapin.md
│   ├── build-snapin.md
│   ├── test-snapin.md
│   ├── plan-implementation.md
│   ├── build-implementation.md
│   ├── test-implementation.md
│   └── improve-skill.md
├── skills/                            # Agent skills + reference knowledge
│   ├── devrev-snapin-pm/
│   ├── devrev-snapin-architect/
│   ├── devrev-snapin-tester/
│   ├── devrev-imp-pm/
│   ├── devrev-imp-architect/
│   ├── devrev-imp-tester/
│   └── devrev-skill-improver/
├── examples/                          # Real PRD/TDD examples
│   ├── example-slack-tdd.md
│   ├── example-monday-tdd.md
│   ├── example-planhat-prd.md
│   ├── example-snowflake-prd.md
│   ├── example-trello-prd.md
│   └── example-trello-tdd.md
├── docs/                              # Changelog + project guide
│   ├── CHANGELOG.md
│   └── PROJECT-GUIDE.md
└── .claude-plugin/plugin.json         # Plugin manifest
```

## Code of conduct

- Test your changes before submitting a PR
- Keep reference files factual — no guessing, no hallucinated API endpoints
- If you're unsure about a DevRev SDK pattern, check the [official docs](https://developer.devrev.ai/) or a working snap-in first
- Use `/devrev:improve-skill` for quick fixes; open a PR for larger changes
