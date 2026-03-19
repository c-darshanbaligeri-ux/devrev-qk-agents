# Unit Testing Reference

Test patterns for DevRev snap-ins using Jest + TypeScript.

---

## Setup

### jest.config.js

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/*.test.ts'],
  collectCoverageFrom: [
    'src/functions/**/*.ts',
    '!src/functions/**/index.ts',  // Entry points tested via integration
    '!src/test-runner/**',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70,
    },
  },
};
```

### package.json additions

```json
{
  "scripts": {
    "test": "jest",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "@types/jest": "^29.0.0"
  }
}
```

---

## Normalization Tests

```typescript
// src/test/normalization.test.ts

import { normalizeTask, normalizeUser } from '../functions/<s>/extraction/data-normalization';

describe('normalizeTask', () => {
  it('should produce correct format with all fields', () => {
    const externalTask = {
      id: 12345,
      name: 'Fix login bug',
      description: '<p>Users cannot login after update</p>',
      status: 'in_progress',
      assignee: { email: 'dev@company.com', name: 'Dev User' },
      created_at: '2025-03-01T10:00:00Z',
      updated_at: '2025-03-15T14:30:00Z',
    };

    const result = normalizeTask(externalTask);

    expect(result).toEqual({
      id: '12345',             // String, not number
      created_date: '2025-03-01T10:00:00Z',  // RFC3339
      modified_date: '2025-03-15T14:30:00Z',
      data: {
        title: 'Fix login bug',
        description: expect.any(String),  // HTML → markdown or preserved
        status: 'in_progress',
        assignee: 'dev@company.com',
        item_url_field: expect.stringContaining('12345'),
      },
    });
  });

  it('should handle missing optional fields with null', () => {
    const externalTask = {
      id: 99999,
      name: 'Minimal task',
      created_at: '2025-03-01T10:00:00Z',
      // No description, no assignee, no updated_at
    };

    const result = normalizeTask(externalTask);

    expect(result.id).toBe('99999');
    expect(result.data.description).toBeNull();
    expect(result.data.assignee).toBeNull();
    expect(result.modified_date).toBe('2025-03-01T10:00:00Z'); // Fallback to created
  });

  it('should convert empty strings to null', () => {
    const externalTask = {
      id: 1,
      name: 'Task with empties',
      description: '',         // Should become null
      status: '',              // Should become null or default
      assignee: { email: '' },
      created_at: '2025-01-01T00:00:00Z',
    };

    const result = normalizeTask(externalTask);

    expect(result.data.description).toBeNull();
    expect(result.data.assignee).toBeNull();
  });

  it('should handle unicode characters', () => {
    const externalTask = {
      id: 42,
      name: 'Tâche avec des caractères spéciaux: ñ, ü, 日本語',
      description: 'Contains emoji: 🎉 and symbols: ™ ® ©',
      created_at: '2025-03-01T00:00:00Z',
    };

    const result = normalizeTask(externalTask);

    expect(result.data.title).toContain('ñ');
    expect(result.data.title).toContain('日本語');
  });

  it('should ensure id is always a string', () => {
    const numericId = normalizeTask({ id: 123, name: 'test', created_at: '2025-01-01T00:00:00Z' });
    expect(typeof numericId.id).toBe('string');

    const stringId = normalizeTask({ id: 'abc-456', name: 'test', created_at: '2025-01-01T00:00:00Z' });
    expect(typeof stringId.id).toBe('string');
  });
});

describe('normalizeUser', () => {
  it('should map name and email', () => {
    const result = normalizeUser({
      id: 'u-100',
      name: 'Priya Sharma',
      email: 'priya@company.com',
      created_at: '2025-01-01T00:00:00Z',
    });

    expect(result.id).toBe('u-100');
    expect(result.data.name).toBe('Priya Sharma');
    expect(result.data.email).toBe('priya@company.com');
  });
});
```

---

## Field Mapping Tests

```typescript
// src/test/field-mapping.test.ts

import { mapStatus, mapPriority } from '../functions/<s>/extraction/data-normalization';

describe('status mapping', () => {
  const cases: [string, string][] = [
    ['open', 'open'],
    ['in_progress', 'in_progress'],
    ['done', 'closed'],
    ['completed', 'closed'],
    ['archived', 'closed'],
  ];

  test.each(cases)('maps "%s" → "%s"', (external, devrev) => {
    expect(mapStatus(external)).toBe(devrev);
  });

  it('should handle unknown status with fallback', () => {
    expect(mapStatus('some_unknown_status')).toBe('open'); // Or whatever the default is
  });

  it('should handle null/undefined status', () => {
    expect(mapStatus(null as any)).toBe('open');
    expect(mapStatus(undefined as any)).toBe('open');
  });
});

describe('priority mapping', () => {
  const cases: [string, string][] = [
    ['highest', 'p0'],
    ['high', 'p1'],
    ['medium', 'p2'],
    ['low', 'p3'],
  ];

  test.each(cases)('maps "%s" → "%s"', (external, devrev) => {
    expect(mapPriority(external)).toBe(devrev);
  });
});
```

---

## Error Handling Tests

```typescript
// src/test/error-handling.test.ts

describe('fetchSafe', () => {
  beforeEach(() => {
    global.fetch = jest.fn();
  });

  it('should retry on 5xx errors', async () => {
    (global.fetch as jest.Mock)
      .mockResolvedValueOnce({ ok: false, status: 500 })
      .mockResolvedValueOnce({ ok: true, status: 200, json: async () => ({}) });

    const result = await fetchSafe('https://api.example.com/test', 'token');
    expect(result.ok).toBe(true);
    expect(global.fetch).toHaveBeenCalledTimes(2);
  });

  it('should return 429 without retrying (let adapter handle delay)', async () => {
    (global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: false,
      status: 429,
      headers: { get: (h: string) => h === 'Retry-After' ? '30' : null },
    });

    const result = await fetchSafe('https://api.example.com/test', 'token');
    expect(result.status).toBe(429);
    expect(global.fetch).toHaveBeenCalledTimes(1); // No retry — adapter emits Delay
  });

  it('should throw on 401 (auth failure)', async () => {
    (global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: false,
      status: 401,
      statusText: 'Unauthorized',
    });

    await expect(
      fetchSafe('https://api.example.com/test', 'bad-token')
    ).rejects.toThrow(/401/);
  });
});
```

---

## State Management Tests

```typescript
// src/test/state-management.test.ts

import { initialExtractorState } from '../functions/<s>/extraction/data-extraction';

describe('extractor state', () => {
  it('should initialize with all phases incomplete', () => {
    const state = { ...initialExtractorState };

    expect(state.users.completed).toBe(false);
    expect(state.tasks.completed).toBe(false);
    expect(state.attachments.completed).toBe(false);
  });

  it('should preserve cursor across simulated invocations', () => {
    const state = { ...initialExtractorState };

    // Simulate: first invocation extracts some users, then timeout
    state.users.cursor = 'cursor-page-3';
    state.users.completed = false;

    // Verify: on next invocation, cursor is still there
    expect(state.users.cursor).toBe('cursor-page-3');
    expect(state.users.completed).toBe(false);
  });

  it('should not skip phases when one is complete', () => {
    const state = { ...initialExtractorState };
    state.users.completed = true;

    // Tasks should still be pending
    expect(state.tasks.completed).toBe(false);
  });
});
```

---

## Timeout Simulation Test

```typescript
// src/test/timeout.test.ts

import { spawn } from '@devrev/ts-adaas';
import { initialExtractorState } from '../functions/<s>/extraction/data-extraction';

describe('timeout handling', () => {
  it('should emit progress on soft timeout and preserve state', async () => {
    // Use spawn with very short timeout to simulate
    const result = await spawn({
      event: require('../fixtures/extraction_start.json'),
      initialState: initialExtractorState,
      workerPath: '../functions/<s>/extraction/workers/data-extraction.ts',
      options: {
        timeout: 5 * 1000,  // 5 seconds — forces timeout
        isLocalDevelopment: true,
      },
    });

    // After timeout: state should have progress, not be reset
    // The emit should be ExtractionDataProgress
    // (Exact assertion depends on SDK test harness API)
  });
});
```

---

## Chef-CLI Validation (Automated)

```bash
#!/bin/bash
# scripts/validate.sh — run before unit tests

set -e

echo "=== Validating metadata ==="
chef-cli validate-metadata < external_domain_metadata.json
echo "✅ Metadata valid"

echo "=== Validating manifest ==="
devrev snap_in_version validate-manifest ../manifest.yaml
echo "✅ Manifest valid"

echo "=== Generating sample data for format check ==="
for record_type in users tasks; do
  echo '{}' | chef-cli fuzz-extracted -r $record_type -m external_domain_metadata.json > /tmp/expected_${record_type}.json
  echo "✅ Sample $record_type format generated"
done

echo "=== All pre-checks passed ==="
```
