# `packx` CLI, `-packagex-detail`, `packx test` & `--dry-run` Specification

## 1. `packx` Developer CLI Commands

| Command | Purpose |
| :--- | :--- |
| `packx build [target] [--silent] [--offline] [--locked] [--sign] [--lock-timeout=<s>]` | Compiles `.px` manifest using the pinned `packx toolbelt` engine and target sub-belts. |
| `packx check [manifest.px]` | Unified pre-build linter & environment health check (separate from `packx test`): validates `.px` OS-namespaced steps, `elevate = false` guards, `reproducible = true` `sha256` pins, Target Capability Matrix, signing checks, UI backend constraints, plus installed belts, keyring signing keys/cert expiry, and `toolbelt.lock` health. |
| `packx test [manifest.px]` | **Dev-Only Desktop Sandbox:** Builds and runs a local test install into `Desktop/<package_name>/install/` + `uninstaller/`, capturing virtualized `$PATH`/Registry/service writes into `simulated-mutations.json` without touching the real host OS. |
| `packx config --set-signkey=<path-or-key>` | **Global Signing Key Store (All Toolbelt Versions):** Auto-detects key type (macOS `.p12`/Developer ID, Windows `.pfx`/Local CA, Linux GPG) and binds it globally to the matching OS target belt across every installed engine version. Supports `--set-signkey-stdin` and stores secrets in the OS native keyring (Windows Credential Manager / macOS Keychain / Linux Secret Service) rather than plaintext JSON. |
| `packx init --from <project>` | Auto-detects `Cargo.toml`, `package.json`, `pyproject.toml`, `go.mod`, or `.csproj` and scaffolds `.px` + `wizard.pxs`. |
| `packx init-ci <github|gitlab>` | Generates a ready-to-run multi-OS CI release workflow. |
| `packx preview <file.pxs>` | Opens a live hot-reload native preview window for `.pxs` installer/updater screens without rebuilding. |
| `packx sign init-ca --org "<Name>"` | Generates a free, offline company-private Root CA + Code Signing certificate via [`rcgen`](https://github.com/rustls/rcgen) for GPO/Intune-trusted internal installers ($0). |
| `packx verify <path> [--diff-show <other>]` | Cryptographically verifies signatures (Authenticode / Apple `codesign` / GPG), PE/ELF/Mach-O headers, and SHA-256 integrity on **any** binary, installer, or custom launcher—including third-party binaries not built by PackageX. Passing `--diff-show <other>` compares two builds to display added/removed/changed files and `zstd` delta savings. |
| `packx clean` | Cleans PackageX-owned dev artifacts on the host: stale `Desktop/<package_name>/` test sandboxes, old built installers, compiled PackageX output binaries/caches, and orphaned `.tmp.<pid>` belt folders. **Never touches** language caches (Rust `target/`, Python, Node.js, Go, etc.), active toolbelts, or keyring signing keys. |
| `packx toolbelt list` | Displays installed toolbelt engine versions (`2.3`, `2.4`) and their attached target sub-belts. |
| `packx toolbelt install --required` | Installs a toolbelt engine and target sub-belt using blocking `toolbelt.lock` + atomic CAS delta downloads. |
| `packx toolbelt remove <package-id>` | Removes a toolbelt version and deletes `owner`-only SDK modules while preserving `shared` modules. |
| `packx toolbelt prune` | Garbage-collects unreferenced SDK modules and leftover `.tmp.<pid>` folders. |
| `packx toolbelt export --required -o <file>.pxbelt` | Exports a portable offline bundle for air-gapped CI/IT servers. |
| `packx toolbelt import <file>.pxbelt` | Imports an offline `.pxbelt` bundle into the local CAS pool. |

### Canonical Toolbelt Install and Nightly Selection

The canonical package manager namespace is packx toolbelt:

~~~powershell
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
~~~

Stable is the default. Nightly is opt-in; its visible identity is nightly.<8hex_commit>, while full commit and internal semantic-version metadata are retained in the package manifest/Toolchain lock. There is no Beta channel. A missing or ambiguous Nightly prefix is a hard error and never falls back to another Nightly or Stable. Exact Toolchain pins are not rewritten by routine package updates.

Required means only the minimum complete compatible dependency closure for the selected Engine, project targets and requested formats. See the [PackX required-resolution playbook](https://github.com/Package-X/packx/blob/main/plan/architecture/required-resolution.md).

Legacy packx belt spelling, if retained, is a compatibility alias to this same manager/state store.

### Offline Mode Error Contract (`--offline` / `--locked`)
Exits with canonical code `13`:
```text
Error: Requires packx 2.4 (Belt_x86_64-windows), found 2.3.
Reason: Offline mode enabled — cannot fetch from network.
Fix: Run `packx belt add 2.4/x86_64-windows` online, or import a .pxbelt bundle.
```

## 2. Supported Runtime Flags on Assembled Installers

On Windows, `px-stub` is compiled with the GUI subsystem + `asInvoker` manifest and calls Win32 `AttachConsole(ATTACH_PARENT_PROCESS)` whenever any CLI flag below is passed, ensuring clean `stdout` output in PowerShell and `cmd.exe` with zero UAC prompt.

| Flag | Behavior |
| :--- | :--- |
| `-packagex-detail` | Prints the formatted execution plan, resolved paths, remote URLs, `.s.px` WASM capabilities, and compile-time estimated uncompressed size. **If password-sealed**, never prompts—dumps only the sealed manifest envelope encoded as **base64 noise** (terminal-safe). Zero UAC, zero network, zero disk writes. |
| `-packagex-detail --json` | Machine-readable JSON output for CI/CD auditing (or sealed base64 envelope JSON when encrypted). |
| `-packagex-detail --files` | Includes the full compile-time per-file SHA-256 inventory (unsealed builds only). |
| `--dry-run` | Probes the **target machine** without writing a byte or triggering UAC: checks disk space, locked files/ports, elevation requirements, and `check_cmd` probes. |
| `--silent` / `/S` | Runs headlessly with standardized exit codes ([`exit-codes-and-errors.md`](exit-codes-and-errors.md)) and optional `--log-json=<path>`. |
| `--password-stdin` / `--recovery-key-stdin` | Reads the unlock password or recovery key from `stdin` for unattended MDM/CI installs of sealed packages (never passed as plain CLI argv). |
| `--dir=<path>` | Overrides `[assembly].install_dir` (when `allow_user_dir_override = true`). |

## 3. Reference `-packagex-detail` Output (Unsealed Build)

```text
========================================================================
 PackageX Installer Inspection (-packagex-detail)
========================================================================
Package Name    : KastrickDevTool
Version         : 1.0.0 (Built with packx toolbelt 2.4 | Reproducible: YES)
Publisher       : Kate-alt-69
Template        : hybrid-app (Scope: user | Elevate: false — No UAC/sudo)
Target Install  : C:\Users\<User>\AppData\Local\kastrick_examples\software\
PATH Additions  : + C:\Users\<User>\AppData\Local\kastrick_examples\software\bin
Payload Size    : Compressed: 6.1 MB | Estimated Uncompressed: 18.4 MB (zstd-19)

[UI Configuration]
  Mode          : schema (PXS — PackageX Schema)
  Entry         : ui/wizard.pxs
  Theme         : winlook.dark
  Native Backend: win10 (WinLook Win32/User32) | Headless Fallback: ratatui TUI
  Window Size   : 680x460 DIPs

[Ordered Assembly Steps (Target: windows)]
  Step 1 | phase: pre     | id: "check-python"      | type: dependency | weight: 30%
  Step 2 | phase: pre     | id: "check-node"        | type: dependency | weight: 30%
  Step 3 | phase: install | id: "extract-app"       | type: extract    | weight: 25%
  Step 4 | phase: post    | id: "configure-runtime" | type: script (WASM / wasmi) | weight: 15%
    - Hook Caps : [fs.ensure_dir, env.add_path, log.info]
========================================================================
```
