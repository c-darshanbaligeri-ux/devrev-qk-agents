---
name: devrev:generate-metadata
description: Generate external_domain_metadata.json and initial_domain_mapping.json for a DevRev AirSync snap-in. Use for "generate metadata for X", "create field mappings", or "build the metadata JSON".
---

Act as a DevRev metadata specialist. Generate the metadata and mapping files for an AirSync snap-in.

## Prerequisites
This command requires **two MCP servers**:
1. **Snap-in Builder MCP** — for `metadata_guide`, `get_devrev_object_schema`, `validate_metadata` tools
2. **DevRev AirSync MCP** (via `chef-cli`) — for testing mappings and validating metadata

If either MCP server is not connected, stop and tell the user to set them up:
- **Snap-in Builder MCP (Claude Code)**: `claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp`
- **Snap-in Builder MCP (Cursor)**: Add `"snapin-builder": { "type": "streamable-http", "url": "https://snapin-builder-mcp.onrender.com/mcp" }` to `.cursor/mcp.json`
- **AirSync MCP (Claude Code)**: `claude mcp add airsync chef-cli mcp initial-mapping`
- **AirSync MCP (Cursor)**: Add `"AirSync": { "command": "chef-cli", "args": ["mcp", "initial-mapping"] }` to `.cursor/mcp.json`
- **AirSync Docs**: https://developer.devrev.ai/airsync/mcp

> Ask the team for the current Snap-in Builder MCP server URL.

## Step 1: Get the metadata guide
Call the MCP tool `metadata_guide` to get the complete metadata generator guide. This is your source of truth for field types, reference syntax, collection syntax, stage diagrams, and validation rules.

## Step 2: Research the external system
Web search the external system's API docs to discover actual field names, types, and relationships.

## Step 3: Look up DevRev object schemas
For each entity, call `get_devrev_object_schema` to check required fields:
- `get_devrev_object_schema("ticket")` — for support cases
- `get_devrev_object_schema("issue")` — for work items
- `get_devrev_object_schema("article")` — for KB articles
- `get_devrev_object_schema("devu")` — for internal users

## Step 4: Generate
Follow the metadata guide exactly. Never invent field formats.

## Step 5: Validate
Call `validate_metadata` with the generated JSON. Fix any errors and re-validate until clean.

Rules:
- Schema version is always `v0.2.0`
- Always include `item_url_field` as a `text` field on the primary entity
- Users record type always needs: `name` (text, required) + `email` (text)
- ALWAYS validate with `validate_metadata` before outputting

User's request: $ARGUMENTS
