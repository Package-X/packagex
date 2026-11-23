# `.px` Package Manifest Specification

The `.px` format uses **TOML 1.0 syntax** extended with **engine version pinning**, **installation scope**, **Electron optimization policy**, and **explicit `elevate` control**, **OS-namespaced steps (`<section>.<os>.<key>`)**, **cross-OS path tokens**, **pre-set templates**, and **AEAD password sealing**.

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

## 2. Installation Scope ([package].scope)

- scope = `user`: install for the invoking account; this is the default.
- scope = `machine`: install in a machine-managed location for all accounts where the selected target format supports it.
- scope = `choice`: expose only the scopes declared by the selected target's SDK capability; `scope_default` preselects the default.
- `scope_default` is valid only with `scope = "choice"`; otherwise it is ignored/rejected by schema validation.
- Scope determines installation ownership and registration. It is separate from `elevate`, which controls permission requirements.
- A denied machine-wide authorization never silently falls back to per-user. Changing scope during an update requires an explicit migration.
- Native target differences and the initial format capability matrix are defined in [Installation Scope](../architecture/installation-scope.md).

## 3. Explicit Elevation ([package].elevate)

- `elevate = false`: forbid privileged operations. A per-user installation is the usual case, but scope remains a separate field. `packx check` rejects any selected operation that requires machine privilege.
- `elevate = true`: permit/request the privilege required by the resolved operation plan. It does not by itself convert a per-user install into machine-wide scope.
- `elevate = "auto"`: request elevation only when a selected scope/operation requires it. Resolve scope first, then calculate privilege.
- The Windows PE manifest remains `asInvoker` for inspection and dry-run. Elevation occurs only after inspection flags are ruled out and the required operation is known.
- See [Installation Scope](../architecture/installation-scope.md) for the scope/elevation compatibility matrix.

## 4. OS-Namespaced Steps (`<section>.<os>.<key>`) & Ordering

To prevent OS-specific scripts (`.ps1`, `.exe`, `powershell`, `registry`) from accidentally breaking cross-OS manifests:
- **`[[assembly.steps]]`** (no middle `.os.`): Strictly **cross-OS general** steps (`extract`, WASM `script`, generic `dependency`). Putting a `.ps1`, `.exe`, `powershell`, or Windows `registry` step here is an immediate `packx check` compile error.
- **`[[assembly.windows.steps]]`**, **`[[assembly.macos.steps]]`**, **`[[assembly.linux.steps]]`**: OS-scoped steps compiled only for that target OS.
- **Deterministic Step Ordering (`phase` + `before`/`after`):** Because general and OS-scoped steps can mix, every step accepts an optional `phase = "pre" | "install" | "post"` (default `"install"`) and optional `before = "<step_id>"` / `after = "<step_id>"` anchors so `px-spec` builds an unambiguous execution graph.

## 5. Pre-Set Templates (`[package].template`)

- **`desktop-app`:** System/user app directory, Start Menu / `.app` / `.desktop` shortcuts, white-label uninstaller.
- **`cli-path-app`:** Defaults `elevate = false` & `scope = "user"`, exports `bin_exports` to `$instdir$/bin`, registers user `$PATH`.
- **`hybrid-app`:** Combines GUI shortcuts (`app_exe`) with CLI `$PATH` registration (`bin_exports`).
- **`system-service`:** Configures Windows Service / macOS `launchd` / Linux `systemd` (requires `elevate = true`) with port/privilege checks in `--dry-run`.

## 6. Electron Detection and Payload Optimization

The SDK reports Electron detection to the Engine. The Engine owns the interactive prompt and resolves this project policy before the target SDK build begins.

~~~toml
[electron]
optimization = "prompt"            # "prompt" | "enabled" | "disabled"
~~~

- Omitted or `prompt`: show the recommended yes/no prompt when the build is interactive. The affirmative choice enables the SDK's default Electron optimization profile.
- In a non-interactive build, `prompt` must not wait for terminal input. It fails with an actionable diagnostic requiring an explicit `enabled` or `disabled` setting/CLI override.
- `enabled`: request app-only Electron payload optimization. The selected target format must advertise protected-reference and restricted-helper support. If it cannot safely reconstruct the runtime, fail the target build rather than silently changing the request.
- `disabled`: preserve the complete original Electron distribution, including its runtime.
- Uncertain Electron detection never removes runtime files, regardless of the selected default profile.
- Electron optimization preserves the original application executable and relative layout. It does not add a PackageX launcher.
- The SDK embeds the exact runtime reference as an opaque authenticated control block alongside the payload, not as a loose readable JSON sidecar. The embedded helper restores the runtime only to authenticated logical slots declared by the SDK writer.
- The exact version, target, URL, expected size, SHA-256, runtime inventory and ABI identity are pinned at build time. No floating latest URL is resolved by the installer.

The format-specific support table and helper security requirements live in the SDK [Electron architecture](https://github.com/Package-X/sdk/blob/main/plan/architecture/electron.md).

## 7. Complete Schema Reference

```toml
[package]
name = "MySoftware"
version = "1.0.0"
engine = "2.4"                       # Pinned `packx toolbelt` engine version
publisher = "Kastrick"
app_id = "com.kastrick.mysoftware"   # Stable ID (UUIDv5 for MSI / bundle ID for macOS / uninstaller name)
template = "hybrid-app"              # "desktop-app" | "cli-path-app" | "hybrid-app" | "system-service"
scope = "user"                       # "user" | "machine" | "choice"
scope_default = "user"               # Required only for scope = "choice"
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

[electron]
optimization = "prompt"            # "prompt" | "enabled" | "disabled"

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
launcher = "auto"                    # Generic launcher policy; Electron optimization preserves the original Electron executable
ui = "./updater.pxs"                 # Optional editable `.pxs` / HTML updater UI
check_interval_hours = 24            # Scheduler follows package.scope: per-user or protected machine-wide

[signing]
windows = "local-ca"                 # "local-ca" | "signtool" | "osslsigncode" | "azure-trusted-signing"
macos_notarize = true
linux_gpg = true
```
