# Technical Design Document (TDD) for Monday.com Integration with DevRev

> **This is a realistic example** demonstrating the TDD format used by DevRev teams, based on the actual Monday.com ADaaS snap-in codebase. It follows the same structure, section ordering, and depth as real DevRev TDDs.

## Index

1. Overview
2. Introduction
3. Requirements
4. Sequence Diagram
5. Data Mapping
6. Endpoints
7. Conclusion

---

## 1. Overview

| Field | Value |
|-------|-------|
| **Solution name** | Monday.com AirSync Connector for DevRev |
| **Author** | Example Author |
| **Reviewers** | Pending |
| **Status** | In progress |
| **Created date** | March 2026 |

## 2. Introduction

This document outlines the technical specifications for integrating Monday.com boards, items, and associated data into DevRev via the AirSync (ADaaS) platform. The integration is a unidirectional connector (Monday.com to DevRev) that syncs eight object types — items, issues, users, workspaces, tags, updates, replies, and attachments — into DevRev as issues, dev users, products, tags, comments, and attachments. It supports time-scoped incremental syncs and handles Monday.com's GraphQL API complexity budget, IP rate limits, and concurrency limits with structured retry and delay logic. A reverse sync stub is included for future bidirectional support.

---

## 3. Requirements

### 3.1 Functional Requirements

System must:
- Import Monday.com Items as DevRev issues (`ext_object_type="Items"`) with P0 priority
- Import Monday.com Issues as DevRev issues (`ext_object_type="Issues"`) with P1 priority
- Import Monday.com Users as DevRev dev users (`devu`)
- Import Monday.com Workspaces as DevRev products (`ext_object_type="Workspaces"`)
- Import Monday.com Tags as DevRev tags
- Import Monday.com Updates (item comments) as DevRev comments
- Import Monday.com Replies (threaded comments) as nested DevRev comments
- Import Monday.com Attachments (files from updates) as DevRev attachments
- Handle 20 Monday.com column types: text, long-text, status, person, multiple-person, date, numbers, checkbox, rating, dropdown, tags, subtasks, country, phone, email, link, doc, timeline, item_id, last_updated, hour
- Authenticate via OAuth 2.0 with Admin and Member connection types
- Support time-scoped incremental syncs using `lastSuccessfulSyncStarted`
- Handle Monday.com's complexity-based rate limiting with appropriate delay intervals
- Implement cursor-based pagination for items and page-based pagination for other entities

### 3.2 Non-functional Requirements

System shall be:
- **Secure**: OAuth tokens stored in DevRev keyrings; two connection types enforce least-privilege access (Member connections have limited edit scope)
- **Scalable**: Batch processing up to 500 items/comments/users per batch, 100 workspaces per batch; handles large Monday.com accounts with thousands of boards
- **Reliable**: State persistence across extraction phases; exponential backoff with jitter on failures; max 3 retries per request
- **Performant**: API page size of 100; concurrent extraction of independent object types; 30-second timeout per API call

**Data volume**: Medium to Large — typical Monday.com accounts have 1,000-50,000 items across multiple workspaces

### 3.3 Constraints

- **Technical**: Integration uses Monday.com GraphQL API v2 (version header `2025-10`). REST API not available for core entities. Complexity budget governs query cost, not simple request count.
- **Rate limits**: Four distinct rate limit error types, each requiring different delay durations (60s-300s). 429 responses must emit delay events rather than auto-retry.
- **External**: Depends on `@devrev/ts-adaas` v1.16.0 SDK. Monday.com API versioning may deprecate queries without notice.
- **Sync direction**: Unidirectional (Monday.com to DevRev) in V1. Reverse sync stub exists but is not implemented.

---

## 4. Sequence Diagram

**Actors**: User, Airdrop Component, Extractor, Monday.com GraphQL API

```
User                  Airdrop              Extractor            Monday.com API
 │                      │                     │                     │
 │── Select Import ────▶│                     │                     │
 │                      │── Sync Unit Start ─▶│                     │
 │                      │                     │── query { boards    │
 │                      │                     │     { id name       │
 │                      │                     │       workspace     │
 │                      │                     │       { id } } } ──▶│
 │                      │                     │◀── List of boards ──│
 │                      │◀── Sync Units Done ─│                     │
 │◀── List of Boards ──│                     │                     │
 │── Select Board(s) ──▶│                     │                     │
 │                      │── Metadata Start ──▶│                     │
 │                      │                     │── query { boards    │
 │                      │                     │     (ids:[...])     │
 │                      │                     │     { columns       │
 │                      │                     │       { id title    │
 │                      │                     │         type } } }─▶│
 │                      │                     │◀── Column schema ───│
 │                      │                     │── Upload metadata   │
 │                      │◀── Metadata Done ───│                     │
 │                      │── Data Start ──────▶│                     │
 │                      │                     │                     │
 │                      │                     │── query { users     │
 │                      │                     │     (page:1,        │
 │                      │                     │      limit:100)     │
 │                      │                     │     { id name       │
 │                      │                     │       email } } ───▶│
 │                      │                     │◀── Users page ──────│
 │                      │                     │── Format Users ─────│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── query { workspaces│
 │                      │                     │     { id name       │
 │                      │                     │       kind } } ────▶│
 │                      │                     │◀── Workspaces ──────│
 │                      │                     │── Format Workspaces ─│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── query { boards    │
 │                      │                     │     (ids:[...])     │
 │                      │                     │     { items_page    │
 │                      │                     │       (limit:100,   │
 │                      │                     │        cursor:...)  │
 │                      │                     │       { cursor      │
 │                      │                     │         items       │
 │                      │                     │         { id name   │
 │                      │                     │           column_   │
 │                      │                     │           values    │
 │                      │                     │           { id text │
 │                      │                     │             value   │
 │                      │                     │           } } } }}─▶│
 │                      │                     │◀── Items page ──────│
 │                      │                     │── (repeat with      │
 │                      │                     │    next cursor      │
 │                      │                     │    until null) ─────│
 │                      │                     │── Format Items ─────│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── query { boards    │
 │                      │                     │     (ids:[...])     │
 │                      │                     │     { items_page    │
 │                      │                     │       { items       │
 │                      │                     │         { updates   │
 │                      │                     │           { id body │
 │                      │                     │             replies │
 │                      │                     │             { id    │
 │                      │                     │               body  │
 │                      │                     │           } } } } }▶│
 │                      │                     │◀── Updates/Replies ─│
 │                      │                     │── Format Comments ──│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │◀── Data Done ───────│                     │
 │◀── Extracted Data ──│                     │                     │
 │── Map Fields ───────▶│                     │                     │
 │                      │── Transform & Load ─│                     │
 │                      │                     │── query { assets    │
 │                      │                     │     (ids:[...])     │
 │                      │                     │     { id public_url │
 │                      │                     │       name } } ────▶│
 │                      │                     │◀── Asset URLs ──────│
 │                      │                     │── Download & upload ─│
 │◀── Import Complete ─│                     │                     │
```

---

## 5. Data Mapping

### Users → DevRev Dev Users

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference |
| name | Display Name | Direct map |
| email | Email | Used for user matching/dedup |
| phone | phone_numbers | If available |
| photo_thumb | — | Not mapped |
| title | — | Not mapped |
| is_admin | — | Not mapped (used for connection type selection) |

### Workspaces → DevRev Products

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference; `ext_object_type="Workspaces"` |
| name | name | Direct map |
| kind | — | Informational only (open/closed) |
| description | description | Direct map |

### Items → DevRev Issues (P0)

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference; `ext_object_type="Items"` |
| name | title | Direct map |
| column_values[long_text].content | body | Mapped as `rich_text` type |
| creator.id | created_by_id | User reference |
| column_values[person/multiple_person] | owned_by_ids | Array of user references |
| board.workspace.id | applies_to_part_id | Product reference (workspace) |
| column_values[tags] | tags | Array of tag references |
| column_values[date] | target_start_date | Date value |
| — (fixed) | status | Always `in_development` |
| — (fixed) | priority | Always `P0` |
| created_at | created_date | Timestamp |
| updated_at | modified_date | Timestamp; used for time-scoped sync |
| column_values[link] | item_url_field | Link back to Monday.com item |

### Issues → DevRev Issues (P1)

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference; `ext_object_type="Issues"` |
| name | title | Direct map |
| column_values[long_text].content | body | Mapped as `rich_text` type |
| creator.id | created_by_id | User reference |
| column_values[person/multiple_person] | owned_by_ids | Array of user references |
| board.workspace.id | applies_to_part_id | Product reference (workspace) |
| column_values[tags] | tags | Array of tag references |
| column_values[date] | target_start_date | Date value |
| — (fixed) | status | Always `in_development` |
| — (fixed) | priority | Always `P1` |
| created_at | created_date | Timestamp |
| updated_at | modified_date | Timestamp |

### Tags → DevRev Tags

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference |
| name | name | Direct map |

### Updates → DevRev Comments

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference |
| body | body | HTML content from Monday.com |
| creator.id | created_by_id | User reference |
| item_id | parent_object_id | Reference to parent item/issue |
| — | parent_object_type | `"issue"` |
| created_at | created_date | Timestamp |
| updated_at | modified_date | Timestamp |

### Replies → DevRev Comments (Nested)

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference |
| body | body | HTML content |
| creator.id | created_by_id | User reference |
| parent_update_id | parent_object_id | Reference to parent comment (update) |
| — | parent_object_type | `"comment"` |
| created_at | created_date | Timestamp |

### Attachments → DevRev Attachments

| Monday.com Field | DevRev Field | Notes |
|-----------------|-------------|-------|
| id | Id | External reference |
| name | file_name | Direct map |
| public_url | download URL | Download and re-upload to DevRev |
| file_extension | mime_type | Mapped from extension |
| file_size | — | Validated against max artifact size |
| created_at | created_date | Timestamp |

### Column Type Inventory

**Available column types in Monday.com (40+):**
Monday.com supports a wide range of column types across boards. The connector handles the 20 most commonly used types.

**Imported in V1 (20 types):**
text, long-text, status, person, multiple-person, date, numbers, checkbox, rating, dropdown, tags, subtasks, country, phone, email, link, doc, timeline, item_id, last_updated, hour

**Not supported in V1 (examples):**
- `formula` — computed values, no stable external representation
- `mirror` — references data from linked boards, would require cross-board extraction
- `dependency` — board-internal relationships, no DevRev equivalent
- `color_picker` — cosmetic, no DevRev field
- `location` — no DevRev geospatial field
- `world_clock` — timezone display, no DevRev equivalent
- `button` — interactive element, not data
- `files` — handled separately via Attachments extraction
- `board_relation` — cross-board links, deferred to V2

---

## 6. Endpoints

### 6.1 DevRev API Endpoints

| Endpoint | Description |
|----------|-------------|
| @devrev/ts-adaas v1.16.0 | ADaaS SDK for Airdrop extraction, batching, state management |
| Chef CLI (latest) | Domain mapping (`chef-cli configure-mappings`) and metadata validation (`chef-cli validate-metadata`) |

### 6.2 External API Endpoints

**Authentication setup:**
1. Go to https://monday.com/developers/apps and create a new app
2. Under "OAuth & Permissions", configure redirect URI to DevRev's callback URL
3. Add required OAuth scopes (see table below)
4. Note the Client ID and Client Secret
5. Configure two connection types in DevRev:
   - **Admin connection**: Full read/write access for initial import and sync
   - **Member connection**: Limited edit permissions for field-level updates

**OAuth 2.0 flow:**
- **Authorization URL**: `https://auth.monday.com/oauth2/authorize`
- **Token URL**: `https://auth.monday.com/oauth2/token`
- **Grant type**: `authorization_code` with `refresh_token` support

**Required OAuth Scopes:**

| Scope | Description |
|-------|-------------|
| me:read | Read authenticated user profile |
| boards:read | Read board structure, columns, and items |
| boards:write | Write board data (reserved for reverse sync) |
| workspaces:read | Read workspace list and metadata |
| workspaces:write | Write workspace data (reserved for reverse sync) |
| users:read | Read user profiles and emails |
| users:write | Write user data (reserved for reverse sync) |
| updates:read | Read item comments (updates) and replies |
| updates:write | Write comments (reserved for reverse sync) |
| account:read | Read account-level metadata |
| assets:read | Read file attachments from updates |
| tags:read | Read tag definitions |

**GraphQL API Endpoint:**

All queries go to a single endpoint:
```
POST https://api.monday.com/v2
Headers:
  Authorization: Bearer {access_token}
  API-Version: 2025-10
  Content-Type: application/json
Timeout: 30 seconds
```

**Key GraphQL Queries:**

| Entity | Query | Pagination | Notes |
|--------|-------|-----------|-------|
| Boards | `{ boards { id name workspace { id } columns { id title type } } }` | — | Returns all accessible boards |
| Items | `{ boards(ids:[...]) { items_page(limit:100, cursor:$cursor) { cursor items { id name column_values { id text value type } creator { id } } } } }` | Cursor-based (`next_items_page`) | Primary data extraction |
| Users | `{ users(page:$page, limit:100) { id name email phone } }` | Page-based offset | All account users |
| Workspaces | `{ workspaces { id name kind description } }` | Page-based offset | All accessible workspaces |
| Updates | `{ boards(ids:[...]) { items_page { items { updates(page:$page, limit:100) { id body creator { id } created_at updated_at } } } } }` | Page-based | Item comments |
| Replies | Nested within Updates query: `replies { id body creator { id } created_at }` | Page-based | Threaded comments |
| Assets | `{ assets(ids:[...]) { id name public_url file_extension file_size created_at } }` | Page-based | File attachments |
| Tags | `{ tags { id name } }` | — | All account tags |

**Rate Limiting Strategy:**

| Error Code | Delay Duration | Description |
|-----------|---------------|-------------|
| COMPLEXITY_BUDGET_EXHAUSTED | 120 seconds | Query complexity exceeded budget; wait for reset |
| IP_RATE_LIMIT_EXCEEDED | 300 seconds | IP-level throttle; longest wait |
| MAX_CONCURRENCY_EXCEEDED | 60 seconds | Too many concurrent requests |
| DEFAULT (other errors) | 100 seconds | Catch-all delay |

**Retry strategy:**
- Max retries: 3
- Backoff: Exponential — `1000ms * 2^attempt + random jitter`
- Max backoff cap: 30 seconds
- 429 errors: Emit `ExtractionDataDelay` event with appropriate delay; do not auto-retry
- 5xx errors: Always retry with exponential backoff

**Batch sizes:**

| Entity | Batch Size |
|--------|-----------|
| Items | 500 |
| Comments (Updates/Replies) | 500 |
| Users | 500 |
| Workspaces | 100 |
| API page size | 100 |

---

## 7. Conclusion

### 7.1 Risks Assumed

- Monday.com's complexity-based rate limiting is unpredictable — deeply nested queries (items with column values, updates, and replies) consume more budget than simple queries, making extraction time hard to estimate
- Monday.com API versioning (`2025-10`) may deprecate query structures; the connector must be updated when API versions expire
- GraphQL schema changes (field renames, removed fields) could silently break extraction without HTTP errors
- Large accounts with 50,000+ items may hit the 10-minute ADaaS task timeout, requiring careful state checkpointing

### 7.2 Trade-offs

- V1 is unidirectional (Monday.com to DevRev only) — a reverse sync stub exists but is not implemented, reducing development scope at the cost of two-way collaboration
- Items and Issues both map to DevRev `issue` type, differentiated only by `ext_object_type` and priority — this simplifies the DevRev schema but loses the Monday.com distinction between board types
- Fixed status (`in_development`) and priority (`P0`/`P1`) values are used rather than mapping Monday.com status columns — this avoids complex per-board status mapping but loses workflow fidelity
- 20 of 40+ column types are supported — covers the most common fields while deferring complex types (formula, mirror, dependency) that require cross-board resolution

### 7.3 Limitations

- No reverse sync — changes in DevRev are not pushed back to Monday.com
- Monday.com status columns are not mapped to DevRev stages (fixed `in_development` for all items)
- Formula, mirror, dependency, and location columns are not imported
- Board-level relationships (`board_relation` columns) are not resolved
- Attachments are only extracted from Updates (comments); inline file columns use the separate `files` column type, which is not supported in V1
- Private boards may not be accessible depending on the OAuth connection type (Admin vs Member)

### 7.4 Future Expansion

- Implement reverse sync using the existing stub: push DevRev issue updates back to Monday.com items via `boards:write` and `updates:write` scopes
- Map Monday.com status column values to DevRev stages per board (requires per-board configuration UI)
- Add support for `formula`, `mirror`, and `board_relation` column types with cross-board resolution
- Implement webhook-based real-time sync using Monday.com's `webhooks` API for instant updates
- Add support for Monday.com automations and integrations metadata
- Support multi-workspace sync in a single import operation
- Add Monday.com sub-items as DevRev sub-issues with parent-child relationships

---

## Appendix: Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| @devrev/ts-adaas | v1.16.0 | ADaaS SDK — extraction, batching, state, events |
| axios | v1.13.5 | HTTP client for GraphQL requests |
| axios-retry | v4.5.0 | Automatic retry with exponential backoff |
| js-jsonl | v1.1.1 | JSONL serialization for batch file uploads |
