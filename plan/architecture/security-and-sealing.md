# Security, Password Sealing, Local Company CA & White-Label Stealth (`px-crypto`)

PackageX supports both public software distribution and **high-security internal company installers** that remain unreadable garbage at rest and in transit without a valid password or recovery key.

## 1. Multi-Slot Envelope Encryption ("App -> Garbage -> App")

Never encrypt files one-by-one inside a readable ZIP/tar—that leaks folder trees, filenames, file counts, and sizes. Instead, `px-assembler` + `px-crypto` wrap the **entire payload, `.s.px` WASM script, and internal manifest** into one opaque encrypted blob:

1. **Single-Blob Pack First:** Deterministically pack and `zstd`-compress all payload files, `.s.px` bytecode, and the internal file manifest into one contiguous stream.
2. **Random Master Payload Key (DEK):** Generate a fresh 256-bit random Data Encryption Key (DEK) at build time.
3. **Chunked AEAD Encryption:** Encrypt the entire compressed blob in streaming 64 KiB chunks using **`XChaCha20-Poly1305`** (or `AES-256-GCM`) via [`RustCrypto/AEADs`](https://github.com/rustcrypto/aeads). AEAD ciphertext is mathematically indistinguishable from uniform random noise—opening the installer in 7-Zip, `strings`, or a hex editor reveals zero filenames, zero folders, and zero readable text.
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
| **Static RE (No Password)** — Disassembling `px-stub` in Ghidra/IDA, `strings`, 7-Zip, hex editor | **Practically uncrackable.** Disassembling `px-stub` exposes only the decryption algorithm, never the key or password. Argon2id + XChaCha20-Poly1305 with a strong company passphrase prevents offline extraction. |
| **Dynamic RE (Authorized User With Password)** — Debugger attached after valid unlock | Once a valid password is entered and files are installed, decrypted files exist on that authorized machine. Password sealing protects the **installer package in transit and at rest**, not post-install memory on an already-authorized host. |

## 3. Free Local Company Signing (`packx sign init-ca`)

High-entropy encrypted `.exe` overlays trigger Windows Defender / SmartScreen heuristics if shipped unsigned. For internal company apps that shouldn't require a paid public EV certificate:

1. **Zero-Cost Local CA Generation:** `packx sign init-ca --org "Kastrick Internal"` uses [`rcgen`](https://github.com/rustls/rcgen) to generate a private Root CA + Code Signing certificate offline ($0).
2. **Cross-OS Signing:** `packx build --sign --ca ./internal-ca` signs the `.exe` / `.msi` / `.msix` using [`osslsigncode`](https://github.com/mtrojnar/osslsigncode) or `signtool`.
3. **Legitimate Enterprise Trust (No Sketchy Bypasses):** PackageX never attempts to illegally bypass SmartScreen on untrusted public PCs. Instead, company IT deploys the single public Root CA `.cer` to **Trusted Root Certification Authorities** via Active Directory GPO or Microsoft Intune—making every PackageX-signed internal installer 100% natively trusted by Windows and SmartScreen across the company for $0.

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
