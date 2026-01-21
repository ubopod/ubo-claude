---
name: async-patterns
description: Asyncio patterns for ubo_app: service event loops, create_task usage, cancellation/shutdown, to_thread, timeouts, and common pitfalls.
---

# Async Patterns (ubo_app / asyncio)

`ubo_app` relies on asyncio **inside service-managed event loops**. The most important constraint is that **each service runs work on its own loop**, so task creation and cross-thread calls must be done carefully.

---

## Non-negotiables (CRITICAL)

### 1) Use `ubo_app.utils.async_.create_task`

Do **not** call `asyncio.create_task` directly in services.

```python
from ubo_app.utils.async_ import create_task

create_task(background_worker())
```

Why: tasks must be scheduled on the **correct service event loop**.

**Important**: `create_task(...)` returns an `asyncio.Handle` (a scheduled callback handle), not an `asyncio.Task`. Don’t treat it as cancellable or awaitable.

### 2) Service `setup()` must complete

Never block forever in `setup()`. Spawn background tasks and return.

```python
from ubo_app.utils.async_ import create_task

async def setup() -> None:
    create_task(background_worker())
    return
```

---

## Cancellation and shutdown (do this by default)

Background tasks must be cancellable and release resources.

Pattern:

```python
import asyncio

async def background_worker() -> None:
    try:
        while True:
            await do_one_iteration()
    except asyncio.CancelledError:
        # Cleanup / release resources here
        raise
```

Guidelines:

- Keep cancellation responsive (don’t swallow `CancelledError`).
- Prefer bounded waits (timeouts) over indefinite awaits.
- Always close hardware/network handles on shutdown.

---

## Timeouts (prefer explicit bounds)

Avoid “hang forever” behavior by bounding external calls:

```python
import asyncio

async def fetch_with_timeout() -> bytes:
    return await asyncio.wait_for(fetch_bytes(), timeout=5.0)
```

Use shorter timeouts for UI-critical paths; use retries/backoff for flaky network operations.

---

## CPU-bound / blocking I/O: use threads safely

If something blocks the event loop (CPU-heavy work, blocking I/O, driver calls), offload it using `ubo_app.utils.async_.to_thread`.

`to_thread(...)` schedules work via the correct coroutine runner and returns an `asyncio.Handle` (not the function result). Use a callback or dispatch an action when it completes.

```python
from ubo_app.utils.async_ import to_thread
from ubo_app.utils.async_ import ToThreadOptions

def on_done(_task) -> None:
    # e.g. dispatch an action / update a queue / log completion
    ...

to_thread(
    expensive_hash,
    ToThreadOptions(callback=on_done, name='hash-work'),
    b'...',
)
```

Guidelines:

- Keep the event loop for coordination, not heavy work.
- Never call blocking hardware APIs in tight async loops; offload or redesign.
- If thread safety is unclear, centralize access in the owning service and serialize operations.

---

## Cross-thread / cross-loop interactions

Avoid reaching into another service’s loop directly.

- ✅ Prefer dispatching an **action/event** through the store.
- ✅ If you must schedule work from another thread, use the project’s approved mechanism (e.g., a thread-safe queue or loop-safe call helper).
- ❌ Don’t pass around raw loop objects or call `run_until_complete` in production code.

---

## Concurrency patterns

### Parallelize independent awaits

```python
results = await asyncio.gather(
    read_sensor_a(),
    read_sensor_b(),
    return_exceptions=False,
)
```

### Rate-limit / serialize access to shared resources

Use `asyncio.Lock` for shared devices (camera, SPI bus, audio device), or build a single “driver worker” task that serializes requests.

```python
import asyncio

_camera_lock = asyncio.Lock()

async def take_photo():
    async with _camera_lock:
        return await camera_capture()
```

---

## Avoid these common bugs

- **Wrong loop**: calling `asyncio.create_task` instead of `ubo_app.utils.async_.create_task`
- **Leaking work**: starting background workers without a shutdown path (stop signal and/or cleanup subscription)
- **Silent failures**: tasks that crash without logging/alerting
- **Blocking the loop**: CPU work or blocking I/O in async functions
- **Unbounded waits**: awaits that can hang forever (no timeout, no stop condition)
- **Busy loops**: `while True` without a sleep/await that yields control

---

## Testing async code

- Use `pytest` + `pytest-asyncio`.
- Prefer deterministic synchronization primitives (events/queues/state transitions) over `sleep`.
- Test cancellation paths for long-running workers.

Example:

```python
import asyncio
import pytest

@pytest.mark.asyncio
async def test_worker_cancels_cleanly():
    task = asyncio.create_task(background_worker())  # in unit tests only
    await asyncio.sleep(0)  # give it a tick
    task.cancel()
    with pytest.raises(asyncio.CancelledError):
        await task
```

