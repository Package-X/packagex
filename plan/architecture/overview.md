# Architecture Overview

## 1. Core Architectural Principles

### A. Thin `packx` Shim + Versioned Toolbelt
Instead of a single monolithic binary that drifts across developer machines and CI servers, `packx` is split into two layers:
1. **`packx` Shim Router:** A tiny, permanent CLI binary installed on `$PATH` with minimal dependencies for instant startup. When invoked (e.g., `packx build app.px`), it reads `[package].engine` from the `.px` manifest and delegates execution to the exact versioned engine inside `~/.packx/toolbelt/<version>/packx-core`.
2. **Modular Sub-Belts & CAS SDK Pool:** Heavy OS-native SDKs, pre-compiled stubs, and sysroots are downloaded per target architecture (e.g., `Belt_x86_64-windows`, `Belt_arm64-macos`, `Belt_x86_64-linux`) and deduplicated at the component level using 32-character SHA-256 content-addressed directories guarded by a kernel-managed blocking lockfile. See [`toolbelt-and-sdk-cas.md`](toolbelt-and-sdk-cas.md).

### B. Standard `std` Rust Core + Audited Crate-Level SIMD ASM
An installer runtime that performs HTTPS TLS downloads (`on_missing`), streaming `zstd` decompression, Argon2id/AEAD decryption, JSON logging, atomic rollback journals, and native GUI rendering requires full OS threads, sockets, and heap allocation—making `no_std` counterproductive.
- **Core Engine & Stubs (Size-Optimized `std` Rust):** Built in standard Rust compiled with `opt-level = "z"`, LTO, `codegen-units = 1`, `panic = "abort"`, and symbol stripping. This produces compact, zero-DLL-dependency binaries cleanly across Windows, macOS, and Linux without reinventing `std`.
- **Surgical Crate-Level SIMD ASM:** Instead of hand-writing raw `.s` files across 6 OS/CPU ABIs (and tripping Windows Defender heuristic packer flags with custom PE entry stubs), PackageX delegates CPU-bound hashing and decompression loops to audited Rust crates that ship tuned AVX2 / AVX-512 / SHA-NI / NEON assembly under the hood: [`blake3`](https://github.com/BLAKE3-team/BLAKE3), [`RustCrypto/hashes`](https://github.com/rustcrypto/hashes) (`sha2` with `asm`), and [`zstd-rs`](https://github.com/gyscos/zstd-rs).

### C. Pre-Compiled Stubs (CI-Built Once Per Release) + Deterministic Overlay
- PackageX compiles `px-stub` for each target OS/UI backend once per engine release inside PackageX's own CI (including a hosted macOS runner for `Belt_arm64-macos` / `Belt_x86_64-macos` stubs) and distributes them via sub-belts.
- Because stubs are pre-compiled, **end-users running `packx build` on Windows or Linux never need Xcode or a Mac** to assemble, sign, and notarize macOS installers.
- During `packx build`, `px-assembler` compiles `.px`, `.pxs` UI schemas, and `<name>.s.px` WASM hooks, packs payload files deterministically (`zstd`), optionally wraps the blob in AEAD password sealing, and emits native OS packages or self-contained overlay installers in seconds.

## 2. Three-Repository Organization & Crate Ownership

PackageX is split across three repositories under [`github.com/package-x`](https://github.com/package-x) with an explicit license and protocol boundary (`shim_protocol = 1` between `packx` and `packx-core`):

| Repository | License | Responsibility & Crates |
| :--- | :--- | :--- |
| **`package-x/packx`** | MIT | Permanent thin `$PATH` CLI shim (`px-shim`), blocking `toolbelt.lock` router (`px-toolbelt`), and global OS keyring signing config (`packx config --set-signkey`). |
| **`package-x/packagex`** | Apache-2.0 | Architecture specs (`/plan/`), core CLI (`px-cli`), `.px` parser/linter (`px-spec`), `.s.px` WASM compiler (`px-script`), AEAD crypto/local CA (`px-crypto`), `.pxs` UI compiler & bridge (`px-ui-compiler`, `px-ui-bridge`), assembler (`px-assembler`), runtime (`px-runtime`), and `px-stub`. |
| **`package-x/sdk`** | MIT *(with isolated `vendor/` MPL-2.0)* | Tiny (~3–8 MB `zstd`) pure-Rust target sub-belt binaries, pre-compiled CI stubs, belt `manifest.json` index + 64-char SHA-256 checksums, and credited MPL-2.0 vendored macOS crates (`apple-codesign`, `apple-dmg`, `apple-flat-package`). |

### Core Engine Workspace Layout (`package-x/packagex`)

```text
PackageX/
├── AGENTS.md                      # Contributor & agent ground rules
├── Cargo.toml
├── README.md
├── plan/                          # Architectural & specification documents (v3 baseline)
├── crates/
│   ├── px-shim/                   # Thin `packx` CLI router & offline/locked version resolver
│   ├── px-toolbelt/               # Toolbelt version manager, blocking lockfile, .pxbelt export, CAS sdk_index
│   ├── px-cli/                    # Core CLI commands (`build`, `check`, `test`, `init`, `inspect`, `sign`, `belt`)
│   ├── px-spec/                   # `.px` parser (`toml`/`serde_spanned`), OS step resolver, `packx check` linter
│   ├── px-script/                 # `<name>.s.px` Python-syntax -> WebAssembly compiler & static capability auditor
│   ├── px-crypto/                 # Multi-Slot Envelope Encryption (`argon2`, `chacha20poly1305`, `zeroize`) & local CA (`rcgen`)
│   ├── px-ui-compiler/            # Compiles `.pxs` (PackageX Schema) or bundles `.html` assets
│   ├── px-assembler/              # Cross-OS emitters (Win .exe/.msi/.msix, macOS .pkg/.dmg, Linux .deb/.rpm/AppImage/Flatpak)
│   ├── px-runtime/                # Same-volume staging, rollback journal, `wasmi` Hook SDK, `--dry-run`, sandbox, delta updater
│   ├── px-ui-bridge/              # Native UI trait bridge (WinLook wl_ui, AppKit/objc2, GTK/Qt, Tauri 2, ratatui TUI)
│   └── px-stub/                   # Size-optimized `std` Rust entrypoint embedded into self-contained installers
├── vendor/                        # Credited MPL-2.0 vendored macOS crates (apple-codesign, apple-dmg, apple-flat-package)
├── stubs/                         # Pre-built stub binaries per target OS & UI backend (CI-published to sub-belts)
└── templates/                     # Built-in `.px` templates (`desktop-app`, `cli-path-app`, `hybrid-app`, `system-service`)
```

## 3. End-to-End Build & Execution Pipeline

```text
                      [ Developer's Project ]
   app.px   +   wizard.pxs (or index.html)   +   setup.s.px   +   dist/*
      │                     │                        │               │
      ▼                     ▼                        ▼               ▼
┌───────────┐       ┌───────────────┐       ┌───────────────┐  ┌─────────────┐
│  px-spec  │       │ px-ui-compiler│       │  px-script    │  │ Reproducible│
│ (OS Steps │       │ (Validate PXS │       │ (.s.px ->     │  │   Packer    │
│ & `check`)│       │ / Bundle HTML)│       │  WASM + caps) │  │ (zstd+SHA)  │
└─────┬─────┘       └───────┬───────┘       └──────┬────────┘  └──────┬──────┘
      └─────────────────────┴──────────────────────┴──────────────────┘
                                       │
                                       ▼ (Optional: px-crypto AEAD Sealing + Recovery Slots)
                        ┌─────────────────────────────┐
                        │  px-assembler (via Toolbelt │
                        │   + CAS Target Sub-Belts)   │
                        └──────┬───────────────┬──────┘
               ┌───────────────┘               └───────────────┐
               ▼                       ▼                       ▼
     [ Windows Targets ]       [ macOS Targets ]       [ Linux Targets ]
    .exe (Signed PE Overlay)   .pkg / .dmg (Pure Rust) .deb / .rpm
    .msi (rust-msi CFBF)       Universal (arm64+x64)   AppImage (backhand) / Flatpak
    .msix (OPC + BlockMap)     Homebrew Cask           .desktop + systemd
    WinGet + Local/Prod Sign   apple-codesign + Notary GPG Signed + SBOM
```

## 4. Runtime Execution Lifecycle (`px-stub`)

When an end-user or IT admin runs an assembled self-contained installer:

1. **Zero-UAC CLI Pre-Check (`asInvoker` PE Manifest):**
	- On Windows, `px-stub` is always stamped `asInvoker` and explicitly calls `AttachConsole(ATTACH_PARENT_PROCESS)` when CLI flags are present so GUI-subsystem installers print cleanly to PowerShell/`cmd.exe`.
	- `-packagex-detail` (with optional `--json` / `--files`): locates `PxFooter` (via PE Certificate Table offset on signed Windows builds), prints the execution plan—or **base64-encoded encrypted noise** if password-sealed, without prompting—and exits `0`.
	- `--dry-run`: inspects the target machine (free disk space, locked files/ports, required elevation) without writing a single byte, then exits with a canonical diagnostic code.
	- `--silent` / `/S`: skips GUI initialization and runs unattended (accepting `--password-stdin` / `--recovery-key-stdin` if sealed) with standardized exit codes and optional `--log-json`.
2. **Footer & Integrity Verification:**
	- Validates `PKX_FOOT` magic bytes (reading `cert_table_offset - 64` when Authenticode-signed, or `EOF - 64` when unsigned) and verifies manifest integrity.
3. **Password Unlock (If Sealed):**
	- Transitions UI to `password_prompt` (or reads `--password-stdin` / `--recovery-key-stdin`), unlocks Key Slot 0 or Recovery Slot 1 via `Argon2id` / X25519, validates the first AEAD chunk tag before touching disk, and wipes key material from RAM via `zeroize`.
4. **Explicit Elevation Gate (`elevate`):**
	- Evaluates `[package].elevate` (`true`, `false`, or `"auto"`). Re-launches elevated (`ShellExecuteW("runas")` on Windows, `sudo`/Authorization Services on Unix) **only** when required and only after CLI inspection flags are ruled out.
5. **UI Initialization (`px-ui-bridge`):**
	- Detects display server availability. Launches native OS toolkit (WinLook `wl_ui`, AppKit, GTK/Qt) or Tauri 2 WebView; automatically falls back to `ratatui` TUI or headless CLI over SSH/Docker/CI.
6. **Transactional Execution (`px-runtime`):**
	- Extracts into a **same-volume hidden staging directory** (or `Desktop/<package_name>/` in `packx test` sandbox mode), verifies every file against the compile-time SHA-256 manifest, executes `.s.px` WASM hooks in `wasmi`, commits atomically, updates `$PATH`/services, and writes the white-labeled uninstaller. Any failure triggers reverse journal rollback.


## 5. Installation Scope, Protected Helper and Electron Runtime Policy

The Engine exposes a unified installation-scope policy across Windows, macOS and Linux: per-user by default, machine-wide when explicitly selected, and an optional installer choice only where the target-format capabilities support it. Scope is separate from `elevate`. The Engine resolves scope, privilege requirements, destinations, scheduler ownership and rollback paths before the SDK writer assembles a target artifact.

The installer contains its own small, restricted helper when the selected format supports helper execution. This helper reads the installer's authenticated protected operation plan and performs only the allowed operations. It is not a global PackageX Runtime manager, a general script shell, a package manager, or an end-user PackageX.exe.

For supported Electron targets:
- SDK detection identifies the original Electron executable, application payload, exact runtime and platform layout.
- The default build prompt asks whether to apply recommended Electron payload optimization. Declining or uncertain detection preserves the complete original distribution.
- The SDK encodes the exact runtime reference into an opaque authenticated payload control block beside the app payload. The final installer does not expose a loose readable .ref.json sidecar.
- The helper downloads and verifies only the exact build-time-pinned runtime and restores it to named logical payload slots in the transaction staging tree.
- The application keeps its original executable and layout. No PackageX Electron launcher is inserted.
- Updates keep the current app/runtime generation running while one next-release runtime may be downloaded and verified. After app exit, the updater commits app and runtime together, health-checks, and retains rollback until success.
- No global end-user Electron cache/store/service is created. The SDK may keep its own verified developer-side artifact cache.

Native format limits remain authoritative. The SDK capability handshake declares which formats safely support the restricted helper, installation scope and Electron runtime restoration. The Engine rejects unsupported combinations or uses a bundled runtime only when the developer has explicitly allowed that fallback.

Source plans:
- [Installation Scope](installation-scope.md)
- [Protected installer payload and helper](assembly-engine.md)
- [Updater and Electron generations](updater-and-launcher.md)
- [SDK Electron adapter](https://github.com/Package-X/sdk/blob/main/plan/architecture/electron.md)
- [SDK installation-scope matrix](https://github.com/Package-X/sdk/blob/main/plan/architecture/installation-scope.md)

