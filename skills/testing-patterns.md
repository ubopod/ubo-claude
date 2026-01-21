---
name: testing-patterns
description: ubo_app testing patterns using pytest + pytest-asyncio, redux_pytest + headless_kivy_pytest snapshots, core fixtures (AppContext/WaitFor/Stability), fakefs, and local vs Docker vs device workflows.
---

# Testing Patterns (ubo_app)

This skill documents the **actual** testing setup in `ubo_app` and the preferred patterns for writing stable tests for a Kivy + Redux + multi-service system.

---

## Testing stack (repo-verified)

- **Runner**: `pytest` (via `uv run poe test`)
- **Async**: `pytest-asyncio` (auto mode)
- **Snapshots**:
  - UI/window snapshots via `headless_kivy_pytest`
  - Store snapshots/monitors via `redux_pytest`
- **Retry/waits**: `tenacity` + `redux_pytest.fixtures.WaitFor`
- **Mocking**: `pytest-mock`
- **Fake FS (optional)**: `pyfakefs` via `--use-fakefs`

---

## Test organization (repo-verified layout)

```
tests/
├── conftest.py
├── fixtures/
│   ├── app.py              # AppContext
│   ├── load_services.py
│   ├── menu.py
│   ├── mock_camera.py      # MockCamera (QR injection via /tmp/qrcode_input.png)
│   ├── mock_environment.py
│   ├── stability.py        # stability() helper built on snapshots + WaitFor
│   └── store.py
├── integration/
├── flows/
├── store/
└── reproduction/
```

---

## How tests start/stop the app (CRITICAL)

Use `AppContext` for full app tests.

- `app_context.set_app()` starts `MenuApp.async_run(async_lib='asyncio')` as an asyncio task.
- Cleanup dispatches `FinishAction()` to the store, waits for handlers, joins the scheduler, cancels Kivy Clock events, stops the app, and waits for `ubo_app.service.worker_thread` to finish.

Practical implication:

- ✅ Prefer `AppContext` over manually creating `MenuApp`.
- ✅ Always ensure tests leave the system “finished” (no leaked tasks).

---

## Core fixtures you should use

These are wired in `tests/conftest.py`.

- **`app_context`**: app lifecycle management (`AppContext`)
- **`wait_for`**: deterministic polling/retry (`redux_pytest.fixtures.WaitFor`)
- **`store_snapshot`**: store snapshot comparisons (`redux_pytest`)
- **`store_monitor`**: assert dispatched actions/events (`redux_pytest.fixtures.StoreMonitor`)
- **`window_snapshot`**: UI snapshot comparisons (`headless_kivy_pytest.fixtures.WindowSnapshot`)
- **`stability`**: wait until both store + window stabilize (uses hashes + snapshot json)
- **`camera`**: mock camera / QR input (`MockCamera`)
- **`mock_environment`**: environment mocking

---

## Running tests (repo-verified commands)

```bash
# Run all tests
uv run poe test

# Target a file
uv run poe test -- tests/integration/test_core.py

# Target a single test
uv run poe test -- tests/integration/test_core.py::test_app_runs_and_exits

# Fake filesystem (optional)
uv run poe test -- --use-fakefs
```

---

## Deterministic waits (avoid `sleep`)

Prefer `wait_for` over arbitrary sleeps.

Pattern:

```python
from tenacity import stop_after_attempt

async def test_menu_loaded(app_context, wait_for):
    app_context.set_app()

    @wait_for(run_async=True, stop=stop_after_attempt(5))
    def stack_loaded() -> None:
        assert len(app_context.app.menu_widget.stack) > 0

    await stack_loaded()
```

---

## Snapshot testing (window + store)

Use snapshots for stable regressions, especially UI or state shape regressions.

Pattern:

```python
async def test_with_snapshots(app_context, window_snapshot, store_snapshot, stability):
    app_context.set_app()
    await stability()

    window_snapshot.take()
    store_snapshot.take()
```

Guidelines:

- Snapshot after `stability()` so you don’t capture transient transitions.
- Keep ordering deterministic (sort lists where appropriate).
- Avoid including secrets in store state/log output (see `skills/security-review/SKILL.md`).

---

## Persistent store defaults + marker (repo-verified)

`tests/conftest.py` writes defaults to the persistent store for each test, and you can override via marker:

```python
import pytest

@pytest.mark.persistent_store({
    'wifi_has_visited_onboarding': False,
    'speech_recognition:is_assistant_active': True,
})
async def test_with_custom_persistent_store(app_context):
    app_context.set_app()
```

---

## QR / camera testing (repo-verified)

Mock QR input by writing a test image to:

- `/tmp/qrcode_input.png`

The `camera` fixture provides:

```python
async def test_qr_flow(app_context, camera):
    app_context.set_app()
    camera.set_image('qrcode/wifi')  # copies tests/data/qrcode/wifi.png -> /tmp/qrcode_input.png
```

---

## Local vs Docker vs on-device

- **Local**: fastest feedback for reducers + service logic with mocks.
- **Docker**: most consistent for headless UI and snapshots.
  - Build images via `uv run poe build-docker-images`.
- **On-device**: only when you need real GPIO/camera/audio/SPI behavior.
  - Use `uv run poe device:test` and friends.

---

## Quick checklist

- Use `AppContext` for app tests; don’t hand-roll app lifecycle
- Use `wait_for` for async conditions; avoid sleeps
- Use `stability()` before snapshots
- Keep tests independent (no shared state across tests)
- Use markers/fixtures for persistent store overrides

