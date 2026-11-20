# Auto-Updater, White-Label Launcher Shim & 24h Schedulers

PackageX ships an optional, 100% white-labeled auto-update system (`%LOCALAPPDATA%\<package_name>\updater\update.exe` on Windows, or native per-user app support paths on macOS/Linux) that adapts to **existing company or open-source JSON APIs** without forcing a custom update server.

## 1. Flexible JSON-Path + Regex + SemVer Checking

Instead of locking developers into a single rigid manifest schema, `[updater]` points at any existing JSON endpoint (GitHub Releases, S3 manifest, or internal REST API) and maps fields declaratively:

1. **JSON Path Extraction:** Extracts `version_path`, `priority_path`, `download_url_path`, and `sha256_path` cleanly (immune to JSON formatting/whitespace changes).
2. **Optional Regex Capture:** Strips version prefixes/suffixes (e.g., `^v?(\d+\.\d+(?:\.\d+)?)$`) before version comparison.
3. **Real SemVer Ordering:** Compares versions using [`semver`](https://github.com/dtolnay/semver) so `1.10.0 > 1.9.0` always evaluates accurately.
4. **Priority Policies:**
	- **`critical`:** Blocks app launch, applies the verified update immediately, then launches the new version.
	- **`normal`:** Displays a non-blocking update prompt when the app launches (skippable by the user).
	- **`background`:** Downloads and verifies silently in the background; swaps cleanly on next app exit.

## 2. Security Enforcement: Dev Warning vs. Production Hard-Stop

- **Production (`packx build`):** **Hard-stops** if the updater URL is not HTTPS, lacks a `sha256_path` (`"sha": "<SHA256>"`) mapping, or lacks a pinned **Ed25519 public key** / Local CA certificate thumbprint embedded into `update.exe` at build time.
- **Dev / Sandbox (`packx test`):** Prints a clear warning (`"Updater URL is not secured — sign updates and include a sha field for production"`) and continues cleanly for local iteration.

## 3. White-Label Launcher Shim & Custom Launcher Sandbox Probe

PackageX **never patches or modifies the user's compiled application binary** (which would break Authenticode/`codesign` signatures and trigger Windows Defender / Gatekeeper heuristics).

### Default Mode (`launcher = "auto"`)
- Places the user's untouched, signed app binary inside `$instdir$/app/<app_exe>`.
- Generates a tiny white-labeled launcher (`<package_name>.exe` / wrapper) that clones the app's **Name and Icon** and sits at the shortcut / `$PATH` entry point.
- On launch, the launcher checks for staged or `critical` updates via `update.exe` *before* starting the real binary—eliminating Windows' `ERROR_SHARING_VIOLATION` locked-file trap cleanly with zero DLLs and zero `.lib` dependencies.

### Custom Launcher Mode (`launcher = "custom"`)
- Developers can disable the default shim and supply their own custom launcher (`launcher_bin = "./dist/my-launcher.exe"`).
- **Runtime Sandbox Probe (`packx test`):** Because static binary inspection cannot reliably prove dynamic launch paths across Rust/C++/Go/C#, `packx test` places tiny **mock spy binaries** in place of `update.exe` and the target app inside `Desktop/<package_name>/`, launches the custom launcher briefly, and verifies both spies were invoked—failing with an actionable error if the custom launcher isn't wired to the updater or app.

## 4. Editable Updater UI (`.pxs` / HTML)

Developers can customize or theme the updater UI using the exact same `.pxs` schema compiler (`[updater].ui = "./updater.pxs"`), supporting `UpdateAvailablePage`, `DownloadingUpdatePage`, and `CriticalUpdatePage` across WinLook `wl_ui`, AppKit, GTK/Qt, Tauri 2, and `ratatui` TUI.

## 5. Cross-OS 24-Hour Per-User Background Schedulers (Zero UAC / Zero `sudo`)

To check for updates periodically (e.g., every 24h) without requiring admin privileges or polluting login-only Registry keys, `px-runtime` registers each OS's native per-user scheduler under `<app_id>` (tracked by the rollback journal and removed cleanly on uninstall):
- **Windows:** Per-user Task Scheduler job (`OnSchedule`, interval from `[updater].check_interval_hours`, default `24`).
- **macOS:** Per-user `launchd` LaunchAgent (`~/Library/LaunchAgents/<app_id>.updater.plist`, `StartInterval = 86400`).
- **Linux:** Per-user `systemd --user` timer (`~/.config/systemd/user/<app_id>-updater.timer`, `OnCalendar=daily`).
