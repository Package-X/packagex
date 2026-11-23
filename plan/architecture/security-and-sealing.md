# Security, Password Sealing, Local Company CA & White-Label Stealth (`px-crypto`)

PackageX supports both public software distribution and **high-security internal company installers** that remain unreadable garbage at rest and in transit without a valid password or recovery key.

## 1. Multi-Slot Envelope Encryption ("App -> Garbage -> App")

Never encrypt files one-by-one inside a readable ZIP/tar—that leaks folder trees, filenames, file counts, and sizes. Instead, `px-assembler` + `px-crypto` wrap the **entire payload, `.s.px` WASM script, and internal manifest** into one opaque encrypted blob:

1. **Single-Blob Pack First:** Deterministically pack and `zstd`-compress all payload files, `.s.px` bytecode, and the internal file manifest into one contiguous stream.
2. **Random Master Payload Key (DEK):** Generate a fresh 256-bit random Data Encryption Key (DEK) at build time.
3. **Formal Chunked AEAD Wire Format (`PKX_SEAL_V1` — `XChaCha20-Poly1305` Only):**
	- **Single Cipher Standard:** Standardize strictly on **`XChaCha20-Poly1305`** (192-bit nonce, constant-time software speed without requiring AES-NI) via [`RustCrypto/AEADs`](https://github.com/rustcrypto/aeads).
	- **Header & Slot Framing:** Magic `PKX_SEAL_V1` + format version + Argon2id params (`m_cost`, `t_cost`, `p_cost`, 16-byte salt) + 24-byte `base_nonce` + key slots (each slot carries slot type, KDF/X25519 ephemeral pubkey, and AEAD-wrapped 32-byte DEK).
	- **Per-Chunk Nonce & Truncation/Reorder Protection:** The compressed payload is split into 64 KiB chunks indexed `0..N-1`. Each chunk derives `chunk_nonce = base_nonce XOR little_endian_u64(chunk_index)` and binds **`AAD = SHA256(header_bytes) || chunk_index || is_final_chunk (u8)`**. Any reordered, dropped, modified, or truncated chunk fails Poly1305 authentication immediately before touching disk.
4. **Independent Key Slots (Zero Payload Duplication, ~100 Bytes/Slot):**
	- **Slot 0 (Primary Password):** Wraps the 32-byte DEK using **`Argon2id`** (memory-hard KDF, e.g., 64–128 MiB RAM, 3–4 iterations) via [`RustCrypto/password-hashes`](https://github.com/RustCrypto/password-hashes). Never store the password or a password hash in the binary.
	- **Slot 1 (Optional Recovery Slot — `[security].recovery`):**
		- `"key"` (Offline Recovery Key): Generates a high-entropy recovery code (`XXXX-XXXX-XXXX-XXXX-XXXX`) printed once at build time for the company's password vault.
		- `"pubkey"` (Company Master X25519 Public Key — Recommended for Enterprise): Wraps the DEK to an IT-owned X25519 public key. Build servers never hold the private recovery key, yet IT can recover any lost-password company installer offline.
		- `"none"`: Disables recovery slots.
5. **Streaming Unlock & RAM Wipe (`px-stub`):**
	- User enters the password (or toggles **"Use Recovery Key"**) in `PasswordPage`, or IT pipes it via `--password-stdin` / `--recovery-key-stdin` (never `--password=<secret>` on argv where process monitors log it).
	- `px-stub` unwraps the DEK, validates the first chunk's AEAD auth tag (instant rejection with exit code `14` and zero disk writes on wrong password), streams decryption straight into `zstd` decompression -> same-volume staging, and scrubs all key/password buffers from RAM using `zeroize` ([`RustCrypto/utils`](https://github.com/rustcrypto/utils)).

### Sealed `-packagex-detail` Behavior
When `[security].encryption = "password"` is enabled, running `-packagex-detail` **never prompts for a password** and **never leaks app metadata**. Instead, it prints only the sealed envelope encoded as **base64** (or hex)—pure unreadable noise that cannot corrupt the user's terminal with raw control bytes.

### Reproducibility Interaction
Random salts and nonces change the binary SHA-256 on every build. `packx check` warns explicitly that `encryption = "password"` disables strict bit-for-bit reproducibility (never reuse a static nonce across different payloads, which breaks AEAD security).

## 2. Reverse-Engineering (RE) Threat Model

| Scenario | Protection Guarantee |
| :--- | :--- |
| **Static RE (No Password)** — Disassembling `px-stub` in Ghidra/IDA, `strings`, 7-Zip, hex editor | **Computationally infeasible offline** under a high-entropy passphrase/recovery key and memory-hard Argon2id parameters. Disassembling `px-stub` reveals only the cipher and header layout, never the DEK or password. |
| **Dynamic RE (Authorized User With Password)** — Debugger attached after valid unlock | Once a valid password is entered and files are installed, decrypted files exist on that authorized machine. Password sealing protects the **installer package in transit and at rest**, not post-install memory on an already-authorized host. |

## 3. Free Local Company Signing (`packx sign init-ca`)

High-entropy encrypted `.exe` overlays trigger Windows Defender / SmartScreen heuristics if shipped unsigned. For internal company apps that shouldn't require a paid public EV certificate:

1. **Zero-Cost Local CA Generation:** `packx sign init-ca --org "Kastrick Internal"` uses [`rcgen`](https://github.com/rustls/rcgen) to generate a private Root CA + Code Signing certificate offline ($0).
2. **Cross-OS Signing:** `packx build --sign --ca ./internal-ca` signs the `.exe` / `.msi` / `.msix` using [`osslsigncode`](https://github.com/mtrojnar/osslsigncode) or `signtool`.
3. **Legitimate Enterprise Trust (Authenticode + SmartScreen Boundary):** PackageX never attempts to bypass SmartScreen on untrusted public PCs. Deploying the private Root CA `.cer` via Active Directory GPO or Microsoft Intune establishes **Windows Authenticode signature trust** ($0), while **Microsoft Defender SmartScreen** evaluates publisher/file reputation independently—so enterprise IT pairs the private Root CA with **WDAC / AppLocker / Defender for Endpoint certificate allow-listing** (or internal trusted-zone policies) to prevent SmartScreen prompts on new internal builds.

## 4. White-Label End-User Stealth Rules

End-users should never see `PackageX` or `px-` branding on their machine:
- **Uninstallers:** `%LOCALAPPDATA%\<package_name>\uninstaller\<app_id>.exe` (`scope = "user"`) or `%ProgramData%\<package_name>\uninstaller\<app_id>.exe` (`scope = "machine"`).
- **Staging & Rollback Temp Folders:** `.stage-<app_id>-<id>/`, `.backup-<app_id>-<id>/`, `.journal-<app_id>-<id>.log`.
- **Shell `$PATH` Markers:** `# >>> <app_id> >>> ... # <<< <app_id> <<<`.
- **PE Version Info:** Stamped with `[package].name`, `version`, and `publisher`—never `px-stub` or `PackageX`.
- **`-packagex-detail`:** Strictly a silent opt-in CLI flag for IT admins, never referenced in GUI screens.

## 5. Global Signing Key Configuration (`packx config --set-signkey`)

Developers can configure signing once globally across **all** installed toolbelt versions (`2.3`, `2.4`, etc.):
- `packx config --set-signkey=<path-or-key>` (or `--set-signkey-stdin` to keep secrets out of shell history) inspects the key format automatically:
	- **macOS Developer ID / `.p12`:** Automatically binds to the global macOS signing slot (`Belt_arm64-macos` / `Belt_x86_64-macos`).
	- **Windows Authenticode `.pfx` / Local CA:** Automatically binds to the global Windows signing slot (`Belt_x86_64-windows` / `Belt_arm64-windows`).
	- **Linux GPG Key:** Automatically binds to the global Linux signing slot.
- Private key material is stored inside the **OS native keyring** (Windows Credential Manager, macOS Keychain, Linux Secret Service)—never in plaintext `~/.packx/config.json`—with explicit override flags (`--target=macos`, `--profile=prod`) when managing multiple identities.


## 6. Protected installer operation plans and Electron references

The SDK may generate a logical Electron reference as JSON during build preparation, but the final installer must not expose that object as a loose, publicly readable JSON sidecar or top-level manifest.

The Engine's protected-payload pipeline serializes the reference into a typed, opaque control block stored alongside the application payload. The block is associated with the installer layout and is authenticated before the embedded installer helper can act on it.

The block carries:
- Exact Electron version/build, target OS and architecture.
- Exact version-pinned artifact URL, expected byte length and full SHA-256.
- Expected extracted runtime inventory/tree identity.
- Original runtime-relative paths and logical destination slot IDs.
- Application payload identity, ABI/native-module requirements and fuse/customization profile.
- Verification policy, license inventory, schema version and installer-layout identity.

The opacity guarantee applies to PackageX's internal Electron reference and operation/layout plan, not every byte of the outer native package. MSI properties/tables, MSIX package identity/manifest, PE headers and publisher/signature metadata may remain readable because the operating system needs them for installation, deployment and trust decisions. Do not claim that a native MSI/MSIX or signed executable is wholly unreadable unless a separate whole-payload sealing mode is enabled and compatible with that format.

### Integrity, authorization and confidentiality are separate

1. **Publisher authorization:** the operation plan is signed by the developer/build signing identity. The verifier key is bound to the trusted installer identity; an arbitrary key supplied by a downloaded manifest is never accepted as its own trust root.
2. **Authenticated envelope:** AEAD authentication detects changes, truncation, reordering or substitution of the protected block before any privileged operation.
3. **Default opaque protection:** the SDK's standard payload-protection format prevents casual inspection of internal paths, download references and operation metadata. It is not represented as password-grade secrecy against the owner of a machine that must execute/decrypt it.
4. **Optional password sealing:** the existing password/recovery envelope is the stronger confidentiality mode when the developer needs the payload to remain unreadable until authorized unlock.
5. **Artifact verification:** the helper checks the downloaded archive against the exact build-time-pinned digest and performs applicable OS-native publisher verification before extraction/commit.

An encryption key available to a local helper cannot be treated as a secret from a sufficiently capable local reverse engineer. Therefore encryption/obfuscation is not the security authority and does not prevent RCE by itself. Signed authorization, native installer/code signatures, authenticated data, bounded capabilities and path validation are the controls that prevent an attacker from substituting instructions or gaining arbitrary write/execute capability.

For production app-only Electron runtime restoration, an unsigned or unverifiable installer must not claim the same tamper-resistance guarantee as a correctly signed build. The build should fail or clearly mark the optimized mode as development/untrusted according to the project's signing policy.

## 7. Embedded installer helper capability boundary

The installer contains a small target-specific helper (the Windows implementation may be named script.exe internally). It is an installer component, not PackageX.exe, a package manager, a general-purpose command interpreter or an installed background runtime service.

Its fixed capabilities may include:
- Read and authenticate the installer's own protected control block.
- Download only the exact URL authorized by that block.
- Enforce HTTPS and the declared origin/redirect policy.
- Stream to bounded staging, verify full SHA-256 and applicable native signatures.
- Safely extract a declared archive within size/file-count limits.
- Write only to named logical payload slots within the active installer transaction.
- Return typed status, progress and errors to the parent installer.

It must reject arbitrary URLs, arbitrary filesystem paths, arbitrary PE/EXE offsets, memory addresses, downloaded scripts, unapproved executable launches and writes outside its transaction-owned staging root. The helper's write primitive resolves a logical slot through the authenticated installer layout; it never treats a caller-provided raw offset as authority.

The helper does not mutate the currently executing installer EXE in place. It restores runtime files into the install transaction's staging tree, then the Engine commits the verified result using the normal same-volume journal/rollback path.

Developer .s.px hooks retain their separate WASM capability allowlist. The developer-controlled updater host is also separate from the embedded installation helper; its custom behavior is subject to the developer's update-signing identity and install-scope protections.

## 8. Security acceptance criteria

- [ ] Final Electron references are not exposed as loose readable JSON sidecars.
- [ ] Protected control blocks are authenticated and bound to the final installer layout.
- [ ] Production authorization is rooted in the trusted publisher/installer identity, not a key or hash downloaded beside the artifact.
- [ ] The exact Electron URL, target, byte length, full SHA-256 and runtime inventory are build-time pinned.
- [ ] The helper accepts logical slot IDs only and cannot write arbitrary offsets or paths.
- [ ] Downloaded content is verified before extraction and before transaction commit.
- [ ] The helper never executes downloaded scripts or arbitrary binaries as instructions.
- [ ] The helper does not patch the running installer EXE in place.
- [ ] Machine-wide helper/updater configuration and staging locations are protected from unprivileged modification.
- [ ] Documentation does not claim default obfuscation provides secrecy against the machine owner or prevents RCE.
