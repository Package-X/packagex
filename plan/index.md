# PackageX (`packx`) Master Planning Index

## 1. Vision & Scope

**PackageX (`packx`)** bridges the gap between legacy script-heavy installer builders (NSIS, Inno Setup) and Windows-only enterprise packagers (WiX Toolset, Master Packager Dev / MPDEV). By combining a **size-optimized standard Rust + crate-level SIMD ASM Assembly Engine**, a **versioned `packx toolbelt` with blocking lockfile and CAS SDK deduplication**, a **TOML-based `.px` manifest with OS-namespaced steps**, **WASM-first Python-syntax `<name>.s.px` scripts**, **AEAD password sealing with recovery key slots**, and **`.pxs` (PackageX Schema) native UIs**, developers can emit white-labeled, reproducible, enterprise-ready native installers across **Windows, macOS, and Linux** from a single source of truth.

### Ecosystem Alignment
- **Windows UI Layer:** Integrates with [`Kate-alt-69/WinLook`](https://github.com/Kate-alt-69/WinLook) (`wl_ui` Rust crate & `wl_animate`).
- **Cross-OS UI Layer:** Compiles `.pxs` (PackageX Schema) to WinLook `wl_ui` (Windows), AppKit via [`objc2`](https://github.com/madsmtm/objc2) (macOS), GTK/Qt (Linux), Tauri 2 (HTML mode), and automatic [`ratatui`](https://github.com/ratatui/awesome-ratatui/blob/main/README.md) TUI / headless CLI fallback.

## 2. Document Map

| Document | Focus Area | Key Decisions Covered |
| :--- | :--- | :--- |
| [`AGENTS.md`](../AGENTS.md) | Repo Ground Rules | Tab indentation, onboarding comments, white-label & UAC invariants, Phase 1 MVP vertical slice |
| [`architecture/overview.md`](architecture/overview.md) | System Architecture | `packx` thin shim, Cargo workspace crates, standard Rust + crate SIMD ASM split, CI-only macOS stub builds |
| [`architecture/toolbelt-and-sdk-cas.md`](architecture/toolbelt-and-sdk-cas.md) | Toolbelt & SDK Pool | Versioned engines, blocking OS lockfile (`toolbelt.lock`), atomic temp-rename extraction, `--offline`/`.pxbelt`, 32-char SHA-256 CAS pool |
| [`architecture/assembly-engine.md`](architecture/assembly-engine.md) | Binary Generation | Authenticode-safe PE footer (`cert_table_offset - 64`), Windows/macOS/Linux formats, reproducible builds, signing & SBOM |
| [`architecture/runtime-and-rollback.md`](architecture/runtime-and-rollback.md) | Runtime & Updates | Same-volume atomic staging, rollback journal, Electron immutable app/runtime generations, scope-aware transaction roots, `--dry-run`, and delta updater |
| [`architecture/installation-scope.md`](architecture/installation-scope.md) | Installation Scope | Per-user/machine-wide policy, native format capability matrix, elevation separation and scope-aware lifecycle |
| [`architecture/security-and-sealing.md`](architecture/security-and-sealing.md) | Security & Sealing | Multi-Slot Envelope Encryption (`Argon2id` + `XChaCha20-Poly1305`), Recovery Key & X25519 Company Public Key slots, sealed `-packagex-detail`, local CA signing, white-label stealth |
| [`architecture/updater-and-launcher.md`](architecture/updater-and-launcher.md) | Updater & Launcher | Signed update feeds, generic launcher behavior, Electron no-launcher exception, one-pending-runtime updates, embedded helper boundary and scope-aware scheduling |
| [`architecture/product-and-editions.md`](architecture/product-and-editions.md) | Open-Core Editions | Free indie/OSS CLI bangers vs. paid enterprise rollouts, MDM policies, Cloud KMS, and recovery escrow |
| [`architecture/sdk-and-crate-map.md`](architecture/sdk-and-crate-map.md) | Crate Pile & Matrix | Candidate dependency map, Apple vendoring/licensing and target capability matrix |
| [`architecture/upstream-implementation-map.md`](architecture/upstream-implementation-map.md) | Upstream Implementation | Engine/SDK/PackX ownership, concrete upstream projects, format integration boundaries, POC order and limitations |
| [`specs/px-format-spec.md`](specs/px-format-spec.md) | `.px` Manifest Spec | `scope`, `scope_default`, `elevate`, `<section>.<os>.<key>` step namespacing + `phase` ordering, `[security]`, mandatory `sha256` pins under `reproducible = true` |
| [`specs/script-s-px-spec.md`](specs/script-s-px-spec.md) | `.s.px` Scripting Spec | Python-syntax `<name>.s.px` -> WASM-first (`wasmi`) execution, sentinel-marked `$PATH` rollback, static capability audit |
| [`specs/ui-schema-and-winlook.md`](specs/ui-schema-and-winlook.md) | `.pxs` UI Compilation | `.pxs` native widget compiler, `password_prompt` + recovery-key toggle, CLI `$PATH` reload hint, headless `ratatui` TUI fallback |
| [`specs/cli-detail-spec.md`](specs/cli-detail-spec.md) | CLI & Inspection | `packx` CLI (`build`, `check`, `test`, `sign init-ca`, `belt`), Windows `AttachConsole`, sealed base64 `-packagex-detail`, `--password-stdin` |
| [`specs/exit-codes-and-errors.md`](specs/exit-codes-and-errors.md) | Exit Codes & Logs | Canonical MDM/CI exit-code table (`0`, `1`, `2`, `10`–`16`) and structured JSON log event schema |

## 3. Phased Implementation Roadmap (Post-Planning)

### Phase 0: Advance Planning & Spec Freeze (Current — v3.3 Baseline)
- [x] Rename CLI to `packx` and UI schema format from `.nes` to `.pxs` (PackageX Schema).
- [x] Define `packx toolbelt`, blocking OS lockfile, atomic CAS extraction, `--offline` / `.pxbelt`, and 32-char SHA-256 ref-counting.
- [x] Shift runtime/stub from `no_std` to size-optimized standard Rust + audited crate SIMD ASM; shift `.s.px` default to WASM-first (`wasmi`).
- [x] Specify Authenticode-safe PE footer discovery, same-volume staging, per-user/machine-wide scope, `elevate` + `asInvoker` runtime UAC, protected helper capabilities and `packx test` Desktop sandbox.
- [x] Specify Multi-Slot Envelope Encryption (`Argon2id` + `XChaCha20-Poly1305` + Recovery Key / X25519 slots), local CA signing (`rcgen`), Target Capability Matrix, and canonical exit codes.
- [ ] Freeze `wl_ui` widget trait interface so PackageX can develop against a mock UI backend while WinLook completes `wl_ui`.

### Implementation detail references

- [SDK upstream integration map](https://github.com/Package-X/sdk/blob/main/plan/implementation/upstream-integration-map.md) — concrete crate/tool APIs, writer recipes, known gaps and platform-native acceptance tests.
- [PackX dependency and resolution playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md) — Rust package-manager dependencies and the optional project-aware `--required` install flow.
- [PackX package-store state/recovery](https://github.com/Package-X/packx/blob/main/plan/implementation/package-store-state-and-recovery.md) — manifest/index/lock models, extraction safety, atomic registration and crash recovery.

### Phase 1: MVP Vertical Slice (`px-spec`, `px-toolbelt`, `px-shim`, Windows `.exe` Overlay)
- [ ] Implement `.px` parser (`toml`/`serde_spanned`), OS-namespaced step resolver, and `packx check` cross-OS linter.
- [ ] Implement `packx` shim router, blocking `toolbelt.lock`, `sdk_index.json` atomic CAS delta installer, and `--offline`/`.pxbelt`.
- [ ] Implement deterministic `zstd` packer, Authenticode-safe PE overlay `.exe` writer (`goblin`), `AttachConsole`, `-packagex-detail`, `--dry-run`, and `packx test` Desktop sandbox.

### Phase 2: Runtime, Rollback, WASM `.s.px` & Security Sealing (`px-runtime`, `px-script`)
- [ ] Implement same-volume hidden staging, reverse rollback journal, sentinel-marked `$PATH` updates, and white-label zero-trace uninstallers.
- [ ] Implement `<name>.s.px` -> WebAssembly compiler and `wasmi` Hook SDK executor inside `px-stub`.
- [ ] Implement Multi-Slot Envelope Encryption (`argon2` + `chacha20poly1305` + `zeroize`), sealed base64 `-packagex-detail`, and `packx sign init-ca` (`rcgen` + `osslsigncode`).

### Phase 3: Cross-OS Native Format Emitters & Auto-Updater (`px-assembler`)
- [ ] Implement Windows `.msi` (`rust-msi`), `.msix`, and WinGet manifest emitters.
- [ ] Implement macOS `.dmg` (`apple-dmg`), `.pkg` (extended vendored `apple-flat-package`), Homebrew Cask, and `apple-codesign` signing/notarization + CI-built macOS `px-stub`.
- [ ] Implement Linux `.deb` (`cargo-deb`), `.rpm`, `AppImage` (`backhand`), and `Flatpak` emitters.
- [ ] Implement uncompressed-stream `zstd` binary delta auto-updater and SPDX/CycloneDX SBOMs.

### Phase 4: `.pxs` UI Compiler & Native OS Backends (`px-ui-compiler`, `px-ui-bridge`)
- [ ] Build `.pxs` -> native widget tree compiler for WinLook `wl_ui` (`win10`/`win11`), macOS AppKit (`objc2`), and Linux GTK/Qt.
- [ ] Build HTML/CSS bundler for Tauri 2 mode and automatic `ratatui` TUI / headless CLI fallback.
