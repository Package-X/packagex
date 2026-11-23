# Auto-Updater, White-Label Launcher Shim & 24h Schedulers

PackageX ships an optional, 100% white-labeled auto-update system (`%LOCALAPPDATA%\<package_name>\updater\update.exe` on Windows, or native per-user app support paths on macOS/Linux) that adapts to **existing company or open-source JSON APIs** without forcing a custom update server.

## 1. Flexible JSON-Path + Regex + SemVer Checking

Instead of locking developers into a single rigid manifest schema, `[updater]` points at any existing JSON endpoint (GitHub Releases, S3 manifest, or internal REST API) and maps fields declaratively:

1. **JSON Path Extraction:** Extracts `version_path`, `priority_path`, `download_url_path`, and `sha256_path` cleanly (immune to JSON formatting/whitespace changes).
2. **Optional Regex Capture:** Strips version prefixes/suffixes (e.g., `^v?(\d+\.\d+(?:\.\d+)?)$`) before version comparison.
3. **Real SemVer Ordering:** Compares versions using [`semver`](https://github.com/dtolnay/semver) so `1.10.0 > 1.9.0` always evaluates accurately.
4. **Priority Policies:**
	- **`critical`:** Blocks app launch, applies the verified update immediately, then launches the new version.
	- **`normal`:** Displays a non-blocking update prompt when the app launches (skippable by the user).
	- **`background`:** Downloads and verifies silently in the background; swaps cleanly on next app exit.

## 2. Security Enforcement: Dev Warning vs. Production Hard-Stop

- **Production (`packx build`):** **Hard-stops** if the updater URL is not HTTPS, lacks a `sha256_path` (`"sha": "<SHA256>"`) mapping, or lacks a pinned **Ed25519 public key** / Local CA certificate thumbprint embedded into `update.exe` at build time.
- **Feed Authenticity, Expiry, Key Rotation & Anti-Downgrade:** Signed production update feeds MUST verify:
	- **Detached Ed25519 Signature + `key_id`:** Supports embedding a primary + next rotation public key (`key_id`) so keys rotate cleanly without stranding older installs.
	- **Freshness (`expires_at`):** UTC ISO-8601 expiry timestamp rejecting stale or frozen replayed manifests.
	- **Monotonic Anti-Downgrade:** `update.exe` refuses any version lower than the currently installed SemVer (blocking rollback-to-vulnerable-version attacks) unless a signed `allow_downgrade = true` emergency directive is present.
- **Dev / Sandbox (`packx test`):** Prints a clear warning (`"Updater URL is not secured — sign updates and include a sha field for production"`) and continues cleanly for local iteration.

## 3. Generic White-Label Launcher Shim & Custom Launcher Sandbox Probe

PackageX **never patches or modifies the user's compiled application binary** (which would break Authenticode/`codesign` signatures and trigger Windows Defender / Gatekeeper heuristics).

### Default Mode (generic applications)
- The generic launcher may place the untouched app binary under an app directory and provide a white-labeled entry point where the selected project policy requires it.
- The launcher may coordinate an update before starting the app, without patching the signed application executable.
- **Electron exception:** Electron optimization does not insert a PackageX launcher. The SDK restores the exact runtime files to the original distribution-relative paths, and the original Electron executable remains the application entry point. A developer's own updater may run separately according to the developer's update policy.

### Custom Launcher Mode (`launcher = "custom"`)
- Developers can disable the default shim and supply their own custom launcher (`launcher_bin = "./dist/my-launcher.exe"`).
- **Runtime Sandbox Probe (`packx test`):** Because static binary inspection cannot reliably prove dynamic launch paths across Rust/C++/Go/C#, `packx test` places tiny **mock spy binaries** in place of `update.exe` and the target app inside `Desktop/<package_name>/`, launches the custom launcher briefly, and verifies both spies were invoked—failing with an actionable error if the custom launcher isn't wired to the updater or app.

## 4. Editable Updater UI (`.pxs` / HTML)

Developers can customize or theme the updater UI using the exact same `.pxs` schema compiler (`[updater].ui = "./updater.pxs"`), supporting `UpdateAvailablePage`, `DownloadingUpdatePage`, and `CriticalUpdatePage` across WinLook `wl_ui`, AppKit, GTK/Qt, Tauri 2, and `ratatui` TUI.

## 5. Scope-Aware Background Update Schedulers

To check periodically (default example: every 24h), `px-runtime` registers a native scheduler under the selected installation scope. All registrations are tracked by the rollback journal and removed cleanly on uninstall.

- **Per-user:** Windows Task Scheduler task for the current account; macOS LaunchAgent in `~/Library/LaunchAgents`; Linux `systemd --user` timer in the user's configuration.
- **Machine-wide:** Windows machine task/service with protected task definition and updater path; macOS system LaunchDaemon in `/Library/LaunchDaemons` when the package and authorization model support it; Linux system `systemd` unit/timer under administrator-owned configuration.
- The machine-wide scheduler must execute only a machine-protected updater binary and load only authenticated update policy/configuration. It must not elevate a per-user updater or execute a user-writable replacement updater.
- Scheduler privileges and registration support are declared per target format. A format unable to register a scope-correct scheduler must report that limitation; it must not silently switch to a per-user task.


## 6. Electron application updates: immutable generations

Electron updates are coordinated as complete application generations. Each generation is a matched pair of application payload and exact Electron runtime identity.

- The currently installed application continues running on its current Electron runtime.
- The updater reads the next release's authenticated application manifest and compares its exact Electron version, target, artifact identity and runtime digest with the installed generation.
- If the runtime identity is unchanged, no Electron runtime download is performed.
- If it differs, download and verify only the exact runtime required by the newest applicable release. Stage at most one pending runtime; do not retain a library of Electron versions.
- Runtime acquisition happens in the background while the current application remains untouched.
- When the application exits, stage/commit the new application payload and runtime together using the same-volume transaction and rollback journal.
- Launch the new application generation and run the configured health check. Keep the old generation and runtime until this succeeds.
- On failed download, digest/signature verification, extraction, commit or health check, discard the pending generation and preserve/restore the previous working generation.
- If a newer release supersedes the staged release, discard the obsolete pending application/runtime and prepare only the newest applicable release.

The installed current runtime is an application dependency, not a cache. The single pending runtime is temporary update staging. There is no global end-user Electron runtime cache, runtime sharing service or reference counter.

## 7. PackageX updater host and developer-controlled behavior

The default white-label updater host executable is supplied by PackageX as an installer component and is installed under the selected application scope. The developer owns the update feed, release policy, application manifest, update UI/configuration and the allowed updater behavior. A developer may provide a custom updater executable/implementation instead.

These are separate from the installer's embedded restricted helper (internally named per-target, e.g. script.exe):

| Component | Purpose | Trust boundary |
|---|---|---|
| Embedded installer helper | Performs installation-time operations authorized by the installer's protected operation plan, including exact Electron runtime restoration when supported | Narrow fixed capabilities; no general command execution or arbitrary writes |
| PackageX updater host | Performs developer-configured update checks, downloads and staged app/runtime update transactions | Verifies signed developer update metadata and pinned artifact identity; executes only the updater behavior authorized by its installed policy |
| Developer-supplied custom updater | Optional replacement or extension when the developer needs full control | Developer-owned executable/code and signing identity; its privileges and update trust must be explicit |

An updater manifest is release-specific. For an Electron application it carries the exact Electron version/target and exact artifact reference needed by that application release. A floating latest Electron URL must never be resolved at install/update time.

A generic SHA-256 field obtained from the same untrusted feed as the artifact is not a trust anchor. Production feeds require an authenticated signature from the configured developer key, freshness/anti-replay metadata, anti-downgrade policy and a digest pinned by the authenticated release description. The updater may support key rotation through explicit key IDs and a signed transition policy.

## 8. Scope-aware updater execution

Updater installation paths, scheduled tasks and write permissions follow the original application install scope:

- Per-user updater: lives in the user's protected application-support location and writes only to that user's application generation.
- Machine-wide updater: lives in a machine-protected support location. Its scheduler/service and update configuration must not be writable by standard users. It must authenticate all update instructions before requesting elevation or writing machine paths.
- A machine-wide updater must not run developer-supplied executable logic or load a replacement updater from a user-writable directory with elevated privileges.
- Changing installation scope is an explicit migration; an ordinary update does not change scope.

Per-user checks can use the account's native scheduler. Machine-wide scheduling needs a protected machine task/service/timer. The two modes have different ownership, privileges and uninstall cleanup; neither is silently substituted for the other.

## 9. Electron update acceptance criteria

- [ ] Existing Electron app executable remains the launch entry; no PackageX Electron launcher is inserted.
- [ ] A release that keeps the same runtime identity does not download Electron again.
- [ ] At most one newer runtime is staged for the latest applicable release.
- [ ] The current app/runtime continues running during background preparation.
- [ ] App payload and runtime switch as one rollback-capable generation after the app exits.
- [ ] Previous generation remains available until new-generation health check passes.
- [ ] Failed or superseded pending updates do not accumulate old Electron runtimes.
- [ ] No global end-user Electron cache/store/daemon exists.
- [ ] The embedded installer helper and updater host remain separate capabilities.
- [ ] Per-user and machine-wide updater paths, schedulers and privileges are protected according to install scope.
