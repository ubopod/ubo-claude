---
name: tdd-workflow
description: TDD workflow for ubo_app (Python 3.11): pytest + pytest-asyncio, reducer-first tests, service/worker tests with mocks, snapshot strategy, and when to run Docker vs on-device.
---

# TDD Workflow (ubo_app)

This skill is the default workflow for **new features**, **bug fixes**, and **refactors** in `ubo_app`.

The core idea: **write the test that proves the behavior first**, then implement the smallest change to pass, then refactor safely.

---

## Core principles (non-negotiable)

- **Tests before code** for new behavior.
- **Reducers first**: if behavior changes state, start with a reducer test (fast, pure).
- **Side effects are isolated**: hardware/network/filesystem work is tested with mocks and boundaries.
- **Deterministic tests**: avoid flaky sleeps; use `wait_for`/fixtures/state predicates.
- **Cancellation matters**: long-running workers must cancel cleanly; test it when relevant.

---

## Test types (ubo_app specific)

- **Unit**: reducers, selectors, pure utilities (fastest).
- **Integration-ish**: service workers with mocked drivers, store/event subscriptions.
- **System**: app context tests (UI + store + services), plus window/store snapshots when stable.
- **On-device**: only for real GPIO/camera/audio/SPI behaviors that can’t be simulated reliably.

---

## Workflow steps

### Step 0: Define the behavior

Write a short spec in one of these forms:

**User journey**
```
As a user, I want to [action], so that [benefit].
```

**Service story**
```
When [input/event] occurs, service [X] dispatches [action], state becomes [Y], and UI shows [Z].
```

### Step 1: Write the smallest failing test

Pick the lowest layer that proves the behavior:

- state change → reducer test
- service reaction → worker test (with mocks)
- UI regression → snapshot/system test

Examples of “smallest” tests:

- reducer returns updated immutable state
- worker emits `XSucceeded` / `XFailed` action
- store subscription receives an event

### Step 2: Run tests (confirm failure)

Use your repo’s task runner (typical patterns):

```bash
uv run poe test -- -q
```

Or target a single file/test:

```bash
uv run poe test -- tests/path/test_file.py::test_name -q
```

### Step 3: Implement minimal code to pass

Rules while implementing:

- reducers remain pure (no I/O)
- background work uses `ubo_app.utils.async_.create_task`
- service `setup()` returns promptly
- add only what’s needed to satisfy the test

### Step 4: Re-run tests (green)

```bash
uv run poe test -- -q
```

### Step 5: Refactor (still green)

- remove duplication
- improve naming
- extract helpers/selectors
- add missing edge-case tests

### Step 6: Add the next failing test (iterate)

Repeat until the behavior is complete.

---

## Coverage expectations (pragmatic)

There isn’t a single magic number; aim for **high confidence**:

- reducers: essentially 100% for changed paths
- workers: cover happy path + error path + cancellation (if applicable)
- snapshots: only when they’re stable and provide regression value

If you add a risky feature (network exposure, filesystem ops, privileged scripts), add **explicit security tests** or reproduction tests.

---

## Snapshot workflow (window + store)

Use snapshots for stable regressions:

- store snapshots: state shape/values
- window snapshots: UI rendering/layout state

Guidelines:

- normalize time/dynamic values
- run snapshot tests in Docker/headless for consistent DPI
- update snapshots only intentionally (explicit flags)

---

## Local vs Docker vs on-device test strategy

- **Local**: fastest for unit tests + mocked service tests.
- **Docker (recommended)**: consistent environment for headless UI + snapshots.
  - Build images: `uv run poe build-docker-images`
- **On-device**: hardware truth tests (GPIO/camera/audio/SPI) and performance.
  - Run: `uv run poe device:test` (and related `device:test:*` tasks)

## “Sanity” command (repo-verified)

Run the full local gate:

```bash
uv run poe sanity
```

---

## “Definition of done” checklist

- [ ] Tests added for new behavior (lowest possible layer)
- [ ] Error paths covered (including invalid inputs)
- [ ] Cancellation/shutdown safe where relevant
- [ ] No flake sleeps; deterministic waits/fixtures used
- [ ] Lint/typecheck still pass (ruff/pyright)
- [ ] Docs updated if behavior affects workflows or service contracts

