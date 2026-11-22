# Rust Crate Pile, Vendored SDK Strategy & Target Capability Matrix

PackageX builds *on top of* battle-tested Rust crates and SDK tools rather than reinventing binary container formats from scratch. The Engine owns the unified `.px` IR, build graph, rollback runtime, `.pxs` UI compiler, and AEAD sealing layer; PackX owns Toolbelt/CAS; the SDK owns target-specific format adapters and writers.

## 1. Core Rust Crate Pile by Workspace Module

| PackageX Crate | Role | Recommended Rust Crates / SDKs |
| :--- | :--- | :--- |
| **`px-cli` (Engine)** | Project/build commands such as `build`, `check`, `test`, `init`, `preview` | [`clap`](https://github.com/clap-rs/clap); keep Engine project semantics here. |
| **PackX shim + Toolbelt (separate repo)** | Permanent command router, Engine selection, SDK installation, signed catalog, CAS, Toolchain locks | [`ureq`](https://docs.rs/ureq/latest/ureq/), [`fs4`](https://docs.rs/fs4/latest/fs4/), [`tempfile`](https://docs.rs/tempfile/latest/tempfile/), `serde`, `semver`, and `minisign-verify` (see [PackX implementation playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md)). |
| **`px-spec`** | `.px` TOML parsing, span-accurate `packx check` diagnostics, SemVer | [`toml` / `toml_edit`](https://github.com/toml-rs/toml) (`serde_spanned`), [`semver`](https://github.com/dtolnay/semver). |
| **`px-toolbelt`** | CAS downloads, 64-char SHA-256 verification, blocking lockfile | [`ureq`](https://github.com/algesten/ureq) (tiny blocking HTTP/TLS, no Tokio bloat), [`RustCrypto/hashes`](https://github.com/rustcrypto/hashes), [`blake3`](https://github.com/BLAKE3-team/BLAKE3). |
| **`px-crypto`** | Multi-Slot AEAD sealing, Argon2id KDF, RAM zeroing, local CA | [`RustCrypto/password-hashes`](https://github.com/RustCrypto/password-hashes) (`argon2`), [`RustCrypto/AEADs`](https://github.com/rustcrypto/aeads) (`chacha20poly1305`), [`RustCrypto/utils`](https://github.com/rustcrypto/utils) (`zeroize`), [`rcgen`](https://github.com/rustls/rcgen). |
| **`px-assembler`** | Deterministic `zstd` packing, PE/ELF/Mach-O inspection, multi-OS formats | [`zstd-rs`](https://github.com/gyscos/zstd-rs), [`goblin`](https://github.com/m4b/goblin), [`cargo-packager`](https://github.com/crabnebula-dev/cargo-packager) (library/reference), [`cargo-dist`](https://github.com/axodotdev/cargo-dist/releases) (feed/CI reference). |
| **Windows Formats & Signing** | `.msi`, `.msix`, PE signing | [`msi`](https://docs.rs/msi/latest/msi/), [`cfb`](https://docs.rs/cfb/latest/cfb/), [`cab`](https://docs.rs/cab/latest/cab/), [Microsoft MSIX Packaging SDK](https://github.com/microsoft/msix-packaging), Windows SignTool; `osslsigncode` is optional GPL-3.0 tooling only after license review. |
| **macOS Formats, UI & Signing** | Zero-Xcode signing/notarizing, `.dmg`, `.pkg`, AppKit UI | Vendored [`apple-codesign`](https://crates.io/crates/apple-codesign) / [`tauri-macos-sign`](https://crates.io/keywords/codesign), [`apple-dmg`](https://crates.io/crates/apple-dmg/dependencies), [`apple-flat-package`](https://docs.rs/apple-flat-package) / [`isideload-apple-flat-package`](https://crates.io/crates/isideload-apple-flat-package), [`apple-bom`](https://lib.rs/crates/apple-bom), [`objc2`](https://github.com/madsmtm/objc2). |
| **Linux Formats & Linking** | `.deb`, `.rpm`, AppImage, Flatpak and cross-linking | [`debian-packaging`](https://docs.rs/debian-packaging/latest/debian_packaging/), [`rpm`](https://docs.rs/rpm/latest/rpm/), [`appimagetool`](https://docs.rs/appimagetool/latest/appimagetool/), official `flatpak-builder`; [`backhand`](https://docs.rs/backhand/latest/backhand/) is for SquashFS rather than the current DWARFS AppImage path, and [`cargo-deb`](https://github.com/kornelski/cargo-deb) remains an optional Cargo-app-oriented path. |
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


## 4. Implementation Feasibility and Upstream Integration Recipes

The table above is the high-level dependency shortlist, not proof that each candidate implements every operation. The detailed capability recipes are maintained in the dedicated [SDK upstream integration map](https://github.com/Package-X/sdk/blob/main/plan/implementation/upstream-integration-map.md).

### Windows

- MSI: use the Rust msi crate as the first portable database writer/editor candidate. Its API exposes read/write open, row selection/insertion/update/deletion, embedded stream operations and flush. PackageX wraps this in staged-copy editing: inspect/hash source, validate a typed change plan, copy to staging, apply supported edits, flush/close, reopen read-only, verify logical structures, then atomically publish. The Windows Installer database API remains a Windows-only backend for native transaction commit and transform behavior.
- MSIX: use Microsoft's MSIX Packaging SDK rather than reimplementing OPC packaging and block-map generation first. It is a cross-platform native library; pack support must be explicitly enabled at build time with MSIX_PACK=on / --pack and USE_VALIDATION_PARSE=on. Signing and actual Windows deployment validation are separate stages.
- WiX: keep it as reference/optional external backend. WiX v6/v7 commercial use may be subject to the Open Source Maintenance Fee; it must not quietly become a mandatory commercial dependency.
- osslsigncode: useful as an optional external signing/verification adapter, but GPL-3.0 requires explicit distribution review. Prefer Windows SignTool for authoritative Windows publisher-signing tests.

### macOS

- Use focused apple-platform-rs crates: apple-bundles, apple-flat-package, apple-xar, apple-bom, apple-dmg, cpio-archive and apple-codesign.
- .pkg creation has a known gap: apple-bom documentation describes BOM writing as work in progress. Prove the current writer against Apple's native package tools before advertising portable .pkg support. Until then use pkgbuild/productbuild on a macOS runner as a declared fallback, or implement only the missing BOM writer behind a dedicated adapter.
- apple-codesign can construct/sign Mach-O and bundles and supports notarization/stapling flows, but native macOS codesign, spctl, pkgutil and actual installer tests remain the final acceptance gate. MPL-2.0 notices and file-level modification rules must be preserved.

### Linux

- DEB: use debian-packaging as a general-purpose Rust packaging candidate; it is MPL-2.0. cargo-deb is useful for Cargo applications but its library API is documented as experimental, so it is not automatically the generic PackageX resource-tree writer.
- RPM: the rpm crate offers Rust programmatic package building/reading, but it does not replace rpmbuild in full or manipulate the system RPM database. Validate with the native RPM CLI and clean distro containers.
- AppImage: evaluate appimagetool with uruntime and mkdwarfs for the current DWARFS-based path. backhand is useful for SquashFS, but it is not a drop-in replacement for a DWARFS AppImage builder.
- Flatpak: invoke the official flatpak-builder/Flatpak toolchain in an isolated Linux worker after generating a pinned manifest; do not reimplement OSTree repository, sandbox and permission semantics.

### Cross-cutting

- Electron app/runtime separation belongs to the SDK's px-electron component, with @electron/get/official release artifacts for exact runtime resolution and @electron/asar when ASAR creation/extraction is required.
- Payload deduplication can use fastcdc; compression can use streaming zstd. Pin chunker algorithm/version/parameters and codec settings; skip recompression when it does not reduce size.
- Keep the Engine's canonical uncompressed resource inventory. SDK PDC representation is derived output and must reconstruct to original per-file hashes.
- External builders/validators are SDK components with pinned versions, hashes, licenses and explicit install conditions; they are not ambient PATH dependencies.

### Source-of-truth links

- [SDK upstream integration map](https://github.com/Package-X/sdk/blob/main/plan/implementation/upstream-integration-map.md)
- [SDK writer/transaction design](https://github.com/Package-X/sdk/blob/main/plan/architecture/writers-and-transactions.md)
- [SDK Electron adapter](https://github.com/Package-X/sdk/blob/main/plan/architecture/electron.md)
- [PackX resolver implementation plan](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md)


## 5. Upstream implementation detail is maintained in the SDK/PackX repositories

This central table is a navigation-level shortlist. Do not treat it as a guarantee that every listed upstream supports every writer operation.

- The [SDK upstream integration map](https://github.com/Package-X/sdk/blob/main/plan/implementation/upstream-integration-map.md) contains API-level integration recipes for MSI/MSIX, Apple bundle/PKG/DMG/signing, DEB/RPM/AppImage/Flatpak, Electron and PDC. It records known gaps such as the apple-bom writer and the current AppImage DWARFS path.
- The [SDK writer/transaction plan](https://github.com/Package-X/sdk/blob/main/plan/architecture/writers-and-transactions.md) defines create/inspect/modify/transform/verify capabilities and the staged MSI editing cycle.
- The [PackX dependency/resolution playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md) defines the Rust package-manager stack and the optional project-aware `--required` install algorithm; direct package installs do not require that flag.
- The [PackX package-store recovery plan](https://github.com/Package-X/packx/blob/main/plan/implementation/package-store-state-and-recovery.md) defines CAS state transitions, signed manifests, safe extraction, registration journals and leases.

The SDK repository owns writer implementation details. The PackX repository owns package acquisition and Toolchain resolution. This Engine repository owns project requirements and the BuildRequest it sends to the SDK.
