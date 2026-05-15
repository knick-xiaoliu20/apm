# PR Review: Lazy Initialization Cache with TTL Support

## Summary

This PR introduces a `LazyCache` class that provides lazy initialization of expensive resources with configurable TTL (time-to-live) expiry and optional background refresh. The implementation includes thread-safe access patterns and a pluggable loader interface.

## Changed Files

### `src/cache/lazy_cache.py` (+187, -0)

```python
import threading
import time
from typing import Callable, Generic, Optional, TypeVar

T = TypeVar("T")


class LazyCache(Generic[T]):
    """
    Thread-safe lazy-initialized cache with TTL expiry.

    The loader function is called at most once until the value expires.
    Background refresh can be enabled to avoid thundering-herd on expiry.
    """

    def __init__(
        self,
        loader: Callable[[], T],
        ttl: float = 300.0,
        background_refresh: bool = False,
    ) -> None:
        self._loader = loader
        self._ttl = ttl
        self._background_refresh = background_refresh
        self._value: Optional[T] = None
        self._expires_at: float = 0.0
        self._lock = threading.Lock()
        self._refresh_thread: Optional[threading.Thread] = None

    def get(self) -> T:
        now = time.monotonic()
        if self._value is not None and now < self._expires_at:
            return self._value
        with self._lock:
            now = time.monotonic()
            if self._value is not None and now < self._expires_at:
                return self._value
            if self._background_refresh and self._value is not None:
                self._schedule_refresh()
                return self._value
            self._value = self._loader()
            self._expires_at = time.monotonic() + self._ttl
            return self._value

    def invalidate(self) -> None:
        with self._lock:
            self._value = None
            self._expires_at = 0.0

    def _schedule_refresh(self) -> None:
        if self._refresh_thread and self._refresh_thread.is_alive():
            return
        self._refresh_thread = threading.Thread(
            target=self._do_refresh, daemon=True
        )
        self._refresh_thread.start()

    def _do_refresh(self) -> None:
        try:
            new_value = self._loader()
            with self._lock:
                self._value = new_value
                self._expires_at = time.monotonic() + self._ttl
        except Exception:
            pass
```

### `tests/cache/test_lazy_cache.py` (+94, -0)

```python
import time
import pytest
from unittest.mock import MagicMock
from src.cache.lazy_cache import LazyCache


def test_loader_called_once_within_ttl():
    loader = MagicMock(return_value="value")
    cache = LazyCache(loader, ttl=60)
    assert cache.get() == "value"
    assert cache.get() == "value"
    loader.assert_called_once()


def test_loader_called_again_after_expiry():
    loader = MagicMock(side_effect=["first", "second"])
    cache = LazyCache(loader, ttl=0.01)
    assert cache.get() == "first"
    time.sleep(0.05)
    assert cache.get() == "second"
    assert loader.call_count == 2


def test_invalidate_forces_reload():
    loader = MagicMock(side_effect=["a", "b"])
    cache = LazyCache(loader, ttl=60)
    cache.get()
    cache.invalidate()
    assert cache.get() == "b"
```

## Review Notes

The double-checked locking pattern is correctly implemented. The background refresh thread is marked as a daemon so it won't block process shutdown. Error suppression in `_do_refresh` is intentional — stale values are preferable to crashes during background refresh.

## Concerns Flagged

- The `background_refresh` path returns a potentially stale value without signalling staleness to the caller. Callers relying on freshness guarantees may be surprised.
- No maximum retry or backoff logic on loader failures during background refresh.
- `ttl=0` edge case: cache will always reload synchronously, which may be intentional but is undocumented.
