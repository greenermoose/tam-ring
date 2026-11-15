# Ideas from oama

## Overview

- **What it is:** An OAuth2 credential manager ("OAuth credential Manager")
  that handles initial browser/device authorization and transparent token
  renewal for CLI mail tools (`mbsync`, `msmtp`, `neomutt`, `aerc`, etc.).
- **Developers:** Created and maintained by Peter Dobsan (`pdobsan`).
- **License:** 3-Clause BSD License (`BSD-3-Clause`).
- **Language:** Haskell.
- **Source Code:** <https://github.com/pdobsan/oama>.

---

## Ideas We Borrow

### 1. Zero-Daemon, On-Demand CLI Architecture (Primary Model)
- `oama` runs only when invoked (`authorize`, `access`, `renew`, `show`,
  `printenv`), making it a drop-in `PassCmd` / `passwordeval` helper with no
  resident daemon to supervise.
- This aligns directly with Tamlinux's minimal-footprint philosophy.

### 2. Strong OAuth2 Security Defaults
- **PKCE (`S256`)** code verifier/challenge (implemented in `tam-ring` using
  `tam_sha256` and `/dev/urandom`).
- Random `state` parameter to prevent CSRF on loopback callbacks.
- Dynamic ephemeral `localhost` `redirect_uri` by default.

### 3. Headless / Device Code Flow (RFC 8628)
- `authorize <account> --device` lets users authenticate on another device
  when running in a bare Linux VT (`TERM=linux`), over SSH, or on a headless
  machine without a local browser.

### 4. Encrypted Disk Storage & External Secret Commands
- Never storing refresh tokens or client secrets in plaintext config files.
- Supporting `client_id_cmd` and `client_secret_cmd` so OAuth app credentials
  can be fetched from an external password manager at runtime.

### 5. Built-In Provider Presets
- Pre-configured endpoint templates for common providers (`google`,
  `microsoft` with `tenant` support) plus custom OIDC endpoints.

---

## Ideas We Adapt

### 1. Implementation Stack & External Tooling
- **oama:** Compiled Haskell binary linking GHC runtime, HTTP/TLS libraries,
  YAML parser, and optional FFI `libsecret`.
- **tam-ring:** Pure C11 + `libtam.a` (`tam_ini`, `tam_exec`, `tam_fs_lock`,
  `tam_fs_write_atomic`, `tam_sha256`), delegating HTTPS token exchange to
  `curl` and encrypted persistence to `age`, `gpg`, or `secret-tool` via
  `tam_exec`.

### 2. Configuration Format
- **oama:** Uses YAML (`~/.config/oama/config.yaml`).
- **tam-ring:** Uses `tam_ini` (`$XDG_CONFIG_HOME/tamlinux/tam-ring/tam-ring.ini`)
  for consistency with all Tamlinux tools and zero YAML parser overhead.

---

## Ideas We Avoid

- **Haskell / GHC Runtime Dependency:** Heavy build toolchain and large
  runtime footprint are incompatible with Tamlinux's C11 + `libtam` stack.
- **Linking `libsecret` / D-Bus Directly:** Calling `secret-tool` (or
  `age`/`gpg`) via `tam_exec` avoids pulling D-Bus/GLib headers and shared
  libraries into the binary.
