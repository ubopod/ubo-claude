---
description: Generate and run end-to-end tests with pytest. Creates test flows, runs tests, captures window snapshots, and validates UI state.
---

# E2E Command

This command invokes the **e2e-runner** agent to generate, maintain, and execute end-to-end tests for ubo_app using pytest and window snapshots.

## What This Command Does

1. **Generate Test Flows** - Create pytest tests for user flows
2. **Run E2E Tests** - Execute tests in Docker environment
3. **Capture Snapshots** - Window snapshots for UI validation
4. **Compare Results** - Verify UI renders correctly against baselines
5. **Identify Flaky Tests** - Detect timing-dependent failures

## When to Use

Use `/e2e` when:
- Testing critical user journeys (menu navigation, settings, service interactions)
- Verifying multi-step flows work end-to-end
- Testing UI interactions with hardware services
- Validating Redux state changes reflect in UI
- Preparing for release deployment

## How It Works

The e2e-runner agent will:

1. **Analyze user flow** and identify test scenarios
2. **Generate pytest test** using ubo_app's testing patterns
3. **Run tests in Docker** (avoids DPI mismatch issues)
4. **Capture window snapshots** for UI verification
5. **Generate report** with results and snapshot diffs
6. **Identify flaky tests** and recommend fixes

## Example Usage

```
User: /e2e Test the WiFi connection flow

Agent (e2e-runner):
# E2E Test Generation: WiFi Connection Flow

## Test Scenario Identified

**User Journey:** Main Menu → Settings → WiFi → Select Network → Enter Password → Connect

**Test Coverage:**
1. Navigate to WiFi settings menu
2. Verify available networks list
3. Select a network
4. Enter password via input
5. Verify connection status
6. Verify state updates in Redux store

## Generated Test Code

The following illustrates how a flow test is structured (based on actual `tests/flows/test_wifi.py`):

```python
# tests/flows/test_wifi.py (simplified example)
"""Test the wireless flow."""

from __future__ import annotations

from typing import TYPE_CHECKING

import pytest

from ubo_app.utils import IS_RPI

if TYPE_CHECKING:
    from headless_kivy_pytest.fixtures import WindowSnapshot
    from redux_pytest.fixtures import StoreSnapshot, WaitFor

    from tests.fixtures import AppContext, LoadServices, MockCamera, Stability
    from tests.fixtures.menu import WaitForMenuItem, WaitForEmptyMenu
    from ubo_app.store.main import RootState
    from ubo_app.store.services.wifi import WiFiState


@pytest.mark.timeout(200)
@pytest.mark.skipif(not IS_RPI, reason='Only runs on Raspberry Pi')
async def test_setup_flow(
    app_context: AppContext,
    window_snapshot: WindowSnapshot,
    store_snapshot: StoreSnapshot[RootState],
    load_services: LoadServices,
    stability: Stability,
    wait_for: WaitFor,
    camera: MockCamera,
    wait_for_menu_item: WaitForMenuItem,
    wait_for_empty_menu: WaitForEmptyMenu,
) -> None:
    """Test the wireless flow."""
    from ubo_app.store.core.types import (
        MenuChooseByIconAction,
        MenuChooseByLabelAction,
        MenuGoBackAction,
    )
    from ubo_app.store.main import store

    def store_snapshot_selector(state: RootState) -> WiFiState:
        return state.wifi

    app_context.set_app()
    unload_waiter = await load_services(
        ['camera', 'display', 'notifications', 'wifi'],
        run_async=True,
    )

    await stability()
    store_snapshot.take(selector=store_snapshot_selector)

    # Navigate: Main menu -> Settings -> Network -> WiFi
    store.dispatch(MenuChooseByIconAction(icon='󰍜'))
    await stability()

    store.dispatch(MenuChooseByLabelAction(label='Settings'))
    await stability()

    store.dispatch(MenuChooseByLabelAction(label='Network'))
    await stability()

    store.dispatch(MenuChooseByLabelAction(label='WiFi'))
    await stability()
    window_snapshot.take()

    # Select "Add" to add a new connection
    store.dispatch(MenuChooseByLabelAction(label='Add'))
    await stability()
    window_snapshot.take()

    # Set QR Code image before camera is started
    camera.set_image('qrcode/wifi')

    # Select "QR code" input method
    store.dispatch(MenuChooseByIconAction(icon='󰄀'))
    await stability()
    window_snapshot.take()

    # Scan QR code
    store.dispatch(MenuChooseByIconAction(icon='󰄀'))
    window_snapshot.take()

    # Wait for connection and verify
    await wait_for_menu_item(label='ubo-test-ssid', icon='󱚽')
    store_snapshot.take(selector=store_snapshot_selector)
    window_snapshot.take()

    await unload_waiter()
```

## Running Tests

```bash
# Build Docker test image first
uv run poe build-docker-images

# Run flow tests in Docker (recommended)
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- -svv tests/flows/test_wifi.py

# Run integration tests
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- -svv tests/integration/test_core.py
```

Example output:
```
tests/flows/test_wifi.py::test_setup_flow PASSED

  1 passed in 45.2s

Snapshots captured in tests/flows/results/
```

## Test Report

```
+--------------------------------------------------------------+
|                    E2E Test Results                          |
+--------------------------------------------------------------+
| Status:     PASSED                                           |
| Total:      3 tests                                          |
| Passed:     3 (100%)                                         |
| Failed:     0                                                |
| Duration:   4.2s                                             |
+--------------------------------------------------------------+

Snapshots:
  New:      0
  Updated:  0
  Matched:  4
  Failed:   0

Update snapshots in Docker:
  docker run ... -- --override-window-snapshots --override-store-snapshots
```

E2E test suite ready for CI integration!
```

## Test Artifacts

When tests run, the following artifacts are captured:

**Snapshot Location:** `tests/<category>/results/<test_file>/<test_name>/`

Example: `tests/flows/results/test_wifi/setup_flow/`

**On All Tests:**
- Window snapshots: `window-000.png`, `window-001.png`, etc.
- Store snapshots: `store-000.json`, `store-001.json`, etc.

**On Failure Only:**
- Snapshot diff images (expected vs actual)
- Redux state dump at failure point

## Viewing Artifacts

Snapshots are stored in `tests/<category>/results/<test_file>/<test_name>/`.

```bash
# View window snapshots
ls tests/flows/results/test_wifi/setup_flow/
open tests/flows/results/test_wifi/setup_flow/window-000.png

# Update snapshots (must run in Docker for consistency)
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- --override-window-snapshots --override-store-snapshots \
  tests/flows/test_wifi.py

# Generate new screenshots
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- --make-screenshots tests/flows/
```

## Flaky Test Detection

If a test fails intermittently:

```
  FLAKY TEST DETECTED: tests/flows/test_wifi_connection.py::test_user_can_connect

Test passed 7/10 runs (70% pass rate)

Common failure:
"AssertionError: Snapshot mismatch - animation frame captured"

Recommended fixes:
1. Add explicit wait: await app_context.wait_for_animation_complete()
2. Increase state wait timeout: wait_for_state(..., timeout=5.0)
3. Mock time-dependent animations
4. Check for race conditions in async handlers

Quarantine: Mark as @pytest.mark.skip(reason='flaky') until fixed
```

## Docker Testing Environment

Tests MUST run in Docker to ensure consistent snapshots:

```bash
# Build test image
uv run poe build-docker-images

# Run all E2E tests (basic)
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test

# Run specific tests with full volume mounts (recommended)
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  -v ubo-app-dev-uv-local:/root/.local/share/uv \
  -v ubo-app-dev-uv-venv:/ubo-app/.venv \
  ubo-app-test -- -svv tests/flows/

# Run with snapshot override flags
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- -svv --make-screenshots --override-store-snapshots --override-window-snapshots

# Interactive debugging
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test bash
# Then inside container: pytest tests/flows/test_wifi.py -v --pdb
```

**Why Docker?**
- Avoids DPI mismatch issues on macOS (known issue with different DPI settings)
- Consistent snapshot baselines across machines
- Matches CI environment
- Proper headless Kivy rendering

## Running Tests on Device

For testing on a physical Raspberry Pi device:

```bash
# One-time setup: copy test files and install dependencies
uv run poe device:test:copy
uv run poe device:test:deps

# Run tests on device
uv run poe device:test
```

**Note:** Requires SSH config for `ubo-development-pod` host. See README for setup instructions.

## Simulating Hardware Input

For testing flows that require hardware input:

```python
# QR Code input - write to file
@pytest.fixture
def qr_code_input(tmp_path):
    """Simulate QR code scan."""
    qr_file = Path('/tmp/qrcode_input.txt')
    qr_file.write_text('wifi:S:MyNetwork;T:WPA;P:password123;;')
    yield
    qr_file.unlink(missing_ok=True)

# Or use PNG for image-based QR
qr_image = Path('/tmp/qrcode_input.png')
```

## Ubo App Critical Flows

For ubo_app, prioritize these E2E tests:

**CRITICAL (Must Always Pass):**
1. Boot sequence completes
2. Main menu navigation works
3. WiFi connection flow (but only if running on ubo pod device, otherwise this will be skipped)
4. Settings modification persists
5. Service start/stop works
6. Docker container management
7. System update flow

**IMPORTANT:**
1. SSH enable/disable
2. Display brightness adjustment
3. Audio volume control
4. Camera capture
5. Sensor readings display
6. Web UI accessibility

## Best Practices

**DO:**
- Run tests in Docker for consistent snapshots
- Use `create_task` from `ubo_app.utils.async_` in test fixtures
- Wait for Redux state changes, not arbitrary timeouts
- Test critical user journeys end-to-end
- Mock hardware at the abstraction layer
- Use `wait_for_state()` for async assertions

**DON'T:**
- Run snapshot tests outside Docker (DPI mismatch)
- Use `time.sleep()` for waiting
- Test implementation details
- Skip snapshot review on failures
- Ignore flaky tests
- Run tests that modify real hardware state

## Important Notes

**CRITICAL for ubo_app:**
- Always run in Docker for consistent snapshots (macOS has different DPI settings that cause mismatches)
- Mock hardware services, never test against real GPIO
- Use `/tmp/qrcode_input.txt` or `/tmp/qrcode_input.png` for QR code simulation
- Tests should clean up any state changes
- Service threads must be properly stopped after tests
- Type checking is best run on the actual device or with stubs from [ubo-non-rpi-stubs](https://github.com/ubopod/ubo-non-rpi-stubs)

## Integration with Other Commands

- Use `/plan` to identify critical flows to test
- Use `/tdd` for unit tests of reducers and actions
- Use `/e2e` for integration and user journey tests
- Use `/code-review` to verify test quality

## Related Agents

This command invokes the `e2e-runner` agent located at:
`~/.claude/agents/e2e-runner.md`

## Quick Commands

```bash
# Build Docker test images (required first)
uv run poe build-docker-images

# Run all tests in Docker
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test

# Run specific test file (real examples)
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/flows/test_wifi.py -svv

docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/integration/test_core.py -svv

# Update/override snapshots
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- --override-window-snapshots --override-store-snapshots

# Generate new screenshots
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- --make-screenshots

# Run on device (one-time setup)
uv run poe device:test:copy
uv run poe device:test:deps

# Run tests on device
uv run poe device:test

# Local run (snapshots may not match on macOS due to DPI)
uv run poe test
```
