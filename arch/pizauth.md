# `pizauth` Architecture & `tam-ring` Adaptation Notes

`tam-ring` is currently in pre-`v0.1.0` planning (`ROADMAP.md`), with no C source files yet, and no `pizauth` Rust source tree is present locally. This document summarizes the architectural mechanisms documented in `ideas/pizauth.md` alongside `tam-ring`'s planned C11/`libtam` design in `ROADMAP.md`.

---

## 1. Concurrency, Serialization & Token Storage

### `pizauth` (`ideas/pizauth.md`)
- **Persistent background daemon + CLI client**: The daemon serializes token refreshes in memory to prevent concurrent clients (e.g. `mbsync` and `msmtp`) from racing on rotating refresh tokens, and holds tokens in **unencrypted RAM** by default.
- Configured via a custom nginx-style `pizauth.conf`.

### Planned `tam-ring` Adaptation (`ROADMAP.md`, `v0.4.0`)
- Replaces the resident daemon and unencrypted RAM cache with **file-locked, encrypted-at-rest state** under `$XDG_STATE_HOME/tamlinux/tam-ring/`:
  1. Acquire an exclusive lock via **`tam_fs_lock`** on the per-account state file.
  2. Decrypt the state (`age` / `gpg` / `secret-tool` via `tam_exec`).
  3. Check `expires_at` against `refresh_before_expiry` (default 90s).
  4. If refresh is needed, invoke `curl` and atomically persist the new token payload via **`tam_fs_write_atomic`** with `0600` permissions before releasing the lock.
- This prevents refresh-token rotation races across concurrent callers without requiring a background daemon.

---

## 2. Operational Lifecycle & Serialization Features

### `pizauth` (`ideas/pizauth.md`)
- **`http_listen`**: Fixed IP:port binding for `ssh -L` port-forwarded remote authorization.
- **`refresh_before_expiry`**: Proactive refresh lead window before token expiration.
- **Lifecycle Hooks**: `auth_notify_cmd` (alerts the user when interactive re-authorization is required) and `token_event_cmd` (fires on token state transitions).
- **State Portability & Machine Output**: `dump` / `restore` (stdout/stdin serialization) and versioned `--json` status output.

### Planned `tam-ring` Adaptation (`ROADMAP.md`)
- **`http_listen`** (`v0.3.0`) and **`refresh_before_expiry`** (`v0.4.0`) integrated into `tam-ring.ini` (`tam_ini`).
- **`show --json`** (`v0.5.0`) serialized via **`tam_str_cat_json`** with a versioned schema.
- **`dump` / `restore`** and **`auth_notify_cmd` / `token_event_cmd`** (`v1.0.0`) executed via `tam_exec` with `TAM_RING_ACCOUNT` and `TAM_RING_URL` environment variables, plus `explicit_bzero` memory scrubbing.
