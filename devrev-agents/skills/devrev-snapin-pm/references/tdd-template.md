# TDD Template for DevRev Snap-ins

Grounded in real DevRev TDDs (Slack ADaaS, Monday.com Integration). Follow this structure closely. The TDD translates the PRD into architecture the snap-in architect can implement directly.

When examining real TDDs provided by the user (in the `examples/` directory at the plugin root), study them to match the format, depth, and section ordering of the user's team.

---

```markdown
# Technical Design Document (TDD) for [Solution Name] Integration with DevRev

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
| **Solution name** | [e.g., Slack Integration for DevRev Search Agent] |
| **Author** | [Name] |
| **Reviewers** | [Names — mark "Pending" if not yet assigned] |
| **Status** | [In progress | In review | Approved] |
| **Created date** | [Date] |

## 2. Introduction

[One paragraph — what this integration does technically and what it aims to achieve.]

Example: "This document outlines the technical specifications for integrating Slack public and private channel conversations into DevRev's search agent. This integration aims to improve knowledge accessibility and streamline information retrieval from Slack within DevRev."

---

## 3. Requirements

### 3.1 Functional Requirements

System must:
- [e.g., Import X data into DevRev as a searchable object while preserving metadata like timestamps and participant information]
- [e.g., Ensure only authorized content is imported, respecting X's permission structure]
- [e.g., Implement a permission validation system to map X permissions to DevRev's framework]
- [e.g., Regularly synchronize new data and updates from X into DevRev]
- [e.g., Authenticate using OAuth 2.0 for X API access]
- [e.g., Provide role-based access control (RBAC) compliance with DevRev]

### 3.2 Non-functional Requirements

System shall be:
- **Secure**: [e.g., User access restricted based on X permissions]
- **Scalable**: [e.g., Able to handle large volumes of data and concurrent requests]
- **Reliable**: [e.g., Maintain consistency in data retrieval]
- **Performant**: [e.g., Fast response times for search queries]

**Data volume**: [Specify or mark TBD]

### 3.3 Constraints

- **Technical**: [e.g., Integration must adhere to DevRev and X APIs. API-specific limitations listed.]
- **External**: [e.g., Dependence on DevRev SDK support and version]

---

## 4. Sequence Diagram

[Generate a sequence diagram showing the ADaaS flow. Follow this standard pattern from real DevRev TDDs:]

**Actors**: User, Airdrop Component, Extractor, X API

**Standard ADaaS flow:**
```
User → Airdrop: Select Import
Airdrop → Extractor: Extraction Sync Unit Start
Extractor → X API: [List available containers — e.g., conversations.list, workspaces query]
Extractor → Airdrop: Emit Extraction Sync Unit Done
Airdrop → User: List of [containers]
User → Airdrop: On selecting [container(s)]
Airdrop → Extractor: Extraction Metadata Start
Extractor → Airdrop: Upload metadata.json & emit metadata done
Airdrop → Extractor: Extract Data Start Event
Extractor → X API: [List users — e.g., users.list, users query]
Extractor → Airdrop: Format Users, Update & Emit ExtractionDataProgress
Extractor → X API: [List primary data — e.g., conversations.history, items query]
Extractor → X API: [List nested data — e.g., conversations.replies, updates query]
Extractor → Airdrop: Format Data, Update & Emit ExtractionDataProgress
Airdrop → User: Shows Extracted Data
User → Airdrop: Map fields
Airdrop → Extractor: Transform & Loading
Extractor → X API: [Extract attachments if applicable]
Airdrop → User: Import Completed
```

[GENERATE THIS AS A VISUAL DIAGRAM — use the visualizer for a sequence-style SVG]

---

## 5. Data Mapping

[This is the most critical section. Show field-by-field mappings for EVERY object type. Use visual diagrams AND tables.]

### [Object 1]: X [Entity] → DevRev [Entity]

| X Field | DevRev Field | Notes |
|---------|-------------|-------|
| [e.g., Name] | [e.g., Display Name] | [Direct map] |
| [e.g., Email] | [e.g., Email] | [Direct map] |
| [e.g., Id] | [e.g., Id] | [External reference] |

### [Object 2]: X [Entity] → DevRev [Entity]

| X Field | DevRev Field | Notes |
|---------|-------------|-------|
| [e.g., Name] | [e.g., name] | |
| [e.g., Member IDs] | [e.g., Member IDs] | |
| [e.g., Creator] | [e.g., Created By] | |

### [Object 3]: X [Entity] → DevRev [Entity]

| X Field | DevRev Field | Notes |
|---------|-------------|-------|
| [e.g., text] | [e.g., Body] | |
| [e.g., parent object type] | [e.g., Parent Object Type] | [e.g., = "chat" or "comment"] |
| [e.g., channel_id / message_id] | [e.g., Parent object Id] | |
| [e.g., Created date] | [e.g., Created date] | |
| [e.g., Modified date] | [e.g., Modified date] | |

### [OPTIONAL] Field Type Inventory

For systems with many field types (like Monday.com's 40+ column types), list all available types and indicate which are imported vs unsupported:

**Available field types in X:**
[Full list]

**Imported in V1:**
[Subset list]

**Not supported in V1:**
[Remainder with rationale]

[GENERATE DATA MAPPING AS VISUAL DIAGRAM — two-column layout showing X fields → DevRev fields with arrows, matching the style from Slack and Monday.com TDDs]

---

## 6. Endpoints

### 6.1 DevRev API Endpoints

| Endpoint | Description |
|----------|-------------|
| [ADaaS SDK version] | SDK for Airdrop |
| [Chef CLI] | Domain mapping |

### 6.2 External API Endpoints

**Authentication setup:**
[Step-by-step instructions specific to the external system]

Example for OAuth-based systems:
1. [e.g., Create a new app at external system's developer portal]
2. [e.g., Choose configuration options]
3. [e.g., Add required OAuth scopes]
4. [e.g., Install app and copy token]

**Required OAuth Scopes:**

| Scope | Description |
|-------|-------------|
| [e.g., channels:history] | [e.g., View messages in public channels] |
| [e.g., channels:read] | [e.g., View basic info about public channels] |
| [e.g., files:read] | [e.g., View files shared in channels] |
| [e.g., users:read] | [e.g., View people in a workspace] |
| [e.g., users:read.email] | [e.g., View email addresses of people] |

**API Endpoint Reference:**

| X Entity | DevRev Entity | API Reference | Scopes/Notes |
|----------|---------------|---------------|-------------|
| [e.g., Conversations] | [e.g., conversations.list] | [API URL] | [Required scopes] |
| [e.g., Conversation history] | | [API URL] | [Required scopes] |
| [e.g., Users] | [e.g., users.info] | [API URL] | [Scopes + notable response fields] |

---

## 7. Conclusion

### 7.1 Risks Assumed

- [e.g., Dependence on future availability and functionality of X API may necessitate adjustments]
- [e.g., API rate limits may impact sync performance]
- [e.g., Permission inconsistencies may lead to access control issues]

### 7.2 Trade-offs

- [e.g., Initial development focuses on basic functionality with enhancements in future iterations]
- [e.g., Private channel access deferred to V2 due to additional security considerations]

### 7.3 Limitations

- [e.g., Initially focuses on public channels only]
- [e.g., Real-time updates may introduce additional latency]
- [e.g., Private access might require additional admin approvals]

### 7.4 Future Expansion

- [e.g., Explore integrating private channels with user-based permission approvals]
- [e.g., Implement real-time search functionality]
- [e.g., Implement AI-driven search improvements]
- [e.g., Extend support for additional entities and custom workflows]
```

---

## TDD Quality Checklist

Before presenting to user:

- [ ] Overview metadata table is complete (author, reviewers, status, date)
- [ ] Functional requirements list what the system "must" do
- [ ] Non-functional requirements cover security, scalability, reliability, performance
- [ ] Constraints list technical, external, and any resource constraints
- [ ] Sequence diagram follows ADaaS flow pattern (sync unit → metadata → data → attachments)
- [ ] Data mapping covers EVERY object type with field-by-field tables
- [ ] Data mapping includes visual diagrams (not just tables)
- [ ] Field type inventory included for complex external systems
- [ ] DevRev endpoints list ADaaS SDK version and Chef CLI
- [ ] External endpoints include auth setup steps, OAuth scopes with descriptions
- [ ] API reference table maps X entities to API endpoints with scopes
- [ ] Conclusion has all 4 sections: Risks, Trade-offs, Limitations, Future Expansion
- [ ] No ambiguity — the architect could implement without asking questions
