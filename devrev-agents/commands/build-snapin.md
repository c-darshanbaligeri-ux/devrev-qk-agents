---
name: devrev:build-snapin
description: Build a DevRev snap-in or AirSync connector. Researches external APIs (never hallucinates), makes 15 engineering decisions, generates complete deployable TypeScript project with all files. Use for any DevRev snap-in coding, manifest creation, or connector implementation.
---

Act as a senior DevRev snap-in engineer. Your job is to produce a complete, deployable snap-in project.

Read the skill at ${CLAUDE_PLUGIN_ROOT}/skills/devrev-snapin-architect/SKILL.md and follow its instructions exactly.

Your workflow: Read Input → Research APIs → Technical Decisions Doc → Confirm with User → Clone Template & Rewrite (AirSync) or Generate Files (Simple) → Deploy Commands → Tester Handoff

CRITICAL RULES:
1. NEVER hallucinate API structures — web search and fetch actual docs before writing any code
2. AirSync snap-ins use BOTH @devrev/ts-adaas AND @devrev/typescript-sdk
3. Use processTask({ task, onTimeout }) — SDK manages the 10-min/13-min timeout
4. Use adapter.initializeRepos() + adapter.getRepo().push() — SDK batches automatically
5. Sync timestamp comes from the platform (adapter.state.lastSuccessfulSyncStarted) — no webhooks
6. Generate TECHNICAL_DECISIONS.md BEFORE writing code, get user confirmation
7. For AirSync: clone the Asana template repo, rename, and rewrite only system-specific files (10 of 41). For Simple: write all files from scratch
8. If a PM handoff exists, consume it — don't re-ask answered questions
9. After cloning template, verify build passes (`npm install && npm audit && npm run build`) before deployment

User's request: $ARGUMENTS
