# Bug #9770 — Touchscreen Input Forwarding

## Problem Statement

When the mouse cursor is on the Linux (Wayland/libei) client machine, touchscreen input on the Windows server machine is incorrectly forwarded to the Linux client instead of being handled locally by Windows. This prevents unlocking the Windows machine via its on-screen keyboard/touchscreen.

## Setup

- **Windows machine** = Deskflow server (has touchscreen)
- **Linux machine** = Deskflow client (Fedora 44, KDE Plasma, Wayland, has touchscreen)
- Both machines have touchscreens

## Observed Behavior

| Scenario | Mouse on | Touch on | Result |
|----------|----------|----------|--------|
| 1 | Windows server | Linux client | WORKS — keystrokes stay on Linux |
| 2 | Linux client | Windows server | BROKEN — keystrokes forwarded to Linux instead of staying on Windows |

## Root Cause Analysis

When the mouse is on the Linux client, the Windows server's screen is "inactive" (the server's `m_active` points to the client). The Windows server captures touch input events via the Windows hook chain, but the deskflow server forwards these events to the Linux client instead of letting Windows handle them locally.

The key issue is in how the server handles input events from its own (now inactive) screen. Touch events from the server's physical screen should be handled locally when the active screen is a client.

### Key Findings

1. **EiScreen.cpp (Linux client)** — Line 812 explicitly says "we don't care about touch" — `EI_DEVICE_CAP_TOUCH` is NOT bound in the libei seat capabilities. The Linux client does NOT process touch events at all.

2. **MSWindowsScreen (Windows server)** — DOES receive touch events through the Windows hook chain (`MSWindowsHook.cpp`).

3. **Server.cpp event routing** — Determines whether events are handled locally or forwarded to the client. When `m_active` points to a client, events from the server's own screen are forwarded to that client.

4. **The fundamental problem** — There is no mechanism to distinguish between:
   - Touch-generated key events (from on-screen keyboard) that should stay on the server
   - Physical keyboard events that should be forwarded to the client

## Investigation Path

### Files Examined

| File | Role |
|------|------|
| `src/lib/server/Server.cpp` | Main server event routing |
| `src/lib/server/Server.h` | Server class definition |
| `src/lib/server/PrimaryClient.cpp` | Server's local screen handler |
| `src/lib/platform/MSWindowsScreen.cpp` | Windows screen (touch input source) |
| `src/lib/platform/MSWindowsHook.cpp` | Windows low-level keyboard hook |
| `src/lib/platform/EiScreen.cpp` | Linux Wayland/libei screen |
| `src/lib/platform/EiEventQueueBuffer.cpp` | Wayland event queue |
| `src/lib/deskflow/IScreen.h` | Screen interface |
| `src/lib/base/EventTypes.h` | Event type definitions |

### Key Functions

- `Server::handleKeyDownEvent()` — how key events are routed
- `Server::handleButtonEvent()` — how button events are routed
- `Server::handleMotionEvent()` — how motion events are routed
- `PrimaryClient::enter()` / `PrimaryClient::leave()` — screen activation/deactivation
- `MSWindowsHook::keyboardHook()` — Windows low-level keyboard hook that captures input

## Solution Approach

### Design Decision: Server-Side Single Toggle

After analysis, the cleanest solution is a **single server-side configuration option** that controls touch event forwarding for all machines:

- **OFF (default):** Touch events stay on the local machine where they originated. If you touch the Windows machine's screen, Windows handles it locally. If you touch a Linux machine's screen, Linux handles it locally. The server never forwards touch events to any client.
- **ON:** Touch events are forwarded to whichever machine has the mouse cursor (legacy behavior).

This design is superior to per-machine configuration because:
1. **Simple:** One toggle on the server, no per-machine settings needed
2. **Consistent:** All machines behave the same way — touch stays local
3. **Intuitive:** The server admin configures it once; clients don't need to know about it
4. **Scalable:** Works with any number of machines, not just two

The option is placed in the **Server Config dialog** (not the Client Config dialog) because it's a server-side setting that controls the server's behavior. Since either machine can be the server, the option is accessible from whichever machine is currently acting as server.

### Part 1: Config Option (IMPLEMENTED)

Add a new server option `forwardTouchscreenEvents` that controls whether touch-generated input events should be forwarded to clients.

**New option:** `forwardTouchscreenEvents` (OptionID: `FTSC`)
- **Default:** `false` (touch events stay on the server — the correct behavior)
- **Config file syntax:** `forwardTouchscreenEvents = true/false`
- **GUI:** New checkbox "Forward touchscreen events" in Server Config dialog

**Files changed:**
- `src/lib/deskflow/OptionTypes.h` — Added `kOptionForwardTouchscreenEvents`
- `src/lib/server/Config.cpp` — Added parsing, name, and value mapping
- `src/lib/server/Server.h` — Added `m_forwardTouchscreenEvents` member
- `src/lib/server/Server.cpp` — Added option processing in `processOptions()`
- `src/lib/gui/config/ServerConfig.h` — Added getter/setter and member variable
- `src/lib/gui/config/ServerConfig.cpp` — Added serialization, equality, commit/recall
- `src/lib/gui/dialogs/ServerConfigDialog.h` — Added toggle slot declaration
- `src/lib/gui/dialogs/ServerConfigDialog.cpp` — Added toggle handler and init
- `src/lib/gui/dialogs/ServerConfigDialog.ui` — Added checkbox widget
- `src/unittests/server/ServerConfigTests.h` — Added 3 new test declarations
- `src/unittests/server/ServerConfigTests.cpp` — Added 3 new test implementations

### Part 2: Hook Behavior Change (IMPLEMENTED)

Modified `MSWindowsHook.cpp` to check `g_forwardTouchscreenEvents` before forwarding touch-generated key events.

When the option is `false` (default):
- Touch-generated key events (identified by `LLKHF_INJECTED` flag in the keyboard hook) are passed through to Windows locally and NOT forwarded to the client
- Touch-generated mouse events (identified by `LLMHF_INJECTED` flag in the mouse hook) are passed through to Windows locally and NOT forwarded to the client
- Physical keyboard and mouse events continue to be forwarded normally
- The `LLKHF_INJECTED` flag is encoded into `lParam` bit 30 for downstream use

When the option is `true`:
- Current behavior is preserved (all events forwarded)

**Implementation note:** Both the keyboard hook AND the mouse hook must filter injected events. Touchscreen input generates both keyboard events (on-screen keyboard) and mouse events (touch taps/clicks). The original code only filtered keyboard events, which is why touch taps were still being forwarded.

**Files changed:**
- `src/lib/platform/MSWindowsHook.h` — Added `setForwardTouchscreenEvents()` declaration
- `src/lib/platform/MSWindowsHook.cpp` — Added global, setter, filtering in `keyboardLLHook`, injected flag encoding
- `src/lib/platform/MSWindowsDesks.cpp` — Added option handling in `setOptions()`

### Part 3: EiScreen Touch Support (IMPLEMENTED)

Added touch support to the Linux client by binding `EI_DEVICE_CAP_TOUCH` in the libei seat capabilities for non-primary screens.

**File changed:**
- `src/lib/platform/EiScreen.cpp` — Added `EI_DEVICE_CAP_TOUCH` binding for non-primary screens

**Note:** Touch event handlers (`EI_EVENT_TOUCH_DOWN/MOTION/UP`) exist but are currently empty (silent discard). Full touch event processing (converting to deskflow mouse events) is a future enhancement.

## Test Plan

### Unit Tests (IMPLEMENTED)

Three new tests in `ServerConfigTests`:

1. **equalityCheck_diff_forwardTouchscreenEvents** — Verifies that two Config objects differing only in `kOptionForwardTouchscreenEvents` are not equal
2. **forwardTouchscreenEvents_roundTrip** — Verifies set/get/remove on global and per-screen options
3. **forwardTouchscreenEvents_parseNameAndValue** — Verifies `getOptionName` returns "forwardTouchscreenEvents" and `getOptionValue` maps 1→"true", 0→"false"

### Integration Tests (MANUAL — Windows required)

1. Windows server + Linux client, touch on Windows → keystrokes should stay on Windows
2. Windows server + Linux client, physical keyboard → keystrokes should forward to client
3. Toggle `forwardTouchscreenEvents` option → behavior should change accordingly

## Status

| Component | Status |
|-----------|--------|
| Config option infrastructure | DONE (in patch) |
| Unit tests | DONE (in patch) |
| MSWindowsHook.cpp behavior change | DONE |
| EiScreen touch support | DONE (seat binding) |
| Manual testing on Windows | PENDING |

## Design Decision Record

### Why Server-Side Only?

**Question:** Should the `forwardTouchscreenEvents` option be on both the server and client dialogs?

**Answer:** No. The option is server-side only. Here's why:

1. **The server makes forwarding decisions.** The client never decides whether to forward events — it just receives whatever the server sends. Putting the option on the client would be misleading.

2. **Either machine can be the server.** Since deskflow is peer-to-peer (any machine can be server or client), the option is accessible from whichever machine is currently the server. No need to duplicate it.

3. **Single source of truth.** Having one toggle on the server avoids conflicting settings between machines. If both client and server had the option, which one wins?

4. **Scalable to N machines.** With multiple machines, having per-machine touch forwarding settings would be a combinatorial nightmare. One server toggle controls all machines.

### Why Default to OFF?

**Question:** Why default to `false` (don't forward) instead of `true` (forward)?

**Answer:** Because the bug report says the current behavior (always forwarding) is broken. The user expects touch events to stay on the local machine. Defaulting to `false` fixes the bug out of the box. Users who want the old behavior can opt-in by setting it to `true`.

## Related

- GitHub issue: #9770
- Patch file: `notes/patches/forward-touchscreen-events.patch`
