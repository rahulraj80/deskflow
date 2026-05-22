# Project: Deskflow -- Implementation Safety Protocol

**Purpose:** This document defines a disciplined, step-by-step process for fixing bugs and making changes to the Deskflow codebase. It enforces strict consistency checks to prevent regressions in this multi-platform (Windows/Linux/macOS) C++ project.

**Audience:** Any AI agent (Hermes, Claude, GPT-4, etc.) tasked with working on this project must follow this protocol exactly.

**Golden Rule:** *Never assume consistency. Always verify. If verification fails, stop and fix before proceeding.*

---

## 0. Project Overview

Deskflow is a keyboard and mouse sharing utility (fork of Synergy/Barrier). It allows sharing a single keyboard and mouse across multiple machines.

**Key characteristics:**
- C++20 project built with CMake 3.24+
- Qt 6.7+ for GUI components
- Multi-platform: Windows (Win32), Linux (X11/Wayland/libei), macOS
- Core protocol: custom TCP protocol between server and client
- Server = the machine with the physical keyboard/mouse
- Client = the machine receiving input events

**Repository:** https://github.com/deskflow/deskflow

---

## 1. Core Safety Mechanisms

### 1.1 Single Source of Truth (SSOT) Strategy
All shared definitions MUST originate from exactly one location:

| Type | SSOT Location | Consumers |
|------|---------------|-----------|
| Event types | `src/lib/base/EventTypes.h` | All platform screens, server, client |
| Protocol messages | `src/lib/deskflow/ProtocolTypes.h` | Server, client, all ClientProxy versions |
| Clipboard IDs | `src/lib/deskflow/Clipboard.h` | Server, client, platform screens |
| Key IDs / masks | `src/lib/deskflow/KeyTypes.h` | Key state classes, server, client |
| Button IDs | `src/lib/deskflow/ButtonTypes.h` | Platform screens, server, client |

**Enforcement:** Before writing any code in a component, check if the needed type/constant/config already exists in the SSOT. If yes, import it. If no, add it to SSOT FIRST, then consume it.

### 1.2 Build System Rules
- CMake 3.24+ required. Never bypass CMake with manual compiler invocations.
- All source files must be registered in the appropriate `CMakeLists.txt`.
- When adding a new file, update the nearest `CMakeLists.txt` immediately.
- Use `CMAKE_CXX_STANDARD 20` — no exceptions.

### 1.3 Multi-Platform Rules
- **Never** use platform-specific code without `#ifdef` guards (`_WIN32`, `__APPLE__`, `__linux__`).
- All platform-specific code lives in `src/lib/platform/`.
- Abstract interfaces live in `src/lib/deskflow/` (e.g., `IScreen.h`, `IPlatformScreen.h`).
- When fixing a bug on one platform, check if the same issue exists on other platforms.

### 1.4 Protocol Compatibility
- The server-client protocol is versioned. Changes to message formats must maintain backward compatibility.
- All protocol changes must be reflected in ALL `ClientProxy1_*.cpp` versions.
- Never change the wire format without updating the protocol version.

---

## 2. Pre-Construction: Foundation Validation

**Before writing any code, execute these checks in order:**

### Step 0: Environment Sanity Check
```bash
# Verify all required tools
cmake --version    # >= 3.24
git --version
git status         # Should be clean or you know what's changed
```

### Step 1: Understand the Bug/Task
- Read the bug report or task description thoroughly.
- Identify which component(s) are affected (server, client, platform screen, etc.).
- Identify which platform(s) are affected (Windows, Linux, macOS).
- Search the codebase for relevant code paths.
- Document your understanding in `notes/DISCUSS.md` before touching any code.

### Step 2: Read Relevant Code
Before modifying any file:
- Read the file you plan to modify in full (or at least the relevant sections).
- Read the callers of any function you plan to change.
- Read the interface/base class that defines the contract.
- Check if there are existing tests for the code you're modifying.

### Step 3: Create a Test Plan
- Define what "fixed" means — specific, observable criteria.
- Identify how to verify the fix (build, unit test, manual test).
- If possible, write a failing test first (TDD approach).

---

## 3. Construction: Bug Fix Protocol

**For bug fixes, follow this order:**

### Phase A: Root Cause Analysis
1. Identify the exact code path that causes the bug.
2. Trace the flow from input (e.g., touch event) to output (e.g., key event sent to client).
3. Document the root cause in `notes/DISCUSS.md`.
4. Confirm your understanding with the user before proceeding.

### Phase B: Minimal Fix
1. Make the smallest possible change that fixes the bug.
2. Do NOT refactor adjacent code.
3. Do NOT "improve" unrelated code.
4. Add comments explaining WHY the fix works, not just WHAT it does.

### Phase C: Verification
1. Build the project: `cmake --build build`
2. Run relevant unit tests: `ctest --test-dir build`
3. Verify the fix doesn't break other platforms.
4. Update `notes/DISCUSS.md` with the fix details.

### Phase D: Documentation
1. Update `notes/CONVERSATION_LOG.md` with the decision and fix.
2. Update `notes/CONTINUATION.md` if there are follow-up tasks.
3. If the fix reveals a pattern worth remembering, update `notes/LEARNINGS.md`.

---

## 4. Consistency Enforcement

### 4.1 Cross-Platform Checks
When modifying platform-specific code, verify:
- [ ] Windows (`src/lib/platform/MSWindows*.cpp`) — does the same issue exist?
- [ ] Linux/X11 (`src/lib/platform/XWindows*.cpp`) — does the same issue exist?
- [ ] Linux/Wayland (`src/lib/platform/Ei*.cpp`) — does the same issue exist?
- [ ] macOS (`src/lib/platform/OSX*.cpp/.mm`) — does the same issue exist?

### 4.2 Event Flow Checks
When modifying event handling, trace the full path:
1. Platform screen captures input → sends event via `sendEvent()`
2. Event queue dispatches to handler
3. Server/client processes the event
4. Server forwards to client (or client forwards to server)
5. Receiving side synthesizes the input

Verify each step is correct for the specific event type.

---

## 5. Git Discipline

**Rule: Commit and Push After Every Significant Change**

### What is "Significant"?
- Bug fixes
- New files or functions
- Configuration changes (CMakeLists.txt, etc.)
- Changes that affect build or behavior

### How to Commit
```bash
cd /home/ubuntu/scripts/deskflow
git add -A
git commit -m "<date> – <brief description>

- Specific change 1
- Specific change 2
- Verification: <what passed>"
git push origin master
```

### Documentation to Update
- `notes/LEARNINGS.md` — Technical findings, what worked, what didn't
- `notes/DISCUSS.md` — Decisions, open questions, risks
- `notes/CONTINUATION.md` — What to do next if interrupted

---

## 6. Error Handling & Rollback

- If a fix introduces a regression, revert immediately.
- If you're unsure about a change, create a new branch: `git checkout -b fix/<description>`
- Never leave the project in a broken state. If you can't fix it, document what you tried and why it failed.

---

## 7. Model Instructions: How to Use This Document

1. **Read this document in full** before starting any task.
2. **Follow the Pre-Construction checks** before writing code.
3. **Follow the Construction phases** (A → B → C → D) in order.
4. **Fail loud** — if you can't complete a step, say so explicitly.
5. **Update the notes files** as you work — they are your memory.

---

**Last updated:** 2026-05-21
**Status:** Active — touchscreen input forwarding bug fix in progress
