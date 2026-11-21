# Upstream Implementation Map and Repository Integration Contract

> Research snapshot: 21 November 2026. Upstream versions and APIs are not pins for a future Cargo.lock. Re-check tags, APIs, licenses, MSRV, advisories and maintenance before implementation.

## 1. Purpose

PackageX should reuse mature open-source format implementations rather than rebuilding every installer format inside the Engine. This document assigns responsibilities across the three repositories and links to detailed implementation playbooks.

- **Engine (this repository):** project semantics, normalized build planning, resource inventory, feature lowering and orchestration.
- **SDK:** format-specific writers/readers/validators, stubs, Electron analysis, payload transforms and build reports.
- **PackX:** permanent CLI shim, Engine/SDK package installation, signed catalog resolution, CAS, Toolchain locks and offline bundles.
- **PackageX Runtime:** end-user installer execution and application runtime lifecycle.

The Engine owns what a project means. The SDK owns how the selected target artifact is created or edited. PackX owns how the correct SDK package becomes available.

## 2. Three-repository ownership

| Repository | Owns | Explicitly does not own |
|---|---|---|
| [packagex](https://github.com/Package-X/packagex) | .px parser/linter, project semantics, resolved build IR, resource inventory, feature lowering, build graph, orchestration, runtime and shared security policy | Global SDK package store or every native format writer |
| [sdk](https://github.com/Package-X/sdk) | Target SDK packages, format writers/readers/validators, transactional edit adapters, Electron analysis, payload optimization and build-side reports | .px interpretation, permanent package manager or end-user installer execution |
| [packx](https://github.com/Package-X/packx) | Permanent shim/CLI, Engine routing, SDK acquisition, signed catalog resolution, compatibility, Toolbelt/CAS, project Toolchain locks and offline bundles | .px parsing policy, format-specific writer internals or app runtime store |

Detailed contracts:
- [SDK Build Contract](https://github.com/Package-X/sdk/blob/main/plan/architecture/build-contract.md)
- [SDK upstream integration map](https://github.com/Package-X/sdk/blob/main/plan/implementation/upstream-integration-map.md)
- [PackX required resolution](https://github.com/Package-X/packx/blob/main/plan/architecture/required-resolution.md)
- [PackX dependency and resolution playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md)

## 3. Engine responsibilities before invoking the SDK

The Engine:
1. Parses .px and resolves defaults, target scopes and feature flags.
2. Validates cross-platform semantics and identifies requested behavior that cannot be represented by the selected native installer.
3. Builds a normalized resource inventory with logical paths, source sizes, hashes and file roles.
4. Resolves requested output formats and required SDK capabilities for every target.
5. Creates a deterministic build graph.
6. Asks PackX/Toolchain resolution for installed, verified SDK package roots.
7. Queries the SDK capability handshake before expensive work.
8. Produces a versioned BuildRequest and controlled resource provider.
9. Receives a structured BuildResult and validates it against project policy.
10. Produces the final build report and provenance.

The Engine must not parse SDK archives or download SDK packages directly, invoke arbitrary PATH tools, treat a version number as proof of capability, or reimplement native MSI/XAR/BOM/OSTree formats.

## 4. Build graph and SDK execution

### 4.1 Normalized Engine input

The SDK request is derived from the resolved build plan, not raw .px text.

Core logical objects:
- **ResolvedProject:** product identity, Engine version, project version and selected target set.
- **ResourceInventory:** normalized logical paths, file roles, sizes, digests and controlled resource IDs.
- **BuildPolicy:** reproducibility, offline mode, signing policy, verification requirements and resource limits.
- **TargetOperation:** OS/architecture/format plus create/inspect/modify/transform/sign/verify operations.
- **SdkSelection:** exact package ID/version/channel/source commit/digest and SDK API identity.
- **OutputPlan:** Engine-owned staging destination and expected artifact roles.

These are conceptual types. The precise Rust structs and serialized schema belong to the SDK Build Contract and must be frozen before implementation.

### 4.2 Capability handshake

For each target/operation, the Engine queries:
- package identity and SDK API version;
- supported Engine range;
- target and format capabilities;
- writer operation matrix;
- host restrictions;
- required optional components;
- limitations and experimental status.

The Engine fails before building if the installed SDK does not declare the required operation. An MSI writer that creates new packages but cannot yet edit third-party MSIs must not be treated as a general-purpose MSI editor.

### 4.3 Two-way resource/data flow

The Engine sends:
- product/project metadata required by the target;
- target and output-format requirements;
- canonical resource inventory;
- controlled read access to resource IDs;
- writer operation and typed format options;
- integrity/reproducibility/signing policy references.

The SDK returns:
- primary and auxiliary artifacts;
- full artifact hashes and sizes;
- changed-object inventory for edits;
- format verification results;
- warnings/errors with stable codes;
- compression/deduplication/build statistics;
- runtime dependency descriptors such as Electron runtime requirements;
- provenance and component identity.

The resource provider streams data rather than serializing every file into one huge request. Absolute host paths and arbitrary filesystem access are not part of the public contract.

### 4.4 Transport remains replaceable

The contract schema must not depend on whether the SDK is:
- an in-process Rust library;
- a native library behind a small C ABI and Rust wrapper;
- an isolated worker executable using structured request/result IPC;
- a combination where portable writers run in-process and native/tool-based workers run out-of-process.

Do not freeze transport before performance, crash-isolation, dynamic-linking and platform-loader experiments. Keep request/result semantics stable.

## 5. Concrete upstream integration direction

The authoritative capability recipes live in the SDK repository's upstream integration map. This summary keeps the central plan aligned with it.

| Format/component | Upstream foundation | PackageX responsibility and restriction |
|---|---|---|
| MSI | Rust msi crate plus cfb and cab; Windows Installer API for native transaction/transform paths | SDK wraps row/stream CRUD in staged editing, reopens/verifies and re-signs. No promise of physical byte-range preservation or arbitrary unknown-table round-trip before conformance tests. |
| MSIX | Microsoft MSIX Packaging SDK | SDK builds its cross-platform native library with packing features enabled, wraps it with a small C ABI/Rust adapter or worker, and uses Windows CI for SignTool/deployment validation. |
| PE installer | PackageX px-stub/overlay plus PE readers such as goblin | SDK owns overlay writing. Preserve PxFooter/Authenticode certificate-table invariants on signed binaries. |
| macOS .app | apple-bundles, plist and filesystem tooling | SDK assembles bundle structure; this alone is not code signing or Gatekeeper validation. |
| macOS .pkg | apple-flat-package, apple-xar, cpio-archive, apple-bom | Known BOM writer gap must be proven. If incomplete, implement only the bounded missing BOM writer or use pkgbuild/productbuild on a macOS runner as fallback. |
| macOS .dmg | apple-dmg | SDK adapter constructs the image; macOS CI validates with hdiutil, mount/contents tests and required signing. |
| Apple signing | apple-codesign | SDK performs supported signing/notarization/stapling; native codesign/spctl acceptance checks remain mandatory. Preserve MPL-2.0 obligations. |
| Debian .deb | debian-packaging; cargo-deb as a reference/optional Cargo-app path | SDK creates control/data archives and the Debian archive; test with native dpkg tools. Do not assume arbitrary .s.px can safely become maintainer scripts. |
| RPM | rpm crate | SDK creates package headers/payload; native rpm CLI validates and installs. The crate is not a full rpmbuild or RPM database replacement. |
| AppImage | appimagetool ecosystem, uruntime, mkdwarfs; backhand for SquashFS-specific work | Use actual AppImage runtime/filesystem path. Do not treat backhand SquashFS as a drop-in for a DWARFS build. |
| Flatpak | Official flatpak-builder, flatpak, OSTree utilities | SDK generates a pinned manifest and invokes an isolated Linux builder; do not implement OSTree repository/sandbox semantics from scratch. |
| Electron | Official Electron layout, @electron/get, @electron/asar, Electron Fuses | SDK separates standard app payload from exact runtime descriptor; never binary-surgery an arbitrary electron.exe. |
| PDC | fastcdc, zstd, approved digest crate | SDK chunks/compresses unique payload bytes; Engine retains canonical uncompressed inventory and native writers retain native semantics. |

### 5.1 License and commercial-dependency boundary

- Check each exact crate and transitive dependency; do not assume a monorepo has one license.
- apple-codesign and debian-packaging use MPL-2.0; preserve notices and keep modifications to MPL-covered files open as required.
- osslsigncode is GPL-3.0; use only as an optional external tool after a distribution review, not as an unnoticed linked dependency.
- WiX v6/v7 has Open Source Maintenance Fee terms that must be evaluated before making it a mandatory commercial build dependency.
- Microsoft MSIX Packaging SDK is MIT-licensed.
- Prefer native signing/validation tools on official host runners when those are the authoritative acceptance mechanism.

## 6. Build-host and target separation

Build host and output target are independent axes.

For example:
- a Windows host may use portable Rust writers to create macOS/Linux package files;
- a macOS host may validate Apple bundles/packages and perform Apple signing/notarization;
- a Linux host may build Flatpak/AppImage formats and use portable SDK components for other formats.

A target package's directory slug identifies its compiled host artifacts, not every format it can emit. The capability handshake states which output target/operations each host package supports.

Do not claim every format can be signed/notarized from every OS simply because its archive writer is portable.

## 7. Toolbelt and Toolchain boundary

The canonical package-management command family is packx toolbelt. Toolbelt is the machine's installed package pool; Toolchain is the exact compatible Engine/SDK selection for a project/build.

The Engine asks for verified SDK package roots but does not own the global download cache. PackX:
- verifies signed compatibility/catalog metadata;
- installs only the Engine-requested SDK package/dependency closure;
- uses immutable CAS objects;
- pins exact Stable/Nightly identities and full hashes;
- preserves multiple Toolchains;
- exports/imports exact .pxbelt offline closures.

Older packx belt examples in central specs are legacy spelling and must be reconciled with the canonical Toolbelt command before CLI implementation. If both spellings remain, they map to one implementation and store.

Detailed PackX plans:
- [Rust dependency and resolution playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md)
- [Package-store state and recovery](https://github.com/Package-X/packx/blob/main/plan/implementation/package-store-state-and-recovery.md)
- [Toolbelt CLI specification](https://github.com/Package-X/packx/blob/main/plan/specs/toolbelt-cli.md)

## 8. Implementation proof-of-concept order

Do not attempt every format simultaneously. Prove the Engine/SDK/PackX integration vertically:

1. **Contract skeleton:** Engine emits machine-readable requirements; PackX resolves one Windows SDK package; SDK reports capability handshake.
2. **MSI vertical slice:** create a basic MSI with cabinet, inspect it, modify one known property in staged copy, reopen/verify, install/uninstall in a Windows VM.
3. **MSIX vertical slice:** compile Microsoft SDK with packing enabled, package app, sign and deploy on Windows CI.
4. **macOS .pkg gap proof:** generate component package using Apple Rust crates, prove BOM correctness, validate/install on macOS; use native tool fallback until portable BOM passes.
5. **macOS .dmg/signing:** build image, sign/notarize/staple and run native macOS acceptance tools.
6. **Linux package slice:** create DEB and RPM from one canonical resource inventory, verify with native tools and disposable distro containers.
7. **AppImage/Flatpak:** generate and run on supported Linux baselines.
8. **Electron:** produce app-only payload/runtime descriptor, download/verify exact runtime, launch in bundled and managed modes.
9. **PDC:** benchmark whole-file dedup, FastCDC and Zstandard separately before combining.

Every proof-of-concept produces a fixture/corpus, exact upstream revision/feature flags, license report, native validation output, accepted capability statement and documented limitations.

## 9. Source-of-truth policy

- This document owns Engine-side boundaries and cross-repository integration.
- SDK API details and upstream limitations live in the SDK upstream integration map.
- Toolbelt resolver, package store and CLI details live in PackX playbooks.
- If an upstream project changes, update the relevant repository's evidence and integration plan before pinning/upgrading it in implementation.
