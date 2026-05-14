# PR Review: Introduce async job queue with Redis backend

## Summary

This PR adds an async job queue system backed by Redis, replacing the current in-process queue implementation. The change affects how background tasks are scheduled and executed across worker processes.

## Changed Files

### `src/queue/job_queue.py` (new)

```python
import redis
import json
import uuid
from typing import Any, Callable, Dict, Optional
from datetime import datetime

REDIS_QUEUE_KEY = "apm:job_queue"
REDIS_DEAD_LETTER_KEY = "apm:job_queue:dead_letter"


class JobQueue:
    """Redis-backed async job queue for background task processing."""

    def __init__(self, redis_url: str, max_retries: int = 3):
        self.client = redis.Redis.from_url(redis_url, decode_responses=True)
        self.max_retries = max_retries

    def enqueue(self, task_name: str, payload: Dict[str, Any]) -> str:
        job_id = str(uuid.uuid4())
        job = {
            "id": job_id,
            "task": task_name,
            "payload": payload,
            "retries": 0,
            "enqueued_at": datetime.utcnow().isoformat(),
        }
        self.client.rpush(REDIS_QUEUE_KEY, json.dumps(job))
        return job_id

    def dequeue(self) -> Optional[Dict[str, Any]]:
        raw = self.client.lpop(REDIS_QUEUE_KEY)
        if raw is None:
            return None
        return json.loads(raw)

    def requeue_or_discard(self, job: Dict[str, Any]) -> None:
        job["retries"] += 1
        if job["retries"] >= self.max_retries:
            self.client.rpush(REDIS_DEAD_LETTER_KEY, json.dumps(job))
        else:
            self.client.lpush(REDIS_QUEUE_KEY, json.dumps(job))
```

### `src/queue/worker.py` (modified)

```python
from .job_queue import JobQueue
from .registry import TASK_REGISTRY
import logging
import time

logger = logging.getLogger(__name__)


def run_worker(redis_url: str, poll_interval: float = 0.5) -> None:
    queue = JobQueue(redis_url)
    logger.info("Worker started, polling queue...")
    while True:
        job = queue.dequeue()
        if job is None:
            time.sleep(poll_interval)
            continue
        task_fn = TASK_REGISTRY.get(job["task"])
        if task_fn is None:
            logger.error(f"Unknown task: {job['task']}, discarding job {job['id']}")
            continue
        try:
            task_fn(job["payload"])
            logger.info(f"Job {job['id']} completed successfully")
        except Exception as exc:
            logger.warning(f"Job {job['id']} failed: {exc}, requeueing")
            queue.requeue_or_discard(job)
```

### `config/redis.yaml` (new)

```yaml
redis:
  url: "redis://localhost:6379/0"
  max_retries: 3
  poll_interval_seconds: 0.5
```

## Discussion Points

- **Operational complexity**: Introducing Redis as a hard dependency increases infrastructure requirements. Teams running apm without Redis will need to provision and maintain a Redis instance.
- **Dead letter queue handling**: Currently dead-lettered jobs are stored in Redis but nothing consumes or alerts on them. There is no UI or tooling to inspect or replay failed jobs.
- **Graceful shutdown**: The worker loop has no signal handling for `SIGTERM`/`SIGINT`, which may cause job loss during deploys.
- **Testing strategy**: The PR lacks integration tests against a real Redis instance. Unit tests mock the client entirely, which may miss serialization edge cases.

## Metrics / Risk

- Lines changed: +187 / -43
- New external dependency: `redis>=4.0`
- Affected components: job scheduling, worker lifecycle, config loading
- No migration path documented for teams currently using the in-process queue
