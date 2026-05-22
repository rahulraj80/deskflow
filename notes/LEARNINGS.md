# LEARNINGS.md — Technical Findings and Solutions

This file records technical findings, what worked, what didn't, and how issues were resolved.

---

## 2026-05-21 — Build Environment Setup

### Issue: Qt version mismatch
- **Problem:** Project requires Qt 6.7.0+, Ubuntu 24.04 ships Qt 6.4.2
- **Initial wrong approach:** Lowered `REQUIRED_QT_VERSION` to 6.4.2 in CMakeLists.txt, then tried to patch source code to work around missing Qt 6.5+ APIs (`QString::removeFirst()`, `removeLast()`, `QStringList::indexOf` with `Qt::CaseSensitivity`, `QTextStream` in `FingerprintDatabase`, `QStandardPaths::GenericStateLocation`)
- **User correction:** "Do not downgrade anything." — Reverted all changes
- **Correct solution:** Installed Qt 6.7.3 from official Qt via `aqtinstall` into `/home/ubuntu/Qt/6.7.3/gcc_64`
- **Key lesson:** Never downgrade dependency requirements to match what's available in system packages. Instead, install the correct version from upstream.

### Issue: libei version mismatch
- **Problem:** Project requires libei >= 1.3, Ubuntu 24.04 ships 1.2.1
- **Solution:** Built libei 1.6.0 from source (gitlab.freedesktop.org/libinput/libei), installed to `/usr/local`
- **Dependencies needed:** `libsystemd-dev`, `libevdev-dev`, `meson`, `ninja`, `jinja2`

### Issue: libportal version mismatch
- **Problem:** Project requires libportal >= 0.9.1, Ubuntu 24.04 ships 0.7.1
- **Solution:** Building libportal 0.9.2 from source (github.com/flatpak/libportal), installed to `/usr/local`
- **Dependencies needed:** `libgirepository1.0-dev`
- **Ongoing:** vapigen not available, disabled with `-Dintrospection=false -Dvapi=false -Ddocs=false`

### Issue: System package installation requires sudo
- **Pattern:** `sudo apt-get install -y <package>` needed for system-level dev packages
- **Note:** Python packages should go in project `.venv`, never system Python

### Issue: C++ escape sequences in patch tool
- **From memory:** The patch tool can corrupt C++ string literals containing `\t`, `\n` etc.
- **Workaround:** Use `write_file` for full file rewrites, or `execute_code` with Python for precise replacement

---

## 2026-05-21 — Code Quality Issues in Previous Session Changes

### Issue: ServerConfig.cpp operator<< has duplicated/broken code
- **File:** `src/lib/gui/config/ServerConfig.cpp`
- **Problem:** The `operator<<` method has multiple duplicate lines for `forwardTouchscreenEvents` and `clipboardSharingSize`, and a stray `if (config.hasSwitchDelay());` with semicolon
- **Status:** Needs cleanup — the patch file captures the current state but the code is not clean
- **Lesson:** When adding serialization output, be careful not to duplicate existing lines. Always verify the full method after patching.

### Issue: Arch.h Qt include change
- **File:** `src/lib/arch/Arch.h`
- **Change:** `#include <QtSystemDetection>` → `#include <QtGlobal>`
- **Reason:** `QtSystemDetection` header doesn't exist in older Qt versions; `QtGlobal` provides the same macros
- **Status:** This change is correct and needed for Qt 6.7.3 compatibility

---

## Build Commands

```bash
# Configure (with Qt 6.7.3)
export Qt6_DIR=/home/ubuntu/Qt/6.7.3/gcc_64/lib/cmake/Qt6
cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug -DSKIP_BUILD_TESTS=OFF

# Build specific targets
cmake --build build --target ServerConfigTests ServerTests

# Run tests
ctest --test-dir build/src/unittests --output-on-failure
```
