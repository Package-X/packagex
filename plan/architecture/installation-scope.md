# Installation Scope: Per-User and Machine-Wide

## 1. Goal and ownership

PackageX exposes one project-level installation-scope policy across Windows, macOS and Linux. The Engine owns the meaning of the policy and resolves it into an install plan. The SDK declares each output format's real scope capabilities and implements native format behavior. The embedded installer helper performs only the operations authorized by the sealed installer plan. PackX remains the developer-side Engine/SDK package manager and does not participate in end-user installation.

Installation scope answers **who receives the application**. Elevation answers **whether a privileged operation is allowed or required**. They are separate policy dimensions.

## 2. Project policy

Canonical .px fields:

~~~toml
[package]
scope = "user"                  # "user" | "machine" | "choice"
scope_default = "user"          # required only when scope = "choice"
elevate = "auto"                # true | false | "auto"
~~~

- **user:** install for the account running setup.
- **machine:** install into a machine-managed location for all local accounts where the target format supports that model.
- **choice:** let the installer offer only scope choices the selected output format can actually support. scope_default preselects a choice; it does not silently enforce it.
- Omitted scope defaults to user.
- A developer can lock the scope to user or machine.
- Scope cannot silently change during an update. User-to-machine or machine-to-user conversion is an explicit migration with separate preflight and consent.

The Engine resolves a normalized InstallScope and includes it in the install/update operation plan. The SDK capability handshake declares which scopes and operations its target writer supports.

## 3. Scope and elevation matrix

| Scope | elevate | Meaning |
|---|---|---|
| user | false | Normal per-user installation without elevation |
| user | auto | Remain per-user; diagnose any requested operation that requires machine privilege |
| user | true | Elevated execution may be used, but all files and registrations remain per-user |
| machine | auto | Request native administrator/root authorization when the selected operations require it |
| machine | true | Explicit machine-wide privileged installation |
| machine | false | Invalid if a requested machine-level write or registration requires privilege; fail at packx check |
| choice | auto | Resolve selected scope at setup, then calculate the required privilege |
| choice | true/false | Offer only choices consistent with the fixed privilege policy |

A denied or unavailable machine authorization must never silently fall back to per-user. Setup exits without partially installing the package.

The Windows PE manifest can remain asInvoker for inspection and dry-run behavior. Runtime elevation happens only after the user has selected/confirmed a scope and the Engine has established that a privileged operation is required.

## 4. Logical path resolution

The Engine resolves tokens after scope selection and passes structured logical destinations to the SDK/runtime. The SDK must not infer scope by inspecting a string path.

| Purpose | Per-user examples | Machine-wide examples |
|---|---|---|
| Windows application | %LOCALAPPDATA%\\Programs\\<app> | %ProgramFiles%\\<app> |
| Windows support/updater | %LOCALAPPDATA%\\<publisher>\\<app> | %ProgramData%\\<publisher>\\<app> |
| macOS application | ~/Applications/<app>.app | /Applications/<app>.app |
| macOS support | ~/Library/Application Support/<app> | /Library/Application Support/<app> |
| Linux application | ~/.local/opt/<app> or user-selected AppImage path | /opt/<app> or native package-manager path |
| Linux application data | $XDG_DATA_HOME/<app> | /var/lib/<app> |

These are defaults/examples, not universal substitutions. Native package conventions and developer overrides apply. Mutable preferences, caches, user documents and per-user state remain in each user's writable data directories even when application binaries are installed machine-wide, unless the developer explicitly defines shared data.

Machine-wide updater binaries, update configuration, protected references and staging areas must not be writable by ordinary users. An elevated updater must never load executable update logic or mutable authorization data from a user-writable path.

## 5. Native format capability model

The Engine must not assume that a common scope field makes every format support both scopes. SDK capability output should distinguish:

- Supported requested scope values (user, machine, choice).
- Native package staging scope.
- Per-user registration and machine provisioning.
- Whether the target format can execute an embedded helper.
- Required privilege for each operation.
- Scope of shortcuts, PATH, services, update schedulers and uninstall records.
- Whether Electron app-only runtime restoration is supported.

Initial capability expectations:

| Target | Per-user | Machine-wide | Notes |
|---|---|---|---|
| Windows custom EXE | Supported candidate | Supported candidate | PackageX helper can resolve user/common paths; machine scope needs admin authorization and protected updater/uninstaller data. |
| Windows MSI | Supported | Supported | Author and test Windows Installer installation contexts, including the supported ALLUSERS / MSIINSTALLPERUSER model. |
| Windows MSIX | Per-user registration | Enterprise provisioning | Staging and registration differ. All-user provisioning is a distinct administrator/IT operation, not equivalent to an MSI machine install. |
| macOS DMG | Supported | Supported with authorized copy to /Applications | Drag-copy to ~/Applications is per-user. Copying to /Applications needs authorization. Ordinary drag-and-drop does not execute a PackageX helper. |
| macOS PKG | Conditional | Supported | Must be authored for the requested install domain/choice. CurrentUserHomeDirectory and LocalSystem are not interchangeable. |
| Linux DEB | Not native | Supported | dpkg/APT package installation is system-managed; don't claim conventional per-user DEB installation. |
| Linux RPM | Not native | Supported | RPM/DNF/Zypper installation is system-managed; don't claim conventional per-user RPM installation. |
| Linux AppImage | Supported | Conditional | Normally user-owned. Common placement is possible but is not native package-manager registration. |
| Linux Flatpak | Supported | Supported | Use Flatpak's native --user / --system installation semantics. |

MSIX uses machine-managed package staging with per-user app registration; enterprise provisioning for all users is a distinct operation. macOS .app placement can be per-user or in /Applications, but a .app signature seals its bundle resources. Linux DEB/RPM operations are package-manager/root operations, whereas Flatpak provides explicit user/system installations and AppImage is normally a user-owned executable image.

When the requested scope cannot be represented, packx check fails the selected target before artifact emission unless the developer explicitly chooses another target/output policy. Never silently widen or narrow installation scope.

## 6. Scope-aware registrations and lifecycle

The Engine resolves the selected scope for:

- Application binary and resource destinations.
- Shortcuts, file associations and PATH registrations.
- Service/daemon installation.
- Updater binary, feed configuration and scheduler.
- Uninstaller and installation receipt.
- Rollback journal, staging and backup roots.

Per-user schedulers run under that account and modify only that account's protected installation. Machine-wide schedulers/services use a protected machine identity and machine-owned configuration. Machine-wide updates require authenticated update instructions and artifact verification before privileged writes.

Uninstall follows the recorded installation scope. An ordinary account cannot remove a machine-wide installation without the required authorization. Removing machine-wide binaries must not remove per-user documents or preferences.

## 7. Electron integration

Electron runtime files are part of the application's installed generation and follow the selected application scope. They are not copied into a global/shared Electron cache or PackageX runtime service.

- Per-user install: runtime and app are restored beneath the user's application root.
- Machine-wide install: runtime and app are restored beneath the protected machine application root.
- First install restores the exact version/target runtime declared by the installer's protected Electron reference.
- Updates keep the current app/runtime generation operational while at most one next-release runtime is staged.
- The updater commits the new app payload and runtime together after the app exits, health-checks the new generation, and retains the old generation for rollback until success.
- Format-specific restrictions remain authoritative. A helper cannot mutate a signed MSIX deployment or add files to an already sealed macOS .app bundle.

See the SDK's [Electron application support plan](https://github.com/Package-X/sdk/blob/main/plan/architecture/electron.md).

## 8. Acceptance criteria

- [ ] .px supports user, machine and capability-gated choice scope.
- [ ] Per-user is the default.
- [ ] Scope is independent from elevate and install_dir.
- [ ] Denied machine authorization never silently becomes per-user.
- [ ] Engine passes a resolved scope in every install/update plan.
- [ ] SDK reports native scope capability per target format.
- [ ] MSI is tested in both installation contexts.
- [ ] MSIX staging, per-user registration and machine provisioning are represented distinctly.
- [ ] macOS app locations and PKG installation domains are validated.
- [ ] DEB/RPM are not advertised as native per-user package installs.
- [ ] Flatpak user/system and AppImage user/common placement are distinguished.
- [ ] Schedulers, updater, uninstaller, PATH, services, receipts and data paths follow the selected scope.
- [ ] Electron runtime files remain app-local and scope-local; no global end-user Electron runtime store is introduced.
