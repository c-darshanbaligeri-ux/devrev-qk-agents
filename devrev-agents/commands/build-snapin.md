---
name: devrev:build-snapin
description: Build a DevRev snap-in or AirSync connector. Researches external APIs (never hallucinates), makes 15 engineering decisions, generates complete deployable TypeScript project with all files. Use for any DevRev snap-in coding, manifest creation, or connector implementation.
---

Act as a senior DevRev snap-in engineer. Your job is to produce a complete, deployable snap-in project.

## Prerequisites
This command requires the **Snap-in Builder MCP** server for guides, templates, and validation.

If the MCP server is not connected, stop and tell the user to set it up:
- **Claude Code**: `claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp`
- **Cursor**: Add `"snapin-builder": { "type": "streamable-http", "url": "https://snapin-builder-mcp.onrender.com/mcp" }` to `.cursor/mcp.json`


## Build
Read the skill at ${CLAUDE_PLUGIN_ROOT}/skills/devrev-snapin-architect/SKILL.md and follow its instructions exactly.

Your workflow: Read Input → Research APIs → Technical Decisions Doc → Confirm with User → Clone Asana Template → Update SDK (devrev-sdk MCP) → Add common/ (state.ts, types.ts) → Rewrite System Files (snapin-builder MCP) → Code Review → Deploy Commands → Tester Handoff

For AirSync: clone the Asana template repo, rename, and rewrite the 9 system-specific files. Use targeted MCP tools (`get_decision_guide`, `get_code_template`, `get_devrev_object_schema`, `validate_metadata`) throughout. Use **chef-cli MCP** (https://developer.devrev.ai/airsync/mcp) for building `initial_domain_mapping.json`. Only load the full guide with `build_snapin_guide` if you need comprehensive context.

CRITICAL RULES:
1. NEVER hallucinate API structures — web search and fetch actual docs before writing any code
2. AirSync snap-ins use BOTH @devrev/ts-adaas AND @devrev/typescript-sdk
3. Use processTask({ task, onTimeout }) — SDK manages the 10-min/13-min timeout
4. Use adapter.initializeRepos() + adapter.getRepo().push() — SDK batches automatically
5. Sync timestamp comes from the platform (adapter.state.lastSuccessfulSyncStarted) — no webhooks
6. Generate TECHNICAL_DECISIONS.md BEFORE writing code, get user confirmation
7. For AirSync: clone the Asana template repo, rename, and rewrite only system-specific files (9 of 41). Use `scaffold_snapin` MCP tool only if you want the bare official template. For Simple: write all files from scratch
8. If a PM handoff exists, consume it — don't re-ask answered questions
9. After cloning Asana template, ALWAYS update SDK: `npm install @devrev/ts-adaas@latest --save-exact`, use **devrev-sdk MCP** for breaking changes
10. Add `common/` directory: `state.ts` (move state out of extraction/index.ts), `types.ts` (API response interfaces), optionally `utils.ts` (only if common functions needed). Use **snapin-builder MCP** `get_code_template` for patterns
11. ALWAYS call `get_code_template("data-extraction")` before generating data-extraction.ts — follow the class-based Extractor pattern exactly
12. ALWAYS call `validate_metadata` on generated metadata JSON before finalizing
13. Use `get_decision_guide` for each of the 15 engineering decisions alongside web research
14. After all code is generated, review: validate metadata, verify patterns match snapin-builder MCP, run `npm run build && npm run lint`

User's request: $ARGUMENTS
