# AirSync Snap-in Template (Generic)

> **Source**: Stripped from [devrev/airdrop-asana-snap-in](https://github.com/devrev/airdrop-asana-snap-in) and cross-referenced with a production Zoho CRM connector.

---

## Build Method: Clone-and-Rename (Primary)

Instead of writing 20+ files from scratch, clone the production Asana snap-in and adapt it. This keeps all config, boilerplate, SDK patterns, and test infrastructure identical across every AirSync connector.

### Step 1: Scaffold from template

```bash
# Clone the production Asana snap-in as template
git clone --depth 1 https://github.com/devrev/airdrop-asana-snap-in.git airdrop-<system>-snap-in
cd airdrop-<system>-snap-in

# Remove template-specific files that don't belong in new snap-ins
rm -rf .git .circleci .github dummy_data_generator

# Rename the system-specific folder
mv code/src/functions/asana code/src/functions/<system>
```

### Step 2: Find-and-replace "asana" references

Apply these replacements across the entire project:

| Pattern | Replace with | Files affected |
|---------|-------------|----------------|
| `asana` (in slugs, paths, imports) | `<system>` | manifest.yaml, package.json, all .ts files |
| `Asana` (in display names) | `<System>` (capitalized) | manifest.yaml, README.md, package.json |
| `AsanaClient` (class name) | `<System>Client` | client.ts, all worker .ts files that import it |
| `asanaClient` (variable name) | `<system>Client` | all worker .ts files |
| `airdrop-asana-snap-in` (project name) | `airdrop-<system>-snap-in` | package.json, manifest.yaml |
| `airdrop-asana-extractor` (slug) | `airdrop-<system>-extractor` | manifest.yaml |

### Step 3: File classification — what to keep, rename, or rewrite

After the find-and-replace, files fall into three categories:

**KEEP AS-IS (23 files, 56%)** — zero changes needed:
- All config: `tsconfig.json`, `.npmrc`, `.prettierrc`, `.prettierignore`, `babel.config.js`, `jest.config.js`, `tsconfig.eslint.json`, `nodemon.json`
- SDK boilerplate: `src/index.ts`, `src/function-factory.ts`, `src/main.ts`, `src/test-runner/test-runner.ts`
- Loading orchestrator: `loading/index.ts`
- Generic workers: `extraction/workers/attachments-extraction.ts`
- Infrastructure: `Makefile`, `.env.example`, `.devrev/repo.yml`, deploy/cleanup scripts
- Test framework: `code/test/` directory (all files)

**ALREADY HANDLED BY FIND-AND-REPLACE (9 files, 22%)**:
- `package.json` — project name/description swapped
- `README.md` — system name swapped (content update recommended)
- `src/fixtures/positive-case.json` — references updated
- `loading/workers/load-attachments.ts` — import paths updated
- `extraction/index.ts` — import path `../asana/` → `../<system>/`
- `extraction/workers/metadata-extraction.ts` — import path `../../asana/` → `../../<system>/`
- `.github/CODEOWNERS` — removed in Step 1

**REWRITE WITH SYSTEM-SPECIFIC LOGIC (9 files, 22%)** — use research from Phase 2:
1. `functions/<system>/client.ts` — API base URL, endpoints, auth, pagination
2. `functions/<system>/data-normalization.ts` — external fields → normalized schema
3. `functions/<system>/data-denormalization.ts` — normalized → external API payload
4. `functions/<system>/external_domain_metadata.json` — external system schema
5. `functions/<system>/initial_domain_mapping.json` — external ↔ DevRev field mappings
6. `extraction/workers/external-sync-units-extraction.ts` — fetch containers/projects
7. `extraction/workers/data-extraction.ts` — extract items with pagination
8. `loading/workers/load-data.ts` — create/update items in external system
9. `manifest.yaml` — auth/keyring config, connection endpoints

### Step 4: Verify the scaffold builds

```bash
cd code && npm install && npm audit && npm run build
```

Fix any TypeScript compilation errors before moving on.

### When to use the reference code below

When rewriting the 10 system-specific files, use the **TODO-annotated templates below** as your guide. They show the exact SDK patterns, function signatures, and data structures expected by the Airdrop platform.

---

## Project Structure

```
manifest.yaml
code/
  .gitignore
  .npmrc
  .prettierrc
  .prettierignore
  babel.config.js
  jest.config.js
  nodemon.json
  package.json
  tsconfig.json
  src/
    index.ts
    function-factory.ts
    main.ts
    test-runner/
      test-runner.ts
    fixtures/
      positive-case.json
    functions/
      external-system/           # <-- All external-system-specific code lives here
        client.ts                # API client class
        data-normalization.ts    # Normalize extracted data
        data-denormalization.ts  # Denormalize for loading
        external_domain_metadata.json
        initial_domain_mapping.json
      extraction/
        index.ts                 # spawn() entry point
        workers/
          external-sync-units-extraction.ts
          metadata-extraction.ts
          data-extraction.ts
          attachments-extraction.ts
      loading/
        index.ts                 # spawn() entry point
        workers/
          load-data.ts
          load-attachments.ts
```

---

## Critical SDK Patterns

### 1. `spawn()` — Event Routing (NEVER use manual switch/case)

The SDK auto-routes events to worker files by filesystem convention:
- `workers/external-sync-units-extraction.ts` handles `ExtractionExternalSyncUnitsStart`
- `workers/metadata-extraction.ts` handles `ExtractionMetadataStart`
- `workers/data-extraction.ts` handles `ExtractionDataStart` + `ExtractionDataContinue`
- `workers/attachments-extraction.ts` handles `ExtractionAttachmentsStart` + `ExtractionAttachmentsContinue`

### 2. `processTask({ task, onTimeout })` — Every worker uses this

The SDK manages the 10-minute soft timeout and 13-minute hard timeout.

### 3. `adapter.postState()` — MUST be called in `onTimeout` before emitting progress

### 4. `adapter.streamAttachments()` — SDK built-in attachment streaming

### 5. Metadata goes to repo — NOT in emit payload

```typescript
adapter.getRepo('external_domain_metadata')?.push([metadata]);
await adapter.emit(ExtractorEventType.ExtractionMetadataDone);
// NOT: await adapter.emit(ExtractorEventType.ExtractionMetadataDone, { metadata });
```

### 6. `connection_data.key` is JSON — MUST `JSON.parse()`

```typescript
const connectionKey = JSON.parse(event.payload.connection_data.key);
const accessToken = connectionKey.access_token;
// Also available: connectionKey.org_id, connectionKey.region, etc.
```

---

## File Contents

### `manifest.yaml`

```yaml
version: "2"

name: Airdrop External System  # TODO: Replace with your system name
description: An Airdrop snap-in for syncing data between External System and DevRev  # TODO

service_account:
  display_name: External System Bot  # TODO

functions:
  - name: extraction
    description: Airdrop extraction function for External System  # TODO
  - name: loading
    description: Airdrop loading function for External System  # TODO

keyring_types:
  - id: external-system-connection  # TODO: unique ID for your connection
    name: External System Connection  # TODO
    description: Connection details for authenticating with External System  # TODO
    kind: "Oauth2"
    external_system_name: External System  # TODO
    scopes:  # TODO: Define scopes for your external system
      - name: scope-name
        description: Description of what this scope allows
        value: "scope.value"
    scope_delimiter: " "

    # --- PAT-based auth (simple) ---
    # Use this if the external system uses personal access tokens.
    # authorize:
    #   token_verification_url: "https://api.external-system.com/verify"
    #   token_verification_method: "GET"
    #   token_verification_headers:
    #     Authorization: "Bearer [ACCESS_TOKEN]"

    # --- Config-type OAuth (client_credentials) ---
    # Use this for systems like Zoho that use server-to-server OAuth.
    # authorize:
    #   type: "config"
    #   token_url: "https://accounts.external-system[REGION]/oauth/v2/token"
    #   grant_type: "client_credentials"
    #   token_headers:
    #     "Content-type": "application/x-www-form-urlencoded"
    #   token_body: "client_id=[CLIENT_ID]&scope=[SCOPES]&client_secret=[CLIENT_SECRET]&grant_type=client_credentials&soid=[ORG_ID]"
    #   fields:
    #     - id: org_id
    #       name: Organization ID
    #       description: Organization ID
    #       input_type: "text"
    #       include_with_secret: true
    #     - id: region
    #       name: Region/Domain
    #       description: Data center region or domain suffix
    #       input_type: "text"
    #       include_with_secret: true

    # organization_data:
    #   type: "config"
    #   url: "https://api.external-system[REGION]/v2/org"
    #   method: "GET"
    #   headers:
    #     "Authorization": "Bearer [ACCESS_TOKEN]"
    #   response_jq: '{\"id\": .org.id, \"name\": .org.name}'

imports:
  - slug: airdrop-external-system-snap-in  # TODO
    display_name: External System  # TODO
    description: Import data from External System using Airdrop  # TODO
    extractor_function: extraction
    allowed_connection_types:
      - external-system-connection  # TODO: must match keyring_types.id
    # capabilities:
    #   - TIME_SCOPED_SYNCS  # Uncomment if your system supports incremental sync
```

### `code/package.json`

```json
{
  "name": "airdrop-external-system-snap-in",
  "version": "1.0.0",
  "description": "DevRev External System Connector",
  "main": "./dist/index.js",
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint --fix -- .",
    "build": "rimraf ./dist && tsc",
    "build:watch": "tsc --watch",
    "prepackage": "npm run build",
    "package": "tar -cvzf build.tar.gz dist package.json package-lock.json .npmrc",
    "start": "ts-node ./src/main.ts",
    "start:watch": "nodemon ./src/main.ts",
    "start:production": "node dist/main.js",
    "test:server": "nodemon --watch src --watch test test/main.ts",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.26.10",
    "@babel/preset-env": "^7.20.2",
    "@babel/preset-typescript": "^7.18.6",
    "@types/jest": "^29.4.0",
    "@types/node": "^18.13.0",
    "@types/yargs": "^17.0.24",
    "@typescript-eslint/eslint-plugin": "^8.32.0",
    "@typescript-eslint/parser": "^8.32.0",
    "babel-jest": "^29.4.2",
    "dotenv": "^16.0.3",
    "eslint": "^9.26.0",
    "eslint-config-prettier": "^9.0.0",
    "eslint-plugin-import": "^2.28.1",
    "eslint-plugin-prettier": "4.0.0",
    "eslint-plugin-simple-import-sort": "^10.0.0",
    "eslint-plugin-sort-keys-fix": "^1.1.2",
    "eslint-plugin-unused-imports": "^4.1.4",
    "jest": "^29.4.2",
    "nodemon": "^3.0.3",
    "prettier": "^2.8.3",
    "prettier-plugin-organize-imports": "^3.2.2",
    "rimraf": "^4.1.2",
    "ts-jest": "^29.0.5",
    "ts-node": "^10.9.1",
    "typescript": "^4.9.5",
    "yargs": "^17.6.2"
  },
  "dependencies": {
    "@devrev/ts-adaas": "1.17.0",
    "axios": "^1.13.5",
    "dotenv": "^16.0.3",
    "js-jsonl": "^1.1.1",
    "p-limit": "3.1.0",
    "yargs": "^17.6.2"
  }
}
```

### `code/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "es2017",
    "module": "commonjs",
    "baseUrl": "./",
    "paths": {
      "*": ["./src/*"]
    },
    "outDir": "./dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "declaration": true,
    "resolveJsonModule": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### `code/babel.config.js`

```js
module.exports = {
  presets: [['@babel/preset-env', { targets: { node: 'current' } }], '@babel/preset-typescript'],
};
```

### `code/jest.config.js`

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```

### `code/nodemon.json`

```json
{
  "execMap": {
    "ts": "ts-node"
  },
  "watch": ["src"]
}
```

### `code/.prettierrc`

```json
{
  "printWidth": 120,
  "tabWidth": 2,
  "useTabs": false,
  "singleQuote": true,
  "jsxSingleQuote": false,
  "semi": true,
  "trailingComma": "es5",
  "arrowParens": "always",
  "endOfLine": "lf",
  "htmlWhitespaceSensitivity": "strict"
}
```

### `code/.npmrc`

```
save-exact=true
```

### `code/.gitignore`

```
node_modules
dist
.env
extracted_files
```

### `code/src/index.ts`

```typescript
export * from './function-factory';
```

### `code/src/function-factory.ts`

```typescript
import extraction from './functions/extraction';
import loading from './functions/loading';

export const functionFactory = {
  extraction,
  loading,
} as const;

export type FunctionFactoryType = keyof typeof functionFactory;
```

### `code/src/main.ts`

```typescript
import yargs from 'yargs';
import { hideBin } from 'yargs/helpers';
import { FunctionFactoryType } from './function-factory';
import { testRunner } from './test-runner/test-runner';

(async () => {
  const argv = await yargs(hideBin(process.argv)).options({
    fixturePath: {
      type: 'string',
      require: true,
    },
    functionName: {
      type: 'string',
      require: true,
    },
  }).argv;

  if (!argv.fixturePath || !argv.functionName) {
    console.error('Please make sure you have passed fixturePath & functionName');
  }

  await testRunner({
    fixturePath: argv.fixturePath,
    functionName: argv.functionName as FunctionFactoryType,
  });
})();
```

### `code/src/test-runner/test-runner.ts`

```typescript
import dotenv from 'dotenv';
import { AirdropEvent } from '@devrev/ts-adaas';
import { functionFactory, FunctionFactoryType } from '../function-factory';

interface TestRunnerProps {
  fixturePath: string;
  functionName: FunctionFactoryType;
}

function addCredentials(events: AirdropEvent[], token: string): AirdropEvent[] {
  return events.map((event) => ({
    ...event,
    context: {
      ...event.context,
      secrets: {
        ...event.context.secrets,
        service_account_token: token,
      },
    },
  }));
}

export async function testRunner({ fixturePath, functionName }: TestRunnerProps) {
  const env = dotenv.config();

  if (!functionFactory[functionName]) {
    console.error(
      `Function ${functionName} not found. Make sure it is registered in function-factory.ts`
    );
    return;
  }

  const fixtures = await import(`../../${fixturePath}`);
  const events: AirdropEvent[] = fixtures.default || fixtures;

  if (env.parsed) {
    const token = env.parsed.SERVICE_ACCOUNT_TOKEN || '';
    await functionFactory[functionName](addCredentials(events, token));
  } else {
    await functionFactory[functionName](events);
  }
}
```

---

## Extraction Function Files

### `code/src/functions/extraction/index.ts`

> **CRITICAL**: Uses `spawn()` — the SDK auto-routes events to workers by convention. NEVER use manual switch/case.

```typescript
import { AirdropEvent, spawn } from '@devrev/ts-adaas';
import initialDomainMapping from '../external-system/initial_domain_mapping.json';

// TODO: Define your extractor state. Each entity type should track pagination
// and completion status. Example:
//
// interface EntityState {
//   offset?: string;
//   completed: boolean;
//   modifiedSince?: string;
// }
export interface ExtractorState {
  // TODO: Add one field per entity type you're extracting.
  // Example:
  // users: EntityState;
  // tickets: EntityState;
  // contacts: EntityState;
  lastSuccessfulSyncStarted?: string;
}

export const initialState: ExtractorState = {
  // TODO: Initialize each entity state.
  // Example:
  // users: { completed: false },
  // tickets: { completed: false },
  // contacts: { completed: false },
};

function getWorkerPerExtractionPhase(event: AirdropEvent) {
  // The SDK resolves worker paths by convention from __dirname + '/workers/'.
  // Return the path to the appropriate worker file.
  // The SDK routes based on event_type:
  //   ExtractionExternalSyncUnitsStart -> external-sync-units-extraction
  //   ExtractionMetadataStart          -> metadata-extraction
  //   ExtractionDataStart/Continue     -> data-extraction
  //   ExtractionAttachmentsStart/Continue -> attachments-extraction
  return __dirname;
}

const run = async (events: AirdropEvent[]) => {
  for (const event of events) {
    await spawn<ExtractorState>({
      event,
      initialState,
      initialDomainMapping,
      // Use baseWorkerPath to let the SDK auto-route:
      baseWorkerPath: getWorkerPerExtractionPhase(event),
    });
  }
};

export default run;
```

### `code/src/functions/extraction/workers/external-sync-units-extraction.ts`

```typescript
import { ExternalSyncUnit, ExtractorEventType, processTask } from '@devrev/ts-adaas';

// TODO: Import your API client
// import { ExternalSystemClient } from '../../external-system/client';

processTask({
  task: async ({ adapter }) => {
    // TODO: Initialize your API client using adapter.event
    // const client = new ExternalSystemClient(adapter.event);

    try {
      // TODO: Fetch the list of sync units (projects, workspaces, etc.) from
      // the external system and transform them into ExternalSyncUnit format.
      //
      // Example:
      // const response = await client.getProjects();
      // const externalSyncUnits: ExternalSyncUnit[] = response.map((project) => ({
      //   id: project.id,
      //   name: project.name,
      //   description: project.description,
      //   item_type: 'tasks',
      //   item_count: project.taskCount,
      // }));

      const externalSyncUnits: ExternalSyncUnit[] = [];  // TODO: Replace

      await adapter.emit(ExtractorEventType.ExtractionExternalSyncUnitsDone, {
        external_sync_units: externalSyncUnits,
      });
    } catch (error) {
      await adapter.emit(ExtractorEventType.ExtractionExternalSyncUnitsError, {
        error: {
          message: `Failed to extract external sync units: ${error}`,
        },
      });
    }
  },
  onTimeout: async ({ adapter }) => {
    await adapter.emit(ExtractorEventType.ExtractionExternalSyncUnitsError, {
      error: {
        message: 'Failed to extract external sync units. Lambda timeout.',
      },
    });
  },
});
```

### `code/src/functions/extraction/workers/metadata-extraction.ts`

> **CRITICAL**: Metadata is pushed to the repo, NOT passed in the emit payload.

```typescript
import { ExtractorEventType, processTask } from '@devrev/ts-adaas';
import externalDomainMetadata from '../../external-system/external_domain_metadata.json';

const repos = [
  {
    itemType: 'external_domain_metadata',
  },
];

processTask({
  task: async ({ adapter }) => {
    adapter.initializeRepos(repos);

    // Push metadata to the repo — this is the ONLY correct way to pass metadata.
    await adapter.getRepo('external_domain_metadata')?.push([externalDomainMetadata]);

    await adapter.emit(ExtractorEventType.ExtractionMetadataDone);
  },
  onTimeout: async ({ adapter }) => {
    await adapter.emit(ExtractorEventType.ExtractionMetadataError, {
      error: { message: 'Failed to extract metadata. Lambda timeout.' },
    });
  },
});
```

### `code/src/functions/extraction/workers/data-extraction.ts`

> **CRITICAL**: Uses `processTask<ExtractorState>`, handles pagination via state, calls `adapter.emit(ExtractionDataProgress)` in `onTimeout`.

```typescript
import {
  axios,
  ErrorRecord,
  EventType,
  ExtractorEventType,
  processTask,
  serializeAxiosError,
  SyncMode,
  WorkerAdapter,
} from '@devrev/ts-adaas';

// TODO: Import your API client and normalizers
// import { ExternalSystemClient } from '../../external-system/client';
// import { normalizeItem, normalizeUser } from '../../external-system/data-normalization';
import { ExtractorState, initialState } from '../index';

interface ExtractListResponse {
  delay?: number;
  error?: ErrorRecord;
}

// Define repos with their normalizer functions.
// The normalize function is called automatically by the SDK when pushing data.
const repos = [
  // TODO: Add one entry per entity type with its normalizer.
  // Example:
  // { itemType: 'users', normalize: normalizeUser },
  // { itemType: 'tickets', normalize: normalizeTicket },
  // { itemType: 'attachments', normalize: normalizeAttachment },
];

// TODO: Define extraction functions for each entity type.
// Each function should:
//   1. Paginate through the API
//   2. Push data to the appropriate repo
//   3. Update adapter.state for resume support
//   4. Return { delay } for rate limits or { error } for failures

// Example extraction function pattern:
//
// async function extractUsers(
//   client: ExternalSystemClient,
//   adapter: WorkerAdapter<ExtractorState>
// ): Promise<ExtractListResponse> {
//   try {
//     let hasNextPage = true;
//     while (hasNextPage) {
//       const response = await client.getUsers({
//         offset: adapter.state.users.offset,
//       });
//       const users = response.data || [];
//       await adapter.getRepo('users')?.push(users);
//
//       if (response.nextOffset) {
//         adapter.state.users.offset = response.nextOffset;
//       } else {
//         hasNextPage = false;
//       }
//     }
//     return {};
//   } catch (error) {
//     return handleExtractionError(error);
//   }
// }

interface ItemTypeToExtract {
  name: string;
  extractFunction: (...params: any[]) => Promise<ExtractListResponse>;
}

// TODO: List all entity types to extract in order.
const itemTypesToExtract: ItemTypeToExtract[] = [
  // { name: 'users', extractFunction: extractUsers },
  // { name: 'tickets', extractFunction: extractTickets },
];

processTask<ExtractorState>({
  task: async ({ adapter }) => {
    adapter.initializeRepos(repos);

    // TODO: Initialize your API client
    // const client = new ExternalSystemClient(adapter.event);

    const { reset_extract_from, extract_from } = adapter.event.payload.event_context;

    // Handle sync start — set up timestamps for incremental sync.
    if (adapter.event.payload.event_type === EventType.ExtractionDataStart) {
      if (extract_from) {
        console.log(`Starting extraction from given timestamp: ${extract_from}.`);
        // TODO: Set modifiedSince on relevant entity states
        // adapter.state.tickets.modifiedSince = extract_from;
      }

      if (adapter.event.payload.event_context.mode === SyncMode.INCREMENTAL) {
        // Reset entity states for incremental sync
        // TODO: Reset each entity state
        // adapter.state.users = initialState.users;
        // adapter.state.tickets = initialState.tickets;

        if (reset_extract_from) {
          console.log(`reset_extract_from is true. Starting from provided timestamp or beginning.`);
        } else {
          console.log(`Starting from lastSuccessfulSyncStarted: ${adapter.state.lastSuccessfulSyncStarted}`);
          // TODO: Set modifiedSince from lastSuccessfulSyncStarted
          // adapter.state.tickets.modifiedSince = adapter.state.lastSuccessfulSyncStarted;
        }
      }
    }

    // Extract each entity type in order, respecting completion state.
    for (const itemType of itemTypesToExtract) {
      if ((adapter.state as any)[itemType.name]?.completed) {
        console.log(`Skipping ${itemType.name} — already completed.`);
        continue;
      }

      // TODO: Pass your client to the extract function
      // const { delay, error } = await itemType.extractFunction(client, adapter);
      const { delay, error } = { delay: undefined, error: undefined } as ExtractListResponse; // TODO: Replace

      if (delay) {
        await adapter.emit(ExtractorEventType.ExtractionDataDelay, { delay });
        return;
      } else if (error) {
        await adapter.emit(ExtractorEventType.ExtractionDataError, { error });
        return;
      } else {
        (adapter.state as any)[itemType.name].completed = true;
        console.log(`Finished extracting ${itemType.name}.`);
      }
    }

    await adapter.emit(ExtractorEventType.ExtractionDataDone);
  },

  onTimeout: async ({ adapter }) => {
    // CRITICAL: Emit progress on timeout so the SDK can resume from current state.
    await adapter.emit(ExtractorEventType.ExtractionDataProgress);
  },
});

// Standard error handler for extraction — handles rate limits and axios errors.
function handleExtractionError(error: unknown): ExtractListResponse {
  if (axios.isAxiosError(error)) {
    if (error.response?.status === 429) {
      const retryAfter = Number(error.response.headers['retry-after'] || 60);
      console.log(`Rate limit hit. Retrying after ${retryAfter} seconds.`);
      return { delay: retryAfter };
    } else {
      console.error('Extraction error:', serializeAxiosError(error));
      return { error: { message: JSON.stringify(error) } };
    }
  } else {
    console.error('Extraction error:', error);
    return { error: { message: JSON.stringify(error) } };
  }
}
```

### `code/src/functions/extraction/workers/attachments-extraction.ts`

> **CRITICAL**: Uses `adapter.streamAttachments()` — NEVER download manually.

```typescript
import {
  axios,
  ExtractorEventType,
  processTask,
  serializeAxiosError,
} from '@devrev/ts-adaas';

// TODO: Import your API client or use axios directly for attachment downloads.
// Attachment streaming requires returning an HTTP stream from the external system.

const BATCH_SIZE_ATTACHMENTS = 50;

// This function is called by adapter.streamAttachments() for each attachment item.
// It must return { httpStream } with an axios response in stream mode.
const getFileStream = async ({ item, event }: { item: { id: string; url: string }; event: any }) => {
  try {
    // TODO: Configure authentication headers for your external system.
    // Example for Bearer token:
    const connectionKey = JSON.parse(event.payload.connection_data.key);
    const accessToken = connectionKey.access_token;

    const response = await axios.get(item.url, {
      headers: {
        Authorization: `Bearer ${accessToken}`,  // TODO: Adjust auth header format
      },
      responseType: 'stream',
    });

    return { httpStream: response };
  } catch (error) {
    if (axios.isAxiosError(error)) {
      console.warn(`Error fetching attachment ${item.id}:`, serializeAxiosError(error));
    } else {
      console.warn(`Error fetching attachment ${item.id}:`, error);
    }
    return { error: { message: `Failed to fetch attachment ${item.id}` } };
  }
};

processTask({
  task: async ({ adapter }) => {
    try {
      // adapter.streamAttachments() handles batching, retries, and progress tracking.
      const response = await adapter.streamAttachments({
        stream: getFileStream,
        batchSize: BATCH_SIZE_ATTACHMENTS,
      });

      if (response.delay) {
        await adapter.emit(ExtractorEventType.ExtractionAttachmentsDelay, {
          delay: response.delay,
        });
      } else if (response.error) {
        await adapter.emit(ExtractorEventType.ExtractionAttachmentsError, {
          error: response.error,
        });
      } else {
        await adapter.emit(ExtractorEventType.ExtractionAttachmentsDone);
      }
    } catch (error) {
      await adapter.emit(ExtractorEventType.ExtractionAttachmentsError, {
        error: { message: `Attachment extraction failed: ${error}` },
      });
    }
  },
  onTimeout: async ({ adapter }) => {
    // Emit progress so the SDK can resume attachment streaming.
    await adapter.emit(ExtractorEventType.ExtractionAttachmentsProgress);
  },
});
```

---

## Loading Function Files

### `code/src/functions/loading/index.ts`

```typescript
import { AirdropEvent, EventType, spawn } from '@devrev/ts-adaas';
import initialDomainMapping from '../external-system/initial_domain_mapping.json';

export type LoaderState = {};

function getWorkerPerLoadingPhase(event: AirdropEvent) {
  let path;
  switch (event.payload.event_type) {
    case EventType.StartLoadingData:
    case EventType.ContinueLoadingData:
      path = __dirname + '/workers/load-data';
      break;
    case EventType.StartLoadingAttachments:
    case EventType.ContinueLoadingAttachments:
      path = __dirname + '/workers/load-attachments';
      break;
  }
  return path;
}

const run = async (events: AirdropEvent[]) => {
  for (const event of events) {
    const file = getWorkerPerLoadingPhase(event);
    await spawn<LoaderState>({
      event,
      initialState: {},
      workerPath: file,
      initialDomainMapping,
    });
  }
};

export default run;
```

### `code/src/functions/loading/workers/load-data.ts`

```typescript
import { LoaderEventType, processTask } from '@devrev/ts-adaas';
import {
  ExternalSystemItem,
  ExternalSystemItemLoadingParams,
  ExternalSystemItemLoadingResponse,
} from '@devrev/ts-adaas';

// TODO: Import your API client and denormalization functions
// import { ExternalSystemClient } from '../../external-system/client';
// import { denormalizeItem } from '../../external-system/data-denormalization';

// TODO: Implement create function for each loadable entity type.
async function createItem({
  item,
  mappers,
  event,
}: ExternalSystemItemLoadingParams<ExternalSystemItem>): Promise<ExternalSystemItemLoadingResponse> {
  // TODO: Initialize client
  // const client = new ExternalSystemClient(event);
  // const denormalized = denormalizeItem(item);

  try {
    // TODO: Create the item in the external system
    // const response = await client.createItem(denormalized);
    // return { id: response.id };
    return { error: 'Not implemented' };  // TODO: Replace
  } catch (error: any) {
    console.log('Could not create item:', error);
    return { error: 'Could not create item.' };
  }
}

// TODO: Implement update function for each loadable entity type.
async function updateItem({
  item,
  mappers,
  event,
}: ExternalSystemItemLoadingParams<ExternalSystemItem>): Promise<ExternalSystemItemLoadingResponse> {
  // TODO: Initialize client
  // const client = new ExternalSystemClient(event);
  // const itemId = item.id.external as string;
  // const denormalized = denormalizeItem(item);

  try {
    // TODO: Update the item in the external system
    // const response = await client.updateItem(itemId, denormalized);
    // return { id: response.id };
    return { error: 'Not implemented' };  // TODO: Replace
  } catch (error: any) {
    console.log('Could not update item:', error);
    return { error: 'Could not update item.' };
  }
}

processTask({
  task: async ({ adapter }) => {
    const { reports, processed_files } = await adapter.loadItemTypes({
      itemTypesToLoad: [
        // TODO: Add one entry per loadable entity type.
        // {
        //   itemType: 'tasks',
        //   create: createItem,
        //   update: updateItem,
        // },
      ],
    });

    await adapter.emit(LoaderEventType.DataLoadingDone, {
      reports,
      processed_files,
    });
  },
  onTimeout: async ({ adapter }) => {
    // CRITICAL: postState() before emitting progress on timeout.
    await adapter.postState();
    await adapter.emit(LoaderEventType.DataLoadingProgress, {
      reports: adapter.reports,
      processed_files: adapter.processedFiles,
    });
  },
});
```

### `code/src/functions/loading/workers/load-attachments.ts`

```typescript
import {
  axios,
  ExternalSystemAttachment,
  ExternalSystemItemLoadingParams,
  LoaderEventType,
  processTask,
  serializeAxiosError,
} from '@devrev/ts-adaas';

// TODO: Import your API client
// import { ExternalSystemClient } from '../../external-system/client';

const create = async ({ item, mappers, event }: ExternalSystemItemLoadingParams<ExternalSystemAttachment>) => {
  // TODO: Initialize client
  // const client = new ExternalSystemClient(event);

  try {
    // Use mappers to find the parent record in the external system.
    const parentRecord = await mappers.getByTargetId({
      sync_unit: event.payload.event_context.sync_unit,
      target: item.parent_reference_id,
    });

    const externalParentId = parentRecord.data.sync_mapper_record?.external_ids?.[0];

    if (externalParentId) {
      // TODO: Create the attachment in the external system
      // await client.createAttachment(item, externalParentId);
    } else {
      console.warn('Attachment has no parent_id:', item);
      return { error: 'Attachment has no parent_id.' };
    }
  } catch (error) {
    if (axios.isAxiosError(error)) {
      if (error.response?.status === 429) {
        console.log(`Rate limit hit. Retrying after ${error.response.headers['Retry-After']} seconds.`);
        return { delay: Number(error.response.headers['Retry-After']) };
      } else {
        console.error('Error creating attachment:', serializeAxiosError(error));
        return { error: JSON.stringify(error) };
      }
    } else {
      console.error('Unexpected error creating attachment:', error);
      return { error: JSON.stringify(error) };
    }
  }

  return { id: item.reference_id };
};

processTask({
  task: async ({ adapter }) => {
    const { reports, processed_files } = await adapter.loadAttachments({
      create,
    });

    await adapter.emit(LoaderEventType.AttachmentLoadingDone, {
      reports,
      processed_files,
    });
  },
  onTimeout: async ({ adapter }) => {
    await adapter.postState();
    await adapter.emit(LoaderEventType.AttachmentLoadingProgress, {
      reports: adapter.reports,
      processed_files: adapter.processedFiles,
    });
  },
});
```

---

## External System Files (TODO: Implement for your system)

### `code/src/functions/external-system/client.ts`

```typescript
import { AirdropEvent } from '@devrev/ts-adaas';
import axios, { AxiosInstance } from 'axios';

// TODO: Replace with your external system's base URL.
const BASE_URL = 'https://api.external-system.com';

export class ExternalSystemClient {
  private client: AxiosInstance;
  private accessToken: string;

  constructor(event: AirdropEvent) {
    // Parse connection_data.key — it's always JSON.
    const connectionKey = JSON.parse(event.payload.connection_data.key);
    this.accessToken = connectionKey.access_token;
    // TODO: Extract other fields as needed (e.g., region, org_id)

    this.client = axios.create({
      baseURL: BASE_URL,
      headers: {
        Authorization: `Bearer ${this.accessToken}`,  // TODO: Adjust auth format
        'Content-Type': 'application/json',
      },
    });

    // TODO: Add retry logic if needed:
    // import axiosRetry from 'axios-retry';
    // axiosRetry(this.client, { retries: 3, retryDelay: axiosRetry.exponentialDelay });

    // TODO: Add rate limiting if needed:
    // import pLimit from 'p-limit';
    // this.limiter = pLimit(1);  // Serial execution
  }

  // TODO: Add API methods for your external system. Examples:
  //
  // async getProjects() {
  //   return this.client.get('/projects');
  // }
  //
  // async getUsers(params?: { offset?: string }) {
  //   return this.client.get('/users', { params });
  // }
  //
  // async getTasks(params?: { offset?: string; modified_since?: string }) {
  //   return this.client.get('/tasks', { params });
  // }
  //
  // async createTask(data: any) {
  //   return this.client.post('/tasks', { data });
  // }
  //
  // async updateTask(taskId: string, data: any) {
  //   return this.client.put(`/tasks/${taskId}`, { data });
  // }
}
```

### `code/src/functions/external-system/data-normalization.ts`

```typescript
import { NormalizedAttachment, NormalizedItem } from '@devrev/ts-adaas';

// Normalization functions transform extracted data into the format expected
// by the DevRev Airdrop SDK. Only include fields that need to be imported.
// Timestamps must be RFC3339 format.

export function formatNormalizeTime(dateStr: string): string {
  const date = new Date(dateStr);
  return date.toISOString();
}

// TODO: Add one normalizer per entity type. Examples:

// export function normalizeUser(item: any): NormalizedItem {
//   return {
//     id: item.id,
//     created_date: formatNormalizeTime(item.created_at),
//     modified_date: formatNormalizeTime(item.updated_at),
//     data: {
//       email: item?.email || null,
//       name: item?.name || null,
//     },
//   };
// }

// export function normalizeTicket(item: any): NormalizedItem {
//   return {
//     id: item.id,
//     created_date: formatNormalizeTime(item.created_at),
//     modified_date: formatNormalizeTime(item.updated_at),
//     data: {
//       title: item?.subject || null,
//       description: item?.description ? [item.description] : null,
//       status: item?.status || null,
//       priority: item?.priority || null,
//       assignee: item?.assignee_id || null,
//     },
//   };
// }

// export function normalizeAttachment(item: any): NormalizedAttachment {
//   return {
//     id: item.id,
//     url: item.download_url,
//     file_name: item.file_name,
//     parent_id: item.parent_id,
//   };
// }
```

### `code/src/functions/external-system/data-denormalization.ts`

```typescript
// Denormalization functions transform normalized DevRev objects back into
// the format expected by the external system's API for create/update operations.

// TODO: Add one denormalizer per loadable entity type. Example:

// export function denormalizeTask(item: any, projectId?: string) {
//   const data: any = {};
//
//   if (item.data?.name != null) {
//     data.name = item.data.name;
//   }
//
//   if (item.data?.assignee != null) {
//     if ('external' in item.data.assignee) {
//       data.assignee = item.data.assignee.external;
//     } else {
//       data.assignee = null;
//     }
//   }
//
//   if (item.data?.description != null) {
//     data.notes = item.data.description?.content?.[0] || '';
//   }
//
//   if (projectId != null) {
//     data.projects = [projectId];
//   }
//
//   return { data };
// }
```

### `code/src/functions/external-system/external_domain_metadata.json`

```json
{
  "schema_version": "v0.2.0",
  "record_types": {
    "ENTITY_TYPE_NAME": {
      "name": "Display Name",
      "is_loadable": true,
      "fields": {
        "field_name": {
          "name": "Field Display Name",
          "is_required": true,
          "type": "text"
        }
      }
    },
    "users": {
      "name": "Users",
      "fields": {
        "email": {
          "name": "Email",
          "is_required": true,
          "type": "text"
        },
        "name": {
          "name": "Name",
          "is_required": true,
          "type": "text"
        }
      }
    }
  }
}
```

> **Field types**: `text`, `rich_text`, `int`, `float`, `bool`, `timestamp`, `enum`, `tokens`, `reference`, `composite`
>
> **Reference fields** must declare `refers_to`:
> ```json
> "assignee": {
>   "name": "Assignee",
>   "type": "reference",
>   "reference": { "refers_to": { "#record:users": {} } }
> }
> ```

### `code/src/functions/external-system/initial_domain_mapping.json`

```json
{
  "additional_mappings": {
    "record_type_mappings": {
      "ENTITY_TYPE_NAME": {
        "default_mapping": {
          "object_type": "issue"
        },
        "possible_record_type_mappings": [
          {
            "devrev_leaf_type": "issue",
            "forward": true,
            "reverse": true,
            "shard": {
              "constructed_custom_fields": {
                "ext_object_type": {
                  "field_descriptor": {
                    "allowed_values": ["EntityType"],
                    "db_name": "ext_object_type",
                    "default_value": "EntityType",
                    "description": "The source object type/subtype that the item was created from.",
                    "field_type": "enum",
                    "is_filterable": true,
                    "is_required": true,
                    "name": "ext_object_type",
                    "oasis": { "name": "ext_object_type" },
                    "ui": {
                      "display_name": "External Object Type",
                      "is_hidden_during_create": true,
                      "is_read_only": true
                    }
                  },
                  "transformation_method": {
                    "can_apply_to_null": false,
                    "custom_field_type": {
                      "allowed_values": ["EntityType"],
                      "db_name": "dummy name",
                      "field_type": "enum",
                      "name": "dummy name",
                      "oasis": { "name": "dummy name" }
                    },
                    "forward_jq": "\"EntityType\"",
                    "transformation_method": "use_raw_jq",
                    "use_primary_input": false
                  }
                }
              },
              "devrev_leaf_type": { "object_type": "issue" },
              "mode": "create_shard",
              "stock_field_mappings": {
                "title": {
                  "forward": true,
                  "primary_external_field": "field_name",
                  "reverse": true,
                  "transformation_method": { "transformation_method": "use_directly" }
                },
                "body": {
                  "forward": true,
                  "primary_external_field": "description",
                  "reverse": true,
                  "transformation_method": { "transformation_method": "use_rich_text" }
                },
                "owned_by_ids": {
                  "forward": true,
                  "primary_external_field": "assignee",
                  "reverse": true,
                  "transformation_method": { "transformation_method": "use_as_array_value" }
                },
                "applies_to_part_id": {
                  "forward": true,
                  "reverse": false,
                  "transformation_method": {
                    "is_array": false,
                    "leaf_type": { "object_type": "product" },
                    "transformation_method": "use_devrev_record"
                  }
                },
                "stage": {
                  "forward": true,
                  "reverse": false,
                  "transformation_method": {
                    "enum": "backlog",
                    "transformation_method": "use_fixed_value",
                    "value": "enum_value"
                  }
                },
                "priority": {
                  "forward": true,
                  "reverse": false,
                  "transformation_method": {
                    "enum": "P0",
                    "transformation_method": "use_fixed_value",
                    "value": "enum_value"
                  }
                }
              }
            }
          }
        ]
      },
      "users": {
        "default_mapping": { "object_type": "devu" },
        "possible_record_type_mappings": [
          {
            "devrev_leaf_type": "devu",
            "forward": true,
            "reverse": false,
            "shard": {
              "constructed_custom_fields": {},
              "devrev_leaf_type": { "object_type": "devu" },
              "mode": "create_shard",
              "stock_field_mappings": {
                "display_name": {
                  "forward": true,
                  "primary_external_field": "name",
                  "reverse": false,
                  "transformation_method": { "transformation_method": "use_directly" }
                },
                "email": {
                  "forward": true,
                  "primary_external_field": "email",
                  "reverse": false,
                  "transformation_method": { "transformation_method": "use_directly" }
                },
                "full_name": {
                  "forward": true,
                  "primary_external_field": "name",
                  "reverse": false,
                  "transformation_method": { "transformation_method": "use_directly" }
                }
              }
            }
          }
        ]
      }
    }
  }
}
```

---

## Test Fixture

### `code/src/fixtures/positive-case.json`

> **NOTE**: This fixture is used for local testing with `npm start -- --functionName extraction --fixturePath src/fixtures/positive-case.json`.
> Customize the events and state for your connector.

```json
[
  {
    "context": {
      "secrets": {
        "service_account_token": "placeholder"
      }
    },
    "payload": {
      "connection_data": {
        "org_id": "org-id",
        "org_name": "Org Name",
        "key": "{\"access_token\": \"placeholder\"}",
        "key_type": "external-system-connection"
      },
      "event_context": {
        "mode": "INITIAL",
        "callback_url": "https://api.devrev.ai/internal/airdrop.callback",
        "dev_org_id": "DEV-xxx",
        "dev_user_id": "DEVU-xxx",
        "external_sync_unit_id": "sync-unit-id",
        "import_slug": "airdrop-external-system",
        "snap_in_slug": "external-system-snap-in",
        "snap_in_id": "don:integration:xxx",
        "sync_run_id": "xxx",
        "sync_tier": "TIER-1",
        "sync_unit": "don:xxx",
        "uuid": "xxx",
        "worker_data": {}
      },
      "event_type": "EXTRACTION_EXTERNAL_SYNC_UNITS_START"
    },
    "execution_metadata": {
      "devrev_endpoint": "https://api.devrev.ai"
    },
    "input_data": {
      "global_values": {},
      "event_sources": {}
    }
  }
]
```

---

## Environment & Dev Setup

### `.env.example`

```
# Same as the URL path for your org, e.g. app.devrev.ai/my-org
DEV_ORG=my-org

# Your email address
USER_EMAIL=my@email.com
```

### `.gitignore` (root)

```
/.vscode/*
!/.vscode/run.xml
/.idea/*
!/.idea/runConfigurations
build.tar.gz
nohup.out
.env
```

### `Makefile`

```makefile
include .env
export

default: build

.PHONY: deps
deps:
	cd code && npm ci

.PHONY: build
build: deps
	cd code && npm run build

.PHONY: auth
auth:
	devrev profiles authenticate -o ${DEV_ORG} -u ${USER_EMAIL} --expiry 7

.PHONY: deploy
deploy: auth
	./code/scripts/deploy.sh

.PHONY: uninstall
uninstall:
	./code/scripts/cleanup.sh
```

---

## Auth Pattern Reference

### PAT-based (e.g., Asana)

- User provides a Personal Access Token
- `connection_data.key` contains JSON: `{"access_token": "pat-token-here", "orgId": "workspace-id"}`
- Auth header: `Authorization: Bearer <token>`

### Config-type OAuth / Client Credentials (e.g., Zoho)

- Client ID + Secret registered via `devrev developer_keyring create oauth-secret`
- Users only enter org_id and region in the snap-in setup
- `connection_data.key` contains JSON: `{"access_token": "auto-refreshed-token", "org_id": "...", "region": "..."}`
- Auth header format varies by system (e.g., `Zoho-oauthtoken <token>` for Zoho, `Bearer <token>` for most others)
- Manifest uses `authorize.type: "config"` with template variables: `[CLIENT_ID]`, `[CLIENT_SECRET]`, `[SCOPES]`, `[ORG_ID]`, `[REGION]`, `[ACCESS_TOKEN]`

---

## DevRev Object Type Mappings

| DevRev Type | Use For |
|---|---|
| `issue` | Tickets, tasks, bugs, stories |
| `ticket` | Support tickets, cases |
| `conversation` | Chats, threads |
| `devu` | Users (dev users) |
| `rev_user` | Customers/contacts |
| `account` | Companies/organizations |
| `opportunity` | Sales opportunities |
| `product` | Products |
| `part` | Components, modules |
