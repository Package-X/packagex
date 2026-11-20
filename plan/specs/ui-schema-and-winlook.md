# UI Pipeline Specification: `.pxs` (PackageX Schema) & Cross-OS Native Backends

## 1. Migration from `.nes` to `.pxs` (PackageX Schema)

PackageX replaces `.nes.json` with **`.pxs` (PackageX Schema)**:
- Clean discovery, linting, and parsing without coupling to `nextjs-engine` internals.
- Zero OS file-association conflicts with Nintendo Entertainment System (`.nes`) ROM files.

## 2. Cross-OS Native Backends + Automatic Headless Fallback

One `.pxs` schema compiles into native OS UI widgets everywhere—never shipping a bloated Electron wrapper unless HTML mode is explicitly chosen:

| OS / Environment | Backend | Underlying Tech | PackageX Role |
| :--- | :--- | :--- | :--- |
| **Windows 10/11** | **WinLook `wl_ui` (`win10`)** | Native Rust Win32 + GDI semantic palette | **Default Windows Backend.** Zero external dependencies (no WebView2 / App SDK); instant launch. |
| **Windows 11** | **WinLook `wl_ui` (`win11`)** | Native Rust + WinUI 3 / Fluent controls | Modern Mica/Acrylic Fluent mode (requires Windows App Runtime). |
| **macOS** | **Native AppKit** | Rust -> [`objc2`](https://github.com/madsmtm/objc2) AppKit bindings | Native macOS sheet/wizard feel with Dark Mode support. |
| **Linux (Desktop)** | **Native GTK / Qt** | Rust -> GTK4 / Qt native bindings | Matches user desktop theme cleanly. |
| **HTML Mode** | **`wl_ui` Tauri 2 Adapter** | OS System WebView + `window.packageX` IPC | Activated when `[ui].mode = "html"` for custom web-styled wizards. |
| **SSH / Docker / CI** | **Auto TUI / Headless CLI** | [`ratatui`](https://github.com/ratatui/awesome-ratatui/blob/main/README.md) TUI / stdout stream | Automatically activated when no display server (`DISPLAY`/`WAYLAND_DISPLAY`/desktop session) is detected. |

### Backend Constraint Validation & CLI UX Hints (`packx check`)
- `px-ui-compiler` validates `.pxs` nodes against the target backend (e.g., warning on CSS-only blur filters when `winlook_backend = "win10"`).
- For `cli-path-app` and `hybrid-app` packages, `FinishPage` surfaces a clean one-line reminder that newly registered `$PATH` binaries take effect in newly opened terminal windows.

## 3. Dual-Mode UI Compilation (`px-ui-compiler`)

### Mode A: Native `.pxs` Schema Mode (`mode = "schema"`)
1. **Compile-Time Validation (`packx build` / `packx check`):**
	- Parses `wizard.pxs` and resolves theme tokens (`winlook.dark`, `winlook.light`, or custom `.etr`).
	- Verifies every `ui_state` referenced in `[[assembly.steps]]` (and `password_prompt` when `[security].encryption = "password"`) has a matching screen binding.
	- Emits a compact binary widget tree into the installer overlay.
2. **Runtime Rendering (`px-stub` -> `px-ui-bridge`):**
	- Binds live `px-runtime` signals (`progress_percent`, `current_step_label`, `log_lines`, `install_dir`, `security.recovery_contact`) directly to native widgets.

### Mode B: HTML Mode (`mode = "html"`)
- Bundles `index.html` + local CSS/JS/SVG assets with `wl_ui` CSS theme variables and exposes `window.packageX` (`startInstall`, `submitPassword`, `submitRecoveryKey`, `cancelInstall`, `selectDirectory`, `onProgress`).

## 4. Standard UI State Machine (Including Password Sealing)

```text
  [password_prompt] (Only when [security].encryption = "password"; supports Recovery Key toggle)
         │ (Valid AEAD unlock)
         ▼
    [welcome] ──(User clicks Install)──▶ [checking_env]
                                                │
                           ┌────────────────────┴────────────────────┐
                           ▼ (Dependency missing)                    ▼ (All deps present)
                [installing_dependency] ────────────────────▶ [extracting_files]
                           │                                         │
                           │ (Download/Script Error)                 ▼
                           │                                  [updating_path]
                           │                                         │
                           ▼                                         ▼
                    [rolling_back] ──▶ [error_rolled_back]       [finished]
```
