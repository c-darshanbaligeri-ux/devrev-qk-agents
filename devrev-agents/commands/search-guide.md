---
name: devrev:search-guide
description: Quick reference lookup for AirSync patterns. Use for "how does X work", "what's the pattern for Y", or questions about event types, error handling, rate limiting, authentication, state management.
---

Answer the user's question using the most targeted MCP tool available:

## Prerequisites
This command requires the **Snap-in Builder MCP** server.

If the MCP server is not connected, stop and tell the user to set it up:
- **Claude Code**: `claude mcp add snapin-builder --transport http -s project https://snapin-builder-mcp.onrender.com/mcp`
- **Cursor**: Add `"snapin-builder": { "type": "streamable-http", "url": "https://snapin-builder-mcp.onrender.com/mcp" }` to `.cursor/mcp.json`


## Lookup strategy

1. **Specific decision?** → Call `get_decision_guide` (e.g., "authentication", "pagination", "bidirectional")
2. **Code pattern?** → Call `get_code_template` (e.g., "oauth-manifest", "nested-children", "permissions")
3. **DevRev object fields?** → Call `get_devrev_object_schema` (e.g., "ticket", "article", "devu")
4. **General search?** → Call `search_snapin_guide` with the query
5. **Still not enough?** → Call `build_snapin_guide` or `metadata_guide` for the full guide

Quote the exact patterns and code from the guide. Keep answers focused.

User's question: $ARGUMENTS
