---
name: service-architecture
description: ubo_app service architecture: lifecycle, threading/event loops, ubo_handle.py contract, reducer registration, subscriptions/cleanup, and cross-service communication rules.
---

# Service Architecture (ubo_app)

`ubo_app` is organized around **modular services**. Services own their domain (hardware, networking, apps, UI helpers) and integrate via a shared **store** (`python-redux`) using **actions → reducers → events**.

This doc defines **how services are structured**, **how they start/stop**, and **how they communicate**.

---

## Architectural rule (most important)

**Services do not call each other for business logic.**

- ✅ Communicate via **actions/events** through the store.
- ❌ Avoid direct cross-service method calls (creates hidden coupling, breaks isolation, complicates threading).

If two services need shared logic, extract it into a **small shared utility module** with a strict API (preferably in `ubo_app/utils/`), not a backdoor service call.

---

## Service layout (standard)

Each service typically lives under `ubo_app/services/XXX-service-name/`:

```
services/XXX-service-name/
├── __init__.py
├── ubo_handle.py      # REQUIRED: service registration + setup entrypoint
├── reducer.py         # reducer + immutable state
├── setup.py           # optional: hardware/device init helpers
└── ...                # additional modules (workers, adapters, drivers)
```

---

## `ubo_handle.py` contract

`ubo_handle.py` is the “entrypoint” for the service.

Responsibilities:

- Register the service (id, label, enabled flag).
- Register the reducer(s).
- Start any background tasks **without blocking** service initialization.
- Return cleanup subscriptions (if used) so the system can stop cleanly.

Template:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ubo_handle import ReducerRegistrar, register
    from ubo_app.utils.types import Subscriptions

# setup signature (verified):
# - can take 0 args, or (register_reducer: ReducerRegistrar)
# - can return: None | Subscriptions | Coroutine[..., Subscriptions | None]
def setup(register_reducer: ReducerRegistrar) -> Subscriptions | None:
    """Service setup - called during initialization."""

    from reducer import reducer

    register_reducer(reducer)

    from setup import init_service

    return init_service()

register(
    service_id='service-name',
    label='User-Facing Label',
    setup=setup,
)
```

### Setup must complete (CRITICAL)

`setup()` must **not** run forever.

If you need continuous work, schedule it as a background task and return.

### No global-scope imports in `ubo_handle.py` (CRITICAL, enforced)

The service loader forbids imports in the global scope of `ubo_handle.py`. Keep imports inside functions (e.g., inside `setup()` or inside workers).

---

## Threading + event loops (model)

The design assumption is:

- Services run in **isolated threads** (or isolated execution contexts).
- Each service has a **dedicated asyncio event loop**.
- Background tasks must be created on the **correct loop**.

### Creating tasks (CRITICAL)

Use `create_task` from `ubo_app.utils.async_` instead of `asyncio.create_task`, so work is scheduled on the right service loop.

```python
from ubo_app.utils.async_ import create_task

create_task(background_worker())
```

Note: `create_task(...)` returns an `asyncio.Handle` (a scheduled callback handle), not an `asyncio.Task`. Don’t treat it as cancellable.

---

## Lifecycle: start → run → stop

At a high level:

1. **Service registration**: `register(...)` declares the service and its setup hook.
2. **Initialization**: the service setup registers reducers and starts background workers.
3. **Runtime**: workers react to store changes/actions/events and interact with hardware/network.
4. **Shutdown**: subscriptions/cleanup handlers cancel tasks and release resources.

### Deterministic shutdown

Your service should be able to stop quickly and safely:

- **Cancel tasks** (handle `asyncio.CancelledError`)
- **Close hardware handles** (GPIO, camera, audio)
- **Close sockets/subprocesses**
- **Unsubscribe watchers** (store subscriptions, file watchers, etc.)

---

## Reducer registration

Reducers are the only place state is updated.

- Reducers must be **pure** (no I/O, no hardware, no network).
- State must be **immutable** (`@dataclass(frozen=True)` and/or `python_immutable.Immutable`).

Example:

```python
from dataclasses import dataclass
from python_immutable import Immutable

@dataclass(frozen=True)
class ServiceState(Immutable):
    status: str = 'idle'

def reducer(state: ServiceState | None, action):
    if state is None:
        return ServiceState()
    # action handling...
    return state
```

---

## Subscriptions & cleanup

Many services need to “listen” for changes (store updates, timers, hardware signals).

Guidelines:

- Prefer explicit **subscription objects** (or cleanup callables) that can be returned from `setup()`.
- Always implement cleanup so the service doesn’t leak threads, tasks, file descriptors, or device handles.

If your architecture uses a `Subscriptions` type, treat it as:

- a collection of `unsubscribe()` / cleanup functions, or
- a structured object holding cleanup handles

### Pattern: start background workers + return cleanup

```python
from ubo_app.utils.async_ import create_task

def setup(register_reducer):
    register_reducer(reducer)

    import asyncio
    stop_requested = asyncio.Event()
    create_task(background_worker())

    def cleanup() -> None:
        stop_requested.set()

    return [cleanup]  # shutdown will call this; cleanup may also return a coroutine
```

---

## Communication patterns

### From worker to state

- Worker observes something (hardware state, network status, timer).
- Worker dispatches an **action**.
- Reducer updates the immutable **state**.
- UI/services observe state changes and react.

### Events vs actions

Use whatever your project defines, but keep the intent consistent:

- **Action**: “something happened / do something” (input to reducer and effects)
- **Event**: “something occurred” (broadcast / notification for observers)

---

## Common pitfalls (and how to avoid them)

- **Blocking in setup**: never `while True` or long awaits in `setup()`; spawn a worker task and return.
- **Wrong loop**: don’t call `asyncio.create_task`; use `ubo_app.utils.async_.create_task`.
- **Reducer side effects**: reducers should not touch hardware, filesystem, network, or time.
- **Hidden coupling**: avoid calling into other services; dispatch actions instead.
- **Leaking resources**: always provide cleanup for tasks/subscriptions/hardware.

---

## Quick “does this service follow the model?” checklist

- **Registration**: has `ubo_handle.py` with `register(...)`
- **Reducer**: registered and pure; state immutable
- **Async**: uses `create_task` for background work
- **Setup**: completes promptly
- **Cleanup**: tasks cancelled + resources released
- **Communication**: actions/events only (no direct cross-service calls)

