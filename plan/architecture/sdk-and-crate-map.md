# Rust Crate Pile, Vendored SDK Strategy & Target Capability Matrix

PackageX builds *on top of* battle-tested Rust crates and SDK tools rather than reinventing binary container formats from scratch, reserving custom engineering for the unified `.px` IR, CAS toolbelt, rollback runtime, `.pxs` UI compiler, and AEAD sealing layer.

## 1. Core Rust Crate Pile by Workspace Module

| PackageX Crate | Role | Recommended Rust Crates / SDKs |
| :--- | :--- | :--- |
| **`px-cli` & `px-shim`** | Subcommands, flags, instant shim dispatch | [`clap`](https://github.com/clap-rs/clap) (`px-cli`); keep `px-shim` minimal-dependency for instant launch. |
| **`px-spec`** | `.px` TOML parsing, span-accurate `packx check` diagnostics, SemVer | [`toml` / `toml_edit`](https://github.com/toml-rs/toml) (`serde_spanned`), [`semver`](https://github.com/dtolnay/semver). |
| **`px-toolbelt`** | CAS downloads, 64-char SHA-256 verification, blocking lockfile | [`ureq`](https://github.com/algesten/ureq) (tiny blocking HTTP/TLS, no Tokio bloat), [`RustCrypto/hashes`](https://github.com/rustcrypto/hashes), [`blake3`](https://github.com/BLAKE3-team/BLAKE3). |
| **`px-crypto`** | Multi-Slot AEAD sealing, Argon2id KDF, RAM zeroing, local CA | [`RustCrypto/password-hashes`](https://github.com/RustCrypto/password-hashes) (`argon2`), [`RustCrypto/AEADs`](https://github.com/rustcrypto/aeads) (`chacha20poly1305`), [`RustCrypto/utils`](https://github.com/rustcrypto/utils) (`zeroize`), [`rcgen`](https://github.com/rustls/rcgen). |
| **`px-assembler`** | Deterministic `zstd` packing, PE/ELF/Mach-O inspection, multi-OS formats | [`zstd-rs`](https://github.com/gyscos/zstd-rs), [`goblin`](https://github.com/m4b/goblin), [`cargo-packager`](https://github.com/crabnebula-dev/cargo-packager) (library/reference), [`cargo-dist`](https://github.com/axodotdev/cargo-dist/releases) (feed/CI reference). |
| **Windows Formats & Signing** | `.msi`, cross-SDK headers, Authenticode signing | [`rust-msi`](https://github.com/mdsteele/rust-msi), [`wixtoolset/wix`](https://github.com/wixtoolset/wix) (reference), [`xwin`](https://github.com/Jake-Shadle/xwin) / [`cargo-xwin`](https://github.com/rust-cross/cargo-xwin) (model for `Belt_x86_64-windows`), [`osslsigncode`](https://github.com/mtrojnar/osslsigncode). |
| **macOS Formats, UI & Signing** | Zero-Xcode signing/notarizing, `.dmg`, `.pkg`, AppKit UI | Vendored [`apple-codesign`](https://crates.io/crates/apple-codesign) / [`tauri-macos-sign`](https://crates.io/keywords/codesign), [`apple-dmg`](https://crates.io/crates/apple-dmg/dependencies), [`apple-flat-package`](https://docs.rs/apple-flat-package) / [`isideload-apple-flat-package`](https://crates.io/crates/isideload-apple-flat-package), [`apple-bom`](https://lib.rs/crates/apple-bom), [`objc2`](https://github.com/madsmtm/objc2). |
| **Linux Formats & Linking** | `.deb`, `AppImage` SquashFS, older-`glibc` cross-linking | [`cargo-deb`](https://github.com/kornelski/cargo-deb), [`backhand`](https://crates.io/crates/backhand), [`cargo-zigbuild`](https://github.com/rust-cross/cargo-zigbuild). |
| **`px-script`** | `<name>.s.px` WASM execution inside `px-stub` | [`wasmi`](https://github.com/wasmi-labs/wasmi) (deterministic, security-audited Rust WebAssembly interpreter). |
| **`px-ui-bridge`** | Native OS UIs + SSH/Docker TUI fallback | `WinLook` (`wl_ui`), [`objc2`](https://github.com/madsmtm/objc2) (AppKit), GTK4/Qt, [`ratatui`](https://github.com/ratatui/awesome-ratatui/blob/main/README.md). |

## 2. macOS Strategy: Zero Xcode for Users, MPL-2.0 Vendoring & CI-Built Stubs

1. **Why We Vendor & Patch `apple-platform-rs`:** Upstream [`indygreg/apple-platform-rs`](https://github.com/indygreg/apple-platform-rs) has slowed to mostly automated dependency bumps, and `.pkg` / BOM writing remains incomplete ([Issue #3](https://github.com/indygreg/apple-platform-rs/issues/3)). PackageX vendors the needed crates under `vendor/` (pulling fresher fixes from active forks like [`isideload-apple-flat-package`](https://crates.io/crates/isideload-apple-flat-package) and [`tauri-macos-sign`](https://crates.io/keywords/codesign)) and extends the `.pkg` BOM writer in-house.
2. **MPL-2.0 License Compliance:** `apple-codesign` and `apple-flat-package` are **MPL-2.0** (file-level copyleft). PackageX retains all upstream copyright/license notices and keeps modifications to those specific vendored MPL-2.0 files open-source under MPL-2.0, without restricting the license of the rest of PackageX.
3. **Dodging the Apple SDK Licensing Wall:** While [`osxcross`](https://github.com/tpoechtrager/osxcross) can cross-compile from Linux, Apple's license forbids redistributing the macOS SDK headers outside Apple hardware. Because PackageX uses **pre-compiled `px-stub` binaries inside `Belt_arm64-macos`**, only PackageX's own CI (a hosted macOS runner) ever compiles `px-stub` once per release—end-users assemble, sign, and notarize macOS installers from Windows or Linux with zero Mac hardware. (Public Gatekeeper trust still requires the publisher's $99/yr Apple Developer ID certificate.)

## 3. Target Capability Matrix (All 11 Formats)

Because native OS packages (`.msi`, `.deb`, `.rpm`, `.pkg`, `.msix`) are executed by the **OS's own installer engine** rather than `px-stub`, `packx check` enforces this exact capability matrix and warns whenever a declared `.px` feature is lowered or unsupported on a target:

| Feature | Win `.exe` (Overlay) | Win `.msi` | Win `.msix` | WinGet | macOS `.pkg` | macOS `.dmg` | Homebrew | Linux `.deb` / `.rpm` | Linux `AppImage` | Linux `Flatpak` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Custom `.pxs` / HTML UI** | Native | Optional (`MsiSetExternalUIW`) | No (OS UI) | No | No (Installer.app) | Background layout | No | No (OS pkg mgr) | Optional (stub) | No |
| **`-packagex-detail` & `--dry-run`** | Native | No | No | Metadata YAML | No | No | Formula RB | No | Native (if stub) | No |
| **AEAD Password Sealing + Recovery** | Native | No | No | No | No | Encrypted DMG (future) | No | No | Native (if stub) | No |
| **`.s.px` WASM Hooks** | Native (`wasmi`) | Lowered (CustomAction) | Warn (AppContainer) | No | Lowered (`postinstall`) | No | Stanza | Lowered (`postinst`/`prerm`) | Native (if stub) | Sandbox-limited |
| **Same-Volume Atomic Rollback** | Native | Native (`msiexec` TX) | Native (AppX) | Delegates | OS receipt | Drag-drop | Cask | OS pkg mgr | Self-contained | OSTree |
| **`elevate = true / false`** | Native (runtime) | `ALLUSERS` property | User-scoped | Manifest scope | `install-location` | User `/Applications` | User/Admin | Root (`dpkg`/`rpm`) | User (no root) | User or system |
| **`packx test` Desktop Sandbox** | Native | No | No | No | No | Native (mount) | No | No | Native | No |
