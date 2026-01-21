---
name: redux-store-patterns
description: python-redux patterns for ubo_app: actions, state slices, reducers, events, side-effect boundaries, and testing guidance.
---

# Redux Store Patterns (ubo_app / python-redux)

`ubo_app` uses a Redux-style architecture: **Actions → Reducers → State (+ Events)**.

This doc focuses on **store conventions** so services remain isolated, state stays immutable, and behavior is testable.

---

## Mental model

- **Action**: an intent or occurrence that drives state transitions (input to reducers and effects).
- **Reducer**: a pure function that computes the next immutable state from (prev_state, action).
- **State**: the current immutable snapshot (root state + per-slice/service state).
- **Event** (if used in your codebase): a notification emitted as a result of an action/state change, consumed by services/UI.

**Rule**: reducers are **pure**; all side effects happen outside reducers (in service workers/tasks).

---

## State organization (slices)

Organize state into cohesive slices:

- **Core**: navigation/menus/input/status icons
- **Settings**: persisted config/preferences
- **Service slices**: each service owns its slice under something like `store/services/...`

Guidelines:

- Keep slice state **small and stable**.
- Prefer **simple primitives** and **tuples** over mutable collections.
- Avoid storing heavyweight runtime objects in state (open sockets, file handles, camera objects).

---

## Action design

### Naming & structure

- Action names should reflect domain intent: `WifiConnected`, `VolumeSet`, `KeyPressed`, `NotificationQueued`.
- Prefer **dataclass-based** action types when available (better typing, better debugging).
- Keep payloads minimal; don’t cram huge blobs into actions unless necessary.

### Action categories (useful convention)

- **Input actions**: from user/hardware input (keypad, touchscreen, QR scan)
- **System actions**: timers, connectivity changes, boot/shutdown signals
- **Service actions**: internal service outputs (device detected, job completed)
- **UI actions**: navigation/page changes

---

## Reducer rules (non-negotiable)

Reducers must:

- Be **pure** (no I/O, no network, no hardware, no filesystem, no time).
- Perform **immutable updates only**.
- Return **the original state** when no change is needed.
- Be explicit about initialization (`state is None` or an `InitAction`).

Example skeleton:

```python
from dataclasses import dataclass, replace
from python_immutable import Immutable

@dataclass(frozen=True)
class ServiceState(Immutable):
    status: str = 'idle'
    last_error: str | None = None

def reducer(state: ServiceState | None, action):
    if state is None:
        return ServiceState()

    if isinstance(action, SomethingStarted):
        return replace(state, status='running', last_error=None)

    if isinstance(action, SomethingFailed):
        return replace(state, status='error', last_error=action.message)

    return state
```

### Immutable update patterns

Prefer:

- `dataclasses.replace(state, field=value)`
- constructing a new dataclass

Avoid:

- mutating lists/dicts in-place
- storing mutable objects inside state and mutating them later

---

## Events (if your architecture uses them)

Use events when you need to notify observers without encoding everything as state.

Examples:

- “play a sound now”
- “show toast notification”
- “trigger a one-off UI animation”

Guidelines:

- Events should be **small** and **serializable-ish** (no open handles).
- Avoid using events as a workaround for missing state modeling.

---

## Side effects: where they belong

Side effects (hardware, network, filesystem, subprocesses) belong in:

- service background workers/tasks
- effect handlers / middleware (if your store has that concept)

Pattern:

1. Worker observes external world
2. Worker dispatches an action
3. Reducer updates state
4. Observers react (UI/services), possibly dispatching more actions

**Do not**: call hardware APIs inside reducers.

---

## Autoruns (reactive subscriptions)

Autoruns are reactive subscriptions that run a callback whenever **selected state** changes. They are the primary mechanism for services and UI to react to state transitions.

### Basic usage

```python
from ubo_app.store.main import store

@store.autorun(lambda state: state.wifi.connections)
def on_wifi_connections_change(connections: tuple[WifiConnection, ...]) -> None:
    """Called whenever wifi connections change."""
    logger.info('WiFi connections updated', extra={'count': len(connections)})
```

The decorator:
1. Extracts `state.wifi.connections` via the selector
2. Compares against previous value (by default using equality)
3. Calls the function **only when value changes**

### Selector patterns

**Simple selector** — single state field:

```python
@store.autorun(lambda state: state.audio.playback_volume)
def set_volume(volume: float) -> None:
    audio_manager.set_playback_volume(volume)
```

**Composite selector** — multiple fields as tuple:

```python
@store.autorun(lambda state: (state.update_manager, state.settings.beta_versions))
def check_updates(data: tuple[UpdateManagerState, bool]) -> None:
    manager_state, beta_enabled = data
    # React to either field changing
```

**Nested selector** — drilling into structures:

```python
@store.autorun(
    lambda state: (
        state.assistant.mcp_servers.get(server_id),
        server_id in state.assistant.enabled_mcp_servers,
    ),
)
def menu(state_data: tuple[McpServerMetadata | None, bool]) -> HeadedMenu:
    server, is_enabled = state_data
    ...
```

### Options

Use `AutorunOptions` for advanced control:

```python
from redux import AutorunOptions

@store.autorun(
    lambda state: external_file_mtime(),  # selector not purely from state
    options=AutorunOptions(memoization=False),  # always re-run even if value same
)
def on_file_change(mtime: float) -> dict[str, str]:
    return load_secrets_from_disk()
```

Common options:
- `memoization=False`: skip equality check, always call on store update (useful when selector depends on external values)
- `default_value=<val>`: value to use if selector returns `None` or raises
- `keep_ref=True/False`: whether to prevent garbage collection of the autorun (default `True`)

### Returning values from autoruns

Autoruns can **return computed values** that become reactive properties:

```python
@store.autorun(lambda state: state.settings.pdb_signal)
def pdb_debug_icon(pdb_signal: bool) -> str:
    return '󰱒' if pdb_signal else '󰄱'

# Later, use as a reactive value:
menu_item = UboDispatchItem(
    label='PDB Signal',
    icon=pdb_debug_icon,  # updates automatically when state changes
)
```

### Async autoruns

Autorun callbacks can be async:

```python
@store.autorun(lambda state: state.rpi_connect)
async def on_rpi_connect_change(state: RPiConnectState) -> None:
    await some_async_operation(state)
```

The store schedules these on the service's coroutine runner.

### Keeping autoruns alive

**Important**: autoruns are garbage-collected if no reference is kept. Always assign to a variable or store in a returned subscription list:

```python
def init_service() -> Subscriptions:
    @store.autorun(lambda state: state.audio.playback_volume)
    def set_volume(volume: float) -> None:
        audio_manager.set_volume(volume)

    # Keep reference alive for service lifetime
    _ = set_volume

    return [audio_manager.close]
```

Or collect them:

```python
def init_service() -> Subscriptions:
    autoruns = []

    @store.autorun(lambda state: state.audio.playback_volume)
    def set_playback_volume(volume: float) -> None:
        audio_manager.set_playback_volume(volume)
    autoruns.append(set_playback_volume)

    @store.autorun(lambda state: state.audio.capture_volume)
    def set_capture_volume(volume: float) -> None:
        audio_manager.set_capture_volume(volume)
    autoruns.append(set_capture_volume)

    # autoruns list keeps references alive
    _ = autoruns
    return []
```

### Autoruns vs event subscriptions

| Pattern | Use when |
|---------|----------|
| `@store.autorun(selector)` | React to **state changes** with computed/derived values |
| `store.subscribe_event(EventType, handler)` | React to **one-off events** (sounds, notifications, external triggers) |

**Rule of thumb**: if it's in state, use autorun. If it's ephemeral, use events.

### Best practices

- **Keep selectors cheap**: selectors run on every state update; avoid heavy computation
- **Minimize selected scope**: select only what you need, not entire slices when possible
- **Autoruns should be idempotent**: may re-run if store re-dispatches
- **Don't dispatch from selectors**: selectors must be pure; dispatch only from the callback body
- **Use `memoization=False` sparingly**: only when selector depends on external state

---

## Common patterns

### “Request / Success / Failure” triad

Use three actions for long-running work:

- `XRequested(...)`
- `XSucceeded(result=...)`
- `XFailed(message=..., code=...)`

This yields:

- clear state transitions
- straightforward tests
- easy UI loading/error states

### Idempotent reducers

If an action repeats, the reducer should not thrash state.

Example: if WiFi is already connected to the same SSID, don’t update state unnecessarily.

---

## Testing guidance

Test reducers as pure functions:

- given state + action, assert next state
- no mocks needed

Test effects/workers separately:

- mock hardware/network boundaries
- dispatch actions and assert emitted actions/events and resulting state changes

If your project uses store/window snapshots:

- prefer snapshotting **state slices** over entire app runtime objects
- write tests that wait for state transitions deterministically (avoid arbitrary sleeps)

---

## Quick checklist

- **Actions**: small, typed, meaningful names
- **State**: immutable, serializable-ish, no runtime handles
- **Reducers**: pure + deterministic + minimal updates
- **Effects**: in workers/middleware, not reducers
- **Tests**: reducer unit tests + worker integration tests

