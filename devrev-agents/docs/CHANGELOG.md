# Skill Updates Log

Track every improvement made to DevRev agent skills based on real usage feedback.

---

<!-- New entries go at the top -->

## v1.0.0 — 2026-03-19

### Initial Release

**Snap-in vertical** (PM, Architect, Tester) ready for production use.

#### Skills
- **devrev-snapin-pm** — Discovery, PRD, TDD, handoff brief generation
- **devrev-snapin-architect** — API research, 15 engineering decisions, full code generation
- **devrev-snapin-tester** — Unit tests (Jest) + DevRev UI automation flows
- **devrev-skill-improver** — Diagnose and fix skill errors from real usage

#### Key changes from pre-release
- Replaced `airsync-snapin.md` with `airsync-template.md` — complete generic scaffold derived from `devrev/airdrop-asana-snap-in` and cross-referenced with a production Zoho CRM connector
- Fixed critical SDK patterns: `spawn()` routing, `adapter.streamAttachments()`, metadata repo push, `connection_data.key` JSON parsing, `adapter.postState()` in onTimeout
- Added both auth patterns (PAT and config-type OAuth with client_credentials)
- Removed unfinished implementation vertical stubs (will ship in a future release)
