# PRD: Planhat – DevRev Airdrop Integration

> **This is a realistic example** demonstrating the PRD format used by DevRev teams, based on the actual Planhat ADaaS snap-in codebase. It follows the same structure, depth, and patterns as real DevRev PRDs.

## Overview

This document outlines the product requirements for building a bidirectional data synchronization between Planhat and DevRev using the AirSync (ADaaS) platform. The goal is to enable customer success teams using Planhat to unify their account health, conversation, and revenue data with DevRev's support and product workflows, powering AI-driven automation and cross-tool visibility.

## Problem Statement

Customer success managers using Planhat maintain account health scores, track conversations, and manage licenses independently from the support and engineering teams working in DevRev. This creates blind spots: support agents cannot see a customer's health score or renewal status when handling a ticket, and CSMs manually re-enter conversation context between systems. The result is duplicated effort, delayed escalations, and missed renewal signals — teams report spending 20+ minutes per escalation gathering cross-tool context.

## Goal

1. Enable bidirectional sync of customer data between Planhat and DevRev across 10 object types
2. Eliminate manual data duplication between customer success and support/engineering workflows
3. Power DevRev AI agents with Planhat signals (health scores, renewal status, product usage) for smarter ticket routing and prioritization
4. Provide unified, searchable context across both platforms for all customer-facing roles
5. Support incremental time-scoped syncs to keep data current with minimal API overhead

---

## Persona Queries

| Persona | Query | Intent/Action |
|---------|-------|---------------|
| Support Agent | What is the health score and renewal status for the customer on ticket TKT-891? | Prioritize support based on account risk and ARR |
| Customer Success Manager | Show me all open conversations for accounts with a renewal in the next 30 days | Proactively address issues before renewal decisions |
| Engineering Lead | Which accounts with the highest product usage have filed the most tickets this quarter? | Identify power users impacted by bugs |
| VP of Customer Success | List all licenses with "At Risk" renewal status and their associated open tickets | Understand churn risk tied to unresolved support issues |
| Product Manager | What feature requests have come from customers with ARR above $100K? | Prioritize roadmap by revenue impact |
| Support Manager | Show conversations synced from Planhat that have no DevRev owner assigned | Identify gaps in support coverage |

---

## Sync Direction

- **Planhat -> DevRev**: Companies, End Users, Users, Tasks, Product Usage
- **Planhat <-> DevRev (Bidirectional)**: Conversations, Comments, Tickets, Licenses, Meetings
- **Sync type**: Bidirectional with TIME_SCOPED_SYNCS capability. Extract runs pull data from Planhat; load runs push DevRev changes back to Planhat.

---

## Authentication

- **Primary method**: Bearer Token (API Key)
- **Keyring**: `planhat-connection`, kind: `Secret`
- **Secret transform**: `.token` — extracts the token value from the stored secret
- **Verification**: `GET https://api.planhat.com/users?limit=1` with `Authorization: Bearer <token>` header. A 200 response confirms the key is valid and has API access.
- **Token storage**: Secure DevRev keyring storage. Token is never logged or exposed in sync payloads.
- **Reference**: https://docs.planhat.com/#authentication

---

## Object Mapping

| Planhat Object | DevRev Object | Sync Direction | Notes |
|----------------|---------------|----------------|-------|
| Company | Account | -> DevRev | Core account record. `name` -> `display_name`, `domains` -> `websites`, `owner` -> `owned_by`, `status` -> state (mapped to ACTIVE) |
| End User | Rev User (End User) | -> DevRev | Customer contacts. `firstName` -> `display_name`, `email` -> `email`, `phone` -> `phone_numbers`, `companyId` -> company reference, `tags` -> `tags` |
| User | Dev User (Team Member) | -> DevRev | Internal Planhat users. `firstName` -> `display_name`, `email` -> `email` |
| Conversation | Conversation (support type) | <-> Both | `subject` -> `title`, `endusers` -> `member_ids`, `owner` -> `owned_by_ids` |
| Comment | Comment | <-> Both | `text` -> `body` (rich_text), `by` -> `created_by_id`, `commentableId` -> `parent_object_id`, visibility fixed to INTERNAL |
| Ticket | Ticket | <-> Both | `subject` -> `title`, `description` -> `body` (rich_text), `owner` -> `created_by_id`, severity fixed to medium, stage fixed to in_development |
| Task | Conversation | -> DevRev | One-way import of Planhat tasks as conversations |
| License | Custom Object | <-> Both | `companyName` -> `title`, includes `product`, `arr`/`mrr`, `renewalStatus` enum, `invoiceCycle` enum |
| Meeting | Custom Object | <-> Both | `subject` -> `title`, `owner` -> `created_by_id`, `companyId` -> company reference |
| Product Usage | Analytics data | -> DevRev | Usage metrics ingested via analytics endpoint |

---

## Permission Mapping

Planhat uses role-based access at the tenant level (Admin, Manager, User, Viewer). V1 does not replicate Planhat's permission model in DevRev — all synced data inherits the default DevRev visibility for the target object type. Conversation and comment visibility is fixed to INTERNAL.

V2 consideration: Map Planhat user roles to DevRev groups for filtered visibility on sensitive account data (e.g., ARR, renewal status).

---

## Proposed Architecture

The snap-in uses the ADaaS (Airdrop Data as a Service) framework for bidirectional sync with the following pipeline:

### Extraction Pipeline (Planhat -> DevRev)

```
Planhat API ──> Extractor ──> Transformer ──> DevRev Airdrop Platform
                  │                │
                  │                └── Field mapping + type coercion
                  └── Paginated fetch (offset-based, limit=500, page=50)
```

1. **Authentication**: User provides Planhat API key through Airdrop settings. Key is stored in DevRev keyring (`planhat-connection`). Verification call confirms access.
2. **Sync unit selection**: Entire Planhat tenant is the sync unit (no sub-workspace selection needed).
3. **Schema interpretation**: Snap-in reads Planhat's custom fields per parent object via `GET /custom_fields` and constructs the external domain metadata. Custom field types supported: text, number, date, boolean, multipicklist.
4. **Field mapping**: Auto-suggested mappings based on the object mapping table above. Custom fields are mapped to DevRev custom fields with type coercion.
5. **Data extraction**: Objects are extracted in a deterministic order to satisfy reference dependencies:

   **Extraction Order**: Companies -> Users -> Conversations -> Licenses -> Tasks -> EndUsers -> Product Usage -> Attachments -> Tickets -> Comments

   This order ensures that when End Users reference a Company, or Comments reference a Conversation, the parent object has already been extracted.

### Loading Pipeline (DevRev -> Planhat)

```
DevRev Airdrop Platform ──> Loader ──> Planhat API
                              │
                              └── Create + Update operations
                                  Link processing (markdown -> HTML)
```

6. **Reverse sync (Loading)**: Handles create and update operations for bidirectional object types:
   - Conversations, Comments, Tickets, Licenses, Meetings
   - Link processing: Converts markdown link syntax `[text](url)` and raw `<url>` to HTML anchor tags `<a href="url">text</a>` before pushing to Planhat

### API Configuration

| Parameter | Value |
|-----------|-------|
| Base URL (main) | `https://api.planhat.com` |
| Base URL (analytics) | `https://analytics.planhat.com` |
| Pagination | Offset-based (`offset` + `limit` params) |
| Default limit | 500 |
| Page size | 50 |
| Timeout | 180,000ms (3 minutes) |
| Max retries | 3 |
| Retry delay | 1s with exponential backoff |
| 429 response | Retry after 15 seconds |
| 5xx response | Retry after 5 minutes |
| Rate limit headers | `x-ratelimit-remaining`, `x-ratelimit-limit`, `x-ratelimit-reset` |

### Mapping Rules

- **Object-to-object mapping**: Each Planhat object maps to exactly one DevRev object type (see Object Mapping table)
- **Field-to-field mapping**: Standard fields are auto-mapped; custom fields are mapped by name with type coercion
- **Validation**: Required fields (e.g., `name` for Companies, `email` for End Users) must be present or the record is skipped with a warning
- **Lookup linking**: `companyId` fields on End Users and Meetings resolve to the synced DevRev Account via external reference ID. `commentableId` on Comments resolves to the parent Conversation or Ticket.
- **Fixed values**: Ticket severity is fixed to `medium`, stage to `in_development`. Comment visibility is fixed to `INTERNAL`. Company status maps to `ACTIVE` state.
- **Rich text handling**: `description` and `text` fields are stored as `rich_text` type in DevRev with markdown-to-rich-text conversion

---

## Functional Requirements

- Support secure authentication using Planhat API key stored in DevRev keyrings
- Support bidirectional sync for Conversations, Comments, Tickets, Licenses, and Meetings
- Support extract-only sync for Companies, End Users, Users, Tasks, and Product Usage
- Import companies with domains, ownership, and custom fields
- Import end users with email, phone, company association, and tags
- Import conversations and comments with ownership and membership
- Import licenses with ARR/MRR, renewal status, and invoice cycle metadata
- Import product usage data into DevRev analytics
- Support per-parent custom field discovery and mapping (text, number, date, boolean, multipicklist)
- Reverse-sync created and updated records back to Planhat with proper link formatting
- Support time-scoped incremental sync via TIME_SCOPED_SYNCS capability
- Synced data is searchable via DevRev AI agents
- Synced data is usable in DevRev workflows (triggers, conditions, actions)
- Synced accounts and contacts are visible in DevRev records and vistas

## Non-Functional Requirements

- **Security**: API tokens stored exclusively in DevRev keyrings (`planhat-connection`, kind: Secret). Tokens are never included in logs, error messages, or sync payloads. Secret transform (`.token`) isolates the credential from other keyring metadata.
- **Performance**: Initial full sync should complete within 60 minutes for a Planhat tenant with up to 50,000 companies and 200,000 end users. Incremental syncs should complete within 10 minutes for typical daily change volumes.
- **Rate limit compliance**: Snap-in respects `x-ratelimit-remaining` headers and implements backoff (15s for 429, 5 min for 5xx) to avoid API throttling. Maximum 3 retries with exponential backoff starting at 1 second.
- **Timeout handling**: Individual API requests time out at 3 minutes (180,000ms). ADaaS task-level timeout is handled via `processTask({ task, onTimeout })` per SDK conventions.
- **Scalability**: Extraction uses offset-based pagination with a page size of 50 and max limit of 500 to handle large datasets without memory pressure. Product usage data is streamed via the analytics endpoint.
- **Reliability**: Deterministic extraction order ensures referential integrity. Failed records are logged and skipped without blocking the sync run.

---

## Constraints

- Product Usage extraction uses a separate analytics endpoint (`https://analytics.planhat.com`) which may have different rate limits than the main API
- Ticket severity and stage are fixed values (medium / in_development) — Planhat does not expose equivalent granularity in V1
- Comment visibility is fixed to INTERNAL — public comment sync deferred to V2
- Custom field types limited to: text, number, date, boolean, multipicklist. Other Planhat custom field types are skipped.
- Reverse sync supports create and update operations only — delete propagation is not supported in V1
- Dependencies: `@devrev/ts-adaas` v1.12.2, `@devrev/typescript-sdk` v1.1.63, `axios`, `js-jsonl`

---

## Success Criteria

- User can authenticate a Planhat connection in DevRev using an API key
- User can initiate a full sync and see Companies appear as Accounts in DevRev
- End Users, Users, Conversations, Comments, Tickets, Tasks, Licenses, Meetings, and Product Usage all sync with correct field mappings
- Custom fields (text, number, date, boolean, multipicklist) are discovered and synced per parent object
- Incremental time-scoped sync imports only records changed since the last successful sync
- Bidirectional objects (Conversations, Comments, Tickets, Licenses, Meetings) created or updated in DevRev are pushed back to Planhat
- Markdown links in reverse-synced content are converted to HTML anchor tags
- Synced data is searchable via DevRev AI search (e.g., "Show me the renewal status for Acme Corp")
- Rate limit compliance: zero 429 errors during normal sync operation
- Sync completes without data loss — failed records are logged with actionable error details

---

## Release Plan

**V1 (Current scope)**: Full bidirectional sync across all 10 object types. Includes extraction pipeline, loading pipeline, custom field support, and time-scoped incremental sync.

**V2 (Future considerations)**:
- Public comment visibility support
- Delete propagation (DevRev -> Planhat)
- Planhat role-to-DevRev group permission mapping
- Webhook-driven real-time sync (if Planhat adds webhook support)
- Health score as a first-class synced metric on DevRev Accounts

## Open Questions

- [ ] Should Product Usage data be aggregated before ingestion, or should raw daily metrics be synced?
- [ ] How should conflicts be resolved when the same Conversation is updated in both Planhat and DevRev between sync runs? (Current: last-write-wins)
- [ ] Should Planhat's NPS/CSAT survey data be included as a future object type?
- [ ] What is the expected custom field volume per tenant? (Impacts schema interpretation performance)
- [ ] Should archived/inactive Companies still sync, or be filtered out by default?

## References

- Planhat API documentation: https://docs.planhat.com/
- Planhat authentication: https://docs.planhat.com/#authentication
- DevRev ADaaS SDK: https://www.npmjs.com/package/@devrev/ts-adaas
- DevRev Airdrop documentation: https://docs.devrev.ai/integrations/airdrop
