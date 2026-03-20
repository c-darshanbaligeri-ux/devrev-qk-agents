---
name: devrev:update-snapin
description: Update or improve an existing DevRev AirSync snap-in. Use for "add a new entity", "fix pagination", "switch auth to OAuth", "add bidirectional loading", "add attachments", "fix rate limiting", "update metadata", or any modification to an existing snap-in project.
---

Act as a senior DevRev AirSync engineer. Your job is to update an existing AirSync snap-in — NOT build a new one from scratch.

## Prerequisites
This command requires the **Snap-in Builder MCP** server.

If the MCP server is not connected, stop and tell the user to set it up:
- **Claude Code**: `claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp`
- **Cursor**: Add `"snapin-builder": { "type": "streamable-http", "url": "https://snapin-builder-mcp.onrender.com/mcp" }` to `.cursor/mcp.json`


## Workflow

### Step 1: Understand the existing project

Read the existing snap-in project to understand what's already built. Check these files:

1. **`manifest.yaml`** — auth method, functions, keyring, imports, inputs
2. **`external_domain_metadata.json`** — which entities and fields are defined
3. **`initial_domain_mapping.json`** — how external fields map to DevRev fields
4. **`extraction/workers/data-extraction.ts`** — which entities are extracted, extraction patterns
5. **`external-system/client.ts`** or similar — API client, base URL, auth headers
6. **`common/constants.ts`** — entity type constants
7. **`common/state.ts`** — state shape for each entity
8. **`data-normalization.ts`** — how external records are transformed

Summarize what the snap-in currently does before making changes.

### Step 2: Understand what needs to change

Ask the user what they want to update if not clear from their request. Common update types:

| Update type | Files affected |
|---|---|
| Add new entity | metadata, mapping, constants, state, normalization, data-extraction, client |
| Fix/change pagination | data-extraction, client |
| Switch auth method | manifest.yaml, client |
| Add bidirectional loading | manifest.yaml, loading workers, denormalization |
| Add attachments | metadata, data-extraction (nested children), attachments-extraction |
| Fix rate limiting | client, data-extraction (emitError) |
| Add nested children (comments) | metadata, state, data-extraction, normalization |
| Update field mappings | metadata, mapping, normalization |
| Add regional URL support | manifest.yaml (keyring fields + URL placeholders), client (dynamic base URL) |

### Step 3: Look up the right patterns

Use targeted MCP tools to get the patterns for the specific change:

| Template | When to use |
|---|---|
| `get_code_template("data-extraction")` | Any data-extraction changes (MUST follow class pattern) |
| `get_code_template("api-service")` | API client changes (new endpoints, auth header, retry) |
| `get_code_template("data-normalization")` | Adding/changing normalization functions |
| `get_code_template("sync-unit-extraction")` | Changing sync unit logic |
| `get_code_template("nested-children")` | Adding comments/attachments per parent |
| `get_code_template("manifest")` | Full manifest template for reference |
| `get_code_template("oauth-manifest")` / `("pat-manifest")` | Auth changes |
| `get_code_template("loading-worker")` | Adding bidirectional loading |
| `get_code_template("attachment-streaming")` | Adding attachment support |
| `get_code_template("validate-input")` | Adding/fixing input validation |
| `get_code_template("pagination")` | Fixing pagination |
| `get_code_template("stage-diagram")` | Adding workflow states |
| `get_code_template("custom-links")` | Adding link types between entities |
| `get_code_template("permissions")` / `("article-permissions")` | Adding permissions |
| `get_decision_guide("<topic>")` | Architectural decisions |
| `get_devrev_object_schema("<type>")` | Required DevRev fields when adding entities |
| `search_snapin_guide("<query>")` | Anything else |

### Step 4: Apply changes

Edit the existing files. Do NOT rewrite files from scratch — make targeted modifications.

When adding a new entity to data-extraction.ts:
1. Add the constant to `constants.ts`
2. Add the state interface to `state.ts`
3. Add the normalize function to `data-normalization.ts`
4. Add the API method to the client
5. Add the repo entry to the `repos` array
6. Add the entity to the `entities` array in `extractData()`
7. Add a new `case` in the switch and a new private extraction method
8. Add the entity to `external_domain_metadata.json`
9. Add the mapping to `initial_domain_mapping.json`
10. Validate with `validate_metadata`

### Step 5: Validate

- If metadata was changed: call `validate_metadata` with the updated JSON
- If extraction logic changed: verify the Extractor class pattern is maintained (class-based, per-entity methods, shared emitError, no inline Progress emissions)
- Review the diff to ensure no existing functionality was broken

## CRITICAL RULES:
1. NEVER rewrite the entire project — only modify what's needed
2. NEVER break the Extractor class pattern — if the existing code uses flat inline blocks, refactor it to the class pattern while making changes
3. NEVER hallucinate API structures — research the external system's docs for any new entities/endpoints
4. ALWAYS validate metadata after changes with `validate_metadata`
5. ALWAYS maintain existing entity extraction — don't remove or break working entities while adding new ones
6. Read the existing code BEFORE suggesting any changes

User's request: $ARGUMENTS
