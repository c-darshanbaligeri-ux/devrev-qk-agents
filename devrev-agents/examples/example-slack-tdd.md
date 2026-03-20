# Technical Design Document (TDD) for Slack ADaaS Integration with DevRev

> **This is a realistic example** demonstrating the TDD format used by DevRev teams, based on the actual Slack ADaaS snap-in codebase. It follows the same structure, section ordering, and depth as real DevRev TDDs.

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
| **Solution name** | Slack AirSync Connector for DevRev |
| **Package slug** | airdrop-slack-snap-in |
| **Marketplace slug** | slack-airsync |
| **Snap-in type** | AirSync (ADaaS) — one-way (Slack to DevRev) |
| **Author** | DevRev Engineering |
| **Reviewers** | Pending |
| **Status** | In progress |
| **Created date** | March 2026 |

## 2. Introduction

This document outlines the technical specifications for integrating Slack public channel conversations, messages, threads, and user data into DevRev via the AirSync (ADaaS) platform. The integration enables teams to import Slack communication history into DevRev as searchable conversations and comments, preserving threading structure, user attribution, mentions, and file attachments. Sync is one-way (Slack to DevRev) with incremental support using a rolling 30-day window.

---

## 3. Requirements

### 3.1 Functional Requirements

System must:
- Import Slack public channels as DevRev conversations (DM objects)
- Import channel messages as DevRev comments, preserving sender, timestamp, and rich text
- Import message threads (replies) as nested DevRev comments linked to the parent message
- Import Slack users as DevRev dev users, resolving by email when available
- Import channel membership as DevRev group members
- Optionally import file attachments as DevRev artifacts (configurable, default off)
- Support dual authentication: OAuth user token (recommended) or Bot token (fallback)
- Convert Slack mention syntax (`<@USER_ID>`) to DevRev mention format
- Convert Slack float timestamps (`1234567890.123456`) to ISO 8601
- Support incremental sync using `lastSuccessfulSyncStarted` with a 30-day lookback window
- Respect per-method Slack API rate limits with adaptive throttling

### 3.2 Non-functional Requirements

System shall be:
- **Secure**: OAuth tokens stored in DevRev keyrings, never logged. Token verification via `GET https://slack.com/api/conversations.list` with Bearer auth before extraction begins
- **Scalable**: Handle workspaces with thousands of channels and hundreds of thousands of messages via cursor-based pagination and batched processing
- **Reliable**: Persist per-entity extraction progress (dm, messages, threads, users cursors) so extraction can resume after Lambda timeout (soft limit: 180 seconds)
- **Performant**: Serial request execution (p-limit=1) with per-method throttle delays to stay within Slack rate limits while maximizing throughput

**Data volume**: High — enterprise Slack workspaces can have 10,000+ channels with millions of messages

### 3.3 Constraints

- **Technical**: Integration uses `@slack/web-api` v7.9.2 via REST. Slack does not expose a GraphQL API. All methods are cursor-paginated.
- **Rate limits**: Slack enforces per-method rate limits (Tier 1-4). The extractor must implement per-method delays — see Section 6 for exact values.
- **Bot access**: Bot must be explicitly `/invite`-d to each channel before messages are accessible. Channels the bot has not joined will be listed but messages cannot be extracted.
- **External**: Requires `@devrev/ts-adaas` v1.13.1 for ADaaS SDK integration. Also depends on `axios`, `p-limit`, and `js-jsonl`.

---

## 4. Sequence Diagram

**Actors**: User, Airdrop Component, Extractor, Slack API

```
User                  Airdrop              Extractor            Slack API
 │                      │                     │                     │
 │── Select Import ────▶│                     │                     │
 │                      │── Sync Unit Start ─▶│                     │
 │                      │                     │── auth.test ───────▶│
 │                      │                     │◀── Verify token ────│
 │                      │                     │── conversations.list ▶│
 │                      │                     │  (cursor-paginated,   │
 │                      │                     │   page_size=10)       │
 │                      │                     │◀── Public channels ──│
 │                      │◀── Sync Units Done ─│                     │
 │◀── List of Channels ─│                     │                     │
 │── Select Channel(s) ▶│                     │                     │
 │                      │── Metadata Start ──▶│                     │
 │                      │                     │── Build metadata.json│
 │                      │                     │   (channel, message,  │
 │                      │                     │    thread, user,      │
 │                      │                     │    attachment schemas) │
 │                      │◀── Metadata Done ───│                     │
 │                      │── Data Start ──────▶│                     │
 │                      │                     │                     │
 │                      │                     │── users.list ───────▶│
 │                      │                     │  (cursor-paginated,   │
 │                      │                     │   page_size=999,      │
 │                      │                     │   delay=6000ms)       │
 │                      │                     │◀── Workspace users ──│
 │                      │                     │── Format Users ──────│
 │                      │                     │   (email fallback:    │
 │                      │                     │    name@shadowdevuser │
 │                      │                     │    .invalid)          │
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── conversations.members▶│
 │                      │                     │  (per channel,        │
 │                      │                     │   delay=1000ms)       │
 │                      │                     │◀── Member IDs ───────│
 │                      │                     │── Format Members ────│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── conversations.history▶│
 │                      │                     │  (per channel,        │
 │                      │                     │   page_size=999,      │
 │                      │                     │   delay=3000ms,       │
 │                      │                     │   oldest=30d ago)     │
 │                      │                     │◀── Channel messages ─│
 │                      │                     │── Batch 100 messages ─│
 │                      │                     │── Convert mentions ──│
 │                      │                     │── Convert timestamps ─│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │                     │── conversations.replies▶│
 │                      │                     │  (per thread parent,  │
 │                      │                     │   page_size=999,      │
 │                      │                     │   delay=3000ms)       │
 │                      │                     │◀── Thread replies ───│
 │                      │                     │── Format Threads ────│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │                     │
 │                      │◀── Data Done ───────│                     │
 │◀── Extracted Data ──│                     │                     │
 │── Map Fields ───────▶│                     │                     │
 │                      │── Transform & Load ─│                     │
 │                      │                     │── [If attachments    │
 │                      │                     │    enabled]:          │
 │                      │                     │── Download via       │
 │                      │                     │   url_private_download│
 │                      │                     │   with Bearer token  │
 │                      │                     │── Upload as Artifacts│
 │◀── Import Complete ─│                     │                     │
```

**Key flow notes:**
- Token is verified at Sync Unit Start via `auth.test` (500ms delay)
- Channels are listed with `page_size=10` due to Slack's Tier 2 limit on `conversations.list` (4000ms delay)
- Users are extracted first so mention resolution (`<@U12345>` to DevRev user) is available during message processing
- Messages and threads are processed per-channel sequentially (p-limit=1)
- State checkpoints are saved after each entity type so extraction can resume after Lambda soft timeout (180s)
- Incremental sync filters messages using `oldest` parameter set to `lastSuccessfulSyncStarted` minus 30 days

---

## 5. Data Mapping

### Slack Users → DevRev Dev Users (devu)

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| id | Id | External reference |
| name | display_name | Fallback if real_name is empty |
| real_name | display_name | Preferred over name |
| profile.email | email | Primary key for user matching |
| — | email (fallback) | `${name}@shadowdevuser.invalid` when profile.email is absent |

### Public Channels → DevRev Conversations (DM)

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| id | Id | External reference |
| name | title | Direct map |
| creator | created_by_id | Resolved to DevRev user via user mapping |
| member_ids | user_ids | Resolved via `conversations.members` |
| purpose.value / topic.value | description | Optional context |
| — | url (item_url_field) | Constructed: `https://app.slack.com/client/{team_id}/{channel_id}` |

### Channel Messages → DevRev Comments

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| ts | Id | Slack timestamp as unique ID (e.g., `1234567890.123456`) |
| text | body | Rich text with mention conversion (`<@U123>` to DevRev format) |
| user | created_by_id | Resolved to DevRev user |
| channel (id) | parent_object_id | Links comment to parent DM conversation |
| — | parent_object_type | = "dm" |
| ts | created_date | Converted: Slack float to ISO 8601 |
| edited.ts | modified_date | If present; converted to ISO 8601 |

### Message Threads (Replies) → DevRev Comments (nested)

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| ts | Id | Reply timestamp as unique ID |
| text | body | Rich text with mention conversion |
| user | created_by_id | Resolved to DevRev user |
| thread_ts (parent message) | parent_object_id | Links reply to parent message comment |
| channel (id) | grandparent_object_id | Links reply chain back to channel |
| — | parent_object_type | = "comment" |
| ts | created_date | Converted to ISO 8601 |

### Channel Members → DevRev Group Members

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| user_id | member_id | Resolved to DevRev user |
| channel_id | group_id | Links to parent DM conversation |

### File Attachments → DevRev Artifacts (optional)

| Slack Field | DevRev Field | Notes |
|-------------|-------------|-------|
| id | Id | External reference |
| name | file_name | Direct map |
| url_private_download | url | Requires Bearer token to download |
| user | author_id | Resolved to DevRev user |
| mimetype | mime_type | Preserve original |
| size | — | Validate within DevRev artifact size limits |

**Attachment extraction is configurable (default: false).** When enabled, files are downloaded via `url_private_download` using the OAuth/Bot token as Bearer auth and re-uploaded to DevRev as artifacts linked to the parent message.

### Mention Transformation

Slack stores mentions as `<@U0123ABCDEF>`. During message body processing, these are:
1. Looked up against the extracted user map
2. Replaced with the corresponding DevRev mention format
3. If the user ID is not found in the map, the raw mention is preserved as plain text

### Timestamp Conversion

Slack timestamps are floating-point strings (e.g., `"1678901234.567890"`) representing Unix epoch seconds with microsecond precision. These are converted to ISO 8601 format (e.g., `2023-03-15T18:27:14.567Z`) for all DevRev date fields.

---

## 6. Endpoints

### 6.1 DevRev API Endpoints

| Endpoint | Description |
|----------|-------------|
| `@devrev/ts-adaas` v1.13.1 | ADaaS SDK — extraction, batching, state management, artifact upload |
| Chef CLI (latest) | `chef-cli configure-mappings` for domain mapping, `chef-cli validate-metadata` for schema validation |

### 6.2 External API Endpoints

**Authentication setup (OAuth — recommended):**
1. Go to https://api.slack.com/apps and create a new Slack App
2. Under **OAuth & Permissions**, add the required Bot Token Scopes (see table below)
3. Install the app to your workspace
4. Copy the **Bot User OAuth Token** (`xoxb-...`) or **User OAuth Token** (`xoxp-...`)
5. In DevRev, paste the token into the snap-in connection keyring
6. The snap-in verifies the token at sync start via `GET https://slack.com/api/conversations.list` with `Authorization: Bearer {token}`

**Authentication setup (Bot Token — fallback):**
1. Use the same Slack App from above
2. Copy the Bot User OAuth Token (`xoxb-...`)
3. Note: Bot tokens require the bot to be `/invite`-d to each channel before messages are accessible

**Required OAuth Scopes:**

| Scope | Description |
|-------|-------------|
| channels:history | View messages and content in public channels |
| channels:read | View basic information about public channels |
| files:read | View files shared in channels and conversations |
| groups:history | View messages and content in private channels (if authorized) |
| groups:read | View basic information about private channels |
| im:history | View messages in direct messages |
| im:read | View basic information about direct messages |
| mpim:history | View messages in group direct messages |
| mpim:read | View basic information about group direct messages |
| users:read | View people in the workspace |
| users:read.email | View email addresses of people in the workspace |

**API Endpoint Reference:**

| Slack Entity | API Method | Endpoint | Rate Limit Delay | Page Size | Notes |
|-------------|-----------|----------|-----------------|-----------|-------|
| Auth verification | auth.test | `https://slack.com/api/auth.test` | 500ms | — | Verify token validity |
| Channels | conversations.list | `https://slack.com/api/conversations.list` | 4000ms | 10 | `types=public_channel`, cursor-paginated via `response_metadata.next_cursor` |
| Channel Members | conversations.members | `https://slack.com/api/conversations.members` | 1000ms | — | Per-channel call, cursor-paginated |
| Messages | conversations.history | `https://slack.com/api/conversations.history` | 3000ms | 999 | Per-channel, `oldest` param for incremental, cursor-paginated |
| Thread Replies | conversations.replies | `https://slack.com/api/conversations.replies` | 3000ms | 999 | Per-thread-parent, `ts` param required, cursor-paginated |
| Users | users.list | `https://slack.com/api/users.list` | 6000ms | 999 | Workspace-wide, cursor-paginated |
| User Detail | users.info | `https://slack.com/api/users.info` | 1000ms | — | Single user lookup (fallback for missing data) |

**Pagination model:**
All list endpoints use cursor-based pagination. The response includes `response_metadata.next_cursor` — when non-empty, pass it as `cursor` in the next request. All delays listed above are enforced between consecutive requests to the same method via adaptive throttling (p-limit with serial execution, limit=1).

**Token verification:**
Before extraction begins, the extractor calls `GET https://slack.com/api/conversations.list` with `Authorization: Bearer {token}`. A successful response confirms the token is valid and has the required scopes. A `401` or `invalid_auth` error aborts the sync with a user-facing error.

---

## 7. Conclusion

### 7.1 Risks Assumed

- Slack API rate limits are per-method and per-workspace. Workspaces with high concurrent API usage from other apps may experience slower extraction if Slack returns `429 Too Many Requests` responses
- Bot tokens require explicit `/invite` to each channel. Channels the bot has not joined will appear in the channel list but message extraction will fail silently (empty results)
- Users without a `profile.email` (e.g., bot accounts, deactivated users) receive a `shadowdevuser.invalid` fallback email and cannot be matched to real DevRev users
- Lambda soft timeout at 180 seconds means very large channels (100k+ messages) may require multiple sync cycles to fully extract

### 7.2 Trade-offs

- Serial request execution (p-limit=1) sacrifices throughput for rate-limit safety. Parallel extraction would be faster but risks hitting Slack's per-app rate limits
- Channel page size of 10 is conservative but avoids Tier 2 rate limit penalties on `conversations.list`
- Attachment extraction is disabled by default to keep initial sync fast and storage costs low
- 30-day incremental window balances completeness against extraction time. Older messages require a full re-sync

### 7.3 Limitations

- Only public channels are imported in V1. Private channels, DMs, and group DMs require additional user consent flows
- No reverse sync. Changes in DevRev are not pushed back to Slack
- Slack message formatting (blocks, Block Kit elements) is simplified to rich text. Complex interactive elements (buttons, modals) are not preserved
- Emoji reactions are not imported (would require per-message `reactions.get` calls)
- Slack Connect (shared channels with external organizations) is not supported
- File attachments larger than DevRev's artifact size limit are skipped
- Deleted messages are not detected during incremental sync (Slack does not surface deletions in `conversations.history`)

### 7.4 Future Expansion

- Add private channel and DM import with user-consent approval flow
- Implement Slack Events API webhook listener for near-real-time sync (requires public callback URL)
- Add emoji reaction import as comment metadata
- Add reverse sync: DevRev comment replies posted back to Slack threads
- Support Slack Connect shared channels for cross-org conversations
- Add message edit and deletion detection using Slack's `conversations.history` with `inclusive=true` and comparing against previously synced data
- Implement parallel extraction with per-method rate limit queues for faster sync on large workspaces
