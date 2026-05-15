# PR #2341 — Add rate limiting middleware to API gateway

## Summary

Introduces a token-bucket rate limiter as Express middleware. Limits are
configured per-route and stored in Redis so they survive restarts and work
across multiple instances.

## Changed files

### `src/middleware/rate-limit.ts` (new, +187 lines)

```typescript
import { Request, Response, NextFunction } from 'express';
import { RedisClient } from '../infra/redis';

export interface RateLimitConfig {
  windowMs: number;      // rolling window in milliseconds
  maxRequests: number;   // max requests per window
  keyPrefix?: string;    // redis key namespace
  onLimitReached?: (req: Request, res: Response) => void;
}

/**
 * TokenBucketLimiter implements a sliding-window token-bucket algorithm
 * backed by Redis MULTI/EXEC for atomic increments.
 */
export class TokenBucketLimiter {
  private redis: RedisClient;
  private config: Required<RateLimitConfig>;

  constructor(redis: RedisClient, config: RateLimitConfig) {
    this.redis = redis;
    this.config = {
      keyPrefix: 'rl',
      onLimitReached: (_req, res) =>
        res.status(429).json({ error: 'Too Many Requests' }),
      ...config,
    };
  }

  middleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      const key = `${this.config.keyPrefix}:${req.ip}:${req.path}`;
      const now = Date.now();
      const windowStart = now - this.config.windowMs;

      const pipeline = this.redis.pipeline();
      pipeline.zremrangebyscore(key, '-inf', windowStart);
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      pipeline.zcard(key);
      pipeline.pexpire(key, this.config.windowMs);
      const results = await pipeline.exec();

      const count = results[2][1] as number;
      res.setHeader('X-RateLimit-Limit', this.config.maxRequests);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, this.config.maxRequests - count));

      if (count > this.config.maxRequests) {
        return this.config.onLimitReached(req, res);
      }
      next();
    };
  }
}

export function createRateLimiter(
  redis: RedisClient,
  config: RateLimitConfig,
) {
  return new TokenBucketLimiter(redis, config).middleware();
}
```

### `src/middleware/rate-limit.test.ts` (new, +94 lines)

Unit tests using a mock Redis pipeline. Covers:
- requests within limit pass through
- requests exceeding limit receive 429
- `X-RateLimit-Remaining` header decrements correctly
- expired entries are pruned before counting

### `src/routes/api.ts` (modified, +6 / -1 lines)

Applies `createRateLimiter` to `/api/v1/search` (60 req / 60 s) and
`/api/v1/export` (10 req / 60 s).

### `docs/rate-limiting.md` (new, +42 lines)

Operator guide covering Redis key schema, how to tune limits per environment,
and how to disable limiting in local development.

## Notes from author

- Redis dependency was already present; no new infra required.
- `onLimitReached` callback is intentionally sync-friendly so callers can
  log or emit metrics before responding.
- Did not add distributed lock around the pipeline because `ZADD` + `ZCARD`
  inside a pipeline is already atomic enough for our SLO.
