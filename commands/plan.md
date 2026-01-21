---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
---

# Plan Command

This command invokes the **planner** agent to create a comprehensive implementation plan before writing any code.

## What This Command Does

1. **Restate Requirements** - Clarify what needs to be built
2. **Identify Risks** - Surface potential issues and blockers
3. **Create Step Plan** - Break down implementation into phases
4. **Wait for Confirmation** - MUST receive user approval before proceeding

## When to Use

Use `/plan` when:
- Starting a new service
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/components will be affected
- Requirements are unclear or ambiguous
- Adding new Redux actions/events
- Modifying hardware abstraction layers

## How It Works

The planner agent will:

1. **Analyze the request** and restate requirements in clear terms
2. **Break down into phases** with specific, actionable steps
3. **Identify dependencies** between components
4. **Assess risks** and potential blockers
5. **Estimate complexity** (High/Medium/Low)
6. **Present the plan** and WAIT for your explicit confirmation

## Example Usage

```
User: /plan I need to add a new service that handles Bluetooth connectivity

Agent (planner):
# Implementation Plan: Bluetooth Service

## Requirements Restatement
- Add a new service to manage Bluetooth connectivity on Raspberry Pi
- Support device discovery, pairing, and connection management
- Expose state through Redux store for UI consumption
- Provide actions for scanning, pairing, and disconnecting devices

## Implementation Phases

### Phase 1: State and Actions
- Define BluetoothState in `ubo_app/store/services/bluetooth.py`
- Create actions: BluetoothScanAction, BluetoothPairAction, BluetoothDisconnectAction
- Create events: BluetoothDeviceDiscoveredEvent, BluetoothConnectionStatusEvent
- Run `uv run poe proto` to regenerate proto files

### Phase 2: Reducer Implementation
- Create `ubo_app/services/030-bluetooth/reducer.py`
- Handle state transitions for scanning, pairing, connected states
- Maintain device list in immutable state tree
- Register reducer with combine_reducers in store setup

### Phase 3: Service Implementation
- Create `ubo_app/services/030-bluetooth/setup.py`
- Use `create_task` from `ubo_app.utils.async_` for background scanning
- Implement D-Bus integration via `sdbus` for BlueZ
- Add hardware abstraction for non-RPi development (mock Bluetooth)

### Phase 4: UI Integration
- Create menu items in `ubo_app/services/030-bluetooth/ubo_handle.py`
- Add Bluetooth settings page with device list
- Implement scan button and pairing flow
- Use notification IDs with `bluetooth:` prefix

### Phase 5: Testing
- Add unit tests in `tests/store/test_bluetooth.py`
- Create integration tests in `tests/integration/`
- Test on both RPi and desktop (with mocks)

## Dependencies
- `sdbus` for D-Bus communication with BlueZ
- BlueZ daemon running on Raspberry Pi
- Mock implementation for desktop development

## Risks
- HIGH: BlueZ D-Bus API complexity and async handling
- MEDIUM: Thread safety when communicating with main store
- MEDIUM: Hardware variations between RPi 4 and RPi 5
- LOW: Service priority conflicts (030-xxx namespace)

## Estimated Complexity: MEDIUM-HIGH
- State/Actions: Phase 1
- Reducer: Phase 2
- Service: Phase 3 (largest effort)
- UI: Phase 4
- Testing: Phase 5

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## Important Notes

**CRITICAL**: The planner agent will **NOT** write any code until you explicitly confirm the plan with "yes" or "proceed" or similar affirmative response.

If you want changes, respond with:
- "modify: [your changes]"
- "different approach: [alternative]"
- "skip phase 2 and do phase 3 first"

## Ubo App Specific Considerations

When planning for ubo_app, always consider:

1. **Service Priority** - Use correct prefix (000-0xx for hardware, 030-0xx for networking, etc.)
2. **Redux Flow** - Actions → Reducer → State → UI (never direct method calls)
3. **Async Patterns** - Use `create_task` from `ubo_app.utils.async_` for background work
4. **Hardware Abstraction** - Plan mock implementations for desktop development
5. **Protobuf Workflow** - Run `uv run poe proto` when actions/events change
6. **Thread Isolation** - Services run in isolated threads with dedicated event loops

## Integration with Other Commands

After planning:
- Use `/tdd` to implement with test-driven development
- Use `/build-and-fix` if build errors occur
- Use `/code-review` to review completed implementation

## Related Agents

This command invokes the `planner` agent located at:
`~/.claude/agents/planner.md`
