---
description: Enforce test-driven development workflow. Scaffold interfaces, generate tests FIRST, then implement minimal code to pass.
---

# TDD Command

This command invokes the **tdd-guide** agent to enforce test-driven development methodology.

## What This Command Does

1. **Scaffold Interfaces** - Define state types and actions first
2. **Generate Tests First** - Write failing tests (RED)
3. **Implement Minimal Code** - Write just enough to pass (GREEN)
4. **Refactor** - Improve code while keeping tests green (REFACTOR)
5. **Verify Coverage** - Run with pytest-cov

## When to Use

Use `/tdd` when:
- Implementing new reducers or actions
- Adding new state types
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## How It Works

The tdd-guide agent will:

1. **Define state types** using `Immutable` and actions using `BaseAction`
2. **Write tests that will FAIL** (because code doesn't exist yet)
3. **Run tests** and verify they fail for the right reason
4. **Write minimal implementation** to make tests pass
5. **Run tests** and verify they pass
6. **Refactor** code while keeping tests green
7. **Check coverage** with pytest-cov

## TDD Cycle

```
RED → GREEN → REFACTOR → REPEAT

RED:      Write a failing test
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## Example Usage

```
User: /tdd I need a reducer to handle sensor temperature readings

Agent (tdd-guide):
# TDD Session: Temperature Sensor Reducer

## Step 1: Define Interface (SCAFFOLD)

```python
# ubo_app/store/services/temperature.py
"""Temperature sensor state and actions."""

from __future__ import annotations

from immutable import Immutable
from redux import BaseAction, BaseEvent


class TemperatureReading(Immutable):
    celsius: float
    timestamp: float
    sensor_id: str


class TemperatureAction(BaseAction): ...


class TemperatureUpdateAction(TemperatureAction):
    reading: TemperatureReading


class TemperatureAlertAction(TemperatureAction):
    threshold: float


class TemperatureEvent(BaseEvent): ...


class TemperatureAlertEvent(TemperatureEvent):
    reading: TemperatureReading
    threshold: float


class TemperatureState(Immutable):
    current_reading: TemperatureReading | None = None
    alert_threshold: float = 80.0
    readings_history: tuple[TemperatureReading, ...] = ()
```

## Step 2: Write Failing Test (RED)

```python
# tests/store/test_temperature.py
"""Tests for temperature sensor reducer."""

from __future__ import annotations

import pytest
from redux import InitAction, InitializationActionError

from ubo_app.store.services.temperature import (
    TemperatureAlertAction,
    TemperatureAlertEvent,
    TemperatureReading,
    TemperatureState,
    TemperatureUpdateAction,
)


class TestTemperatureReducer:
    """Test temperature reducer state transitions."""

    def test_init_returns_default_state(self) -> None:
        """Test reducer initializes with default state."""
        from ubo_app.services.000_sensors.reducer import reducer

        result = reducer(None, InitAction())

        assert isinstance(result.state, TemperatureState)
        assert result.state.current_reading is None
        assert result.state.alert_threshold == 80.0
        assert result.state.readings_history == ()

    def test_init_with_wrong_action_raises_error(self) -> None:
        """Test reducer raises error on non-init action when state is None."""
        from ubo_app.services.000_sensors.reducer import reducer

        with pytest.raises(InitializationActionError):
            reducer(None, TemperatureUpdateAction(reading=TemperatureReading(
                celsius=25.0,
                timestamp=1000.0,
                sensor_id='sensor-1',
            )))

    def test_update_action_sets_current_reading(self) -> None:
        """Test update action sets current reading and adds to history."""
        from ubo_app.services.000_sensors.reducer import reducer

        state = TemperatureState()
        reading = TemperatureReading(
            celsius=25.5,
            timestamp=1000.0,
            sensor_id='sensor-1',
        )

        result = reducer(state, TemperatureUpdateAction(reading=reading))

        assert result.state.current_reading == reading
        assert reading in result.state.readings_history

    def test_alert_event_emitted_when_threshold_exceeded(self) -> None:
        """Test alert event is emitted when temperature exceeds threshold."""
        from ubo_app.services.000_sensors.reducer import reducer

        state = TemperatureState(alert_threshold=30.0)
        reading = TemperatureReading(
            celsius=35.0,  # Exceeds threshold
            timestamp=1000.0,
            sensor_id='sensor-1',
        )

        result = reducer(state, TemperatureUpdateAction(reading=reading))

        assert any(isinstance(e, TemperatureAlertEvent) for e in result.events)

    def test_no_alert_when_below_threshold(self) -> None:
        """Test no alert event when temperature is below threshold."""
        from ubo_app.services.000_sensors.reducer import reducer

        state = TemperatureState(alert_threshold=30.0)
        reading = TemperatureReading(
            celsius=25.0,  # Below threshold
            timestamp=1000.0,
            sensor_id='sensor-1',
        )

        result = reducer(state, TemperatureUpdateAction(reading=reading))

        assert not any(isinstance(e, TemperatureAlertEvent) for e in result.events)

    def test_alert_threshold_can_be_changed(self) -> None:
        """Test alert threshold can be updated."""
        from ubo_app.services.000_sensors.reducer import reducer

        state = TemperatureState(alert_threshold=80.0)

        result = reducer(state, TemperatureAlertAction(threshold=50.0))

        assert result.state.alert_threshold == 50.0
```

## Step 3: Run Tests - Verify FAIL

```bash
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/store/test_temperature.py -svv

FAILED tests/store/test_temperature.py::TestTemperatureReducer::test_init_returns_default_state
    ModuleNotFoundError: No module named 'ubo_app.services.000_sensors.reducer'

6 tests failed
```

Tests fail as expected. Ready to implement.

## Step 4: Implement Minimal Code (GREEN)

```python
# ubo_app/services/000-sensors/reducer.py
"""Temperature sensor reducer."""

from __future__ import annotations

from dataclasses import replace

from redux import (
    BaseAction,
    CompleteReducerResult,
    InitAction,
    InitializationActionError,
    ReducerResult,
)

from ubo_app.store.services.temperature import (
    TemperatureAction,
    TemperatureAlertAction,
    TemperatureAlertEvent,
    TemperatureEvent,
    TemperatureState,
    TemperatureUpdateAction,
)


def reducer(
    state: TemperatureState | None,
    action: TemperatureAction,
) -> ReducerResult[TemperatureState, BaseAction, TemperatureEvent]:
    if state is None:
        if isinstance(action, InitAction):
            return CompleteReducerResult(state=TemperatureState())
        raise InitializationActionError(action)

    match action:
        case TemperatureUpdateAction():
            new_history = (*state.readings_history, action.reading)
            events: list[TemperatureEvent] = []

            if action.reading.celsius > state.alert_threshold:
                events.append(
                    TemperatureAlertEvent(
                        reading=action.reading,
                        threshold=state.alert_threshold,
                    )
                )

            return CompleteReducerResult(
                state=replace(
                    state,
                    current_reading=action.reading,
                    readings_history=new_history,
                ),
                events=events,
            )

        case TemperatureAlertAction():
            return CompleteReducerResult(
                state=replace(state, alert_threshold=action.threshold),
            )

        case _:
            return state

    return state
```

## Step 5: Run Tests - Verify PASS

```bash
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/store/test_temperature.py -svv

PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_init_returns_default_state
PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_init_with_wrong_action_raises_error
PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_update_action_sets_current_reading
PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_alert_event_emitted_when_threshold_exceeded
PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_no_alert_when_below_threshold
PASSED tests/store/test_temperature.py::TestTemperatureReducer::test_alert_threshold_can_be_changed

6 tests passed
```

All tests passing!

## Step 6: Check Coverage

```bash
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/store/test_temperature.py --cov=ubo_app.services.000_sensors --cov-report=term-missing

Name                                      Stmts   Miss  Cover
-------------------------------------------------------------
ubo_app/services/000-sensors/reducer.py      28      0   100%

Coverage: 100%
```

TDD session complete!
```

## TDD Best Practices

**DO:**
- Write the test FIRST, before any implementation
- Run tests and verify they FAIL before implementing
- Write minimal code to make tests pass
- Refactor only after tests are green
- Use `Immutable` for state classes
- Use `replace()` for immutable state updates
- Test reducer state transitions explicitly

**DON'T:**
- Write implementation before tests
- Skip running tests after each change
- Write too much code at once
- Ignore failing tests
- Test implementation details (test behavior)
- Mutate state directly (always use `replace()`)

## Test Types for ubo_app

**Unit Tests** (Reducer-level):
- Initial state creation (`InitAction`)
- State transitions for each action type
- Event emission conditions
- Edge cases (empty, None, boundary values)

**Store Tests** (State slice tests):
- Action/event combinations
- Cross-reducer interactions
- Selector functions

**Integration Tests** (Service-level):
- Service setup and teardown
- Event subscriptions
- Hardware mock interactions

**Flow Tests** (use `/e2e` command):
- Critical user journeys
- Multi-step UI flows

## Running Tests

```bash
# Build Docker test images (required first)
uv run poe build-docker-images

# Run specific test file
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/store/test_temperature.py -svv

# Run with coverage
docker run --rm -it --name ubo-app-test \
  -v .:/ubo-app \
  -v ubo-app-dev-uv-cache:/root/.cache/uv \
  ubo-app-test -- tests/store/ --cov=ubo_app --cov-report=term-missing

# Run locally (faster iteration, but snapshots may differ)
uv run poe test tests/store/test_temperature.py
```

## Important Notes

**MANDATORY**: Tests must be written BEFORE implementation. The TDD cycle is:

1. **RED** - Write failing test
2. **GREEN** - Implement to pass
3. **REFACTOR** - Improve code

Never skip the RED phase. Never write code before tests.

**ubo_app Specifics:**
- State classes must inherit from `Immutable`
- Actions inherit from `BaseAction`
- Events inherit from `BaseEvent`
- Reducers return `ReducerResult` or `CompleteReducerResult`
- Use `replace()` from dataclasses for state updates
- Run `uv run poe proto` after adding new actions/events

## Integration with Other Commands

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/e2e` for flow tests with UI snapshots
- Use `/code-review` to review implementation

## Related Agents

This command invokes the `tdd-guide` agent located at:
`~/.claude/agents/tdd-guide.md`
