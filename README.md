# PackageX (`packx`)

**A Cross-OS Rust installer assembly engine, developer Toolbelt, and embedded installer runtime.**

PackageX builds native, white-labeled installers for **all modern operating systems**—Windows (`.exe`, `.msi`, `.msix`, WinGet), macOS (`.pkg`, `.dmg`, Homebrew Cask), and Linux (`.deb`, `.rpm`, `AppImage`, `Flatpak`)—from a single declarative `.px` manifest, outclassing legacy script-heavy builders and Windows-only packagers like Master Packager Dev (MPDEV).

> **Project Status:** **Advance Planning Phase (v3.3 Spec Baseline).** No `.rs` implementation code is present yet. All architectural blueprints, binary format specifications, `packx toolbelt` CAS SDK designs, AEAD password sealing & recovery-slot specs, Rust crate mappings, `.pxs` UI contracts, `.s.px` WASM scripting specs, and standardized exit codes live inside `/plan/`.

## Core Capabilities (Planned)

1. **Cross-OS Multi-Target Assembly Engine (`px-assembler`):**
	- Built in **standard `std` Rust** (size-optimized with `opt-level = "z"`, LTO, and symbol stripping) rather than `no_std`, delegating SIMD assembly hotspots to audited crates ([`blake3`](https://github.com/BLAKE3-team/BLAKE3), [`RustCrypto/hashes`](https://github.com/rustcrypto/hashes), [`zstd-rs`](https://github.com/gyscos/zstd-rs)) to avoid Windows Defender false positives and 6-ABI maintenance friction.
	- **Windows (`x86_64` / `arm64`):** Custom Authenticode-safe PE overlay `.exe` stub, pure-Rust `.msi` ([`rust-msi`](https://github.com/mdsteele/rust-msi)), signed `.msix` (OPC + `AppxBlockMap.xml`), and auto-generated WinGet manifests.
	- **macOS (`arm64` / `x86_64` Universal):** Zero-Xcode signing, notarization, and stapling via vendored [`apple-codesign`](https://crates.io/crates/apple-codesign) / active forks, `.dmg` via [`apple-dmg`](https://crates.io/crates/apple-dmg/dependencies), `.pkg` via extended [`apple-flat-package`](https://docs.rs/apple-flat-package), and Homebrew Cask definitions. Pre-compiled macOS stubs are built once per release in CI so end-users never need a Mac.
	- **Linux (`x86_64` / `arm64`):** Native `.deb` ([`cargo-deb`](https://github.com/kornelski/cargo-deb)), `.rpm`, `AppImage` ([`backhand`](https://crates.io/crates/backhand) SquashFS), and `Flatpak` packages with `.desktop` and `systemd` unit generation.
2. **Versioned `packx toolbelt`, Blocking Lockfile & CAS SDK Pool:**
	- `packx` acts as a thin shim router that reads the pinned engine version from `.px` (e.g., `engine = "2.4"`) and dispatches to side-by-side engines under `~/.packx/toolbelt/<version>/`.
	- Kernel-managed blocking file lock (`~/.packx/toolbelt.lock`) prevents concurrent `belt` runs from corrupting `sdk_index.json` while displaying live wait diagnostics (`--lock-timeout` for CI).
	- **Component-Level Content-Addressed Storage (CAS):** SDK modules live under `<32-char-sha256>_<component>/` (dodging Windows `MAX_PATH`) with full 64-char SHA-256 verification, atomic temp-rename extraction, delta installs, and `shared` vs. `owner` reference counting.
	- **Strict `--offline` / `--locked` Mode & `.pxbelt` Bundles:** Air-gapped enterprise builds fail cleanly with actionable diagnostics and support portable `.pxbelt` export/import.
3. **WASM-First Python-Syntax Scripting (`<name>.s.px`) & Cross-OS Hook SDK:**
	- Developers write clean Python-like install logic in `<name>.s.px` using rollback-tracked primitives (`packx.env.add_path()`, `packx.service.register()`, `packx.fs.ensure_dir()`).
	- Compiled by default to portable **WebAssembly** executed inside `px-stub` via [`wasmi`](https://github.com/wasmi-labs/wasmi)—true write-once cross-OS execution, instant sandboxing, and static capability auditing without per-OS cross-linkers.
4. **Enterprise Security, Password Sealing & White-Label Stealth:**
	- **White-Label End-User Footprint:** Zero `PackageX` or `px-` branding in user install paths, uninstaller folders (`%LOCALAPPDATA%\<package_name>\uninstaller\<app_id>.exe` / `%ProgramData%`), temp staging, shell `$PATH` markers, or PE Version Info.
	- **Internal Password Sealing + Recovery Key Slots:** Single-blob `zstd` packing wrapped in Multi-Slot Envelope Encryption (`Argon2id` + `XChaCha20-Poly1305`) with optional offline Recovery Key or Company X25519 Master Public Key slots.
	- **Free Local Company Signing + Production Signing:** Local offline Root CA generation (`packx sign init-ca` via [`rcgen`](https://github.com/rustls/rcgen) + [`osslsigncode`](https://github.com/mtrojnar/osslsigncode)) for $0 GPO/Intune-trusted internal installers, plus Azure Trusted Signing, Apple Notary API, Linux GPG, and SPDX/CycloneDX SBOMs.
5. **Dual-Mode Cross-OS UI (`.pxs` PackageX Schema) & Developer Sandbox:**
	- Compiles `.pxs` natively to **WinLook `wl_ui`** (Windows), **AppKit** via [`objc2`](https://github.com/madsmtm/objc2) (macOS), **GTK/Qt** (Linux), **Tauri 2** HTML mode, and automatic [`ratatui`](https://github.com/ratatui/awesome-ratatui/blob/main/README.md) TUI / headless CLI fallback.
	- **`elevate = true | false | "auto"`** with `asInvoker` PE manifest + runtime self-elevation keeps `-packagex-detail` and `--dry-run` 100% UAC-free.
	- **`packx test` Desktop Sandbox:** Installs into `Desktop/<package_name>/` and logs virtualized `$PATH`/Registry/service mutations to `simulated-mutations.json` so dev machines stay spotless.

## Repository Planning Index

Explore `/plan/` for the complete v3 technical specifications:

- [`AGENTS.md`](AGENTS.md) — Contributor & agent coding conventions, white-label invariants, and Phase 1 MVP scope
- [`plan/index.md`](plan/index.md) — Master workspace index & phased roadmap
- [`plan/architecture/overview.md`](plan/architecture/overview.md) — `packx` shim, Cargo workspace layout, standard Rust + crate SIMD ASM split, and runtime lifecycle
- [`plan/architecture/toolbelt-and-sdk-cas.md`](plan/architecture/toolbelt-and-sdk-cas.md) — `packx toolbelt`, blocking lockfile, atomic CAS extraction, `--offline`/`.pxbelt`, and 32-char SHA-256 ref-counting
- [`plan/architecture/assembly-engine.md`](plan/architecture/assembly-engine.md) — Cross-OS binary layouts, Authenticode-safe PE footer discovery, reproducible builds, and signing/SBOM
- [`plan/architecture/runtime-and-rollback.md`](plan/architecture/runtime-and-rollback.md) — Same-volume atomic staging, rollback journal, white-label zero-trace uninstalls, `--dry-run`, `packx test` sandbox, and uncompressed `zstd` delta updater
- [`plan/architecture/installation-scope.md`](plan/architecture/installation-scope.md) — Per-user and machine-wide install scope across Windows, macOS and Linux\n- [`plan/architecture/security-and-sealing.md`](plan/architecture/security-and-sealing.md) — Multi-Slot Envelope Encryption (`Argon2id` + `XChaCha20-Poly1305`), Recovery Key / X25519 Company Public Key slots, sealed `-packagex-detail`, local CA signing, and white-label stealth rules
- [`plan/architecture/updater-and-launcher.md`](plan/architecture/updater-and-launcher.md) — Signed update feeds, generic launcher policy, Electron no-launcher behavior, one-pending-runtime updates, embedded helper boundary and scope-aware scheduling
- [`plan/architecture/product-and-editions.md`](plan/architecture/product-and-editions.md) — Open-core free indie/OSS features vs. paid enterprise fleet/compliance tier
- [`plan/architecture/sdk-and-crate-map.md`](plan/architecture/sdk-and-crate-map.md) — High-level crate shortlist, Apple MPL-2.0 vendoring and target capability matrix
- [`plan/architecture/upstream-implementation-map.md`](plan/architecture/upstream-implementation-map.md) — How Engine, SDK and PackX integrate concrete upstream projects, target writers and validation
- [`plan/specs/px-format-spec.md`](plan/specs/px-format-spec.md) — Complete `.px` TOML schema, `scope`, `scope_default`, `elevate`, `<section>.<os>.<key>` step namespacing + `phase` ordering, `[security]`, and reproducible SHA-256 pinning
- [`plan/specs/script-s-px-spec.md`](plan/specs/script-s-px-spec.md) — `<name>.s.px` Python-syntax scripting -> WASM-first (`wasmi`) execution and sentinel-marked Hook SDK
- [`plan/specs/ui-schema-and-winlook.md`](plan/specs/ui-schema-and-winlook.md) — `.pxs` (PackageX Schema) compiler -> WinLook `wl_ui` / AppKit / GTK-Qt / Tauri 2 / `ratatui` TUI + `password_prompt` state
- [`plan/specs/cli-detail-spec.md`](plan/specs/cli-detail-spec.md) — `packx` CLI (`build`, `check`, `test`, `sign init-ca`, `belt`), Windows `AttachConsole`, sealed `-packagex-detail` base64 output, and `--dry-run`
- [`plan/specs/exit-codes-and-errors.md`](plan/specs/exit-codes-and-errors.md) — Canonical standardized exit codes and structured JSON log event schema for MDM/CI
- [`plan/examples/kastrick-devtool.px`](plan/examples/kastrick-devtool.px) — Reference cross-OS `.px` manifest with OS-namespaced steps, `elevate = false`, `.pxs` UI, and WASM `.s.px` hook
- [`plan/examples/internal-sealed-app.px`](plan/examples/internal-sealed-app.px) — Reference internal company manifest with password sealing, X25519 recovery slot, and local CA signing
- [`plan/examples/wizard.pxs`](plan/examples/wizard.pxs) — Reference `.pxs` native UI wizard with `PasswordPage` and recovery-key toggle
- [`plan/examples/setup.s.px`](plan/examples/setup.s.px) — Reference `<name>.s.px` Python-syntax install script using the Hook SDK
