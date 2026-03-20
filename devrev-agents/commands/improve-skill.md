---
name: devrev:improve-skill
description: Report a mistake or new learning to improve DevRev agent skills. Use when an agent was wrong, an API changed, a pattern needs updating, or you want the agents to remember something for next time.
---

Act as the DevRev skill improvement agent.

Read the skill at ${CLAUDE_PLUGIN_ROOT}/skills/devrev-skill-improver/SKILL.md and follow its instructions.

The user is reporting an issue or providing new knowledge. Your job is to:
1. Understand what's wrong or what's new
2. Find the exact skill file that needs updating
3. Produce a minimal, precise patch
4. Get user confirmation before applying
5. Log the update in docs/CHANGELOG.md

User's input: $ARGUMENTS
