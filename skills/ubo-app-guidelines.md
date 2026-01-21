---
name: ubo-app-guidelines
description: Project-specific guidelines for ubo_app - architecture, file structure, conventions, testing, and development workflow for the Raspberry Pi IoT platform.
---

# Ubo App Project Guidelines

Comprehensive guide for developing on the `ubo_app` codebase - a Python application for Raspberry Pi that provides a unified interface for hardware-integrated apps.

## Architecture Overview

**Tech Stack:**

- **Language**: Python 3.11 (requires >=3.11, <3.12)
- **State Management**: `python-redux` - Redux pattern with immutable state
- **UI**: `ubo-gui` built on `kivy` for LCD display
- **Rendering**: `headless-kivy` for SPI display without X11
- **Async**: Python asyncio with custom task orchestration
- **Package Manager**: `uv` (fast Python package manager)
- **Build**: `hatchling` with VCS versioning
- **Task Runner**: `poethepoet` (poe)

**Core Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      Menu App (Kivy UI)                      │
│            MenuAppHeader / MenuAppCentral / MenuAppFooter    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    UboStore (python-redux)                   │
│     Centralized state, Actions → Reducers → Events           │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Services │   │ Services │   │ Services │
        │  (000-*) │   │  (030-*) │   │  (090-*) │
        │ Hardware │   │ Network  │   │   Apps   │
        └──────────┘   └──────────┘   └──────────┘
```

**Key Principles:**

- Event-driven and reactive architecture
- Services communicate via Redux actions/events, NOT direct method calls
- Each service runs in its own isolated thread with dedicated event loop
- Immutable state updates only

---

## File Structure

```
ubo_app/
├── main.py                      # App entry point
├── service.py                   # Service registration
├── service_thread.py            # Service lifecycle management (~700 lines)
├── setup.py                     # App initialization
├── constants.py                 # Global constants
├── display.py                   # Display utilities
├── logger.py                    # Logging configuration
│
├── store/                       # Redux state management
│   ├── main.py                  # Root store configuration (~600 lines)
│   ├── core/                    # Core state (menus, navigation)
│   ├── services/                # Service-specific state slices
│   ├── settings/                # App settings state
│   ├── input/                   # Input state
│   ├── status_icons/            # Status bar icons
│   └── update_manager/          # Update state
│
├── services/                    # Modular services (21+)
│   ├── 000-audio/               # Hardware: audio
│   ├── 000-display/             # Hardware: LCD display
│   ├── 000-keypad/              # Hardware: keypad input
│   ├── 010-notifications/       # Core: notification system
│   ├── 010-speech-synthesis/    # Core: TTS (piper-tts)
│   ├── 020-keyboard/            # Input: on-screen keyboard
│   ├── 030-ethernet/            # Network: ethernet
│   ├── 030-ip/                  # Network: IP management
│   ├── 030-wifi/                # Network: WiFi/hotspot
│   ├── 040-camera/              # Sensors: camera
│   ├── 040-rgb-ring/            # Sensors: LED ring
│   ├── 040-sensors/             # Sensors: temp, light
│   ├── 050-lightdm/             # Access: display manager
│   ├── 050-rpi-connect/         # Access: RPi Connect
│   ├── 050-ssh/                 # Access: SSH
│   ├── 050-users/               # Access: user management
│   ├── 050-vscode/              # Access: VS Code tunnel
│   ├── 080-docker/              # Apps: Docker containers
│   ├── 090-assistant/           # Advanced: voice AI
│   ├── 090-file-system/         # Advanced: file operations
│   ├── 090-infrared/            # Advanced: IR control
│   ├── 090-speech-recognition/  # Advanced: Vosk STT
│   └── 090-web-ui/              # Advanced: web dashboard
│
├── menu_app/                    # Kivy UI application
│   └── menu.py                  # Main MenuApp class
│
├── rpc/                         # gRPC bindings
│   ├── proto/                   # Protocol buffer definitions
│   └── ubo_bindings/            # Generated Python bindings
│
├── engines/                     # AI engine abstractions
│
├── utils/                       # Shared utilities
│   ├── async_.py                # Async task management (CRITICAL)
│   ├── service.py               # Service utilities
│   └── persistent_store.py      # Data persistence
│
└── system/                      # System-level integrations
    ├── scripts/                 # Installation scripts
    └── system_manager/          # Root-privilege operations
```

**Service Priority Numbering:**

- `000-*`: Hardware drivers (audio, display, keypad)
- `010-*`: Core services (notifications, speech)
- `020-*`: Input services (keyboard)
- `030-*`: Network services (wifi, ethernet, IP)
- `040-*`: Sensor services (camera, RGB ring, sensors)
- `050-*`: Remote access (SSH, VS Code, users)
- `080-*`: Application services (Docker)
- `090-*`: Advanced services (assistant, web-ui)

---

## Service Structure

Each service follows a standard structure:

```
services/XXX-service-name/
├── __init__.py
├── ubo_handle.py      # Service registration (REQUIRED)
├── reducer.py         # Redux reducer for state
├── setup.py           # Hardware initialization (optional)
└── ...                # Additional modules
```

### `ubo_handle.py` Pattern

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ubo_handle import ReducerRegistrar, register
    from ubo_app.utils.types import Subscriptions


def setup(register_reducer: ReducerRegistrar) -> Subscriptions | None:
    # IMPORTANT: avoid imports in global scope of ubo_handle.py
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

### `reducer.py` Pattern

```python
from dataclasses import dataclass
from python_immutable import Immutable

@dataclass(frozen=True)
class ServiceState(Immutable):
    """Immutable state for this service."""

    value: str = ''

def reducer(
    state: ServiceState | None,
    action: Action,
) -> ReducerResult[ServiceState, ResultAction, BaseEvent]:
    if isinstance(action, InitAction):
        return ServiceState()
    # Handle other actions...
    return state
```

---

## Conventions

### Naming

- **Environment variables**: `UBO_` prefix (e.g., `UBO_DEBUG`)
- **Notification IDs**: `ubo:` for core, `<service_name>:` for services
- **Icon IDs**: Same as notifications
- **Service IDs**: Lowercase, hyphenated (e.g., `speech-synthesis`)

### Code Style

- **Linter**: `ruff` with ALL rules enabled
- **Type checker**: `pyright`
- **Quotes**: Single quotes for strings, double for docstrings
- **Immutability**: All state objects must be `@dataclass(frozen=True)`

### Async Patterns

**CRITICAL**: Use `create_task` from `ubo_app.utils.async_`, NOT `asyncio.create_task`:

```python
# CORRECT
from ubo_app.utils.async_ import create_task

create_task(my_coroutine())

# WRONG - runs in wrong event loop
import asyncio

asyncio.create_task(my_coroutine())
```

**Service setup must complete** - no infinite loops or forever-awaiting:

```python
# CORRECT - background task via create_task
async def setup():
    register_reducer(reducer)
    create_task(background_worker())  # OK: runs in background
    return  # Setup completes

# WRONG - blocks forever
async def setup():
    register_reducer(reducer)
    while True:  # BAD: never completes
        await do_something()
```

---

## Development Workflow

### Setup

```bash
# Clone with LFS
git clone https://github.com/ubopod/ubo_app.git
git lfs install && git lfs pull

# Create venv (use --system-site-packages on RPi)
uv venv --system-site-packages  # RPi only
uv sync --dev

# Run locally
HEADLESS_KIVY_DEBUG=true uv run ubo
```

### Common Commands

```bash
# Linting
uv run poe lint           # Check
uv run poe lint --fix     # Auto-fix

# Type checking
uv run poe typecheck

# Testing
uv run poe test                    # Local
docker run ... ubo-app-test        # Docker (recommended)
uv run poe device:test             # On-device

# Protobuf
uv run poe proto                   # Generate + compile

# Device deployment
uv run poe device:deploy           # Deploy code
uv run poe device:deploy:restart   # Deploy + restart
uv run poe device:deploy:complete  # Full setup
```

### Build Docker Images

```bash
uv run poe build-docker-images
docker run --rm -it -v .:/ubo-app ubo-app-test
```

### Web App (for `090-web-ui` service)

```bash
cd ubo_app/services/090-web-ui/web-app
npm install
npm run proto:compile
npm run build
# Or watch mode: npm run build:watch
```

---

## Testing

### Framework

- **pytest** with `pytest-asyncio` (auto mode)
- **Timeout**: 160 seconds per test
- **Snapshots**: Window and store state snapshots

### Test Commands

```bash
# Run all tests
uv run poe test

# With pytest args
uv run poe test -- -svv --make-screenshots

# Override snapshots
uv run poe test -- --override-store-snapshots --override-window-snapshots
```

### Key Fixtures

```python
from tests.fixtures import AppContext, StoreMonitor, wait_for

async def test_example(app_context: AppContext):
    # Test with full app context
    await wait_for(lambda: condition_met())
```

### QR Code Testing

In dev environment without camera, write to:

- `/tmp/qrcode_input.txt` - text content
- `/tmp/qrcode_input.png` - QR code image

---

## Critical Rules

1. **Immutability**: All state must be immutable (`@dataclass(frozen=True)`)
2. **Async**: Always use `create_task` from `ubo_app.utils.async_`
3. **Services**: Setup functions must complete (no infinite loops)
4. **Communication**: Services communicate via actions/events, not direct calls
5. **Type hints**: Required for all public APIs
6. **No hardcoded secrets**: Use environment variables

---

## Related Skills

- `coding-standards.md`: General coding best practices (this repo’s version is TS/JS/React-oriented; we can add a Python-specific standards skill next if you want).
- `backend-patterns.md`: General backend patterns (useful for services + APIs; some sections may be web-centric).

## Planned (optional) add-on skills

If you want these split out into smaller, reusable docs, we can add them one-by-one:

- `python-coding-standards.md`
- `service-architecture.md`
- `redux-store-patterns.md`
- `async-patterns.md`
- `testing-patterns.md`

