# `<name>.s.px` Scripting & Cross-OS Hook SDK Specification

## 1. Design Decision: Why WASM-First `<name>.s.px` (Replacing `.pyc` and Native Cross-Compilers)

Traditional installers force developers to maintain separate `.bat`/PowerShell scripts for Windows and `.sh` scripts for macOS/Linux.

- **Rejected Approach 1 (`.pyc` -> Bundled CPython):** Requires embedding a 15–30 MB CPython runtime, triggers heavy antivirus false positives, and breaks cross-OS portability.
- **Rejected Approach 2 (Native-First `script.exe` / ELF / Mach-O per OS):** Compiling a custom Python-like language to native machine code across 6 OS/CPU targets requires maintaining a full cross-compiler and per-OS linker inside every sub-belt.
- **Selected Approach (WASM-First `<name>.s.px` -> `wasmi`):**
	- **File Extension:** `<file_name>.s.px` (e.g., `setup.s.px`, `migrate.s.px`).
	- **Syntax:** Clean Python-like syntax (tabs for indentation, zero boilerplate).
	- **Compilation (`compile_mode = "wasm"` default):** `px-script` compiles `<name>.s.px` into compact, deterministic **WebAssembly bytecode** embedded directly inside the installer overlay and executed inside `px-stub` via [`wasmi`](https://github.com/wasmi-labs/wasmi) (a lightweight, security-audited Rust WebAssembly interpreter). One compiled `.wasm` block runs identically on Windows, macOS, and Linux with zero cross-linkers and strict memory sandboxing.

## 2. Sandboxed Cross-OS Hook SDK (`packx.*`)

Every `<name>.s.px` script executes against a deterministic, rollback-aware Hook SDK exposed by `px-runtime` to `wasmi`. All mutating SDK calls automatically register entries in the **Atomic Rollback Journal** (or write to `simulated-mutations.json` during `packx test` Desktop sandbox runs).

| SDK Primitive | Cross-OS Behavior |
| :--- | :--- |
| `packx.fs.ensure_dir(path)` | Creates directory with normalized tokens (`$instdir$`, `$appdata$`); tracked for rollback. |
| `packx.fs.copy(src, dst)` | Copies files safely with backup snapshots if overwriting. |
| `packx.env.add_path(dir)` | Updates Windows `HKCU`/`HKLM` + `WM_SETTINGCHANGE`, or macOS/Linux user shell profiles wrapped in `# >>> <app_id> >>>` sentinel blocks (uses `/etc/paths.d` only when `elevate = true`). |
| `packx.env.set(key, value)` | Sets persistent environment variable with rollback snapshot. |
| `packx.service.register(name, exe, args)` | Registers Windows Service, macOS `launchd` plist, or Linux `systemd` unit (requires `elevate = true` for system-wide daemons). |
| `packx.os.name()` / `packx.os.arch()` | Returns `"windows" | "macos" | "linux"` and `"x86_64" | "arm64"`. |
| `packx.log.info(msg)` / `packx.log.warn(msg)` | Streams structured log lines to UI / TUI / `--log-json`. |
| `packx.ui.set_progress(pct, label)` | Updates active installer progress bar and status text. |

## 3. Lifecycle Hooks in `.px`

```toml
[[assembly.steps]]
id = "configure-runtime"
phase = "post"
label = "Configuring cross-OS runtime & PATH"
type = "script"
entry = "./setup.s.px"
compile_mode = "wasm"     # Default: portable sandboxed WASM executed via `wasmi`
weight = 15
ui_state = "updating_path"
```

## 4. Security & Static Audit Guarantee

- **Static Capability Summary in `-packagex-detail`:** `px-script` extracts every `packx.*` host import called by `<name>.s.px` at compile time and embeds the list (e.g., `[fs.ensure_dir, env.add_path, log.info]`) into the manifest for IT auditing (hidden inside the AEAD blob if password-sealed).
- **No Unaudited Dynamic Eval:** `eval()`, `exec()`, and arbitrary raw syscalls are disallowed inside `.s.px`—scripts can *only* touch the host OS through the rollback-tracked `packx.*` host imports.
