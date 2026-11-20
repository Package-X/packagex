# `.px` Package Manifest Specification

The `.px` format uses **TOML 1.0 syntax** extended with **engine version pinning**, **explicit `elevate` control**, **OS-namespaced steps (`<section>.<os>.<key>`)**, **cross-OS path tokens**, **pre-set templates**, and **AEAD password sealing**.

## 1. Cross-OS Built-In Path Tokens

All path strings support case-insensitive `$token$` interpolation across Windows, macOS, and Linux:

| Token | Windows Expansion | macOS Expansion | Linux Expansion |
| :--- | :--- | :--- | :--- |
| `$appdata$` | `%APPDATA%` | `~/Library/Application Support` | `$XDG_CONFIG_HOME` (`~/.config`) |
| `$localappdata$` | `%LOCALAPPDATA%` | `~/Library/Caches` | `$XDG_DATA_HOME` (`~/.local/share`) |
| `$appdata$/local/...` | `%LOCALAPPDATA%\...` | `~/Library/Application Support/...` | `~/.local/share/...` |
| `$programfiles$` | `%ProgramFiles%` | `/Applications` | `/opt` (or `/usr/local`) |
| `$temp$` | `%TEMP%` | `$TMPDIR` | `/tmp` |
| `$desktop$` | User Desktop | `~/Desktop` | `~/Desktop` |
| `$instdir$` | Resolved `install_dir` | Resolved `install_dir` | Resolved `install_dir` |

> **Trailing Wildcard Convention (`/*`):** Ending `install_dir` with `/*` marks the directory as an isolated package root managed exclusively by the app's uninstaller for clean removal.

## 2. Explicit Elevation (`[package].elevate`)

- `elevate = false`: Strictly per-user install—never prompts for UAC/sudo. `packx check` fails at build time if any step writes to `$programfiles$`, `HKLM`, `/usr/local`, `/opt`, or registers a `system-service`.
- `elevate = true`: Always elevates at install start via runtime self-elevation (`ShellExecuteW("runas")` / `sudo`), while keeping the Windows PE manifest `asInvoker` so `-packagex-detail` and `--dry-run` stay UAC-free.
- `elevate = "auto"`: Infers elevation from `scope` and resolved paths.

## 3. OS-Namespaced Steps (`<section>.<os>.<key>`) & Ordering

To prevent OS-specific scripts (`.ps1`, `.exe`, `powershell`, `registry`) from accidentally breaking cross-OS manifests:
- **`[[assembly.steps]]`** (no middle `.os.`): Strictly **cross-OS general** steps (`extract`, WASM `script`, generic `dependency`). Putting a `.ps1`, `.exe`, `powershell`, or Windows `registry` step here is an immediate `packx check` compile error.
- **`[[assembly.windows.steps]]`**, **`[[assembly.macos.steps]]`**, **`[[assembly.linux.steps]]`**: OS-scoped steps compiled only for that target OS.
- **Deterministic Step Ordering (`phase` + `before`/`after`):** Because general and OS-scoped steps can mix, every step accepts an optional `phase = "pre" | "install" | "post"` (default `"install"`) and optional `before = "<step_id>"` / `after = "<step_id>"` anchors so `px-spec` builds an unambiguous execution graph.

## 4. Pre-Set Templates (`[package].template`)

- **`desktop-app`:** System/user app directory, Start Menu / `.app` / `.desktop` shortcuts, white-label uninstaller.
- **`cli-path-app`:** Defaults `elevate = false` & `scope = "user"`, exports `bin_exports` to `$instdir$/bin`, registers user `$PATH`.
- **`hybrid-app`:** Combines GUI shortcuts (`app_exe`) with CLI `$PATH` registration (`bin_exports`).
- **`system-service`:** Configures Windows Service / macOS `launchd` / Linux `systemd` (requires `elevate = true`) with port/privilege checks in `--dry-run`.

## 5. Complete Schema Reference

```toml
[package]
name = "MySoftware"
version = "1.0.0"
engine = "2.4"                       # Pinned `packx toolbelt` engine version
publisher = "Kastrick"
app_id = "com.kastrick.mysoftware"   # Stable ID (UUIDv5 for MSI / bundle ID for macOS / uninstaller name)
template = "hybrid-app"              # "desktop-app" | "cli-path-app" | "hybrid-app" | "system-service"
scope = "user"                       # "user" | "machine"
elevate = false                      # true | false | "auto" (PE manifest always stays `asInvoker`)
targets = ["exe", "msi", "dmg", "pkg", "deb", "appimage"]
app_exe = "bin/mysoftware"
bin_exports = ["bin/mysoftware"]

[assembly]
install_dir = "$appdata$/local/kastrick_examples/software/*"
allow_user_dir_override = true
add_to_path = ["$instdir$/bin"]
compression = "zstd"                 # "zstd" (default) | "lzma" | "none"
compression_level = 19
reproducible = true                  # Enforces sorted files, frozen timestamps, relative paths, and sha256 dep pins
generate_sbom = "cyclonedx"          # "cyclonedx" | "spdx" | "none"

[security]
encryption = "none"                  # "none" | "password" (Argon2id + XChaCha20-Poly1305 single-blob sealing)
recovery = "none"                    # "none" | "key" (offline recovery code) | "pubkey" (Company X25519 pubkey)
recovery_pubkey = ""                 # Path to company X25519 public key when recovery = "pubkey"
recovery_contact = "Contact IT Helpdesk"

[ui]
mode = "schema"                      # "schema" (Native `.pxs`) | "html" (Tauri 2 WebView)
entry = "./wizard.pxs"
theme = "winlook.dark"               # "winlook.auto" | "winlook.dark" | "winlook.light" | path to .etr
winlook_backend = "win10"            # "win10" (Win32) | "win11" (WinUI 3) | "auto"
headless_fallback = "tui"            # "tui" (ratatui) | "cli"
window_size = [680, 460]

[updater]
enabled = true
feed_url = "https://releases.kastrick.dev/devtool/update-feed.json"
version_path = "$.release.version"   # JSON path to version field in any existing API
version_regex = "^v?(\\d+\\.\\d+(?:\\.\\d+)?)$"
priority_path = "$.release.priority" # "critical" | "normal" | "background"
download_url_path = "$.release.url"
sha256_path = "$.release.sha"        # Mandatory in production (`packx build`); warns in `packx test`
delta_patching = "zstd"              # Uncompressed-stream zstd binary delta patches
launcher = "auto"                    # "auto" (white-label shim) | "custom" (verified by `packx test` probe)
ui = "./updater.pxs"                 # Optional editable `.pxs` / HTML updater UI
check_interval_hours = 24            # Cross-OS per-user scheduler (Task Scheduler / LaunchAgent / systemd --user)

[signing]
windows = "local-ca"                 # "local-ca" | "signtool" | "osslsigncode" | "azure-trusted-signing"
macos_notarize = true
linux_gpg = true
```
