# CONTINUATION.md — Session Continuity

This file records what was being done and how to resume if interrupted.
**Always read from the BOTTOM** — the most recent entry is the one to follow.

---

## 2026-05-21 — Session Start

**What was being done:**
- Adapting notes/*.md files from PSync Health project to Deskflow project
- Understanding the touchscreen input forwarding bug (#9770)
- Created: `notes/IMPLEMENTATION_SAFETY_PROTOCOL.md`, `notes/PATHS.md`, `notes/DISCUSS.md`
- Still need to create: `notes/CONVERSATION_LOG.md`, `notes/LEARNINGS.md`

**What to do next:**
1. Create `notes/CONVERSATION_LOG.md` and `notes/LEARNINGS.md`
2. Begin deep investigation of the touchscreen bug
3. Trace the event flow from Windows MSWindowsScreen touch input through Server routing to client forwarding
4. Identify the exact code change needed

**Context:**
- Bug: When mouse is on Linux client, Windows server touchscreen keystrokes are forwarded to Linux instead of staying on Windows
- Windows = server, Linux (Wayland/libei) = client
- EiScreen.cpp explicitly does NOT bind EI_DEVICE_CAP_TOUCH (line 812)
- Need to understand how Server.cpp routes events when m_active != m_primaryClient
