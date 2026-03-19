# UI Automation Flow Reference

Step-by-step DevRev UI flow for end-to-end testing using Claude in Chrome. The tester drives the actual browser to prove the connector works in real conditions.

---

## Pre-requisites Checklist

Before starting UI automation, verify:

- [ ] Snap-in code is built (`npm run build && npm run package`)
- [ ] Snap-in is deployed to test org (`devrev snap_in_version create-one --archive build.tar.gz`)
- [ ] Snap-in is in DRAFT or ACTIVE state (`devrev snap_in_version show`)
- [ ] External system test account exists with credentials
- [ ] Test data exists in external system (or tester creates it via API)
- [ ] DevRev test org URL is known (e.g., `https://app.devrev.ai/<org-slug>`)

---

## Phase A: Create Test Data in Source System

**Ask the user first**: "Do you have a test account in [external system] with sample data? If not, I can create test records through the API."

If creating test data, use the external system's API to create a controlled dataset:

### What to create:

| Record type | Count | Variety | Purpose |
|------------|-------|---------|---------|
| Users | 3 | Different roles, emails | Test user mapping |
| Primary records (tasks/tickets) | 8-10 | Mix of statuses, priorities, assignees | Test field mapping |
| Records with attachments | 2 | Different file types (pdf, png) | Test attachment extraction |
| Edge case record | 1 | Unicode title, very long description, null optional fields | Test normalization |
| Empty/minimal record | 1 | Only required fields filled | Test null handling |

### Document what was created:

```markdown
## Test Data Created in [External System]

| ID | Title | Status | Priority | Assignee | Has Attachments | Notes |
|----|-------|--------|----------|----------|----------------|-------|
| EXT-1 | Fix login page timeout | Open | High | user1@test.com | No | Normal record |
| EXT-2 | Dashboard widget crash | In Progress | Critical | user2@test.com | Yes (screenshot.png) | Has attachment |
| EXT-3 | Tâche spéciale: テスト | Done | Low | — | No | Unicode + null assignee |
| ... | ... | ... | ... | ... | ... | ... |
```

This document is used later to verify every record imported correctly.

---

## Phase B: Configure and Activate Snap-in

### Steps in DevRev UI:

```
1. Navigate to: https://app.devrev.ai/<org-slug>
2. Click Settings (gear icon) in left nav
3. Click Snap-ins in the settings menu
4. Find the snap-in by name
5. Click on the snap-in to open configuration

6. Configure connection / keyring:
   - Click "Add Connection" or configure the keyring section
   - For API token: paste the external system's API token
   - For OAuth: follow the OAuth flow (authorize → callback → token saved)
   - Click Save

7. Configure any inputs (if the snap-in has configurable inputs):
   - Fill in required inputs
   - Set optional inputs as needed
   - Click Save

8. Click "Deploy" or "Activate"
9. Wait for status to change to ACTIVE
   - If ERROR: read the error message
   - Common errors: invalid keyring, missing required input, function build failure
```

---

## Phase C: Start the Import

### Steps in DevRev UI:

```
1. Go to Settings → Integrations → Airdrops
2. Click "+ Import" or "Start Airdrop"
3. Select the connector from the dropdown list
   (Should show the snap-in's display_name from manifest)

4. Select or create a connection:
   - If connection already configured in Step B: select it
   - If new connection needed: create one with credentials

5. Wait for sync unit extraction:
   - The snap-in calls the external API to list available containers
   - A list appears (projects / workspaces / channels / boards)

6. Select the sync unit containing your test data
   - This is the project/workspace where you created test records in Phase A

7. Click Next / Continue
```

---

## Phase D: Field Mapping Screen (MOST CRITICAL STEP)

This is where most issues are caught. The mapping screen shows how external fields map to DevRev fields.

### What to verify on the mapping screen:

```
1. Record type mappings:
   - External "task" → DevRev "issue" (or "ticket") — is this correct per the TDD?
   - External "user" → DevRev "dev_user" (or "rev_user") — correct?
   - Any unmapped record types that should be mapped?

2. Field mappings (for each record type):
   - Title/name → title: should be auto-mapped
   - Description → body: should be auto-mapped
   - Status → stage: check value mappings
   - Priority → priority: check value mappings
   - Assignee → owned_by: check reference resolution
   - Created date → created_date: should be auto-mapped
   - Modified date → modified_date: should be auto-mapped

3. Value mappings (for enum fields):
   - Status values: verify every external status maps to correct DevRev stage
     - Example: "open" → "backlog", "in_progress" → "in_development", "done" → "done"
   - Priority values: verify priority mapping
     - Example: "critical" → "p0", "high" → "p1", "medium" → "p2", "low" → "p3"

4. Custom field mappings:
   - External custom fields → DevRev custom fields
   - Some may need manual mapping

5. Fix any incorrect mappings:
   - Click on a field mapping to change it
   - Add mappings for unmapped fields
   - Remove incorrect auto-suggested mappings

6. Screenshot the final mapping configuration
   (for the test report)

7. Click "Import" / "Start" to begin extraction
```

### Common mapping issues to watch for:

| Issue | How it appears | Fix |
|-------|---------------|-----|
| Status values not mapped | Empty DevRev stage column | Manually map each value |
| Priority not recognized | All imported records have default priority | Map priority values in the UI |
| User not resolved | Assignee shows as "Unknown" | Verify email match between systems |
| Custom fields missing | External fields not shown | Check external_domain_metadata.json includes them |
| Wrong record type | Tasks mapped to tickets instead of issues | Change record type mapping |

---

## Phase E: Monitor Extraction

```
1. After clicking Import, the Airdrops UI shows progress
2. Watch the phases:
   Phase 1: "Extracting sync units" (should already be done)
   Phase 2: "Extracting metadata" (usually quick)
   Phase 3: "Extracting data" (may take time for large datasets)
   Phase 4: "Loading data" (transforming and importing)
   Phase 5: "Extracting attachments" (if applicable)
   Final: "Import Completed"

3. Note the duration (for test report)

4. If the status shows "Error":
   - Click on the import to see error details
   - Common errors:
     - Rate limit exceeded (retry usually fixes this)
     - Auth token expired during extraction
     - Invalid data format (normalization bug)
     - Timeout (large dataset, needs state management fix)
   - Report as bug with the exact error message
```

---

## Phase F: Verify Imported Data

The most detailed verification step. Compare every test record against what was created in Phase A.

```
1. Navigate to the relevant vista:
   - Tickets: Support → Tickets
   - Issues: Build → Issues
   - Articles: Knowledge Base → Articles
   - Accounts: Grow → Accounts

2. Filter or search for imported records
   - Search by title of test records
   - Or filter by "imported" tag if the snap-in adds one

3. For EACH test record (use the table from Phase A):

   Record EXT-1: "Fix login page timeout"
   - [ ] Title matches: "Fix login page timeout"
   - [ ] Status/stage mapped correctly: Open → [expected DevRev stage]
   - [ ] Priority mapped correctly: High → [expected DevRev priority]
   - [ ] Assignee: user1@test.com → correct DevRev user
   - [ ] Created date matches
   - [ ] Description/body present and formatted

   Record EXT-2: "Dashboard widget crash" (has attachment)
   - [ ] All fields mapped (same as above)
   - [ ] Attachment is present on the record
   - [ ] Attachment is downloadable
   - [ ] Attachment name matches original

   Record EXT-3: "Tâche spéciale: テスト" (unicode + null fields)
   - [ ] Unicode characters preserved in title
   - [ ] Null assignee: owned_by is empty (not "Unknown" or error)
   - [ ] Status "Done" mapped correctly

4. Check timeline entries:
   - Click on any imported record
   - Look for sync activity in the timeline
   - Should show "Imported from [External System]" or similar

5. Document any mismatches for the bug report
```

---

## Phase G: Test Incremental Sync

```
1. Go to external system:
   - Modify 1 existing record (change status from "Open" to "In Progress")
   - Create 1 new record (EXT-NEW)

2. In DevRev: trigger incremental sync
   - Settings → Integrations → Airdrops
   - Find the import → ⋮ → "Sync [External] to DevRev"
   - OR: ⋮ → "Set Periodic Sync" → wait for next cycle

3. After sync completes, verify:
   - Modified record: status changed from old DevRev stage to new mapped stage
   - New record: appears in the vista with all fields mapped
   - Unchanged records: no modifications (check modified_date unchanged)

4. Document results
```

---

## Result Documentation Template

After completing all phases, document in TEST_RESULTS.md:

```markdown
## UI Automation Results

### Environment
- DevRev org: [org-slug]
- External system: [system] test account
- Snap-in version: [version]
- Test date: [date]

### Test data summary
[Paste the table from Phase A]

### Phase results
| Phase | Status | Duration | Notes |
|-------|--------|----------|-------|
| A: Test data created | ✅/❌ | — | [N records created] |
| B: Snap-in activated | ✅/❌ | — | [Any config issues] |
| C: Import started | ✅/❌ | — | [Sync unit selected] |
| D: Mapping verified | ✅/❌ | — | [Manual fixes needed?] |
| E: Extraction completed | ✅/❌ | [Xm Xs] | [Record count] |
| F: Data verified | ✅/❌ | — | [Mismatches found] |
| G: Incremental sync | ✅/❌ | — | [New + modified verified] |

### Field-level verification
| Record | Title | Status | Priority | Assignee | Attachment | Notes |
|--------|-------|--------|----------|----------|------------|-------|
| EXT-1 | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | N/A | |
| EXT-2 | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | |
| EXT-3 | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | N/A | Unicode test |

### Bugs found
[List with severity, steps to reproduce, expected vs actual]
```
