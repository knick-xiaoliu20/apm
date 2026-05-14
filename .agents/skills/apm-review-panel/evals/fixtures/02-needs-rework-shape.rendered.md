# PR Review: Centralize configuration loading with `ConfigLoader` service

**PR #2041** · `feat/config-loader` → `main`

---

## Summary

Introduces a `ConfigLoader` singleton that consolidates all environment-variable parsing, validation, and default-value injection into one place. Previously each service module called `process.env` directly, leading to scattered validation logic and silent failures when required variables were absent.

---

## Changed Files

### `src/config/loader.ts` _(new)_

```typescript
import { z } from 'zod';

const ConfigSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  JWT_SECRET: z.string().min(32),
  FEATURE_NEW_DASHBOARD: z.coerce.boolean().default(false),
});

export type AppConfig = z.infer<typeof ConfigSchema>;

let _config: AppConfig | null = null;

export function loadConfig(env: NodeJS.ProcessEnv = process.env): AppConfig {
  if (_config) return _config;
  const result = ConfigSchema.safeParse(env);
  if (!result.success) {
    const issues = result.error.issues
      .map(i => `  • ${i.path.join('.')}: ${i.message}`)
      .join('\n');
    throw new Error(`Invalid configuration:\n${issues}`);
  }
  _config = result.data;
  return _config;
}

export function resetConfig(): void {
  _config = null;
}
```

### `src/config/index.ts` _(new)_

```typescript
export { loadConfig, resetConfig } from './loader';
export type { AppConfig } from './loader';
```

### `src/db/client.ts` _(modified)_

```diff
- const dbUrl = process.env.DATABASE_URL;
- if (!dbUrl) throw new Error('DATABASE_URL is not set');
+ import { loadConfig } from '../config';
+ const { DATABASE_URL: dbUrl } = loadConfig();
```

### `src/cache/redis.ts` _(modified)_

```diff
- const redisUrl = process.env.REDIS_URL ?? 'redis://localhost:6379';
+ import { loadConfig } from '../config';
+ const { REDIS_URL: redisUrl = 'redis://localhost:6379' } = loadConfig();
```

### `src/config/loader.test.ts` _(new)_

```typescript
import { loadConfig, resetConfig } from './loader';

beforeEach(() => resetConfig());

test('parses valid environment', () => {
  const cfg = loadConfig({
    DATABASE_URL: 'postgresql://localhost/test',
    JWT_SECRET: 'a'.repeat(32),
  });
  expect(cfg.PORT).toBe(3000);
  expect(cfg.LOG_LEVEL).toBe('info');
});

test('throws on missing required vars', () => {
  expect(() => loadConfig({})).toThrow('Invalid configuration');
});

test('returns cached instance on second call', () => {
  const env = { DATABASE_URL: 'postgresql://localhost/test', JWT_SECRET: 'a'.repeat(32) };
  const a = loadConfig(env);
  const b = loadConfig({});
  expect(a).toBe(b);
});
```

---

## Concerns Raised During Review

1. **Singleton reset in production** — `resetConfig()` is exported publicly. If called accidentally in production code (not just tests), the next `loadConfig()` call re-parses `process.env`, which is fine, but the export surface is wider than necessary. Consider exporting it only from a test-utilities barrel.

2. **No support for `.env` files** — The loader reads only `process.env`. Projects that rely on `dotenv` must ensure it is loaded before `loadConfig()` is called. This ordering constraint is undocumented.

3. **`REDIS_URL` default duplication** — The schema marks `REDIS_URL` as optional, but `redis.ts` re-specifies a default (`redis://localhost:6379`). The source of truth for defaults should be the schema alone.

4. **Missing integration test** — Unit tests cover the loader in isolation, but there is no test confirming that `db/client.ts` and `cache/redis.ts` actually consume the centralised config rather than falling back to direct `process.env` access.

5. **Coercion of `FEATURE_NEW_DASHBOARD`** — `z.coerce.boolean()` converts the string `'false'` to `true` because any non-empty string is truthy after coercion. A custom refinement or `z.preprocess` is needed for correct boolean flag parsing from env strings.

---

## Verdict

**Needs Rework** — The direction is correct and the centralisation is valuable, but items 3 and 5 are bugs that must be fixed before merge. Items 1, 2, and 4 should be addressed or explicitly deferred with a tracking issue.
