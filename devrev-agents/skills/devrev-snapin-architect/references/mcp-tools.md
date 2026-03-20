# Snap-in Builder MCP Tools Reference

The Snap-in Builder MCP server provides 9 tools for building DevRev AirSync snap-ins.

## Setup

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

## Full Guide Tools

### `build_snapin_guide`
Returns the complete builder guide. Use when you need comprehensive context for generating a full snap-in codebase. Covers: authentication patterns, extraction phases, entity mapping, state management, rate limiting, error handling, and file-by-file generation rules.

- **Parameters**: none
- **When to use**: Only when targeted tools don't have enough context

### `metadata_guide`
Returns the complete metadata generator guide. Use when generating `external_domain_metadata.json` or `initial_domain_mapping.json`. Covers: field types, reference syntax, collection syntax, stage diagrams, DevRev object type mappings, and validation rules.

- **Parameters**: none
- **When to use**: Only when `get_devrev_object_schema` and `validate_metadata` aren't sufficient

## Targeted Tools (prefer these)

### `scaffold_snapin`
Clone the official DevRev AirSync template (`devrev/airdrop-template`) and get setup instructions.

- **Parameters**:
  - `system_name` (required) — External system name (e.g., "Wrike", "HubSpot")
  - `target_dir` (optional) — Directory to clone into
- **When to use**: Starting a new AirSync snap-in project

### `get_decision_guide`
Look up a specific architectural decision with code examples.

- **Parameters**:
  - `decision` (required) — Decision number (e.g., "1") or keyword (e.g., "authentication", "pagination", "sync unit", "bidirectional", "extraction order")
- **When to use**: Making engineering decisions during Phase 2/3

### `get_code_template`
Return a specific code pattern from the guides.

- **Parameters**:
  - `template` (required) — Template name or keyword
- **Available templates**:
  - `oauth-manifest` — OAuth2 auth + manifest pattern
  - `pat-manifest` — API key/PAT auth + manifest pattern
  - `manifest` — Full manifest.yaml template
  - `static-metadata` / `dynamic-metadata` — Metadata approaches
  - `data-extraction` — **MANDATORY** before generating data-extraction.ts (class-based Extractor pattern)
  - `sync-unit-extraction` — Sync unit worker (Pattern A: API-listed, Pattern B: org)
  - `attachment-streaming` — Attachments worker + collection patterns
  - `nested-children` — Comments/attachments extracted inline with parent
  - `loading-worker` — Bidirectional loading worker
  - `pagination` — Cursor/offset/page pagination patterns
  - `api-service` — API client class (axios, retry, rate limiting)
  - `data-normalization` — Normalization functions (external → DevRev)
  - `validate-input` — Input validation (verify connection, check config)
  - `stage-diagram` — Stage diagrams for workflow states
  - `custom-links` — Custom link types between entities
  - `permissions` / `article-permissions` — Permission fields + scope patterns
- **When to use**: Generating each system-specific file during Phase 4

### `get_devrev_object_schema`
Look up required and optional fields for a DevRev object type.

- **Parameters**:
  - `object_type` (required) — e.g., "ticket", "issue", "task", "opportunity", "incident", "article", "devu", "revu", "account", "group", "comment", "link", "tag", "new_custom_object"
- **When to use**: Before generating metadata JSON — ensures required fields are included

### `validate_metadata`
Validate `external_domain_metadata.json` against DevRev AirSync rules.

- **Parameters**:
  - `metadata_json` (required) — The JSON content as a string
- **Catches**: Missing schema_version, forbidden fields (id, created_date), invalid reference syntax, broken stage diagrams, hallucinated types, invalid collection syntax
- **When to use**: ALWAYS after generating or modifying metadata JSON. Fix errors and re-validate until clean.

### `search_snapin_guide`
Full-text search across both guides.

- **Parameters**:
  - `query` (required) — Topic to search (e.g., "rate limiting", "ExtractionDataDelay", "OAuth2")
- **When to use**: When you need information on a specific topic but don't know which tool has it

### `suggest_guide_update`
Report a mistake or correction to improve the guides.

- **Parameters**:
  - `guide` (required) — "snapin" or "metadata"
  - `trigger` (required) — What went wrong
  - `current_content` (required) — Exact text that is wrong
  - `corrected_content` (required) — Corrected replacement text
  - `reason` (optional) — Why this fix is correct
- **When to use**: When you discover the guide has incorrect information
