# PRD Template for DevRev Snap-ins

Grounded in real DevRev PRDs (Planhat Airdrop, Snowflake ADaaS). Follow this structure. Sections marked [OPTIONAL] include when contextually relevant.

When examining real PRDs provided by the user (uploaded to `references/examples/`), study them to match the tone, depth, and format of the user's team.

---

```markdown
# PRD: [Solution Name] – DevRev [Airdrop|Integration]

## Overview

This document outlines the [product requirements | approach] for [building a one-way sync from X to DevRev | enabling two-way data synchronization between X and DevRev | integrating X into DevRev's search agent]. The goal is to [enable X customers to... | unify X and DevRev workflows by...].

## Problem Statement

[2-3 sentences. Specific pain — who suffers, what they do manually, what data is trapped where.]

## Goal

1. [Concrete goal — e.g., Enable two-way sync of critical data between X and DevRev]
2. [e.g., Reduce manual updates and duplications across tools]
3. [e.g., Power AI-based automation in DevRev using X signals]
4. [e.g., Enable unified, searchable context across tools]

---

## [OPTIONAL] Persona Queries

Include when integration powers search, AI agents, or cross-tool visibility. Shows business value through real queries real users would make.

| Persona | Query | Intent/Action |
|---------|-------|---------------|
| [Role] | [Natural language query they'd type] | [What they want to accomplish] |

---

## Sync Direction

- **X → DevRev**: [List objects]
- **DevRev → X**: [List objects]
- **Sync type**: [One-way | Bidirectional | V1 scope note]

---

## Authentication

- **Primary method**: [OAuth 2.0 | API token | Key Pair (JWT) | Basic auth]
- **Tenant handling**: [Tenant-specific credentials | Subdomain-based]
- **Token storage**: [Secure keyring storage, auto-refresh]
- **Alternatives**: [Less secure options for testing]
- **Reference**: [Link to external system's auth docs]

---

## Object Mapping

| X Object | DevRev Object | Sync Direction | Notes |
|----------|---------------|----------------|-------|
| | | [→ DevRev / ← DevRev / ↔ Both] | |

---

## Permission Mapping

[How external permissions translate to DevRev access control. Even if deferred, state it.]

- [Mapping rules or "Not needed for V1 — deferred to V2 with rationale"]

---

## Proposed Architecture

[For AirSync: describe the ADaaS pipeline]

1. **Authentication**: User provides credentials through Airdrop settings
2. **Sync unit selection**: [What the user selects — workspace, project, table]
3. **Schema interpretation**: [How structure is read]
4. **Field mapping**: [Auto-suggested or manual]
5. **Data extraction**: Extracted from X → pushed to DevRev via Airdrop

### Mapping Rules
- Object-to-object mapping: [Rule]
- Field-to-field mapping: [Rule]
- Validation: [Required fields, data types]
- Lookup linking: [How custom objects reference stock objects]

[INCLUDE ARCHITECTURE DIAGRAM — Source → Extractor → Transformer → Loader → DevRev]

---

## Functional Requirements

- Support secure authentication for both X and DevRev tenants
- Support sync with object-level and field-level configurations
- [Ability to search synced data via DevRev AI agents]
- [Ability to use synced data in workflows — triggers, conditions, actions]
- [Ability to view synced data in records, vistas, dashboards]
- [Honor permissions and visibility controls]

## Non-Functional Requirements

- **Security**: [Record/field level permissions, visibility mapping]
- **Performance**: [Transactions per minute, initial sync baseline]
- **Search latency**: [Thought streaming, complete response targets]
- **Scalability**: [Load limits, large table handling]
- **Credential handling**: [Secure storage]

---

## Constraints

- [What's explicitly out of scope — e.g., write-back not supported]
- [API dependencies]
- [SDK version requirements]

---

## Success Criteria

- [User can authenticate X connections in DevRev]
- [User can sync X data into DevRev objects via Airdrop]
- [User can identify errors in migration]
- [User can setup periodic sync]
- [Data is visible and searchable in DevRev]

---

## Release Plan

[Single release or phased. V1 vs V2 scope.]

## Open Questions

- [ ] [Unresolved technical or product question]

## References

- [Links to similar integrations, API docs, internal recordings]
```

---

## PRD Quality Checklist

- [ ] Problem statement names the pain and who feels it
- [ ] Goals are numbered and concrete
- [ ] Sync direction is explicit (→ ← ↔) for every object
- [ ] Auth method specified with link to external docs
- [ ] Object mapping table covers all entities
- [ ] Permission mapping addressed (even if "deferred")
- [ ] Architecture flow described step-by-step
- [ ] Functional reqs cover: auth, sync, search, workflows, vistas
- [ ] Non-functional reqs cover: security, performance, latency, scalability
- [ ] Constraints listed
- [ ] Success criteria are measurable
- [ ] Persona queries included if integration powers search/AI
