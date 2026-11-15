# `oama` Architecture & `tam-ring` Adaptation Notes

`tam-ring` is currently in pre-`v0.1.0` planning (`ROADMAP.md`), with no C source files yet, and no `oama` Haskell source tree is present locally. This document summarizes the architectural mechanisms and OAuth2 algorithms documented in `ideas/oama.md` alongside `tam-ring`'s planned C11/`libtam` design in `ROADMAP.md`.

---

## 1. Execution Model & Runtime Footprint

### `oama` (`ideas/oama.md`)
- **Zero-daemon, on-demand CLI** (`authorize`, `access`, `renew`, `show`, `printenv`): each invocation reads config, decrypts stored credentials, refreshes the access token over HTTPS if needed, writes updated state, and exits.
- Implemented in Haskell, linking the GHC runtime, HTTP/TLS libraries, a YAML parser (`~/.config/oama/config.yaml`), and optional FFI `libsecret` bindings.

### Planned `tam-ring` Adaptation (`ROADMAP.md`)
- Preserves the **daemonless, on-demand CLI** model (targeting 0–2 ms cold start and sub-megabyte RSS) in pure C11 + `libtam.a`.
- Eliminates in-process TLS/HTTP/D-Bus libraries by delegating HTTPS token requests to **`curl`** (passing secrets via stdin / `--config -`, never `argv`) and encrypted storage to **`age`**, **`gpg`**, or **`secret-tool`** via `tam_exec` (`v0.2.0`, `v0.5.0`).
- Replaces YAML with `tam_ini` (`$XDG_CONFIG_HOME/tamlinux/tam-ring/tam-ring.ini`).

---

## 2. OAuth2 Flows & Cryptographic Primitives

### `oama` (`ideas/oama.md`)
- **Authorization Code + PKCE (`S256`)**: Generates a code verifier and `S256` challenge plus a random `state` CSRF token, binds an ephemeral `localhost` HTTP callback listener, and exchanges the authorization code for access/refresh tokens.
- **RFC 8628 Device Code Flow (`--device`)**: Polls the token endpoint for headless/remote environments without a local browser callback.
- **Provider Presets & Secret Commands**: Built-in `google` and `microsoft` (with `tenant`) presets, custom OIDC endpoints, and `client_id_cmd` / `client_secret_cmd` hooks.

### Planned `tam-ring` Data Structures & Algorithms (`ROADMAP.md`)
- **PKCE & CSRF State (`v0.3.0`)**: Generates high-entropy verifiers and `state` tokens from `/dev/urandom`, computes `S256` digests using **`tam_sha256`** + URL-safe base64, and runs a **single-shot POSIX loopback HTTP listener** (`127.0.0.1` ephemeral port or configured `http_listen` IP:port).
- **Device Code Polling (`v0.5.0`)**: Implements RFC 8628 polling loop with `slow_down` interval backoff.
- **Minimal C11 JSON Extractor (`v0.2.0`)**: Extracts `access_token`, `refresh_token`, `expires_in`, `token_type`, `error`, and `error_description` from OAuth2 responses without a general-purpose JSON DOM, wiping secret/token buffers with **`explicit_bzero`** (`v1.0.0`).
