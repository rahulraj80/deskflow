# Deskflow Paths Reference

This file records absolute and relative paths for critical components, tools, and configuration files in the Deskflow project.

**Last Updated:** 2026-05-21

---

## System & Tool Paths

| Tool/Path | Value | Notes |
|-----------|-------|-------|
| **CMake** | `/usr/bin/cmake` | >= 3.24 required |
| **Git** | `/usr/bin/git` | For version info in builds |
| **Qt 6** | System install | >= 6.7.0 required |
| **libei** | System install | >= 1.3 required (Linux/Wayland) |
| **libportal** | System install | >= 0.9.1 required (Linux/Wayland) |
| **OpenSSL** | System install | >= 3.0 required |

---

## Project Structure

| Directory | Path | Purpose |
|-----------|------|---------|
| **Root** | `/home/ubuntu/scripts/deskflow` | Project root |
| **Apps** | `/home/ubuntu/scripts/deskflow/src/apps/` | Applications (GUI, daemon, core) |
| **Lib** | `/home/ubuntu/scripts/deskflow/src/lib/` | Core libraries |
| **Tests** | `/home/ubuntu/scripts/deskflow/src/unittests/` | Unit tests |
| **Notes** | `/home/ubuntu/scripts/deskflow/notes/` | Planning, learnings, specs |
| **CMake** | `/home/ubuntu/scripts/deskflow/cmake/` | CMake modules and config |
| **Deploy** | `/home/ubuntu/scripts/deskflow/deploy/` | Deployment scripts |
| **Docs** | `/home/ubuntu/scripts/deskflow/docs/` | Documentation |
| **Translations** | `/home/ubuntu/scripts/deskflow/translations/` | i18n translation files |

---

## Key Source Directories

| Directory | Path | Purpose |
|-----------|------|---------|
| **Platform (Linux/Wayland)** | `src/lib/platform/Ei*.cpp` | libei/portal Wayland screen implementation |
| **Platform (Linux/X11)** | `src/lib/platform/XWindows*.cpp` | X11 screen implementation |
| **Platform (Windows)** | `src/lib/platform/MSWindows*.cpp` | Windows screen implementation |
| **Platform (macOS)** | `src/lib/platform/OSX*.cpp/.mm` | macOS screen implementation |
| **Deskflow Core** | `src/lib/deskflow/` | Core abstractions (IScreen, IPlatformScreen, etc.) |
| **Server** | `src/lib/server/` | Server logic (Server.cpp, PrimaryClient.cpp, ClientProxy*.cpp) |
| **Client** | `src/lib/client/` | Client logic (Client.cpp) |
| **Base** | `src/lib/base/` | Base types (EventTypes.h, Log, etc.) |
| **Net** | `src/lib/net/` | Network layer (TCPSocket, SecureSocket, etc.) |
| **GUI** | `src/lib/gui/` | GUI components |

---

## Key Files

| File | Path | Purpose |
|------|------|---------|
| **Root CMakeLists.txt** | `/home/ubuntu/scripts/deskflow/CMakeLists.txt` | Top-level build config |
| **Event Types** | `src/lib/base/EventTypes.h` | All event type definitions (SSOT) |
| **Protocol Types** | `src/lib/deskflow/ProtocolTypes.h` | Protocol message types (SSOT) |
| **EiScreen** | `src/lib/platform/EiScreen.cpp` | Linux Wayland/libei screen (touch input here) |
| **EiScreen Header** | `src/lib/platform/EiScreen.h` | EiScreen class definition |
| **Server** | `src/lib/server/Server.cpp` | Main server logic (event routing) |
| **Server Header** | `src/lib/server/Server.h` | Server class definition |
| **PrimaryClient** | `src/lib/server/PrimaryClient.cpp` | Server's local screen handler |
| **ClientProxy** | `src/lib/server/ClientProxy.cpp` | Client connection handler |
| **ClientProxy1_0** | `src/lib/server/ClientProxy1_0.cpp` | Protocol v1.0 client handler |
| **InputFilter** | `src/lib/server/InputFilter.cpp` | Input filtering rules |
| **IScreen** | `src/lib/deskflow/IScreen.h` | Screen interface |
| **IPlatformScreen** | `src/lib/deskflow/IPlatformScreen.h` | Platform screen interface |
| **Screen** | `src/lib/deskflow/Screen.cpp` | Screen implementation |
| **Client** | `src/lib/client/Client.cpp` | Client-side logic |

---

## Build Commands

```bash
# Configure
cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug

# Build
cmake --build build

# Run tests
ctest --test-dir build

# Clean
cmake --build build --target clean
```

---

## Notes

- Whenever a new important path or tool is added, update this file.
- Keep this file synchronized with actual locations.
- This file is **not exhaustive**; add entries as the project grows.
