---
name: update
description: Update DevRev Agents plugin to the latest version
---

# Update DevRev Agents

You are updating the DevRev Agents plugin to the latest version from GitHub.

## Steps

1. **Find the plugin install directory**

   Run this command to locate where the plugin is installed:
   ```bash
   find ~/.claude/plugins -name "plugin.json" -path "*devrev*" 2>/dev/null
   ```

   If not found, check:
   ```bash
   find . -name "plugin.json" -path "*devrev*" 2>/dev/null
   ```

2. **Pull the latest version**

   Navigate to the plugin's root directory (the git repo root, not `.claude-plugin/`) and pull:
   ```bash
   cd <plugin-root-directory>
   git fetch origin main
   git pull origin main
   ```

3. **Show what changed**

   Display the changelog to the user:
   ```bash
   cat devrev-agents/docs/CHANGELOG.md
   ```

4. **Read the new version**

   ```bash
   cat .claude-plugin/plugin.json | grep '"version"'
   ```

5. **Confirm to the user**

   Tell the user:
   - The version they updated to
   - A summary of what changed (from the changelog)
   - That they should restart their Claude Code session to load the new skills

## If the plugin was installed from GitHub

If the plugin directory is inside `~/.claude/plugins/`, just `git pull` in that directory.

## If the plugin is loaded locally via `--plugin-dir`

If the user cloned the repo and is using `--plugin-dir`, just `git pull` in their local clone.

## Error handling

- If `git pull` fails due to local changes: suggest `git stash && git pull && git stash pop`
- If the repo is not a git directory: suggest re-installing with `/plugin install --github QK-SnapIn/devrev-qk-agents`
- Always show the user what version they're now on
