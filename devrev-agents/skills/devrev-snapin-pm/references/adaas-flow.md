# ADaaS (Airdrop-as-a-Service) Flow Reference

This document describes the standard ADaaS protocol that all DevRev Airdrop connectors follow. Every TDD for an AirSync connector should map its implementation to these phases.

---

## The ADaaS Pipeline

Every AirSync connector follows this pipeline:

```
Source System → Extractor → Transformer → Loader → DevRev
```

The Airdrop platform orchestrates **sync runs** — directed operations that span multiple invocations of the snap-in. There are two directions:

- **Forward sync**: External system → DevRev (extraction)
- **Reverse sync**: DevRev → External system (loading)

---

## Forward Sync Phases (Extraction)

### Phase 1: External Sync Units Extraction

**Purpose**: Discover what data containers exist in the external system.

**What happens**:
1. Airdrop signals `Extraction Sync Unit Start`
2. Extractor calls external API to list available containers (workspaces, projects, channels, boards, tables)
3. Extractor returns list of sync units with metadata (name, description, item count)
4. Airdrop shows list to user for selection
5. User selects which sync unit(s) to import

**Example sync units by system**:
| System | Sync Unit | API Call |
|--------|-----------|---------|
| Slack | Channels (public + private) | `conversations.list` |
| Monday.com | Workspaces / Boards | GraphQL `workspaces` query |
| Jira | Projects | `/rest/api/3/project` |
| Snowflake | Database / Schema / Table | `SHOW TABLES IN SCHEMA` |
| Planhat | Company segments | `/companies` |

### Phase 2: Metadata Extraction

**Purpose**: Fetch schema information — field definitions, custom fields, statuses, types.

**What happens**:
1. Airdrop signals `Extraction Metadata Start` for selected sync unit(s)
2. Extractor calls external API to get schema/field definitions
3. Extractor generates `metadata.json` describing available record types and fields
4. Extractor emits `metadata done` event
5. Airdrop uses metadata to auto-generate field mapping suggestions

**metadata.json structure**:
```json
{
  "record_types": [
    {
      "name": "task",
      "display_name": "Task",
      "fields": [
        { "name": "title", "type": "string", "required": true },
        { "name": "status", "type": "enum", "values": ["open", "done"] }
      ]
    }
  ]
}
```

### Phase 3: Data Extraction

**Purpose**: Pull actual records from the external system.

**What happens**:
1. Airdrop signals `Extract Data Start Event`
2. Extractor first extracts **users** (needed for mapping authors/assignees)
   - Calls users API (e.g., `users.list`, `users.info`)
   - Formats users and emits `ExtractionDataProgress` event
3. Extractor then extracts **primary data** (the main records)
   - Calls data API with pagination (e.g., `conversations.history`, items query)
   - For nested data: calls detail APIs (e.g., `conversations.replies`, updates query)
   - Formats data and emits `ExtractionDataProgress` event
4. Airdrop shows extracted data to user
5. User maps fields (auto-suggested or manual)

**Pagination handling**:
- Always paginate. Never assume all data fits in one call.
- Use cursor-based pagination where available.
- Respect rate limits — back off on 429 responses.
- Track state for resumability.

**Diff mode (incremental sync)**:
- The DevRev Airdrop platform provides the last sync timestamp to the extractor in the event payload on each periodic sync invocation.
- On initial sync: no timestamp is provided — extract all records.
- On incremental sync: use the platform-provided timestamp to filter the external API for records modified since last sync (e.g., `modified_since`, `updated_after`).
- The snap-in does NOT track its own sync timestamp or manage sync scheduling.

### Phase 4: Attachments Extraction

**Purpose**: Download file attachments associated with records.

**What happens**:
1. Extractor identifies records with attachments
2. Downloads files from external system
3. Uploads to DevRev as artifacts
4. Links artifacts to the corresponding DevRev objects

### Phase 5: Transform & Loading

**Purpose**: Apply field mappings and create DevRev objects.

**What happens**:
1. Airdrop applies the field mapping configuration
2. Transforms external data format to DevRev format
3. Creates/updates DevRev objects via API
4. Signals `Import Completed` to user

---

## Reverse Sync Phases (Loading)

### Phase 1: Load Data

1. Airdrop detects changes in DevRev objects marked for sync
2. Loader function receives changed records
3. Maps DevRev fields back to external system fields
4. Creates or updates records in external system via API

### Phase 2: Load Attachments

1. Downloads new/changed attachments from DevRev
2. Uploads to external system

---

## State Management

The extractor must manage state across invocations:

- **Batch tracking**: Which records have been processed in the current run
- **Cursor/pagination state**: Where to resume if interrupted
- **Error tracking**: Which records failed and need retry

Note: The sync timestamp is managed by the Airdrop platform and provided to the extractor in each event payload. The extractor does NOT need to track last sync time itself.

---

## Error Handling Patterns

| Scenario | Standard Approach |
|----------|-------------------|
| Rate limit (429) | Respect `Retry-After` header, exponential backoff |
| Auth failure (401) | Stop sync, notify admin, don't retry |
| Network timeout | Retry up to 3 times with exponential backoff |
| Missing required field | Log warning, skip record, continue |
| External API down | Pause sync, retry after delay |
| Invalid data format | Log error with record ID, skip, continue |

---

## DevRev ADaaS Tooling

Every TDD should reference these DevRev-specific tools:

| Tool | Purpose | Current Version |
|------|---------|-----------------|
| **ADaaS SDK** | Core SDK for building Airdrop extractors/loaders | Check latest on developer.devrev.ai |
| **Chef CLI** | Domain mapping between external and DevRev objects | Latest |
| **DevRev CLI** | Snap-in build, deploy, activate | Latest |
| **Airdrop Template** | GitHub template for new connectors | github.com/devrev/airdrop-template |

---

## Standard TDD Sequence Diagram Pattern

Every AirSync TDD should include a sequence diagram following this pattern. The actors are always:

```
User | Airdrop Component | Extractor | External API
```

The flow is always:
1. User triggers import
2. Sync unit extraction (list containers)
3. User selects container(s)
4. Metadata extraction
5. Data extraction (users first, then primary data, then nested data)
6. User maps fields
7. Transform and loading
8. Attachment extraction
9. Import complete

Customize the specific API calls per external system, but the orchestration pattern stays the same.

---

## Standard Data Mapping Pattern

Data mapping should cover these entity categories (not all systems have all):

| Category | External Example | DevRev Target |
|----------|-----------------|---------------|
| **Users/People** | Slack users, Monday users | Dev users or Rev users |
| **Containers** | Channels, Boards, Workspaces | Parts, DMs, Custom objects |
| **Primary records** | Messages, Items, Tickets | Conversations, Issues, Tickets, Custom objects |
| **Nested records** | Threads, Replies, Updates | Comments |
| **Attachments** | Files, Images | Artifacts |
| **Metadata** | Tags, Labels, Status values | Tags, Custom fields, Stages |

Each mapping should show:
- The source field name
- The target DevRev field name
- Any transformation (e.g., HTML → Markdown, timestamp format)
- Whether it's a direct copy, lookup, or computed field
