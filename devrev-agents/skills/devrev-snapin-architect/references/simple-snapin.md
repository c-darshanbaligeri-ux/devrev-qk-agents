# Simple Snap-in Development Reference

Based on `devrev/snap-in-examples` repository (15 examples). Simple snap-ins use `@devrev/typescript-sdk`.

---

## Project Structure

```
my-snap-in/
├── manifest.yaml
├── code/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts              # Entry point — routes to function-factory
│   │   ├── function-factory.ts   # Maps function names → handlers
│   │   ├── functions/
│   │   │   └── <function_name>/
│   │   │       └── index.ts      # Business logic
│   │   └── fixtures/
│   │       └── <event>.json      # Test event payloads
│   └── test/
└── README.md
```

## Boilerplate Files

### code/src/index.ts

```typescript
import { functionFactory } from './function-factory';

export const run = async (functionName: string, events: any[]) => {
  const handler = functionFactory[functionName as keyof typeof functionFactory];
  if (!handler) {
    throw new Error(`No handler found for function: ${functionName}`);
  }
  return handler(events);
};
```

### code/src/function-factory.ts

```typescript
import myFunction from './functions/my_function';
import anotherFunction from './functions/another_function';

export const functionFactory = {
  my_function: myFunction,
  another_function: anotherFunction,
} as const;
```

### code/package.json

```json
{
  "name": "snap-in-name",
  "version": "1.0.0",
  "description": "Description",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "package": "tar -czf build.tar.gz dist package.json",
    "start:watch": "ts-node src/test-runner/test-runner.ts"
  },
  "dependencies": {
    "@devrev/typescript-sdk": "latest"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "ts-node": "^10.9.0",
    "@types/node": "^20.0.0"
  }
}
```

### code/tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "test"]
}
```

---

## Complete Manifest Reference

```yaml
version: "2"

name: my-snap-in
description: What this snap-in does

service_account:
  display_name: "My Snap-in Bot"

# --- Functions ---
functions:
  - name: on_work_created
    description: Handles ticket/issue creation events
  - name: on_command
    description: Handles slash command invocations
  - name: on_webhook
    description: Handles external webhook events
  - name: on_timer
    description: Handles scheduled timer events
  - name: on_activate_hook
    description: Runs when snap-in is activated

# --- Automations ---
automations:
  - name: work_creation_handler
    source: devrev-webhook
    event_type: work_created
    function: on_work_created

  - name: timer_handler
    source: timer-event-source
    event_type: timer.tick
    function: on_timer

# --- Event Sources ---
event_sources:
  devrev-webhook:
    display_name: DevRev Events
    type: devrev-webhook
    config:
      event_types:
        - work_created
        - work_updated
        - conversation_created
        - conversation_updated

  external-webhook:
    display_name: External Webhook
    description: Receives webhooks from external system
    type: flow-custom-webhook
    config:
      policy: {}

  timer-event-source:
    display_name: Scheduled Timer
    type: timer-events
    config:
      timer:
        cron: "0 */6 * * *"      # Every 6 hours
        # OR interval: 3600      # Every hour in seconds

# --- Commands ---
commands:
  - name: my-command
    namespace: my-snap-in
    description: Does something useful
    surfaces:
      - surface: discussions
        object_types:
          - conversation
          - ticket
          - issue
    usage_hint: "/my-command [args]"
    function: on_command

# --- Keyring Types ---
keyring_types:
  # Secret (API key, PAT)
  - id: external-api-key
    name: External API Key
    description: API key for external service
    kind: secret
    is_subdomain: false
    external_system_name: ExternalService
    secret_config:
      - id: api_key
        name: API Key
        description: Your API key
        type: text
        is_optional: false
    token_verification:
      url: "https://api.external.com/v1/me"
      method: GET

  # OAuth2
  - id: external-oauth
    name: External OAuth
    description: OAuth2 connection
    kind: oauth2
    external_system_name: ExternalService
    oauth2_config:
      authorize_url: "https://external.com/oauth/authorize"
      token_url: "https://external.com/oauth/token"
      scope: "read write"

# --- Inputs ---
inputs:
  organization:
    - name: slack_channel
      description: Slack channel for notifications
      field_type: text
      is_required: false
      default_value: ""
    - name: enable_auto_reply
      description: Enable automatic replies
      field_type: bool
      is_required: false
      default_value: "false"
    - name: priority_filter
      description: Minimum priority to act on
      field_type: enum
      is_required: false
      default_value: "all"
      allowed_values:
        - "all"
        - "p0"
        - "p1"
        - "p2"

# --- Hooks ---
hooks:
  - type: on_activate
    function: on_activate_hook
  - type: on_deactivate
    function: on_deactivate_hook
  - type: on_update
    function: on_config_update

# --- Snap-kit Actions ---
snap_kit_actions:
  - name: quick_action
    display_name: Quick Action Button
    description: Performs an action from the work item detail view
    function: on_snap_kit_action
    surfaces:
      - surface: work_item_details
```

---

## Event Types (Complete)

| Event | Trigger | Payload Key |
|-------|---------|-------------|
| `work_created` | Ticket, issue, task, opportunity created | `payload.work_created.work` |
| `work_updated` | Work item field changed | `payload.work_updated.work` |
| `work_deleted` | Work item deleted | `payload.work_deleted.work` |
| `conversation_created` | New conversation | `payload.conversation_created.conversation` |
| `conversation_updated` | Conversation updated | `payload.conversation_updated.conversation` |
| `tag_created` | New tag | `payload.tag_created.tag` |
| `part_created` | Product part created | `payload.part_created.part` |
| `part_updated` | Product part updated | `payload.part_updated.part` |
| `account_created` | Account created | `payload.account_created.account` |
| `account_updated` | Account updated | `payload.account_updated.account` |
| `dev_user_created` | New internal user | `payload.dev_user_created.dev_user` |
| `rev_user_created` | New customer user | `payload.rev_user_created.rev_user` |
| `sla_tracker_updated` | SLA status change | `payload.sla_tracker_updated.sla_tracker` |
| `timer_expired` | Scheduled timer fires | `payload.timer_expired` |

---

## Function Patterns

### Base function structure (all types)

```typescript
import { client } from "@devrev/typescript-sdk";

interface EventPayload {
  execution_metadata: {
    devrev_endpoint: string;
    service_account_token: string;
  };
  input_data: {
    keyrings: Record<string, { secret: string }>;
    global_values: Record<string, string>;
  };
  payload: any;
}

export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const devrevClient = client.setup({
      endpoint: event.execution_metadata.devrev_endpoint,
      token: event.execution_metadata.service_account_token,
    });

    const apiKey = event.input_data.keyrings["external-api-key"]?.secret;
    const config = event.input_data.global_values;

    // Business logic here
  }
};

export default run;
```

### Automation: React to DevRev events

```typescript
export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const devrevClient = client.setup({
      endpoint: event.execution_metadata.devrev_endpoint,
      token: event.execution_metadata.service_account_token,
    });

    const work = event.payload.work_created?.work;
    if (!work) continue;

    // Example: Post to Slack on high-priority ticket
    if (work.priority === "p0" || work.priority === "p1") {
      const webhookUrl = event.input_data.global_values["slack_webhook_url"];
      if (webhookUrl) {
        await fetch(webhookUrl, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            text: `🚨 ${work.priority.toUpperCase()} ticket: ${work.display_id} — ${work.title}`,
          }),
        });
      }
    }

    // Example: Auto-assign based on part
    await devrevClient.worksUpdate({
      id: work.id,
      owned_by: { set: ["DEVU-target-user"] },
    });
  }
};
```

### Command: Slash command handler

```typescript
export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const devrevClient = client.setup({
      endpoint: event.execution_metadata.devrev_endpoint,
      token: event.execution_metadata.service_account_token,
    });

    const commandPayload = event.payload;
    const sourceId = commandPayload.source_id;      // Object the command ran on
    const parameters = commandPayload.parameters;    // Command arguments

    // Example: /summarize command adds a summary comment
    await devrevClient.timelineEntriesCreate({
      object: sourceId,
      type: "timeline_comment",
      body: `Summary generated by snap-in: ${parameters?.join(" ") || "N/A"}`,
    });
  }
};
```

### External webhook handler

```typescript
export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const devrevClient = client.setup({
      endpoint: event.execution_metadata.devrev_endpoint,
      token: event.execution_metadata.service_account_token,
    });

    // Webhook payload from external system
    const webhookData = event.payload;

    // Validate webhook signature if supported
    // const signature = webhookData.headers?.["x-webhook-signature"];

    // Map external event to DevRev action
    const externalEvent = webhookData.body;
    if (externalEvent.action === "issue_updated") {
      // Find corresponding DevRev ticket by external ref
      const works = await devrevClient.worksList({
        type: ["ticket"],
      });
      // Update matching ticket...
    }
  }
};
```

### Hook: Lifecycle handler (on_activate)

```typescript
// Use to register external webhooks when snap-in activates
export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const apiKey = event.input_data.keyrings["external-api-key"]?.secret;
    const webhookUrl = event.payload.event_source_url; // DevRev provides callback URL

    // Register webhook with external system
    await fetch("https://api.external.com/webhooks", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        url: webhookUrl,
        events: ["issue.updated", "issue.created"],
      }),
    });
  }
};
```

### Timer: Scheduled execution

```typescript
// Runs on cron schedule — good for polling external systems
export const run = async (events: EventPayload[]) => {
  for (const event of events) {
    const devrevClient = client.setup({
      endpoint: event.execution_metadata.devrev_endpoint,
      token: event.execution_metadata.service_account_token,
    });

    // Poll external system for updates since last check
    const apiKey = event.input_data.keyrings["external-api-key"]?.secret;
    const response = await fetch("https://api.external.com/updates?since=last_hour", {
      headers: { "Authorization": `Bearer ${apiKey}` },
    });
    const updates = await response.json();

    // Process each update
    for (const update of updates.items) {
      await devrevClient.worksCreate({
        type: "ticket",
        title: update.title,
        body: update.description,
      });
    }
  }
};
```

---

## Testing Locally

```bash
# Run with fixture event
npm install
npm run start:watch -- --functionName=my_function --fixturePath=my_event.json

# Or with ngrok for live events
ngrok http 8000
devrev snap_in_version create-one --manifest ../manifest.yaml \
  --testing-url https://<ngrok-url> && devrev snap_in draft | jq
```

### Fixture event structure

```json
{
  "execution_metadata": {
    "devrev_endpoint": "https://api.devrev.ai",
    "service_account_token": "your-test-pat-token"
  },
  "input_data": {
    "keyrings": {
      "external-api-key": {
        "secret": "test-api-key-value"
      }
    },
    "global_values": {
      "slack_webhook_url": "https://hooks.slack.com/test"
    }
  },
  "payload": {
    "work_created": {
      "work": {
        "id": "don:core:dvrv-us-1:devo/test:ticket/1",
        "display_id": "TKT-1",
        "title": "Test ticket",
        "priority": "p0",
        "type": "ticket"
      }
    }
  }
}
```
