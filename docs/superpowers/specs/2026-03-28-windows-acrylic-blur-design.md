# Windows 11 Acrylic Blur — Design Spec

**Date:** 2026-03-28
**Status:** Approved

---

## Background

Alacritty exposes a `window.blur` config option that works on macOS and Linux compositors. On Windows it is accepted without error but does nothing — winit's `with_blur()` is a no-op on the Windows backend. This spec describes a minimal implementation that activates Windows 11's Acrylic backdrop material when `blur = true`.

---

## Scope

- **In scope:** Windows 11 22H2+ Acrylic backdrop via `DWMWA_SYSTEMBACKDROP_TYPE`
- **Out of scope:** Windows 10 fallback (`SetWindowCompositionAttribute`), Mica, Mica Alt / Tabbed backdrop types, any config schema changes

---

## User-Visible Behavior

No new config fields are introduced. Users configure blur the same way as on other platforms:

```toml
[window]
blur = true
opacity = 0.85
```

- `blur = true` activates the Acrylic backdrop on Windows 11 22H2+. On older Windows versions the call fails silently and no blur is shown (existing behavior preserved).
- `opacity` controls how much of the Acrylic is visible — the GL surface clears with `alpha = opacity`, so `opacity = 1.0` renders a fully opaque background that hides the Acrylic. Users must set `opacity < 1.0` for the effect to be visible. This is consistent with macOS and Linux behavior.
- Runtime config reload (`set_blur`) is supported — toggling `blur` at runtime applies or removes the Acrylic backdrop.

---

## Architecture

The change is self-contained within `alacritty/src/display/window.rs` and `alacritty/Cargo.toml`. No new modules or files are created.

### DWM API sequence

Two Win32 DWM calls are required after window creation:

1. **`DwmExtendFrameIntoClientArea(hwnd, &MARGINS { -1, -1, -1, -1 })`** — extends the DWM backdrop frame to cover the entire client area. Without this, the Acrylic is clipped to the non-client (title bar) region.
2. **`DwmSetWindowAttribute(hwnd, DWMWA_SYSTEMBACKDROP_TYPE, &backdrop_type, 4)`** — sets the backdrop material. `DWMSBT_TRANSIENTWINDOW = 3` enables Acrylic; `DWMSBT_DISABLE = 0` removes it.

To disable Acrylic (when `blur` is toggled off), the same `DwmSetWindowAttribute` call is made with `DWMSBT_DISABLE`.

### Constants

`windows-sys 0.59` may not export `DWMWA_SYSTEMBACKDROP_TYPE` or `DWMSBT_*` values. They are defined inline:

```rust
const DWMWA_SYSTEMBACKDROP_TYPE: u32 = 38;
const DWMSBT_DISABLE: u32 = 0;
const DWMSBT_TRANSIENTWINDOW: u32 = 3;
```

### Transparency prerequisites

The existing window setup already satisfies all prerequisites for Acrylic to composite correctly:
- `with_transparent(true)` is set on all platforms unconditionally
- The GL config selects a pixel format with alpha support
- The renderer clears with `alpha = window_opacity()`

No changes to the rendering pipeline are required.

---

## Code Changes

### `alacritty/Cargo.toml`

Add `Win32_Graphics_Dwm` to the `windows-sys` features:

```toml
[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.59", features = [
    "Win32_UI_WindowsAndMessaging",
    "Win32_System_Threading",
    "Win32_System_Console",
    "Win32_Foundation",
    "Win32_Graphics_Dwm",   # <-- add
]}
```

### `alacritty/src/display/window.rs`

**1. New private free function** (Windows-only, outside `impl Window`, placed near the `#[cfg(windows)]` `get_platform_window` method):

```rust
#[cfg(windows)]
fn apply_dwm_acrylic(hwnd: isize, enable: bool) {
    use std::ffi::c_void;
    use windows_sys::Win32::Graphics::Dwm::{
        DwmExtendFrameIntoClientArea, DwmSetWindowAttribute, MARGINS,
    };

    const DWMWA_SYSTEMBACKDROP_TYPE: u32 = 38;
    const DWMSBT_DISABLE: u32 = 0;
    const DWMSBT_TRANSIENTWINDOW: u32 = 3;

    let backdrop = if enable { DWMSBT_TRANSIENTWINDOW } else { DWMSBT_DISABLE };

    unsafe {
        let margins = MARGINS { cxLeftWidth: -1, cxRightWidth: -1, cyTopHeight: -1, cyBottomHeight: -1 };
        let hr = DwmExtendFrameIntoClientArea(hwnd as _, &margins);
        if hr != 0 {
            log::debug!("DwmExtendFrameIntoClientArea failed: {hr:#x}");
            return;
        }

        let hr = DwmSetWindowAttribute(
            hwnd as _,
            DWMWA_SYSTEMBACKDROP_TYPE,
            &backdrop as *const u32 as *const c_void,
            std::mem::size_of::<u32>() as u32,
        );
        if hr != 0 {
            log::debug!("DwmSetWindowAttribute(DWMWA_SYSTEMBACKDROP_TYPE) failed: {hr:#x}");
        }
    }
}
```

**2. In `Window::new()`**, after `event_loop.create_window(window_attributes)?`:

```rust
#[cfg(windows)]
if config.window.blur {
    if let RawWindowHandle::Win32(handle) = window.window_handle().unwrap().as_raw() {
        apply_dwm_acrylic(handle.hwnd.get() as _, true);
    }
}
```

**3. In `Window::set_blur()`**, alongside the existing winit call:

```rust
pub fn set_blur(&self, blur: bool) {
    self.window.set_blur(blur);
    #[cfg(windows)]
    if let RawWindowHandle::Win32(handle) = self.window.window_handle().unwrap().as_raw() {
        apply_dwm_acrylic(handle.hwnd.get() as _, blur);
    }
}
```

---

## Error Handling

All DWM call failures are logged at `debug` level and do not surface to the user. This means:
- Windows 10: `DwmSetWindowAttribute` returns `E_INVALIDARG` for unknown attribute 38 → logged, ignored
- Windows 11 before 22H2 (builds 22000–22620): same result
- Windows 11 22H2+: `S_OK`, Acrylic is active

No panics, no user-visible errors, no behavioral regression on unsupported versions.

---

## Testing

Manual testing on Windows 11 22H2+ with:
```toml
[window]
blur = true
opacity = 0.85
```
Expected: Acrylic frosted-glass backdrop visible behind terminal content.

Runtime toggle: change `blur` in config file while Alacritty is running → backdrop appears/disappears on config reload.

There are no automated tests for Win32 visual effects; this is consistent with the rest of the Windows-specific code in the codebase.
