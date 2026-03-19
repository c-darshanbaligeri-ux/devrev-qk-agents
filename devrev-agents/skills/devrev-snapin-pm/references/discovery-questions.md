# Discovery Questions Bank

Organized by snap-in type. Use the relevant section based on what the user wants to build. Don't ask every question — pick the ones that fill gaps in what you already know.

---

## Universal Questions (Ask for Every Snap-in)

### Problem & Context
- What problem does this solve? What's painful about the current process?
- Who uses this? (Roles: support agents, developers, PMs, admins, customers?)
- How many people/teams will use this?
- Is there an existing tool or manual process this replaces?
- What does success look like? (Metric: reduced time, fewer missed SLAs, data accuracy?)

### Scope & Priority
- What's the MVP — the smallest version that delivers value?
- What's explicitly out of scope for V1?
- Any hard deadlines or dependencies blocking other work?
- Should this be private (your org only) or published to the DevRev marketplace?

### Security & Compliance
- Does the data include PII or sensitive information?
- Any data residency requirements? (data must stay in specific regions?)
- Who should be able to configure/install this snap-in? (admins only? any member?)
- Are there audit logging requirements?

---

## Automation Snap-ins

*"When X happens in DevRev, do Y automatically"*

### Trigger
- What DevRev event triggers the action? (Be specific: ticket created? Any ticket or specific conditions?)
- Should it trigger on every instance, or only when conditions are met? (e.g., only P0 tickets, only from enterprise accounts)
- What fields on the object matter for the trigger condition?
- Should there be a delay? (e.g., "if no response in 30 minutes")

### Action
- What should happen when triggered?
- Does the action modify the DevRev object? (change stage, assign, add tag, add comment?)
- Does the action involve an external system? (send Slack message, create Jira issue?)
- Should the action be visible to the customer? (timeline comment vs internal note?)
- What if the action fails? (retry? alert? skip?)

### Configuration
- Should the trigger conditions be configurable by the installer? (e.g., "which priority levels trigger this")
- Should the notification targets be configurable? (e.g., "which Slack channel")

---

## Integration Snap-ins

*"Connect DevRev with External System X in real-time"*

### External System
- What external system are we connecting to?
- Does it have a REST API? GraphQL? Webhooks? SDK?
- Where are the API docs? (URL)
- What authentication method? (API key, OAuth2, PAT, basic auth, HMAC?)
- Are there rate limits? What are they?
- Is there a sandbox/test environment available?
- Does the external system support webhooks? (for real-time events FROM the external system)

### Data Flow Direction
- DevRev → External only? (e.g., post to Slack when ticket created)
- External → DevRev only? (e.g., create ticket when GitHub issue is opened)
- Bidirectional? (sync state between both systems)

### Field Mapping
- What data fields need to move between systems?
- Are there enum/status values that need mapping? (e.g., "In Progress" in Jira = "in_progress" in DevRev)
- Are there fields that exist in one system but not the other?
- How should unmapped values be handled? (default value? skip? error?)

### Conflict Resolution (if bidirectional)
- If the same record is updated in both systems simultaneously, which wins?
- Should there be a "last write wins" policy or manual conflict resolution?
- How do we prevent infinite sync loops? (A updates B, B updates A, A updates B...)

---

## AirSync Connector Snap-ins

*"Migrate data from External System X and keep it in sync"*

### Source System
- What external system are we importing from?
- Is there a native DevRev connector? (check the list before building custom)
- What data containers exist? (projects, workspaces, boards, repos?)
- Does the user want to choose which containers to sync? (selective sync)

### Data Types
- What record types need to be imported? (tasks→tickets, users→rev_users, projects→parts?)
- For each record type: what fields are available? What fields matter?
- Are there attachments? (files, images, documents?)
- What's the data volume? (hundreds, thousands, millions of records?)

### Sync Requirements
- One-time bulk import only? Or ongoing sync?
- If ongoing: how often should it sync? (real-time webhooks, polling interval?)
- Forward sync only (external → DevRev)? Or also reverse sync (DevRev → external)?
- Should deletes in the external system propagate to DevRev?

### User Mapping
- How should external users map to DevRev users? (by email match? manual mapping?)
- What happens when an external user doesn't exist in DevRev? (create? assign to default?)

### Status & Field Mapping
- How do external statuses map to DevRev stages?
- How do priority levels map?
- Are there custom fields that need to carry over?
- What date fields matter? (created, updated, due date, resolved?)

---

## Command Snap-ins

*"Add a slash command that users can invoke from the DevRev UI"*

### Command Design
- What should the command be called? (e.g., `/summarize`, `/escalate`, `/link-jira`)
- Where should it be available? (conversations? tickets? issues? everywhere?)
- Does it need arguments? (e.g., `/assign-team platform-engineering`)
- Should it show a confirmation or result after execution?

### Behavior
- What does the command do when invoked?
- Does it read data, write data, or both?
- Does it call an external system?
- How long does it take to execute? (instant vs async?)
- What feedback should the user see? (success message, error message, result data?)

---

## Snap-kit UI Snap-ins

*"Add custom buttons, forms, or widgets inside DevRev"*

### UI Location
- Where should the UI element appear? (work item details, conversation sidebar, custom surface?)
- Is it a button, a form, a data display, or something else?
- What triggers the UI to show? (always visible? conditional?)

### Interaction
- What happens when the user interacts with it? (function call, navigation, modal?)
- Does it need to collect input from the user? (form fields?)
- What feedback does the user get after interaction?

---

## Cross-cutting Concerns (Ask When Relevant)

### Error Handling
- What should happen when the external API returns an error?
- Should there be automatic retries? How many? With what backoff?
- Should errors create DevRev tickets for the admin?
- Is there a fallback behavior?

### Monitoring & Observability
- How should the team know if the snap-in is working correctly?
- Are there health check requirements?
- Should there be usage analytics?

### Versioning & Updates
- How will the snap-in be updated when the external API changes?
- Should it be backward-compatible with older configurations?

### Testing
- Is there a test/sandbox environment for the external system?
- Are there test accounts or API keys for development?
- What scenarios need to be tested before going live?
