# Skill Updates Log

Track every improvement made to DevRev agent skills based on real usage feedback.

---

<!-- New entries go at the top -->

## v1.1.0 — 2026-03-20

### Single-command install and self-update

#### Root-level plugin.json for GitHub install
- **Trigger**: Plugin install required users to know the `devrev-agents/` subdirectory — `/plugin install --github` couldn't find `.claude-plugin/` at repo root
- **File added**: `.claude-plugin/plugin.json` (repo root, paths point into `devrev-agents/`)
- **Change**: Users can now install with a single command: `/plugin install --github QK-SnapIn/devrev-qk-agents`

#### `/devrev:update` command
- **File added**: `commands/update.md`
- **Change**: New command lets users update the plugin from within Claude Code — pulls latest from GitHub, shows changelog, confirms new version
- **Files changed**: `.claude-plugin/plugin.json` (root + inner), `CLAUDE.md` — all updated to include the new command

#### README setup simplified
- **Files changed**: `README.md` (root + inner)
- **Change**: Installation section replaced with single-command quick setup, added commands table with all 8 commands including `/devrev:update`

### AirSync clone-and-rename build method

#### Template-based AirSync snap-in creation
- **Trigger**: Building AirSync snap-ins from scratch required writing 20+ files line by line — slow, error-prone, and inconsistent across connectors
- **Change**: AirSync snap-ins now clone the production Asana snap-in repo as a template, rename system references, and rewrite only 9 system-specific files (22% of project). The other 32 files (78%) are production-ready from the template (23 keep-as-is + 9 handled by find-and-replace)
- **Files changed**: `skills/devrev-snapin-architect/SKILL.md` (Phase 4 rewritten), `skills/devrev-snapin-architect/references/airsync-template.md` (clone procedure added), `commands/build-snapin.md` (rule 7 updated, rule 9 added), `agents/snapin-architect.md` (workflow and rules updated)
- **Impact**: Faster builds, consistent config/boilerplate across all connectors, architect focuses only on system-specific logic
- **Simple snap-ins**: Unchanged — still written from scratch (lighter, 3-5 files)

### Bug fixes & documentation improvements

#### Path reference fixes
- **Trigger**: PM SKILL.md, prd-template.md, and tdd-template.md referenced `references/examples/` but actual path is `examples/`
- **Files changed**: `skills/devrev-snapin-pm/SKILL.md`, `skills/devrev-snapin-pm/references/prd-template.md`, `skills/devrev-snapin-pm/references/tdd-template.md`
- **Change**: Updated all example path references to point to `examples/` directory at plugin root

#### File name reference fixes
- **Trigger**: Architect agent and skill-improver referenced `airsync-snapin.md` but file was renamed to `airsync-template.md` in v1.0.0
- **Files changed**: `agents/snapin-architect.md` (1 occurrence), `skills/devrev-skill-improver/SKILL.md` (5 occurrences)
- **Change**: Updated all references from `airsync-snapin.md` → `airsync-template.md`

#### Architect fallback for missing PM handoff
- **Trigger**: Architect agent rule #8 said "consume PM handoff" but had no explicit fallback when invoked directly
- **File changed**: `agents/snapin-architect.md`
- **Change**: Added explicit fallback questions (what system, what data/direction, API doc links, sync type)

#### Bug routing documentation
- **Trigger**: Pipeline diagram showed bugs flowing back upstream but didn't specify routing rules
- **Files changed**: `CLAUDE.md`, `README.md`
- **Change**: Added explicit routing: code bugs → Architect, requirements bugs → PM, systematic errors → Skill Improver

#### Real snap-in examples added
- **Trigger**: README referenced Slack, Monday.com, Planhat, and Snowflake examples but only Trello existed
- **Files added**: `examples/example-slack-tdd.md`, `examples/example-monday-tdd.md`, `examples/example-planhat-prd.md`, `examples/example-snowflake-prd.md`
- **Change**: Created 4 production-quality examples based on actual snap-in codebases

#### Project guide
- **File added**: `docs/PROJECT-GUIDE.md`
- **Change**: Comprehensive reference covering architecture, step-by-step snap-in creation, bug fixing, and DevRev platform facts

#### PM SKILL.md — stale PDF filenames replaced with actual .md filenames
- **Trigger**: Code review caught that PM SKILL.md listed `.pdf` filenames that don't exist
- **File changed**: `skills/devrev-snapin-pm/SKILL.md`
- **Change**: Replaced PDF refs with actual markdown example filenames, added Trello examples to the list

#### Orphaned duplicate examples removed
- **Trigger**: Refactor scan found duplicate Trello examples in `skills/devrev-snapin-pm/references/examples/`
- **Directory removed**: `skills/devrev-snapin-pm/references/examples/` (identical copies of files in `examples/`)

#### Consistent example disclaimers
- **Trigger**: 2 of 4 new examples had "This is a realistic example" disclaimers, 2 did not
- **Files changed**: `examples/example-slack-tdd.md`, `examples/example-planhat-prd.md`
- **Change**: Added consistent disclaimer blockquote to all 4 new examples

#### Top-level docs updated with all 6 examples
- **Trigger**: CLAUDE.md, README.md only listed 4 examples; Trello examples were missing
- **Files changed**: `CLAUDE.md`, `README.md`, `CONTRIBUTING.md`
- **Change**: All 6 examples now listed with actual filenames and descriptions

#### Snap-in command frontmatter consistency
- **Trigger**: Snap-in commands missing `name:` field that implementation commands had
- **Files changed**: `commands/plan-snapin.md`, `commands/build-snapin.md`, `commands/test-snapin.md`, `commands/improve-skill.md`
- **Change**: Added `name:` field to all 4 snap-in command frontmatters

---

## v1.0.0 — 2026-03-19

### Initial Release

**Snap-in vertical** (PM, Architect, Tester) ready for production use.

#### Skills
- **devrev-snapin-pm** — Discovery, PRD, TDD, handoff brief generation
- **devrev-snapin-architect** — API research, 15 engineering decisions, full code generation
- **devrev-snapin-tester** — Unit tests (Jest) + DevRev UI automation flows
- **devrev-skill-improver** — Diagnose and fix skill errors from real usage

#### Key changes from pre-release
- Replaced `airsync-snapin.md` with `airsync-template.md` — complete generic scaffold derived from `devrev/airdrop-asana-snap-in` and cross-referenced with a production Zoho CRM connector
- Fixed critical SDK patterns: `spawn()` routing, `adapter.streamAttachments()`, metadata repo push, `connection_data.key` JSON parsing, `adapter.postState()` in onTimeout
- Added both auth patterns (PAT and config-type OAuth with client_credentials)
- Removed unfinished implementation vertical stubs (will ship in a future release)
