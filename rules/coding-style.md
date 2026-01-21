# Coding Style

## Immutability (CRITICAL)

ALWAYS use `Immutable` base class and `replace()` for updates, NEVER mutate:

```python
# WRONG: Mutation
def update_connection(state: WiFiState, ssid: str) -> WiFiState:
    state.current_ssid = ssid  # MUTATION!
    return state

# CORRECT: Immutability with replace()
from dataclasses import replace

def update_connection(state: WiFiState, ssid: str) -> WiFiState:
    return replace(state, current_ssid=ssid)
```

State classes MUST inherit from `Immutable`:

```python
from immutable import Immutable

class WiFiState(Immutable):
    connections: Sequence[WiFiConnection] | None = None
    current_connection: WiFiConnection | None = None
```

## Type Annotations (REQUIRED)

ALWAYS use type hints with future annotations:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence
    from ubo_app.store.main import RootState

def process_data(items: Sequence[str]) -> list[str]:
    return [item.upper() for item in items]
```

## Quote Style

Follow ruff configuration:
- **Single quotes** for inline strings: `'hello'`
- **Double quotes** for docstrings: `"""Docstring here."""`

```python
# CORRECT
message = 'Hello world'
config = {'key': 'value'}

def process() -> None:
    """Process the data."""
    pass
```

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, reducers can be longer
- Extract utilities from large modules
- Organize services by priority prefix (000-xxx, 030-xxx, etc.)

Service structure:
```
ubo_app/services/030-wifi/
├── setup.py          # Service initialization
├── reducer.py        # State reducer
├── ubo_handle.py     # Menu/UI registration
├── constants.py      # Service constants
└── pages/            # UI pages (if needed)
```

## Error Handling

Use structured error handling:

```python
from redux import InitializationActionError

def reducer(state: MyState | None, action: MyAction) -> ReducerResult:
    if state is None:
        if isinstance(action, InitAction):
            return MyState()
        raise InitializationActionError(action)

    # Handle actions...
```

For async operations:

```python
from tenacity import retry, wait_fixed, stop_after_attempt

@retry(wait=wait_fixed(1), stop=stop_after_attempt(3))
async def fetch_data() -> Data:
    """Fetch data with retry logic."""
    ...
```

## Async Patterns (CRITICAL)

ALWAYS use `create_task` from `ubo_app.utils.async_` for background tasks:

```python
# WRONG: Uses wrong event loop
import asyncio
asyncio.create_task(background_work())

# CORRECT: Runs in service's event loop
from ubo_app.utils.async_ import create_task
create_task(background_work())
```

Using `await` inside async functions is always fine.

## Redux Patterns

Actions inherit from `BaseAction`, events from `BaseEvent`:

```python
from redux import BaseAction, BaseEvent

class MyServiceAction(BaseAction): ...

class MyUpdateAction(MyServiceAction):
    value: str

class MyServiceEvent(BaseEvent): ...

class MyUpdateEvent(MyServiceEvent):
    result: str
```

Reducers return `ReducerResult` or `CompleteReducerResult`:

```python
from redux import CompleteReducerResult, ReducerResult

def reducer(
    state: MyState | None,
    action: MyAction,
) -> ReducerResult[MyState, BaseAction, MyEvent]:
    match action:
        case MyUpdateAction():
            return CompleteReducerResult(
                state=replace(state, value=action.value),
                events=[MyUpdateEvent(result='done')],
            )
        case _:
            return state
```

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Actions | PascalCase + `Action` suffix | `WiFiUpdateAction` |
| Events | PascalCase + `Event` suffix | `WiFiUpdateEvent` |
| State | PascalCase + `State` suffix | `WiFiState` |
| Services | Priority prefix + name | `030-wifi` |
| Notification IDs | `service:description` | `wifi:connection_failed` |
| Icon IDs | `service:name` | `wifi:state` |
| Environment vars | `UBO_` prefix | `UBO_LOG_LEVEL` |

## Code Quality Checklist

Before marking work complete:
- [ ] Type hints on all functions
- [ ] `from __future__ import annotations` at top
- [ ] No mutation (use `replace()` for state updates)
- [ ] Async tasks use `create_task` from `ubo_app.utils.async_`
- [ ] Single quotes for strings, double for docstrings
- [ ] Linting passes: `uv run poe lint`
- [ ] Type checking passes: `uv run poe typecheck`
- [ ] No hardcoded values (use constants)
- [ ] Proper Redux patterns (actions, events, reducers)

## Linting

Project uses ruff with ALL rules enabled:

```bash
# Check linting
uv run poe lint

# Auto-fix issues
uv run poe lint --fix
```

## Type Checking

Project uses pyright:

```bash
# Run type checker
uv run poe typecheck
```

For type-only imports, use `TYPE_CHECKING`:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ubo_app.store.main import RootState
```
