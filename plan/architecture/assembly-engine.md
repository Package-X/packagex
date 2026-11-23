# Assembly Engine Binary Specifications (`px-assembler`)

`px-assembler` takes a validated `ResolvedPackage` IR and compiles native, white-labeled, enterprise-ready installers across **Windows, macOS, and Linux** using the active `packx toolbelt`, target sub-belts, and the Rust crate stack documented in [`sdk-and-crate-map.md`](sdk-and-crate-map.md).

## 1. Reproducible Builds Contract (Deterministic Packing)

Running `packx build` on two different machines or OSs with the same `.px` manifest, pinned `engine` version, and input payload produces **byte-for-byte identical output files (identical SHA-256)** when `reproducible = true` (and password sealing is disabled; see [`security-and-sealing.md`](security-and-sealing.md)).

`px-assembler` enforces four strict determinism rules across all targets:
1. **Alphabetical Ordering:** All input files, manifest tables, and archive entries are lexicographically sorted (UTF-8 byte order) prior to writing.
2. **Frozen Timestamps:** Embedded file modification timestamps are clamped to `0` (Unix epoch) or `SOURCE_DATE_EPOCH`—never wall-clock build time.
3. **Relative Paths Only:** All internal paths are normalized to forward-slash relative paths (`bin/app.exe`); absolute host paths are stripped.
4. **Mandatory `sha256` Pins on Remote Dependencies:** When `reproducible = true`, `packx check` requires a 64-char `sha256` pin on every `on_missing` URL (unless `--allow-unpinned-deps` is explicitly passed).
5. **Pinned Build Toolchain Inputs:** Every toolbelt release pins exact `rustc` version, target triple, `Cargo.lock`, `libzstd` version, and pre-compiled `px-stub` SHA-256 digests in `manifest.json` so reproducibility holds across hosts.

Additionally, `px-assembler` embeds a per-file SHA-256 manifest (using [`RustCrypto/hashes`](https://github.com/rustcrypto/hashes) / [`blake3`](https://github.com/BLAKE3-team/BLAKE3)) plus exact compressed size and compile-time estimated uncompressed install size.

## 2. Windows Targets (`Belt_x86_64-windows` / `Belt_arm64-windows`)

### Target W1: Custom `.exe` Installer (Authenticode-Safe PE Overlay Format)
Appends compressed payload, compiled `<name>.s.px` WASM hook module, UI bundle, and manifest to a pre-compiled size-optimized `std` Rust PE stub (`px-stub.exe`) without mutating PE section headers.

- **White-Label PE Metadata Stamping:** Before sealing the overlay, `px-assembler` stamps the publisher's `name`, `version`, `publisher`, and icon into the PE Version Resource (`FileDescription`, `CompanyName`, `ProductName`) and stamps `requestedExecutionLevel = asInvoker` into the PE Application Manifest (zero UAC on `-packagex-detail` / `--dry-run`).
- **Authenticode Signing Collision Fix:** Because `signtool` and [`osslsigncode`](https://github.com/mtrojnar/osslsigncode) append the PKCS#7 Certificate Table **after the PE overlay at the new EOF**, `px-stub` MUST NOT blindly read `EOF - 64`. Instead, `px-stub` parses its own PE Optional Header via [`goblin`](https://github.com/m4b/goblin) to locate `IMAGE_DIRECTORY_ENTRY_SECURITY` (`cert_table_offset`) and reads `PxFooter` at **`cert_table_offset - 64`** (falling back to `EOF - 64` only when unsigned).

```text
+-------------------------------------------------------------------+
| 1. Pre-Compiled `std` Rust PE Stub (px-stub.exe, `asInvoker`)     |
|    - Stamped with Publisher Version Info & App Icon (White-Label) |
+-------------------------------------------------------------------+  <-- overlay_offset
| 2. Deterministic Compressed Payload (zstd-19, 4096-byte aligned)  |
|    (OR Single Opaque AEAD-Encrypted Blob when `[security]` on)    |
+-------------------------------------------------------------------+  <-- script_offset
| 3. Compiled `<name>.s.px` WASM Hook Block (`wasmi` bytecode)      |
+-------------------------------------------------------------------+  <-- ui_offset
| 4. Compiled UI Bundle (`.pxs` binary AST or HTML archive)         |
+-------------------------------------------------------------------+  <-- manifest_offset
| 5. Compiled Execution Manifest (Bincode/MessagePack + SHA-256s)   |
+-------------------------------------------------------------------+  <-- cert_table_offset - 64
| 6. Fixed 64-Byte Footer (`PxFooter`, magic: "PKX_FOOT" + crypto)  |
+-------------------------------------------------------------------+  <-- cert_table_offset (or EOF)
| 7. Optional Authenticode PKCS#7 Certificate Table (signtool/ossl) |
+-------------------------------------------------------------------+  <-- EOF
```

### Target W2: Native `.msi` Database (`CFBF` + `.cab`)
Pure-Rust synthesis of Windows Installer relational tables (`Property`, `Directory`, `Component`, `File`, `Registry`, `Environment`, `Shortcut`, `CustomAction`) built on [`rust-msi`](https://github.com/mdsteele/rust-msi) with deterministic UUIDv5 Component GUIDs, embedded `disk1.cab`, and optional WiX reference fallback ([`wixtoolset/wix`](https://github.com/wixtoolset/wix)).

### Target W3: `.msix` Modern App Package (`OPC` + `BlockMap`)
Pure-Rust generation of `AppxManifest.xml`, 64 KiB DEFLATE-chunked SHA-256 `AppxBlockMap.xml`, and `[Content_Types].xml`, with `packx check` warnings for AppContainer-incompatible steps.

### Target W4: WinGet Manifest Export
Emits ready-to-submit YAML manifests (`version`, `installer`, `defaultLocale`) matching the compiled installer SHA-256 and silent flags.

## 3. macOS Targets (`Belt_arm64-macos` / `Belt_x86_64-macos`)

All macOS packaging, signing, and notarization runs in pure Rust on Windows, Linux, or macOS with **zero Xcode required on the developer's host** (see [`sdk-and-crate-map.md`](sdk-and-crate-map.md)):
- **`.dmg` (Apple Disk Image):** Pure-Rust read/write synthesis via vendored [`apple-dmg`](https://crates.io/crates/apple-dmg/dependencies) with `.app` bundle layout and Applications symlink.
- **`.pkg` (XAR Installer Archive):** Synthesizes `Distribution` XML, `PackageInfo`, `Bom`, and deterministic `Payload` (`cpio` + `gzip`/`zstd`) by extending vendored [`apple-flat-package`](https://docs.rs/apple-flat-package) / [`isideload-apple-flat-package`](https://crates.io/crates/isideload-apple-flat-package) and [`apple-bom`](https://lib.rs/crates/apple-bom), supporting universal (`arm64` + `x86_64`) binaries and `launchd` plists.
- **Homebrew Cask Definition:** Auto-generates a `.rb` Cask formula with SHA-256 and uninstall stanza.

## 4. Linux Targets (`Belt_x86_64-linux` / `Belt_arm64-linux`)

- **`.deb` (Debian/Ubuntu):** Pure-Rust `ar` archive (`debian-binary`, `control.tar.zst`, `data.tar.zst`) built on [`cargo-deb`](https://github.com/kornelski/cargo-deb) patterns.
- **`.rpm` (Fedora/RHEL/SUSE):** Pure-Rust RPM v4 lead/signature/header synthesis with `cpio` + `zstd` payload.
- **`AppImage` & `Flatpak`:** Pure-Rust SquashFS image generation via [`backhand`](https://crates.io/crates/backhand) for AppImage runtime overlays, plus Flatpak manifest/bundle export.
- **Desktop & Service Integration:** Auto-generates freedesktop `.desktop` entries, icon hierarchy, and `systemd` unit files from `[package]` templates.

## 5. Built-In Signing, Local Company CA, MDM & SBOM

Every build pipeline supports enterprise trust artifacts via `packx build --sign` or `packx sign`:
- **Windows:** Authenticode via `signtool.exe`, Azure Trusted Signing, cross-OS [`osslsigncode`](https://github.com/mtrojnar/osslsigncode), or **free local company Root CA signing** (`packx sign init-ca` via [`rcgen`](https://github.com/rustls/rcgen); see [`security-and-sealing.md`](security-and-sealing.md)).
- **macOS:** Pure-Rust signing, Apple Notary API submission, and ticket stapling via vendored [`apple-codesign`](https://crates.io/crates/apple-codesign) / [`tauri-macos-sign`](https://crates.io/keywords/codesign) (requires an Apple Developer ID certificate for public Gatekeeper trust).
- **Linux:** Detached and inline GPG signing for `.deb` / `.rpm` repositories.
- **Supply-Chain Artifacts:** Auto-generates `.sha256` checksum files and an **SBOM** in **SPDX** or **CycloneDX** JSON format.
- **MDM Metadata:** Emits deployment descriptors for Microsoft Intune/SCCM (Windows), Jamf Pro/Kandji (macOS), and Ansible/Fleet (Linux).


## 6. Protected payload blocks and the embedded restricted installer helper

The custom EXE assembly path can carry a target-specific installer helper and protected operation-plan blocks inside the installer. The helper is a capability-limited part of the installer's own stub/container, not a separately installed PackageX Runtime, package manager or end-user PackageX.exe.

For Electron optimization, the SDK supplies:
- application-owned payload resources;
- exact Electron runtime archive identity and expected hashes;
- a logical runtime reference encoded into an opaque authenticated control block beside the application payload;
- named destination slot IDs and the layout identity used by the target writer.

The PE writer maps logical slot IDs to the final overlay payload index. The helper resolves those slots only through the authenticated layout. The reference never supplies arbitrary raw EXE offsets and no helper operation patches the running installer executable in place.

The build-time logical source descriptor can be JSON, but the assembled installer must not expose it as a loose readable JSON file. The protected control block is authenticated and bound to the final payload layout and publisher/install authorization identity. Opaque/default encryption protects casual inspection; signature verification, AEAD authentication, exact pinned artifact digests, OS-native signature checks and constrained write capabilities provide the integrity/authorization guarantees. Do not claim encryption alone prevents RCE or hides data from a determined local machine owner.

The helper may download the exact version-pinned Electron archive, verify expected size/full SHA-256 and applicable native publisher signatures, safely extract it, verify the extracted inventory, then write only to declared slots in the active same-volume staging tree. It rejects arbitrary URLs, paths, offsets, scripts, executable launch requests and writes outside the active transaction. It must not execute downloaded material as installation instructions.

For MSI, helper execution must remain inside an installer-compatible transaction/rollback design. For MSIX, macOS signed app bundles, drag-and-drop DMGs, AppImage and Flatpak, use the format's declared capability; do not mutate sealed/immutable package contents or assume a helper can run during ordinary install/launch. The SDK Electron capability matrix is authoritative.

## 7. Installation-scope policy integration

The Engine's resolved per-user/machine-wide scope is carried into the assembly request. The writer must report and preserve its native semantics rather than equating an install directory with scope.

- Custom EXE helper resolves destinations and permission requirements using the authenticated operation plan.
- MSI uses Windows Installer installation context; MSI authoring must support/test both contexts when advertised.
- MSIX reports per-user registration and enterprise all-user provisioning as distinct behaviors.
- macOS PKG declares supported installer domains; DMG copy operations are not represented as helper-driven installs.
- DEB/RPM use system package-manager semantics. AppImage and Flatpak declare their respective user/system behavior.

The [Installation Scope architecture](installation-scope.md) owns the normalized policy and capability table. The [Electron SDK architecture](https://github.com/Package-X/sdk/blob/main/plan/architecture/electron.md) owns app/runtime separation and format-specific restoration support.

