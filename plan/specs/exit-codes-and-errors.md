# Canonical Exit Codes & Structured JSON Log Schema

All `packx` CLI commands and assembled `px-stub` installers emit deterministic exit codes and optional structured JSON Lines (`--log-json=<path>`) so Intune, SCCM, Jamf, Ansible, and CI pipelines can automate retries and alerting reliably.

## 1. Standardized Exit Codes

| Exit Code | Constant Name | Applies To | Meaning |
| :--- | :--- | :--- | :--- |
| `0` | `OK` | CLI & Installer | Operation or installation completed cleanly. |
| `1` | `GENERIC_ERROR` | CLI & Installer | Unclassified fatal error (details in `stderr` / JSON log). |
| `2` | `ROLLED_BACK` | Installer | Step failed or user cancelled; rollback journal restored host cleanly. |
| `3` | `ROLLBACK_PARTIAL` | Installer | Rollback encountered an unrecoverable OS error (manual cleanup path logged). |
| `10` | `ELEVATION_REQUIRED` | CLI (`check`) & Installer | Target path/service requires admin/root privileges while `elevate = false` or running un-elevated in `--silent` mode. |
| `11` | `RESOURCE_LOCKED` | Installer (`--dry-run` / install) | Target binary is locked by a running process or required network port is occupied. |
| `12` | `INSUFFICIENT_DISK` | Installer (`--dry-run` / install) | Target volume lacks free space for `estimated_size_bytes` + staging headroom. |
| `13` | `OFFLINE_BELT_MISSING` | CLI (`--offline` / `--locked`) | Required toolbelt engine or target sub-belt is missing locally and network access is disabled. |
| `14` | `SEALED_AUTH_FAILED` | Installer | Invalid password or recovery key (AEAD auth tag mismatch); zero disk writes occurred. |
| `15` | `INTEGRITY_MISMATCH` | CLI & Installer | SHA-256 / BLAKE3 verification failed on a downloaded SDK, dependency, or embedded payload file. |
| `16` | `LOCK_TIMEOUT` | CLI (`belt`) | Exceeded `--lock-timeout` waiting for another process holding `~/.packx/toolbelt.lock`. |

## 2. Structured JSON Log Event Schema (`--log-json`)

Each line written to `--log-json` is a self-contained JSON object (NDJSON) using neutral, white-label fields:

```json
{"ts":"2026-11-20T04:15:02Z","level":"info","app_id":"com.kastrick.devtool","phase":"install","step_id":"extract-app","event":"step_complete","progress":85,"message":"Verified 12 files (SHA-256)"}
```

Required fields: `ts` (UTC ISO-8601), `level` (`info` | `warn` | `error`), `app_id`, `phase`, `event`, and `message`. Optional fields: `step_id`, `progress` (`0`–`100`), `exit_code`, and `rollback_actions`.
