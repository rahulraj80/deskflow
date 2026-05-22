# CONVERSATION_LOG.md — Decisions and Findings

This file records important decisions, findings, and user comments for future reference.

---

## 2026-05-21

### Project Setup
- User forked the deskflow repo to make fixes
- Added CLAUDE.md as the project's "bible" — 12-rule coding discipline + safety protocol
- Copied notes/ folder from another project (PSync Health) — needed adaptation to deskflow
- Adapted all notes files for deskflow (C++/CMake project, not TypeScript/Next.js)

### Bug Report: Touchscreen Input Forwarding (#9770)
- **Issue:** When mouse is on Linux client, Windows server touchscreen input is forwarded to Linux instead of being handled locally
- **Impact:** Cannot unlock Windows machine via on-screen keyboard when mouse is on client
- **Platforms affected:** Windows server, Linux (Wayland) client
- **Key finding:** EiScreen.cpp line 812 explicitly says "we don't care about touch" — EI_DEVICE_CAP_TOUCH is not bound
- **Root cause hypothesis:** Server's event routing forwards touch-generated key events to client when server screen is inactive

### OpenRouter Model Visibility Question
- User asked why Hermes shows fewer OpenRouter models than opencode
- Root cause: Hermes uses a curated model catalog from Nous Research, not OpenRouter's full /models API
- Solutions discussed: add OpenRouter as custom provider, disable model catalog, or manually enumerate models
- **Deferred:** User said "we will come to model visibility in some time"

---

## 2026-05-22

### Session Restart and Context Recovery
- Session was restarted on a new machine (x86_64)
- User asked to read CLAUDE.md, notes/ folder, and patch file to recover context
- All context successfully recovered from files

### Architecture Decision: No Downgrades
- **Decision:** Never downgrade dependency requirements to match system packages
- **Rationale:** The project requires Qt 6.7.0+, libei >= 1.3, libportal >= 0.9.1
- Ubuntu 24.04 ships older versions (Qt 6.4.2, libei 1.2.1, libportal 0.7.1)
- **Correct approach:** Install required versions from upstream (aqtinstall for Qt, source build for libei/libportal)
- **User explicitly said:** "Do not downgrade anything" and "Please do not downgrade anything"

### Patch File Created
- Created `notes/patches/forward-touchscreen-events.patch` containing all changes from previous session
- Patch includes: config option infrastructure, GUI changes, server changes, unit tests
- Patch does NOT include: MSWindowsHook.cpp behavior change, EiScreen touch support (still pending)

### Bug Documentation
- Created `notes/BUG_9770_TOUCHSCREEN_FORWARDING.md` with full bug story
- Documents: problem, setup, observed behavior, root cause, investigation path, solution approach, test plan, status

### Strategy: Push First, Build Locally in Parallel
- User's plan: Push patch to GitHub first so CI starts running
- Meanwhile, set up local build environment
- By the time local build tools are installed, GitHub Actions may have feedback
- **Benefit:** Parallelizes CI feedback and local development

### Code Quality Issues Found and Fixed
- **ServerConfig.cpp operator<<** — Had duplicated lines and stray semicolon from previous session's patching. Cleaned up.
- **ServerConfig::numScreens()** — Was accidentally deleted. Restored.
- **Arch.h Qt include** — Changed from `<QtSystemDetection>` to `<QtGlobal>` (needed for compatibility)
