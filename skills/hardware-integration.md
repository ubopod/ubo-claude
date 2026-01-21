---
name: hardware-integration
description: Hardware integration patterns for ubo_app on Raspberry Pi: service ownership, GPIO/sensors/camera/audio/LEDs, async/thread boundaries, safety, cleanup, and testing/mocking.
---

# Hardware Integration (ubo_app / Raspberry Pi)

Hardware integration in `ubo_app` must be **safe**, **isolated per service**, and **deterministic**. Hardware bugs are often nondeterministic (timing, power, drivers), so the architecture needs strict boundaries and clean shutdown behavior.

---

## Core rules (CRITICAL)

### 1) Hardware access is owned by a service

- ✅ One service owns each hardware domain (camera, keypad, audio, LEDs, sensors).
- ✅ Other code interacts via **actions/events/state**, not direct driver calls.
- ❌ Don’t share device handles across services.

### 2) Never block the service event loop on hardware I/O

If the hardware API is blocking or slow:

- use `asyncio.to_thread(...)`, or
- run a dedicated worker that serializes access.

### 3) Always clean up

Hardware resources must be released on shutdown:

- GPIO cleanup
- camera/audio handle close
- subprocess termination
- file descriptor close

---

## Safety & robustness

- **Fail closed**: if hardware init fails, disable that capability and expose a clear status, rather than crashing the whole app.
- **Timeout external calls**: camera capture, network-dependent drivers, device discovery.
- **Retry with backoff** where appropriate (hotplug, transient I/O).
- **Validate inputs** from sensors/QR/serial to avoid crashes from malformed data.

---

## GPIO patterns

Guidelines:

- Keep GPIO code in the owning service (e.g., `000-*` hardware services).
- Centralize pin assignments and document them (avoid “magic pin numbers”).
- Handle debounce for mechanical inputs (keypad/buttons).

Cleanup pattern (conceptual):

```python
def cleanup_gpio() -> None:
    # release pins, stop PWM, etc.
    ...
```

---

## Sensors (I2C/SPI/1-Wire/etc.)

Guidelines:

- Serialize access to a shared bus (SPI/I2C) to prevent collisions.
- Normalize readings (units, rounding, error codes) before emitting actions/events.
- Rate-limit updates to avoid overwhelming the store/UI (e.g., emit at 2–10 Hz, not 200 Hz).

---

## Camera patterns

Guidelines:

- Camera access should be single-owner; camera hardware is easy to deadlock if opened twice.
- Prefer a “camera worker” that handles requests via a queue and emits results via actions/events.
- Time out capture operations and surface failure state explicitly.

Testing:

- Prefer a mock camera fixture that can simulate QR scans and image capture.
- Support dev-mode QR injection via files (e.g., `/tmp/qrcode_input.txt` / `/tmp/qrcode_input.png`) if present in your codebase.

---

## Audio patterns (playback/TTS/mic)

Guidelines:

- Treat audio device access as exclusive (avoid multiple writers).
- Ensure volume/mute state is store-driven and consistent.
- For TTS/STT, isolate heavy work in background tasks and stream results via events.

---

## LEDs / RGB ring

Guidelines:

- Model LEDs as a state machine (idle/listening/thinking/error) to avoid conflicting animations.
- Use a single animator task that consumes intents (actions/events) and renders at a controlled FPS.
- Always provide a “safe default” pattern on failure (e.g., off or error blink).

---

## Async/thread boundaries

Guidelines:

- Create background tasks using `ubo_app.utils.async_.create_task` (so they run on the correct loop).
- For blocking drivers, offload via `asyncio.to_thread`.
- Never update Kivy widgets directly from hardware threads; dispatch actions and let UI react, or schedule on Kivy’s thread.

---

## Testing hardware-integrated services

### What to test off-device

- reducers and state transitions
- service logic with mocked drivers
- serialization/queue behavior (no concurrent camera opens)
- error handling + retries/backoff

### What to test on-device

- timing-sensitive GPIO behavior
- camera driver quirks
- audio latency / device selection
- SPI display behaviors

Guidelines:

- Mock driver boundaries; do not “half talk” to real devices in CI.
- Keep on-device tests clearly marked and optional.

---

## Common pitfalls (and fixes)

- **Two services touch same hardware**
  - Fix: consolidate ownership; expose state/actions instead.
- **Blocking calls inside async worker**
  - Fix: `asyncio.to_thread` or move blocking I/O into dedicated thread worker.
- **Resource leaks**
  - Fix: deterministic cleanup; cancel tasks; close handles.
- **High-frequency sensor spam**
  - Fix: rate-limit / debounce; emit only meaningful changes.

---

## Quick checklist

- Hardware ownership is clear (one service per device domain)
- No cross-service driver calls
- Blocking I/O is offloaded
- Updates are rate-limited/debounced
- Shutdown cancels tasks and releases resources
- Tests use mocks off-device; device-only tests are explicit

