# DISCUSS.md — Active Items and Decisions

This is a living document for tracking actionables, open questions, decisions, and risks.

---

## Active Bug: Touchscreen Input Forwarding (#9770)

**Status:** In Progress
**Priority:** High
**Reported:** 2026-05-20

### Problem
When the mouse cursor is on the Linux (Wayland/libei) client machine, touchscreen input on the Windows server machine is incorrectly forwarded to the Linux client instead of being handled locally by Windows. This prevents unlocking the Windows machine via its on-screen keyboard/touchscreen.

### Setup
- Windows machine = Deskflow server
- Linux machine (Fedora 44, KDE Plasma, Wayland) = Deskflow client
- Both machines have touchscreens

### Observed Behavior
1. Mouse on Windows → touch Linux → WORKS (keystrokes stay on Linux)
2. Mouse on Linux → touch Windows → BROKEN (keystrokes forwarded to Linux instead of staying on Windows)

### Root Cause Analysis
When the mouse is on the Linux client, the Windows server's screen is "inactive" (the server's `m_active` points to the client). The Windows server captures touch input events but the deskflow server forwards these events to the Linux client instead of letting Windows handle them locally.

The key issue is in how the server handles input events from its own (now inactive) screen. Touch events from the server's physical screen should be handled locally when the active screen is a client.

### Investigation Notes
- `EiScreen.cpp` line 812: `// we don't care about touch` — EI_DEVICE_CAP_TOUCH is NOT bound in the libei seat capabilities
- The Linux client (EiScreen) does NOT process touch events at all
- The Windows server (MSWindowsScreen) DOES receive touch events
- The server's event routing in `Server.cpp` determines whether events are handled locally or forwarded to the client
- Key functions to investigate:
  - `Server::handleKeyDownEvent()` — how key events are routed
  - `Server::handleButtonEvent()` — how button events are routed
  - `Server::handleMotionEvent()` — how motion events are routed
  - `PrimaryClient::enter()` / `PrimaryClient::leave()` — screen activation/deactivation
  - The `m_active` pointer in Server — determines which screen is currently active

### Possible Fix Directions
1. **Server-side:** When the server's screen is inactive, prevent forwarding of certain input events (touch-generated key events) to the client.
2. **Client-side (EiScreen):** Add touch support to the Linux client by binding `EI_DEVICE_CAP_TOUCH` in the libei seat capabilities.
3. **InputFilter:** Add a rule to filter out touch-generated events when the server screen is inactive.

### Open Questions
- How does the server distinguish between touch-generated key events and physical keyboard events?
- Is there a flag or property on the event that indicates it came from a touchscreen?
- Should the fix be in the server's event routing or in the platform screen's event generation?
- Does the Windows MSWindowsScreen properly tag touch-generated input events?

### Decisions
- (To be filled as decisions are made)

### Related Files
- `src/lib/server/Server.cpp` — main server event routing
- `src/lib/server/Server.h` — server class definition
- `src/lib/server/PrimaryClient.cpp` — server's local screen handler
- `src/lib/platform/MSWindowsScreen.cpp` — Windows screen (touch input source)
- `src/lib/platform/EiScreen.cpp` — Linux Wayland screen (touch input sink)
- `src/lib/platform/EiEventQueueBuffer.cpp` — Wayland event queue
- `src/lib/deskflow/IScreen.h` — screen interface
- `src/lib/base/EventTypes.h` — event type definitions

---

## General Notes

### Project Conventions
- C++20, CMake 3.24+, Qt 6.7+
- Platform code in `src/lib/platform/`, abstract interfaces in `src/lib/deskflow/`
- All event types defined in `src/lib/base/EventTypes.h` (SSOT)
- Protocol messages in `src/lib/deskflow/ProtocolTypes.h` (SSOT)

### Build Info
- Build with: `cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug`
- Test with: `ctest --test-dir build`
