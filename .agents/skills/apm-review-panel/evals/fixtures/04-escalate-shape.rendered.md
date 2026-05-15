# PR Review: Distributed Lock Manager — `DistributedLock` core

## Summary

This PR introduces a `DistributedLock` class backed by Redis, intended to replace the
existing in-process `threading.Lock` usage across worker nodes. The implementation
uses `SET NX PX` for atomic acquisition and a Lua script for safe release.

## Changed Files

### `src/locks/distributed.py` (+187 / -0)

```python
import uuid
import time
import logging
from typing import Optional

import redis

log = logging.getLogger(__name__)

LUA_RELEASE_SCRIPT = """
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
"""


class DistributedLock:
    """Redis-backed distributed lock with automatic expiry."""

    def __init__(
        self,
        client: redis.Redis,
        name: str,
        ttl_ms: int = 30_000,
        retry_interval_ms: int = 100,
        retry_timeout_ms: int = 5_000,
    ) -> None:
        self._client = client
        self._name = f"dlock:{name}"
        self._ttl_ms = ttl_ms
        self._retry_interval_ms = retry_interval_ms
        self._retry_timeout_ms = retry_timeout_ms
        self._token: Optional[str] = None
        self._release_script = client.register_script(LUA_RELEASE_SCRIPT)

    def acquire(self, blocking: bool = True) -> bool:
        token = str(uuid.uuid4())
        deadline = time.monotonic() + self._retry_timeout_ms / 1000.0

        while True:
            ok = self._client.set(
                self._name, token, nx=True, px=self._ttl_ms
            )
            if ok:
                self._token = token
                log.debug("Acquired lock %s (token=%s)", self._name, token)
                return True
            if not blocking:
                return False
            if time.monotonic() >= deadline:
                log.warning("Timed out waiting for lock %s", self._name)
                return False
            time.sleep(self._retry_interval_ms / 1000.0)

    def release(self) -> bool:
        if self._token is None:
            raise RuntimeError("Cannot release a lock that has not been acquired")
        result = self._release_script(keys=[self._name], args=[self._token])
        self._token = None
        if result == 0:
            log.warning("Lock %s was already expired or stolen", self._name)
            return False
        log.debug("Released lock %s", self._name)
        return True

    def __enter__(self) -> "DistributedLock":
        if not self.acquire():
            raise TimeoutError(f"Could not acquire lock: {self._name}")
        return self

    def __exit__(self, *_) -> None:
        if self._token is not None:
            self.release()
```

### `tests/locks/test_distributed.py` (+94 / -0)

```python
import pytest
from unittest.mock import MagicMock, patch
from src.locks.distributed import DistributedLock


@pytest.fixture
def mock_redis():
    client = MagicMock()
    client.register_script.return_value = MagicMock(return_value=1)
    return client


def test_acquire_success(mock_redis):
    mock_redis.set.return_value = True
    lock = DistributedLock(mock_redis, "test-resource")
    assert lock.acquire() is True
    assert lock._token is not None


def test_release_calls_lua(mock_redis):
    mock_redis.set.return_value = True
    lock = DistributedLock(mock_redis, "test-resource")
    lock.acquire()
    result = lock.release()
    assert result is True
    assert lock._token is None


def test_context_manager_releases_on_exit(mock_redis):
    mock_redis.set.return_value = True
    lock = DistributedLock(mock_redis, "test-resource")
    with lock:
        assert lock._token is not None
    assert lock._token is None
```

### `config/redis.yaml` (+12 / -0)

```yaml
redis:
  host: "${REDIS_HOST:-localhost}"
  port: 6379
  db: 0
  socket_timeout: 2.0
  socket_connect_timeout: 1.0
  health_check_interval: 30
```

## Concerns Flagged by Author

- **Clock drift**: The author notes that `ttl_ms` is set on the acquiring node's
  wall clock, but Redis expiry is server-side. Under heavy GC pauses the lock
  could expire before `release()` is called, leading to silent double-ownership.
- **No fencing token**: The implementation has no monotonic counter / fencing
  token, so a slow consumer holding a stale lock can still corrupt shared state
  even after expiry.
- **Single Redis node**: The author explicitly defers Redlock (multi-node)
  to a follow-up, but the issue tracker shows two open incidents caused by
  Redis failover dropping locks silently.
- **Test coverage gap**: The retry / timeout path in `acquire` has no test.

## Linked Issues

- Closes #4021 — Replace in-process locks in worker pool
- Related #3887 — Redis failover caused duplicate job execution (open)
- Related #4103 — Investigate fencing tokens for distributed coordination (open)
