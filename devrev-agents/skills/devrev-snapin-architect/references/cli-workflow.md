# DevRev CLI & Chef-CLI Workflow Reference

The exact CLI workflow for building, testing, and deploying DevRev snap-ins. These are the real commands used in production.

---

## 1. Authentication

```bash
devrev profiles authenticate -o <org-slug> -u <user-email> --expiry 7
```

- `org-slug`: Your DevRev org name (e.g., `test-snap-in`)
- `user-email`: Your DevRev email (e.g., `c-user@devrev.ai`)
- `--expiry 7`: Token valid for 7 days
- This verifies the user who is creating/deploying the snap-in

---

## 2. Context Management

Snap-in context tracks which project you're working on. Required before package operations.

```bash
# Set active context for your project
devrev snap_in_context set <project-name>

# List all contexts
devrev snap_in_context list

# Show current context
devrev snap_in_context show

# Switch to a different context
devrev snap_in_context checkout <project-name>
```

---

## 3. Package Creation

Create the snap-in package (one-time per snap-in):

```bash
devrev snap_in_package create-one --slug <airdrop-snapin-name> | jq .
```

The slug is the machine-readable name. For AirSync: use `airdrop-<s>-snap-in`.

### OAuth Developer Keyring (if OAuth auth is used)

If the external system uses OAuth2, create the developer keyring with client credentials:

```bash
echo '{"client_id":"<client-id>","client_secret":"<client-secret>"}' | \
  devrev developer_keyring create oauth-secret <keyring-name>
```

This stores the OAuth client credentials securely. The keyring name must match the `keyring_types.id` in your manifest.

---

## 4. Manifest Validation

Always validate before deploying:

```bash
devrev snap_in_version validate-manifest ../manifest.yaml
```

Catches: invalid YAML, missing required fields, function name mismatches, keyring configuration errors.

---

## 5. Local Development (ngrok Testing Mode)

This deploys the snap-in to your org but routes function execution to your local machine for debugging.

```bash
# Terminal 1: Start ngrok tunnel
ngrok http 8000

# Terminal 2: Create version with testing URL
devrev snap_in_version create-one \
  --manifest ../manifest.yaml \
  --testing-url https://<ngrok-id>.ngrok-free.dev && \
  devrev snap_in draft | jq
```

**Key points:**
- Code changes auto-reload — no need to redeploy for code-only changes
- Manifest changes require an upgrade:
  ```bash
  devrev snap_in_version upgrade \
    --manifest ../manifest.yaml \
    --testing-url <updated-ngrok-url>
  ```
- Force upgrade for breaking manifest changes:
  ```bash
  devrev snap_in_version upgrade --force \
    --manifest ../manifest.yaml \
    --testing-url <updated-ngrok-url>
  ```
- Service account token in payloads is valid for only 30 minutes. Events older than this return 401.
- Access the ngrok inspector UI at `http://127.0.0.1:4040/` to view request payloads and replay events.

---

## 6. Production Build & Deploy

### First deployment:

```bash
# Build and package
cd code
npm run build && npm run package
# This produces build.tar.gz

# Create version with the archive
devrev snap_in_version create-one \
  --manifest ../manifest.yaml \
  --archive build.tar.gz

# Install as draft
devrev snap_in draft | jq
```

### Check version state:

```bash
devrev snap_in_version show
# State should be "ready" — if "build_failed", check logs
```

### Upgrade existing version:

```bash
npm run build && npm run package

devrev snap_in_version upgrade \
  --manifest ../manifest.yaml \
  --archive build.tar.gz

devrev snap_in draft | jq
```

---

## 7. Snap-in Activation

After drafting, configure keyrings and inputs in the DevRev UI, then:

```bash
devrev snap_in update
devrev snap_in activate
```

**Lifecycle**: DRAFT → configure keyrings/inputs → ACTIVATING → ACTIVE (or ERROR)

---

## 8. Chef-CLI for AirSync Mapping

Chef-CLI is the auxiliary CLI for AirSync snap-in recipe development. It handles object/field mapping between the external system and DevRev.

### Install chef-cli

Download the binary for your OS from [devrev/adaas-chef-cli releases](https://github.com/devrev/adaas-chef-cli/releases) and add to your PATH.

### Validate metadata

Validates `external_domain_metadata.json` against the AirSync schema:

```bash
chef-cli validate-metadata < external_domain_metadata.json
```

Run this after every change to the metadata file.

### Generate metadata from example data

```bash
chef-cli generate-metadata --from-examples <directory-of-example-json-files>
```

### Configure mappings (interactive UI)

Launches an interactive web UI to map external objects/fields to DevRev objects/fields. On save, generates `initial_domain_mapping.json`.

```bash
export DEVREV_TOKEN="$(devrev profiles get-token access)" && \
  CTX_ID="$(chef-cli ctx switch --env prod | awk -F: 'NR==1{print $1}')" && \
  eval "$(chef-cli ctx switch --env prod --id "$CTX_ID")" && \
  ACTIVE_PARTITION=dvrv-in-1 chef-cli configure-mappings \
    --env prod --idm initial_domain_mapping.json
```

**What happens:**
1. Chef-CLI reads the metadata and any existing mappings
2. Opens a web UI showing source objects/fields → DevRev objects/fields
3. User maps record types, fields, and enum values
4. On "Export", downloads `initial_domain_mapping.json`
5. This file is required for the ADaaS platform to understand mappings

### MCP integration (experimental)

For AI-assisted mapping creation (works with Cursor, Claude Code):

```bash
chef-cli mcp initial-mapping
```

Then in your AI editor, direct it to create/edit the initial domain mapping file.

### Reverse mappings

For loading (DevRev → external), use the `--reverse` flag:

```bash
chef-cli configure-mappings --env prod --reverse --idm initial_domain_mapping.json
```

---

## 9. Cleanup & Deletion

```bash
# Delete current snap-in version
devrev snap_in_version delete-one

# Only one non-published version per package is allowed
# You must delete the existing version before creating a new one
```

---

## 10. Common Issues & Solutions

| Issue | Cause | Fix |
|-------|-------|-----|
| `snap_in_version is not ready` | Build failed | Check state: `devrev snap_in_version show`. If `build_failed`, check logs |
| `can't create new snap_in_version` | Existing non-published version | Delete existing: `devrev snap_in_version delete-one` |
| 401 Unauthorized in function | Token expired | Service account tokens expire after 30 min. Don't cache tokens across invocations |
| ngrok URL changed | ngrok restarted | Update: `devrev snap_in_version upgrade --testing-url <new-url>` |
| `build.tar.gz` stale | Forgot to rebuild | Always run `npm run build && npm run package` before deploy |
| Keyring not found in payload | Keyring not configured | Configure keyrings in DevRev UI before activating |
| Manifest validation failed | YAML errors or mismatches | Run `devrev snap_in_version validate-manifest` and fix reported issues |
| Chef-CLI mapping fails | Token expired | Re-export: `export DEVREV_TOKEN="$(devrev profiles get-token access)"` |

---

## Complete Workflow Summary

### Simple snap-in:
```bash
devrev profiles authenticate -o <org> -u <email> --expiry 7
devrev snap_in_context set <name>
devrev snap_in_package create-one --slug <name> | jq .
devrev snap_in_version validate-manifest ../manifest.yaml
# Test locally with ngrok, then:
cd code && npm run build && npm run package
devrev snap_in_version create-one --manifest ../manifest.yaml --archive build.tar.gz
devrev snap_in draft | jq
# Configure keyrings/inputs in UI
devrev snap_in update
devrev snap_in activate
```

### AirSync snap-in:
```bash
devrev profiles authenticate -o <org> -u <email> --expiry 7
devrev snap_in_context set <name>
devrev snap_in_package create-one --slug airdrop-<s>-snap-in | jq .
# If OAuth:
echo '{"client_id":"...","client_secret":"..."}' | devrev developer_keyring create oauth-secret <name>
devrev snap_in_version validate-manifest ../manifest.yaml
chef-cli validate-metadata < external_domain_metadata.json
# Test locally, then:
cd code && npm run build && npm run package
devrev snap_in_version create-one --manifest ../manifest.yaml --archive build.tar.gz
devrev snap_in draft | jq
# Configure connection in UI
devrev snap_in update
devrev snap_in activate
# Configure mappings:
export DEVREV_TOKEN="$(devrev profiles get-token access)"
chef-cli configure-mappings --env prod --idm initial_domain_mapping.json
```
