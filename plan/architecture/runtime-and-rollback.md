# Installer Runtime, Rollback Engine, Sandbox & Delta Updater (`px-runtime`)

## 1. Same-Volume Transactional Staging & Atomic Rollback Journal

Custom installers must never leave behind half-extracted files, orphaned services, or corrupted `$PATH` entries. `px-runtime` uses a **same-volume two-phase transactional staging model** across Windows, macOS, and Linux:

1. **Same-Volume Stage & Verify (Prevents `EXDEV`):** Extract and decompress all payload files into a hidden sibling staging folder on the **exact same filesystem/volume as `$instdir$`** (e.g., `<parent_of_instdir>/.stage-<app_id>-<session_id>/`—never cross-volume from `$temp$`), and verify every file against its compile-time SHA-256 hash.
2. **Single Atomic Directory Swap + Crash-Recovery Journal:** Rather than renaming files individually (which is not atomic across power loss), `px-runtime` writes a durable `COMMITTING` checkpoint to `.journal-<app_id>-<session_id>.log` (`sync_all`), swaps the **entire staged directory** (`.stage-<app_id>-<session_id>/` -> `$instdir$` or atomic `current/` junction/symlink flip, moving any prior install to `.backup-<app_id>-<id>/`), then applies OS mutations (`$PATH`, services, shortcuts) and marks `COMMITTED`. If power loss or process termination interrupts mid-commit, the next launch detects the incomplete journal and cleanly finishes rollback or recovery.

| Action Type | Pre-Mutation Snapshot Saved | Rollback Operation (Reverse Order) |
| :--- | :--- | :--- |
| `CreateDir` | Directory path | Remove directory if empty |
| `WriteNewFile` | Target file path | Delete newly created file |
| `OverwriteFile` | Backup copy in `.backup-<app_id>-<id>/` | Restore original file back to target path |
| `SetRegistryOrPlist` | Previous value + type (or `None`) | Restore previous value or delete key |
| `UpdatePathEnv` | Exact sentinel block (`# >>> <app_id> >>>`) or HKCU/HKLM entry | Surgically remove only `<app_id>` block / restore `PATH` + broadcast `WM_SETTINGCHANGE` |
| `RegisterService` | Previous service state (Windows Service / `launchd` / `systemd`) | Stop & remove newly registered service |
| `CreateShortcut` | `.lnk` / `.desktop` / `.app` alias path | Delete shortcut |

If the user clicks **Cancel** or any step fails, UI transitions to `rolling_back`, `px-runtime` replays the journal backward to index `0`, cleans staging/backup folders, transitions to `error_rolled_back`, and exits with code `2`.

## 2. White-Label Zero-Trace Uninstalls (Self-Delete Safe)

To keep end-user machines 100% first-party (zero `PackageX` folders) **and** avoid Windows' running-`.exe` file-lock trap:

- **Uninstaller Placement Outside `$instdir$`:**
	- **Windows (`scope = "user"`):** `%LOCALAPPDATA%\<package_name>\uninstaller\<app_id>.exe`
	- **Windows (`scope = "machine"`):** `%ProgramData%\<package_name>\uninstaller\<app_id>.exe` (so any admin account on the PC can uninstall via Add/Remove Programs).
	- **macOS / Linux:** Stored under the app's native receipt/support directory outside `$instdir$`.
- **Smart Hash-Verified Removal:** Re-hashes files in `$instdir$` against the compile-time SHA-256 manifest:
	- **Untouched files** (hash matches) -> deleted cleanly with zero residue.
	- **User-modified files** (e.g., edited `config.toml`) -> preserved or prompted (`--purge` removes all).
- **Clean Self-Cleanup on Windows:** After `$instdir$` is completely deleted (unlocked), the uninstaller removes the empty `uninstaller\` directory via a temp-copied cleanup helper or Win32 `MoveFileExW(..., MOVEFILE_DELAY_UNTIL_REBOOT)`.
- **Shell `$PATH` Sentinel Removal (macOS/Linux):** Removes *only* the lines between `# >>> <app_id> >>>` and `# <<< <app_id> <<<` inside `~/.zshrc`, `~/.bashrc`, or `~/.config/fish/config.fish` so user edits made after installation are never overwritten.

## 3. Developer Test Sandbox Mode (`packx test`)

Testing installers locally should never pollute a developer's real `AppData`, Registry, or `$PATH`. Running `packx test app.px` (strictly a dev-only profile—never baked into production builds):
- Redirects `$instdir$` to **`Desktop/<package_name>/install/`**.
- Redirects the uninstaller to **`Desktop/<package_name>/uninstaller/`**.
- **Virtualizes OS side effects:** Intercepts `$PATH`, Windows Registry, macOS plist, and service registrations and writes them to **`Desktop/<package_name>/simulated-mutations.json`** instead of mutating the real host OS.
- Deleting the single `Desktop/<package_name>/` folder leaves the dev machine 100% clean.

## 4. Live Target-Machine `--dry-run` Mode

While `-packagex-detail` inspects the **package itself**, `--dry-run` evaluates the **target machine** without writing a byte or triggering UAC:
- **Disk Space Check:** Compares free space on the target volume against `estimated_size_bytes` + staging headroom (exit code `12` if insufficient).
- **Lock & Port Check:** Detects running processes locking target binaries or occupied network ports required by `system-service` packages (exit code `11`).
- **Privilege Check:** Confirms whether `elevate` and resolved paths require elevation (exit code `10` if mismatched).
- **Dependency Check:** Runs read-only `check_cmd` probes to report satisfied vs. missing prerequisites.

## 5. Dependency Check & Remote Bootstrapper Engine

1. **Version Detection:** Runs `check_cmd` hidden (no console flash), parses SemVer via [`semver`](https://github.com/dtolnay/semver), and compares against `min_version`.
2. **External Side-Effect Boundary (`on_missing` & `--dry-run`):**
	- Third-party dependency installers (`python.exe`, `node.msi`) run outside PackageX's transactional sandbox and **cannot be rolled back** by `px-runtime`. `-packagex-detail` and `--dry-run` explicitly tag every `on_missing` step as **`[External / Non-Rollbackable]`**, and `--dry-run` permits only read-only `check_cmd` probes.
3. **Secure Download & Execution (`on_missing`):**
	- Enforces HTTPS unless `allow_insecure_http = true`.
	- Streams downloads via [`ureq`](https://github.com/algesten/ureq) with live UI progress and validates mandatory 64-char `sha256` pins when `reproducible = true`.
	- Dispatches OS-scoped bootstrappers, refreshes process `$PATH`, and re-verifies `check_cmd`.

## 6. Cross-OS Auto-Updater with Uncompressed-Stream `zstd` Delta Patching

- **Unified Update Feed:** `packx build` emits a signed `update-feed.json` listing latest versions, OS/arch targets, full installers, and delta patches.
- **Uncompressed-Stream Binary Diffs:** Because diffing two *already-compressed* installers fails (a 1-byte change shifts the compressed stream), PackageX computes `zstd --patch-from` deltas against the **uncompressed deterministic payload stream** (or per-file) between release `N-1` and `N`, then compresses the resulting patch—turning a 150 MB release into a ~2 MB delta download.
- **Per-User & Service-Assisted Updates:** Supports silent background updates without admin prompts.


## 7. Electron immutable application/runtime generations

An optimized Electron install is one application generation containing the app payload and the exact runtime that app was built and signed against. The runtime is app-local and scope-local; it is not registered in a shared machine runtime store.

### First installation

1. Authenticate the installer and its protected Electron control block.
2. Verify target OS, architecture, exact Electron version/build, pinned URL, expected size, full artifact SHA-256 and runtime inventory identity.
3. Download to a bounded same-volume staging area; do not execute or launch the artifact.
4. Verify the complete archive digest and applicable OS publisher signatures.
5. Safely extract with path traversal, unsafe-link, collision, file-count and expanded-size checks.
6. Restore runtime files to their original relative paths in the application staging generation.
7. Verify the reconstructed app/runtime inventory and applicable app signatures.
8. Commit the full generation through the normal transaction journal. Launch is allowed only after required verification succeeds.

If any step fails, discard/quarantine staging and leave an existing installation untouched.

### Background update preparation

The updater compares the current installed generation with the newest authenticated application release:

- If the exact Electron runtime identity is unchanged, update only the application payload.
- If Electron changes, download and verify only the exact runtime required by that release.
- Keep the currently installed app and runtime operational while the new payload/runtime are downloaded, verified and staged.
- Retain at most one pending update generation. If a newer release supersedes it, discard the obsolete pending payload/runtime and stage only the newest applicable release.
- Never replace runtime files currently loaded by a running app.

### Commit and rollback

Once the application exits:

1. Acquire the application update lock and re-check the current generation identity.
2. Complete any remaining pre-commit checks.
3. Use a same-volume transaction to switch the application payload and runtime together. The journal must identify old/new generation IDs and exact runtime identities.
4. Start the new generation and run the configured health check.
5. Mark the new generation healthy only after the check succeeds.
6. Remove old generation files and rollback data only after the health-check/retention policy allows cleanup.

On failure, restore the previous complete generation. Do not combine a new application payload with an old incompatible Electron runtime or vice versa.

The installed runtime is a dependency of the current application, not a cache. The one pending runtime is temporary update staging. PackageX does not retain every previous Electron release or create a global user/machine Electron cache.

## 8. Scope-aware transaction roots

The Engine resolves the selected install scope before the transaction begins. Every path is passed as a typed logical destination rather than re-derived from an untrusted script or string.

- Per-user app, updater, staging, journal and rollback data remain inside user-owned application/support locations.
- Machine-wide app, updater, staging, journal and rollback data use protected machine-owned locations.
- A machine-wide updater must not load its update policy, executable, plugins or scripts from user-writable directories before elevated writes.
- User documents, preferences and user-specific mutable data are not moved into a machine-owned directory merely because the executable is machine-wide.
- A scope change is an explicit migration with a separate rollback plan, not an ordinary version update.

## 9. Electron transaction acceptance

- [ ] Running Electron files are never overwritten while in use.
- [ ] The currently installed app/runtime remains operational while the next generation is prepared.
- [ ] At most one pending Electron runtime is retained.
- [ ] A same-runtime update does not download Electron again.
- [ ] App payload and runtime are committed as one generation.
- [ ] The old generation is retained until the new one passes health checks.
- [ ] Failed verification or health checks restore the previous generation.
- [ ] Per-user and machine-wide transaction roots have correct ownership and permissions.
- [ ] No global end-user Electron runtime store or reference counter exists.
