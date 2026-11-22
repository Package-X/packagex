# `packx toolbelt`, Blocking Lockfile & CAS SDK Pool

## 1. Motivation: Reproducible Cross-OS Builds Without Version Drift

If a developer on Linux has `packx 2.3` installed while a teammate on Windows has `packx 2.4`, building the same `.px` project without engine isolation breaks **reproducible builds** and causes subtle SDK mismatches. At the same time, shipping every target OS packager and pre-compiled stub inside the `packx` shim binary would bloat the CLI unnecessarily.

PackageX solves both problems by separating the **thin `packx` shim** from **versioned toolbelt engines**, **modular target sub-belts**, a **kernel-managed blocking lockfile**, and a **Content-Addressed Storage (CAS) SDK pool**.

## 2. Directory Layout

```text
~/.packx/ (or packageX/packX/)
├── bin/
│   └── packx                                  # Thin shim router binary (permanent on $PATH)
├── toolbelt.lock                              # Kernel-managed blocking lockfile (contains active PID + command)
├── sdk_index.json                             # Central reference-count & ownership index (updated atomically)
├── toolbelt/
│   ├── 2.3/
│   │   ├── packx-core                         # Self-contained engine binary for v2.3
│   │   ├── manifest.json                      # Pinned list of 64-char SHA-256 SDK modules
│   │   └── sub-belts/
│   │       └── Belt_x86_64-linux.json         # Active target sub-belt descriptor
│   └── 2.4/
│       ├── packx-core                         # Self-contained engine binary for v2.4
│       ├── manifest.json                      # Pinned list of 64-char SHA-256 SDK modules
│       └── sub-belts/
│           ├── Belt_x86_64-windows.json       # Active target sub-belt descriptor
│           └── Belt_arm64-macos.json          # Pre-compiled macOS stubs (no Xcode needed on host)
└── sdk/
    ├── x86_64-windows/
    │   ├── a1b2c3d4e5f60718293a4b5c6d7e8f90_core/     # Shared read-only module (used by 2.3 & 2.4)
    │   ├── asd249dj02kd201jdf93h0123456789a_api/      # v3.4 API module (owned by 2.3)
    │   └── dha92hjfo24jgo28hf48hofh8a1b2c3d_api/      # Renewed v4.0 API module (owned by 2.4)
    └── x86_64-linux/
        └── 9f8e7d6c5b4a39281726354453627180_sysroot/
```

## 3. Concurrency Safety: Kernel-Managed Blocking Lockfile (No Daemon Required)

If two terminals or parallel CI jobs run a direct `packx toolbelt install <package-id>` operation, a project-aware `packx toolbelt install --required`, or `packx build` simultaneously, unprotected writes would corrupt `sdk_index.json` reference counts or read half-extracted SDK directories.

Instead of running a persistent background daemon (which consumes idle RAM and risks stale sockets on crash), `px-toolbelt` uses a **kernel-managed advisory file lock** (`LockFileEx` on Windows, `flock` on Unix) on `~/.packx/toolbelt.lock`:

1. **Priority Queue & Live Wait UX:** The first process acquires an exclusive lock and writes its PID and active operation (`PID 14820 | belt add 2.4/x86_64-windows`) into `toolbelt.lock`. Any second process attempting a mutating toolbelt operation sees the active lock, prints:
	```text
	Waiting: Another toolbelt operation is in progress (PID 14820: belt add 2.4/x86_64-windows). Waiting until it finishes...
	```
	and blocks cleanly until the first process finishes, then re-checks `sdk_index.json` (often discovering the module is now already installed!).
2. **Crash-Proof Auto-Release:** If the first terminal crashes, is killed (`SIGKILL`), or gets `Ctrl+C`'d mid-download, the OS kernel **immediately releases the file lock**—zero zombie daemons and zero manual lockfile cleanup.
3. **CI Timeout Guard:** `--lock-timeout=<seconds>` (default `300` in CI) lets automated runners fail cleanly with exit code `16` instead of hanging indefinitely.
4. **Atomic Extraction & Index Swaps:**
	- New CAS modules always extract into `<32-char-hash>_<component>.tmp.<pid>` and are moved into their final read-only directory via atomic `rename()`.
	- `sdk_index.json` is never edited in place—updates write to `sdk_index.json.tmp.<pid>`, flush to disk (`sync_all`), and atomically rename over `sdk_index.json`.

## 4. Version Routing & Side-by-Side Coexistence

1. Every `.px` manifest may pin an engine version: `engine = "2.4"`.
2. When a user runs `packx build`, the `packx` shim checks `~/.packx/toolbelt/2.4/packx-core`.
3. **Online Auto-Fetch:** If `2.4` is not installed and online mode is active, `packx` acquires `toolbelt.lock`, fetches the `2.4` engine + required target sub-belt via [`ureq`](https://github.com/algesten/ureq), verifies full 64-char SHA-256 digests, installs it side-by-side, and executes the build.
4. **Independent Uninstall:** Users can run `2.3` and `2.4` side-by-side and uninstall either via `packx toolbelt remove 2.3` without touching the `packx` shim or other versions.

## 5. Strict `--offline` / `--locked` Mode & `.pxbelt` Bundles

Enterprise CI runners and air-gapped build servers must never silently access the internet.

### Actionable Offline Diagnostics
Running `packx build msi --silent --offline` when the required engine or sub-belt is missing fails loudly with exit code `13`:

```text
Error: Requires packx 2.4 (Belt_x86_64-windows), found 2.3.
Reason: Offline mode enabled — cannot fetch from network.
Fix: From the project directory, run `packx toolbelt install --required` online, or import the project's required `.pxbelt` bundle.
```

### Portable `.pxbelt` Export / Import
- **On an online machine:** `packx toolbelt export --required 2.4/x86_64-windows -o win-2.4.pxbelt`
- **On an air-gapped server:** `packx toolbelt import win-2.4.pxbelt`

## 6. Component-Level CAS SDK Pool (32-Char SHA-256 + Delta Installs)

To prevent version collisions while keeping target sub-belts tiny (**~3–8 MB** `zstd` pure-Rust packager + pre-built stub archives published from `package-x/sdk` GitHub Releases, with zero 400+ MB C/C++ sysroots):
- **Folder Name:** `<32-char-sha256-prefix>_<component>` (128 bits of entropy—eliminates collisions while halving path length to avoid Windows' 260-character `MAX_PATH` limit).
- **Manifest Verification:** Each toolbelt's `manifest.json` stores the **full 64-character SHA-256** digest for every component; integrity checks always verify all 64 characters.
- **Delta Installs:** Adding `2.4` on a machine with `2.3` skips unchanged hashes (appending `"2.4"` to `owners` and flipping `mode` to `"shared"`) and downloads only changed modules.

```json
{
	"x86_64-windows/a1b2c3d4e5f60718293a4b5c6d7e8f90_core": {
		"sha256": "a1b2c3d4e5f60718293a4b5c6d7e8f9011223344556677889900aabbccddeeff",
		"mode": "shared",
		"owners": ["2.3", "2.4"]
	},
	"x86_64-windows/dha92hjfo24jgo28hf48hofh8a1b2c3d_api": {
		"sha256": "dha92hjfo24jgo28hf48hofh8a1b2c3d99887766554433221100ffeeddccbbaa",
		"mode": "owner",
		"owners": ["2.4"]
	}
}
```

### Immutability & Safe Uninstall Rules
- **Read-Only Locking:** Every `shared` module directory is locked read-only immediately after atomic rename. Version-specific caches live under `~/.packx/toolbelt/<version>/`.
- **Safe Reference-Counted Uninstall (`packx toolbelt remove 2.4`):** Deletes `owner`-only modules immediately; decrements `owners` on `shared` modules and flips back to `owner` when one remains.
- **Garbage Collection:** `packx toolbelt prune` sweeps orphaned modules and cleans leftover `.tmp.<pid>` folders from crashed runs.


## 7. Canonical CLI spelling and detailed resolver

The public package-management command is packx toolbelt. Older packx belt commands in early planning examples are legacy spelling; if retained for compatibility, they must be a thin alias to the same implementation and state store, never a second manager.

The user-facing installation forms are:

~~~powershell
packx toolbelt install <package-id[@identity]>
packx toolbelt install <package-id> --nightly
packx toolbelt install <package-id> --nightly=a1b2c3d4
packx toolbelt install --required
packx toolbelt install --required --nightly
packx toolbelt install --required --nightly=a1b2c3d4
~~~

A package can be installed directly by ID/identity. `--required` is optional: it asks the Engine for the selected project's machine-readable requirement graph, then installs only the minimum complete compatible package/dependency closure. It does not download every published OS, architecture, format or optional component. PackX does not parse full .px project semantics a second time.

Stable is the default channel. Nightly must be selected explicitly; `--nightly` without a commit prefix means latest compatible Nightly, while `--nightly=<8hex>` selects the exact eight-character GitHub commit prefix. The full commit is verified and stored in the package manifest/Toolchain lock. Exact project pins take precedence unless deliberately changed. There is no Beta channel.

Authoritative implementation detail:
- [PackX dependency and resolution playbook](https://github.com/Package-X/packx/blob/main/plan/implementation/dependency-and-resolution-playbook.md)
- [PackX package-store state and recovery](https://github.com/Package-X/packx/blob/main/plan/implementation/package-store-state-and-recovery.md)
