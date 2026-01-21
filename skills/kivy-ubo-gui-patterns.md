---
name: kivy-ubo-gui-patterns
description: UI patterns for ubo_app using Kivy + ubo-gui + headless-kivy: MenuApp structure, navigation, store-driven UI, threading rules, rendering performance, and snapshot-test stability.
---

# Kivy / ubo-gui UI Patterns (ubo_app)

`ubo_app` renders a Kivy UI on an LCD/SPI display (often via `headless-kivy`, without X11). The UI must stay **responsive**, **store-driven**, and **deterministic** for snapshot tests.

This skill defines how to build screens/menus/pages, wire them to the Redux store, and avoid common Kivy pitfalls in a multi-threaded, asyncio-heavy app.

---

## Core rule: UI is reactive to Store state

- ✅ UI reads from **store state** and reacts to state changes.
- ✅ UI initiates change by dispatching **actions**.
- ❌ UI should not call into services directly for business logic.

If the user presses a button:

1. UI dispatches an action (e.g., `WifiConnectRequested`)
2. Service reacts asynchronously and dispatches results (`WifiConnected`, `WifiFailed`)
3. Reducer updates state
4. UI updates from state

---

## UI structure (recommended mental model)

At a high level, the “Menu App” is composed of:

- **Header**: status icons, clock, connectivity state
- **Central**: the active page/menu content (stack)
- **Footer**: contextual actions (back/home/confirm), hints

Keep responsibilities crisp:

- Header/Footer: **pure presentation** + action dispatch
- Central: **navigation + page composition**

---

## Threading & scheduling (CRITICAL)

Kivy UI work must happen on the **Kivy/main thread**.

Rules:

- ❌ Don’t update Kivy widgets from service threads.
- ✅ When reacting to store changes from a background thread, schedule UI updates via Kivy’s scheduler (typically `Clock`).

Example pattern:

```python
from kivy.clock import Clock

def on_store_update(new_state) -> None:
    def apply_ui_update(_dt: float) -> None:
        # read from new_state; update widgets here
        ...

    Clock.schedule_once(apply_ui_update, 0)
```

If you need to do heavier work in response to UI input:

- keep UI handler small
- dispatch an action
- let a service do the expensive work

---

## Navigation patterns (menu stack)

### Prefer stack-based navigation

- Push pages/menus onto a stack
- Pop on back
- Keep “home/root” explicit

Guidelines:

- Navigation should be expressible as **state** (e.g., current route/page id + params).
- Avoid hidden widget state being the source of truth for “where the app is.”

### Keep pages small and composable

Pages should be small Kivy widgets composed of smaller widgets.

Avoid:

- pages that directly manage multiple unrelated domains (wifi + audio + camera)
- deeply nested layouts that are hard to snapshot and hard to maintain

---

## Rendering performance on small devices

Raspberry Pi + LCD/SPI means:

- avoid expensive layout recalculation
- avoid constant re-creating widgets
- keep frame work bounded

Guidelines:

- Prefer updating **properties** over reconstructing widget trees.
- Debounce rapid updates (e.g., network RSSI) so UI updates at a controlled rate.
- For list-like UIs, reuse row widgets when feasible (virtualization if needed).

---

## State → UI mapping patterns

### Pattern: selector function per widget

Instead of sprinkling state lookups everywhere, define small “selectors”:

```python
def select_wifi_status(state) -> str:
    return state.services.wifi.status

def select_header_icons(state):
    return state.status_icons.visible
```

This improves:

- testability (selectors are pure)
- consistency (same mapping used across UI)

### Avoid “two-way binding” complexity

Keep directionality clear:

- UI updates from state (render)
- User interactions dispatch actions (intent)

---

## Input handling (keypad/touch)

Guidelines:

- Translate raw input into **semantic actions** (e.g., `NavigateBack`, `SelectItem(index)`)
- Keep repeat/long-press logic out of widgets when possible; centralize it
- Always handle “back/home” consistently across pages

---

## Headless rendering considerations

Headless environments can differ from desktop:

- DPI/metrics differences impact snapshots
- some providers/drivers behave differently (fonts, text measurement)

Guidelines:

- Keep fonts and layout deterministic (avoid runtime font discovery where possible).
- Prefer Docker/headless runs for snapshot consistency.

---

## Snapshot test stability

To keep window snapshots stable:

- Avoid rendering timestamps or “live” values unless mocked/frozen in tests.
- Normalize dynamic text (e.g., “Connected in 3s”) for snapshots.
- When showing lists, keep ordering deterministic.

If using a “stability” fixture:

- wait for the UI to settle before snapshotting
- avoid arbitrary sleeps; wait for known conditions

---

## Common pitfalls (and fixes)

- **Updating widgets from service thread**
  - Fix: schedule via `Clock.schedule_once`
- **UI logic doing network/hardware calls**
  - Fix: dispatch action; let service handle it
- **Rebuilding widget trees every tick**
  - Fix: update properties; cache widgets
- **Flaky snapshots**
  - Fix: freeze time, normalize dynamic strings, run in Docker/headless

---

## Quick checklist

- UI reads from **store state**, writes via **actions**
- All widget updates happen on **Kivy/main thread**
- Navigation is **stack-based** and consistent
- UI handlers are lightweight; services do heavy work
- Snapshots are deterministic (time/ordering normalized)

