# Product Strategy & Open-Core Editions

PackageX follows an **open-core distribution model**: the free core delivers effortless cross-OS packaging and white-label polish for open-source and indie developers, while paid Pro/Enterprise tiers target company fleet scale, governance, and compliance.

## 1. Free Open-Source & Indie Core
- Full cross-OS installer assembly (Windows, macOS without Xcode, Linux), CAS toolbelt, rollback runtime, WASM `.s.px`, `.pxs` UI, `packx test` Desktop sandbox, local CA signing, and basic auto-updater.
- **High-Velocity Dev CLI Features:**
	- `packx init --from`: Auto-detects `Cargo.toml`, `package.json`, `pyproject.toml`, `go.mod`, or `.csproj` and scaffolds a working `.px` manifest + `wizard.pxs`.
	- `packx init-ci github`: Emits a ready-to-push GitHub Actions / GitLab CI release workflow for multi-OS builds and `update-feed.json`.
	- **One-Line Web Bootstrappers:** Optional `install.sh` (`curl | sh`) and `install.ps1` (`irm | iex`) emitter alongside native installers.
	- `packx preview wizard.pxs`: Live hot-reload native UI preview without rebuilding the installer.

## 2. Paid Company / Enterprise Tier
- **Staged Rollouts & Instant Kill-Switch:** Canary/beta/stable update channels, percentage rollouts (`5%` -> `25%` -> `100%`), and instant feed-level pause/rollback switches.
- **Enterprise Fleet Policy Overrides:** Generates Windows GPO `.admx` templates and macOS `.mobileconfig` profiles so IT admins can lock install directories, force silent mode, disable auto-updates on air-gapped subnets, or point to internal mirrors without rebuilding.
- **Cloud KMS / HSM Signing & Compliance Packs:** Native plugins for AWS KMS, Google Cloud KMS, HashiCorp Vault, and YubiHSM, plus automated SLSA provenance and SOC-2 audit bundles (SBOM + `-packagex-detail --json` + signing receipts).
- **Self-Hosted `.pxbelt` Mirror & Recovery Key Escrow:** SSO-protected (OIDC/SAML) internal mirror for air-gapped SDK belts, company installers, and centralized **X25519 Recovery Key Escrow** so sealed apps remain recoverable across employee turnover.
- **Install & Rollback Telemetry Webhooks:** Opt-in enterprise endpoint reporting canonical exit codes (`exit-codes-and-errors.md`) for real-time fleet health monitoring.
