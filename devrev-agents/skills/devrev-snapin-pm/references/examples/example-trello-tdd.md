# Technical Design Document (TDD) for Trello Integration with DevRev (Example)

> **This is a fictional example** demonstrating the TDD format used by DevRev teams. It follows the same structure, section ordering, and depth as real DevRev TDDs.

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
| **Solution name** | Trello AirSync Connector for DevRev |
| **Author** | Example Author |
| **Reviewers** | Pending |
| **Status** | In progress |
| **Created date** | March 2025 |

## 2. Introduction

This document outlines the technical specifications for integrating Trello boards and cards into DevRev via the AirSync platform. The integration enables teams to import project data from Trello into DevRev as searchable issues, supporting both initial bulk import and ongoing periodic sync.

---

## 3. Requirements

### 3.1 Functional Requirements

System must:
- Import Trello boards as DevRev parts (capabilities)
- Import Trello cards as DevRev issues, preserving metadata (title, description, due date, labels, members, checklists)
- Import Trello members and map to DevRev dev users by email
- Import card attachments as DevRev artifacts
- Map Trello labels to DevRev tags
- Support incremental sync using card `dateLastActivity` timestamps
- Respect Trello API rate limits (100 requests per 10-second window)

### 3.2 Non-functional Requirements

System shall be:
- **Secure**: API tokens stored in DevRev keyrings, never exposed in logs
- **Scalable**: Handle boards with up to 10,000 cards
- **Reliable**: Resume extraction after timeout using state persistence

**Data volume**: Medium — typical boards have 500-5,000 cards

### 3.3 Constraints

- **Technical**: Integration must use Trello REST API (v1). GraphQL not available.
- **Rate limits**: 100 requests per 10-second window per API token. Extraction must implement backoff.
- **External**: Trello custom fields (Power-Up) require paid account — not supported in V1

---

## 4. Sequence Diagram

**Actors**: User, Airdrop Component, Extractor, Trello API

```
User                  Airdrop              Extractor            Trello API
 │                      │                     │                     │
 │── Select Import ────▶│                     │                     │
 │                      │── Sync Unit Start ─▶│                     │
 │                      │                     │── GET /members/me/boards ──▶│
 │                      │                     │◀── List of boards ──│
 │                      │◀── Sync Units Done ─│                     │
 │◀── List of Boards ──│                     │                     │
 │── Select Board ─────▶│                     │                     │
 │                      │── Metadata Start ──▶│                     │
 │                      │                     │── GET /boards/{id}/lists ──▶│
 │                      │                     │── GET /boards/{id}/labels ─▶│
 │                      │◀── Metadata Done ───│                     │
 │                      │── Data Start ──────▶│                     │
 │                      │                     │── GET /boards/{id}/members ─▶│
 │                      │                     │── Format Users ──────│
 │                      │                     │── Emit DataProgress ─│
 │                      │                     │── GET /boards/{id}/cards (paginated) ─▶│
 │                      │                     │── For each card: GET /cards/{id}/checklists ─▶│
 │                      │                     │── Format Cards ──────│
 │                      │                     │── Emit DataProgress ─│
 │                      │◀── Data Done ───────│                     │
 │◀── Extracted Data ──│                     │                     │
 │── Map Fields ───────▶│                     │                     │
 │                      │── Transform & Load ─│                     │
 │                      │                     │── Extract Attachments│
 │                      │                     │── GET /cards/{id}/attachments ─▶│
 │                      │                     │── Download files ────▶│
 │◀── Import Complete ─│                     │                     │
```

---

## 5. Data Mapping

### Members → DevRev Dev Users

| Trello Field | DevRev Field | Notes |
|-------------|-------------|-------|
| fullName | Display Name | Direct map |
| email | Email | Used for user matching |
| id | Id | External reference |
| username | — | Not mapped (DevRev uses email as primary key) |

### Boards → DevRev Parts

| Trello Field | DevRev Field | Notes |
|-------------|-------------|-------|
| name | name | Direct map |
| desc | description | Direct map |
| id | Id | External reference |
| dateLastActivity | Modified Date | Timestamp |

### Cards → DevRev Issues

| Trello Field | DevRev Field | Notes |
|-------------|-------------|-------|
| name | title | Direct map |
| desc | body | Markdown preserved |
| idList | stage | Mapped via list name → stage value |
| idMembers[] | owned_by | Resolved by member email |
| labels[] | tags | Created if not exists, matched by name |
| due | target_close_date | Nullable |
| dateLastActivity | Modified Date | Timestamp |
| id | Id | External reference |
| shortUrl | item_url_field | Link back to Trello card |
| closed | stage | If true → "closed" stage |

### Card Status Mapping (List → Stage)

| Trello List Name | DevRev Stage | Notes |
|-----------------|-------------|-------|
| To Do | backlog | Default for new items |
| In Progress | in_development | Active work |
| In Review | in_review | Pending review |
| Done | done | Completed |
| (any other) | backlog | Fallback for unknown lists |

### Checklists → Custom Field

| Trello Field | DevRev Field | Notes |
|-------------|-------------|-------|
| name | checklist_name (custom) | Checklist title |
| checkItems[].name | checklist_items (custom) | JSON array of items |
| checkItems[].state | checklist_items (custom) | "complete" or "incomplete" |

### Attachments → DevRev Artifacts

| Trello Field | DevRev Field | Notes |
|-------------|-------------|-------|
| name | file name | Direct map |
| url | download URL | Download and re-upload to DevRev |
| bytes | — | Validate < max artifact size |
| mimeType | mime_type | Preserve original type |

---

## 6. Endpoints

### 6.1 DevRev API Endpoints

| Endpoint | Description |
|----------|-------------|
| ADaaS SDK (latest) | SDK for Airdrop extraction and loading |
| Chef CLI (latest) | Domain mapping and metadata validation |

### 6.2 External API Endpoints

**Authentication setup:**
1. Go to https://trello.com/power-ups/admin
2. Create a new Power-Up (or use API key directly)
3. Generate an API key from https://trello.com/app-key
4. Generate a token by visiting: `https://trello.com/1/authorize?expiration=never&scope=read&response_type=token&key={API_KEY}`
5. Copy the API key and token

**All requests use query parameters for auth:**
```
?key={API_KEY}&token={TOKEN}
```

**API Endpoint Reference:**

| Trello Entity | Endpoint | Method | Scopes/Notes |
|--------------|----------|--------|-------------|
| Boards | `https://api.trello.com/1/members/me/boards` | GET | Lists boards for authenticated user. Filter: `?filter=open` |
| Board details | `https://api.trello.com/1/boards/{id}` | GET | Full board metadata |
| Lists | `https://api.trello.com/1/boards/{id}/lists` | GET | All lists in a board |
| Labels | `https://api.trello.com/1/boards/{id}/labels` | GET | All labels on a board |
| Members | `https://api.trello.com/1/boards/{id}/members` | GET | All board members with email |
| Cards | `https://api.trello.com/1/boards/{id}/cards` | GET | All cards. Params: `?fields=all&limit=1000&since={timestamp}` |
| Card checklists | `https://api.trello.com/1/cards/{id}/checklists` | GET | Checklists for a card |
| Card attachments | `https://api.trello.com/1/cards/{id}/attachments` | GET | Attachments for a card |

**Rate limits:**
- 100 API requests per 10-second window per token
- 429 status returned with `Retry-After` header when exceeded
- Back off using `ExtractionDataDelay` with the `Retry-After` value

---

## 7. Conclusion

### 7.1 Risks Assumed

- Trello API rate limits (100/10s) may make large board imports slow — a 5,000 card board with checklists may require ~100 requests for cards + 5,000 for checklists = ~50 10-second windows = ~8 minutes
- Trello free accounts have limited API access; some endpoints may return partial data
- Member email visibility depends on Trello privacy settings — some users may not resolve

### 7.2 Trade-offs

- V1 imports standard fields only — Trello Power-Up custom fields require paid account investigation
- Checklists stored as JSON custom field rather than native DevRev structure (no native equivalent)
- Card comments not imported in V1 (would require per-card API call, significantly impacting rate limits)

### 7.3 Limitations

- Archived boards and cards excluded from initial import
- Trello card comments (actions) not imported — too many API calls per card
- Trello stickers, Power-Up data, and board backgrounds not imported
- No reverse sync — changes in DevRev not pushed back to Trello

### 7.4 Future Expansion

- Add card comments import (with rate-limit-aware batching)
- Add reverse sync (DevRev issue updates → Trello card updates)
- Support Trello Power-Up custom fields for paid accounts
- Implement real-time sync using Trello webhooks (requires callback URL)
- Add support for importing from multiple boards simultaneously
