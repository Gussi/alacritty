# Windows 11 Acrylic Blur Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `window.blur = true` activate Windows 11's Acrylic backdrop material in Alacritty.

**Architecture:** After winit creates the window, call winit's built-in `WindowExtWindows::set_system_backdrop(BackdropType::TransientWindow)` to activate Acrylic. The same call is added to `set_blur()` for runtime config-reload support. No direct Win32 calls needed — winit 0.30.13 already wraps `DwmSetWindowAttribute(DWMWA_SYSTEMBACKDROP_TYPE)`. Acrylic becomes visible only when `window.opacity < 1.0`, consistent with macOS/Linux behavior.

**Tech Stack:** Rust, winit 0.30.13 (`winit::platform::windows::{BackdropType, WindowExtWindows}`), existing `alacritty/src/display/window.rs`

> **Note:** This implementation is simpler than the approved spec. Winit 0.30.13 exposes `BackdropType` and `set_system_backdrop()` directly, so direct `windows-sys` DWM calls and `Cargo.toml` changes are not needed.

---

## File Map

| File | Change |
|------|--------|
| `alacritty/src/display/window.rs` | Add `BackdropType` + `WindowExtWindows` to Windows import; call `set_system_backdrop` in `Window::new()` and `set_blur()` |

No other files change.

---

### Task 1: Add `set_system_backdrop` call in `Window::new()`

**Files:**
- Modify: `alacritty/src/display/window.rs:35` (imports), `alacritty/src/display/window.rs:186-197` (after window creation)

- [ ] **Step 1: Update the Windows-only import on line 35**

The current line reads:
```rust
use winit::platform::windows::{IconExtWindows, WindowAttributesExtWindows};
```

Replace with:
```rust
use winit::platform::windows::{BackdropType, IconExtWindows, WindowAttributesExtWindows, WindowExtWindows};
```

- [ ] **Step 2: Add Acrylic call after `create_window` in `Window::new()`**

The current code at lines 186–197 reads:
```rust
        let window = event_loop.create_window(window_attributes)?;

        // Text cursor.
        let current_mouse_cursor = CursorIcon::Text;
        window.set_cursor(current_mouse_cursor);

        // Enable IME.
        window.set_ime_allowed(true);
        window.set_ime_purpose(ImePurpose::Terminal);

        // Set initial transparency hint.
        window.set_transparent(config.window_opacity() < 1.);
```

Insert a `#[cfg(windows)]` block immediately after `event_loop.create_window(window_attributes)?;`:
```rust
        let window = event_loop.create_window(window_attributes)?;

        // Apply Acrylic backdrop on Windows 11 22H2+.
        #[cfg(windows)]
        if config.window.blur {
            window.set_system_backdrop(BackdropType::TransientWindow);
        }

        // Text cursor.
        let current_mouse_cursor = CursorIcon::Text;
        window.set_cursor(current_mouse_cursor);

        // Enable IME.
        window.set_ime_allowed(true);
        window.set_ime_purpose(ImePurpose::Terminal);

        // Set initial transparency hint.
        window.set_transparent(config.window_opacity() < 1.);
```

- [ ] **Step 3: Verify the file compiles (cross-check target)**

Install the Windows check target if not present:
```bash
rustup target add x86_64-pc-windows-gnu
```

Run a type-check for Windows target (requires `mingw-w64`; if not available, the step below is sufficient):
```bash
# If mingw-w64 is installed:
cargo check --target x86_64-pc-windows-gnu -p alacritty 2>&1 | tail -10

# If not (Linux/CI without cross-compiler), at minimum verify no compilation
# errors on the native target by checking the non-cfg'd code compiles:
cargo check -p alacritty --no-default-features 2>&1 | grep "^error" | head -20
```

Expected for native check: no `error[...]` lines (fontconfig warnings are OK — they come from the build script, not our code).

- [ ] **Step 4: Commit**

```bash
git add alacritty/src/display/window.rs
git commit -m "feat: enable Acrylic backdrop on Windows 11 when blur is set"
```

---

### Task 2: Add `set_system_backdrop` call to `set_blur()` (runtime toggle)

**Files:**
- Modify: `alacritty/src/display/window.rs:375-377` (`set_blur` method)

The current `set_blur` method reads (lines 375–377):
```rust
    pub fn set_blur(&self, blur: bool) {
        self.window.set_blur(blur);
    }
```

Winit's `set_blur` is a no-op on Windows (`pub fn set_blur(&self, _blur: bool) {}`). We keep the call for other platforms and add the Windows Acrylic toggle alongside it.

- [ ] **Step 1: Update `set_blur` to also set the system backdrop on Windows**

```rust
    pub fn set_blur(&self, blur: bool) {
        self.window.set_blur(blur);
        #[cfg(windows)]
        self.window.set_system_backdrop(if blur {
            BackdropType::TransientWindow
        } else {
            BackdropType::Auto
        });
    }
```

`BackdropType::Auto` (= `DWMSBT_AUTO`) restores the system default — it removes the Acrylic when the user sets `blur = false` at runtime.

- [ ] **Step 2: Verify imports compile for Windows target**

```bash
# Cross-check if mingw available:
cargo check --target x86_64-pc-windows-gnu -p alacritty 2>&1 | tail -10

# Native fallback:
cargo check -p alacritty --no-default-features 2>&1 | grep "^error" | head -20
```

Expected: no `error[...]` lines.

- [ ] **Step 3: Commit**

```bash
git add alacritty/src/display/window.rs
git commit -m "feat: support runtime blur toggle for Acrylic on Windows 11"
```

---

## Manual Testing (Windows 11 22H2+)

Set the following in `~/.config/alacritty/alacritty.toml`:
```toml
[window]
blur = true
opacity = 0.85
```

Expected: Alacritty window shows a frosted-glass Acrylic effect behind terminal content.

**Runtime toggle test:** While Alacritty is running, change `blur = false` in the config file and save. The backdrop should disappear (window becomes solid). Change back to `blur = true` — backdrop reappears.

**Windows 10 / pre-22H2 test:** Run on Windows 10 or Windows 11 < build 22621. Expected: no Acrylic visible, no crash, no error shown to user (the DWM call returns `E_INVALIDARG` silently).

---

## Building a Windows Executable

### On a Windows machine (recommended)

```powershell
# Install Rust: https://rustup.rs
# Install Visual Studio Build Tools (MSVC)
git clone https://github.com/alacritty/alacritty
cd alacritty
git checkout <your-branch>
cargo build --release
# Output: target/release/alacritty.exe
```

### Cross-compiling from Linux (requires mingw-w64)

```bash
sudo apt-get install mingw-w64
rustup target add x86_64-pc-windows-gnu

# OpenSSL and other system deps need Windows versions — simplest path:
cargo build --release --target x86_64-pc-windows-gnu
# Output: target/x86_64-pc-windows-gnu/release/alacritty.exe
```

> **Note:** Cross-compilation of Alacritty from Linux to Windows is non-trivial because of native dependencies (fontconfig/freetype are not used on Windows, but the build scripts may still run). The recommended path is building natively on Windows. The CI pipeline (`release.yml`) builds the official Windows binaries using GitHub Actions with a Windows runner.
