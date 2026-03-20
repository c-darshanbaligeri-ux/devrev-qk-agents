# PRD: Snowflake – DevRev Airdrop Integration

> **This is a realistic example** demonstrating the PRD format used by DevRev teams. It follows the same structure, depth, and patterns as real DevRev PRDs.

## Overview

This document outlines the approach to build a one-way sync from Snowflake to DevRev using the AirSync (ADaaS) platform. The goal is to enable teams storing structured data in Snowflake to import tables as custom objects in DevRev, making warehouse data searchable, actionable in workflows, and available to AI agents without manual export/import cycles.

## Problem Statement

Teams using Snowflake as their data warehouse cannot surface table data inside DevRev without manual CSV exports or custom scripts. Customer-facing teams lack visibility into operational data stored in Snowflake, and analysts cannot correlate warehouse metrics with support tickets or product issues. This disconnect forces manual data pulls that are error-prone, stale within hours, and consume ~30 minutes per report cycle.

## Goal

1. Enable one-way sync of Snowflake tables into DevRev as custom objects via Airdrop
2. Support both key-pair (JWT) and OAuth client credentials authentication
3. Provide automatic schema discovery and type mapping from Snowflake columns to DevRev fields
4. Support incremental sync via Snowflake Streams to keep data current without full re-extraction
5. Power AI-based search and automation in DevRev using Snowflake data context

## Persona Queries

| Persona | Query | Intent/Action |
|---------|-------|---------------|
| Support Agent | Show me the latest customer health scores from Snowflake for account Acme Corp | Pull real-time customer data to inform support response |
| Product Manager | Which products have the highest return rates this quarter? | Surface warehouse analytics in DevRev for product decisions |
| Data Analyst | When was the last successful sync from Snowflake ANALYTICS.PUBLIC.ORDERS? | Verify data freshness before reporting |
| Engineering Lead | List all Snowflake tables currently synced to DevRev with row counts | Audit integration health and coverage |
| Customer Success | Show me all customers whose MRR dropped more than 20% last month | Identify at-risk accounts using synced financial data |

## Sync Direction

- **Snowflake -> DevRev**: Databases, Schemas, Tables (as sync units), Rows (as custom object items), Columns (as fields)
- **DevRev -> Snowflake**: Not supported. One-way only.
- **Sync type**: One-way with support for both full re-extraction and incremental sync via Snowflake Streams.

## Authentication

Two authentication methods are supported, configured at install time.

### Method 1: Key-Pair (JWT) -- Primary

- **Fields**: `username`, `privateKey` (PEM format), `privateKeyPassphrase` (optional), `account_identifier`, `warehouse_name`, `role` (optional)
- **PEM handling**: The snap-in auto-normalizes PEM keys to enforce 64-character line breaks. Encrypted private keys are supported when a passphrase is provided.
- **Token storage**: Private keys and passphrases stored in DevRev secure keyrings, never logged or exposed.
- **Reference**: https://docs.snowflake.com/en/developer-guide/sql-api/authenticating#using-key-pair-authentication

### Method 2: OAuth Client Credentials (M2M)

- **Fields**: `oauthClientId`, `oauthClientSecret`, `oauthTokenRequestUrl`, `oauthScope`, `account_identifier`, `warehouse_name`, `role` (optional)
- **Token storage**: Client secrets stored in DevRev secure keyrings, auto-refreshed on expiry.
- **Reference**: https://docs.snowflake.com/en/user-guide/oauth-custom#using-client-credentials

## Object Mapping

| Snowflake Entity | DevRev Object | Sync Direction | Notes |
|------------------|---------------|----------------|-------|
| Database + Schema + Table | External Sync Unit | -> DevRev | Named as `DB.SCHEMA.TABLE`; user selects which to import |
| Table | Custom Object (type) | -> DevRev | One custom object type created per table |
| Row | Custom Object Item | -> DevRev | Each row becomes one item in the custom object |
| Column | Field | -> DevRev | Column names lowercased; types mapped per conversion table below |

### Excluded Databases

The following system databases are excluded from sync unit discovery:
- `SNOWFLAKE`
- `SNOWFLAKE_SAMPLE_DATA`
- `SNOWFLAKE_LEARNING_DB`

### Type Conversion Table

| Snowflake Type(s) | DevRev Field Type | Notes |
|--------------------|-------------------|-------|
| BOOLEAN, BOOL | bool | |
| INT, INTEGER, BIGINT, SMALLINT, TINYINT, NUMBER, DECIMAL, NUMERIC | int | |
| FLOAT, FLOAT4, FLOAT8, DOUBLE, REAL | float | |
| VARCHAR, CHAR, STRING, TEXT | text | |
| DATE | date | |
| TIMESTAMP, TIMESTAMP_LTZ, TIMESTAMP_NTZ, TIMESTAMP_TZ, DATETIME, TIME | timestamp | Converted to RFC 3339 format |
| VARIANT, OBJECT, ARRAY, MAP | struct | Stored as structured JSON |
| All other types | text | Fallback: serialized to string |

### Stock Fields

Every synced item receives two auto-generated stock fields:

- **title**: Auto-generated as `"{TABLE}: {OFFSET+IDX}"` where IDX is the row index within the current batch
- **item_url_field**: Back-link URL to the Snowflake table preview in the Snowflake web UI

## Permission Mapping

Not needed for V1. The snap-in imports only tables the authenticated Snowflake user/role has `SELECT` permission on, which implicitly honors Snowflake's RBAC. DevRev-side access control applies to the custom objects after import. V2 may add role-to-group mapping.

## Proposed Architecture

1. **Authentication**: User provides Key-Pair or OAuth credentials through Airdrop settings at install time
2. **Sync unit selection**: Snap-in queries `SHOW DATABASES`, `SHOW SCHEMAS`, and `SHOW TABLES` to present available tables. User selects which `DB.SCHEMA.TABLE` combinations to import.
3. **Schema interpretation**: For each selected table, the snap-in runs `DESC TABLE` to discover columns, types, and nullable status
4. **Field mapping**: Auto-suggested mappings based on the type conversion table. Column names are lowercased. Manual override available for DevRev field names.
5. **Data extraction**: Rows extracted in batches of 1,000 via SQL queries, pushed to DevRev in batches of 2,000 via Airdrop SDK

### Architecture Diagram

```
Snowflake Warehouse
    |
    | SQL queries (batched, 1000 rows/query)
    v
[Extractor: airdrop-snowflake-snap-in]
    |
    | snowflake-sdk v2.3.4
    | @devrev/ts-adaas v1.14.1
    |
    | adapter.getRepo(tableName).push(items)  (2000 rows/push)
    v
[DevRev Airdrop Platform]
    |
    | Transformer (type mapping, field normalization)
    v
[DevRev Custom Objects]
    |
    +-- Searchable via AI agents
    +-- Available in vistas, dashboards
    +-- Usable in workflow automations
```

### Mapping Rules

- **Table-to-custom-object mapping**: Each Snowflake table becomes one DevRev custom object type. The type name derives from the table name.
- **Row-to-item mapping**: Each row becomes one custom object item. Primary key is required per table; extraction fails with an error if no PK exists.
- **ID generation**: For single-column primary keys, the item ID is `{PK_VALUE}`. For composite primary keys, the ID is `{PK1_ENCODED}~{PK2_ENCODED}~...` where values are URL-encoded and sorted alphabetically by column name.
- **Field-to-field mapping**: Columns map to DevRev fields using the type conversion table. Unknown types fall back to `text`.
- **Lookup linking**: Not applicable in V1 (no foreign key resolution across tables).

### Incremental Sync via Snowflake Streams

- On first sync, the snap-in creates a Snowflake Stream for each selected table, named `{DB}_{SCHEMA}_{TABLE}_stream`
- Subsequent syncs query the stream for changed rows (inserts, updates, deletes)
- If a stream becomes stale (Snowflake retention exceeded), the snap-in detects this and falls back to a full table scan automatically
- Sync mode is configurable per table: `FULL` (re-extract all rows every sync) or `INCREMENTAL` (use streams)

## Functional Requirements

- Support secure authentication via Key-Pair (JWT) or OAuth Client Credentials
- Support table-level sync unit selection across all accessible databases and schemas
- Automatically discover table schemas using `DESC TABLE`
- Map Snowflake column types to DevRev field types per the conversion table
- Extract rows in configurable batches (default: 1,000 per query, 2,000 per push)
- Support full and incremental sync modes per table
- Auto-create and manage Snowflake Streams for incremental sync
- Detect stale streams and fall back to full extraction
- Enforce primary key requirement per table with clear error messaging
- Generate unique item IDs from primary key values (single and composite)
- Generate stock `title` and `item_url_field` for every synced item
- Ability to search synced Snowflake data via DevRev AI agents
- Ability to use synced data in DevRev workflows (triggers, conditions, actions)
- Ability to view synced data in DevRev vistas and dashboards

## Non-Functional Requirements

- **Security**: Private keys and OAuth secrets stored in DevRev keyrings, never logged or exposed in error messages. PEM keys auto-normalized to prevent formatting issues.
- **Performance**: Handle tables with 100,000+ rows. Target: initial full sync under 60 minutes for 100,000 rows. Incremental sync under 5 minutes for 5,000 changed rows.
- **Rate limiting**: Automatic retry with 60-second delays when Snowflake returns rate limit errors.
- **Scalability**: Support syncing up to 50 tables per workspace. Batch sizes tuned to avoid Snowflake query timeout limits.
- **Credential handling**: Encrypted private keys supported. PEM auto-normalization handles malformed line breaks from copy-paste.

## Constraints

- One-way only: Snowflake to DevRev. No write-back or reverse sync.
- Physical tables only. Views, external tables, and materialized views are not supported.
- Primary key required on every synced table. Tables without a primary key cannot be imported.
- Column names are lowercased during import. Columns differing only by case will collide.
- Schema changes in Snowflake after initial import require manual field remapping in DevRev.
- The authenticated user/role must have `SELECT` permission on all target tables.
- System databases (`SNOWFLAKE`, `SNOWFLAKE_SAMPLE_DATA`, `SNOWFLAKE_LEARNING_DB`) are excluded from discovery.
- **SDK dependencies**: `@devrev/ts-adaas` v1.14.1, `snowflake-sdk` v2.3.4
- **Package slug**: `airdrop-snowflake-snap-in`
- **Marketplace slug**: `snowflake-airdrop`

## State Management

The snap-in tracks sync state at two levels:

### Per-Table State
| Field | Description |
|-------|-------------|
| `completed` | Whether extraction for this table finished |
| `extractedCount` | Total rows extracted so far |
| `lastProcessedId` | Last primary key value processed (for resumption) |
| `lastProcessedOffset` | Last row offset (for batch pagination) |

### Global State
| Field | Description |
|-------|-------------|
| `lastSyncStarted` | Timestamp of the most recent sync attempt |
| `lastSuccessfulSyncStarted` | Timestamp of the last sync that completed without error |

## Success Criteria

- User can authenticate a Snowflake connection using either Key-Pair or OAuth credentials
- User can browse and select tables from accessible databases and schemas
- Selected tables appear as custom object types in DevRev with correct field types
- Rows appear as custom object items with accurate data, title, and back-link URL
- Incremental sync imports only changed rows via Snowflake Streams
- Stale stream detection triggers automatic fallback to full sync
- Tables without primary keys produce a clear, actionable error
- Composite primary keys generate stable, deterministic item IDs
- Synced data is searchable via DevRev AI search
- Synced data is visible in DevRev vistas and usable in workflows

## Release Plan

**V1** (single release): One-way Snowflake to DevRev sync. Key-Pair and OAuth authentication. Full and incremental sync. Schema discovery. Type mapping. Stream-based change detection.

**V2** (future consideration):
- Foreign key resolution across synced tables (lookup fields)
- View and materialized view support
- Role-to-group permission mapping
- Configurable sync schedules
- Column-level inclusion/exclusion filters

## Open Questions

- [ ] Should the snap-in support syncing Snowflake Secure Views in V2?
- [ ] How should VARIANT columns with deeply nested JSON be rendered in DevRev — flatten to top-level keys or store as raw struct?
- [ ] Should there be a configurable row limit per table to prevent runaway syncs on very large tables?
- [ ] Should column-level comments from Snowflake be imported as field descriptions in DevRev?

## References

- Snowflake Key-Pair Authentication: https://docs.snowflake.com/en/developer-guide/sql-api/authenticating
- Snowflake OAuth: https://docs.snowflake.com/en/user-guide/oauth-custom
- Snowflake Streams: https://docs.snowflake.com/en/user-guide/streams-intro
- DevRev ADaaS SDK: https://developer.devrev.ai/snap-in-development/adaas
- DevRev Airdrop: https://developer.devrev.ai/snap-in-development/adaas/airdrop
