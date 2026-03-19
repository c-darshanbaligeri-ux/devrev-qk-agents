# Handoff Template

The handoff brief is what the snap-in architect skill receives. Self-contained — the architect should not need to read the full PRD/TDD to start coding.

---

```markdown
## Handoff: [Snap-in Name]

**PRD Status**: Approved by [user] on [date]
**TDD Status**: Approved by [user] on [date]

---

### What to build
[One paragraph — what this snap-in does]

### Snap-in type
[airsync | automation | integration | command | hybrid]

### ADaaS pipeline (for AirSync connectors)

**Sync direction**: [X → DevRev | DevRev → X | Both]

**Sync units**: [What the user selects — channels, boards, tables, etc.]

**Extraction order**:
1. Users — via [API endpoint]
2. [Primary records] — via [API endpoint]
3. [Nested records] — via [API endpoint]  
4. Attachments — via [API endpoint]

### Authentication
- **Method**: [OAuth2 | API token | JWT key pair]
- **Setup**: [Brief steps — e.g., "Create Slack app → add scopes → install → copy token"]
- **Required scopes**: [List]
- **Keyring type**: [secret | oauth2]
- **Is subdomain**: [true/false]

### Object mapping

| X Entity | DevRev Entity | Direction |
|----------|---------------|-----------|
| | | |

### Field mapping (key fields per object)

**[Object 1]: X → DevRev**

| X Field | DevRev Field | Transform |
|---------|-------------|-----------|
| | | |

**[Object 2]: X → DevRev**

| X Field | DevRev Field | Transform |
|---------|-------------|-----------|
| | | |

### External API reference

| Method | Endpoint | Purpose | Rate Limit |
|--------|----------|---------|------------|
| | | | |

### Error handling
- Rate limit: [Strategy]
- Auth failure: [Strategy]
- Diff mode: [How incremental sync works]
- State management: [How batch state is tracked]

### DevRev tooling
- ADaaS SDK: [version]
- Chef CLI: [for domain mapping]

### Risks & limitations
- [Key risk from TDD]
- [Key limitation]

### Out of scope for V1
- [Deferred item 1]
- [Deferred item 2]

### Test scenarios the tester will validate

| Scenario | What to verify |
|----------|---------------|
| Full import | All sync units extracted, mapped, imported |
| Incremental sync | Only changed records imported on re-run |
| Rate limit handling | Graceful backoff on 429 |
| Auth failure | Clear error, admin notified |
| Attachment import | Files downloaded and linked |
| Permission mapping | Access controls honored |
```

---

## Handoff Protocol

When handing off, tell the user:

> "The plan is approved. Here's the handoff brief for the snap-in architect.
>
> **What happens next:**
> 1. The snap-in architect generates the complete code (manifest + TypeScript functions following the ADaaS pattern)
> 2. They provide deployment commands
> 3. The tester verifies the full extraction → mapping → import flow
>
> You can hand this to the architect by saying: *'Build the [snap-in name] connector — here's the approved plan.'*"

## When to Loop Back

Send back to PM if:
- Requirements changed after approval
- Architect discovers API limitation not covered in TDD
- User adds new scope during implementation
- External system's API behavior differs from documented
- A feasibility assumption was wrong (e.g., API doesn't support incremental sync)
