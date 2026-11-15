# Ideas from pizauth

## Overview

- **What it is:** A lightweight OAuth2 authentication daemon and CLI client
  designed to supply access tokens (`XOAUTH2` / `OAUTHBEARER`) to Unix mail
  tools (`neomutt`, `msmtp`, `fdm`, `mbsync`).
- **Developers:** Created and maintained by Laurence Tratt (`ltratt`).
- **License:** MIT / Apache-2.0 dual license.
- **Language:** Rust.
- **Source Code:** <https://github.com/ltratt/pizauth>.

---

## Ideas We Borrow

### 1. `http_listen` Fixed IP:Port Directive
- While ephemeral loopback ports are great locally, a configurable
  `http_listen` directive lets users bind the OAuth callback listener to a
  fixed IP/port—making remote authentication over `ssh -L` tunnels seamless.

### 2. `refresh_before_expiry` Lead Window
- Proactively refreshing tokens when remaining TTL falls below a configurable
  threshold (e.g., 90 seconds) prevents an access token from expiring mid-sync
  right after being handed to `mbsync` or `msmtp`.

### 3. Lifecycle Hooks (`auth_notify_cmd` & `token_event_cmd`)
- `auth_notify_cmd`: runs a user command (passing account and auth URL) when
  a refresh token is missing/expired and interactive login is needed, instead
  of silently hanging cron/background syncs.
- `token_event_cmd`: runs a user command whenever tokens change state (useful
  for custom backups or status updates).

### 4. `dump` / `restore` & Versioned `--json` Output
- `dump` / `restore` to stdout/stdin enables composable backup and external
  encryption pipelines (e.g., piping into `age` or `gpg`).
- Machine-readable `--json` status output (with a format version field) makes
  scripting and status-bar integration clean.

---

## Ideas We Adapt

### 1. Concurrency & State Safety Without a Daemon
- **pizauth:** Uses a persistent daemon to serialize refreshes and keep tokens
  in RAM.
- **tam-ring:** Achieves safe concurrent `access` calls (e.g., `mbsync` and
  `msmtp` running simultaneously) without a daemon by using `tam_fs_lock` +
  `tam_fs_write_atomic` around the encrypted token file.

### 2. Configuration Syntax
- **pizauth:** Uses a custom nginx-like `pizauth.conf`.
- **tam-ring:** Uses `tam_ini` (`tam-ring.ini`) to stay uniform with
  `tam-mail`, `tam-data`, and `libtam`.

---

## Ideas We Avoid

- **Mandatory Resident Daemon:** Running a persistent background server for
  occasional mail checks adds process supervision overhead and goes against
  `tam-ring`'s `oama`-style on-demand CLI design.
- **Unencrypted RAM-Only Default Requiring Manual `dump`/`restore`:**
  For an on-demand CLI, encrypted-at-rest persistence (`age`/`gpg`) with
  `tam_fs_lock` is simpler and survives reboots automatically.
- **Rust Toolchain & Crate Dependency Tree:** Avoided in favor of C11 +
  `libtam.a`.
