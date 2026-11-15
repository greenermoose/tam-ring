# tam-ring Roadmap

This document outlines the architectural plan and phased roadmap for `tam-ring`:
a pure-C11, zero-runtime-bloat OAuth2 credential manager built on
[`libtam`](https://github.com/greenermoose/libtam).

**Current status:** Initial repository setup. No implementation exists yet.

---

## Architectural Synthesis: `oama` + `pizauth`

`tam-ring` follows **`oama`'s on-demand CLI model** as its core architecture
because it fits Tamlinux's zero-daemon, minimal-footprint philosophy: no
long-lived background process is required just to check mail. At the same time,
`tam-ring` selectively incorporates **`pizauth`'s strongest operational
features**:

| Idea | Origin | How `tam-ring` Uses It |
| :--- | :--- | :--- |
| On-demand `authorize` / `access` / `renew` / `show` CLI | `oama` | Primary UX; called directly by `PassCmd` / `passwordeval` / `tam-mail` |
| PKCE (`S256`) + random `state` CSRF + ephemeral `localhost` redirect | `oama` | Built with `tam_sha256` and POSIX loopback sockets |
| Device Code Flow (RFC 8628) | `oama` | `--device` flag for headless / Linux VT / SSH workflows |
| Encrypted disk backends (`gpg`, `age`, `secret-tool`) | `oama` | Delegated via `tam_exec` with `tam_fs_lock` + `tam_fs_write_atomic` |
| `client_id_cmd` / `client_secret_cmd` | `oama` | Pull OAuth client credentials from external password stores |
| `http_listen` fixed IP:port directive | `pizauth` | Supports remote auth over `ssh -L` tunnels when ephemeral ports aren't usable |
| `refresh_before_expiry` lead window | `pizauth` | Proactive refresh when remaining TTL drops below threshold (default e.g. 90s) |
| `auth_notify_cmd` & `token_event_cmd` hooks | `pizauth` | Run user commands when re-auth is needed or tokens change |
| `dump` / `restore` state serialization | `pizauth` | Composable stdout/stdin state export for custom pipelines |
| Machine-readable `--json` status output | `pizauth` | Versioned JSON via `tam_str_cat_json` for scripts and status bars |

---

## Release Versioning Plan

`tam-ring` follows [Semantic Versioning (SemVer 2.0.0)](https://semver.org/):

| Version | Phase | Milestone Scope | Status |
| :--- | :--- | :--- | :--- |
| **`v0.1.0`** | **Phase 0** | Repo skeleton, Makefile, ASan/UBSan tests, `tam_ini` config, XDG paths, `printenv` | Planned |
| **`v0.2.0`** | **Phase 1** | `curl` HTTPS token client, minimal JSON extractor, `age`/`gpg` encrypted store (**MVP**) | Planned |
| **`v0.3.0`** | **Phase 2** | `authorize` flow: PKCE (`tam_sha256`), `state`, local HTTP callback listener, `http_listen` | Planned |
| **`v0.4.0`** | **Phase 3** | `access` & `renew`, `refresh_before_expiry`, `tam_fs_lock` concurrency, provider presets | Planned |
| **`v0.5.0`** | **Phase 4** | Device Code Flow (RFC 8628), `client_*_cmd`, `secret-tool` backend, `show --json` | Planned |
| **`v1.0.0`** | **Phase 5** | `auth_notify_cmd` / `token_event_cmd`, `dump`/`restore`, security audit & hardening | Planned |

---

## Phase 0: Foundation & Configuration (v0.1.0)

**Goal:** Establish the build system, sanitizer harness, and INI configuration
model.

- `Makefile` with strict C11 flags (`-std=c11 -Wall -Wextra -Werror`) and
  `make test` under ASan/UBSan.
- `tam-ring.ini` parser using `tam_ini` and `tam_xdg_path`:
  - `[storage]` backend selection (`age`, `gpg`, `secret-tool`, or `plain` for
    tests only with explicit opt-in)
  - `[account.<name>]` sections (`provider`, `client_id`, `client_secret`,
    `client_id_cmd`, `client_secret_cmd`, `scopes`, `tenant`, `http_listen`,
    `refresh_before_expiry`)
- `tam-ring printenv` to inspect resolved config/state paths and parsed
  accounts (redacting secrets).

## Phase 1: Token Store & HTTPS Exchange MVP (v0.2.0)

**Goal:** Safely persist encrypted token records and exchange/refresh tokens
over HTTPS via `curl`.

- Encrypted state store in `$XDG_STATE_HOME/tamlinux/tam-ring/`:
  - Exclusive file locking via `tam_fs_lock` so concurrent `mbsync` + `msmtp`
    invocations never race on a refresh token rotation.
  - Atomic updates via `tam_fs_write_atomic` with `0600` permissions.
  - Pluggable encryption/decryption via `tam_exec` (`age`, `gpg`, or
    `secret-tool`).
- `tam_exec` wrapper around `curl` (strict timeouts, TLS verification enabled,
  secrets passed via stdin/`--config -` rather than argv so they never leak in
  `ps`).
- Minimal, strict JSON token-response parser in C11 (extracting `access_token`,
  `refresh_token`, `expires_in`, `token_type`, `error`, `error_description`).

## Phase 2: Interactive Authorization & PKCE (v0.3.0)

**Goal:** Complete end-to-end browser-based OAuth2 login.

- PKCE (`S256`) code verifier and challenge generation using `tam_sha256` and
  `/dev/urandom` + URL-safe base64.
- Cryptographic `state` parameter verification to prevent CSRF.
- Lightweight single-shot POSIX loopback HTTP server (`127.0.0.1` ephemeral
  port by default, or fixed `http_listen` address:port for `ssh -L` remote
  authentication).
- Clean HTML/text confirmation page on callback and automatic socket shutdown.

## Phase 3: Transparent Access, Renewal & Provider Presets (v0.4.0)

**Goal:** Make `tam-ring access <account>` rock-solid for daily mail sync.

- `tam-ring access <account>`:
  - Acquires `tam_fs_lock`, decrypts token record, checks `expires_at` against
    `refresh_before_expiry` (default 90s).
  - If valid, prints `access_token` to stdout and exits `0`.
  - If expired/expiring, exchanges `refresh_token` via `curl`, atomically saves
    the updated token record (preserving rotated refresh tokens), and prints the
    new `access_token`.
- `tam-ring renew <account>` to force an immediate refresh.
- Built-in endpoint presets for `google` and `microsoft` (with `tenant` support)
  plus `custom` OIDC endpoints (`auth_url`, `token_url`, `device_url`).

## Phase 4: Device Code Flow, Secret Commands & Keyring (v0.5.0)

**Goal:** Support headless/VT environments and external secret managers.

- RFC 8628 Device Authorization Grant (`tam-ring authorize <account> --device`)
  with polling interval and `slow_down` handling.
- `client_id_cmd` and `client_secret_cmd` execution via `tam_exec` so no client
  secrets need to be stored in `tam-ring.ini`.
- FreeDesktop Secret Service backend via `secret-tool` (`tam_exec`).
- `tam-ring show <account> [--json]` outputting versioned metadata via
  `tam_str_cat_json` (never printing raw tokens unless explicitly requested).

## Phase 5: Hooks, Dump/Restore & v1.0.0 Hardening (v1.0.0)

**Goal:** Operational completeness, scriptability, and security hardening.

- `pizauth`-inspired lifecycle hooks via `tam_exec`:
  - `auth_notify_cmd` when a refresh token is missing/revoked and user
    re-authorization is required (passing `TAM_RING_ACCOUNT` and
    `TAM_RING_URL` in environment).
  - `token_event_cmd` when tokens are acquired or refreshed.
- `tam-ring dump` / `tam-ring restore` for portable, composable backup or
  pipe-through-`age` workflows.
- Explicit memory zeroing (`explicit_bzero`) on token/secret buffers,
  comprehensive ASan/UBSan unit and mock-HTTP integration tests, and man-page
  documentation.
