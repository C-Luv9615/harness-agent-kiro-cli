# Jest / Vitest Testing Patterns

**Load this reference when:** detected framework is Jest or Vitest in a JS/TS project.

## Project Structure

```
src/
├── utils/
│   ├── auth.ts
│   └── auth.test.ts          ← co-located (preferred)
├── services/
│   ├── api.ts
│   └── __tests__/
│       └── api.test.ts       ← __tests__ dir (alternative)
```

**Convention detection:** check existing test files first. If project uses `__tests__/`, follow that. If co-located, follow that. Default: co-located.

## Naming

- Test file: `{module}.test.ts` or `{module}.spec.ts` (match existing convention)
- Describe block: module/class name
- Test name: behavior description — `'returns null when user not found'`, not `'test getUserById'`

## Test Structure

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest'; // or jest globals
import { UserService } from '../services/user-service';

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    service = new UserService();
  });

  afterEach(() => {
    vi.restoreAllMocks(); // vitest
    // jest.restoreAllMocks(); // jest
  });

  describe('getUserById', () => {
    it('returns user for valid id', async () => {
      const user = await service.getUserById('123');
      expect(user).toEqual({ id: '123', name: 'Alice' });
    });

    it('returns null for non-existent id', async () => {
      const user = await service.getUserById('999');
      expect(user).toBeNull();
    });

    it('throws on invalid id format', async () => {
      await expect(service.getUserById('')).rejects.toThrow('Invalid ID');
    });
  });
});
```

## Mock Strategy

### When to mock

| Dependency | Mock? | How |
|-----------|-------|-----|
| External API / HTTP | ✅ Always | `vi.mock` / `jest.mock` or msw |
| Database | ✅ Usually | Mock repository layer, not DB driver |
| File system | ✅ Usually | `vi.mock('fs')` |
| Pure utility functions | ❌ Never | Use real implementation |
| Same-module helpers | ❌ Never | Use real implementation |
| Time/Date | ✅ When needed | `vi.useFakeTimers()` / `jest.useFakeTimers()` |

### Module mock

```typescript
// Mock entire module
vi.mock('../services/api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
}));

// Mock single export (keep others real)
vi.mock('../services/api', async () => {
  const actual = await vi.importActual('../services/api');
  return {
    ...actual,
    fetchUser: vi.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
  };
});
```

### Spy on methods

```typescript
const spy = vi.spyOn(service, 'save');
spy.mockResolvedValue(true);

await service.createUser({ name: 'Bob' });

expect(spy).toHaveBeenCalledWith(expect.objectContaining({ name: 'Bob' }));
```

## Config Integration

### Vitest (`vitest.config.ts`)

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node', // or 'jsdom' for browser
    include: ['src/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.test.ts', 'src/**/*.d.ts'],
    },
  },
});
```

### Jest (`jest.config.ts`)

```typescript
export default {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['<rootDir>/src/**/*.test.ts'],
  collectCoverageFrom: ['src/**/*.ts', '!src/**/*.test.ts', '!src/**/*.d.ts'],
};
```

## Async Testing

```typescript
// Promise-based
it('fetches data', async () => {
  const data = await fetchData();
  expect(data).toBeDefined();
});

// Error assertion
it('rejects on network error', async () => {
  vi.mocked(fetch).mockRejectedValue(new Error('Network error'));
  await expect(fetchData()).rejects.toThrow('Network error');
});

// Callback-based (avoid if possible)
it('calls callback', (done) => {
  subscribe((data) => {
    expect(data).toBe('value');
    done();
  });
});
```

## Anti-Patterns

- ❌ `test('works')` — vague name, says nothing about behavior
- ❌ Mocking the module under test — test real code, mock dependencies
- ❌ `expect(mock).toHaveBeenCalled()` without checking args — proves nothing
- ❌ Snapshot tests for logic — snapshots are for UI, not business logic
- ❌ Shared mutable state between tests — each test must be independent
- ❌ `any` in test files — type your test data properly
