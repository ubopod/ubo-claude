---
name: coding-standards
description: Project-specific coding standards for ubo_app (Python 3.11): typing, immutability, asyncio/service patterns, testing, and tooling (ruff + pyright).
---

# Coding Standards (ubo_app / Python)

These standards are optimized for `ubo_app`: a Raspberry Pi Python app with an event-driven architecture, `python-redux` state management, and threaded services with dedicated event loops.

## Core principles

- **Readability wins**: prefer clear code over clever code.
- **Small, composable units**: small functions, small modules, clear boundaries.
- **Fail safely**: validate inputs at boundaries, handle errors intentionally, log without leaking secrets.
- **Deterministic state**: state is immutable; changes happen via actions → reducers → events.

---

## Tooling requirements (non-negotiable)

- **Python**: 3.11 (>=3.11, <3.12)
- **Linter/formatter**: `ruff` (assume strict config; comply with repo config)
- **Type checking**: `pyright`
- **Tests**: `pytest` + `pytest-asyncio` (auto mode)

If tools disagree, **fix the code** (don’t fight the tools).

Repo-verified notes (from `pyproject.toml`):

- `ruff` runs with **`select = ["ALL"]`** (strict), with project-specific ignores.
- Quotes are enforced: **single** for strings, **double** for docstrings.

---

## Naming conventions

- **Modules/files**: `snake_case.py`
- **Functions/vars**: `snake_case`
- **Classes**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE`
- **Env vars**: `UBO_` prefix (e.g., `UBO_DEBUG`)
- **Service IDs**: lowercase, hyphenated (e.g., `speech-synthesis`)
- **Notification/icon IDs**: `ubo:` for core; `<service_name>:` for services

---

## Typing rules (pyright-friendly)

- **Type hints required** for all public APIs and any cross-module boundary.
- Prefer **concrete types** over `Any`.
- Use `| None` (PEP 604) instead of `Optional[...]`.
- Prefer `collections.abc` types (`Sequence`, `Mapping`, `Callable`, `Awaitable`) over concrete containers in APIs.

Example:

```python
from collections.abc import Sequence

def format_items(items: Sequence[str]) -> str:
    return ', '.join(items)
```

### “Unknown” input

Use `object` / `Unknown`-equivalent patterns and validate:

```python
def parse_limit(value: object) -> int:
    if not isinstance(value, int):
        raise TypeError('limit must be an int')
    if value < 0:
        raise ValueError('limit must be >= 0')
    return value
```

---

## Immutability (CRITICAL)

### State must be immutable

- All store/service state objects must be immutable (`@dataclass(frozen=True)` and/or `python_immutable.Immutable`).
- Reducers must **return new state**; never mutate existing objects.

```python
from dataclasses import dataclass, replace
from python_immutable import Immutable

@dataclass(frozen=True)
class ServiceState(Immutable):
    is_enabled: bool = True
    volume: int = 0

def set_volume(state: ServiceState, volume: int) -> ServiceState:
    return replace(state, volume=volume)
```

### Collections inside state

- Prefer immutable-friendly choices (e.g., tuples) in state.
- If you must use lists/dicts, ensure they’re not mutated after construction (copy/replace on update).

---

## Async + service-thread rules (CRITICAL)

### Always use `ubo_app.utils.async_.create_task`

Do **not** use `asyncio.create_task` directly in services; tasks must be created in the correct event loop.

```python
from ubo_app.utils.async_ import create_task

create_task(background_worker())
```

Note: `create_task(...)` returns an `asyncio.Handle` (not an awaitable Task). Design workers to stop via a stop signal / shutdown, not by cancelling the handle.

### Setup must complete

Service `setup()` must not block forever. Spawn background work via `create_task` and return.

```python
from ubo_app.utils.async_ import create_task

async def setup() -> None:
    create_task(background_worker())
    return
```

### Cancellation & shutdown

- Background tasks must support cancellation (`asyncio.CancelledError`) and exit quickly.
- Prefer explicit “stop” signals / events over ad-hoc globals.
- Ensure hardware resources are released (GPIO cleanup, audio handles, camera, etc.).

---

## Service boundaries (architecture rule)

- **No direct cross-service method calls** for business logic.
- Communicate via **actions/events** through the store.
- Keep hardware access inside the relevant service (or a shared hardware utility with strict API).

---

## Error handling & logging

- Use **structured, actionable logs**. Include context (service id, action type, device state) but not secrets.
- Don’t swallow exceptions silently; either:
  - handle locally and emit an event/action, or
  - log and re-raise if it’s unrecoverable.

### Don’t leak secrets

- Never log tokens, WiFi passwords, private keys, or full request payloads with secrets.
- Redact sensitive values.

---

## Imports & module structure

- Prefer absolute imports from `ubo_app...` (consistent and pyright-friendly).
- Keep modules cohesive:
  - reducers/state definitions close together,
  - hardware drivers in the service that owns them,
  - shared utilities in `ubo_app/utils/` with minimal dependencies.

---

## Reducers (python-redux)

- Reducers must be **pure**:
  - no I/O
  - no network calls
  - no hardware calls
  - no time-based nondeterminism
- Side effects happen in **service tasks** that react to actions/events.

---

## Style: docstrings, comments, formatting

- **Strings**: single quotes
- **Docstrings**: double quotes
- Comment **why**, not what.

```python
def compute_backoff_ms(retry_count: int) -> int:
    """Exponential backoff to avoid overwhelming services during outages."""
    return min(1000 * (2**retry_count), 30_000)
```

---

## Testing standards

- Use **pytest** + **pytest-asyncio** for async behavior.
- Tests must cover:
  - happy path
  - error path
  - cancellation/shutdown (where relevant)
  - boundary conditions (empty, max, invalid)
- Prefer **behavior-focused** tests:
  - store changes (state snapshots)
  - emitted actions/events
  - UI-visible outcomes (for Kivy layers)

Example (async):

```python
import pytest

@pytest.mark.asyncio
async def test_example(app_context):
    await app_context.start()
    assert app_context.is_running
```

---

## Security basics (device + embedded)

- **No hardcoded secrets**: environment variables or secure device storage.
- Validate all inputs that cross a boundary:
  - network, RPC/gRPC, filesystem paths, QR inputs, web UI inputs.
- Be careful with filesystem operations:
  - avoid path traversal (never join user input directly to paths).

---

## Code review checklist (quick)

- **State**: immutable updates only; reducers pure
- **Async**: uses `ubo_app.utils.async_.create_task`; setup returns
- **Threads/loops**: work happens in the right service loop
- **Types**: pyright clean; no `Any` creep
- **Lint**: ruff clean (format + rules)
- **Tests**: new behavior covered; error/cancel paths tested

